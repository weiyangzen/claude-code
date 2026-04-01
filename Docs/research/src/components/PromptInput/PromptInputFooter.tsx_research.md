# PromptInputFooter.tsx 深度研究文档

> 研究对象：`src/components/PromptInput/PromptInputFooter.tsx`  
> 研究范围：代码、调用链、依赖模块、状态管理、相关测试与构建上下文  
> 执行器：kimi (k2p5)  
> 生成时间：2026-04-01

---

## 1. 场景与职责

`PromptInputFooter.tsx` 是 Claude Code 终端 UI 中**提示输入框底部栏（Footer）的顶层渲染组件**。它负责在用户输入区域的下方呈现所有与输入状态、系统状态、导航提示相关的视觉元素。

### 1.1 所在架构位置

- **父级调用方**：`src/components/PromptInput/PromptInput.tsx`（第 2274 行）
- **同级兄弟**：`PromptInput.tsx` 内部还包含输入框主体（`TextInput` / `VimTextInput`）、各种 Dialog（`AutoModeOptInDialog`、`BridgeDialog` 等）、以及绝对定位的 `Notifications` 覆盖层
- **渲染目标**：在 Ink（基于 React 的终端渲染框架）节点树中，该 Footer 作为底部固定区域的一部分输出

### 1.2 核心职责

1. **多态 Footer 渲染**：根据当前状态在以下三种互斥视图间切换：
   - **Suggestion 视图**：当存在输入建议（slash command / 文件补全等）且非全屏时，渲染建议列表
   - **Help 视图**：当 `helpOpen` 为 true 时，渲染快捷键帮助菜单
   - **标准 Footer 视图**：默认状态，渲染左右分栏的底部信息
2. **左右分栏信息聚合**：
   - **左侧**：`StatusLine`（用户自定义状态栏）、`PromptInputFooterLeftSide`（模式指示器、任务/团队 pill、Vim 模式、历史搜索、操作提示）
   - **右侧**：`Notifications`（API Key、IDE 连接、自动更新、Token 警告、语音状态等）、`BridgeStatusIndicator`（Remote Control 连接状态）
3. **全屏模式 Portal**：在全屏（fullscreen）模式下，将 suggestion 数据通过 `useSetPromptOverlay` 注入到 `FullscreenLayout` 的浮动覆盖层，避免被 `overflowY:hidden` 裁剪
4. **任务面板挂载**：在 ant 构建下（`"external" === 'ant'`），挂载 `CoordinatorTaskPanel` 以展示后台 Agent 任务列表

---

## 2. 功能点目的

### 2.1 建议列表渲染（Suggestion List）

- **目的**：在用户输入 `@`、`/`、`!` 等触发符时，展示可选择的补全项
- **行为差异**：
  - 非全屏：Footer 直接内联渲染 `<PromptInputFooterSuggestions>`
  - 全屏：Footer 不直接渲染建议 UI，而是通过 `useSetPromptOverlay` 将 `{suggestions, selectedSuggestion, maxColumnWidth}` 数据 portal 到 `FullscreenLayout`，由后者在 ScrollBox 外部渲染浮动覆盖层（解决 CC-668 裁剪问题）

### 2.2 状态栏（StatusLine）

- **目的**：当用户在 `settings.json` 中配置了 `statusLine` 字段时，执行用户自定义的状态栏命令并显示其输出
- **条件渲染**：仅在 `mode === 'prompt'`、非短屏、非粘贴中、无退出消息、且 `statusLineShouldDisplay(settings)` 返回 true 时显示
- **数据依赖**：需要 `messagesRef` 和 `lastAssistantMessageId`（通过 `getLastAssistantMessageId` 计算），用于在 StatusLine 内部 debounce 刷新

### 2.3 底部左侧信息（PromptInputFooterLeftSide）

- **目的**：聚合所有左下角的状态指示与操作提示
- **包含内容**：
  - 退出确认消息（`exitMessage`）
  - 粘贴中提示（`isPasting`）
  - 历史搜索输入框（`HistorySearchInput`）
  - Vim INSERT 模式提示
  - 权限模式指示器（`ModeIndicator`）
  - 后台任务/团队/Tmux pill 及导航提示

### 2.4 通知区（Notifications）

