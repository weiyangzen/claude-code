# TaskListV2.tsx 研究文档

## 场景与职责

`TaskListV2.tsx` 是 Claude Code REPL 界面中 **任务列表（Todo V2）** 的核心展示组件。它在主会话区域渲染当前任务列表，显示每个任务的状态（已完成/进行中/待处理）、主题、负责人、阻塞关系以及负责人的实时活动。该组件支持两种展示模式：

- **内嵌模式**（`isStandalone = false`）：直接嵌入 REPL 的消息流中，仅显示任务项。
- **独立模式**（`isStandalone = true`）：在任务展开视图（`expandedView === 'tasks'`）中显示，额外在顶部渲染任务统计摘要（总数/已完成/进行中/待处理）。

## 功能点目的

1. **任务状态可视化**：用不同图标和颜色区分 `completed`（✓ 绿色）、`in_progress`（■ 橙色）、`pending`（□ 默认色）。
2. **负责人与活动关联**：在 Agent Swarms 启用时，显示任务负责人的颜色（来自 teammate color）和当前活动描述，帮助用户追踪谁在做什么。
3. **阻塞关系展示**：若任务被其他未完成任务阻塞，显示 `blocked by #id` 提示。
4. **动态截断与终端适配**：根据终端列数 `columns` 截断任务主题和活动描述，避免换行破坏布局。
5. **最近完成任务的保留显示**：任务完成后 30 秒内仍视为 "recent completed"，在列表截断时优先保留，给用户一个视觉反馈窗口。
6. **列表截断与优先级排序**：当任务数超过根据终端高度计算出的 `maxDisplay` 时，按优先级显示：最近完成 > 进行中 > 待处理（未阻塞优先）>  older 完成，并在底部汇总隐藏任务的数量与状态。

## 具体技术实现

### 关键流程

#### 1. 最近完成时间戳追踪
- 使用 `completionTimestampsRef`（`Map<string, number>`）记录每个任务首次变为 `completed` 的时间。
- 使用 `previousCompletedIdsRef` 对比前后两次 `tasks` 的变化，只在任务**首次**进入 completed 时写入时间戳。
- `RECENT_COMPLETED_TTL_MS = 30_000`。
- `useEffect` 监听 `tasks` 变化，计算最早过期时间并设置 `setTimeout` 触发 `forceUpdate`，确保 30 秒后自动将任务从 "recent completed" 降级。

#### 2. 任务截断与排序
- `maxDisplay` 根据终端 `rows` 动态计算：
  - `rows <= 10` 时显示 0 个（避免小终端过度拥挤）。
  - 否则 `Math.min(10, Math.max(3, rows - 14))`。
- 截断时：
  1. 将 completed 任务按时间戳分为 `recentCompleted`（< 30s）和 `olderCompleted`。
  2. `inProgress` 任务单独提取。
  3. `pending` 任务按是否被阻塞排序（未阻塞在前）。
  4. 合并为 `[...recentCompleted, ...inProgress, ...pending, ...olderCompleted]`，取前 `maxDisplay` 个。
- 隐藏任务在底部以 `… +N in progress, M pending, K completed` 形式汇总。

#### 3. 负责人颜色与活动映射
- `teammateColors`：遍历 `teamContext.teammates`，将 teammate 名字映射到 `AGENT_COLOR_TO_THEME_COLOR` 中的主题色键。
- `teammateActivity`：遍历 `appStateTasks`，对状态为 `running` 的 `in_process_teammate` 任务，使用 `summarizeRecentActivities` 汇总其最近活动，回退到 `lastActivity.activityDescription`。
- `activeTeammates`：记录当前仍在运行的 teammate，用于控制是否显示负责人后缀（`columns >= 60` 且 `ownerActive` 为 true 时才显示）。

#### 4. TaskItem 渲染
- `getTaskIcon(status)` 返回对应图标与主题色。
- 主题截断：`maxSubjectWidth = Math.max(15, columns - 15 - ownerWidth)`，使用 `truncateToWidth`。
- 活动截断：`maxActivityWidth = Math.max(15, columns - 15)`。
- 样式：
  - `completed` → `strikethrough` + `dimColor`
  - `in_progress` → `bold`
  - `blocked` → `dimColor`
  - 负责人显示为 `(@name)`，若存在 `ownerColor` 则用 `ThemedText` 着色。

### 数据结构

- `Props`：
  - `tasks: Task[]` — 要展示的任务数组。
  - `isStandalone?: boolean` — 是否显示顶部统计摘要。
- `TaskItemProps`：
  - `task`, `ownerColor?`, `openBlockers`, `activity?`, `ownerActive`, `columns`。
- `Task`（来自 `src/utils/tasks.ts`）：
  - `id`, `subject`, `description`, `activeForm?`, `owner?`, `status`, `blocks`, `blockedBy`, `metadata?`。

### 协议/命令

- 无网络协议或子进程命令。
- 纯展示组件，任务数据的变更由外部通过 `tasks` prop 传入（通常来自 `useAppState(s => s.tasks)` 与 `useTasksV2WithCollapseEffect` 的聚合）。

