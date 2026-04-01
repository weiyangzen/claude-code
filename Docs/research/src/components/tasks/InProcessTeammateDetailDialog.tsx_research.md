# InProcessTeammateDetailDialog.tsx 研究文档

## 场景与职责

`InProcessTeammateDetailDialog.tsx` 是 Claude Code 中用于展示 **in-process teammate（运行中队友代理）** 后台任务详情的专用对话框组件。 teammate 是 Agent Swarm / Team 功能中的并发子代理，由主会话（leader）派生并在后台独立运行。该组件的核心职责包括：

1. **展示 teammate 的运行时状态**：包括身份标识（带颜色）、当前活动描述、已耗时、token 消耗、tool 使用次数等。
2. **展示 teammate 的进度与上下文**：渲染最近的 tool activities、用户下发的 Prompt，以及失败时的错误信息。
3. **提供用户控制入口**：支持关闭对话框、返回列表、终止 teammate，以及将运行中的 teammate **切换到前台（foreground）** 进行交互。

该组件与 `DreamDetailDialog`、`ShellDetailDialog`、`AsyncAgentDetailDialog` 等并列，共同构成 `BackgroundTasksDialog` 中的详情视图矩阵。

---

## 功能点目的

### 1. 详情展示
- **标题（title）**：显示队友的标识名（`@agentName`），并应用该 teammate 的主题颜色（通过 `toInkColor` 映射）。如果存在活动描述（activity），则在名字后括号显示。
- **副标题（subtitle）**：
  - 若任务已结束，显示状态文本（`Completed` / `Failed` / `Stopped`）并着色。
  - 显示已耗时（`elapsedTime`）。
  - 条件显示 token 数量（如 `1.2k tokens`）和 tool 使用次数（如 `5 tools`）。
- **Progress 区域**：当 teammate 处于 `running` 状态且 `progress.recentActivities` 非空时，列出最近的活动记录。最新一条前缀 `›`，其余前缀两个空格；旧条目以 `dimColor` 淡化显示。每条活动通过 `renderToolActivity` 解析为可读的工具调用描述。
- **Prompt 区域**：显示触发该 teammate 的原始 prompt，经过 `truncateToWidth(..., 300)` 截断以适应终端宽度。
- **Error 区域**：当 `status === "failed"` 且存在 `error` 时，以红色展示错误文本。

### 2. 键盘交互
- **Space / Enter / Esc**：关闭详情对话框（触发 `onDone`）。
- **← (Left Arrow)**：返回后台任务列表（触发 `onBack`，如果提供）。
- **x**：终止正在运行中的 teammate（触发 `onKill`，仅在 `status === "running"` 且提供了 `onKill` 时可用）。
- **f**：将正在运行中的 teammate **切换到前台**（触发 `onForeground`，仅在 `status === "running"` 且提供了 `onForeground` 时可用）。这是 teammate 独有的交互能力。

### 3. 键绑定注册
- 通过 `useKeybindings` 注册 `"confirm:yes"` 到 `onDone`，上下文为 `Confirmation`。
- 通过原生 `onKeyDown` 处理 `left`、`x`、`f` 等组件特定快捷键。

---

## 具体技术实现

### 关键流程

#### 渲染流程
1. **获取主题与工具列表**：
   - `useTheme()` 获取当前主题。
   - `getTools(getEmptyToolPermissionContext())` 获取全部可用工具（用于 `renderToolActivity` 解析工具名）。该调用被 memo 缓存（React Compiler 的 `$[0]` 哨兵模式）。
2. **计算已耗时**：`useElapsedTime(teammate.startTime, teammate.status === "running", 1000, teammate.totalPausedMs ?? 0)`，支持扣除暂停时间。
3. **解析活动描述**：`describeTeammateActivity(teammate)` 根据 `shutdownRequested`、`awaitingPlanApproval`、`isIdle` 等标志返回人类可读的活动字符串。
4. **准备显示数据**：
   - `tokenCount`：优先取 `result.totalTokens`，否则取 `progress.tokenCount`。
   - `toolUseCount`：优先取 `result.totalToolUseCount`，否则取 `progress.toolUseCount`。
   - `displayPrompt`：对 `teammate.prompt` 做 `truncateToWidth(..., 300)` 截断。