- **目的**：展示右侧的临时或持久系统通知
- **包含**：API Key 验证状态、自动更新状态、IDE 连接状态、MCP 客户端状态、Token 使用警告、语音指示器等
- **窄屏适配**：当终端宽度 `< 80` 时，Footer 整体切换为 column 布局，通知区与左侧信息垂直堆叠

### 2.5 Bridge 状态指示器（BridgeStatusIndicator）

- **目的**：显示 Remote Control（CCR 桥接）的连接状态
- **构建时门控**：整个子组件被 `feature('BRIDGE_MODE')` 包裹，若构建未开启该特性，则编译期消除
- **运行时门控**：还需 `isBridgeEnabled()` 和 `replBridgeEnabled` 状态同时满足
- **状态映射**：通过 `getBridgeStatus({connected, sessionActive, reconnecting})` 映射为标签文本和颜色（success/warning/error）

### 2.6 Coordinator 任务面板

- **目的**：在 ant 构建中，于 Footer 下方额外渲染可操控的后台 Agent 任务列表（`CoordinatorTaskPanel`）
- **交互**：支持 Enter 查看/steer、x 停止/清除

---

## 3. 具体技术实现

### 3.1 组件签名与 Props

```tsx
type Props = {
  apiKeyStatus: VerificationStatus;
  debug: boolean;
  exitMessage: { show: boolean; key?: string };
  vimMode: VimMode | undefined;
  mode: PromptInputMode;
  autoUpdaterResult: AutoUpdaterResult | null;
  isAutoUpdating: boolean;
  verbose: boolean;
  onAutoUpdaterResult: (result: AutoUpdaterResult) => void;
  onChangeIsUpdating: (isUpdating: boolean) => void;
  suggestions: SuggestionItem[];
  selectedSuggestion: number;
  maxColumnWidth?: number;
  toolPermissionContext: ToolPermissionContext;
  helpOpen: boolean;
  suppressHint: boolean;
  isLoading: boolean;
  tasksSelected: boolean;
  teamsSelected: boolean;
  bridgeSelected: boolean;
  tmuxSelected: boolean;
  teammateFooterIndex?: number;
  ideSelection: IDESelection | undefined;
  mcpClients?: MCPServerConnection[];
  isPasting?: boolean;
  isInputWrapped?: boolean;
  messages: Message[];
  isSearching: boolean;
  historyQuery: string;
  setHistoryQuery: (query: string) => void;
  historyFailedMatch: boolean;
  onOpenTasksDialog?: (taskId?: string) => void;
};
```

所有 Props 均由父组件 `PromptInput.tsx` 传入，无自身状态（纯展示组件）。

### 3.2 关键渲染流程

```
PromptInputFooter 渲染入口
│
├─ 如果 suggestions.length > 0 且 !isFullscreen
│   └─ 渲染 <PromptInputFooterSuggestions>（内联建议列表）
│
├─ 如果 helpOpen
│   └─ 渲染 <PromptInputHelpMenu>
│
└─ 默认分支
    ├─ 左侧列
    │   ├─ 条件渲染 <StatusLine>
    │   └─ <PromptInputFooterLeftSide>
    ├─ 右侧列
    │   ├─ 条件渲染 <Notifications>
    │   ├─ "external" === 'ant' && isUndercover() 时显示 undercover 文本
    │   └─ <BridgeStatusIndicator>
    └─ "external" === 'ant' 时渲染 <CoordinatorTaskPanel>
```

### 3.3 全屏 Portal 机制

```tsx
const overlayData = useMemo(() =>
  isFullscreen && suggestions.length
    ? { suggestions, selectedSuggestion, maxColumnWidth }
    : null,
  [isFullscreen, suggestions, selectedSuggestion, maxColumnWidth]
);
useSetPromptOverlay(overlayData);
```

- `useSetPromptOverlay` 来自 `src/context/promptOverlayContext.tsx`
- 该 Hook 在 effect 中将数据写入 `SetContext`，由 `FullscreenLayout.tsx` 通过 `usePromptOverlay` 读取
- 数据清除：组件 unmount 或 `overlayData` 变为 null 时，effect cleanup 自动调用 `set(null)`

### 3.4 任务 Pill 选中逻辑（pillSelected）

