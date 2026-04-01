# PromptInputFooterLeftSide.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位

`PromptInputFooterLeftSide.tsx` 是 Claude Code 终端 UI 中**输入框底部左侧状态栏**的核心组件，负责渲染提示输入区域下方的所有状态指示器和快捷提示信息。它是 `PromptInputFooter.tsx` 的子组件，位于整个输入系统的最底层展示层。

### 1.2 核心职责

| 职责领域 | 具体功能 |
|---------|---------|
| **模式指示** | 渲染当前权限模式（Permission Mode）的视觉指示器 |
| **任务状态** | 显示后台任务（Background Tasks）的状态胶囊 |
| **团队状态** | 展示 Agent Swarms/Teammates 的运行状态 |
| **快捷提示** | 提供键盘快捷键的上下文提示（如 "esc to interrupt"） |
| **历史搜索** | 在 Ctrl+R 历史搜索模式下渲染搜索输入框 |
| **Vim 模式** | 显示 Vim INSERT/NORMAL 模式状态 |
| **语音模式** | 语音输入状态的提示和热身指示 |
| **PR 状态** | GitHub PR 审查状态的徽章展示 |
| **远程会话** | 远程模式（--remote/teleport）的指示器 |

### 1.3 调用关系

```
REPL.tsx
  └── PromptInput.tsx
        └── PromptInputFooter.tsx
              └── PromptInputFooterLeftSide.tsx (本组件)
                    ├── ModeIndicator (内部子组件)
                    ├── ProactiveCountdown (内部子组件)
                    ├── HistorySearchInput (外部组件)
                    ├── BackgroundTaskStatus (外部组件)
                    ├── TeamStatus (外部组件)
                    ├── PrBadge (外部组件)
                    ├── VoiceWarmupHint (外部组件)
                    └── KeyboardShortcutHint (设计系统组件)
```

---

## 2. 功能点目的

### 2.1 主入口组件: `PromptInputFooterLeftSide`

**Props 接口定义** (`Props` 类型，第52-73行):

```typescript
type Props = {
  exitMessage: { show: boolean; key?: string }     // 退出确认消息
  vimMode: VimMode | undefined                      // 当前 Vim 模式
  mode: PromptInputMode                             // 输入模式 (prompt/bash/orphaned-permission/task-notification)
  toolPermissionContext: ToolPermissionContext      // 工具权限上下文
  suppressHint: boolean                             // 是否抑制提示显示
  isLoading: boolean                                // AI 是否正在响应
  showMemoryTypeSelector?: boolean                  // 是否显示内存类型选择器
  tasksSelected: boolean                            // 任务胶囊是否被选中
  teamsSelected: boolean                            // 团队胶囊是否被选中
  tmuxSelected: boolean                             // tmux 胶囊是否被选中
  teammateFooterIndex?: number                      // 队友导航索引
  isPasting?: boolean                               // 是否正在粘贴
  isSearching: boolean                              // 是否处于历史搜索模式
  historyQuery: string                              // 历史搜索查询
  setHistoryQuery: (query: string) => void          // 设置历史搜索查询
  historyFailedMatch: boolean                       // 历史搜索是否无匹配
  onOpenTasksDialog?: (taskId?: string) => void     // 打开任务对话框回调
}
```

**渲染优先级逻辑** (第147-224行):

1. **退出消息优先** (第147-156行): 当 `exitMessage.show` 为 true 时，显示 "Press {key} again to exit"
2. **粘贴状态** (第158-166行): 当 `isPasting` 为 true 时，显示 "Pasting text…"
3. **历史搜索输入** (第179-188行): 当 `isSearching` 为 true 时，渲染 `HistorySearchInput` 组件
4. **Vim INSERT 模式** (第169-196行): 当 Vim 模式启用且处于 INSERT 模式时，显示 "-- INSERT --"
5. **模式指示器** (第198-223行): 默认情况下渲染 `ModeIndicator` 组件

