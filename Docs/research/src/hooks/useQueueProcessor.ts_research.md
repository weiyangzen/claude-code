# Research: src/hooks/useQueueProcessor.ts

## 场景与职责

`useQueueProcessor` 是一个用于**消费统一命令队列**的 React Hook。在 Claude Code 的 REPL 中，用户输入、任务通知、MCP/Bridge 转发消息、Chrome 扩展 prompt 等多种来源都会产生待处理的命令。这些命令被集中放入一个模块级（module-level）的队列中，而 `useQueueProcessor` 负责在合适的时机将它们批量或逐个取出并执行。

该 Hook 的核心职责是：
1. **监听队列变化**：通过 `useSyncExternalStore` 订阅命令队列，绕过 React Context 的传播延迟。
2. **监听查询状态**：通过 `useSyncExternalStore` 订阅 `QueryGuard`，确保不会在有活跃查询时并发执行新命令。
3. **触发队列处理**：当条件满足（无活跃查询、队列非空、无本地 JSX UI 阻塞）时，调用 `processQueueIfReady` 消费队列。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **队列订阅** | 使用 `useSyncExternalStore(subscribeToCommandQueue, getCommandQueueSnapshot)` 保证队列变更能即时触发重渲染，解决 Ink Context 通知延迟导致的漏处理。 |
| **查询守卫订阅** | 使用 `useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)` 获取同步的查询活跃状态。 |
| **条件门控** | 仅在 `!isQueryActive && !hasActiveLocalJsxUI && queueSnapshot.length > 0` 时触发处理，防止并发执行和 UI 冲突。 |
| **委托实际处理** | 调用 `processQueueIfReady({ executeInput: executeQueuedInput })`，由 `queueProcessor.ts` 负责具体的出队、批处理、slash 命令隔离等逻辑。 |

## 具体技术实现

### Hook 签名

```ts
type UseQueueProcessorParams = {
  executeQueuedInput: (commands: QueuedCommand[]) => Promise<void>
  hasActiveLocalJsxUI: boolean
  queryGuard: QueryGuard
}

export function useQueueProcessor({
  executeQueuedInput,
  hasActiveLocalJsxUI,
  queryGuard,
}: UseQueueProcessorParams): void
```

### 订阅实现

```ts
const isQueryActive = useSyncExternalStore(
  queryGuard.subscribe,
  queryGuard.getSnapshot,
)

const queueSnapshot = useSyncExternalStore(
  subscribeToCommandQueue,
  getCommandQueueSnapshot,
)
```

- `queryGuard` 是一个同步状态机（`idle | dispatching | running`），专门用于解决 "React state 异步批处理 vs 实际查询已启动" 的竞态问题。
- `subscribeToCommandQueue` / `getCommandQueueSnapshot` 来自 `messageQueueManager.ts`，底层使用自定义的 `createSignal()` 实现发布-订阅。

### Effect 触发逻辑

```ts
useEffect(() => {
  if (isQueryActive) return
  if (hasActiveLocalJsxUI) return
  if (queueSnapshot.length === 0) return

  processQueueIfReady({ executeInput: executeQueuedInput })
}, [
  queueSnapshot,
  isQueryActive,
  executeQueuedInput,
  hasActiveLocalJsxUI,
  queryGuard,
])
```

### 关于 Reservation 的注释

源码中有大段注释解释为什么不需要在 Hook 内做 `reserve()`：

> "Reservation is now owned by handlePromptSubmit (inside executeUserInput's try block). The sync chain executeQueuedInput → handlePromptSubmit → executeUserInput → queryGuard.reserve() runs before the first real await, so by the time React re-runs this effect (due to the dequeue-triggered snapshot change), isQueryActive is already true (dispatching) and the guard above returns early."

