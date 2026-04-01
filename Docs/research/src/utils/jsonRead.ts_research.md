# src/utils/jsonRead.ts 研究文档

## 场景与职责

`jsonRead.ts` 是一个极简的叶子工具模块，只有一个职责：**去除 UTF-8 BOM（Byte Order Mark）**。它从 `json.ts` 中提取出来，目的是打破循环依赖：`json.ts` 需要 `stripBOM`，但如果 `stripBOM` 放在 `json.ts` 中，会导致 `settings → json → log → types/logs → … → settings` 的导入循环。

通过将 `stripBOM` 提取到一个无依赖的叶子模块中，任何需要读取 JSON/JSONC 文件的模块都可以安全导入，而不会触发不必要的依赖链。

调用方包括：
- `src/utils/json.ts`：`safeParseJSON`, `safeParseJSONC`
- `src/utils/config.ts`：配置文件读取
- `src/utils/syncCacheState.ts`（通过内联 `jsonParse` + `stripBOM`）

## 功能点目的

### `stripBOM`
去除字符串开头的 UTF-8 BOM 字符（`\uFEFF`）。

```typescript
const UTF8_BOM = '\uFEFF'

export function stripBOM(content: string): string {
  return content.startsWith(UTF8_BOM) ? content.slice(1) : content
}
```

### 为什么需要它？
PowerShell 5.x 的 `Out-File` 和 `Set-Content` 命令默认以 UTF-8 with BOM 编码写入文件。如果用户在这些环境中编辑了 Claude Code 的配置文件（如 `~/.claude/settings.json`），文件开头会包含 `EF BB BF`（UTF-8 BOM 的字节表示）。

如果不去除 BOM，`JSON.parse` 会将其解析为不可见字符并抛出 `SyntaxError: Unexpected token`。

## 具体技术实现

### 实现极简性
整个模块仅 16 行代码，无外部依赖，无状态，无副作用。每次调用都是纯函数式的字符串操作。

### 性能特征
- `String.prototype.startsWith` 是 O(1) 操作（通常由引擎优化为直接比较第一个 code unit）。
- `slice(1)` 在现代 JS 引擎中对于小字符串通常是 O(1) 的（copy-on-write 或 rope 优化），即使不是，也只是复制一个字符串引用。

### 与 Buffer BOM 处理的关系
注意：`stripBOM` 操作的是**字符串**，而非 `Buffer`。在 `json.ts` 的 `parseJSONLBuffer` 中，Buffer 的 BOM 是通过直接检查字节值处理的：

```typescript
if (buf[0] === 0xef && buf[1] === 0xbb && buf[2] === 0xbf) {
  start = 3
}
```

`stripBOM` 本身不处理 Buffer，因为调用方通常在 `fs.readFile(path, 'utf-8')` 之后获得字符串。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/jsonRead.ts:12-16` | `stripBOM` 函数 |
| `src/utils/json.ts:33,71` | `safeParseJSON` 和 `safeParseJSONC` 中调用 `stripBOM` |
| `src/utils/config.ts:26` | 配置文件读取中调用 `stripBOM` |

## 依赖与外部交互

### 依赖
- 无任何内部或外部依赖。

### 调用方
- `src/utils/json.ts`
- `src/utils/config.ts`
- `src/utils/syncCacheState.ts`（注释提到使用 `stripBOM + jsonParse inline`）

## 风险、边界与改进建议

### 风险与边界
1. **仅处理 UTF-8 BOM**：`\uFEFF` 也是 UTF-16 BOM 的字符表示，但 `stripBOM` 只去除开头的第一个 `\uFEFF`。如果文件使用 UTF-16 LE/BE 编码，第一个 code unit 不会是 `\uFEFF`（在 Node.js 的 `readFile(..., 'utf-8')` 中，UTF-16 文件会被正确解码为字符串，BOM 仍表现为开头的 `\uFEFF`），所以 `stripBOM` 仍然有效。但如果文件以其他编码（如 UTF-32）保存且未正确解码，问题不在 `stripBOM` 的职责范围内。
2. **不处理中间 BOM**：`stripBOM` 只去除字符串**开头**的 BOM。如果 BOM 出现在文件中间（这通常是文件损坏的表现），它不会被处理。但这符合预期——中间 BOM 是异常数据，应该暴露出来以便发现文件损坏。
3. **空字符串安全**：`''.startsWith('\uFEFF')` 返回 `false`，`''.slice(1)` 返回 `''`，所以空字符串输入是安全的。
4. **多次调用无害**：若一个字符串已经被 `stripBOM` 处理过，再次调用不会产生任何变化（因为开头已经没有 BOM 了）。

### 改进建议
1. **扩展为通用文本清理器**：可以考虑将模块扩展为 `textRead.ts`，增加去除尾随空白、统一换行符（`\r\n` → `\n`）等功能。但这需要评估是否值得打破其"叶子模块"的简洁性。
2. **文档化循环依赖背景**：模块顶部的注释已经说明了提取原因，但可以更具体地指出循环依赖涉及哪些模块，帮助未来维护者理解为什么不能将 `stripBOM` 移回 `json.ts`。
3. **添加单元测试**：虽然函数极其简单，但它是大量配置解析的基础。一个单元测试可以覆盖：带 BOM 字符串、不带 BOM 字符串、空字符串、仅 BOM 字符串。
4. **考虑 Buffer 版本**：虽然当前调用方都使用字符串，但如果未来有模块需要在 Buffer 阶段就去 BOM，可以考虑添加 `stripBOMBuffer(buf: Buffer): Buffer` 函数。
