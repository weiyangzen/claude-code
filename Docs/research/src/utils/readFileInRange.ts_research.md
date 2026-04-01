# 研究文档：src/utils/readFileInRange.ts

## 场景与职责

`readFileInRange.ts` 是 Claude Code 中所有“按行范围读取文件”功能的底层引擎。它服务于两类截然不同的调用方：

1. **工具层**：`FileReadTool` 在读取超大文本文件时，通过 `offset` + `limit` 分块返回内容，避免一次性将数百 MB 文本塞进上下文。
2. **交互层**：`QuickOpenDialog` 在文件预览窗格中需要快速拉取前 N 行做语法高亮预览；`GlobalSearchDialog` 与 `memoryScan.ts` 等也依赖它做局部文件读取。

模块的核心职责是：**在支持 UTF-8 BOM 与 CRLF 清洗的前提下，以最低开销返回指定行区间 `[offset, offset+maxLines)` 的内容，同时给出总行数、总字节数、mtime 等元数据。**

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **Fast Path** | 对 <10 MB 的普通文件使用 `fs/promises.readFile` 一次性读入内存后按 `\n` 切分，避开 `createReadStream` 的 per-chunk async 开销，实测约 2× 提速。 |
| **Streaming Path** | 对超大文件、FIFO、设备文件等使用 `createReadStream` + 手动 `indexOf('\n')` 扫描，只保留目标区间内的行，保证读取 100 GB 文件的第 1 行也不会爆 RSS。 |
| **BOM / CRLF 清洗** | 两种路径均统一去掉 UTF-8 BOM (`0xfeff`) 与行尾 `\r`，确保下游拿到的是干净 LF 文本。 |
| **maxBytes 双模式** | `truncateOnByteLimit=false`（默认）时文件/流超限时抛 `FileTooLargeError`；`true` 时则在目标区间内按完整行截断，返回 `truncatedByBytes` 标记且不抛错。 |
| **AbortSignal 支持** | 两种路径均接受 `AbortSignal`，支持用户取消或超时中断。 |

---

## 具体技术实现

### 1. 入口函数 `readFileInRange`

```ts
export async function readFileInRange(
  filePath: string,
  offset = 0,
  maxLines?: number,
  maxBytes?: number,
  signal?: AbortSignal,
  options?: { truncateOnByteLimit?: boolean },
): Promise<ReadFileRangeResult>
```

流程：
1. `signal?.throwIfAborted()` 前置检查。
2. `fsStat(filePath)` 决定走哪条路径：
   - 目录 → 抛 `EISDIR` 错误。
   - 普通文件且 `size < 10 MB` → Fast Path。
   - 其他（大文件、pipe、device）→ Streaming Path。

### 2. Fast Path：`readFileInRangeFast`

- 一次性读入完整文本，BOM 清洗后逐字符扫描 `\n`。
- 使用 `Buffer.byteLength(line)` 精确计算 UTF-8 字节数，用于 `truncateOnByteLimit` 判定。
- 时间复杂度 O(文件大小)，空间复杂度 O(选中行数)。

### 3. Streaming Path：`readFileInRangeStreaming`

核心设计：**所有事件处理器都是模块级具名函数，通过 `.bind(state)` 传递上下文，避免闭包捕获。**

```ts
type StreamState = {
  stream: ReturnType<typeof createReadStream>
  offset: number
  endLine: number
  maxBytes: number | undefined
  truncateOnByteLimit: boolean
  resolve: (value: ReadFileRangeResult) => void
  totalBytesRead: number
  selectedBytes: number
  truncatedByBytes: boolean
  currentLineIndex: number
  selectedLines: string[]
  partial: string
  isFirstChunk: boolean
  resolveMtime: (ms: number) => void
  mtimeReady: Promise<number>
}
```

事件流：
- `'open'` (once) → `fstat(fd)` 获取 `mtimeMs`，解决 `mtime` 来源。
- `'data'` → `streamOnData`：
  - 首 chunk 去 BOM。
  - 累加 `totalBytesRead`；非 truncate 模式下若超 `maxBytes` 则 `stream.destroy(new FileTooLargeError(...))`。
  - 按 `\n` 切分完整行，仅当 `currentLineIndex` 落在 `[offset, endLine)` 时入 `selectedLines`。
  - 行尾残留 fragment 仅在目标区间内才保留到 `partial`，否则直接丢弃，防止超大单行文件内存泄漏。
- `'end'` → `streamOnEnd`：
  - 处理最后一行 `partial`。
  - 等待 `mtimeReady` 后 `resolve` 结果。
- `'error'` (once) → 直接 `reject`。

