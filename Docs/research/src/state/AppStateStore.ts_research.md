# src/state/AppStateStore.ts 研究文档

## 场景与职责

`AppStateStore.ts` 是 Claude Code 整个前端状态的 **类型权威与默认值工厂**。它承担以下核心职责：

1. **定义 `AppState` 类型** — 覆盖从设置、任务、权限、MCP、插件、队友（teammates）、桥接（bridge）、推测（speculation）到 UI 焦点状态等几乎所有运行时状态。
2. **定义 `AppStateStore` 类型** — 即 `Store<AppState>` 的别名，为命令式 store 提供类型。
3. **提供 `getDefaultAppState()`** — 构造全新会话的初始状态，被 `main.tsx`、测试、以及 `AppStateProvider` 调用。
4. **导出辅助类型与常量** — 如 `CompletionBoundary`、`SpeculationState`、`IDLE_SPECULATION_STATE`。

该文件是 **纯 TypeScript（.ts）**，不含任何 JSX 或 React 导入（除了通过 `Store` 类型间接引用），因此是 .ts 调用方应该导入的推荐入口。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `AppState` 类型 | 作为整个应用的状态契约，约 450 行，涵盖 40+ 个顶级字段。任何新增的全局状态都应先在此声明。 |
| `DeepImmutable<...>` | 对大部分字段施加深度只读，防止组件或工具意外 mutate 状态；但 `tasks`、`agentNameRegistry` 等含函数/Map 的字段被显式排除。 |
| `getDefaultAppState()` | 统一初始化入口，确保新会话的所有字段都有确定性默认值，避免 `undefined` 导致的运行时防御代码泛滥。 |
| `IDLE_SPECULATION_STATE` | 推测（prompt suggestion 预执行）的 idle 哨兵对象，被多处引用以重置状态。 |
| 懒加载 `teammate.js` | 在 `getDefaultAppState` 中通过 `require('../utils/teammate.js')` 懒加载，打破与 `teammate.ts` 的循环依赖。 |

## 具体技术实现

### 1. AppState 类型结构

`AppState` 使用 `DeepImmutable<...>` 包裹一个巨大对象类型，并显式将某些字段排除在不可变之外：

```ts
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  // ... 约 40 个字段
}> & {
  tasks: { [taskId: string]: TaskState }              // 含函数类型，排除 DeepImmutable
  agentNameRegistry: Map<string, AgentId>             // Map 排除 DeepImmutable
  // ...
}
```

关键字段分组：

- **模型与设置**：`settings`, `mainLoopModel`, `mainLoopModelForSession`, `thinkingEnabled`, `effortValue`, `fastMode`, `advisorModel`
- **UI 布局**：`expandedView`, `footerSelection`, `statusLineText`, `isBriefOnly`, `activeOverlays`, `showTeammateMessagePreview`
- **任务与代理**：`tasks`, `agentNameRegistry`, `foregroundedTaskId`, `viewingAgentTaskId`, `coordinatorTaskIndex`, `viewSelectionMode`
- **队友/蜂群（Swarm）**：`teamContext`, `standaloneAgentContext`, `inbox`, `workerSandboxPermissions`, `pendingWorkerRequest`, `pendingSandboxRequest`
- **远程与会话**：`remoteSessionUrl`, `remoteConnectionStatus`, `remoteBackgroundTaskCount`, `tungstenActiveSession`, `tungstenPanelVisible`, `tungstenPanelAutoHidden`
- **桥接（Always-on Bridge）**：`replBridgeEnabled`, `replBridgeConnected`, `replBridgeSessionActive`, `replBridgeReconnecting`, `replBridgeConnectUrl`, `replBridgeSessionUrl`, `replBridgeEnvironmentId`, `replBridgeSessionId`, `replBridgeError`, `replBridgeInitialName`, `showRemoteCallout`
- **MCP**：`mcp`（clients, tools, commands, resources, pluginReconnectKey）
- **插件**：`plugins`（enabled, disabled, commands, errors, installationStatus, needsRefresh）
- **权限与安全**：`toolPermissionContext`, `denialTracking`, `initialMessage`（含 allowedPrompts）, `isUltraplanMode`
- **推测与提示**：`speculation`, `speculationSessionTimeSavedMs`, `promptSuggestion`, `promptSuggestionEnabled`
- **其他工具状态**：`bagelActive`, `bagelUrl`, `bagelPanelVisible`, `computerUseMcpState`, `replContext`
- **生命周期**：`authVersion`, `sessionHooks`, `fileHistory`, `attribution`, `todos`, `remoteAgentTaskSuggestions`, `notifications`, `elicitation`, `skillImprovement`, `pendingPlanVerification`, `ultraplanLaunching`, `ultraplanSessionUrl`, `ultraplanPendingChoice`, `ultraplanLaunchPending`

### 2. 推测（Speculation）状态机

```ts
export type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: { text: string; promptId: 'user_intent' | 'stated_intent'; generationRequestId: string | null } | null
    }
```

- 使用 mutable ref（`messagesRef`, `writtenPathsRef`, `contextRef`）避免每次消息到达都触发数组展开和状态复制。
- `CompletionBoundary` 标记推测完成的边界事件（complete / bash / edit / denied_tool）。

### 3. getDefaultAppState 实现细节

```ts
export function getDefaultAppState(): AppState {
  const teammateUtils = require('../utils/teammate.js') as typeof import('../utils/teammate.js')
  const initialMode: PermissionMode =
    teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired() ? 'plan' : 'default'

  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,
    // ...
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    // ...
  }
}
```

