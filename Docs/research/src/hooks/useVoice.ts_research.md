# useVoice.ts 研究文档

## 场景与职责

`useVoice` 是一个 React Hook，为 Claude Code 提供语音输入功能（hold-to-talk 和 focus 模式）。它通过 WebSocket 连接到 Anthropic 的 `voice_stream` 端点，实现实时语音转文字。

核心职责：
1. **音频录制**：使用原生音频模块（macOS）或 SoX 录制用户语音
2. **WebSocket 连接**：管理到 voice_stream 端点的 WebSocket 连接
3. **语音状态管理**：管理 `idle` -> `recording` -> `processing` -> `idle` 状态流转
4. **按键检测**：检测按住说话（hold-to-talk）的按键释放
5. **Focus 模式**：支持终端聚焦时自动开始录音的模式
6. **错误处理与重试**：处理连接错误、静默丢弃等情况
7. **音频可视化**：提供音频电平数据用于波形显示

该 Hook 被 `useVoiceIntegration.tsx` 使用，后者处理与文本输入的集成。

## 功能点目的

### 1. 语音状态机

```typescript
type VoiceState = 'idle' | 'recording' | 'processing'
```

- **idle**：空闲状态，等待用户触发
- **recording**：正在录音，音频数据通过 WebSocket 发送
- **processing**：录音结束，等待服务器返回最终转录结果

### 2. 按键释放检测

使用定时器检测按键释放（因为没有真正的 keyup 事件）：

```typescript
const RELEASE_TIMEOUT_MS = 200  // 自动重复键事件间隔超过 200ms 视为释放
const REPEAT_FALLBACK_MS = 600  // 首次按键后的回退定时器
export const FIRST_PRESS_FALLBACK_MS = 2000  // 修饰符组合首次按键的回退时间
```

- 终端自动重复通常每 30-80ms 触发一次
- 200ms 的阈值可以区分正常打字（100-300ms 间隔）和按住按键

### 3. 语言支持

支持 20 种语言的语音识别：

```typescript
const LANGUAGE_NAME_TO_CODE: Record<string, string> = {
  english: 'en', spanish: 'es', french: 'fr', japanese: 'ja',
  german: 'de', portuguese: 'pt', italian: 'it', korean: 'ko',
  hindi: 'hi', indonesian: 'id', russian: 'ru', polish: 'pl',
  turkish: 'tr', dutch: 'nl', ukrainian: 'uk', greek: 'el',
  czech: 'cs', danish: 'da', swedish: 'sv', norwegian: 'no',
}
```

语言代码需要与 GrowthBook 的 `speech_to_text_voice_stream_config` 允许列表匹配。

### 4. Focus 模式

```typescript
const FOCUS_SILENCE_TIMEOUT_MS = 5_000  // 5 秒静默后自动结束
```

- 终端获得焦点时自动开始录音
- 每次收到转录结果后重置静默定时器
- 失去焦点或静默超时时结束录音

### 5. 静默丢弃重试

```typescript
const fullAudioRef = useRef<Buffer[]>([])
const silentDropRetriedRef = useRef(false)
```

当服务器接受音频但未返回转录结果时（约 1% 的会话），使用缓存的音频在全新连接上重试一次。

### 6. 早期错误重试

```typescript
if (!opts?.fatal && !sawTranscript && stateRef.current === 'recording') {
  if (!retryUsedRef.current) {
    retryUsedRef.current = true
    // 250ms 后退后重试
  }
}
```

在收到任何转录结果之前发生连接错误时，自动重试一次。

## 具体技术实现

### 关键流程

#### 1. 开始录音会话

```typescript
async function startRecordingSession(): Promise<void> {
  // 1. 同步切换到 recording 状态（在 await 之前）
  updateState('recording')
  recordingStartRef.current = Date.now()
  
  // 2. 检查录音可用性
  const availability = await voiceModule.checkRecordingAvailability()
  if (!availability.available) {
    // 显示错误，返回 idle
  }
  
  // 3. 开始录音（立即开始，音频缓存到 WebSocket 连接完成）
  const started = await voiceModule.startRecording(
    (chunk: Buffer) => {
      // 缓存到 fullAudioRef（用于静默丢弃重试）
      // 发送已连接的 WebSocket，否则缓存到 audioBuffer
      // 计算音频电平用于可视化
    },
    () => { /* 外部结束回调 */ },
    { silenceDetection: false }
  )
  
  // 4. 连接 WebSocket
  void getVoiceKeyterms().then(attemptConnect)
}
```

#### 2. WebSocket 连接

