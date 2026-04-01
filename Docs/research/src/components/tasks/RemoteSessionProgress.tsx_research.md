# RemoteSessionProgress.tsx 研究文档

## 场景与职责

`RemoteSessionProgress.tsx` 是 Claude Code 终端 UI 中负责渲染**远程会话（Remote Agent）进度指示器**的专用组件。它主要被 `BackgroundTask.tsx`（行 10 引用）和 `RemoteSessionDetailDialog.tsx`（行 29 引用）消费，用于在背景任务列表的 pill 区域以及详情弹窗的进度行中，统一展示远程任务的状态。

该组件需要处理三类远程会话的视觉呈现：
1. **Ultrareview（远程代码审查）**：需要彩虹渐变动画、阶段流水线计数（finding → verifying → synthesizing）。
2. **Ultraplan**：由 `RemoteSessionDetailDialog.tsx` 单独处理，本组件仅作为 pill 中的简短状态展示。
3. **普通远程会话**：展示 todo 列表完成进度或简单的 running/completed/failed 文本。

组件的设计目标之一是**防止 pill 与详情弹窗的进度文案出现漂移（drift）**，因此将阶段计数格式化逻辑 `formatReviewStageCounts` 提取为共享导出函数，供详情弹窗直接复用。

---

## 功能点目的

### 1. `formatReviewStageCounts`
将 ultrareview 的三个阶段（`finding` / `verifying` / `synthesizing`）以及对应的 bug 计数格式化为人类可读的短文本。

- **finding**：只显示 `N found`，若 `found === 0` 则显示 `finding`。
- **verifying**：显示 `N found · M verified`，可选追加 `X refuted`（仅当 refuted > 0）。
- **synthesizing**：显示 `M verified · X refuted · deduping`（refuted 同样可选）。
- **无 stage**：回退到 `N found · M verified`（兼容早期 orchestrator 未写入 stage 字段的情况）。

### 2. `RemoteSessionProgress`
入口组件，接收 `session: DeepImmutable<RemoteAgentTaskState>`，按优先级分支渲染：
- `session.isRemoteReview === true` → 渲染 `ReviewRainbowLine`。
- `status === "completed"` → `<Text bold color="success" dimColor>done</Text>`。
- `status === "failed"` → `<Text bold color="error" dimColor>error</Text>`。
- `todoList.length === 0` → 显示 `status…`（如 `running…`）。
- 否则 → 显示 `completed/total`（基于 `count(session.todoList, _ => _.status === "completed")`）。

### 3. `ReviewRainbowLine`
Ultrareview 的核心视觉组件，负责：
- 使用 `useAnimationFrame(TICK_MS = 80)` 驱动彩虹文字动画。
- 尊重用户设置中的 `prefersReducedMotion`：若开启，则冻结动画相位并直接 snap 到目标计数。
- 使用自定义 Hook `useSmoothCount` 让数字从当前值平滑递增到目标值（每帧 +1），避免计数跳变。
- 对 `completed` / `failed` / `running` 三种状态分别输出固定文案：
  - completed：`◆ ultrareview ready · shift+↓ to view`
  - failed：`◆ ultrareview · error`
  - running/pending：`◇ ultrareview · {stageCounts}`（`ultrareview` 文字带彩虹渐变动画）。

### 4. `RainbowText`
内部辅助组件，将字符串按字符拆分为 `<Text>` 数组，每个字符应用 `getRainbowColor(i + phase)`，实现逐字符彩虹渐变。

### 5. `useSmoothCount`
内部 Hook，基于 `useRef` 保存当前显示值和上一帧时间戳。当 `target > displayed` 且 `time` 发生变化时，每帧递增 1；若 `snap` 为 true 或 target 减小，则直接赋值。

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 核心类型
```ts
type ReviewStage = NonNullable<
  NonNullable<RemoteAgentTaskState['reviewProgress']>['stage']
>;
// => 'finding' | 'verifying' | 'synthesizing'
```

### 动画时钟
- `TICK_MS = 80`：动画帧间隔。
- `useAnimationFrame(running && !reducedMotion ? TICK_MS : null)`：来自 `src/ink.js` 的 Ink 自定义 Hook，返回 `[unused, time]`。
- `phase = Math.floor(time / (TICK_MS * 3)) % 7`：彩虹相位每 240ms 切换一次，循环 7 色。

### 颜色系统
- `getRainbowColor(index: number)` 来自 `src/utils/thinking.ts`，返回 `keyof Theme`（`rainbow_red` … `rainbow_violet`）。
- 主题颜色通过 Ink 的 `<Text color={...}>` 注入。

