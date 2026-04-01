# src/utils/log.ts 研究文档

## 场景与职责

`log.ts` 是 Claude Code CLI 的核心日志和错误处理基础设施模块。它提供了多层级的错误记录能力，包括内存中的错误缓存、持久化错误日志文件、MCP 错误/调试日志，以及 API 请求捕获功能。该模块的设计兼顾了性能、隐私和可调试性。

主要职责：
1. **错误记录 (`logError`)**：将错误记录到内存缓存和持久化日志文件（仅内部 ant 用户）。
2. **MCP 日志 (`logMCPError` / `logMCPDebug`)**：记录 MCP 服务器的错误和调试信息。
3. **错误日志加载 (`loadErrorLogs` / `getErrorLogByIndex`)**：加载和检索按日期排序的错误日志。
4. **API 请求捕获 (`captureAPIRequest`)**：捕获最后一次 API 请求的参数（不含消息），用于 bug 报告。
5. **错误日志 Sink 机制 (`attachErrorLogSink`)**：解耦错误记录与文件 I/O，允许在应用启动后再挂载真正的日志后端。

调用方极其广泛，几乎覆盖整个代码库。

## 功能点目的

### 1. `logError`
错误记录的主入口：
- **Hard Fail 模式**：若进程参数包含 `--hard-fail` 且构建宏 `feature('HARD_FAIL')` 启用，`logError` 会直接 `console.error` 并调用 `process.exit(1)`。这是用于 CI/测试的严格模式。
- **隐私保护**：在 Bedrock/Vertex/Foundry 环境、设置了 `DISABLE_ERROR_REPORTING`、或启用了 `isEssentialTrafficOnly()` 时，直接静默返回，不记录任何内容。
- **内存缓存**：将错误字符串和时间戳加入 `inMemoryErrorLog`（最多保留 100 条，FIFO 淘汰）。
- **Sink 分发**：若 `errorLogSink` 已挂载，直接调用 `sink.logError()`；否则将事件加入 `errorQueue`，待 Sink 挂载后批量处理。

### 2. `attachErrorLogSink`
在应用启动时挂载错误日志后端（如 `src/utils/errorLogSink.ts` 中实现的文件写入器）。挂载后会立即 drain `errorQueue` 中积压的事件，确保启动早期的错误不会丢失。

### 3. `ErrorLogSink` 接口
```typescript
export type ErrorLogSink = {
  logError: (error: Error) => void
  logMCPError: (serverName: string, error: unknown) => void
  logMCPDebug: (serverName: string, message: string) => void
  getErrorsPath: () => string
  getMCPLogsPath: (serverName: string) => string
}
```

### 4. `loadErrorLogs` / `getErrorLogByIndex`
从 `CACHE_PATHS.errors()` 目录加载错误日志文件列表，按文件名/日期排序，并解析为 `LogOption` 对象。每个错误日志文件包含一个错误对象的 JSON 数组。

### 5. `captureAPIRequest`
捕获最后一次 API 请求参数（排除 `messages` 数组，避免保留整个对话历史）。对于内部 `ant` 用户，还会额外保留完整的 `messages` 数组引用，供 `/share` 的 `serialized_conversation.json` 使用。

### 6. `getLogDisplayTitle`
为日志/会话生成显示标题，支持多层 fallback：
```
agentName > customTitle > summary > strippedFirstPrompt > defaultTitle > 'Autonomous session' > sessionId.slice(0,8) > ''
```
同时会去除 display-unfriendly 的 XML 标签（如 `<ide_opened_file>`）。

## 具体技术实现

### Sink 与队列模式
```typescript
const errorQueue: QueuedErrorEvent[] = []
let errorLogSink: ErrorLogSink | null = null

export function attachErrorLogSink(newSink: ErrorLogSink): void {
  if (errorLogSink !== null) return
  errorLogSink = newSink
  if (errorQueue.length > 0) {
    const queuedEvents = [...errorQueue]
    errorQueue.length = 0
    for (const event of queuedEvents) {
      // 分发到 sink
    }
  }
}
```

这种设计允许在 CLI 启动早期（如模块加载阶段、配置解析阶段）就开始调用 `logError()`，而不必等待文件系统初始化完成。

### 内存错误缓存
```typescript
const MAX_IN_MEMORY_ERRORS = 100
let inMemoryErrorLog: Array<{ error: string; timestamp: string }> = []

function addToInMemoryErrorLog(errorInfo): void {
  if (inMemoryErrorLog.length >= MAX_IN_MEMORY_ERRORS) {
    inMemoryErrorLog.shift()
  }
  inMemoryErrorLog.push(errorInfo)
}
```

### 错误日志文件解析
`loadLogList` 读取错误日志目录中的所有文件，对每个文件：
1. `readFile(fullPath, 'utf8')`
2. `jsonParse(content) as SerializedMessage[]`
3. 提取 `firstPrompt`、`lastMessage`、文件 `stat` 信息
4. 标记 `isSidechain`（通过文件名判断是否包含 `sidechain`）
5. 按日期排序

### API 请求捕获的隐私设计
```typescript
const { messages, ...paramsWithoutMessages } = params
setLastAPIRequest(paramsWithoutMessages)
setLastAPIRequestMessages(process.env.USER_TYPE === 'ant' ? messages : null)
```

