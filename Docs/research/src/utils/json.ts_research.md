# src/utils/json.ts 研究文档

## 场景与职责

`json.ts` 是 Claude Code CLI 的 JSON 处理工具箱，提供安全、高性能、带缓存的 JSON/JSONC/JSONL 解析能力。由于 CLI 需要频繁读取和解析配置文件（如 `.mcp.json`、`settings.json`）、会话转录文件（`.jsonl`）和工具结果，该模块在性能和健壮性上做了大量优化。

核心职责：
1. **安全 JSON 解析**：带 LRU 缓存的 `safeParseJSON`，避免重复解析相同的小字符串。
2. **JSONC（带注释的 JSON）解析**：支持 VS Code 风格的配置文件。
3. **JSONC 数组修改**：在不破坏注释和格式的前提下向 JSONC 数组追加元素。
4. **JSONL 解析**：支持字符串和 Buffer 输入，优先使用 Bun 的原生 `Bun.JSONL.parseChunk` 以获得最佳性能。
5. **大 JSONL 文件读取**：支持只读取文件尾部（最多 100MB），避免加载数 GB 的会话文件到内存。

调用方非常广泛，包括 `src/utils/config.ts`、`src/utils/sessionStorage.ts`、`src/utils/cronTasks.ts`、`src/utils/sessionTitle.ts` 等。

## 功能点目的

### 1. `safeParseJSON`
带 LRU 缓存的 `JSON.parse` 包装器：
- **缓存策略**：使用 `memoizeWithLRU`，最多缓存 50 条解析结果。
- **缓存 key 大小限制**：输入字符串超过 8KB 时跳过缓存，防止大配置文件（如 `~/.claude.json`）占用过多内存。
- **错误处理**：解析失败时可选地调用 `logError` 记录错误，返回 `null`。
- **null 值缓存**：使用 discriminated union `{ ok: true; value: unknown } | { ok: false }` 包装结果，解决 `JSON.parse("null")` 返回 `null` 与 memoize 的 `NonNullable` 要求冲突的问题。

### 2. `safeParseJSONC`
基于 `jsonc-parser/lib/esm/main.js` 的 JSONC 解析器，支持注释、尾随逗号等。会先调用 `stripBOM` 去除 UTF-8 BOM（PowerShell 5.x 默认输出带 BOM）。

### 3. `parseJSONL` / `readJSONLFile`
JSON Lines 解析器：
- **Bun 优化路径**：若运行在 Bun 环境中且 `Bun.JSONL.parseChunk` 可用，优先使用原生实现。该实现支持从流中恢复——即使遇到损坏行，也会跳过换行后继续解析。
- **回退路径**：字符串和 Buffer 分别使用 `indexOf('\n')` / `indexOf(0x0a)` 逐行扫描，跳过损坏行和空行。
- **大文件支持**：`readJSONLFile` 对超过 100MB 的文件只读取尾部 100MB，并跳过第一个不完整的行。这避免了在处理超大会话文件时 OOM。

### 4. `addItemToJSONCArray`
向 JSONC 数组追加元素，同时保留注释和格式：
- 使用 `jsonc-parser` 的 `parse` → `modify` → `applyEdits` 流程。
- 若目标为空数组，在索引 0 插入；否则在末尾追加（`isArrayInsertion: true`）。
- 多种降级策略：若 JSONC 解析失败、内容为空、非数组，则回退到标准 `JSON.stringify`。

## 具体技术实现

### LRU 缓存实现细节
```typescript
const PARSE_CACHE_MAX_KEY_BYTES = 8 * 1024

function parseJSONUncached(json: string, shouldLogError: boolean): CachedParse {
  try {
    return { ok: true, value: JSON.parse(stripBOM(json)) }
  } catch (e) {
    if (shouldLogError) logError(e)
    return { ok: false }
  }
}

const parseJSONCached = memoizeWithLRU(parseJSONUncached, json => json, 50)
```

注意：`shouldLogError` 被**故意排除在缓存 key 之外**（注释说明：匹配旧 lodash memoize 的默认行为，只以第一个参数为 key）。这意味着 `safeParseJSON(badJson, false)` 和 `safeParseJSON(badJson, true)` 会共享缓存结果。

### Bun JSONL 解析的容错
```typescript
function parseJSONLBun<T>(data: string | Buffer): T[] {
  const parse = bunJSONLParse as BunJSONLParseChunk
  const len = data.length
  const result = parse(data)
  if (!result.error || result.done || result.read >= len) {
    return result.values as T[]
  }
  // 遇到错误时，从 result.read 位置开始，找到下一个换行符继续解析
  let values = result.values as T[]
  let offset = result.read
  while (offset < len) {
    const newlineIndex = /* ... */
    if (newlineIndex === -1) break
    offset = newlineIndex + 1
    const next = parse(data, offset)
    if (next.values.length > 0) values = values.concat(next.values as T[])
    if (!next.error || next.done || next.read >= len) break
    offset = next.read
  }
  return values
}
```

