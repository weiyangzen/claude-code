# 研究文档：src/utils/sdkEventQueue.ts

## 场景与职责

`sdkEventQueue.ts` 是 Claude Code **headless / streaming 模式** 下的 SDK 事件总线。当 Claude Code 作为子进程被外部 SDK（如 VS Code 扩展、Scuttle、其他 orchestrator）调用时，外部消费者需要实时了解内部任务状态：

- 任务何时开始（`task_started`）
- 任务进度如何（`task_progress`）
- 任务何时结束（`task_notification`）
- 会话主生成器是否空闲（`session_state_changed`）

该模块提供一个**进程内内存队列**，生产者（各种内部模块）将事件入队，消费者（`print.ts` 中的 headless 输出循环）在合适的时机批量 drain 并序列化为 NDJSON / stream-json 输出。

**关键约束**：TUI（交互式）模式下事件不入队，因为不会被读取，避免内存泄漏。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `enqueueSdkEvent(event)` | 将 SDK 事件推入内存队列。TUI 模式下直接返回；队列长度超过 `MAX_QUEUE_SIZE = 1000` 时丢弃最旧事件。 |
| `drainSdkEvents()` | 一次性取出队列中所有事件，为每个事件注入 `uuid` 与 `session_id`，返回给 headless 输出层。 |
| `emitTaskTerminatedSdk(taskId, status, opts?)` | 专门用于任务到达终态（completed / failed / stopped）时发射 `task_notification` 事件。 |

---

## 具体技术实现

### 1. 事件类型定义

```ts
type TaskStartedEvent = {
  type: 'system'
  subtype: 'task_started'
  task_id: string
  tool_use_id?: string
  description: string
  task_type?: string
  workflow_name?: string
  prompt?: string
}

type TaskProgressEvent = {
  type: 'system'
  subtype: 'task_progress'
  task_id: string
  tool_use_id?: string
  description: string
  usage: { total_tokens: number; tool_uses: number; duration_ms: number }
  last_tool_name?: string
  summary?: string
  workflow_progress?: SdkWorkflowProgress[]
}

type TaskNotificationSdkEvent = {
  type: 'system'
  subtype: 'task_notification'
  task_id: string
  tool_use_id?: string
  status: 'completed' | 'failed' | 'stopped'
  output_file: string
  summary: string
  usage?: { total_tokens: number; tool_uses: number; duration_ms: number }
}

type SessionStateChangedEvent = {
  type: 'system'
  subtype: 'session_state_changed'
  state: 'idle' | 'running' | 'requires_action'
}
```

导出联合类型 `SdkEvent = TaskStartedEvent | TaskProgressEvent | TaskNotificationSdkEvent | SessionStateChangedEvent`。

### 2. 队列实现

```ts
const MAX_QUEUE_SIZE = 1000
const queue: SdkEvent[] = []
```

- `enqueueSdkEvent`：
  ```ts
  if (!getIsNonInteractiveSession()) return
  if (queue.length >= MAX_QUEUE_SIZE) queue.shift()
  queue.push(event)
  ```
- `drainSdkEvents`：
  ```ts
  if (queue.length === 0) return []
  const events = queue.splice(0)
  return events.map(e => ({
    ...e,
    uuid: randomUUID(),
    session_id: getSessionId(),
  }))
  ```

### 3. `emitTaskTerminatedSdk`

设计为 `task_started` 的闭合书end。注释中强调：

> "Call this from any exit path that sets a task terminal WITHOUT going through enqueuePendingNotification-with-<task-id> (print.ts parses that XML into the same SDK event, so paths that do both would double-emit)."