```tsx
const coordinatorTaskCount = useCoordinatorTaskCount();
const coordinatorTaskIndex = useAppState(s => s.coordinatorTaskIndex);
const pillSelected = tasksSelected && (coordinatorTaskCount === 0 || coordinatorTaskIndex < 0);
```

- `pillSelected` 决定“tasks” pill 是否处于高亮状态
- 当 Coordinator 任务面板存在可见行且 `coordinatorTaskIndex >= 0` 时，高亮从 pill 转移到具体任务行，pill 取消高亮
- **注意**：当前 `useCoordinatorTaskCount()` 的实现（`src/components/CoordinatorAgentStatus.tsx` 第 83-88 行）直接返回 `0`，是一个被精简过的存根实现；实际可见任务计数由 `getVisibleAgentTasks` 计算，但 Hook 本身未重新计算

### 3.5 窄屏与短屏适配

```tsx
const isNarrow = columns < 80;
const isFullscreen = isFullscreenEnvEnabled();
const isShort = isFullscreen && rows < 24;
```

- **窄屏（`isNarrow`）**：Footer 主容器 `flexDirection` 从 `row` 切换为 `column`，`justifyContent` 从 `space-between` 切换为 `flex-start`
- **短屏（`isShort`）**：在全屏且行数 `< 24` 时，隐藏 `StatusLine`（因为每多一行都是从 `ScrollBox` 中“偷”来的空间）

### 3.6 BridgeStatusIndicator 的实现细节

```tsx
function BridgeStatusIndicator({ bridgeSelected }: BridgeStatusProps): React.ReactNode {
  if (!feature('BRIDGE_MODE')) return null;
  const enabled = useAppState(s => s.replBridgeEnabled);
  const connected = useAppState(s => s.replBridgeConnected);
  const sessionActive = useAppState(s => s.replBridgeSessionActive);
  const reconnecting = useAppState(s => s.replBridgeReconnecting);
  const explicit = useAppState(s => s.replBridgeExplicit);
  // ...
}
```

