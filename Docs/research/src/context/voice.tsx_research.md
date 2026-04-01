# voice.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位

`src/context/voice.tsx` 是 Claude Code 语音输入功能的**状态管理上下文(Context)**，负责：

- 提供语音状态的全局存储（基于自定义 Store 模式）
- 管理语音输入的生命周期状态（idle/recording/processing）
- 协调 UI 组件与语音服务之间的状态同步
- 支持 React 组件对语音状态的订阅和更新

### 1.2 业务场景

| 场景 | 说明 |
|------|------|
| 按住说话 (Push-to-Talk) | 用户按住空格键（可配置）开始录音，松开结束 |
| 焦点模式 (Focus Mode) | 终端获得焦点时自动开始录音，失去焦点结束 |
| 实时转写预览 | 录音过程中显示中间转写结果 (interim transcript) |
| 音频可视化 | 录音时显示音频电平波形 |
| 错误处理 | 麦克风权限、网络连接、STT 服务错误提示 |

### 1.3 构建时特性控制

该模块通过 `feature('VOICE_MODE')` 进行**编译时死代码消除(DCE)**：
- 仅在 Anthropic 内部构建中启用 (`ant` 构建)
- 外部构建获得空实现（passthrough provider）

---

## 2. 功能点目的

### 2.1 状态类型定义

```typescript
export type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing'  // 录音生命周期
  voiceError: string | null                        // 错误信息
  voiceInterimTranscript: string                   // 中间转写文本
  voiceAudioLevels: number[]                       // 音频电平数组（可视化）
  voiceWarmingUp: boolean                          // 预热状态（按键检测中）
}
```

### 2.2 各状态字段用途

| 字段 | 用途 |
|------|------|
| `voiceState` | 核心状态机：idle → recording → processing → idle |
| `voiceError` | 显示错误通知（如麦克风权限被拒绝） |
| `voiceInterimTranscript` | 实时显示识别的中间结果 |
| `voiceAudioLevels` | 驱动 TextInput 光标波形动画和音频可视化 |
| `voiceWarmingUp` | 显示 "keep holding..." 提示，防止误触发 |

### 2.3 导出 API

| API | 类型 | 用途 |
|-----|------|------|
| `VoiceProvider` | Component | 包裹应用，提供 voice store |
| `useVoiceState(selector)` | Hook | 订阅特定状态切片，避免不必要重渲染 |
| `useSetVoiceState()` | Hook | 获取 setState 方法（稳定引用） |
| `useGetVoiceState()` | Hook | 获取 getState 方法（同步读取） |

---

## 3. 具体技术实现

### 3.1 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                      VoiceProvider                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              VoiceStore (createStore)               │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  VoiceState: {                              │   │   │
│  │  │    voiceState, voiceError,                  │   │   │
│  │  │    voiceInterimTranscript,                  │   │   │
│  │  │    voiceAudioLevels, voiceWarmingUp         │   │   │
│  │  │  }                                          │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   useVoiceState      useSetVoiceState     useGetVoiceState
   (订阅状态)           (更新状态)           (同步读取)
        │                   │                   │
        ▼                   ▼                   ▼
   VoiceIndicator    useVoice.ts          VoiceKeybindingHandler
   TextInput          (录音逻辑)            (按键处理)
   Notifications
```

### 3.2 Store 实现机制

基于 `src/state/store.ts` 的通用 Store 实现：

```typescript
// 自定义 Store 模式（非 Redux，轻量级）
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

**关键特性**：
- 使用 `Object.is()` 进行状态变更检测
- 支持 `onChange` 回调（用于调试/分析）
- 订阅模式：发布-订阅，组件只重渲染当选择器返回值变化

### 3.3 React 集成

```typescript
// 使用 useSyncExternalStore 实现与 React 的同步
export function useVoiceState(selector) {
  const store = useVoiceStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

**性能优化**：
- 使用 `useSyncExternalStore` 替代 `useContext` + `useEffect`，避免撕裂(tearing)
- 选择器模式允许组件只订阅需要的字段
- `useSetVoiceState` 和 `useGetVoiceState` 返回稳定引用，不会触发重渲染

### 3.4 与 AppState 的集成

在 `src/state/AppState.tsx` 中，VoiceProvider 被包裹在应用根组件中：

```typescript
const VoiceProvider = feature('VOICE_MODE') 
  ? require('../context/voice.js').VoiceProvider 
  : ({ children }) => children  // 外部构建：透传

