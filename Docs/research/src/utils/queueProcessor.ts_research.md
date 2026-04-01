# src/utils/queueProcessor.ts 研究文档

## 场景与职责

`queueProcessor.ts` 是 REPL 主线程的命令队列消费调度器。它决定“什么时候从统一命令队列里取出一个或一批命令，并交给 `executeInput` 执行”。该模块与 `messageQueueManager.ts`（队列存储）和 `useQueueProcessor.ts`（React Hook 触发器）共同构成完整的命令队列流水线。

## 功能点目的

1. **单条 vs 批量处理策略**
   - **Slash 命令**（以 `/` 开头）和 **bash 模式命令**必须单条处理，以保证每条命令都能独立走 slash 命令路由、错误隔离、退出码和进度 UI。
   - **其他非 slash 命令**可以批量 drain：一次性取出所有与队首命令 `mode` 相同的命令，合并为一个数组交给 `executeInput`。不同 mode（如 `prompt` vs `task-notification`）不会被混批。

2. **主线程过滤**
   - 通过 `cmd.agentId === undefined` 过滤掉属于子 Agent 的命令，防止子 Agent 通知卡住主线程队列。

3. **队列状态查询（`hasQueuedCommands`）**
   - 暴露 `hasCommandsInQueue()` 的别名，供调用方判断是否需要触发后续处理。

## 具体技术实现

- `processQueueIfReady` 是同步函数，执行后立即返回 `{ processed: boolean }`。
- 使用 `peek(isMainThread)` 查看队首，但不移除；只有在决定处理时才通过 `dequeue` 或 `dequeueAllMatching` 移除。
- `executeInput(commands)` 被 `void` 包裹，表示不等待其完成；调用方（`useQueueProcessor` 的 effect）会在命令执行完成后通过队列变更信号再次触发。
- `isSlashCommand` 同时支持 `string` 值和 `ContentBlockParam[]` 值：对数组类型，检查第一个 `text` block 的内容。

## 关键代码路径与文件引用

- **本文件**：`src/utils/queueProcessor.ts`
- **调用方**：
  - `src/hooks/useQueueProcessor.ts` — React effect 在 `!isQueryActive && !hasActiveLocalJsxUI && queue.length > 0` 时调用 `processQueueIfReady`。
- **依赖**：
  - `src/utils/messageQueueManager.js` — `dequeue`、`dequeueAllMatching`、`hasCommandsInQueue`、`peek`。
  - `src/types/textInputTypes.js` — `QueuedCommand` 类型。

## 依赖与外部交互

- 无网络 I/O。
- 纯内存队列操作，依赖 `messageQueueManager` 的模块级状态。

## 风险、边界与改进建议

1. **子 Agent 命令的永久跳过**：注释明确提到，若 `peek` 返回子 Agent 命令而 `dequeueAllMatching` 找不到 `agentId===undefined` 的匹配项，会返回 `processed: false` 且队列不变。如果队列前端长期被子 Agent 命令占据，而子 Agent 侧没有自己的消费逻辑，这些命令可能永远挂起。当前设计假设子 Agent 命令由其他路径消费，但这里没有任何断言或日志。
2. **`void executeInput` 的异常吞没**：若 `executeInput` 同步抛错，由于被 `void` 且未 `try/catch`，异常会变成未处理的 Promise rejection（如果 `executeInput` 返回 Promise）或全局未捕获异常（如果同步抛错）。实际上 `executeInput` 内部通常有 `try/finally`，但边界上仍需注意。
3. **批量处理的顺序**：`dequeueAllMatching` 保持原有优先级顺序，但队首决定 targetMode，可能导致高优先级但不同 mode 的命令被低优先级同 mode 命令批量“插队”。例如队列是 `[mode=A priority=now, mode=B priority=later]`，如果 `A` 是 slash 命令被单条处理后，下一次队首变成 `B`，`B` 会正常处理；但如果 `A` 不是 slash 且被批量 drain，则只 drain `A` 的 mode。此行为符合设计，但调用方需理解 mode 优先于优先级。
4. **改进建议**：
   - 增加 debug 日志，记录每次 `processQueueIfReady` 的决策（队首 mode、处理数量、是否因主线程过滤而跳过）。
   - 考虑在 `isMainThread` 过滤后队首为空但队列非空时，触发一个 warning 或 metric，帮助发现子 Agent 命令泄漏。
