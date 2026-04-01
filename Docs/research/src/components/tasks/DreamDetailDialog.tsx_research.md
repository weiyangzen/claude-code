# DreamDetailDialog.tsx 研究文档

## 场景与职责

`DreamDetailDialog.tsx` 是 Claude Code 中用于展示 **auto-dream（记忆整合子代理）** 后台任务详情的专用对话框组件。Dream 任务本身是一个在后台运行的 forked agent，负责会话记忆的整合与压缩。该组件的核心职责是：

1. **将不可见的后台代理可视化**：通过任务注册表（task registry）把原本无 UI 的 dream agent 呈现在 footer pill 和 `Shift+Down` 后台任务对话框中。
2. **提供任务详情查看**：展示 dream 任务的运行状态、已耗时、正在回顾的会话数、已触碰的文件数以及最近的 assistant turns。
3. **支持用户交互**：允许用户通过键盘快捷键关闭对话框、返回列表视图，或在任务运行中时终止（kill）任务。

该组件属于 `src/components/tasks/` 目录下的后台任务详情视图家族，与 `ShellDetailDialog`、`AsyncAgentDetailDialog`、`InProcessTeammateDetailDialog` 等并列。

---

## 功能点目的

### 1. 详情展示
- **标题**：固定显示为 `"Memory consolidation"`，明确告知用户这是记忆整合任务。
- **元信息栏（subtitle）**：显示格式化后的已耗时（`elapsedTime`）、正在回顾的会话数量（`sessionsReviewing`）、以及被触碰的文件数量（`filesTouched.length`）。
- **状态展示**：区分 `running`（高亮背景色）、`completed`（success 绿色）、`failed`/`killed`（error 红色）三种状态。
- **Turns 列表**：展示 dream agent 最近的 assistant turns。由于 turns 可能很多，组件只渲染最近 `VISIBLE_TURNS = 6` 条非空 turn，更早的 turns 折叠为 `"(N earlier turns)"` 的提示文本。每条 turn 还会附带该 turn 中的 tool use 数量。

### 2. 键盘交互
- **Space / Enter / Esc**：关闭详情对话框（触发 `onDone`）。
- **← (Left Arrow)**：返回后台任务列表（触发 `onBack`，如果提供）。
- **x**：终止正在运行中的 dream 任务（触发 `onKill`，仅在 `task.status === "running"` 且提供了 `onKill` 时可用）。

### 3. 键绑定注册
- 通过 `useKeybindings` 注册 `"confirm:yes"` 动作到 `onDone`，使用 `Confirmation` 上下文，确保与全局键绑定系统兼容。
- 通过原生 `onKeyDown` 处理方向键和 `x` 键，因为这些是组件特定的硬编码快捷方式，不通过可配置键绑定系统解析。

---

## 具体技术实现

### 关键流程

#### 渲染流程
1. **计算已耗时**：调用 `useElapsedTime(task.startTime, task.status === "running", 1000, 0)`，每秒更新一次格式化时长。
2. **过滤可见 turns**：`task.turns.filter(t => t.text !== "")` 去掉空文本 turn，然后 `slice(-VISIBLE_TURNS)` 取最近 6 条。
3. **构建 Dialog**：使用 `Dialog` 组件包裹内容，传入标题、subtitle、取消回调和自定义 `inputGuide`。
4. **渲染内容**：
   - 外层 `Box`（`flexDirection="column"`）
   - 状态行：`Status: <running|completed|failed>`
   - Turns 区域：若为空显示 `"Starting…"` 或 `"(no text output)"`；否则显示折叠提示 + 各 turn 的文本和 tool use 计数。

#### 键盘事件处理流程
```
onKeyDown(e)
  ├─ e.key === " "      → preventDefault() → onDone()
  ├─ e.key === "left"   → preventDefault() → onBack() (if provided)
  └─ e.key === "x"      → preventDefault() → onKill() (if running & provided)
```

### 数据结构

#### Props
```typescript
type Props = {
  task: DeepImmutable<DreamTaskState>
  onDone: () => void
  onBack?: () => void
  onKill?: () => void
}
```

#### DreamTaskState（来自 `src/tasks/DreamTask/DreamTask.ts`）
```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]
  abortController?: AbortController
  priorMtime: number
}
```

#### DreamTurn
```typescript
type DreamTurn = {
  text: string
  toolUseCount: number
}
```