- 使用了多次 `useAppState` 选择器读取桥接状态
- 代码中带有 `// biome-ignore lint/correctness/useHookAtTopLevel` 注释，说明这些 Hook 调用在 `feature()` 返回 false 的分支后；但由于 `feature()` 是编译期常量，构建工具会做死代码消除，因此运行时不会违反 Rules of Hooks
- 对于隐式（config-driven）桥接，仅在 `reconnecting` 状态时才显示指示器

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/components/PromptInput/PromptInput.tsx` | **唯一调用方**，负责传入所有 Props 和管理 Footer 交互状态 |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 渲染 Footer 左侧所有指示器和提示 |
| `src/components/PromptInput/PromptInputFooterSuggestions.tsx` | 渲染建议列表的 UI 和截断逻辑 |
| `src/components/PromptInput/PromptInputHelpMenu.tsx` | 渲染 `?` 快捷键帮助菜单 |
| `src/components/PromptInput/Notifications.tsx` | 渲染 Footer 右侧通知聚合 |
| `src/components/StatusLine.tsx` | 用户自定义状态栏命令执行与显示 |
| `src/components/CoordinatorAgentStatus.tsx` | `CoordinatorTaskPanel`、`useCoordinatorTaskCount`、`getVisibleAgentTasks` |
| `src/context/promptOverlayContext.tsx` | `useSetPromptOverlay` Hook 与 Portal Context |
| `src/bridge/bridgeEnabled.ts` | `isBridgeEnabled()` 运行时门控 |
| `src/bridge/bridgeStatusUtil.ts` | `getBridgeStatus()` 状态映射 |
| `src/state/AppStateStore.ts` | `AppState` 类型定义，包含 `footerSelection`、`coordinatorTaskIndex`、`replBridge*` 等字段 |
| `src/state/AppState.ts` | `useAppState`、`useSetAppState` |
| `src/hooks/useSettings.ts` | `useSettings` Hook |
| `src/hooks/useTerminalSize.ts` | `useTerminalSize` Hook |
| `src/utils/fullscreen.ts` | `isFullscreenEnvEnabled()` |
| `src/utils/undercover.ts` | `isUndercover()` |

### 4.2 状态数据流

```
AppStateStore (Zustand-like store)
│
├─ footerSelection: FooterItem | null        ← PromptInput.tsx 写入，PromptInputFooter 只读（通过 props 传递布尔展开）
├─ coordinatorTaskIndex: number              ← PromptInput.tsx 写入，PromptInputFooter 读取用于 pillSelected
├─ replBridgeEnabled: boolean                ← BridgeStatusIndicator 读取
├─ replBridgeConnected: boolean              ← BridgeStatusIndicator 读取
├─ replBridgeSessionActive: boolean          ← BridgeStatusIndicator 读取
├─ replBridgeReconnecting: boolean           ← BridgeStatusIndicator 读取
├─ replBridgeExplicit: boolean               ← BridgeStatusIndicator 读取
├─ tasks: { [taskId: string]: TaskState }    ← CoordinatorTaskPanel 读取
└─ settings: SettingsJson                    ← useSettings 读取（影响 StatusLine 显示）
```

### 4.3 构建时特性门控

- `feature('BRIDGE_MODE')`：控制 Bridge 相关代码是否编译进产物
- `feature('COORDINATOR_MODE')`：在 `PromptInputFooterLeftSide.tsx` 中通过条件 `require` 引入 coordinator 模块
- `"external" === 'ant'`：代码中多次出现的硬编码构建标识，用于 ant 内部构建专属功能（undercover 文本、CoordinatorTaskPanel、Tmux pill 等）

---

## 5. 依赖与外部交互

### 5.1 React / Ink 运行时依赖

- **React Compiler 输出**：文件内容显示为 React Compiler（React 19）编译后的产物，包含 `_c`（compiler runtime cache）调用模式。这意味着原始源码经过编译器自动优化了重渲染路径
- **Ink 组件**：`Box`、`Text` 来自 `src/ink.js`，是终端布局的核心原语
- **memo**：组件默认导出被 `memo()` 包裹，减少父级重渲染带来的不必要刷新

### 5.2 与 PromptInput.tsx 的紧耦合

`PromptInputFooter` 不直接订阅 `AppState` 中的 `footerSelection`，而是接收 `PromptInput.tsx` 预处理后的布尔展开值（`tasksSelected`、`teamsSelected`、`bridgeSelected`、`tmuxSelected`）。这种设计：
- **优点**：Footer 组件本身更简单，无需了解 `FooterItem` 枚举的具体值
- **缺点**：新增 FooterItem 类型时，必须同步修改 `PromptInput.tsx` 的解构逻辑和 `PromptInputFooter` 的 Props 类型

### 5.3 与 FullscreenLayout 的跨层级协作

通过 `promptOverlayContext.tsx` 实现跨层级数据传递：
- `PromptInputFooter` 是数据的**生产者**（suggestion 数据）
- `FullscreenLayout` 是数据的**消费者**（读取后渲染 `PromptInputFooterSuggestions`）
- 该机制专门用于解决全屏模式下 `overflowY:hidden` 对浮动覆盖层的裁剪问题（CC-668）

---

## 6. 风险、边界与改进建议

### 6.1 已识别的风险点

#### R1. `useCoordinatorTaskCount()` 存根实现
- **现状**：`src/components/CoordinatorAgentStatus.tsx` 中 `useCoordinatorTaskCount` 直接 `return 0`
- **影响**：`pillSelected` 的计算逻辑中 `coordinatorTaskCount === 0` 永远为 true，导致在存在 Coordinator 任务时 pill 高亮行为可能与设计意图不符（虽然 `coordinatorTaskIndex < 0` 仍能起到部分作用）
- **建议**：将该 Hook 实现为 `getVisibleAgentTasks(tasks).length`，与 `getVisibleAgentTasks` 保持一致

#### R2. Hook 规则抑制注释的维护负担
- **现状**：`BridgeStatusIndicator` 中多次使用 `// biome-ignore lint/correctness/useHookAtTopLevel`
- **风险**：若未来 `feature()` 不再是严格的编译期常量（例如改为运行时配置），这些被门控的 Hook 调用将在运行时触发 React 警告或崩溃
- **建议**：将 `feature('BRIDGE_MODE')` 的判断提取到组件外部（如构建时通过 dead code elimination 完全删除该组件），或在组件内部使用早期 return 的替代模式（如将 Hook 调用提升到条件分支之前）

