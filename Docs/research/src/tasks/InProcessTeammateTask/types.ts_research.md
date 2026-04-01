# types.ts 研究文档

> 文件路径：`src/tasks/InProcessTeammateTask/types.ts`  
> 研究时间：2026-04-01  
> 关联文件：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`、`src/utils/swarm/spawnInProcess.ts`、`src/utils/swarm/inProcessRunner.ts`、`src/Task.ts`

---

## 1. 场景与职责

`types.ts` 是 In-Process Teammate 的**类型定义与类型守卫中枢**。它承担以下职责：

1. **定义队友身份模型**：`TeammateIdentity` 将运行时 `TeammateContext`（基于 `AsyncLocalStorage`）中的身份字段抽离为可序列化的纯数据对象，用于持久化到 `AppState`。
2. **定义任务状态类型**：`InProcessTeammateTaskState` 扩展 `TaskStateBase`，涵盖身份、执行、权限模式、对话镜像、生命周期、进度追踪等全部状态字段。
3. **提供类型守卫**：`isInProcessTeammateTask` 是跨组件识别队友任务的唯一类型收窄入口。
4. **提供消息数组上限工具**：`appendCappedMessage` 与常量 `TEAMMATE_MESSAGES_UI_CAP` 控制 `task.messages` 的内存占用，防止长会话下 AppState 膨胀。

---

## 2. 功能点目的

| 功能 | 目的 |
|------|------|
| `TeammateIdentity` | 将运行时上下文中的 `agentId`、`agentName`、`teamName`、`color`、`planModeRequired`、`parentSessionId` 抽取为纯数据类型，供 AppState 持久化与跨模块传递。 |
| `InProcessTeammateTaskState` | 作为 `TaskStateBase` 的特化，完整描述进程内队友在 UI、执行、生命周期中的全部状态。 |
| `isInProcessTeammateTask` | 运行时类型守卫，确保在 `Record<string, TaskStateBase>` 中安全地识别并收窄为 `InProcessTeammateTaskState`。 |
| `TEAMMATE_MESSAGES_UI_CAP` | 限制 `task.messages` 的最大长度（50 条），防止 zoomed transcript view 的镜像数组无界增长。 |
| `appendCappedMessage` | 在保持不可变性的前提下，向消息数组追加元素并丢弃最旧条目，始终返回新数组。 |

---

## 3. 具体技术实现

### 3.1 TeammateIdentity 结构

```ts
export type TeammateIdentity = {
  agentId: string            // 如 "researcher@my-team"
  agentName: string          // 如 "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean  // 是否强制进入 plan mode
  parentSessionId: string    // 队长的 session ID，用于 task list / transcript 关联
}
```

- 与 `TeammateContext`（`src/utils/teammateContext.ts`）形状一致，但**明确注释**“NOT a reference to AsyncLocalStorage”，强调其可序列化、可持久化特性。
- `parentSessionId` 是关键关联字段：队长以自身 `sessionId` 创建 task list，队友通过该 ID 去 claim task。

### 3.2 InProcessTeammateTaskState 字段详解

```ts
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController              // 生命周期：杀死整个队友
  currentWorkAbortController?: AbortController   // 当前 turn：Escape 仅中断本轮
  unregisterCleanup?: () => void                 // 进程退出时的清理回调
  awaitingPlanApproval: boolean                  // Plan mode 审批中
  permissionMode: PermissionMode                 // 独立循环的权限模式（Shift+Tab）
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  messages?: Message[]                           // UI 镜像（非 mailbox 消息）
  inProgressToolUseIDs?: Set<string>             // 转录视图动画用
  pendingUserMessages: string[]                  // 用户注入的消息队列
  spinnerVerb?: string
  pastTenseVerb?: string
  isIdle: boolean
  shutdownRequested: boolean
  onIdleCallbacks?: Array<() => void>           // 队长 waitForIdle 的回调
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

#### 关键字段设计意图

- **`abortController` vs `currentWorkAbortController`**
  - `abortController`：由 `spawnInProcess.ts` 创建，用于**整个队友生命周期**。调用 `killInProcessTeammate` 时会 `abort()` 它，导致 `inProcessRunner.ts` 的主循环彻底退出。
  - `currentWorkAbortController`：每轮 `runAgent` 前由 `inProcessRunner.ts` 创建，用于**仅中断当前 turn**。用户在 zoomed view 按 Escape 时触发的是它，队友保持 alive 并回到 idle 状态。

- **`messages` 与 `pendingUserMessages` 的分离**
  - `messages`：供 `InProcessTeammateDetailDialog` 等 UI 组件展示的**对话镜像**，受 50 条上限保护。
  - `pendingUserMessages`：供 `inProcessRunner.ts` 在 idle 轮询中消费的**纯文本队列**，无上限（但通常很短）。

- **`onIdleCallbacks`**
  - 运行时专用，不序列化到磁盘。`inProcessRunner.ts` 在队友进入 idle 时调用这些回调，使队长能高效等待而无需轮询。

- **`lastReportedToolCount` / `lastReportedTokenCount`**
  - 用于通知系统计算增量，避免重复播报同一队友的进度。