### 协议/命令
- 无网络协议或外部命令调用。所有交互均为本地 React 事件和回调。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/DreamDetailDialog.tsx` | **本组件**。详情对话框实现。 |
| `src/tasks/DreamTask/DreamTask.ts` | Dream 任务的状态类型、注册/更新/完成/终止逻辑。 |
| `src/hooks/useElapsedTime.ts` | 提供格式化已耗时的 Hook，基于 `useSyncExternalStore` + `setInterval`。 |
| `src/keybindings/useKeybinding.ts` | 提供 `useKeybindings`，用于注册 `"confirm:yes"` 等可配置键绑定。 |
| `src/components/design-system/Dialog.tsx` | 通用对话框壳组件，处理标题、subtitle、边框、默认取消键绑定。 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 渲染键盘快捷键提示文本（如 `"Esc/Enter/Space to close"`）。 |
| `src/components/design-system/Byline.tsx` | 用 middot 分隔符连接多个提示项。 |
| `src/utils/stringUtils.ts` | 提供 `plural()` 辅助函数处理单复数。 |
| `src/ink.ts` | 导出 `Box`、`Text` 等 Ink 组件（终端 UI 渲染）。 |

### 调用方
- **`src/components/tasks/BackgroundTasksDialog.tsx`**（第 395-398 行）：后台任务列表在切换到 detail 视图时，根据 `task.type === 'dream'` 渲染 `<DreamDetailDialog>`。

---

## 依赖与外部交互

### 直接依赖模块
1. **React / Ink**：使用 `react` 和自定义 `ink.ts` 中的 `Box`、`Text` 进行终端 UI 渲染。
2. **DreamTask 模块**：导入 `DreamTaskState` 类型，用于 Props 类型约束。
3. **设计系统组件**：`Dialog`、`Byline`、`KeyboardShortcutHint` 提供统一的对话框和提示样式。
4. **Hooks**：`useElapsedTime` 负责时间显示；`useKeybindings` 负责可配置键绑定。
5. **工具函数**：`plural` 用于单复数文本生成。

### 外部交互
- **AppState（间接）**：通过 `BackgroundTasksDialog` 传入的 `task` 对象读取状态，自身不直接操作全局状态。
- **Kill 回调（间接）**：`onKill` 由 `BackgroundTasksDialog` 提供，内部调用 `DreamTask.kill(taskId, setAppState)`，会触发 `abortController.abort()` 并回滚 consolidation lock 的 mtime。

---

## 风险、边界与改进建议

### 风险
1. **Turn 数据丢失错觉**：`VISIBLE_TURNS = 6` 的硬编码限制意味着用户只能直接看到最近 6 条 turn，更早的仅显示为计数。对于长时间运行的 dream 任务，用户无法在此对话框中查看完整历史。
2. **filesTouched 的不完整性**：`DreamTaskState` 的注释明确指出 `filesTouched` 只是通过 `onMessage` 模式匹配到的 Edit/Write tool_use 路径，遗漏了 bash 介导的写入。UI 上未对此不确定性做任何免责声明，用户可能误以为列表是完整的。
3. **编译后代码可读性**：该文件已被 React Compiler（`react/compiler-runtime`）编译，原始 TSX 被转换为大量 memo cache（`$[n]`）操作，增加了调试和维护难度。

### 边界情况
1. **空 turns**：当 `shown.length === 0` 时，根据状态显示 `"Starting…"`（running）或 `"(no text output)"`（已结束）。
2. **无 onBack/onKill**：如果调用方未提供这些回调，对应的键盘快捷键和 UI 提示不会出现。
3. **任务已结束**：`useElapsedTime` 的 `isRunning` 参数为 `false`，计时器停止更新；`endTime` 未传入该 Hook，但 `task.startTime` 和 `task.endTime` 存在于 `TaskStateBase` 中，当前实现依赖 `Date.now()` 冻结在关闭时刻，若任务结束后长时间打开对话框，时长会继续增长（`useElapsedTime` 未接收 `endTime`）。

### 改进建议
1. **暴露 `endTime` 给 `useElapsedTime`**：将 `task.endTime` 传入 `useElapsedTime` 的 `endTime` 参数，确保已结束任务的时长在详情页中固定不变。
2. **增加 `filesTouched` 免责声明**：在 subtitle 或 turns 区域附近添加一行小字，提示 `"Files touched may be incomplete (tool-use only)"`，避免用户误解。
3. **可配置的 VISIBLE_TURNS 或展开能力**：考虑将 `VISIBLE_TURNS` 提升为 prop 或支持按键（如 `+`/`-`）调整显示数量，改善长任务的可观测性。
4. **Source map 与调试**：当前文件包含内联 source map，但编译后的主体代码极难阅读。建议在开发/调试流程中保留未编译的源文件映射，或提供更易读的源码入口。