- **懒加载 teammate.js**：打破 `teammate.ts -> AppState.tsx -> ... -> main.tsx` 的循环依赖链。
- **初始权限模式**：若当前进程是 teammate 且要求 plan mode，则默认 `'plan'`，否则 `'default'`。
- **空集合初始化**：`new Map()`, `new Set<string>()`, `[]`, `{}` 等确保引用稳定且不会共享可变状态。

## 关键代码路径与文件引用

| 代码路径 | 说明 |
|----------|------|
| `AppState` 类型定义 (L89) | 被 **200+ 文件** 直接或间接引用，是整个应用的状态契约。 |
| `AppStateStore` 类型 (L454) | `Store<AppState>` 别名，被 `AppState.tsx`、各种工具函数、测试使用。 |
| `getDefaultAppState()` (L456) | 被 `src/main.tsx`、`src/state/AppState.tsx`、大量测试与恢复逻辑调用。 |
| `IDLE_SPECULATION_STATE` (L79) | 被 `src/services/PromptSuggestion/speculation.ts`、`src/main.tsx` 等引用。 |
| `CompletionBoundary` (L41) | 被推测系统与工具结果处理使用。 |

## 依赖与外部交互

### 直接依赖（类型与值）

- `./store.js` → `Store<T>` 类型
- `../utils/settings/settings.js` → `getInitialSettings()`
- `../utils/settings/types.js` → `SettingsJson`
- `../utils/permissions/PermissionMode.js` → `PermissionMode`
- `../utils/permissions/permissionSetup.js` → `getEmptyToolPermissionContext()`（通过 `../Tool.js` 间接）
- `../utils/teammate.js` → 懒加载，判断是否为 teammate 进程
- `../services/mcp/types.js` → `MCPServerConnection`, `ServerResource`
- `../tasks/types.js` → `TaskState`
- `../types/message.js` → `Message`, `UserMessage`
- `../utils/commitAttribution.js` → `AttributionState`
- `../utils/fileHistory.js` → `FileHistoryState`
- `../utils/hooks/sessionHooks.js` → `SessionHooksState`
- `../services/PromptSuggestion/promptSuggestion.js` → `shouldEnablePromptSuggestion()`
- `../utils/thinking.js` → `shouldEnableThinkingByDefault()`
- 以及大量其他类型导入（Agent、Plugin、Todo、Notification、Elicitation 等）

### 调用方（上游）

- `src/state/AppState.tsx`：re-export 类型与 `getDefaultAppState`。
- `src/main.tsx`：构造初始状态、创建 store。
- `src/state/onChangeAppState.ts`：读取 `AppState` 类型。
- `src/state/selectors.ts`：基于 `AppState` 做派生计算。
- `src/state/teammateViewHelpers.ts`：操作 `AppState` 中的任务与视图字段。
- 几乎所有组件、Hook、工具函数都会通过 `AppState.tsx` 的 re-export 间接依赖本文件的类型。

## 风险、边界与改进建议

### 风险

1. **类型文件过度膨胀**  
   `AppState` 已接近 450 行，新增字段缺乏模块化分区。任何修改都可能触发大范围类型重新检查，增加编译时间。

2. **循环依赖的隐患**  
   `getDefaultAppState` 中通过运行时 `require` 打破循环依赖，但这意味着 TypeScript 静态分析无法捕获 `teammate.js` 的变更影响，且对 tree-shaking 不友好。

3. **DeepImmutable 的盲区**  
   `tasks` 与 `agentNameRegistry` 被显式排除在 `DeepImmutable` 外，因为包含函数与 Map。这导致这些字段在类型层面是可变的，实际运行中若被 mutate 会破坏不可变假设。

4. **`computerUseMcpState` 的结构内联**  
   为避免 ant-only 依赖导致外部构建类型检查失败，`computerUseMcpState` 的字段被手动内联复制。若上游包修改结构，此处容易失步。

### 边界

- `getDefaultAppState()` 返回的是全新对象，但内部嵌套的 `new Map()`、`new Set()` 等集合在每次调用时都会新建，适合初始化；若用于测试中的“重置状态”，需注意不会自动清理外部副作用（如 WebSocket、abort controller）。
- `AppState` 中大量 `undefined | T` 的字段增加了运行时防御代码，但也反映了功能开关（feature flags）与跨构建差异。

### 改进建议

1. **按领域拆分 AppState 子类型**  
   将 `AppState` 拆分为 `UiState`、`TaskState`、`BridgeState`、`McpState`、`PluginState` 等子类型并组合，降低单文件认知负荷，也方便未来按领域做状态隔离。

2. **用 import type 替代部分值导入**  
   文件中混合了 `import type` 与 `import { value }`，可进一步审计哪些值导入只是为了 `getDefaultAppState` 中的默认值，考虑将默认值逻辑下沉到各自领域模块（如 `getDefaultMcpState()`）。

3. **消除运行时 require**  
   评估是否可通过将 teammate 检测逻辑上提到 `main.tsx` 并在构造初始状态时传入 `initialMode`，从而彻底移除 `getDefaultAppState` 中的 `require`。

4. **增加字段弃用与迁移注释**  
   对于像 `expandedView` 这种已替代旧字段（`showExpandedTodos` / `showSpinnerTree`）的状态，应在类型注释中明确标注迁移路径，防止新开发者误用旧概念。