### 3.3 类型守卫实现

```ts
export function isInProcessTeammateTask(task: unknown): task is InProcessTeammateTaskState {
  return (
    typeof task === 'object' &&
    task !== null &&
    'type' in task &&
    task.type === 'in_process_teammate'
  )
}
```

- 采用宽松检查（仅校验 `type` 字段），因为 `task` 实际来源是 `AppState.tasks`，运行时类型安全由 `registerTask` 保证。
- 该守卫被**大量上游文件**直接导入使用，是识别队友任务的“标准入口”。

### 3.4 appendCappedMessage 实现

```ts
export const TEAMMATE_MESSAGES_UI_CAP = 50

export function appendCappedMessage<T>(prev: readonly T[] | undefined, item: T): T[] {
  if (prev === undefined || prev.length === 0) {
    return [item]
  }
  if (prev.length >= TEAMMATE_MESSAGES_UI_CAP) {
    const next = prev.slice(-(TEAMMATE_MESSAGES_UI_CAP - 1))
    next.push(item)
    return next
  }
  return [...prev, item]
}
```

- **不可变性**：当需要截断时，先 `slice` 再 `push`（`next` 是新数组）；未达上限时用 spread 语法。两种路径都返回新数组，满足 React immutability 要求。
- **性能权衡**：`slice` + `push` 比 `concat` 或多次 spread 更高效，且避免了 `prev` 被修改（`prev` 是 `readonly`）。

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入来源 | 用途 |
|---------|------|
| `../../Task.js` | `TaskStateBase` |
| `../../tools/AgentTool/agentToolUtils.js` | `AgentToolResult` |
| `../../tools/AgentTool/loadAgentsDir.js` | `AgentDefinition` |
| `../../types/message.js` | `Message` |
| `../../utils/permissions/PermissionMode.js` | `PermissionMode` |
| `../LocalAgentTask/LocalAgentTask.js` | `AgentProgress` |

### 4.2 上游消费者（按功能分类）

#### 类型与守卫消费者
- **`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`**
  - 导入 `InProcessTeammateTaskState`、`appendCappedMessage`、`isInProcessTeammateTask`。
- **`src/tasks/types.ts`**
  - 将 `InProcessTeammateTaskState` 纳入 `TaskState` / `BackgroundTaskState` 联合类型。
- **`src/state/selectors.ts`**
  - 使用 `isInProcessTeammateTask` 实现 `getViewedTeammateTask`。
- **`src/components/tasks/taskStatusUtils.tsx`**
  - 导入 `InProcessTeammateTaskState` 类型用于 `describeTeammateActivity`。
- **`src/components/tasks/InProcessTeammateDetailDialog.tsx`**
  - 导入 `InProcessTeammateTaskState` 作为 props 类型。
- **`src/components/Spinner.tsx`** / **`src/components/TaskListV2.tsx`** / **`src/components/tasks/BackgroundTasksDialog.tsx`**
  - 使用 `isInProcessTeammateTask` 进行类型收窄与条件渲染。

#### 状态创建与修改消费者
- **`src/utils/swarm/spawnInProcess.ts`**
  - 导入 `InProcessTeammateTaskState`、`TeammateIdentity` 用于构造初始 task state。
- **`src/utils/swarm/inProcessRunner.ts`**
  - 导入 `InProcessTeammateTaskState`、`TeammateIdentity`、`appendCappedMessage`。
- **`src/utils/swarm/backends/InProcessBackend.ts`**
  - 间接通过 `InProcessTeammateTask.tsx` 消费类型。
- **`src/utils/inProcessTeammateHelpers.ts`**
  - 导入 `InProcessTeammateTaskState`、`isInProcessTeammateTask` 实现 plan approval helper。
- **`src/tools/shared/spawnMultiAgent.ts`**
  - 导入 `InProcessTeammateTaskState` 为 out-of-process 队友注册同名 task stub。

#### Hook / 命令消费者
- **`src/hooks/useBackgroundTaskNavigation.ts`** / **`src/hooks/useTeammateViewAutoExit.ts`** / **`src/hooks/notifs/useTeammateShutdownNotification.ts`** / **`src/hooks/useInboxPoller.ts`**
  - 均使用 `isInProcessTeammateTask` 进行过滤或类型收窄。
- **`src/hooks/useGlobalKeybindings.tsx`**
  - 动态 require `InProcessTeammateTask.tsx`，间接依赖本类型。
- **`src/commands/clear/conversation.ts`**
  - 使用 `isInProcessTeammateTask` 判断哪些任务应在 `/clear` 后保留。

---

## 5. 依赖与外部交互

### 5.1 与 TaskState 联合类型的关系

`src/tasks/types.ts` 将 `InProcessTeammateTaskState` 纳入：