// 渲染层次
<AppStateProvider>
  <MailboxProvider>
    <VoiceProvider>
      {children}
    </VoiceProvider>
  </MailboxProvider>
</AppStateProvider>
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
voice.tsx
├── 依赖导入
│   ├── react (createContext, useContext, useState, useSyncExternalStore)
│   └── ../state/store.js (createStore, Store)
│
├── 被依赖（调用方）
│   ├── src/state/AppState.tsx (VoiceProvider 包裹)
│   ├── src/hooks/useVoice.ts (useSetVoiceState 更新状态)
│   ├── src/hooks/useVoiceIntegration.tsx (useVoiceState/useSetVoiceState/useGetVoiceState)
│   ├── src/hooks/useVoiceEnabled.ts (独立，使用 AppState)
│   ├── src/components/TextInput.tsx (useVoiceState 读取状态)
│   ├── src/components/PromptInput/VoiceIndicator.tsx (展示组件)
│   ├── src/components/PromptInput/Notifications.tsx (voiceError 展示)
│   └── src/components/PromptInput/PromptInputFooterLeftSide.tsx (voiceWarmingUp)
│
└── 相关服务
    ├── src/services/voice.ts (录音服务)
    ├── src/services/voiceStreamSTT.ts (WebSocket STT 连接)
    ├── src/services/voiceKeyterms.ts (STT 关键词优化)
    └── src/voice/voiceModeEnabled.ts (功能开关)
```

### 4.2 关键调用路径

**路径 1：录音状态更新流程**
```
useVoice.ts:startRecordingSession()
  → updateState('recording') 
  → setVoiceState(prev => ({...prev, voiceState: 'recording'}))
  → VoiceStore.setState()
  → 通知所有订阅者
  → VoiceIndicator/TextInput 重渲染
```

**路径 2：音频电平更新**
```
voice.ts:startRecording() 回调
  → computeLevel(chunk) 计算 RMS
  → setVoiceState(prev => ({...prev, voiceAudioLevels: snapshot}))
  → TextInput 光标波形更新
```

**路径 3：按键触发流程**
```
useVoiceKeybindingHandler:handleKeyDown()
  → 检测按住模式（warmup → recording）
  → voiceHandleKeyEvent() 
  → useVoice:handleKeyEvent()
  → startRecordingSession()
  → 更新 voiceState + voiceWarmingUp
```

**路径 4：转写结果流程**
```
voiceStreamSTT.ts:onTranscript()
  → useVoice.ts:handleVoiceTranscript()
  → setVoiceState(prev => ({...prev, voiceInterimTranscript: preview}))
  → useVoiceIntegration.tsx:interim effect
  → 更新输入框文本
