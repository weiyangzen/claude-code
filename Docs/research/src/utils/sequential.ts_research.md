# 研究文档：src/utils/sequential.ts

## 场景与职责

`sequential.ts` 提供了一个**异步函数顺序执行包装器**，用于将可能并发调用的异步函数转换为严格串行执行。这在 Claude Code 中非常关键，因为许多操作（如文件写入、配置更新、状态变更）若并发执行会产生竞态条件（race conditions），导致数据损坏或状态不一致。

该模块的设计目标是：
- **保证调用顺序**：并发请求按到达顺序排队，依次执行。
- **保留返回值**：每个调用者最终都能拿到正确的 `resolve` / `reject` 结果。
- **零外部依赖**：纯 TypeScript 实现，不依赖任何第三方队列库。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `sequential(fn)` | 接收一个异步函数，返回一个包装函数。对包装函数的并发调用会被串行化，内部维护一个 FIFO 队列。 |

---

## 具体技术实现

### 1. 类型定义

```ts
type QueueItem<T extends unknown[], R> = {
  args: T
  resolve: (value: R) => void
  reject: (reason?: unknown) => void
  context: unknown
}
```

- `args`：调用时的参数列表。
- `resolve` / `reject`：该次调用的 Promise 控制器。
- `context`：通过 `.apply(context, args)` 保留的 `this` 上下文。

### 2. `sequential` 实现

```ts
export function sequential<T extends unknown[], R>(
  fn: (...args: T) => Promise<R>,
): (...args: T) => Promise<R> {
  const queue: QueueItem<T, R>[] = []
  let processing = false

  async function processQueue(): Promise<void> {
    if (processing) return
    if (queue.length === 0) return

    processing = true

    while (queue.length > 0) {
      const { args, resolve, reject, context } = queue.shift()!

      try {
        const result = await fn.apply(context, args)
        resolve(result)
      } catch (error) {
        reject(error)
      }
    }

    processing = false

    // 处理在 while 循环期间新入队的项目
    if (queue.length > 0) {
      void processQueue()
    }
  }

  return function (this: unknown, ...args: T): Promise<R> {
    return new Promise((resolve, reject) => {
      queue.push({ args, resolve, reject, context: this })
      void processQueue()
    })
  }
}
```

### 3. 关键机制解析

- **单消费者模型**：`processing` 标志确保同一时刻只有一个 `processQueue` 在运行。
- **`while` 而非递归**：在一个事件循环 tick 内尽可能多地处理队列项，减少不必要的微任务调度。
- **尾部检查**：当 `while` 结束后，若队列又有新项（说明在 `await fn.apply(...)` 期间有新调用到达），再次触发 `processQueue()`。
- **无锁/无定时器**：完全基于 Promise 和事件循环，没有 `setTimeout` 或锁机制。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sequential.ts:19-56` | `sequential` 完整实现。 |
| `src/ink/root.ts` | 调用方：Ink 渲染根组件，可能用于串行化终端输出或状态更新。 |
| `src/bridge/replBridge.ts` | 调用方：REPL bridge 的某些操作需要串行化。 |
| `src/bridge/replBridgeTransport.ts` | 调用方：bridge 传输层。 |
| `src/services/MagicDocs/magicDocs.ts` | 调用方：MagicDocs 的缓存或文件写入。 |
| `src/services/SessionMemory/sessionMemory.ts` | 调用方：会话内存的持久化写入。 |
| `src/services/tools/toolExecution.ts` | 调用方：工具执行状态更新。 |
| `src/services/api/sessionIngress.ts` | 调用方：session ingress 的某些网络请求序列化。 |
| `src/services/teamMemorySync/index.ts` | 调用方：团队内存同步写入。 |
| `src/utils/collapseHookSummaries.ts` | 调用方：hook 摘要的折叠更新。 |
| `src/utils/readEditContext.ts` | 调用方：读/编辑上下文更新。 |
| `src/utils/commitAttribution.ts` | 调用方：提交归因的并发写入保护。 |
| `src/utils/sessionStorage.ts` | 调用方：会话存储的并发写入保护。 |
| `src/utils/mcpOutputStorage.ts` | 调用方：MCP 输出存储。 |
| `src/utils/swarm/backends/TmuxBackend.ts` / `ITermBackend.ts` | 调用方：终端后端操作序列化。 |

---

## 依赖与外部交互

- **无第三方依赖**。
- **无 Node.js 内置模块依赖**。
- **调用方**：遍布整个代码库，主要用于文件/状态/存储的并发保护。

---

## 风险、边界与改进建议

### 风险与边界

1. **`fn` 必须是返回 Promise 的函数**：如果传入同步函数，`await fn.apply(...)` 会将其包装为 resolved Promise，行为上仍然正确，但语义上有些 misleading。类型签名 `(...args: T) => Promise<R>` 已经约束了这一点。

2. **队列无限增长**：没有队列长度上限。在极端情况下（如高频并发调用且 `fn` 执行极慢），队列可能无限膨胀导致内存泄漏。当前调用方（如文件写入、状态更新）的并发频率和 `fn` 执行时间都在可控范围内，但在守护进程或长时间运行场景中可能成为隐患。

3. **`processing` 标志的竞态**：
   ```ts
   processing = false
   if (queue.length > 0) {
     void processQueue()
   }
   ```
   在 `processing = false` 和 `if (queue.length > 0)` 之间，若有新项入队并触发 `processQueue()`，此时 `processing` 已为 `false`，会启动第二个 `processQueue`。但由于 `processing = true` 在 `while` 之前设置，第二个实例会在入口处发现 `processing === true` 而立即返回。这是安全的，但会多产生一次无意义的函数调用。

4. **错误隔离**：队列中某一项如果 `reject`，不会影响后续项的执行。这是正确的设计，但调用方需要确保对包装函数的 Promise 做 `.catch()` 处理，否则未捕获的 rejection 可能在 Node.js 15+ 中触发 `unhandledRejection` 警告。

5. **无超时/取消机制**：调用方无法单独取消队列中的某一项，也无法设置全局超时。若 `fn` 挂起（如网络请求无响应），整个队列都会被阻塞。

### 改进建议

1. **增加队列长度上限与背压策略**：当队列超过某个阈值（如 1000）时，新调用直接 reject 或返回一个特殊信号，防止内存无限增长。

2. **支持 AbortSignal 透传**：若 `fn` 支持 `AbortSignal` 参数，可考虑在 `sequential` 中增加信号管理，允许调用者取消 pending 的队列项。但这会显著增加复杂度，需要仔细设计。

3. **优化尾部检查逻辑**：将尾部检查改为在 `finally` 块中无条件触发，可消除上述竞态窗口：
   ```ts
   async function processQueue(): Promise<void> {
     if (processing) return
     processing = true
     try {
       while (queue.length > 0) { /* ... */ }
     } finally {
       processing = false
       if (queue.length > 0) {
         void processQueue()
       }
     }
   }
   ```
   这样更 robust，且能确保异常情况下也能继续处理队列。

4. **增加 `sequential.sync` 变体（可选）**：虽然当前设计针对 async 函数，但可以考虑提供一个同步版本 `sequentialSync`，用于需要串行化同步副作用（如某些全局状态修改）的场景。

5. **单元测试覆盖**：建议增加以下测试：
   - 并发调用确保顺序执行
   - 错误项不影响后续项
   - `this` 上下文正确传递
   - 返回值正确传递
   - 极端并发（如 1000 次同时调用）的性能与内存表现
