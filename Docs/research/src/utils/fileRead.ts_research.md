# 研究文档：src/utils/fileRead.ts

## 场景与职责

`fileRead.ts` 是 Claude Code 中**同步文件读取的叶子模块**，专门从 `src/utils/file.ts` 中拆分出来，以打破一个严重的设置层强连通分量（SCC）：`file.ts` 原本通过 `log.ts → types/logs.ts → types/message.ts → Tool.ts → commands.ts` 拉入大量无关依赖。任何只需要 `readFileSync` 的模块如果直接从 `file.ts` 导入，会连带编译/打包整个链条。

该模块只依赖：
- `./debug.js`（`logForDebugging`）
- `./fsOperations.js`（`getFsImplementation`, `safeResolvePath`）

两者最终只终止于 Node.js 内置模块，因此是**leaf-safe**的。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `detectEncodingForResolvedPath(resolvedPath)` | 通过读取文件前 4096 字节检测编码（UTF-8、UTF-16LE、BOM）。 |
| `detectLineEndingsForString(content)` | 对字符串采样统计 `\r\n` 与 `\n` 数量，判定换行风格（CRLF / LF）。 |
| `readFileSyncWithMetadata(filePath)` | 一次性读取文件并返回 `{ content, encoding, lineEndings }`，且将 CRLF 统一规范化为 LF。 |
| `readFileSync(filePath)` | `readFileSyncWithMetadata` 的简化版，只返回规范化后的内容字符串。 |
| `LineEndingType` | `'CRLF' | 'LF'` 类型别名。 |

## 具体技术实现

### 编码检测（`detectEncodingForResolvedPath`）

```ts
const { buffer, bytesRead } = getFsImplementation().readSync(resolvedPath, { length: 4096 })
```

逻辑：
1. `bytesRead === 0` → 空文件默认 `'utf8'`（修复了早期用 `'ascii'` 导致 emoji/CJK 写入损坏的 bug）。
2. 前 2 字节为 `0xFF 0xFE` → `'utf16le'`。
3. 前 3 字节为 `0xEF 0xBB 0xBF`（UTF-8 BOM）→ `'utf8'`。
4. 其他非空文件 → 默认 `'utf8'`（而非 ascii，确保 Unicode 兼容）。

### 换行检测（`detectLineEndingsForString`）

逐字符扫描：
- 遇到 `\n` 时检查前一个字符是否为 `\r`，分别累加 `crlfCount` 和 `lfCount`。
- `crlfCount > lfCount` 返回 `'CRLF'`，否则 `'LF'`。

### 元数据读取（`readFileSyncWithMetadata`）

```ts
const { resolvedPath, isSymlink } = safeResolvePath(fs, filePath)
const encoding = detectEncodingForResolvedPath(resolvedPath)
const raw = fs.readFileSync(resolvedPath, { encoding })
const lineEndings = detectLineEndingsForString(raw.slice(0, 4096))
return {
  content: raw.replaceAll('\r\n', '\n'),
  encoding,
  lineEndings,
}
```

- 通过 `safeResolvePath` 处理符号链接，若穿透 symlink 则打 debug log。
- 换行检测在 CRLF 规范化**之前**进行，避免信息丢失。
- 只取前 4096 个 code unit 做换行采样（对 ASCII 换行符而言与 4096 字节等价）。

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/utils/file.ts:24` | `detectEncodingForResolvedPath`, `detectLineEndingsForString`, `LineEndingType` | `file.ts` 重新包装为带错误处理和路径解析的 `detectFileEncoding` / `detectLineEndings`。 |
| `src/utils/queryHelpers.ts:25` | `readFileSyncWithMetadata` | 在 query helper 中读取文件。 |

### 被调用方

- `src/utils/debug.js`：`logForDebugging`
- `src/utils/fsOperations.js`：`getFsImplementation`, `safeResolvePath`

## 依赖与外部交互

- 完全无外部网络或配置依赖。
- 所有 IO 通过 `getFsImplementation()` 获取的 fs 抽象层进行，便于测试注入/mock。

## 风险、边界与改进建议

### 风险

1. **同步 IO 阻塞**：所有读取都是 `readSync` / `readFileSync`，在慢速网络磁盘或超大文件场景下会阻塞事件循环。不过该模块的定位就是“同步快速读取”，调用方已知情。
2. **编码检测局限**：只读前 4096 字节，若文件前半部分纯 ASCII、后半部分含 UTF-16LE 内容，会误判为 utf8。
3. **换行风格误判**：若文件前 4096 字符的换行比例不具有代表性（如头部是 LF、主体是 CRLF），`lineEndings` 可能给出错误信号。

### 边界

- 不处理 Mac 经典换行 `\r`（Classic Mac OS 9 之前）。
- 不返回原始 BOM 信息（读取后由调用方决定是否保留/剥离）。
- `readFileSyncWithMetadata` 总是将内容规范化为 LF；若调用方需要写回 CRLF，需结合返回的 `lineEndings` 在写入时重新转换（`file.ts` 的 `writeTextContent` 已做此处理）。

### 改进建议

1. **流式大文件支持**：若未来需要读取远超内存的文件，可在此模块外新增流式接口，保持该模块的 leaf-safe 定位不变。
2. **更精确的编码检测**：可考虑引入 `chardet` 或 `jschardet` 做更全面的编码推断，但需权衡增加的 bundle 体积。
3. **测试覆盖**：建议补充对空文件、BOM 文件、UTF-16LE 文件、纯 CRLF / 混合换行文件的单元测试。