```typescript
const attemptConnect = (keyterms: string[]): void => {
  void connectVoiceStream(
    {
      onTranscript: (text: string, isFinal: boolean) => {
        // Focus 模式：立即刷新每个最终结果
        // Hold-to-talk：累积最终结果
        // 更新 interim 预览
      },
      onError: (error: string, opts?: { fatal?: boolean }) => {
        // 早期错误重试逻辑
        // 显示错误通知
      },
      onClose: () => { /* 生命周期由 cleanup 处理 */ },
      onReady: conn => {
        // 刷新缓存的音频
        // 设置释放定时器
      },
    },
    { language: stt.code, keyterms }
  )
}
```

#### 3. 结束录音

```typescript
function finishRecording(): void {
  // 1. 递增 attemptGenRef 使旧的错误回调失效
  attemptGenRef.current++
  
  // 2. 捕获关键状态（在异步边界之前）
  const recordingDurationMs = Date.now() - recordingStartRef.current
  const hadAudioSignal = hasAudioSignalRef.current
  const wsConnected = everConnectedRef.current
  
  // 3. 停止录音，切换到 processing
  updateState('processing')
  voiceModule?.stopRecording()
  
  // 4. 发送 finalize，等待 WebSocket 关闭
  const finalizePromise = connectionRef.current
    ? connectionRef.current.finalize()
    : Promise.resolve(undefined)
  
  void finalizePromise.then(async finalizeSource => {
    // 5. 静默丢弃检测和重试
    if (finalizeSource === 'no_data_timeout' && hadAudioSignal && wsConnected && 
        !focusTriggered && focusFlushedChars === 0 && 
        accumulatedRef.current.trim() === '' && !silentDropRetriedRef.current) {
      // 重试逻辑
    }
    
    // 6. 注入转录结果或显示错误
    if (text) {
      onTranscriptRef.current(text)
    } else if (/* 条件 */) {
      onErrorRef.current?.('No speech detected.')
    }
    
    // 7. 清理，返回 idle
    updateState('idle')
  })
}
```

#### 4. 按键事件处理

```typescript
const handleKeyEvent = useCallback((fallbackMs = REPEAT_FALLBACK_MS): void => {
  if (!enabled || !isVoiceStreamAvailable()) return
  
  // Focus 模式忽略按键事件
  if (focusTriggeredRef.current) return
  
  // 静默超时后重新激活
  if (focusMode && silenceTimedOutRef.current) {
    silenceTimedOutRef.current = false
    focusTriggeredRef.current = true
    void startRecordingSession()
    armFocusSilenceTimer()
    return
  }
  
  const currentState = stateRef.current
  
  if (currentState === 'processing') return
  
  if (currentState === 'idle') {
    // 开始新会话
    void startRecordingSession()
    // 设置回退定时器（在没有自动重复时启动释放检测）
  } else if (currentState === 'recording') {
    // 标记已看到自动重复
    seenRepeatRef.current = true
    // 清除回退定时器
  }
  
  // 重置释放定时器
  if (releaseTimerRef.current) clearTimeout(releaseTimerRef.current)
  if (stateRef.current === 'recording' && seenRepeatRef.current) {
    // 设置新的释放定时器
  }
}, [enabled, focusMode, cleanup])
```

### 数据结构

#### 会话代际管理

```typescript
const sessionGenRef = useRef(0)      // 会话代际
const attemptGenRef = useRef(0)      // 连接尝试代际

// 开始新会话时
const myGen = ++sessionGenRef.current
const isStale = () => sessionGenRef.current !== myGen

// 开始新连接尝试时
const myAttemptGen = attemptGenRef.current
// 在回调中检查
if (attemptGenRef.current !== myAttemptGen) return  // 忽略过期的回调
```

这种代际机制确保旧的 WebSocket 回调不会干扰新的会话。

#### 音频电平计算

```typescript
export function computeLevel(chunk: Buffer): number {
  const samples = chunk.length >> 1  // 16-bit = 2 bytes per sample
  if (samples === 0) return 0
  let sumSq = 0
  for (let i = 0; i < chunk.length - 1; i += 2) {
    const sample = ((chunk[i]! | (chunk[i + 1]! << 8)) << 16) >> 16
    sumSq += sample * sample
  }
  const rms = Math.sqrt(sumSq / samples)
  const normalized = Math.min(rms / 2000, 1)
  return Math.sqrt(normalized)  // sqrt 曲线扩展低电平范围
}
```

### 错误处理策略

