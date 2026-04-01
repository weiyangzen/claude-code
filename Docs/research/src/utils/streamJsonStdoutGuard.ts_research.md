# streamJsonStdoutGuard.ts 研究文档

## 场景与职责

`streamJsonStdoutGuard.ts` 提供了一个运行时保护机制，用于 `--output-format=stream-json` 模式下的 stdout 输出。该保护器拦截所有写入 stdout 的数据，确保只有有效的 JSON 行通过，将非 JSON 内容（如调试日志、依赖库的 console.log）重定向到 stderr。

## 功能点目的

### NDJSON 流保护
- **问题**: SDK 客户端以 NDJSON (Newline Delimited JSON) 格式解析 stdout，任何非 JSON 输出都会破坏解析
- **解决方案**: 包装 `process.stdout.write`，缓冲内容直到遇到换行符，验证每行是否为有效 JSON
- **效果**: 非 JSON 行被标记并发送到 stderr，保持 stdout 流干净

### 调试可见性
- 被拦截的非 JSON 行带有 `[stdout-guard]` 标记，便于日志抓取和测试
- 通过 `logForDebugging` 记录被拦截的内容

## 具体技术实现

### 核心算法

```typescript
function isJsonLine(line: string): boolean {
  if (line.length === 0) return true  // 空行允许
  try {
    JSON.parse(line)
    return true
  } catch {
    return false
  }
}
```

### 安装机制

```typescript
export function installStreamJsonStdoutGuard(): void {
  if (installed) return  // 防止重复安装
  installed = true

  originalWrite = process.stdout.write.bind(process.stdout)
  
  process.stdout.write = function (
    chunk: string | Uint8Array,
    encodingOrCb?: BufferEncoding | ((err?: Error) => void),
    cb?: (err?: Error) => void
  ): boolean {
    // 缓冲 + 行验证逻辑
  }
}
```

### 缓冲与处理流程

1. **数据缓冲**: 所有写入的数据追加到 `buffer` 字符串
2. **行分割**: 按 `\n` 分割，逐行处理
3. **JSON 验证**: 每行通过 `isJsonLine` 验证
4. **分流输出**:
   - JSON 行 → 原始 `stdout.write`
   - 非 JSON 行 → `stderr.write` (带 `[stdout-guard]` 标记)
5. **回调处理**: 通过 `queueMicrotask` 异步触发回调

### 清理机制

```typescript
registerCleanup(async () => {
  // 刷新缓冲区中的剩余内容
  if (buffer.length > 0) {
    if (originalWrite && isJsonLine(buffer)) {
      originalWrite(buffer + '\n')
    } else {
      process.stderr.write(`${STDOUT_GUARD_MARKER} ${buffer}\n`)
    }
  }
  // 恢复原始 write 函数
  if (originalWrite) {
    process.stdout.write = originalWrite
    originalWrite = null
  }
  installed = false
})
```

## 关键代码路径与文件引用

### 本文件导出
- `installStreamJsonStdoutGuard()`: 安装保护器
- `STDOUT_GUARD_MARKER`: 标记常量 `[stdout-guard]`
- `_resetStreamJsonStdoutGuardForTesting()`: 测试重置函数

### 依赖模块

| 模块 | 用途 |
|------|------|
| `./cleanupRegistry.js` | `registerCleanup` 注册清理函数 |
| `./debug.js` | `logForDebugging` 调试日志 |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/cli/print.ts` | 在 `runHeadless` 中，当 `options.outputFormat === 'stream-json'` 时安装 |

### 调用代码片段

```typescript
// src/cli/print.ts
if (options.outputFormat === 'stream-json') {
  installStreamJsonStdoutGuard()
}
```

## 依赖与外部交互

### 与 asciicast 的关系
注释中提到该保护器与 `asciicast.ts` 在同一层包装 `process.stdout.write`，两者需要协调避免冲突。

### 与 structuredIO 的关系
注释说明 "blessed JSON path" 是：
```
structuredIO.write → writeToStdout → stdout.write
```
保护器允许这个路径通过，只拦截 "out-of-band" 写入。

### 清理注册
通过 `cleanupRegistry.js` 注册清理函数，确保：
- 进程退出时刷新缓冲区
- 恢复原始 `stdout.write`
- 重置 `installed` 标志

## 风险、边界与改进建议

### 潜在风险

1. **性能影响**: 每行都进行 `JSON.parse`，对于高频小写入可能有性能开销
2. **内存使用**: 缓冲区可能积累大量数据直到遇到换行符
3. **编码问题**: 当前只处理 UTF-8，其他编码可能有问题
4. **与其他包装器冲突**: 如果其他代码也包装 `stdout.write`，可能导致意外行为

### 边界情况

1. **无换行符的长行**: 如果写入大量数据不含换行符，缓冲区会持续增长
2. **JSON 片段**: 不完整的 JSON 行会被视为非 JSON 并发送到 stderr
3. **二进制数据**: `Uint8Array` 输入被转换为 UTF-8 字符串，可能丢失信息
4. **重复安装**: 通过 `installed` 标志防止，但测试需要 `_resetStreamJsonStdoutGuardForTesting`

### 改进建议

1. **缓冲区大小限制**: 添加最大缓冲区大小，防止内存无限增长
```typescript
const MAX_BUFFER_SIZE = 1024 * 1024 // 1MB
if (buffer.length > MAX_BUFFER_SIZE) {
  // 强制刷新或报错
}
```

2. **流式 JSON 解析**: 使用流式 JSON 解析器处理不完整行
```typescript
// 使用 JSON 流式解析库如 @streamparser/json
```

3. **性能优化**: 对于已知安全的写入（如来自 structuredIO），提供快速路径
```typescript
process.stdout.write = function (chunk, encoding, cb) {
  if (isFromBlessedPath()) {
    return originalWrite!(chunk, encoding, cb)
  }
  // ... 正常验证逻辑
}
```

4. **更详细的日志**: 记录被拦截内容的来源（堆栈跟踪）
```typescript
logForDebugging(
  `streamJsonStdoutGuard diverted non-JSON stdout line: ${line.slice(0, 200)}`,
  { stack: new Error().stack }
)
```

5. **配置选项**: 允许配置标记文本和目标（stderr 或文件）
```typescript
interface GuardOptions {
  marker?: string
  divertTo?: 'stderr' | 'file' | 'discard'
  logFile?: string
}
```

6. **统计信息**: 提供拦截统计用于监控
```typescript
export function getGuardStats(): {
  totalLines: number
  jsonLines: number
  divertedLines: number
}
```
