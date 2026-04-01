# src/state/teammateViewHelpers.ts 研究文档

## 场景与职责

`teammateViewHelpers.ts` 是 Claude Code ** teammate（队友/蜂群成员）视图状态转换** 的专用工具模块。它封装了用户在不同 teammate 之间切换、退出 teammate 视图、以及停止/解散 agent 时的状态更新逻辑。

核心职责：

1. **`enterTeammateView`** — 进入某个 teammate 的 transcript 视图，设置 `viewingAgentTaskId`，并对 `local_agent` 任务启用 `retain` 标志（阻止 eviction、启用流式追加、触发磁盘引导）。
2. **`exitTeammateView`** — 退出 teammate 视图，返回 leader 主会话，释放 `retain`，若任务已终止则设置 `evictAfter` 延迟清理。
3. **`stopOrDismissAgent`** — 上下文敏感的停止/解散操作：running 状态则 abort，terminal 状态则立即标记 `evictAfter=0` 隐藏。

该模块是命令式工具函数集合，不依赖 React，可被组件、Hook、命令处理器等多种调用方使用。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `enterTeammateView` | 支持用户通过 `f` 键或 UI 点击切换到 teammate 的 transcript；同时处理从上一个 agent 切走时的释放逻辑。 |
| `exitTeammateView` | 支持用户返回 leader 视图（如按 Esc 或选择 leader）；清理 `retain` 与 `messages`，控制内存占用。 |
| `stopOrDismissAgent` | 为 BackgroundTasksDialog 和 PromptInput 提供统一的 `x` 键行为：running → abort，terminal → dismiss。 |
| `release` (内部) | 将 `local_agent` 任务还原为“stub 形态”：丢弃 `retain`、清空 `messages`、若终端状态则设置 `evictAfter`。 |

## 具体技术实现

### 1. release 内部辅助函数

```ts
function release(task: LocalAgentTaskState): LocalAgentTaskState {
  return {
    ...task,
    retain: false,
    messages: undefined,
    diskLoaded: false,
    evictAfter: isTerminalTaskStatus(task.status)
      ? Date.now() + PANEL_GRACE_MS
      : undefined,
  }
}
```

- `PANEL_GRACE_MS = 30_000`（与 `framework.ts` 保持一致）。
- `isTerminalTaskStatus` 判断任务是否处于终止状态（如 `completed`、`failed`、`killed`）。
- 清空 `messages` 是为了减少内存占用；`diskLoaded: false` 表示下次 retain 时需要重新从磁盘引导。

### 2. enterTeammateView

```ts
export function enterTeammateView(
  taskId: string,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  logEvent('tengu_transcript_view_enter', {})
  setAppState(prev => {
    const task = prev.tasks[taskId]
    const prevId = prev.viewingAgentTaskId
    const prevTask = prevId !== undefined ? prev.tasks[prevId] : undefined
    const switching =
      prevId !== undefined &&
      prevId !== taskId &&
      isLocalAgent(prevTask) &&
      prevTask.retain
    const needsRetain =
      isLocalAgent(task) && (!task.retain || task.evictAfter !== undefined)
    const needsView =
      prev.viewingAgentTaskId !== taskId ||
      prev.viewSelectionMode !== 'viewing-agent'
    if (!needsRetain && !needsView && !switching) return prev
    let tasks = prev.tasks
    if (switching || needsRetain) {
      tasks = { ...prev.tasks }
      if (switching) tasks[prevId] = release(prevTask)
      if (needsRetain) {
        tasks[taskId] = { ...task, retain: true, evictAfter: undefined }
      }
    }
    return {
      ...prev,
      viewingAgentTaskId: taskId,
      viewSelectionMode: 'viewing-agent',
      tasks,
    }
  })
}
```

- **analytics**：进入时记录 `tengu_transcript_view_enter`。
- **switching 检测**：若之前在看另一个已被 retain 的 `local_agent`，先 `release` 它。
- **needsRetain**：目标任务是 `local_agent` 且尚未 retain 或已有 eviction 倒计时，则设置 `retain: true` 并清除 `evictAfter`。
- **needsView**：避免无意义的重复进入同一视图。
- **不可变更新**：`tasks` 对象只在需要时才浅拷贝，否则返回原 `prev` 引用短路。

### 3. exitTeammateView