### 2.2 模式指示器: `ModeIndicator`

**Props 接口定义** (`ModeIndicatorProps` 类型，第226-235行):

```typescript
type ModeIndicatorProps = {
  mode: PromptInputMode
  toolPermissionContext: ToolPermissionContext
  showHint: boolean
  isLoading: boolean
  tasksSelected: boolean
  teamsSelected: boolean
  tmuxSelected: boolean
  teammateFooterIndex?: number
  onOpenTasksDialog?: (taskId?: string) => void
}
```

**核心功能逻辑** (第237-482行):

#### 2.2.1 Bash 模式特殊处理 (第317-318行)
```typescript
if (mode === 'bash') {
  return <Text color="bashBorder">! for bash mode</Text>
}
```

#### 2.2.2 权限模式渲染 (第348-355行)
- 显示当前权限模式的符号和标题
- 支持 `shift+tab` 切换模式的快捷提示
- 远程模式下不显示（因为本地权限模式不反映远程代理状态）

#### 2.2.3 主要状态胶囊 (parts 数组构建，第359-368行)

| 胶囊类型 | 条件 | 组件 |
|---------|------|------|
| 远程会话 | `remoteSessionUrl` 存在 | `Link` 包裹的 "remote" 文本 |
| Tmux 会话 | `"external" === 'ant'` 且 `hasTmuxSession` | `TungstenPill` |
| 团队状态 | `isAgentSwarmsEnabled()` 且 `hasTeams` | `TeamStatus` |
| PR 状态 | `shouldShowPrStatus` 为 true | `PrBadge` |

#### 2.2.4 提示信息生成 (`getSpinnerHintParts` 函数，第484-512行)

根据当前状态生成动态提示:

| 状态 | 提示内容 |
|------|---------|
| `isLoading` | "esc to interrupt" |
| `hasRunningAgentTasks` 且非确认中 | "ctrl+x ctrl+k to stop agents" |
| `showToggleHint` | "ctrl+t to {show/hide} tasks" |

### 2.3 主动模式倒计时: `ProactiveCountdown`

**功能**: 在 PROACTIVE 或 KAIROS 功能启用时，显示距离下一次主动检查的倒计时。

**实现细节** (第74-126行):
- 使用 `useSyncExternalStore` 订阅主动模式状态变化
- 使用 `useState` 和 `useEffect` 管理倒计时状态
- 每秒更新一次剩余秒数
- 使用 `formatDuration` 格式化显示时间

### 2.4 语音模式提示

**条件渲染逻辑** (第424-448行):

1. **热身状态优先** (第424-425行): 当 `voiceWarmingUp` 为 true 时，显示 "keep holding…"
2. **选择提示** (第426-444行): 全屏环境下显示选择复制提示
3. **语音使用提示** (第445-448行): 当语音启用且空闲时，显示 "hold {key} to speak"

**提示显示次数限制** (第291-309行):
- 最大显示次数: `MAX_VOICE_HINT_SHOWS = 3`
- 使用全局配置 `voiceFooterHintSeenCount` 跟踪
- 通过 `useEffect` 在首次显示时递增计数

### 2.5 PR 状态徽章

**显示条件** (第334行):
```typescript
const shouldShowPrStatus = 
  isPrStatusEnabled() && 
  prStatus.number !== null && 
  prStatus.reviewState !== null && 
  prStatus.url !== null && 
  primaryItemCount < 2 && 
  (primaryItemCount === 0 || columns >= 80)
```

**逻辑说明**:
- PR 徽章只在空间充足时显示（主项目数 < 2）
- 80列以上的终端才显示（避免拥挤）

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 权限模式配置