5. **构建 Dialog**：传入动态生成的 `title`、`subtitle`、`inputGuide` 和 `onCancel`。
6. **渲染内容**：Progress（条件渲染）→ Prompt（始终渲染）→ Error（条件渲染）。

#### 键盘事件处理流程
```
onKeyDown(e)
  ├─ e.key === " "      → preventDefault() → onDone()
  ├─ e.key === "left"   → preventDefault() → onBack() (if provided)
  ├─ e.key === "x"      → preventDefault() → onKill() (if running & provided)
  └─ e.key === "f"      → preventDefault() → onForeground() (if running & provided)
```

### 数据结构

#### Props
```typescript
type Props = {
  teammate: DeepImmutable<InProcessTeammateTaskState>
  onDone: () => void
  onKill?: () => void
  onBack?: () => void
  onForeground?: () => void
}
```

#### InProcessTeammateTaskState（来自 `src/tasks/InProcessTeammateTask/types.ts`）
```typescript
type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController
  currentWorkAbortController?: AbortController
  unregisterCleanup?: () => void
  awaitingPlanApproval: boolean
  permissionMode: PermissionMode
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  messages?: Message[]
  inProgressToolUseIDs?: Set<string>
  pendingUserMessages: string[]
  spinnerVerb?: string
  pastTenseVerb?: string
  isIdle: boolean
  shutdownRequested: boolean
  onIdleCallbacks?: Array<() => void>
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

#### TeammateIdentity
```typescript
type TeammateIdentity = {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
}
```

### 协议/命令
- 无直接网络协议调用。
- `getTools()` 调用会组装本地工具列表（含 MCP 工具过滤），属于本地同步/异步计算。
- `renderToolActivity` 会尝试 `safeParse` 工具输入并调用工具的 `userFacingName` 和 `renderToolUseMessage` 方法。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/InProcessTeammateDetailDialog.tsx` | **本组件**。 teammate 详情对话框实现。 |
| `src/tasks/InProcessTeammateTask/types.ts` | `InProcessTeammateTaskState`、`TeammateIdentity`、`appendCappedMessage` 等类型与工具函数。 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | teammate 任务的注册、kill、状态更新逻辑。 |
| `src/components/tasks/renderToolActivity.tsx` | 将 `ToolActivity` 渲染为人类可读的工具调用描述（查找工具、解析输入、调用 `renderToolUseMessage`）。 |
| `src/components/tasks/taskStatusUtils.tsx` | 提供 `describeTeammateActivity`，根据状态标志返回活动字符串。 |
| `src/hooks/useElapsedTime.ts` | 格式化已耗时的 Hook。 |
| `src/keybindings/useKeybinding.ts` | 提供 `useKeybindings`。 |
| `src/components/design-system/Dialog.tsx` | 通用对话框壳组件。 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 快捷键提示组件。 |
| `src/components/design-system/Byline.tsx` | middot 分隔的提示行组件。 |
| `src/utils/format.ts` | 提供 `formatNumber`、`truncateToWidth`。 |
| `src/utils/ink.ts` | 提供 `toInkColor`，将 agent 颜色映射为 Ink 主题色。 |
| `src/tools.ts` | 提供 `getTools`，组装完整工具列表。 |
| `src/Tool.ts` | 提供 `getEmptyToolPermissionContext`、`Tool` 类型、`findToolByName`。 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 提供 `AgentProgress`、`ToolActivity` 类型。 |
| `src/ink.ts` | 导出 `Box`、`Text`、`useTheme`。 |

### 调用方
- **`src/components/tasks/BackgroundTasksDialog.tsx`**（第 382-388 行）：在 detail 视图中，当 `task.type === 'in_process_teammate'` 时渲染本组件，并传入 `onKill`（调用 `InProcessTeammateTask.kill`）和 `onForeground`（调用 `enterTeammateView`）。

---

## 依赖与外部交互

