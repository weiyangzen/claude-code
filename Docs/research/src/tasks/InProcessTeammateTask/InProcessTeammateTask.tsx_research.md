# InProcessTeammateTask.tsx 研究文档

> 文件路径：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`  
> 研究时间：2026-04-01  
> 关联文件：`src/tasks/InProcessTeammateTask/types.ts`、`src/utils/swarm/spawnInProcess.ts`、`src/utils/swarm/inProcessRunner.ts`、`src/utils/swarm/backends/InProcessBackend.ts`

---

## 1. 场景与职责

`InProcessTeammateTask.tsx` 是 **In-Process Teammate（进程内队友）** 的任务生命周期管理入口。它实现了统一的 `Task` 接口（定义于 `src/Task.ts`），负责：

- **注册与销毁**：通过 `Task.kill` 接口提供队友的强制终止能力。
- **状态变更**：支持请求优雅关闭（`shutdownRequested`）、追加对话消息、注入用户消息。
- **查询与枚举**：按 `agentId` 查找队友任务、获取全部/正在运行的队友列表（按 `agentName` 字母排序）。

与 `LocalAgentTask`（后台子代理）不同，In-Process Teammate：
1. 运行在同一个 Node.js 进程中，利用 `AsyncLocalStorage` 做上下文隔离；
2. 具备团队感知身份（`agentName@teamName`）；
3. 支持 Plan Mode 审批流；
4. 可在空闲（idle）与活跃（active）状态之间切换，而非一次性执行即退出。

---

## 2. 功能点目的

| 功能 | 目的 |
|------|------|
| `InProcessTeammateTask.kill` | 当用户或系统需要强制终止队友时，调用 `killInProcessTeammate` 中止其 `AbortController` 并清理状态。 |
| `requestTeammateShutdown` | 优雅关闭：设置 `shutdownRequested = true`，让 `inProcessRunner.ts` 的轮询逻辑检测到后，将 shutdown request 交给模型决策（approve/reject），而非直接杀死。 |
| `appendTeammateMessage` | 为“放大视图（zoomed view）”维护队友的对话历史，使用 `appendCappedMessage` 限制内存。 |
| `injectUserMessageToTeammate` | 当用户正在查看某队友的 transcript 时，允许直接输入消息并推入该队友的 `pendingUserMessages` 队列，供 `inProcessRunner.ts` 的 idle 轮询消费。 |
| `findTeammateTaskByAgentId` | 按 `agentId`（如 `researcher@my-team`）查找队友任务；**优先返回 `status === 'running'` 的任务**，以应对旧任务未从 AppState 中清除的情况。 |
| `getAllInProcessTeammateTasks` | 过滤出所有 `type === 'in_process_teammate'` 的任务。 |
| `getRunningTeammatesSorted` | 返回按 `agentName` 字母排序的运行中队友列表。该顺序被 `TeammateSpinnerTree`、`PromptInput` 底部选择器、`useBackgroundTaskNavigation` 共同依赖，**必须保持一致**。 |

---

## 3. 具体技术实现

### 3.1 Task 接口实现

```ts
export const InProcessTeammateTask: Task = {
  name: 'InProcessTeammateTask',
  type: 'in_process_teammate',
  async kill(taskId, setAppState) {
    killInProcessTeammate(taskId, setAppState);
  }
};
```

- `kill` 是 `Task` 接口唯一必须实现的方法（`spawn`/`render` 已在历史重构中移除）。
- 实际逻辑委托给 `src/utils/swarm/spawnInProcess.ts` 中的 `killInProcessTeammate`。

### 3.2 状态更新范式

所有状态变更均通过 `updateTaskState<InProcessTeammateTaskState>(taskId, setAppState, updater)` 完成：

- `updater` 接收旧 task，返回新 task；若引用未变则跳过更新（`framework.ts` 内做了短路优化）。
- 所有更新都是**不可变**的，符合 React / AppState 的 immutability 要求。

### 3.3 `findTeammateTaskByAgentId` 的 fallback 逻辑

```ts
export function findTeammateTaskByAgentId(agentId, tasks) {
  let fallback;
  for (const task of Object.values(tasks)) {
    if (isInProcessTeammateTask(task) && task.identity.agentId === agentId) {
      if (task.status === 'running') return task; // 优先返回运行中
      if (!fallback) fallback = task;             // 记录第一个匹配作为 fallback
    }
  }
  return fallback;
}
```

该设计解决了“旧 killed 任务与新 running 任务共存于 AppState”的边界情况（例如任务刚被 kill 但尚未被 `evictTerminalTask` 清理）。

### 3.4 `injectUserMessageToTeammate` 的终端状态保护

```ts
if (isTerminalTaskStatus(task.status)) {
  logForDebugging(`Dropping message for teammate task ${taskId}: task status is "${task.status}"`);
  return task;
}
```

仅当任务处于非终端状态（`pending` / `running`）时才允许注入消息，防止向已完成的队友发送消息导致状态不一致。

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入来源 | 用途 |
|---------|------|
| `../../Task.js` | `isTerminalTaskStatus`, `SetAppState`, `Task`, `TaskStateBase` |
| `../../types/message.js` | `Message` 类型 |
| `../../utils/debug.js` | `logForDebugging` |
| `../../utils/messages.js` | `createUserMessage` |
| `../../utils/swarm/spawnInProcess.js` | `killInProcessTeammate` |
| `../../utils/task/framework.js` | `updateTaskState` |
| `./types.js` | `InProcessTeammateTaskState`, `appendCappedMessage`, `isInProcessTeammateTask` |

### 4.2 调用方（上游）

- **`src/utils/swarm/backends/InProcessBackend.ts`**
  - 调用 `findTeammateTaskByAgentId` 查找任务。
  - 调用 `requestTeammateShutdown` 发送优雅关闭请求。
  - 调用 `InProcessTeammateTask.kill` / `killInProcessTeammate` 强制杀死队友。

- **`src/utils/swarm/inProcessRunner.ts`**
  - 调用 `appendTeammateMessage` 将 shutdown request / peer message 追加到 task.messages。

- **`src/hooks/useBackgroundTaskNavigation.ts`**
  - 导入 `getRunningTeammatesSorted`、`InProcessTeammateTask`。
  - `Shift+Up/Down` 导航时依赖 `getRunningTeammatesSorted` 的排序一致性。
  - `k` 键杀死选中队友时调用 `InProcessTeammateTask.kill`。

- **`src/hooks/useGlobalKeybindings.tsx`**
  - 动态 `require` 导入 `getAllInProcessTeammateTasks`，用于 `Ctrl+T` 切换 todo / teammates 视图。

- **`src/state/selectors.ts`**
  - 导入 `isInProcessTeammateTask` 用于 `getViewedTeammateTask`。

- **`src/components/tasks/taskStatusUtils.tsx`**
  - 导入 `InProcessTeammateTaskState` 类型，用于 `describeTeammateActivity`。

- **`src/commands/clear/conversation.ts`**
  - 导入 `isInProcessTeammateTask` 判断是否需要保留队友的 `agentId`。

- **`src/hooks/useTeammateViewAutoExit.ts`** / **`src/hooks/notifs/useTeammateShutdownNotification.ts`**
  - 导入 `isInProcessTeammateTask` 进行类型收窄与生命周期通知。

- **`src/tools/shared/spawnMultiAgent.ts`**
  - 导入 `InProcessTeammateTaskState` 类型，用于为 out-of-process 队友注册同名 task stub。

---

## 5. 依赖与外部交互

### 5.1 与 `inProcessRunner.ts` 的协作

`InProcessTeammateTask.tsx` 本身**不包含**任何模型调用或主循环逻辑。真正的执行循环在 `inProcessRunner.ts` 中：

1. `spawnInProcess.ts` 创建 `InProcessTeammateTaskState` 并注册到 AppState；
2. `InProcessBackend.spawn` 调用 `startInProcessTeammate(config)`（`inProcessRunner.ts` 导出）；
3. `inProcessRunner.ts` 在 `runInProcessTeammate` 中：
   - 使用 `runWithTeammateContext` + `runWithAgentContext` 包裹 `runAgent`；
   - 每轮对话结束后进入 idle 状态，轮询 `pendingUserMessages` / mailbox / task list；
   - 检测到 `shutdownRequested` 或 mailbox 中的 shutdown request 后，将请求格式化为 `<teammate-message>` XML 交给模型决策；
   - 生命周期结束（completed / failed / killed）后更新 task status 并清理。

### 5.2 与 Mailbox 系统的交互

In-process 队友与 out-of-process（tmux/iTerm2）队友共用同一套 **file-based mailbox**（`src/utils/teammateMailbox.js`）：

- `inProcessRunner.ts` 通过 `readMailbox` / `writeToMailbox` 收发消息；
- `InProcessBackend.sendMessage` / `terminate` 也通过 `writeToMailbox` 与队友通信。

### 5.3 与 UI 的交互

- **TeammateSpinnerTree** / **PromptInputFooterLeftSide**：使用 `getRunningTeammatesSorted` 获取列表；
- **InProcessTeammateDetailDialog**：展示 `task.messages`、`task.progress`、`task.error` 等字段；
- **BackgroundTasksDialog**：通过 `isInProcessTeammateTask` 过滤并渲染队友任务卡片。

---

## 6. 风险、边界与改进建议

### 6.1 风险与边界

1. **AppState 中旧任务残留**
   - `findTeammateTaskByAgentId` 的 fallback 机制虽能缓解，但若调用方未区分 running / terminal，仍可能误操作旧任务。
   - `killInProcessTeammate` 和 `evictTerminalTask` 存在时序竞争，快速连续 kill/spawn 同一 `agentId` 可能导致状态混乱。

2. **`getRunningTeammatesSorted` 的排序契约**
   - 该排序被多处 UI 共享，且 `selectedIPAgentIndex` 直接映射到该数组下标。若未来某处更改排序规则（如改为按 `startTime`），会导致键盘导航与显示错位。

3. **`injectUserMessageToTeammate` 的终端状态检查**
   - 当前仅拒绝 `isTerminalTaskStatus`，但 `pending` 状态的任务也能接收消息。若队友尚未启动（`spawnInProcess` 成功但 `inProcessRunner` 未开始），消息可能被积压但永远不会消费，造成用户困惑。

4. **内存与消息上限**
   - `appendTeammateMessage` 使用 `appendCappedMessage`（上限 `TEAMMATE_MESSAGES_UI_CAP = 50`），但 `task.messages` 只是** UI 镜像**；真正的完整对话在 `inProcessRunner.ts` 的 `allMessages` 中维护。若 `allMessages` 无上限，长会话仍可能 OOM。

5. **`kill` 的异步语义**
   - `Task.kill` 签名要求 `async`，但 `InProcessTeammateTask.kill` 内部调用的是同步的 `killInProcessTeammate`。虽然当前无问题，但若未来 `killInProcessTeammate` 改为异步（例如加入磁盘清理），需确保 await 被正确传递。

### 6.2 改进建议

1. **统一任务查找接口**
   - 建议将 `findTeammateTaskByAgentId` 增加可选参数 `requireRunning?: boolean`，使调用方显式声明意图，减少 fallback 误用。

2. **`injectUserMessageToTeammate` 增加 `isIdle` 或 `status === 'running'` 限制**
   - 只允许向 `running` 或 `isIdle === true` 的队友注入消息，避免消息在 `pending` 状态无限积压。

3. **排序契约文档化或提取为 selector**
   - 将 `getRunningTeammatesSorted` 的排序逻辑提升为 `src/state/selectors.ts` 中的 selector，并添加注释说明“修改排序需同步检查所有 `selectedIPAgentIndex` 消费者”。

4. **考虑将 `InProcessTeammateTask.tsx` 中的纯函数拆分为更细粒度模块**
   - 当前文件混合了 Task 接口实现、状态更新 helper、查询 helper。随着功能增长，可考虑拆分为：
     - `inProcessTeammateQueries.ts`（查找/枚举）
     - `inProcessTeammateMutations.ts`（shutdown / inject / append）

5. **增加 `kill` 的幂等性保证**
   - 在 `killInProcessTeammate` 中已做 `status !== 'running'` 短路，但可在 `InProcessTeammateTask.kill` 层增加日志或返回值，便于上层判断是否真的执行了 kill。

---

*文档结束*
