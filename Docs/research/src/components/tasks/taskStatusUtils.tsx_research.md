# taskStatusUtils.tsx 研究文档

## 场景与职责

`taskStatusUtils.tsx` 是 Claude Code 任务子系统中负责**状态展示语义映射**的共享工具库。它不渲染任何 UI，而是提供一组纯函数，将抽象的 `TaskStatus` 和任务状态标志转换为具体的**图标（icon）**、**颜色（color）**、**活动描述（activity description）**以及**布局可见性决策（footer 隐藏逻辑）**。

该文件被多个 UI 组件和任务管理模块广泛引用：
- `src/components/tasks/AsyncAgentDetailDialog.tsx`：`getTaskStatusColor`, `getTaskStatusIcon`
- `src/components/tasks/BackgroundTask.tsx`：`describeTeammateActivity`
- `src/components/tasks/BackgroundTaskStatus.tsx`：`shouldHideTasksFooter`
- `src/components/CoordinatorAgentStatus.tsx`：`getTaskStatusIcon`, `getTaskStatusColor`
- `src/components/PromptInput/PromptInputFooterLeftSide.tsx`：`getTaskStatusIcon`
- `src/components/tasks/InProcessTeammateDetailDialog.tsx`：`describeTeammateActivity`

---

## 功能点目的

### 1. `isTerminalStatus(status: TaskStatus): boolean`
判断任务是否已进入终态（`completed` | `failed` | `killed`）。
- 用于阻止向已终止的 teammate 注入消息、触发任务清理、控制动画停止等。

### 2. `getTaskStatusIcon(status, options?)`
根据状态及附加标志返回对应的 `figures` 图标字符：

| 条件 | 图标 |
|------|------|
| `hasError === true` | `✖` (cross) |
| `awaitingApproval === true` | `?` (questionMarkPrefix) |
| `shutdownRequested === true` | `⚠` (warning) |
| `status === "running" && isIdle` | `…` (ellipsis) |
| `status === "running"` | `▶` (play) |
| `status === "completed"` | `✔` (tick) |
| `status === "failed" \| "killed"` | `✖` (cross) |
| 默认 | `•` (bullet) |

### 3. `getTaskStatusColor(status, options?)`
返回 Ink 语义颜色名称（`'success' | 'error' | 'warning' | 'background'`）：

| 条件 | 颜色 |
|------|------|
| `hasError === true` | `error` |
| `awaitingApproval === true` | `warning` |
| `shutdownRequested === true` | `warning` |
| `isIdle === true` | `background` |
| `status === "completed"` | `success` |
| `status === "failed"` | `error` |
| `status === "killed"` | `warning` |
| 默认 | `background` |

### 4. `describeTeammateActivity(t)`
为 `InProcessTeammateTaskState` 生成人类可读的活动描述字符串，优先级如下：
1. `t.shutdownRequested` → `"stopping"`
2. `t.awaitingPlanApproval` → `"awaiting approval"`
3. `t.isIdle` → `"idle"`
4. `summarizeRecentActivities(t.progress.recentActivities)`（来自 `collapseReadSearch.ts`）
5. `t.progress.lastActivity.activityDescription`
6. 最终回退 → `"working"`

### 5. `shouldHideTasksFooter(tasks, showSpinnerTree): boolean`
决定背景任务 footer（底部摘要 pill）是否应该隐藏。逻辑：
- 若 `showSpinnerTree === false`，直接返回 `false`（不隐藏）。
- 遍历 `tasks` 中所有 `isBackgroundTask(t)` 且**不是** panel-managed agent 任务（`isPanelAgentTask`）的可见任务。
- 若存在任意一个可见任务且其类型**不是** `in_process_teammate`，返回 `false`。
- 若所有可见任务都是 `in_process_teammate` 且至少有一个可见任务，返回 `true`。

**设计意图**：当 spinner tree（队友树）展开时，in-process teammate 已经在树中展示，footer 中的重复 pill 应该被隐藏；但如果有其他类型的背景任务（如 bash、remote agent），footer 仍需保留以展示它们。

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 类型依赖
```ts
import type { TaskStatus } from 'src/Task.js';
import type { InProcessTeammateTaskState } from 'src/tasks/InProcessTeammateTask/types.js';
import { isPanelAgentTask } from 'src/tasks/LocalAgentTask/LocalAgentTask.js';
import { isBackgroundTask, type TaskState } from 'src/tasks/types.js';
import type { DeepImmutable } from 'src/types/utils.js';
import { summarizeRecentActivities } from 'src/utils/collapseReadSearch.js';
```

### `figures` 图标库
使用 npm 包 `figures` 提供跨平台 Unicode 图标：
- `figures.tick` → `✔`
- `figures.cross` → `✖`
- `figures.play` → `▶`
- `figures.ellipsis` → `…`
- `figures.warning` → `⚠`
- `figures.bullet` → `•`
- `figures.questionMarkPrefix` → `?`

### `summarizeRecentActivities` 的调用
`describeTeammateActivity` 的第 4 优先级调用了 `summarizeRecentActivities`，该函数来自 `src/utils/collapseReadSearch.ts`。它会将最近 5 个工具活动折叠成一句摘要，例如：
- `"Read 3 files, searched for 2 patterns"`
- `"Recalled 2 memories"`