```ts
export function exitTeammateView(
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  logEvent('tengu_transcript_view_exit', {})
  setAppState(prev => {
    const id = prev.viewingAgentTaskId
    const cleared = {
      ...prev,
      viewingAgentTaskId: undefined,
      viewSelectionMode: 'none' as const,
    }
    if (id === undefined) {
      return prev.viewSelectionMode === 'none' ? prev : cleared
    }
    const task = prev.tasks[id]
    if (!isLocalAgent(task) || !task.retain) return cleared
    return {
      ...cleared,
      tasks: { ...prev.tasks, [id]: release(task) },
    }
  })
}
```

- **analytics**：退出时记录 `tengu_transcript_view_exit`。
- 若当前没有在看任何 agent，仅当 `viewSelectionMode` 不是 `'none'` 时才更新（防御性清理）。
- 若在看 `local_agent` 且已 retain，则 `release` 该任务。

### 4. stopOrDismissAgent

```ts
export function stopOrDismissAgent(
  taskId: string,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  setAppState(prev => {
    const task = prev.tasks[taskId]
    if (!isLocalAgent(task)) return prev
    if (task.status === 'running') {
      task.abortController?.abort()
      return prev
    }
    if (task.evictAfter === 0) return prev
    const viewingThis = prev.viewingAgentTaskId === taskId
    return {
      ...prev,
      tasks: {
        ...prev.tasks,
        [taskId]: { ...release(task), evictAfter: 0 },
      },
      ...(viewingThis && {
        viewingAgentTaskId: undefined,
        viewSelectionMode: 'none',
      }),
    }
  })
}
```

- **running**：直接调用 `abortController.abort()`，注意这里 **mutate 了 task 对象**（调用 abort），但返回原 `prev` 以短路重渲染。abort 会触发任务内部的清理逻辑，最终通过其他路径更新状态。
- **已 dismiss**：若 `evictAfter === 0` 则幂等返回。
- **terminal**：`release` 后强制 `evictAfter: 0`，使 UI 立即过滤掉该任务；如果当前正在看该任务，则同步退出视图。

### 5. 类型守卫的内联实现

```ts
function isLocalAgent(task: unknown): task is LocalAgentTaskState {
  return (
    typeof task === 'object' &&
    task !== null &&
    'type' in task &&
    task.type === 'local_agent'
  )
}
```

- 注释说明这是为了避免从 `teammateViewHelpers.ts → LocalAgentTask` 引入运行时循环依赖（通过 `BackgroundTasksDialog`）。
- 与 `src/tasks/LocalAgentTask/LocalAgentTask.tsx` 中导出的 `isLocalAgentTask` 逻辑一致，但内联于此以打破循环。

## 关键代码路径与文件引用

| 代码路径 | 说明 |
|----------|------|
| `enterTeammateView` (L46) | 被 `src/components/tasks/BackgroundTasksDialog.tsx`（`f` 键/Foreground）、`src/components/PromptInput/PromptInput.tsx`（ teammate 切换）、`src/hooks/useBackgroundTaskNavigation.ts`、`src/components/CoordinatorAgentStatus.tsx` 等引用。 |
| `exitTeammateView` (L88) | 被 `src/components/tasks/BackgroundTasksDialog.tsx`（leader 选择/返回）、`src/hooks/useTeammateViewAutoExit.ts`（自动退出）、`src/components/PromptInput/PromptInput.tsx`、`src/hooks/useCancelRequest.ts` 等引用。 |
| `stopOrDismissAgent` (L116) | 被 `src/components/tasks/BackgroundTasksDialog.tsx`（`x` 键）、`src/components/tasks/BackgroundTaskStatus.tsx`、`src/components/PromptInput/PromptInput.tsx`、`src/hooks/useCancelRequest.ts` 等引用。 |
| `release` (L28) | 内部辅助，被 `enterTeammateView` 与 `exitTeammateView` 共享。 |

## 依赖与外部交互

### 直接依赖

- `./AppState.js`（实际为 `./AppState.tsx` 的编译产物）→ `AppState` 类型
- `../services/analytics/index.js` → `logEvent`
- `../Task.js` → `isTerminalTaskStatus`
- `../tasks/LocalAgentTask/LocalAgentTask.js` → `LocalAgentTaskState` 类型（仅类型导入，值层面的 `isLocalAgentTask` 被内联避免循环）

### 调用方（上游）