## 关键代码路径与文件引用

- **本文件**：`src/components/TaskListV2.tsx`
- **调用方**：
  - `src/screens/REPL.tsx` — 主 REPL 屏幕，在合适条件下渲染 `<TaskListV2 tasks={tasksV2} />` 或独立版本。
  - `src/components/Spinner.tsx` — 可能引用（grep 命中，需结合具体使用场景确认）。
- **依赖类型与工具**：
  - `src/utils/tasks.ts` — `Task` 类型、`isTodoV2Enabled()`。
  - `src/tasks/InProcessTeammateTask/types.ts` — `isInProcessTeammateTask`、`InProcessTeammateTaskState`。
  - `src/tools/AgentTool/agentColorManager.ts` — `AGENT_COLOR_TO_THEME_COLOR`、`AgentColorName`。
  - `src/utils/agentSwarmsEnabled.ts` — `isAgentSwarmsEnabled()`（运行时开关）。
  - `src/utils/array.ts` — `count()` 辅助函数。
  - `src/utils/collapseReadSearch.ts` — `summarizeRecentActivities()` 汇总 teammate 最近活动。
  - `src/utils/theme.ts` — `Theme` 类型。
  - `src/components/design-system/ThemedText.tsx` — 主题感知的文本组件。
  - `src/hooks/useTerminalSize.ts` — 获取终端尺寸。
  - `src/state/AppState.js` — `useAppState`。
  - `src/ink/stringWidth.ts` / `src/utils/format.ts` — 宽度计算与截断。
  - `figures` — 终端图标库。

## 依赖与外部交互

| 依赖 | 路径 | 用途 |
|------|------|------|
| `useTerminalSize` | `src/hooks/useTerminalSize.ts` | 获取终端行列数 |
| `useAppState` | `src/state/AppState.js` | 读取全局状态（teamContext、tasks） |
| `stringWidth` | `src/ink/stringWidth.ts` | 计算负责人后缀宽度 |
| `truncateToWidth` | `src/utils/format.ts` | 截断主题与活动描述 |
| `summarizeRecentActivities` | `src/utils/collapseReadSearch.ts` | 汇总 teammate 最近活动为可读字符串 |
| `AGENT_COLOR_TO_THEME_COLOR` | `src/tools/AgentTool/agentColorManager.ts` | 将 agent 颜色名映射到主题色键 |
| `isAgentSwarmsEnabled` | `src/utils/agentSwarmsEnabled.ts` | 控制 teammate 相关功能是否显示 |
| `isTodoV2Enabled` | `src/utils/tasks.ts` | 控制任务列表是否渲染 |
| `ThemedText` | `src/components/design-system/ThemedText.tsx` | 主题色文本渲染 |
| `figures` | npm | 终端图标（tick、squareSmallFilled 等） |

## 风险、边界与改进建议

### 风险与边界

1. **React Compiler 生成代码的可读性与调试难度**：文件已被 React Compiler 编译，出现 `_c(37)` 等缓存数组操作。人工阅读和维护成本高，调试时难以追踪渲染路径。
2. **定时器与闭包耦合**：`completionTimestampsRef` + `useEffect` + `forceUpdate` 的组合虽然功能正确，但在任务频繁更新时可能产生大量 `setTimeout` 注册/清理操作，存在微性能开销。
3. **`maxDisplay` 的硬编码阈值**：`rows - 14` 和 `maxDisplay > 0` 的魔法数字与 REPL 其他组件（如输入框、header）高度耦合，若 REPL 布局调整，此处容易遗漏同步更新。
4. **负责人匹配逻辑**：`teammateActivity` 同时用 `agentName` 和 `agentId` 做键，虽然兼容了不同格式，但也可能导致同一 teammate 的活动被重复存储或覆盖。
5. **无测试覆盖**：glob 搜索未找到针对 `TaskListV2` 的测试文件，截断、排序、时间戳逻辑均缺乏自动化验证。

### 改进建议

1. **提取纯逻辑到独立模块**：将任务排序/截断逻辑（`byIdAsc`、优先级分组、`maxDisplay` 计算）提取到 `src/utils/taskListDisplay.ts` 等纯函数模块中，便于测试和复用。
2. **统一布局常量**：将 `rows - 14` 等魔法数字与 REPL 布局常量表合并，或至少添加注释说明其计算依据（如 header 高度、输入框高度、margin 等）。
3. **优化时间戳管理**：考虑使用 `useMemo` 或自定义 hook（如 `useRecentCompletedTasks(tasks, ttl)`）封装时间戳逻辑，减少主组件中的副作用密度。
4. **增加单元测试**：
   - 测试 `byIdAsc` 对数字 ID 和字符串 ID 的排序。
   - 测试优先级排序（recent → in_progress → pending → older）。
   - 测试截断宽度计算（含/不含负责人后缀）。
   - 测试 `summarizeRecentActivities` 的集成行为。
5. **考虑虚拟滚动**：当任务数极大（如 100+）时，即使截断到 10 条也可能不够；可评估是否需要与 `VirtualMessageList` 类似的虚拟滚动方案。
