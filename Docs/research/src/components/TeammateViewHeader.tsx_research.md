# TeammateViewHeader.tsx 研究文档

## 场景与职责

`TeammateViewHeader.tsx` 是 Claude Code REPL 中用于 ** teammate 转录视图（transcript view）** 的页头组件。当用户通过快捷键或命令进入某个 teammate 的详细会话视图时，该组件在视图顶部渲染一条简洁的头部信息，告知用户当前正在查看哪位 teammate 的转录，并显示该 teammate 的任务描述以及退出提示。

## 功能点目的

1. **视图上下文提示**：明确告知用户当前处于 "Viewing @<agentName>" 的 teammate 转录模式，防止用户误以为自己还在与主 Claude 对话。
2. **品牌/身份识别**：将 teammate 的名字以 **粗体 + 主题色** 显示，利用颜色区分不同 teammate（与 Agent Swarms 中的颜色体系一致）。
3. **任务描述展示**：在名字下方显示 teammate 当前正在执行的 `prompt`（任务描述），帮助用户快速理解该 teammate 的工作内容。
4. **退出操作引导**：通过 `KeyboardShortcutHint` 组件提示用户按 `esc` 即可返回主会话（return）。
5. **离屏冻结优化**：使用 `OffscreenFreeze` 包裹内容，当头部滚动出终端可视区域时停止不必要的重渲染，降低 log-update 的终端刷新开销。

## 具体技术实现

### 关键流程

1. **获取当前查看的 teammate 任务**
   - 调用 `useAppState(_temp)`，其中 `_temp` 是一个选择器函数：`s => getViewedTeammateTask(s)`。
   - 若返回 `undefined`（未进入 teammate 视图），直接返回 `null`，不渲染任何内容。

2. **颜色解析**
   - `toInkColor(viewedTeammate.identity.color)` 将 teammate 的 `identity.color`（如 `'blue'`、`'green'`）映射为 Ink `Text` 组件可识别的主题色键（如 `blue_FOR_SUBAGENTS_ONLY`）。
   - 若 `color` 未定义，则回退到 `DEFAULT_AGENT_THEME_COLOR = 'cyan_FOR_SUBAGENTS_ONLY'`。

3. **渲染结构**
   - 外层：`OffscreenFreeze` → `Box flexDirection="column" marginBottom={1}`
   - 内层第一行：`Box` 包含：
     - `<Text>Viewing </Text>`
     - `<Text color={nameColor} bold>@{agentName}</Text>`
     - `<Text dimColor> · <KeyboardShortcutHint shortcut="esc" action="return" /></Text>`
   - 内层第二行：`<Text dimColor>{prompt}</Text>`

4. **React Compiler 缓存**
   - 文件已被 React Compiler 编译，使用 `_c(14)` 缓存数组对 `nameColor`、`agentName`、`prompt` 等派生值和子元素进行 memoization，减少重渲染。

### 数据结构

- 无显式 Props 类型定义（组件不接受外部 props，完全自包含）。
- 内部依赖的数据类型：
  - `InProcessTeammateTaskState`（来自 `src/tasks/InProcessTeammateTask/types.ts`）
    - `identity: TeammateIdentity` → `{ agentId, agentName, teamName, color?, planModeRequired, parentSessionId }`
    - `prompt: string`

### 协议/命令

- 无网络协议或子进程命令。
- 纯展示组件，数据完全来自全局 AppState。

## 关键代码路径与文件引用

- **本文件**：`src/components/TeammateViewHeader.tsx`
- **调用方**：
  - `src/screens/REPL.tsx` — 主 REPL 屏幕，在 `viewingAgentTaskId` 存在时渲染该头部组件。
- **依赖类型与工具**：
  - `src/state/AppState.js` — `useAppState` hook。
  - `src/state/selectors.ts` — `getViewedTeammateTask` 选择器。
  - `src/utils/ink.ts` — `toInkColor` 颜色转换函数。
  - `src/components/design-system/KeyboardShortcutHint.tsx` — 键盘快捷键提示组件。
  - `src/components/OffscreenFreeze.tsx` — 离屏冻结优化组件。
  - `src/ink.js` — `Box`、`Text` 组件。

## 依赖与外部交互

| 依赖 | 路径 | 用途 |
|------|------|------|
| `useAppState` | `src/state/AppState.js` | 订阅全局状态，获取当前查看的 teammate 任务 |
| `getViewedTeammateTask` | `src/state/selectors.ts` | 纯函数选择器，从 AppState 中提取并校验 teammate 任务 |
| `toInkColor` | `src/utils/ink.ts` | 将 agent 颜色字符串转换为 Ink 主题色键 |
| `KeyboardShortcutHint` | `src/components/design-system/KeyboardShortcutHint.tsx` | 渲染 "esc to return" 提示 |
| `OffscreenFreeze` | `src/components/OffscreenFreeze.tsx` | 当组件滚动出可视区域时冻结子树，避免不必要的重渲染 |
| `Box`, `Text` | `src/ink.js` | Ink 布局与文本渲染 |

## 风险、边界与改进建议

### 风险与边界

1. **选择器耦合**：`getViewedTeammateTask` 对 `viewingAgentTaskId` 和 `tasks` 的校验逻辑若变更，会直接影响本组件的显隐行为。当前选择器在任务类型不是 `in_process_teammate` 时返回 `undefined`，这种保守策略是合理的，但需确保与任务创建逻辑保持一致。
2. **颜色回退单一**：所有未定义颜色的 teammate 都回退到 `cyan_FOR_SUBAGENTS_ONLY`，在 teammate 数量较多时可能产生颜色冲突，降低可区分性。
3. **prompt 可能过长**：`prompt` 字符串直接以 `dimColor` 渲染，未做截断处理。若 teammate 接收到的 prompt 极长（如包含大量文件内容），可能占用过多终端行数。不过 teammate prompt 通常由系统控制长度，实际风险较低。
4. **无错误边界**：若 `viewedTeammate.identity` 结构异常（如 `agentName` 缺失），组件将渲染 `undefined` 或抛出，缺乏防御性处理。

### 改进建议

1. **增加截断或折叠机制**：对 `prompt` 增加可选的 `truncateToWidth` 截断，或限制最大行数（如最多 2 行），防止极端长 prompt 撑开头部。
2. **颜色分配增强**：考虑在 `agentColorManager.ts` 中引入自动循环分配逻辑，确保未显式指定颜色的 teammate 也能获得多样化主题色，而非全部 cyan。
3. **添加单元测试**：
   - 测试 `getViewedTeammateTask` 的返回行为（存在/不存在/类型不匹配）。
   - 测试 `toInkColor` 的映射与回退。
   - 测试组件在 `viewedTeammate` 为 `undefined` 时返回 `null`。
4. **提取为更通用的 Header 模式**：若未来需要为其他视图（如本地 agent 视图、远程会话视图）添加类似头部，可将 "Viewing @X · esc to return" 的模式抽象为通用 `ViewHeader` 组件，接收 `name`、`color`、`subtitle`、`exitHint` 等 props。
