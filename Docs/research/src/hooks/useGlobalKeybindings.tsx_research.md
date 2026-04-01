# useGlobalKeybindings.tsx 研究文档

## 场景与职责

`GlobalKeybindingHandlers`（位于 `useGlobalKeybindings.tsx` 中）是一个**纯副作用的 React 组件/Hook**，负责注册 Claude Code 全局层面的键盘快捷键。它本身不渲染任何 UI（返回 `null`），只是集中管理所有应用级快捷键的注册逻辑。

这些快捷键包括：
- 切换待办事项视图（`ctrl+t`）
- 切换对话历史/转录模式（`ctrl+o`）
- 在转录模式中显示所有消息（`ctrl+e`）
- 退出转录模式（`ctrl+c` / `esc`）
- 切换 Brief 模式（`ctrl+shift+b`，受 feature flag 控制）
- 切换 teammate 消息预览（`app:toggleTeammatePreview`）
- 切换内置终端面板（`meta+j`，受 feature flag 控制）
- 强制重绘屏幕（`ctrl+l`）

## 功能点目的

1. **集中管理全局快捷键**：将所有跨屏幕、跨组件的通用快捷键统一在一个地方注册，避免分散在各个组件中导致维护困难。

2. **上下文感知激活**：部分快捷键只在特定上下文中激活（如 `transcript:toggleShowAll` 只在转录模式且虚拟滚动未激活时有效）。

3. **与 AppState 集成**：快捷键处理函数直接读写全局 AppState（如 `expandedView`、`isBriefOnly`、`showTeammateMessagePreview`）。

4. **Feature Flag 控制**：`KAIROS`、`KAIROS_BRIEF`、`TERMINAL_PANEL` 等编译时 feature flag 决定某些快捷键是否注册。

5. **Analytics 事件追踪**：几乎每个快捷键动作都会通过 `logEvent` 发送分析事件。

## 具体技术实现

### 组件结构

```tsx
export function GlobalKeybindingHandlers({
  screen,
  setScreen,
  showAllInTranscript,
  setShowAllInTranscript,
  messageCount,
  onEnterTranscript,
  onExitTranscript,
  virtualScrollActive,
  searchBarOpen = false
}: Props): null {
  // ... handlers defined with useCallback ...
  useKeybinding('app:toggleTodos', handleToggleTodos, { context: 'Global' })
  useKeybinding('app:toggleTranscript', handleToggleTranscript, { context: 'Global' })
  // ... more useKeybinding calls ...
  return null
}
```

### 关键处理函数

#### 1. `handleToggleTodos`（`ctrl+t`）
- 读取 `expandedView` 当前状态
- 检查是否有正在运行的 teammate 任务
- 循环切换：`none` → `tasks` → `teammates` → `none`（如果有 teammates）
- 或 `none` ↔ `tasks`（如果没有 teammates）
- 使用 `require` 动态加载 `InProcessTeammateTask.js`（避免顶层导入带来的循环依赖或包体积问题）

#### 2. `handleToggleTranscript`（`ctrl+o`）
- 在 `prompt` 和 `transcript` 屏幕之间切换
- 对于 `KAIROS` / `KAIROS_BRIEF` 模式，有一个特殊处理：如果 `isBriefOnly` 被卡住（如 GB kill-switch 关闭但状态仍开启），按 `ctrl+o` 会先清除 `isBriefOnly` 状态
- 进入/退出转录模式时触发 `onEnterTranscript` / `onExitTranscript` 回调
- 发送 `tengu_toggle_transcript` 分析事件

#### 3. `handleToggleShowAll`（`ctrl+e`，仅在转录模式）
- 切换 `showAllInTranscript` 布尔值
- 发送 `tengu_transcript_toggle_show_all` 事件

#### 4. `handleExitTranscript`（`ctrl+c` / `esc`）
- 将屏幕切回 `prompt`
- 重置 `showAllInTranscript = false`
- 仅在 `isInTranscript && !searchBarOpen` 时激活

#### 5. `handleToggleBrief`（`ctrl+shift+b`）
- 切换 `isBriefOnly` 状态
- 受 `isBriefEnabled()` 保护（GB kill-switch）
- OFF 转换总是允许（用户可以用同一个键退出 brief 模式）

#### 6. `handleToggleTerminal`（`meta+j`）
- 调用 `getTerminalPanel().toggle()`
- 使用 `spawnSync` 阻塞直到用户从 tmux  detach
- 受 `TERMINAL_PANEL` feature flag 和 `tengu_terminal_panel` GB 配置双重控制

#### 7. `handleRedraw`（`ctrl+l`）
- 调用 `instances.get(process.stdout)?.forceRedraw()`
- 用于修复外部清屏（如 macOS `Cmd+K`）后 Ink diff 引擎未重绘的问题

### Feature Flag 条件注册

