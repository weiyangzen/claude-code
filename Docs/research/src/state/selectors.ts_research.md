# src/state/selectors.ts 研究文档

## 场景与职责

`selectors.ts` 是 Claude Code `AppState` 的 **派生状态计算层**。它的职责非常聚焦：

1. **提供纯函数式的 selector**，从 `AppState` 中提取或组合出组件/工具所需的视图模型。
2. **保持 selectors 纯净** — 文件顶部注释明确要求“just data extraction, no side effects”。
3. **封装类型安全的输入路由判断** — 特别是 teammate（队友）视图与主会话之间的输入流向。

该文件是状态管理架构中的“只读层”，与 `useAppState` 的 selector 模式配合使用，帮助组件避免在 render 中直接做复杂的状态推导。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getViewedTeammateTask` | 判断当前是否正在查看某个 teammate 的任务，并返回其 `InProcessTeammateTaskState`；若不存在或类型不符则返回 `undefined`。 |
| `getActiveAgentForInput` | 决定用户输入应该被路由到哪里：leader 主会话、正在查看的 teammate、还是具名的 local agent。返回 discriminated union 保证类型安全。 |

## 具体技术实现

### 1. getViewedTeammateTask

```ts
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>,
): InProcessTeammateTaskState | undefined {
  const { viewingAgentTaskId, tasks } = appState

  if (!viewingAgentTaskId) {
    return undefined
  }

  const task = tasks[viewingAgentTaskId]
  if (!task) {
    return undefined
  }

  if (!isInProcessTeammateTask(task)) {
    return undefined
  }

  return task
}
```

- 使用 `Pick<AppState, ...>` 作为参数类型，明确声明该 selector 只依赖两个字段，便于单元测试和重构。
- 三层防御式返回：
  1. 没有 `viewingAgentTaskId` → `undefined`
  2. 任务 ID 在 `tasks` 映射中不存在 → `undefined`
  3. 任务存在但类型不是 `in_process_teammate` → `undefined`
- 类型收窄：通过 `isInProcessTeammateTask` 类型守卫，返回值为 `InProcessTeammateTaskState | undefined`。

### 2. getActiveAgentForInput

```ts
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }

export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) {
    return { type: 'viewed', task: viewedTask }
  }

  const { viewingAgentTaskId, tasks } = appState
  if (viewingAgentTaskId) {
    const task = tasks[viewingAgentTaskId]
    if (task?.type === 'local_agent') {
      return { type: 'named_agent', task }
    }
  }

  return { type: 'leader' }
}
```

- **优先级**：先检查是否在看 teammate（`in_process_teammate`），再检查是否在看 `local_agent`，最后回退到 `leader`。
- **Discriminated Union**：通过 `type` 字段区分三种路由目标，调用方可用 `switch (result.type)` 做 exhaustive type check，编译器会强制处理所有分支。
- 被输入提交逻辑（如 `src/utils/handlePromptSubmit.ts`、附件生成逻辑）用于决定将用户消息发送给谁。

## 关键代码路径与文件引用

| 代码路径 | 说明 |
|----------|------|
| `getViewedTeammateTask` (L18) | 被 `getActiveAgentForInput` 内部调用；也被 `src/utils/attachments.ts` 直接引用，用于生成 teammate 相关的附件。 |
| `getActiveAgentForInput` (L59) | 被 `src/components/PromptInput/PromptInput.tsx`、`src/components/PromptInput/useSwarmBanner.ts`、`src/components/Spinner.tsx`、`src/components/TeammateViewHeader.tsx`、`src/utils/attachments.ts` 等引用。 |
| `ActiveAgentForInput` 类型 (L46) | 被调用方用于类型安全的输入路由分支处理。 |

## 依赖与外部交互

### 直接依赖

- `./AppStateStore.js` → `AppState` 类型
- `../tasks/InProcessTeammateTask/types.js` → `InProcessTeammateTaskState` 类型与 `isInProcessTeammateTask` 类型守卫
- `../tasks/LocalAgentTask/LocalAgentTask.js` → `LocalAgentTaskState` 类型

### 调用方（上游）

- `src/utils/attachments.ts`：在 `getTeammateMailboxAttachments` / `getTeamContextAttachment` 等函数中调用 `getViewedTeammateTask`。
- `src/components/PromptInput/PromptInput.tsx`：决定输入框 placeholder、提交路由、footer 显示。
- `src/components/PromptInput/useSwarmBanner.ts`：判断是否在 teammate 视图以显示 swarm banner。
- `src/components/Spinner.tsx`：判断当前活跃代理以渲染正确的 spinner 行。
- `src/components/TeammateViewHeader.tsx`：基于 `getActiveAgentForInput` 渲染 teammate 视图的头部信息。

## 风险、边界与改进建议

### 风险

1. **selector 与 AppState 结构耦合**  
   `getViewedTeammateTask` 直接依赖 `tasks[taskId]` 的映射结构。若未来 `tasks` 的存储方式改为嵌套或分片，所有依赖此 selector 的调用方都需要同步修改。

2. **`local_agent` 与 `in_process_teammate` 的优先级隐含约定**  
   `getActiveAgentForInput` 先判断 teammate 再判断 local_agent，这意味着如果 `viewingAgentTaskId` 同时满足两种条件（理论上不应发生），输入会流向 teammate。这种优先级是隐含的，没有文档化，新开发者容易忽略。

3. **类型守卫的重复实现**  
   `isInProcessTeammateTask` 在 `types.ts` 中已有定义，但 `teammateViewHelpers.ts` 中为了打破循环依赖又内联了一个类似的检查。若类型守卫逻辑变更，两处可能失步。

### 边界

- selectors 只处理 **同步、纯函数** 的推导；任何异步或带副作用的逻辑（如 API 调用、磁盘读取）都不应放入本文件。
- `getViewedTeammateTask` 的参数使用 `Pick` 而非完整 `AppState`，这允许测试时传入最小化 stub，但也意味着调用方若传入完整 `AppState` 依然兼容（TypeScript 结构化类型）。

### 改进建议

1. **增加单元测试覆盖边界条件**  
   当前未找到对应的 `.test.ts` 文件。建议为以下场景补充测试：
   - `viewingAgentTaskId` 存在但 `tasks` 中已删除（eviction 场景）
   - `viewingAgentTaskId` 指向 `local_agent` 而非 teammate
   - `viewingAgentTaskId` 为 `''`（空字符串）时的行为

2. **将优先级策略显式文档化**  
   在 `getActiveAgentForInput` 的 JSDoc 中明确说明路由优先级：`teammate > local_agent > leader`，并解释原因（teammate 是 in-process 的实时协作代理，优先级更高）。

3. **考虑引入 reselect / memoization**  
   当前 selectors 非常轻量，无需 memo。但随着 `tasks` 对象增大，`getViewedTeammateTask` 若被高频调用（如每次按键），可考虑引入 `reselect` 或简单的 memo 包装，避免重复的对象查找和类型守卫调用。

4. **统一类型守卫来源**  
   评估是否可通过调整导入顺序或创建共享的 `taskGuards.ts`，消除 `teammateViewHelpers.ts` 中内联类型守卫的重复代码。