- `src/components/tasks/BackgroundTasksDialog.tsx`： teammate / leader 切换、停止 teammate 的核心 UI。
- `src/hooks/useTeammateViewAutoExit.ts`：监听 teammate 被 kill / error / evict，自动调用 `exitTeammateView`。
- `src/components/PromptInput/PromptInput.tsx`：处理用户输入时的 teammate 切换逻辑。
- `src/hooks/useBackgroundTaskNavigation.ts`：键盘导航到 teammate 时进入视图。
- `src/hooks/useCancelRequest.ts`：取消请求时可能需要退出 teammate 视图。
- `src/components/CoordinatorAgentStatus.tsx`：协调器状态显示中的 teammate 交互。

## 风险、边界与改进建议

### 风险

1. **循环依赖的规避成本**  
   为了打破循环而内联 `isLocalAgent`，导致与 `LocalAgentTask.tsx` 中的官方类型守卫重复。若 `LocalAgentTaskState` 的判别条件未来改变（如增加 `type` 以外的必填字段），此处容易失步。

2. **`stopOrDismissAgent` 中的 mutate**  在 running 分支中直接调用 `task.abortController?.abort()` 是对 `prev.tasks[taskId]` 对象的直接修改，虽然随后返回原 `prev` 短路了 React 重渲染，但这违反了不可变约定。若未来有其他订阅者（非 React）依赖引用相等做 diff，可能观察不到 abort 的发生。

3. **`release` 丢失 `messages` 的不可恢复性**  一旦 `exitTeammateView` 调用 `release`，`messages` 被设为 `undefined`。虽然完整历史保存在磁盘或 mailbox 中，但 UI 上的 `task.messages` 被清空后，若用户快速再次进入同一 teammate，会经历一次 `diskLoaded: false` 的重新加载，可能产生闪烁或性能抖动。

4. **`PANEL_GRACE_MS` 的硬编码同步风险**  注释明确说明该值必须与 `framework.ts` 中的 `PANEL_GRACE_MS` 保持一致。若框架侧调整而此处未同步，会导致 terminal 任务的 lingering 时间不一致。

### 边界

- 这些辅助函数只操作 `AppState` 中 `viewingAgentTaskId`、`viewSelectionMode`、`tasks` 三个字段，不触碰 mailbox 或实际子进程。
- `enterTeammateView` 和 `exitTeammateView` 的 analytics 事件是 fire-and-forget，不阻塞状态更新。
- `stopOrDismissAgent` 对非 `local_agent` 类型（如 `in_process_teammate`）的 running 任务不会执行 abort；abort 逻辑由各自任务模块（如 `InProcessTeammateTask.kill`）处理。这导致 `x` 键在 teammate 上的实际行为由 `BackgroundTasksDialog.tsx` 路由到 `InProcessTeammateTask.kill`，而非本函数。

### 改进建议

1. **统一类型守卫来源**  评估是否可将 `isLocalAgentTask` 提取到一个无循环依赖的共享模块（如 `src/tasks/guards.ts`），消除内联重复。

2. **将 abort 包装为不可变更新**  在 `stopOrDismissAgent` 的 running 分支中，即使不修改 AppState，也应避免 mutate `task` 对象。可改为：
   ```ts
   task.abortController?.abort()
   // 返回新对象以遵守不可变约定（虽然字段值相同）
   return {
     ...prev,
     tasks: { ...prev.tasks, [taskId]: { ...task } }
   }
   ```
   或更简洁地，将 abort 逻辑上提到调用方（如 `BackgroundTasksDialog` 在调用 `stopOrDismissAgent` 前先判断类型并调用 `LocalAgentTask.kill`）。

3. **增加 `release` 行为的单元测试**  当前未找到对应测试文件。建议覆盖：
   - 进入 teammate A 再进入 teammate B 时，A 被正确 release
   - exit 后 `retain=false`、`messages=undefined`、`evictAfter` 正确设置
   - `stopOrDismissAgent` 对 running / terminal / already-dismissed 的幂等行为

4. **考虑将 `retain` 语义重命名为更直观的名称**  `retain` 对新人来说含义不够自解释（它表示“UI 正在持有该任务，阻止 eviction 并启用流式追加”）。可考虑改名为 `isHeldByUI` 或 `uiRetained`，并在 `LocalAgentTaskState` 的 JSDoc 中补充说明。