```typescript
// src/utils/permissions/PermissionMode.ts (第42-91行)
const PERMISSION_MODE_CONFIG: Partial<Record<PermissionMode, PermissionModeConfig>> = {
  default: {
    title: 'Default',
    shortTitle: 'Default',
    symbol: '',
    color: 'text',
    external: 'default',
  },
  plan: {
    title: 'Plan Mode',
    shortTitle: 'Plan',
    symbol: PAUSE_ICON,  // ⏸
    color: 'planMode',
    external: 'plan',
  },
  acceptEdits: {
    title: 'Accept edits',
    shortTitle: 'Accept',
    symbol: '⏵⏵',
    color: 'autoAccept',
    external: 'acceptEdits',
  },
  // ... 其他模式
}
```

#### 3.1.2 AppState 中的相关状态

```typescript
// src/state/AppStateStore.ts (第89-161行，节选)
type AppState = DeepImmutable<{
  // ...
  expandedView: 'none' | 'tasks' | 'teammates'
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  viewingAgentTaskId?: string
  footerSelection: FooterItem | null  // 'tasks' | 'tmux' | 'bagel' | 'teams' | 'bridge' | 'companion'
  coordinatorTaskIndex: number  // -1 = pill, 0 = main, 1..N = agent rows
  toolPermissionContext: ToolPermissionContext
  // ...
}> & {
  tasks: { [taskId: string]: TaskState }
  teamContext?: {
    teamName: string
    teammates: { [teammateId: string]: { name: string; color?: string; /* ... */ } }
    // ...
  }
}
```

#### 3.1.3 任务状态类型

```typescript
// src/tasks/types.ts (第12-29行)
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState

export type BackgroundTaskState = TaskState  // 所有任务类型都可以是后台任务

// 判断是否为后台任务
export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') {
    return false
  }
  if ('isBackgrounded' in task && task.isBackgrounded === false) {
    return false
  }
  return true
}
```

### 3.2 关键流程

#### 3.2.1 队友视图切换流程

```typescript
// src/state/teammateViewHelpers.ts (第46-81行)
export function enterTeammateView(
  taskId: string,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  setAppState(prev => {
    const task = prev.tasks[taskId]
    const prevId = prev.viewingAgentTaskId
    const prevTask = prevId !== undefined ? prev.tasks[prevId] : undefined
    const switching =
      prevId !== undefined &&
      prevId !== taskId &&
      isLocalAgent(prevTask) &&
      prevTask.retain
    
    const needsRetain =
      isLocalAgent(task) && (!task.retain || task.evictAfter !== undefined)
    
    // ... 状态更新逻辑
    return {
      ...prev,
      viewingAgentTaskId: taskId,
      viewSelectionMode: 'viewing-agent',
      tasks,
    }
  })
}
```

#### 3.2.2 PR 状态轮询流程

```typescript
// src/hooks/usePrStatus.ts (第35-106行)
export function usePrStatus(isLoading: boolean, enabled = true): PrStatusState {
  const [prStatus, setPrStatus] = useState<PrStatusState>(INITIAL_STATE)
  
  useEffect(() => {
    if (!enabled) return
    
    async function poll() {
      // 检查空闲时间，超过 60 分钟停止轮询
      if (Date.now() - lastActivityTimestamp >= IDLE_STOP_MS) {
        return
      }
      
      const result = await fetchPrStatus()
      setPrStatus(prev => ({ /* 更新状态 */ }))
      
      // 如果请求超过 4 秒，永久禁用
      if (Date.now() - start > SLOW_GH_THRESHOLD_MS) {
        disabledRef.current = true
        return
      }
      
      // 安排下一次轮询
      timeoutRef.current = setTimeout(poll, POLL_INTERVAL_MS)
    }
    
    // 每 60 秒轮询一次
    void poll()
  }, [isLoading, enabled])
  
  return prStatus
}
```

#### 3.2.3 任务 V2 加载流程

```typescript
// src/hooks/useTasksV2.ts (第218-229行)
export function useTasksV2(): Task[] | undefined {
  const teamContext = useAppState(s => s.teamContext)
  
  // 只在启用 TodoV2 且是团队领导时启用
  const enabled = isTodoV2Enabled() && (!teamContext || isTeamLead(teamContext))
  
  const store = enabled ? getStore() : null
  
  return useSyncExternalStore(
    store ? store.subscribe : NOOP_SUBSCRIBE,
    store ? store.getSnapshot : NOOP_SNAPSHOT,
  )
}
```