普通用户不保留 `messages`，避免在内存中长期持有敏感对话内容。ant 用户保留是因为他们需要 `/share` 功能来导出完整对话。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/log.ts:30-58` | `getLogDisplayTitle` 日志标题生成 |
| `src/utils/log.ts:66-77` | `addToInMemoryErrorLog` 内存错误缓存 |
| `src/utils/log.ts:109-134` | `attachErrorLogSink` Sink 挂载与队列排空 |
| `src/utils/log.ts:154-199` | `logError` 核心错误记录逻辑 |
| `src/utils/log.ts:209-223` | `loadErrorLogs` / `getErrorLogByIndex` 错误日志加载 |
| `src/utils/log.ts:231-283` | `loadLogList` 内部日志列表加载与解析 |
| `src/utils/log.ts:300-326` | `logMCPError` / `logMCPDebug` MCP 日志 |
| `src/utils/log.ts:331-352` | `captureAPIRequest` API 请求捕获 |
| `src/utils/log.ts:358-362` | `_resetErrorLogForTesting` 测试重置 |
| `src/utils/cachePaths.ts` | `CACHE_PATHS.errors()` 错误日志目录 |
| `src/utils/displayTags.ts` | `stripDisplayTags`, `stripDisplayTagsAllowEmpty` |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly` |
| `src/bootstrap/state.ts` | `setLastAPIRequest`, `setLastAPIRequestMessages` |
| `src/types/logs.ts` | `LogOption`, `SerializedMessage`, `sortLogs` |
| `src/utils/errorLogSink.ts` | `ErrorLogSink` 的实际实现 |

## 依赖与外部交互

### 内部依赖
- `../bootstrap/state.js`：`setLastAPIRequest`, `setLastAPIRequestMessages`
- `../constants/xml.js`：`TICK_TAG`
- `../types/logs.js`：`LogOption`, `SerializedMessage`, `sortLogs`
- `./cachePaths.js`：`CACHE_PATHS`
- `./displayTags.js`：`stripDisplayTags`, `stripDisplayTagsAllowEmpty`
- `./envUtils.js`：`isEnvTruthy`
- `./errors.js`：`toError`
- `./privacyLevel.js`：`isEssentialTrafficOnly`
- `./slowOperations.js`：`jsonParse`
- `fs/promises`：`readdir`, `readFile`, `stat`
- `path`：`join`
- `lodash-es/memoize.js`：`memoize`
- `bun:bundle`：`feature`

### 调用方
- 极其广泛，几乎覆盖整个 `src/` 目录。主要调用方包括：
  - `src/utils/errorLogSink.ts`
  - `src/utils/config.ts`
  - `src/utils/ide.ts`
  - `src/utils/sessionStorage.ts`
  - `src/utils/auth.ts`
  - `src/utils/attachments.ts`
  - `src/utils/imageResizer.ts`
  - 几乎所有组件、hooks、服务和命令模块

## 风险、边界与改进建议

### 风险与边界
1. **`logError` 的静默失败**：`logError` 内部被巨大的 `try/catch` 包裹，任何错误（包括 Sink 写入失败）都会被吞掉。这确保了错误记录不会导致级联崩溃，但也意味着如果 Sink 配置错误，开发者可能永远不知道日志没有写入。
2. **Hard Fail 模式的杀伤力**：`feature('HARD_FAIL') && isHardFailMode()` 会直接 `process.exit(1)`。在测试或 CI 中，如果某个非致命错误触发了 `logError`，可能导致整个进程意外终止。
3. **`loadLogList` 的 JSON 解析风险**：错误日志文件可能包含损坏的 JSON（如进程崩溃时半写）。`jsonParse` 在解析失败时可能返回 `null` 或抛出，但 `loadLogList` 中没有显式处理 `null` 情况，后续 `messages[0]` 访问可能导致运行时错误。
4. **内存缓存的 FIFO 淘汰**：`inMemoryErrorLog` 使用 `shift()` 进行 FIFO 淘汰，这在数组较大时（100 条时其实很小）是 O(n) 操作。虽然 100 条的上限使得这可以忽略，但如果未来扩大上限，应考虑使用环形缓冲区或 `Map`。
5. **`captureAPIRequest` 的 querySource 过滤**：只捕获 `querySource.startsWith('repl_main_thread')` 的请求。如果未来有其他主线程查询源（如不同的输出风格变体），可能需要更新过滤逻辑。

### 改进建议
1. **Sink 失败的降级告警**：在 `logError` 的 catch 块中，如果检测到是 Sink 写入失败，可以尝试写入 `console.error`（至少确保在开发环境可见），而不是完全静默。
2. **错误日志文件校验**：在 `loadLogList` 中增加对 `jsonParse` 返回值的校验，若解析失败或返回非数组，记录 debug 日志并跳过该文件。
3. **结构化日志输出**：当前错误日志文件是简单的 JSON 数组。可以考虑迁移到 JSONL 格式，与主会话存储保持一致，并支持追加写入而无需重写整个文件。
4. **错误采样/去重**：对于高频重复错误（如网络超时），可以在 Sink 层实现去重或采样，避免错误日志文件迅速膨胀。
5. **增强 `getLogDisplayTitle` 的可测试性**：该函数有复杂的 fallback 链和标签剥离逻辑，建议增加单元测试覆盖各种边界情况（如纯 XML 提示、autonomous 模式、缺失所有元数据）。