### 计数平滑
```ts
const found = useSmoothCount(p?.bugsFound ?? 0, time, snap);
const verified = useSmoothCount(p?.bugsVerified ?? 0, time, snap);
const refuted = useSmoothCount(p?.bugsRefuted ?? 0, time, snap);
```
`useSmoothCount` 的 `snap` 条件：
- `reducedMotion === true`（无障碍需求）
- `!running`（任务已结束，无需动画）

### React Compiler
文件顶部显式导入 `import { c as _c } from "react/compiler-runtime"`，说明该文件经过 React Compiler（React Forget）编译，所有组件内部使用 `$` 数组进行 memo cache 管理。这是编译产物特征，不是手写代码。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/RemoteSessionProgress.tsx` | 本文件 |
| `src/components/tasks/BackgroundTask.tsx` | 调用 `RemoteSessionProgress` 渲染 remote_agent 类型任务的 pill |
| `src/components/tasks/RemoteSessionDetailDialog.tsx` | 调用 `formatReviewStageCounts` 和 `RemoteSessionProgress` |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 定义 `RemoteAgentTaskState` 及 `reviewProgress` 结构 |
| `src/utils/thinking.ts` | `getRainbowColor` |
| `src/utils/array.ts` | `count` |
| `src/ink.js` | `Text`, `useAnimationFrame` |
| `src/hooks/useSettings.js` | `useSettings` |
| `src/constants/figures.js` | `DIAMOND_FILLED`, `DIAMOND_OPEN` |

---

## 依赖与外部交互

### 运行时依赖
- **Ink（`src/ink.js`）**：提供终端 React 渲染基元 `Text` 和动画 Hook `useAnimationFrame`。
- **Settings（`useSettings`）**：读取 `prefersReducedMotion` 以决定是否启用动画。
- **RemoteAgentTaskState**：由 `RemoteAgentTask.tsx` 维护，包含 `reviewProgress`（orchestrator 通过 hook_progress 回写）。

### 数据流
1. `RemoteAgentTask.tsx` 的轮询器每 1 秒解析 `<remote-review-progress>` XML 标签，更新 `reviewProgress` 字段。
2. `BackgroundTask.tsx` 从 `AppState.tasks` 读取任务状态，将 `session` 传入 `RemoteSessionProgress`。
3. `RemoteSessionProgress` 根据 `isRemoteReview` 和 `status` 分支渲染。
4. `RemoteSessionDetailDialog.tsx` 在详情弹窗中复用 `formatReviewStageCounts` 保证文案一致。

---

## 风险、边界与改进建议

### 风险
1. **编译产物可读性差**：文件是 React Compiler 编译输出，包含大量 `t1`, `t2`, `$[n]` 等机器生成变量。人工直接修改源码后若未重新编译，运行时代码与源码会不一致。
2. **动画帧与计数 tick 耦合**：`useSmoothCount` 依赖 `time` 变化来递增计数。如果 `useAnimationFrame` 在后台标签页被节流（虽然终端应用通常不会），计数更新也会变慢。
3. **refuted 为 0 的隐藏逻辑**：`formatReviewStageCounts` 在 refuted === 0 时隐藏该字段，这是产品决策，但可能导致用户误以为没有 refutation 阶段。

### 边界
- `reviewProgress` 可能为 `undefined`（orchestrator 尚未写入第一条心跳），此时 `ReviewRainbowLine` 回退到 `"setting up"`。
- `todoList` 为空时显示 `status…`，不会显示 `0/0`。
- 已完成/已失败的 ultrareview 不再显示阶段计数，而是显示固定文案。

### 改进建议
1. **源码与编译产物分离**：当前仓库似乎直接提交编译后的 `.tsx`（或构建流程在编译后覆盖原文件）。建议明确区分 `src/`（源码）与 `dist/`（编译产物），避免研究者/开发者误改编译输出。
2. **抽离 `useSmoothCount`**：该 Hook 与 `SpinnerAnimationRow` 中的 token counter 模式相同，可统一提取到 `src/hooks/useSmoothCount.ts` 减少重复。
3. **增加 `reviewProgress` 缺失时的阶段提示**：目前直接显示 `"setting up"`，可以考虑显示更具体的提示（如 `"booting container…"`），但这需要 orchestrator 协议支持。
4. **测试覆盖**：该组件涉及动画和颜色，现有测试可能以快照为主。建议增加对 `formatReviewStageCounts` 各分支的单元测试，以及对 `useSmoothCount` 边界（target 减小、snap 模式）的测试。