```ts
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

这意味着任何处理 `TaskState` 的通用逻辑（如 `framework.ts` 的 `updateTaskState`、`evictTerminalTask`、`BackgroundTasksDialog` 的渲染）都必须能正确识别并兼容 `InProcessTeammateTaskState`。

### 5.2 与 LocalAgentTask 的复用关系

`InProcessTeammateTaskState` 复用了 `LocalAgentTask` 中的两个类型：

- `AgentProgress`：进度追踪（recentActivities、tokenCount、toolUseCount 等）。
- `AgentToolResult`：队友执行完成后，将其结果存储在 `result` 字段中。

这种复用体现了“队友本质上也是 agent”的设计哲学，但 `InProcessTeammateTaskState` 额外增加了团队相关的身份与生命周期字段。

### 5.3 与 AppState 序列化的边界

部分字段明确标注为 **Runtime only**（不序列化到磁盘）：

- `abortController`
- `currentWorkAbortController`
- `unregisterCleanup`
- `onIdleCallbacks`

这些字段在 `spawnInProcess.ts` 创建 task state 时注入，在 `killInProcessTeammate` 或 `runInProcessTeammate` 完成时被显式置为 `undefined`，以防止不可序列化对象进入磁盘快照或导致 JSON 序列化失败。

---

## 6. 风险、边界与改进建议

### 6.1 风险与边界

1. **`TEAMMATE_MESSAGES_UI_CAP = 50` 的硬编码**
   - 注释中引用了 BQ 分析（2026-03-20）说明内存问题，但 50 条是经验值。对于超长 tool result 消息，50 条仍可能占用数 MB；对于极短消息，50 条可能不足以提供有效上下文。
   - 当前上限仅作用于 `task.messages`（UI 镜像），`inProcessRunner.ts` 中的 `allMessages` 不受此限制，真正的内存大户可能在那里。

2. **`isInProcessTeammateTask` 的宽松检查**
   - 仅检查 `type === 'in_process_teammate'`，不验证其他必填字段（如 `identity`、`prompt`）。若某处手动构造了残缺对象并放入 `AppState.tasks`，后续代码可能在使用 `identity.agentId` 时抛出运行时错误。

3. **`pendingUserMessages` 无上限**
   - 虽然正常使用场景下用户不会一次性注入数百条消息，但恶意或异常调用可能导致该数组无限增长，进而影响 AppState 大小与 idle 轮询性能。

4. **`InProcessTeammateTaskState` 的字段膨胀**
   - 当前类型已包含 20+ 字段，混合了 UI 状态（`spinnerVerb`、`pastTenseVerb`、`inProgressToolUseIDs`）、执行状态（`abortController`、`currentWorkAbortController`）、权限状态（`permissionMode`、`awaitingPlanApproval`）和生命周期状态（`isIdle`、`shutdownRequested`）。随着功能迭代，类型可能进一步膨胀，增加理解与维护成本。

5. **`Set<string>` 的序列化风险**
   - `inProgressToolUseIDs` 是 `Set<string>`。虽然运行时明确清理，但若某条代码路径忘记将其置 `undefined` 就尝试 `JSON.stringify` AppState，会丢失 Set 内容或导致序列化结果不符合预期。

### 6.2 改进建议

1. **将 `TEAMMATE_MESSAGES_UI_CAP` 改为可配置或按 token 估算**
   - 与其固定 50 条，不如根据消息内容的估算 token 数动态截断，使内存占用更可控。或者将其提升为 `GlobalConfig` 中的隐藏配置项，便于 A/B 测试。

2. **增强 `isInProcessTeammateTask` 的健壮性**
   - 建议增加对 `identity` 和 `prompt` 的存在性检查：
     ```ts
     'identity' in task &&
     typeof task.identity === 'object' &&
     task.identity !== null &&
     'agentId' in task.identity
     ```
   - 这能在早期捕获残缺对象，避免下游出现难以调试的 `Cannot read property 'agentId' of undefined`。

3. **为 `pendingUserMessages` 增加上限保护**
   - 可引入类似 `appendCappedMessage` 的机制，或当队列长度超过阈值时丢弃最旧消息并记录 warning，防止异常注入导致内存泄漏。

4. **类型拆分：将 UI-only 字段提取为子对象**
   - 建议将纯 UI 字段（`spinnerVerb`、`pastTenseVerb`、`inProgressToolUseIDs`）提取到 `uiState` 子对象中：
     ```ts
     uiState: {
       spinnerVerb?: string
       pastTenseVerb?: string
       inProgressToolUseIDs?: Set<string>
     }
     ```
   - 这样能让 `InProcessTeammateTaskState` 的主结构更清晰，也便于 UI 组件做细粒度订阅优化。

5. **统一 Runtime-only 字段的标记方式**
   - 当前通过注释标记 runtime-only 字段，容易被忽略。可考虑使用 TypeScript 的 branded type 或单独定义一个 `SerializedInProcessTeammateTaskState` 类型，在持久化前通过类型系统强制剥离不可序列化字段。

6. **文档化 `lastReportedToolCount` / `lastReportedTokenCount` 的使用契约**
   - 这两个字段目前仅在通知系统内部使用，但定义在核心类型中。建议添加 JSDoc 说明其消费者（如 `useTeammateShutdownNotification` 或相关通知 hook），防止未来被误用或误删。

---

*文档结束*