```

### 4.3 关键代码段

**状态默认值** (`voice.tsx:11-17`):
```typescript
const DEFAULT_STATE: VoiceState = {
  voiceState: 'idle',
  voiceError: null,
  voiceInterimTranscript: '',
  voiceAudioLevels: [],
  voiceWarmingUp: false
}
```

**Provider 实现** (`voice.tsx:23-42`):
```typescript
export function VoiceProvider({ children }: Props): React.ReactNode {
  const [store] = useState(() => createStore(DEFAULT_STATE))
  return <VoiceContext.Provider value={store}>{children}</VoiceContext.Provider>
}
```

**选择器 Hook** (`voice.tsx:55-69`):
```typescript
export function useVoiceState(selector) {
  const store = useVoiceStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| React 19 | external | Compiler runtime, hooks |
| `createStore` | `../state/store.js` | 状态管理基础设施 |

### 5.2 服务层交互

| 服务 | 文件 | 职责 |
|------|------|------|
| voice.ts | `src/services/voice.ts` | 音频录制（原生/SoX/arecord） |
| voiceStreamSTT.ts | `src/services/voiceStreamSTT.ts` | WebSocket STT 连接管理 |
| voiceKeyterms.ts | `src/services/voiceKeyterms.ts` | 动态关键词生成 |
| voiceModeEnabled.ts | `src/voice/voiceModeEnabled.ts` | 功能开关与授权检查 |

### 5.3 配置与开关

| 配置项 | 位置 | 说明 |
|--------|------|------|
| `VOICE_MODE` | `bun:bundle` feature flag | 编译时开关 |
| `tengu_amber_quartz_disabled` | GrowthBook | 运行时 kill-switch |
| `settings.voiceEnabled` | AppState | 用户设置 |
| `authVersion` | AppState | 授权状态（影响 useVoiceEnabled） |

### 5.4 外部系统交互

```
┌─────────────┐     WebSocket      ┌─────────────────┐
│  voice.tsx  │◄──────────────────►│  Anthropic STT  │
│  (状态管理)  │   voice_stream     │  (Deepgram Nova)│
└──────┬──────┘                    └─────────────────┘
       │
       │ 音频数据
       ▼
┌─────────────┐
│  voice.ts   │
│ (音频录制)   │
└─────────────┘
```

**WebSocket 协议** (`voiceStreamSTT.ts`):
- 连接: `wss://api.anthropic.com/api/ws/speech_to_text/voice_stream`
- 参数: `encoding=linear16`, `sample_rate=16000`, `channels=1`
- 消息类型: `KeepAlive`, `CloseStream`, `TranscriptText`, `TranscriptEndpoint`, `TranscriptError`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 严重度 | 说明 |
|------|--------|------|
| 状态竞争 | 中 | `useGetVoiceState` 用于同步读取，但 effect 清理顺序可能导致竞态（注释提到 Effect 3 cleanup 在 Effect 2 finishRecording 之前运行） |
| 内存泄漏 | 低 | `fullAudioRef` 在 focus 模式下不缓冲，但普通模式可能累积最多 2MB 音频数据 |
| WebSocket 僵尸连接 | 中 | 已实现 `sessionGenRef` 和 `attemptGenRef` 防止，但仍有边界情况 |
| 静默丢弃(silent drop) | 中 | ~1% 会话会遇到 CE pod 问题，已实现重试逻辑 |

### 6.2 边界条件

| 边界 | 处理 |
|------|------|
| 无麦克风权限 | `checkRecordingAvailability()` 返回 `available: false` + 原因 |
| 无 OAuth 令牌 | `isVoiceStreamAvailable()` 返回 false，提示 /login |
| 远程环境 | `isRunningOnHomespace()` 检测，禁用语音 |
| 快速按键 | `RAPID_KEY_GAP_MS=120ms` 阈值，防止误触发 |
| 长录音 | 无硬性限制，但 `fullAudioRef` 最多约 2MB |

### 6.3 代码复杂度热点

1. **useVoice.ts** (1144 行)：
   - 状态机复杂（idle/recording/processing + focus mode + warmup）
   - 多个 timer ref 管理（releaseTimer, cleanupTimer, focusSilenceTimer）
   - 重试逻辑（early-error retry + silent-drop replay）

2. **useVoiceIntegration.tsx** (677 行)：
   - 按键处理逻辑复杂（bare char vs modifier combo）
   - 输入框文本同步（prefix/suffix/anchor 管理）
   - 与 useVoice.ts 的协调

### 6.4 改进建议

| 建议 | 优先级 | 说明 |
|------|--------|------|
| 状态机重构 | 中 | 考虑使用 XState 或类似库管理复杂的录音状态机 |
| 测试覆盖 | 高 | 当前测试主要依赖集成测试，单元测试覆盖不足 |
| 类型安全 | 低 | `useVoiceState` 的选择器可以加强类型约束 |
| 性能优化 | 低 | `voiceAudioLevels` 数组每次新建，可考虑使用 immutable 数据结构 |
| 文档完善 | 中 | 关键注释已很详细，但架构文档可补充更多时序图 |

### 6.5 关键注释摘录

来自代码中的关键实现说明：

> "Store is created once — stable context value means the provider never triggers re-renders. Consumers subscribe to slices via useVoiceState."

> "Transition to 'recording' synchronously, BEFORE any await. Callers read state synchronously right after `void startRecordingSession()`"

> "Session ending — stale any in-flight attempt so its late onError doesn't double-fire"

> "Nova 3's interims are cumulative across segments AND can revise earlier text... auto-finalize is never correct for it"

---

## 7. 总结

`voice.tsx` 是 Claude Code 语音功能的**状态管理中枢**，设计简洁但与其他模块有复杂交互：

- **优点**：使用轻量级 Store 模式，避免 Redux 开销；`useSyncExternalStore` 保证 React 18+ 并发安全；编译时 DCE 控制包大小
- **复杂度**：真正的业务复杂度在 `useVoice.ts`（录音控制）和 `useVoiceIntegration.tsx`（按键集成），voice.tsx 本身保持纯粹的状态管理职责
- **维护要点**：修改状态结构需要同步更新所有选择器调用点；timer 和 ref 的生命周期需要仔细管理

该模块体现了 Claude Code 的状态管理哲学：**分散的专用 Store**（而非单一全局 Store）+ **编译时特性控制** + **性能优先的订阅模式**。