### 大 JSONL 文件尾部读取
```typescript
const MAX_JSONL_READ_BYTES = 100 * 1024 * 1024

export async function readJSONLFile<T>(filePath: string): Promise<T[]> {
  const { size } = await stat(filePath)
  if (size <= MAX_JSONL_READ_BYTES) {
    return parseJSONL<T>(await readFile(filePath))
  }
  await using fd = await open(filePath, 'r')
  const buf = Buffer.allocUnsafe(MAX_JSONL_READ_BYTES)
  // ... 从 size - MAX_JSONL_READ_BYTES 处读取 ...
  const newlineIndex = buf.indexOf(0x0a)
  if (newlineIndex !== -1 && newlineIndex < totalRead - 1) {
    return parseJSONL<T>(buf.subarray(newlineIndex + 1, totalRead))
  }
  return parseJSONL<T>(buf.subarray(0, totalRead))
}
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/json.ts:31-58` | `safeParseJSON` 带缓存安全解析 |
| `src/utils/json.ts:65-76` | `safeParseJSONC` JSONC 解析 |
| `src/utils/json.ts:89-190` | `parseJSONL` JSONL 解析（含 Bun 优化路径） |
| `src/utils/json.ts:194-226` | `readJSONLFile` 大 JSONL 文件尾部读取 |
| `src/utils/json.ts:228-277` | `addItemToJSONCArray` JSONC 数组追加 |
| `src/utils/jsonRead.ts` | `stripBOM` BOM 去除工具 |
| `src/utils/memoize.ts` | `memoizeWithLRU` LRU 缓存实现 |
| `src/utils/slowOperations.ts` | `jsonStringify` 慢操作包装 |
| `src/utils/log.ts` | `logError` 错误日志 |

## 依赖与外部交互

### 内部依赖
- `./jsonRead.js`：`stripBOM`
- `./log.js`：`logError`
- `./memoize.js`：`memoizeWithLRU`
- `./slowOperations.js`：`jsonStringify`
- `fs/promises`：`open`, `readFile`, `stat`

### 外部依赖
- `jsonc-parser/lib/esm/main.js`：`parse`, `modify`, `applyEdits`
- `Bun.JSONL.parseChunk`（运行时可选）

### 调用方
- `src/utils/config.ts`
- `src/utils/sessionStorage.ts`
- `src/utils/cronTasks.ts`
- `src/utils/sessionTitle.ts`
- `src/utils/cronTasksLock.ts`
- `src/utils/claudeDesktop.ts`
- `src/utils/teleport.tsx`
- `src/utils/attribution.ts`
- `src/utils/stats.ts`
- `src/utils/messages.ts`

## 风险、边界与改进建议

### 风险与边界
1. **缓存 key 过大问题**：虽然设置了 8KB 的缓存 key 上限，但 LRU 本身还会存储 key 列表。如果输入字符串频繁在 8KB 附近波动，缓存的内存占用仍可能较高。
2. **`shouldLogError` 不在缓存 key 中**：这可能导致 `safeParseJSON(badJson, true)` 命中了 `safeParseJSON(badJson, false)` 的缓存，从而没有记录错误。虽然这在大多数情况下是可接受的，但在调试时可能造成困惑。
3. **Bun JSONL 的兼容性问题**：`Bun.JSONL.parseChunk` 是 Bun 特有的非标准 API。如果未来迁移到 Node.js 或 Deno，这部分优化路径将完全失效，但回退逻辑能正常工作。
4. **`readJSONLFile` 的 100MB 假设**：注释说明 "the longest context window we support is ~2M tokens, which is well under 100 MB of JSONL"。但如果未来支持更长的上下文窗口，或单条消息变得非常大（如包含大量 base64 图像），100MB 可能不足以覆盖最近的有意义内容。
5. **`addItemToJSONCArray` 的格式硬编码**：`tabSize: 4` 和 `insertSpaces: true` 是硬编码的。如果用户的 JSONC 文件使用 2 空格或 tab 缩进，追加后的格式会不一致。

### 改进建议
1. **动态缩进检测**：在 `addItemToJSONCArray` 中检测现有 JSONC 文件的缩进风格（2/4 空格或 tab），并传入对应的 `formattingOptions`。
2. **缓存命中率监控**：在 debug 模式下记录 `safeParseJSON` 的缓存命中/未命中次数，帮助优化缓存大小限制。
3. **流式 JSONL 解析器**：对于 Node.js 环境，可以考虑引入 `stream-json` 或类似库，提供与 Bun `JSONL.parseChunk` 相近的性能，避免纯 JavaScript 回退路径的逐行扫描开销。
4. **更智能的尾部读取**：`readJSONLFile` 目前固定读取 100MB。可以根据文件大小和平均行长度动态调整，或支持按"最后 N 条消息"而非"最后 100MB"读取。
5. **错误上下文增强**：`safeParseJSON` 在记录错误时，可以附带输入字符串的前 200 个字符（去除敏感信息），帮助定位是哪个配置文件损坏。