### 直接依赖模块
1. **React / Ink**：终端 UI 基础组件（`Box`、`Text`、`useTheme`）。
2. **Teammate 类型与状态模块**：定义了 `InProcessTeammateTaskState` 的完整形状。
3. **工具系统**：`getTools` + `getEmptyToolPermissionContext` 用于获取工具列表，供 `renderToolActivity` 解析最近活动。
4. **设计系统组件**：`Dialog`、`Byline`、`KeyboardShortcutHint`。
5. **Hooks**：`useElapsedTime`、`useKeybindings`。
6. **格式化与颜色工具**：`formatNumber`、`truncateToWidth`、`toInkColor`。

### 外部交互
- **AppState（间接）**：通过 `BackgroundTasksDialog` 传入的 `teammate` 对象读取状态；通过 `onKill` / `onForeground` 回调间接修改全局状态。
- **Kill 流程（间接）**：`onKill` 最终调用 `InProcessTeammateTask.kill`，会 abort `abortController` 和 `currentWorkAbortController`，并清理注册表。
- **Foreground 流程（间接）**：`onForeground` 调用 `enterTeammateView(teammateId, setAppState)`，将当前 teammate 设为用户主视图中的前台代理，允许用户与其 transcript 直接交互。

---

## 风险、边界与改进建议

### 风险
1. **工具列表的静态获取**：组件在渲染时调用 `getTools(getEmptyToolPermissionContext())`，虽然被 React Compiler memo 缓存，但它使用的是**空权限上下文**（`mode: 'default'`，无额外工作目录、无规则）。如果某些工具的 `userFacingName` 或 `renderToolUseMessage` 依赖权限上下文做决策，可能产生不准确的显示结果。
2. **`renderToolActivity` 的异常吞没**：`renderToolActivity` 内部用 `try/catch` 包裹工具解析，失败时回退到原始 `toolName`。虽然避免了崩溃，但用户可能看到不友好的原始工具名。
3. **Prompt 截断的硬编码宽度**：`truncateToWidth(teammate.prompt, 300)` 中的 `300` 是字符宽度的硬编码，未考虑终端实际宽度。在窄终端中仍可能溢出，在宽终端中则浪费了可显示空间。
4. **编译后代码维护成本**：与 `DreamDetailDialog` 类似，该文件已被 React Compiler 编译，大量 `$[n]` memo cache 逻辑使源码难以直接阅读和调试。

### 边界情况
1. **无 progress 数据**：若 `teammate.progress` 为 undefined 或 `recentActivities` 为空，Progress 区域完全不渲染。
2. **无 token/tool 数据**：`tokenCount` 和 `toolUseCount` 可能为 `undefined`，subtitle 中对应的统计项会被条件省略。
3. **已结束任务**：`status !== "running"` 时，`onKill` 和 `onForeground` 均不会被传入，UI 中也不会显示 `x` 和 `f` 的快捷键提示。
4. **颜色缺失**：若 `teammate.identity.color` 未定义，`toInkColor` 会回退到默认的 `cyan_FOR_SUBAGENTS_ONLY`。

### 改进建议
1. **动态获取权限上下文**：如果可行，将调用方 `BackgroundTasksDialog` 的 `toolUseContext` 中的权限上下文透传给 `InProcessTeammateDetailDialog`，再传给 `getTools`，以确保工具解析的准确性。
2. **根据终端宽度动态截断 Prompt**：使用 `useTerminalSize()` 获取当前终端列数，计算 Prompt 的最大显示宽度（如 `columns - 10`），替代硬编码的 `300`。
3. **增加 `renderToolActivity` 失败时的降级提示**：在回退到原始 `toolName` 时，可添加一个 `dimColor` 的 `?` 或 `"(unknown)"` 后缀，提示用户该活动未能被完整解析。
4. **Progress 区域的可展开性**：当前只显示 `recentActivities`（最多 5 条，由 `LocalAgentTask` 的 `MAX_RECENT_ACTIVITIES` 限制）。考虑在详情页中提供查看完整 transcript 的入口（如按 `t` 键跳转），而不仅限于最近活动摘要。
5. **Source map 与源码可读性**：建议在仓库中保留未编译的 TSX 源文件作为 primary source，或将编译产物与源码分离，降低后续维护成本。
