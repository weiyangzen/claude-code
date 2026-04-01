# src/utils/bufferedWriter.ts 深入研究

## 场景与职责

`bufferedWriter.ts` 实现了一个**同步写入、异步刷盘**的字符缓冲写入器。它的核心目标是：
- 将高频、小粒度的写操作（如逐字符的日志、渲染输出）聚合成批量写入，减少 I/O 次数。
- 在缓冲区溢出或达到字节上限时，通过 `setImmediate` 将实际写操作推迟到下一个事件循环，避免阻塞当前执行路径（如用户输入处理或渲染线程）。

该模块被用于 `errorLogSink.ts`、`asciicast.ts` 和 `debug.ts` 等需要频繁追加文本的场景。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `createBufferedWriter(options)` | 工厂函数，创建带缓冲的写入器实例 |
| `write(content)` | 将内容加入缓冲区；若触发上限则通过 `setImmediate` 异步刷盘 |
| `flush()` | 立即同步刷空缓冲区与待处理溢出队列 |
| `dispose()` | 生命周期结束时强制 `flush()`，确保数据不丢失 |

## 具体技术实现

### 数据结构
```ts
let buffer: string[] = []        // 当前累积的字符串片段
let bufferBytes = 0              // 当前累积字节数（content.length）
let flushTimer: NodeJS.Timeout | null = null  // setTimeout 定时器
let pendingOverflow: string[] | null = null   // 已 detach 但未写入的溢出批次
```

### 写入流程
1. `immediateMode === true`：直接调用 `writeFn(content)`，无缓冲。
2. 正常模式：
   - `buffer.push(content)`，累加 `bufferBytes`
   - 启动 `setTimeout(flush, flushIntervalMs)`（若尚未启动）
   - 若 `buffer.length >= maxBufferSize` 或 `bufferBytes >= maxBufferBytes`，调用 `flushDeferred()`

### 溢出异步刷盘（`flushDeferred`）
- 将当前 `buffer` 整体赋值给 `pendingOverflow`，清空 `buffer` 与 `bufferBytes`。
- 通过 `setImmediate(() => writeFn(pendingOverflow.join('')))` 在下一个 tick 执行实际写入。
- 若此时又有新的溢出，则合并到现有的 `pendingOverflow` 中，保证**写入顺序**。

### 定时刷盘（`flush`）
- 先处理 `pendingOverflow`（若存在）。
- 再处理 `buffer`，调用 `writeFn(buffer.join(''))`。
- 清空计数器并取消定时器。

### 配置项默认值
| 选项 | 默认值 | 说明 |
|------|--------|------|
| `flushIntervalMs` | `1000` | 定时刷盘间隔 |
| `maxBufferSize` | `100` | 缓冲区最大条目数 |
| `maxBufferBytes` | `Infinity` | 缓冲区最大字节数 |
| `immediateMode` | `false` | 是否关闭缓冲直接写入 |

## 关键代码路径与文件引用

```
src/utils/errorLogSink.ts
  └── createBufferedWriter({ writeFn: appendToErrorLog, flushIntervalMs: 1000, maxBufferSize: 50 })
      [错误日志缓冲写入，减少同步 appendFileSync 对主线程的阻塞]

src/utils/asciicast.ts
  └── createBufferedWriter({ writeFn, maxBufferSize: 100 })
      [asciicast 录制输出缓冲]

src/utils/debug.ts
  └── createBufferedWriter({ writeFn: appendToDebugLog, flushIntervalMs: 1000, maxBufferSize: 50 })
      [调试日志缓冲]
```

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| 调用方提供的 `writeFn` | 直接回调 | 通常是同步 I/O（如 `appendFileSync`）或网络发送 |
| Node.js 事件循环 | `setTimeout` / `setImmediate` | 延迟刷盘，避免阻塞用户代码路径 |

## 风险、边界与改进建议

### 风险
1. **`bufferBytes` 使用 `content.length` 而非 `Buffer.byteLength`**：对于多字节字符（如中文、emoji），`content.length` 小于实际 UTF-8 字节数，导致 `maxBufferBytes` 在中文场景下不能准确限制内存占用。
2. **`pendingOverflow` 在进程崩溃时丢失**：`setImmediate` 尚未执行前若进程异常退出，该批次数据会丢失。
3. **无背压机制**：若 `writeFn` 极慢（如网络日志服务阻塞），`pendingOverflow` 可能持续堆积，但当前实现未限制队列深度。
4. **`dispose()` 仅调用 `flush()`**：若 `pendingOverflow` 已通过 `setImmediate` 调度但未执行，`flush()` 会同步写入它；但如果 `flush()` 本身抛异常，`dispose` 没有 catch。

### 边界
- `write(content)` 是同步函数，但内部可能触发 `setImmediate`；调用方不应假设 `write` 返回时内容已落盘。
- `immediateMode` 为 `true` 时完全绕过缓冲，适用于需要强一致性的场景。
- `maxBufferBytes` 默认 `Infinity`，意味着默认行为仅受 `maxBufferSize` 约束。

### 改进建议
1. **准确的字节计数**：将 `bufferBytes += content.length` 改为 `bufferBytes += Buffer.byteLength(content, 'utf8')`，使 `maxBufferBytes` 在多语言环境下更准确。
2. **进程退出保护**：在 `errorLogSink.ts` 等关键路径中，注册 `process.on('beforeExit')` 或 `process.on('exit')` 调用 `writer.dispose()`，降低崩溃丢日志概率。
3. **背压/队列上限**：为 `pendingOverflow` 增加最大合并批次限制，超限时强制同步 `writeFn` 或丢弃并报警。
4. **返回写入状态**：`write()` 可返回当前缓冲深度，便于调用方在极端场景下做流控决策。
5. **支持异步 writeFn**：当前 `WriteFn` 签名是同步的 `(content: string) => void`；若未来需要异步落盘（如 HTTP 批量上报），可考虑提供 `writeFnAsync` 变体。