**高水位线与性能**：`highWaterMark: 512 * 1024`（512 KB），在吞吐与内存之间取平衡。

### 4. `FileTooLargeError`

继承 `Error`，携带 `sizeInBytes` 与 `maxSizeBytes`，错误消息中调用 `formatFileSize` 做人类可读格式化，提示用户使用 `offset` 与 `limit` 分块读取。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/readFileInRange.ts:73-122` | 公共入口 `readFileInRange`，路径决策。 |
| `src/utils/readFileInRange.ts:128-194` | Fast Path 实现。 |
| `src/utils/readFileInRange.ts:200-342` | Streaming Path 状态机与事件处理器。 |
| `src/utils/readFileInRange.ts:344-383` | `readFileInRangeStreaming` Promise 包装与流生命周期管理。 |
| `src/tools/FileReadTool/FileReadTool.ts:496-651` | 主要调用方：FileReadTool 的 `call()` 与 `callInner()`。 |
| `src/components/QuickOpenDialog.tsx:99-118` | 交互调用方：预览前 N 行。 |
| `src/utils/format.ts:9-23` | 被依赖：`formatFileSize` 用于错误消息。 |

---

## 依赖与外部交互

- **Node.js 内置模块**：`fs` (`createReadStream`, `fstat`)、`fs/promises` (`stat`, `readFile`)。
- **内部依赖**：`./format.js` 中的 `formatFileSize`。
- **无第三方 npm 依赖**：纯 Node.js 实现，保证在 bundled/Bun 环境下同样可用。
- **调用方**：
  - `src/tools/FileReadTool/FileReadTool.ts`（工具层主入口）
  - `src/components/QuickOpenDialog.tsx`（TUI 预览）
  - `src/components/GlobalSearchDialog.tsx`
  - `src/cli/print.ts`
  - `src/memdir/memoryScan.ts`
  - `src/utils/attachments.ts`

---

## 风险、边界与改进建议

### 风险与边界

1. **Streaming Path 的 `totalBytesRead` 统计偏差**：`totalBytesRead` 累加的是 chunk 的 `Buffer.byteLength(chunk)`，而 `totalBytes` 返回的是这个值。对于多字节 UTF-8 字符被截断在 chunk 边界的情况，虽然 Node.js `createReadStream({ encoding: 'utf8' })` 会自动保证 chunk 边界在合法码点处，但 `totalBytesRead` 仍是各 chunk 字节长度之和，与文件实际大小理论上相等，对 pipe/FIFO 则只是“流过的字节数”。

2. **超大单行文件的 truncate 模式**：注释中已指出，若目标区间内存在一条没有换行的超大行，`partial` 在 truncate 模式下仍可能无限增长。当前代码在 fragment 阶段做了预算检查（`fragBytes > maxBytes`）并会丢弃，但 fast path 中不存在类似保护——fast path 本来就把整文件读进内存，所以这是设计上的取舍。

3. **EISDIR 与特殊设备**：`BLOCKED_DEVICE_PATHS` 的拦截实际在 `FileReadTool.ts` 中完成，`readFileInRange.ts` 只做了目录检查。调用方若直接传入 `/dev/urandom` 等路径，streaming path 会尝试打开并可能永远读不到 EOF，直到 `maxBytes` 触发 `FileTooLargeError` 或外部 abort。

4. **mtime 为 0 的降级**：`fstat` 出错时 `resolveMtime(err ? 0 : stats.mtimeMs)`，调用方（如 FileReadTool）依赖 mtime 做 dedup，若返回 0 可能导致 dedup 失效或误命中。

### 改进建议

1. **统一特殊文件黑名单**：将 `FileReadTool.ts` 中的 `BLOCKED_DEVICE_PATHS` 检查下沉到 `readFileInRange.ts`，使所有调用方（包括 `QuickOpenDialog`）都受益，避免意外阻塞。

2. **Fast Path 的 BOM 处理优化**：当前 `raw.charCodeAt(0) === 0xfeff` 对空字符串会返回 `NaN`，逻辑上没问题，但可显式 `raw.length > 0` 提高可读性。

3. **Streaming Path 增加 `bytesRead` 精确性注释**：在 `ReadFileRangeResult` 的 JSDoc 中明确说明 `totalBytes` 在 pipe/FIFO 场景下是“已流过的字节数”而非文件大小，减少下游误解。

4. **考虑 SIMD 或 `readline` 模块**：未来若需要进一步提速，可评估 Node.js `readline.createInterface({ crlfDelay: Infinity })` 或实验性 `fs.readv` + 自定义扫描，但当前实现已足够高效，改动收益比不高。