即：如果 `print.ts` 的 XML 通知解析路径已经会生成同样的 SDK 事件，则不应再调用此函数，否则消费者会看到重复通知。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sdkEventQueue.ts:6-73` | SDK 事件类型定义与队列实现。 |
| `src/utils/sdkEventQueue.ts:77-101` | `enqueueSdkEvent` 与 `drainSdkEvents`。 |
| `src/utils/sdkEventQueue.ts:114-134` | `emitTaskTerminatedSdk` 及注释。 |
| `src/cli/print.ts` | 主要消费者：headless 输出循环中调用 `drainSdkEvents()` 并写入 stream-json。 |
| `src/tasks/LocalMainSessionTask.ts` | 生产者：本地主会话任务状态变化时入队。 |
| `src/tasks/stopTask.ts` | 生产者：任务停止时入队。 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 生产者：远程 agent 任务事件。 |
| `src/utils/task/framework.ts` | 生产者：任务框架状态变更。 |
| `src/utils/task/sdkProgress.ts` | 生产者：SDK 进度上报。 |
| `src/utils/sessionState.ts` | 生产者：`session_state_changed` 事件。 |
| `src/utils/swarm/spawnInProcess.ts` / `inProcessRunner.ts` | 生产者：子进程任务事件。 |
| `src/tools/AgentTool/AgentTool.tsx` | 生产者：Agent 工具生命周期。 |
| `src/hooks/useCancelRequest.ts` | 可能涉及任务取消事件。 |

---

## 依赖与外部交互

- **Node.js 内置模块**：`crypto`（`randomUUID`）。
- **内部依赖**：
  - `../bootstrap/state.js`：`getIsNonInteractiveSession`, `getSessionId`
  - `../types/tools.js`：`SdkWorkflowProgress`
- **无第三方依赖**。
- **调用方/消费方**：
  - 消费方：`src/cli/print.ts`（`drainSdkEvents`）
  - 生产方：任务系统、Agent 工具、session 状态管理、swarm 子进程等 10+ 处

---

## 风险、边界与改进建议

### 风险与边界

1. **TUI 模式完全丢弃事件**：`enqueueSdkEvent` 在交互模式下直接 `return`。这意味着如果未来有需求让 TUI 也支持某种“本地 SDK 消费者”（如 VS Code 通过本地 socket 监听），需要重构此处的模式判断。

2. **`MAX_QUEUE_SIZE = 1000` 的硬编码**：对于极长会话或高频进度更新，1000 条可能不够，导致旧事件被丢弃。虽然当前 `task_progress` 的更新频率受控，但若并发大量子 agent，队列可能快速填满。

3. **`drainSdkEvents` 的 UUID 与 session_id 注入时机**：UUID 是在 drain 时生成的，而不是入队时。这意味着如果同一条事件被 drain 两次（理论上不会，因为 `splice(0)` 会清空队列），UUID 会不同。当前设计下这是安全的，但将 UUID 生成推迟到 drain 意味着事件在队列中期间没有唯一标识，不利于调试或中间状态 dump。

4. **`emitTaskTerminatedSdk` 的双发风险**：注释已充分说明，但代码层面没有任何机制防止 double-emit。开发者必须手动确保调用路径不与 `print.ts` 的 XML 解析路径重叠，这对新加入的开发者是认知负担。

5. **无持久化**：队列是纯内存数组，进程崩溃或意外退出时所有未 drain 的事件丢失。对于 headless 模式，这通常可接受（消费者应通过最终 result 推断状态），但在调试任务中断原因时可能信息不足。

### 改进建议

1. **按事件类型分队列或加权丢弃**：当前 FIFO 丢弃可能丢掉重要的 `task_notification` 而保留大量 `task_progress`。可引入优先级队列，或在满时优先丢弃 `task_progress` 而保留 `task_notification`。

2. **入队时即生成 UUID**：将 `randomUUID()` 移到 `enqueueSdkEvent` 中，使每条事件自诞生起就有稳定标识，便于日志追踪、重复检测和中间 dump。

3. **增加队列水位 telemetry**：在丢弃事件或队列接近上限时记录 `logEvent`，帮助评估 1000 是否足够，以及哪些会话产生了事件风暴。

4. **抽象事件总线接口**：当前模块是全局单例。若未来需要支持多会话并行或单元测试隔离，可将队列封装成类实例，通过依赖注入传递，而非直接操作模块级 `queue`。

5. **`session_state_changed` 的 `idle` 语义保证**：注释说明 `idle` 是在 `heldBackResult` flush 且 bg-agent 循环退出后才触发。建议在 `sessionState.ts` 中增加断言或单元测试，确保没有竞态导致 `idle` 提前发射。