这意味着队列的并发保护已经下沉到 `executeUserInput` 的同步调用链中，`useQueueProcessor` 本身只负责"在看起来安全的时候尝试处理"，真正的原子性由 `QueryGuard` 保证。

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/useQueueProcessor.ts` | 本 Hook：订阅队列和查询状态，条件触发处理。 |
| `src/screens/REPL.tsx` | 调用方：传入 `executeQueuedInput`（实际为 `onSubmit` 的包装）、`hasActiveLocalJsxUI`、`queryGuard`。 |
| `src/utils/messageQueueManager.ts` | 统一命令队列的实现：提供 `subscribeToCommandQueue`、`getCommandQueueSnapshot`、`enqueue`、`dequeue` 等 API。 |
| `src/utils/queueProcessor.ts` | `processQueueIfReady` 实现：负责具体的出队策略、slash 命令单独处理、同模式批处理。 |
| `src/utils/QueryGuard.ts` | `QueryGuard` 类：同步查询生命周期状态机，兼容 `useSyncExternalStore`。 |
| `src/utils/handlePromptSubmit.ts` | `handlePromptSubmit` 内部调用 `queryGuard.reserve()`，确保提交前原子性地占用查询槽位。 |

## 依赖与外部交互

### 运行时依赖
- **React**：`useEffect`、`useSyncExternalStore`。
- **自定义 Signal**：`messageQueueManager.ts` 中的 `createSignal()` 提供轻量级订阅机制。
- **QueryGuard**：同步状态机，避免 React state 的批处理延迟。

### 与 REPL 的交互
- `REPL.tsx` 创建并持有一个稳定的 `queryGuard` 实例（`React.useRef(new QueryGuard()).current`），将其传给 `useQueueProcessor`。
- `executeQueuedInput` 在 `REPL.tsx` 中通常指向一个包装了 `onSubmit` 的函数，负责将 `QueuedCommand[]` 转换为实际的用户消息并启动查询。

### 与队列处理器的交互
- `processQueueIfReady` 的返回值 `{ processed: boolean }` 在当前 Hook 中被忽略，因为无论是否实际处理了什么，effect 都会在下次依赖变化时再次运行。
- `queueProcessor.ts` 内部会调用 `dequeue()` 或 `dequeueAllMatching()`，这些操作会修改模块级队列并触发 `notifySubscribers()`，从而再次引起 `useSyncExternalStore` 的更新。

## 风险、边界与改进建议

### 风险与边界
1. **`queueSnapshot` 是只读快照**：`getCommandQueueSnapshot` 返回的是 `Object.freeze([...commandQueue])`，虽然数组本身不可变，但数组元素（`QueuedCommand` 对象）仍然是可变的。不过实际代码中不会修改队列元素。
2. **`queryGuard` 实例必须稳定**：如果调用方每次渲染都创建新的 `QueryGuard`，订阅会不断重建，导致状态丢失。`REPL.tsx` 使用 `useRef` 保证了稳定性。
3. **`hasActiveLocalJsxUI` 是外部传入的布尔值**：该值由 `REPL.tsx` 根据 `isLocalJSXCommandActive` 计算，若该状态的更新有延迟，可能导致本地 JSX UI 尚未完全渲染时就开始处理队列，产生 UI 竞争。
4. **Effect 依赖包含 `queryGuard` 对象本身**：虽然 `queryGuard` 是稳定引用，但将其放入依赖数组在 ESLint 规则下通常需要额外说明。当前代码没有 `eslint-disable` 注释，说明 lint 配置允许稳定对象引用作为依赖。
5. **无错误恢复机制**：如果 `processQueueIfReady` 或 `executeQueuedInput` 抛出异常，该异常不会被本 Hook 捕获，可能向上传播并导致 React 渲染崩溃。

### 改进建议
1. **增加错误边界保护**：在 `processQueueIfReady` 调用处增加 `try/catch`，对执行异常进行日志记录，并考虑将失败的命令重新入队或标记为失败，避免整个队列处理中断。
2. **细化 `hasActiveLocalJsxUI` 的判定**：当前仅基于一个布尔值，未来若支持多个并发的本地 JSX UI，可改为传入一个优先级/阻塞原因枚举，让队列处理器根据命令优先级决定是否绕过阻塞。
3. **暴露处理状态**：当前 Hook 返回 `void`，调用方无法知道是否正在处理队列。可考虑返回 `isProcessing` 状态，供 UI 显示加载指示或调试信息。
4. **队列积压告警**：若队列长度持续增长（如 `executeQueuedInput` 持续失败），当前没有任何告警机制。可在 `useEffect` 中增加对 `queueSnapshot.length` 的监控，超过阈值时发送调试日志或通知。
