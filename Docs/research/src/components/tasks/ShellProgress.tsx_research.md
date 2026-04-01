# ShellProgress.tsx 研究文档

## 场景与职责

`ShellProgress.tsx` 是 Claude Code 终端 UI 中**最轻量级的任务状态展示组件之一**，专门负责将本地 Shell 任务（`LocalShellTaskState`）的运行状态转换为带语义颜色的简短文本标签。

该组件主要被以下文件消费：
- `src/components/tasks/BackgroundTask.tsx`（行 11 引用）：在背景任务列表的 pill 中展示 Shell 任务状态。
- `src/tools/AgentTool/AgentTool.tsx`、`src/tools/BashTool/UI.tsx`、`src/tools/PowerShellTool/UI.tsx` 等：在工具执行进度或结果展示中复用 `TaskStatusText`。
- `src/components/shell/ShellProgressMessage.tsx`、`src/components/BashModeProgress.tsx`：在 Shell 执行过程中的即时反馈 UI 中展示状态。

组件设计遵循**单一职责原则**：不做任何数据获取、不处理用户交互、仅做状态到文本/颜色的纯映射。

---

## 功能点目的

### 1. `TaskStatusText`
通用状态文本组件，接收：
```ts
type TaskStatusTextProps = {
  status: TaskStatus;
  label?: string;   // 覆盖默认显示的文本
  suffix?: string;  // 追加在标签后的后缀
};
```

颜色映射规则：
| status | color |
|--------|-------|
| `completed` | `success` |
| `failed` | `error` |
| `killed` | `warning` |
| `running` / `pending` | `undefined`（继承默认色） |

渲染结果示例：
- `<TaskStatusText status="completed" label="done" />` → `(done)` 绿色
- `<TaskStatusText status="failed" label="error" />` → `(error)` 红色
- `<TaskStatusText status="running" suffix=", unread" />` → `(running, unread)` 无色

### 2. `ShellProgress`
针对 `LocalShellTaskState` 的包装组件，基于 `shell.status` 做 switch 分支：
| status | 渲染结果 |
|--------|----------|
| `completed` | `<TaskStatusText status="completed" label="done" />` |
| `failed` | `<TaskStatusText status="failed" label="error" />` |
| `killed` | `<TaskStatusText status="killed" label="stopped" />` |
| `running` / `pending` | `<TaskStatusText status="running" />` |

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 类型依赖
```ts
import type { TaskStatus } from 'src/Task.js';
import type { LocalShellTaskState } from 'src/tasks/LocalShellTask/guards.js';
import type { DeepImmutable } from 'src/types/utils.js';
```

### 渲染实现
- 使用 Ink 的 `<Text color={color} dimColor>` 包裹显示文本。
- `dimColor` 始终为 `true`，使状态文本在视觉上比主内容更弱，避免抢夺注意力。
- 组件经过 React Compiler 编译，内部使用 `$` cache 数组对 props 进行 memoization。

### 为什么 `pending` 也映射到 `running`？
在 Shell 任务的语义中，`pending` 通常表示任务已注册但子进程尚未完全启动，或正在等待权限确认。从用户视角看，这仍然是"进行中"的状态，因此 UI 上统一显示为 `running`，不区分 `pending`。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/ShellProgress.tsx` | 本文件 |
| `src/components/tasks/BackgroundTask.tsx` | 调用 `ShellProgress` 渲染 local_bash 任务状态 |
| `src/tasks/LocalShellTask/guards.ts` | `LocalShellTaskState` 类型与 `isLocalShellTask` guard |
| `src/Task.ts` | `TaskStatus` 联合类型定义 |
| `src/ink.js` | `Text` 组件 |
| `src/tools/BashTool/UI.tsx` | 调用 `TaskStatusText` |
| `src/tools/PowerShellTool/UI.tsx` | 调用 `TaskStatusText` |
| `src/components/shell/ShellProgressMessage.tsx` | 调用 `TaskStatusText` |
| `src/components/BashModeProgress.tsx` | 调用 `TaskStatusText` |

---

## 依赖与外部交互

### 运行时依赖
- **Ink `Text`**：唯一的 UI 基元依赖。
- **`TaskStatus` 类型**：来自 `src/Task.ts`，是全局统一的任务状态枚举。

### 无外部副作用
该组件是纯展示组件：
- 不调用 Hook（除 React Compiler 生成的 memo cache 外）。
- 不发起网络请求、不读取文件、不订阅状态。
- 所有数据通过 props 自上而下传递。

---

## 风险、边界与改进建议

### 风险
1. **编译产物特征**：文件经过 React Compiler 编译，包含 `const $ = _c(4)` 等机器生成代码。若仓库构建流程要求先编译再运行，直接修改此 `.tsx` 文件可能导致源码与运行时不一致。
2. **颜色映射过于简单**：`killed` 映射到 `warning`（黄色），在某些终端主题下可能与 `pending` 的默认色区分度不足。
3. **无 hover/交互状态**：作为纯文本组件，无法向用户传达"这是可点击的"或"长按可查看详情"等交互暗示。

### 边界
- `label` 为可选值，若未提供则回退到 `status` 原始字符串（如 `(running)`）。
- `suffix` 直接拼接在 `label` 之后，无自动逗号或空格处理，调用方需自行在 `suffix` 前加空格或逗号。
- 对于 `LocalShellTaskState` 的 `kind === "monitor"` 场景，`ShellProgress` 本身不做特殊处理，调用方（如 `BackgroundTask.tsx`）会在 `ShellProgress` 之外额外展示任务描述。

### 改进建议
1. **统一状态标签组件**：`TaskStatusText` 的映射逻辑（`completed→success`, `failed→error`, `killed→warning`）与 `taskStatusUtils.tsx` 中的 `getTaskStatusColor` 高度重合。可以考虑让 `TaskStatusText` 内部复用 `getTaskStatusColor`，避免两处逻辑漂移。
2. **支持动画状态**：`running` / `pending` 状态可以考虑增加一个微弱的脉冲动画（如颜色深浅交替），让用户更容易感知任务仍在进行。但这需要与 `prefersReducedMotion` 设置联动。
3. **增加 `title` 属性**：在 `Text` 上增加 `title`（或 Ink 的等效 tooltip）显示完整状态描述，对屏幕阅读器更友好。
4. **类型安全增强**：当前 `TaskStatusTextProps` 的 `status` 直接使用 `TaskStatus`，但组件内部 switch 只处理了 5 种状态。由于 TypeScript 的穷尽检查，这实际上是安全的，无需额外 `default` 分支。
