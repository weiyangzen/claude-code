# src/utils/inProcessTeammateHelpers.ts 研究文档

## 场景与职责

`inProcessTeammateHelpers.ts` 为 Claude Code 的"进程内队友"（In-Process Teammate）功能提供状态管理辅助函数。进程内队友是 Agent Swarm 架构的一部分，允许主 Claude 进程在同一个运行时中启动并管理子代理任务，而不是通过外部进程或网络通信。

该模块的职责包括：
1. **任务 ID 查找**：根据队友的 agent 名称查找对应的任务 ID。
2. **计划审批状态管理**：设置/重置队友的 `awaitingPlanApproval` 状态。
3. **计划审批响应处理**：当队友收到计划审批响应消息时更新状态。
4. **权限相关消息检测**：判断收到的消息是否为权限委托响应（工具权限或沙盒权限）。

调用方主要是 `src/hooks/useInboxPoller.ts`，用于轮询队友收件箱并处理 incoming 消息。

## 功能点目的

### 1. `findInProcessTeammateTaskId`
在 `appState.tasks` 中遍历所有任务，通过 `isInProcessTeammateTask` 类型守卫筛选出进程内队友任务，再匹配 `task.identity.agentName`。用于将 agent 名称映射到具体的任务实例 ID。

### 2. `setAwaitingPlanApproval`
使用 `updateTaskState` 泛型辅助函数，安全地更新指定任务的状态字段 `awaitingPlanApproval`。这是 React 状态更新模式（函数式 `setAppState`）的封装。

### 3. `handlePlanApprovalResponse`
计划审批响应的回调处理函数。当前实现非常简单：仅调用 `setAwaitingPlanApproval(taskId, setAppState, false)` 重置等待状态。注释说明：响应中的 `permissionMode` 由 agent 循环（Task #11）单独处理，不在此处消费。

### 4. `isPermissionRelatedResponse`
检测消息文本是否为权限相关响应。底层委托给 `teammateMailbox.ts` 中的 `isPermissionResponse` 和 `isSandboxPermissionResponse`。用于 inbox poller 快速识别需要特殊处理的队友消息。

## 具体技术实现

### 类型定义
```typescript
type SetAppState = (updater: (prev: AppState) => AppState) => void
```

所有状态更新函数都接受 `SetAppState` 作为参数，确保与 React 的 `useState` 或 `useReducer` 的 setter 签名兼容。

### 任务遍历与类型守卫
```typescript
export function findInProcessTeammateTaskId(agentName: string, appState: AppState): string | undefined {
  for (const task of Object.values(appState.tasks)) {
    if (
      isInProcessTeammateTask(task) &&
      task.identity.agentName === agentName
    ) {
      return task.id
    }
  }
  return undefined
}
```

这里使用了 `Object.values(appState.tasks)` 进行线性扫描。`appState.tasks` 通常数量很小（个位数），所以线性扫描的性能开销可以忽略。

### 状态更新封装
```typescript
export function setAwaitingPlanApproval(taskId: string, setAppState: SetAppState, awaiting: boolean): void {
  updateTaskState<InProcessTeammateTaskState>(taskId, setAppState, task => ({
    ...task,
    awaitingPlanApproval: awaiting,
  }))
}
```

`updateTaskState` 来自 `./task/framework.js`，它内部会处理任务不存在的情况（通常是 no-op 或安全跳过）。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/inProcessTeammateHelpers.ts:33-46` | `findInProcessTeammateTaskId` 任务查找 |
| `src/utils/inProcessTeammateHelpers.ts:55-64` | `setAwaitingPlanApproval` 状态更新 |
| `src/utils/inProcessTeammateHelpers.ts:77-83` | `handlePlanApprovalResponse` 响应处理 |
| `src/utils/inProcessTeammateHelpers.ts:97-102` | `isPermissionRelatedResponse` 权限检测 |
| `src/state/AppState.ts` | `AppState` 类型定义 |
| `src/tasks/InProcessTeammateTask/types.ts` | `InProcessTeammateTaskState`, `isInProcessTeammateTask` |
| `src/utils/task/framework.ts` | `updateTaskState` 泛型状态更新辅助 |
| `src/utils/teammateMailbox.ts` | `isPermissionResponse`, `isSandboxPermissionResponse`, `PlanApprovalResponseMessage` |

## 依赖与外部交互

### 内部依赖
- `../state/AppState.js`：`AppState`
- `../tasks/InProcessTeammateTask/types.js`：`InProcessTeammateTaskState`, `isInProcessTeammateTask`
- `./task/framework.js`：`updateTaskState`
- `./teammateMailbox.js`：`isPermissionResponse`, `isSandboxPermissionResponse`, `PlanApprovalResponseMessage`

### 调用方
- `src/hooks/useInboxPoller.ts`：轮询队友收件箱时调用

## 风险、边界与改进建议

### 风险与边界
1. **线性扫描性能瓶颈**：`findInProcessTeammateTaskId` 使用 `Object.values` 遍历所有任务。虽然当前任务数量少，但如果未来 Swarm 规模扩大（数十上百个队友），每次消息处理都线性扫描可能成为瓶颈。建议增加反向索引（`agentName -> taskId` 的 Map）。
2. **`handlePlanApprovalResponse` 的空实现风险**：当前函数仅重置 `awaitingPlanApproval`，注释说 `permissionMode` 由 Task #11 处理。这种分散处理模式容易导致逻辑遗漏——如果 Task #11 的代码被重构或删除，此处不会有任何编译时提示。
3. **无错误处理**：`setAwaitingPlanApproval` 依赖 `updateTaskState`，若传入的 `taskId` 不存在，行为取决于 `updateTaskState` 的实现（通常是静默忽略）。这在调试时可能隐藏问题。
4. **权限检测的字符串匹配脆弱性**：`isPermissionRelatedResponse` 最终依赖 `teammateMailbox.ts` 中的字符串匹配来识别权限响应。如果消息格式发生变化（如增加新字段、改变前缀），检测可能失效。
5. **硬编码的 Task #11 引用**：注释中提到的 "Task #11" 是外部项目管理的引用，对代码阅读者没有直接价值，且可能随项目管理工具变化而过时。

### 改进建议
1. **增加反向索引缓存**：在 `AppState` 或相关模块中维护 `Map<agentName, taskId>`，将查找复杂度从 O(n) 降到 O(1)。
2. **统一计划审批响应处理**：考虑将 `permissionMode` 的解析和应用也纳入 `handlePlanApprovalResponse`，或至少在此处添加断言/校验，确保 Task #11 确实处理了响应。
3. **增强类型安全**：若 `updateTaskState` 在任务不存在时返回某种指示（如 `boolean`），可以向上传播该信息，便于调用方决定是否记录警告。
4. **单元测试覆盖**：该模块逻辑简单但处于 Swarm 消息处理的关键路径，建议为 `findInProcessTeammateTaskId` 和 `isPermissionRelatedResponse` 编写单元测试，覆盖任务存在/不存在、权限消息匹配/不匹配等场景。
5. **文档化 Swarm 状态流转**：在模块顶部添加更详细的注释或指向设计文档的链接，说明 `awaitingPlanApproval` 在整个队友生命周期中的状态流转图。