### 3.3 条件编译与特性开关

组件大量使用 `feature()` 函数进行条件编译:

```typescript
// 第3行: 从 bun:bundle 导入 feature 函数
import { feature } from 'bun:bundle'

// 第6行: COORDINATOR_MODE 特性条件导入
const coordinatorModule = feature('COORDINATOR_MODE') 
  ? require('../../coordinator/coordinatorMode.js') 
  : undefined

// 第47行: PROACTIVE/KAIROS 特性条件导入
const proactiveModule = feature('PROACTIVE') || feature('KAIROS') 
  ? require('../../proactive/index.js') 
  : null

// 第266-272行: VOICE_MODE 特性条件 Hook
const voiceEnabled = feature('VOICE_MODE') ? useVoiceEnabled() : false
const voiceState = feature('VOICE_MODE') ? useVoiceState(s => s.voiceState) : 'idle'
const voiceWarmingUp = feature('VOICE_MODE') ? useVoiceState(s => s.voiceWarmingUp) : false
```

### 3.4 React Compiler 优化

组件使用 React Compiler 的缓存机制 (`_c` 函数) 进行性能优化:

```typescript
// 第1行: 导入 React Compiler 运行时
import { c as _c } from "react/compiler-runtime"

// 第127行: 组件使用编译器缓存
export function PromptInputFooterLeftSide(t0) {
  const $ = _c(27)  // 27 个缓存槽位
  // ...
}

// 缓存模式示例 (第148-156行):
if (exitMessage.show) {
  let t1
  if ($[0] !== exitMessage.key) {
    t1 = <Text dimColor={true} key="exit-message">Press {exitMessage.key} again to exit</Text>
    $[0] = exitMessage.key
    $[1] = t1
  } else {
    t1 = $[1]  // 使用缓存
  }
  return t1
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖文件

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `src/ink.js` | `Box`, `Text`, `Link` | Ink 渲染组件 |
| `src/types/textInputTypes.js` | `VimMode`, `PromptInputMode` | 输入类型定义 |
| `src/Tool.js` | `ToolPermissionContext` | 工具权限上下文类型 |
| `src/utils/permissions/PermissionMode.js` | `isDefaultMode`, `permissionModeSymbol`, `permissionModeTitle`, `getModeColor` | 权限模式工具函数 |
| `src/components/tasks/BackgroundTaskStatus.js` | `BackgroundTaskStatus` | 后台任务状态组件 |
| `src/components/teams/TeamStatus.js` | `TeamStatus` | 团队状态组件 |
| `src/components/PrBadge.js` | `PrBadge` | PR 状态徽章 |
| `src/components/design-system/KeyboardShortcutHint.js` | `KeyboardShortcutHint` | 键盘快捷键提示 |
| `src/components/design-system/Byline.js` | `Byline` | 行内分隔组件 |
| `src/hooks/usePrStatus.js` | `usePrStatus` | PR 状态 Hook |
| `src/hooks/useTasksV2.js` | `useTasksV2` | 任务 V2 Hook |
| `src/hooks/useTerminalSize.js` | `useTerminalSize` | 终端尺寸 Hook |
| `src/hooks/useVoiceEnabled.js` | `useVoiceEnabled` | 语音启用状态 |
| `src/state/AppState.js` | `useAppState`, `useAppStateStore` | 应用状态管理 |

### 4.2 间接依赖文件

| 文件路径 | 关系 | 说明 |
|---------|------|------|
| `src/state/AppStateStore.ts` | 类型定义 | AppState 完整类型定义 |
| `src/state/teammateViewHelpers.ts` | 状态操作 | 队友视图切换辅助函数 |
| `src/tasks/types.ts` | 类型定义 | TaskState 联合类型 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 类型守卫 | `isPanelAgentTask` 判断 |
| `src/components/tasks/taskStatusUtils.tsx` | 工具函数 | `shouldHideTasksFooter` 等 |
| `src/components/CoordinatorAgentStatus.tsx` | 组件/函数 | `getVisibleAgentTasks` 等 |
| `src/utils/agentSwarmsEnabled.ts` | 特性开关 | `isAgentSwarmsEnabled()` |
| `src/utils/swarm/backends/registry.ts` | 特性检测 | `isInProcessEnabled()` |
| `src/utils/fullscreen.ts` | 环境检测 | `isFullscreenEnvEnabled()` |
| `src/utils/format.ts` | 格式化 | `formatDuration()` |
| `src/utils/config.ts` | 配置读写 | `getGlobalConfig()`, `saveGlobalConfig()` |

### 4.3 调用方文件

| 文件路径 | 调用方式 | 说明 |
|---------|---------|------|
| `src/components/PromptInput/PromptInputFooter.tsx` | 直接导入 | 父组件，传递所有 Props |
| `src/components/StatusLine.tsx` | 引用检测 | `statusLineShouldDisplay` 影响 `suppressHint` |

---

## 5. 依赖与外部交互

### 5.1 状态管理依赖

```typescript
// 从 AppState 读取的状态 (第252-311行):
const tasks = useAppState(s => s.tasks)
const teamContext = useAppState(s => s.teamContext)
const viewSelectionMode = useAppState(s => s.viewSelectionMode)
const viewingAgentTaskId = useAppState(s => s.viewingAgentTaskId)
const expandedView = useAppState(s => s.expandedView)
const hasTmuxSession = useAppState(s => "external" === 'ant' && s.tungstenActiveSession !== undefined)
const isKillAgentsConfirmShowing = useAppState(s => s.notifications.current?.key === 'kill-agents-confirm')
```

### 5.2 全局配置依赖

```typescript
// 第41行: 配置读写
import { getGlobalConfig, saveGlobalConfig } from '../../utils/config.js'