### `shouldHideTasksFooter` 中的硬编码字符串
```ts
if (!isBackgroundTask(t) || "external" === 'ant' && isPanelAgentTask(t)) {
  continue;
}
```
注意 `"external" === 'ant'` 永远为 `false`。这段代码看起来是编译器优化或条件编译的残留痕迹（可能是为了在不同构建目标中生成不同分支）。实际运行时，条件简化为 `!isBackgroundTask(t)`。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/taskStatusUtils.tsx` | 本文件 |
| `src/Task.ts` | `TaskStatus` 类型定义 |
| `src/tasks/types.ts` | `TaskState`、`BackgroundTaskState`、`isBackgroundTask` |
| `src/tasks/InProcessTeammateTask/types.ts` | `InProcessTeammateTaskState` |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `isPanelAgentTask` |
| `src/utils/collapseReadSearch.ts` | `summarizeRecentActivities` |
| `src/components/tasks/AsyncAgentDetailDialog.tsx` | 调用 `getTaskStatusColor`, `getTaskStatusIcon` |
| `src/components/tasks/BackgroundTaskStatus.tsx` | 调用 `shouldHideTasksFooter` |
| `src/components/tasks/BackgroundTask.tsx` | 调用 `describeTeammateActivity` |
| `src/components/CoordinatorAgentStatus.tsx` | 调用 `getTaskStatusIcon`, `getTaskStatusColor` |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 调用 `getTaskStatusIcon` |
| `src/components/tasks/InProcessTeammateDetailDialog.tsx` | 调用 `describeTeammateActivity` |

---

## 依赖与外部交互

### 运行时依赖
- **`figures`**：跨平台 Unicode 图标库。
- **`collapseReadSearch.ts`**：提供 `summarizeRecentActivities`，将工具活动列表折叠为自然语言摘要。
- **任务类型系统**：依赖 `isBackgroundTask` 和 `isPanelAgentTask` 进行任务过滤。

### 数据流
1. `AppState.tasks` 中存储了所有任务的 `TaskState`。
2. `BackgroundTaskStatus.tsx` 读取任务列表，调用 `shouldHideTasksFooter` 决定是否渲染 footer pill。
3. `BackgroundTask.tsx` 为每个 teammate 任务调用 `describeTeammateActivity` 生成活动文本。
4. `AsyncAgentDetailDialog.tsx` 在渲染代理状态时调用 `getTaskStatusColor` 和 `getTaskStatusIcon`。

---

## 风险、边界与改进建议

### 风险
1. **`"external" === 'ant'` 死代码**：`shouldHideTasksFooter` 中存在永远为 `false` 的条件表达式。虽然不影响运行时行为，但会增加代码阅读成本，并可能让静态分析工具产生误报。
2. **图标与颜色逻辑重复**：`getTaskStatusIcon` 和 `getTaskStatusColor` 的优先级判断逻辑高度相似（`hasError` > `awaitingApproval` > `shutdownRequested` > `isIdle` > `status`）。如果产品需求调整优先级，需要同时修改两个函数，容易遗漏。
3. **`isTerminalStatus` 与 `isTerminalTaskStatus` 重复**：`src/Task.ts` 中已经定义了 `isTerminalTaskStatus(status)`，功能完全相同。`taskStatusUtils.tsx` 中的 `isTerminalStatus` 是冗余的，可能导致维护者困惑该用哪一个。
4. **`describeTeammateActivity` 的 fallback 链较长**：从 `shutdownRequested` 到 `working` 有 6 层回退，虽然逻辑清晰，但调试时难以快速判断最终输出来自哪一层。

### 边界
- `getTaskStatusIcon` 和 `getTaskStatusColor` 的 `options` 参数是可选的，不传时所有标志视为 `undefined`。
- `describeTeammateActivity` 要求 `t.progress` 至少存在才能访问 `recentActivities` 和 `lastActivity`，但代码通过可选链 `?.` 安全处理缺失情况。
- `shouldHideTasksFooter` 只关心**可见的**背景任务。如果所有背景任务都是 `in_process_teammate` 但没有任何一个通过 `isBackgroundTask` 过滤（例如全部已完成），函数返回 `false`（因为 `hasVisibleTask` 为 `false`）。

### 改进建议
1. **移除或合并重复的 `isTerminalStatus`**：建议删除 `taskStatusUtils.tsx` 中的 `isTerminalStatus`，统一使用 `src/Task.ts` 中的 `isTerminalTaskStatus`，并在本文件中重新导出以兼容现有调用方。
2. **提取共享的优先级决策函数**：将 `getTaskStatusIcon` 和 `getTaskStatusColor` 中的条件优先级提取为一个内部辅助函数 `resolveStatusPriority(status, options)`，返回一个标准化的优先级结果对象，然后 `getTaskStatusIcon` 和 `getTaskStatusColor` 分别映射为图标和颜色。
3. **清理死代码**：将 `"external" === 'ant'` 替换为更明确的条件编译注释或构建时宏，避免运行时保留无意义表达式。
4. **为 `describeTeammateActivity` 增加调试模式**：在开发构建中，可以返回一个带来源标记的字符串（如 `"working [fallback]"`），帮助开发者追踪哪一层 fallback 被触发。
5. **单元测试覆盖**：建议对 `shouldHideTasksFooter` 的各种任务组合（空任务、纯 teammate、混合任务、无 background 任务）增加单元测试，因为该函数直接影响底部 footer 的显隐，是 UI 布局的关键决策点。