| 错误类型 | 处理策略 |
|---------|---------|
| 录音不可用 | 显示错误，返回 idle |
| WebSocket 连接失败 | 早期错误重试（一次） |
| 静默丢弃（no_data_timeout） | 使用缓存音频重试（一次） |
| 无音频信号 | 提示检查麦克风权限 |
| 无语音检测 | 提示没有检测到语音 |
| 致命错误（4xx） | 不重试，显示错误 |

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useVoice.ts` - Hook 实现

### 依赖文件
| 文件 | 用途 |
|------|------|
| `../context/voice.js` | 语音状态上下文（useSetVoiceState） |
| `../ink/hooks/use-terminal-focus.js` | 终端焦点检测 |
| `../services/analytics/index.js` | 分析事件上报 |
| `../services/voiceKeyterms.js` | 获取语音关键词 |
| `../services/voiceStreamSTT.js` | WebSocket STT 连接 |
| `../utils/debug.js` | 调试日志 |
| `../utils/intl.js` | 系统区域语言检测 |
| `../utils/settings/settings.js` | 用户设置 |

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useVoiceIntegration.tsx` - 语音与输入集成

### 相关文件
| 文件 | 用途 |
|------|------|
| `../services/voice.js` | 原生音频录制模块（懒加载） |
| `../context/voice.tsx` | 语音状态全局管理 |

## 依赖与外部交互

### VoiceStreamConnection 接口

```typescript
type VoiceStreamConnection = {
  send: (audioChunk: Buffer) => void
  finalize: () => Promise<FinalizeSource>
  close: () => void
  isConnected: () => boolean
}
```

### VoiceModule 接口（懒加载）

```typescript
type VoiceModule = {
  startRecording: (
    onData: (chunk: Buffer) => void,
    onEnd: () => void,
    options: { silenceDetection: boolean }
  ) => Promise<boolean>
  stopRecording: () => void
  checkRecordingAvailability: () => Promise<{ available: boolean; reason?: string }>
}
```

### WebSocket 协议

- **请求参数**：`encoding=linear16`, `sample_rate=16000`, `channels=1`
- **控制消息**：`KeepAlive`, `CloseStream`
- **响应消息**：`TranscriptText`, `TranscriptEndpoint`, `TranscriptError`

## 风险、边界与改进建议

### 潜在风险

1. **竞态条件**：
   - 多个 `sessionGen` 和 `attemptGen` 检查点，逻辑复杂
   - 如果代际检查遗漏，可能导致过期回调干扰当前会话

2. **内存使用**：
   - `fullAudioRef` 缓存所有音频数据（最多约 2MB，32KB/s * 60s）
   - Focus 模式不缓存，但 hold-to-talk 长时间录音可能占用较多内存

3. **按键释放检测不可靠**：
   - 依赖自动重复事件的间隔时间
   - 如果终端不发送自动重复，或用户按键非常慢，可能误判为释放

4. **WebSocket 连接时序**：
   - 音频在 WebSocket 连接完成前就开始录制
   - 如果连接失败，缓存的音频可能丢失（除非静默丢弃重试触发）

### 边界情况

| 场景 | 处理 |
|------|------|
| 录音期间禁用语音 | `useEffect` 清理，调用 `cleanup()` |
| 组件卸载 | `useEffect` 返回的清理函数 |
| 快速连续按键 | `sessionGen` 检查使旧会话的回调失效 |
| WebSocket 连接超时 | `finalize` 的 `safety_timeout` |
| 服务器无响应 | `no_data_timeout` 触发静默丢弃重试 |
| Focus 模式长时间录音 | 5 秒静默超时自动结束 |
| 切换出 Focus 模式 | 如果正在录音，调用 `finishRecording()` |

### 改进建议

1. **按键检测改进**：
   - 考虑使用更可靠的按键释放检测机制
   - 或者提供显式的开始/停止按钮作为备选

2. **音频压缩**：
   - 当前使用线性 PCM 16-bit 16kHz 单声道（32KB/s）
   - 可考虑使用 Opus 压缩减少带宽和内存使用

3. **流式转录优化**：
   - 当前只在收到 `TranscriptEndpoint` 时返回最终结果
   - 可考虑更细粒度的流式返回

4. **错误恢复**：
   - 当前静默丢弃重试只进行一次
   - 可考虑指数退避的多次重试

5. **测试覆盖**：
   - 添加单元测试模拟 WebSocket 各种状态
   - 测试代际检查逻辑
   - 测试按键释放检测

6. **可配置参数**：
   - 将 `RELEASE_TIMEOUT_MS`、`FOCUS_SILENCE_TIMEOUT_MS` 等设为可配置
   - 允许用户调整灵敏度