// 使用场景:
// 1. voiceFooterHintSeenCount - 语音提示显示次数
// 2. prStatusFooterEnabled - PR 状态页脚启用
// 3. copyOnSelect - 选择时复制
```

### 5.3 特性标志依赖

| 特性标志 | 用途 | 相关代码 |
|---------|------|---------|
| `COORDINATOR_MODE` | 协调器模式检测 | 第6行, 第276行 |
| `PROACTIVE` | 主动模式倒计时 | 第47行, 第380-381行 |
| `KAIROS` | 主动模式倒计时 | 第47行, 第380-381行 |
| `VOICE_MODE` | 语音功能 | 第266-295行, 第424-448行 |
| `TRANSCRIPT_CLASSIFIER` | 自动模式 | PermissionMode.ts 第80-90行 |
| `ULTRAPLAN` | 超计划会话 | PromptInput.tsx 第521-522行 |
| `BRIDGE_MODE` | 桥接模式 | PromptInputFooter.tsx 第160-189行 |
| `BUDDY` | 伙伴功能 | PromptInput.tsx 第309-316行 |

### 5.4 平台/环境检测

```typescript
// 第42行: 平台检测
import { getPlatform } from '../../utils/platform.js'

// 第38行: 全屏环境检测
import { isFullscreenEnvEnabled } from '../../utils/fullscreen.js'

// 第39行: xterm.js 检测
import { isXtermJs } from '../../ink/terminal.js'
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 编译时条件导入的循环依赖风险

```typescript
// 第6行: 条件 require 可能引入循环依赖
const coordinatorModule = feature('COORDINATOR_MODE') 
  ? require('../../coordinator/coordinatorMode.js') 
  : undefined
```

**风险**: 如果 coordinatorMode.js 又间接依赖本组件，可能形成循环依赖。

#### 6.1.2 Hook 条件调用