```tsx
if (feature('KAIROS') || feature('KAIROS_BRIEF')) {
  // biome-ignore lint/correctness/useHookAtTopLevel: feature() is a compile-time constant
  useKeybinding('app:toggleBrief', handleToggleBrief, { context: 'Global' })
}
```
注释明确说明 `feature()` 是编译时常量，因此条件调用 Hook 是安全的（Bun bundler 会在构建时做死代码消除）。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useGlobalKeybindings.tsx` | 本组件实现 |
| `src/screens/REPL.tsx` | 唯一调用方，渲染在 `KeybindingSetup` 内部 |
| `src/keybindings/useKeybinding.ts` | `useKeybinding`、`useKeybindings` |
| `src/state/AppState.tsx` / `src/state/AppStateStore.js` | `useAppState`、`useSetAppState`、AppState 类型定义 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.js` | `getAllInProcessTeammateTasks`（动态 require） |
| `src/tools/BriefTool/BriefTool.js` | `isBriefEnabled`（动态 require） |
| `src/utils/terminalPanel.ts` | `getTerminalPanel`、内置终端面板 |
| `src/ink/instances.js` | `instances`、Ink 实例管理 |
| `src/services/analytics/index.js` | `logEvent` |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` |

## 依赖与外部交互

### 内部依赖
- **React**：`useCallback`
- **Ink 实例**：`instances.get(process.stdout)?.forceRedraw()`
- **键绑定系统**：`useKeybinding`
- **全局状态**：`useAppState`、`useSetAppState`
- **编译时 feature**：`feature('bun:bundle')`

### 外部交互
- **tmux**：`handleToggleTerminal` 通过 `spawnSync` 调用 `tmux`，进入/退出 alternate screen
- **Analytics**：每个主要动作都发送 `logEvent`

## 风险、边界与改进建议

### 风险与边界

1. **大量动态 `require`**：组件内部有多个 `require('../tasks/InProcessTeammateTask/InProcessTeammateTask.js')`、`require('../tools/BriefTool/BriefTool.js')` 等动态导入。虽然目的是避免循环依赖和减少包体积，但动态 require 在运行时首次执行会有性能开销，且类型安全较弱（使用了 `as typeof import(...)` 断言）。

2. **`feature()` 条件 Hook 的 lint 压制**：虽然 `feature()` 是编译时常量，但 Biome/ESLint 无法识别这一点，因此代码中使用了 `// biome-ignore lint/correctness/useHookAtTopLevel`。如果未来有人将条件改为运行时变量（如 `useAppState(s => s.someFlag)`），会违反 React Hooks 规则并导致难以调试的 bug。

3. **`handleToggleTerminal` 的阻塞调用**：`getTerminalPanel().toggle()` 内部使用 `spawnSync`，会完全阻塞 Node.js 事件循环。虽然这是终端面板的预期行为（用户需要与 shell 交互），但如果该操作被意外触发（如自动化测试、远程会话），会导致整个应用冻结。

4. **转录模式快捷键的竞态**：`handleExitTranscript` 和 `handleToggleTranscript` 都操作 `screen` 状态。如果用户极快地连续按键，React 的状态批处理通常能处理，但在某些边缘情况下可能出现状态抖动。

5. **`searchBarOpen` 的退出逻辑注释与实际行为**：注释说明 "Esc exits transcript directly"，但 `handleExitTranscript` 的 `isActive` 条件是 `!searchBarOpen`。这意味着当搜索栏打开时，Esc 不会退出转录模式——这与注释一致，但用户可能期望 Esc 能先关闭搜索栏再退出转录模式（目前关闭搜索栏由 `useSearchInput` 自己处理）。

6. **`virtualScrollActive` 对 `ctrl+e` 的禁用**：当虚拟滚动激活时，`transcript:toggleShowAll` 被禁用。这是为了避免快捷键与虚拟滚动导航冲突，但没有给用户任何视觉反馈说明为什么 `ctrl+e` 失效了。

### 改进建议

1. **将动态 require 改为顶层静态导入**：如果循环依赖问题已经通过其他方式解决（如提取共享类型到更底层模块），建议恢复为静态导入。这样可以获得更好的类型检查、Tree-shaking 和首次执行性能。

2. **为 `feature()` 条件 Hook 增加运行时保护**：可以在开发模式下增加一个断言：
   ```ts
   if (process.env.NODE_ENV === 'development') {
     console.assert(typeof feature('KAIROS') === 'boolean', 'feature() must be compile-time constant')
   }
   ```

3. **终端面板增加非阻塞模式（可选）**：对于自动化测试或 headless 环境，可以考虑让 `getTerminalPanel().toggle()` 支持一个非阻塞的 mock 模式，避免 `spawnSync` 冻结测试进程。

4. **统一快捷键帮助提示**：当前各个快捷键分散在代码中，用户很难发现所有快捷键。建议在 `HelpV2.tsx` 或一个专门的快捷键帮助页面中，自动从键绑定配置生成帮助文本，而不是手动维护。

5. **增加 `ctrl+e` 禁用时的视觉提示**：当虚拟滚动激活时，可以在转录模式 footer 中显示一个提示（如 "Show all disabled during scroll"），让用户知道 `ctrl+e` 为什么没反应。

6. **将 `GlobalKeybindingHandlers` 拆分为多个专注的组件**：当前这个组件负责了 7+ 个快捷键，逻辑越来越庞大。可以考虑按功能拆分为 `TranscriptKeybindingHandlers`、`ViewKeybindingHandlers`、`SystemKeybindingHandlers` 等，提高可维护性。