#### R3. 硬编码构建标识 `"external" === 'ant'`
- **现状**：代码中多处出现该表达式，用于区分 ant 内部构建与外部发布构建
- **风险**：
  - 字符串字面量 `'ant'` 在代码中分散，若构建标识变更需要全局替换
  - 该表达式在运行时求值，虽然 bundler 可能做常量折叠，但不如 `feature()` 明确
- **建议**：统一封装为 `feature('ANT_BUILD')` 或 `isAntBuild()` 常量函数

#### R4. Props 数量膨胀
- **现状**：`Props` 类型包含 30+ 个字段，且多为从 `PromptInput.tsx` 透传
- **风险**：新增功能时持续增加 Props，导致类型维护成本上升、组件接口脆弱
- **建议**：考虑将部分相关状态分组为子对象（如 `footerSelection: { tasksSelected, teamsSelected, bridgeSelected, tmuxSelected }`），或让 `PromptInputFooterLeftSide` / `Notifications` 直接订阅 `AppState` 以减少中间透传

#### R5. 全屏与非全屏建议渲染路径分叉
- **现状**：建议列表在全屏和非全屏模式下由完全不同的渲染路径处理（Portal vs 内联）
- **风险**：UI 行为不一致的 bug 容易在此处引入（例如键盘导航、选中样式、截断逻辑）
- **建议**：确保 `FullscreenLayout` 中渲染的 `PromptInputFooterSuggestions` 与内联版本使用完全一致的 Props 和子组件（目前确实复用了同一组件，这是好的实践，应继续保持）

### 6.2 边界行为

| 边界条件 | 行为 |
|---------|------|
| `suggestions.length > 0 && !isFullscreen` | Footer 完全替换为建议列表，不显示 StatusLine、Notifications、Help 等任何其他内容 |
| `helpOpen === true` | 同样完全替换为 HelpMenu，与 suggestions 互斥（由 PromptInput.tsx 保证状态不重叠） |
| `isFullscreen && rows < 24` | 隐藏 StatusLine，优先保证输入区域最小可视行数 |
| `columns < 80` | Footer 主容器切换为 column 布局，左右信息垂直堆叠 |
| `suppressHint === true` | 左侧不显示 `? for shortcuts` 提示（在输入非空、自定义 statusLine、或历史搜索时触发） |
| `feature('BRIDGE_MODE') === false` | `BridgeStatusIndicator` 在编译期被消除，不产生任何运行时开销 |

### 6.3 改进建议汇总

1. **修复 `useCoordinatorTaskCount`**：使其返回真实的可见任务数量，而非硬编码 0
2. **减少 Props 透传**：让 `PromptInputFooterLeftSide` 和 `Notifications` 直接读取 `AppState` 中它们需要的状态，降低 `PromptInputFooter` 的接口表面积
3. **统一构建标识**：将 `"external" === 'ant'` 替换为显式的 feature flag 或编译期常量
4. **提取 Bridge 子组件**：`BridgeStatusIndicator` 逻辑较独立，可进一步拆分为单独文件，减少主文件的视觉噪音
5. **增加单元测试**：当前未找到针对 `PromptInputFooter` 的测试文件。建议补充：
   - 三种视图（suggestion / help / default）的切换渲染测试
   - `pillSelected` 在不同 `coordinatorTaskIndex` 下的高亮行为测试
   - 全屏模式下 `useSetPromptOverlay` 的调用/清除测试

---

## 7. 附录：相关代码片段索引

- **PromptInput.tsx 调用点**：第 2274 行
- **footerSelection 状态管理**：`src/state/AppStateStore.ts` 第 108 行
- **CoordinatorTaskPanel 可见任务过滤**：`src/components/CoordinatorAgentStatus.tsx` 第 31-33 行
- **useCoordinatorTaskCount 存根**：`src/components/CoordinatorAgentStatus.tsx` 第 83-91 行
- **promptOverlayContext 实现**：`src/context/promptOverlayContext.tsx` 第 22-95 行
- **Bridge 状态映射**：`src/bridge/bridgeStatusUtil.ts` 第 124-141 行
- **Bridge 运行时门控**：`src/bridge/bridgeEnabled.ts` 第 28-36 行