```typescript
// 第266-272行: 条件调用 Hook (通过 feature() 守卫)
const voiceEnabled = feature('VOICE_MODE') ? useVoiceEnabled() : false
```

**风险**: 虽然 `feature()` 是编译时常量，但如果未来改为运行时判断，会违反 React Hooks 规则。

#### 6.1.3 硬编码的 "external" === 'ant' 检查

```typescript
// 第263行, 第367行, 第402行等
"external" === 'ant'
```

**风险**: 这是编译时常量折叠，用于区分内部/外部构建。如果构建配置改变，这些条件可能失效。

### 6.2 边界情况

#### 6.2.1 终端宽度不足

```typescript
// 第334行: PR 状态显示条件包含列数检查
const shouldShowPrStatus = /* ... */ && (primaryItemCount === 0 || columns >= 80)
```

**边界**: 小于 80 列的终端可能看不到 PR 状态。

#### 6.2.2 空状态处理

```typescript
// 第464-465行: 空状态返回空格保持高度
if (parts.length === 0 && !tasksPart && !modePart) {
  return isFullscreenEnvEnabled() ? <Text> </Text> : null
}
```

**边界**: 全屏模式下必须保持 1 行高度，避免布局跳动。

#### 6.2.3 语音提示计数上限

```typescript
// 第51行: 最大显示次数限制
const MAX_VOICE_HINT_SHOWS = 3
```

**边界**: 用户只能看到 3 次语音使用提示。

### 6.3 改进建议

#### 6.3.1 重构硬编码的构建类型检查

**建议**: 使用统一的构建类型常量替代 `"external" === 'ant'`:

```typescript
// 建议新增: src/constants/build.ts
export const IS_ANT_BUILD = "external" === 'ant'
export const IS_EXTERNAL_BUILD = "external" !== 'ant'
```

#### 6.3.2 提取 Magic Numbers

**建议**: 将分散的魔法数字提取为命名常量:

```typescript
// 当前分散的魔法数字:
// 第51行: MAX_VOICE_HINT_SHOWS = 3
// 第334行: columns >= 80 (终端最小宽度)
// 第328行: primaryItemCount < 2 (主项目数阈值)

// 建议集中到配置对象
const FOOTER_LAYOUT = {
  MIN_COLUMNS_FOR_PR_STATUS: 80,
  MAX_PRIMARY_ITEMS_BEFORE_HIDING_HINTS: 2,
  MAX_VOICE_HINT_SHOWS: 3,
} as const
```

#### 6.3.3 优化 useCoordinatorTaskCount

**当前实现** (CoordinatorAgentStatus.tsx 第83-88行):
```typescript
export function useCoordinatorTaskCount() {
  const tasks = useAppState(_temp)
  let t0
  t0 = 0
  return t0  // 始终返回 0?
}
```

**问题**: 该 Hook 似乎未完成实现，始终返回 0。

#### 6.3.4 类型安全改进

**建议**: 为 `footerSelection` 的导航逻辑添加更严格的类型:

```typescript
// 当前: 使用字符串比较
const tasksSelected = footerItemSelected === 'tasks'

// 建议: 使用类型守卫
function isFooterItem(value: string): value is FooterItem {
  return ['tasks', 'tmux', 'bagel', 'teams', 'bridge', 'companion'].includes(value)
}
```

#### 6.3.5 测试覆盖

**建议增加测试的场景**:
1. 各种权限模式下的渲染输出
2. 不同终端宽度下的布局调整
3. 语音提示计数器的持久化
4. PR 状态轮询的启停逻辑
5. 队友视图切换时的状态一致性

### 6.4 性能考虑

1. **React Compiler 缓存**: 组件已使用编译器优化，但缓存槽位数量 (27) 需要随功能增加而调整
2. **useSyncExternalStore 订阅**: 多个外部存储订阅 (proactiveModule, voiceState) 可能增加渲染开销
3. **PR 状态轮询**: 即使组件未显示 PR 徽章，轮询仍在后台运行（受 `enabled` 参数控制）
