# useVoiceIntegration.tsx 深度研究文档

## 1. 场景与职责

### 1.1 定位
`useVoiceIntegration.tsx` 是 Claude Code 语音输入功能的核心集成层，位于 `src/hooks/` 目录下。它作为 **UI 层与语音服务层之间的桥梁**，负责：
- 将语音转录文本无缝集成到 PromptInput 输入框
- 处理按住说话（hold-to-talk）的按键交互逻辑
- 管理语音输入过程中的文本锚点、临时转录显示
- 协调键盘事件与语音状态机的同步

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| **按住说话** | 用户按住配置的快捷键（默认空格键）开始录音，松开结束 |
| **焦点模式** | 终端获得焦点时自动开始录音，失去焦点时结束（多窗口工作流） |
| **临时转录预览** | 录音过程中实时显示识别中的文本（interim transcript） |
| **文本插入** | 将最终转录文本插入到输入框光标位置，保留前后文 |

### 1.3 调用方
- **REPL.tsx**: 主屏幕组件，通过 `useVoiceIntegration()` 获取语音处理能力，通过 `VoiceKeybindingHandler` 组件处理键盘事件
- **PromptInput.tsx**: 输入框组件，接收 `voiceInterimRange` 用于淡化显示临时转录文本
- **PromptInputFooterLeftSide.tsx**: 底部状态栏，显示语音预热提示（VoiceWarmupHint）

---

## 2. 功能点目的

### 2.1 核心功能模块

#### 2.1.1 useVoiceIntegration Hook
```typescript
export function useVoiceIntegration(args: UseVoiceIntegrationArgs): UseVoiceIntegrationResult
```
**目的**: 为输入框提供语音转录的完整集成能力

**关键能力**:
| 能力 | 说明 |
|------|------|
| `stripTrailing` | 剥离尾部按住键字符（如空格），防止激活字符泄漏到输入框 |
| `resetAnchor` | 激活失败时恢复输入框状态，清理临时插入的间隔空格 |
| `handleKeyEvent` | 转发按键事件到语音核心（useVoice） |
| `interimRange` | 返回临时转录文本在输入框中的字符范围，用于UI淡化 |

#### 2.1.2 useVoiceKeybindingHandler Hook
```typescript
export function useVoiceKeybindingHandler(props: VoiceKeybindingHandlerProps): { handleKeyDown }
```
**目的**: 处理按住说话的键盘交互逻辑，区分"打字"和"按住"

**核心机制**:
- **预热检测（Warmup）**: 检测快速连续按键（<120ms间隔），区分正常打字和按住意图
- **阈值激活**: 裸字符（如空格）需要连续5次快速按键才激活；修饰符组合（如meta+k）首次即激活
- **字符剥离**: 激活时自动剥离已泄漏到输入框的预热字符

#### 2.1.3 VoiceKeybindingHandler 组件
**目的**: JSX 组件包装器，用于在组件树中挂载键盘处理器（向后兼容 shim）

---

## 3. 具体技术实现

### 3.1 关键流程

#### 3.1.1 按住说话激活流程
```
用户按住空格键
    ↓
[KeyDown事件] → matchesKeyboardEvent() 匹配配置的按键
    ↓
rapidCountRef 计数增加
    ↓
<WARMUP_THRESHOLD (2)? → 字符流入输入框（正常打字）
    ↓
≥WARMUP_THRESHOLD → 显示 VoiceWarmupHint（"keep holding…"）
    ↓
≥HOLD_THRESHOLD (5)? → 激活录音
    ↓
stripTrailing(anchor=true) 剥离字符并记录锚点
    ↓
voiceHandleKeyEvent() 启动录音会话
    ↓
持续按住期间：按键被吞掉（stopImmediatePropagation）
    ↓
用户松开按键 → RELEASE_TIMEOUT_MS (200ms) 超时后停止录音
    ↓
finalize() 等待最终转录 → handleVoiceTranscript() 插入文本
```

#### 3.1.2 文本锚点机制
```typescript
// 激活时记录锚点
voicePrefixRef.current = stripped;  // 光标前文本
voiceSuffixRef.current = afterCursor;  // 光标后文本
lastSetInputRef.current = newValue;    // 上次设置的完整值

// 临时转录更新时
const newValue = prefix_0 + leadingSpace + voiceInterimTranscript + trailingSpace + suffix_0;

// 最终转录插入时
const newInput = prefix_1 + leadingSpace_0 + text + trailingSpace_0 + suffix_1;
voicePrefixRef.current = prefix_1 + leadingSpace_0 + text;  // 更新前缀用于连续转录
```

#### 3.1.3 焦点模式流程
```
终端获得焦点 → useEffect 触发
    ↓
检查 voiceEnabled && focusMode && !silenceTimedOutRef
    ↓
focusTriggeredRef.current = true
    ↓
startRecordingSession() 开始录音
    ↓
armFocusSilenceTimer() 启动5秒静音超时
    ↓
每收到最终转录 → 立即 flush 到输入框并重置计时器
    ↓
终端失去焦点 → finishRecording() 结束录音
```

### 3.2 数据结构

#### 3.2.1 核心类型定义
```typescript
// src/hooks/useVoiceIntegration.tsx
interface UseVoiceIntegrationArgs {
  setInputValueRaw: React.Dispatch<React.SetStateAction<string>>;
  inputValueRef: React.RefObject<string>;
  insertTextRef: React.RefObject<InsertTextHandle | null>;
}

interface InsertTextHandle {
  insert: (text: string) => void;
  setInputWithCursor: (value: string, cursor: number) => void;
  cursorOffset: number;
}

interface InterimRange {
  start: number;
  end: number;
}

interface StripOpts {
  char?: string;      // 要剥离的字符（默认空格）
  anchor?: boolean;   // 是否记录锚点
  floor?: number;     // 最小保留字符数
}
```

#### 3.2.2 VoiceState 状态
```typescript
// src/context/voice.tsx
type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing';
  voiceError: string | null;
  voiceInterimTranscript: string;
  voiceAudioLevels: number[];
  voiceWarmingUp: boolean;
};
```

### 3.3 关键常量
```typescript
const RAPID_KEY_GAP_MS = 120;           // 快速按键间隔阈值
const HOLD_THRESHOLD = 5;               // 激活所需快速按键数
const WARMUP_THRESHOLD = 2;             // 显示预热提示的阈值
const MODIFIER_FIRST_PRESS_FALLBACK_MS = 2000;  // 修饰符组合首次按键回退超时
```

### 3.4 按键匹配算法
```typescript
function matchesKeyboardEvent(e: KeyboardEvent, target: ParsedKeystroke): boolean {
  // KeyboardEvent.key 是规范化名称（如 'space', 'return'）
  // ParsedKeystroke.key 存储 ' ' 表示空格，'enter' 表示回车
  const key = e.key === 'space' ? ' ' : e.key === 'return' ? 'enter' : e.key.toLowerCase();
  if (key !== target.key) return false;
  if (e.ctrl !== target.ctrl) return false;
  if (e.shift !== target.shift) return false;
  // meta 折叠 alt|option（终端限制）
  if (e.meta !== (target.alt || target.meta)) return false;
  if (e.superKey !== target.super) return false;
  return true;
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图
```
useVoiceIntegration.tsx
├── useVoice.ts              # 核心语音录制与STT连接
├── useVoiceEnabled.ts       # 语音功能使能检查
├── context/voice.tsx        # VoiceState 全局状态
├── context/notifications.tsx # 错误通知
├── context/overlayContext.tsx # 模态遮罩检测
├── keybindings/resolver.ts  # keystrokesEqual 工具
├── keybindings/KeybindingContext.tsx # 按键绑定上下文
├── ink/events/keyboard-event.ts # KeyboardEvent 类型
└── utils/stringUtils.ts     # normalizeFullWidthSpace
```

### 4.2 关键代码路径

#### 路径1: 按键激活 → 开始录音
```
useVoiceKeybindingHandler.handleKeyDown (line 468)
    ↓ 匹配 voiceKeystroke
    ↓ rapidCountRef >= HOLD_THRESHOLD
    ↓ stripTrailing(anchor=true) (line 573)
    ↓ voiceHandleKeyEvent() (line 578)
        ↓ useVoice.ts handleKeyEvent (line 1022)
            ↓ startRecordingSession()
```

#### 路径2: 临时转录 → UI更新
```
useVoice.ts onTranscript callback (line 783)
    ↓ setVoiceState({ voiceInterimTranscript })
        ↓ useVoiceIntegration.tsx useEffect (line 253)
            ↓ 计算 interimRange
                ↓ PromptInput.tsx highlights (line 675)
                    ↓ 淡化显示临时文本
```

#### 路径3: 最终转录 → 插入输入框
```
useVoice.ts finalize() Promise resolved
    ↓ onTranscriptRef.current(text) (line 490)
        ↓ useVoiceIntegration.tsx handleVoiceTranscript (line 281)
            ↓ 计算 prefix + text + suffix
            ↓ insertTextRef.current.setInputWithCursor()
                ↓ PromptInput.tsx 输入框更新
```

### 4.3 文件引用详情

| 文件 | 引用内容 | 用途 |
|------|----------|------|
| `src/hooks/useVoice.ts` | `useVoice()` hook | 核心录音逻辑、STT连接 |
| `src/context/voice.tsx` | `useVoiceState`, `useSetVoiceState`, `useGetVoiceState` | 全局语音状态管理 |
| `src/context/notifications.tsx` | `useNotifications` | 语音错误提示 |
| `src/context/overlayContext.tsx` | `useIsModalOverlayActive` | 模态框打开时禁用语音 |
| `src/keybindings/resolver.ts` | `keystrokesEqual` | 按键匹配比较 |
| `src/keybindings/KeybindingContext.tsx` | `useOptionalKeybindingContext` | 读取按键绑定配置 |
| `src/keybindings/defaultBindings.ts` | `space: 'voice:pushToTalk'` | 默认空格绑定 |
| `src/ink/events/keyboard-event.ts` | `KeyboardEvent` 类型 | 终端键盘事件 |
| `src/utils/stringUtils.ts` | `normalizeFullWidthSpace` | CJK全角空格处理 |
| `src/services/voiceStreamSTT.ts` | `connectVoiceStream` | WebSocket STT连接 |
| `src/services/voice.ts` | `startRecording`, `stopRecording` | 音频录制控制 |
| `src/voice/voiceModeEnabled.ts` | `isVoiceGrowthBookEnabled`, `hasVoiceAuth` | 功能开关检查 |

---

## 5. 依赖与外部交互

### 5.1 外部服务依赖

#### 5.1.1 Anthropic voice_stream 服务
```typescript
// src/services/voiceStreamSTT.ts
const VOICE_STREAM_PATH = '/api/ws/speech_to_text/voice_stream'
```
- **协议**: WebSocket (wss://)
- **认证**: OAuth Bearer Token (与 Claude Code 共用)
- **音频格式**: linear16, 16kHz, 单声道
- **STT引擎**: Deepgram Nova 3 (通过 `tengu_cobalt_frost` feature flag 控制)

#### 5.1.2 音频录制后端
| 平台 | 优先级 | 说明 |
|------|--------|------|
| audio-capture-napi (cpal) | 首选 | 原生模块，支持 macOS/Linux/Windows |
| SoX `rec` | 备选 | 跨平台命令行工具 |
| ALSA `arecord` | 备选 | Linux 专用 |

### 5.2 配置依赖
```typescript
// 功能开关（编译时）
feature('VOICE_MODE')  // bun:bundle 条件编译

// 运行时配置
settings.voiceEnabled  // 用户设置
isVoiceGrowthBookEnabled()  // GrowthBook kill-switch
hasVoiceAuth()  // OAuth 认证状态
```

### 5.3 关键绑定配置
```typescript
// src/keybindings/defaultBindings.ts
{ context: 'Chat', bindings: { space: 'voice:pushToTalk' } }
```
用户可通过 `keybindings.json` 自定义：
- 裸字符: `space`, `v`（需按住阈值）
- 修饰符组合: `meta+k`, `ctrl+x`（首次即激活）

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 竞态条件风险
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 提交竞态 | 用户在 finalize 等待期间按 Enter 提交，导致转录回填已清空的输入框 | `lastSetInputRef` 对比检查 (line 293) |
| 会话代际混淆 | 慢速 WebSocket 连接在新会话开始后完成，污染状态 | `sessionGenRef` / `attemptGenRef` 代际检查 |
| 焦点模式重入 | 快速焦点切换导致多个并发录音会话 | `silenceTimedOutRef` 状态锁 |

#### 6.1.2 平台兼容性风险
- **Windows Terminal**: 修饰符+空格（如 ctrl+space）被解析为 NUL → ctrl+backtick，无法使用
- **CJK输入法**: 全角空格（U+3000）需要特殊处理（`normalizeFullWidthSpace`）
- **Linux无音频**: ALSA 无 soundcard 时 cpal 写入 stderr，需预检 `/proc/asound/cards`

#### 6.1.3 静默丢包（Silent Drop）
- **现象**: 服务器接受音频但返回零转录（~1% 会话粘性到故障 CE pod）
- **检测**: `finalizeSource === 'no_data_timeout' && hadAudioSignal && wsConnected`
- **缓解**: 自动重播缓冲音频到新鲜连接（line 388-454 in useVoice.ts）

### 6.2 边界情况

#### 6.2.1 按键绑定边界
```typescript
// 裸字符绑定（如 'v'）的问题
// 输入 "hav" 后按住 'v' 激活，会多剥离一个 'v' 变成 "ha"
// 验证器警告：binding to voice:pushToTalk prints into input during warmup
```

#### 6.2.2 输入框状态边界
- **空输入框**: `prefix=''`, `suffix=''`，需正确处理空字符串的 `startsWith`/`endsWith`
- **光标在开头/结尾**: 不需要前导/尾随空格分隔
- **选择文本**: 语音输入时保留选择状态（通过 `insertTextRef` 处理）

#### 6.2.3 并发边界
- **语音+粘贴**: 粘贴大文本时语音转录到达，需确保不覆盖
- **语音+历史搜索**: 历史搜索模式下禁用语音（`!isActive || isModalOverlayActive`）

### 6.3 改进建议

#### 6.3.1 架构层面
1. **提取语音状态机**: 将 `useVoice.ts` 中的复杂状态机提取为独立模块，便于测试和复用
2. **统一按键处理**: 当前 `useVoiceKeybindingHandler` 和 `useInput` 存在双轨制，建议完全迁移到 `onKeyDown` 事件系统
3. **音频缓冲抽象**: 将 `fullAudioRef` 重播逻辑提取为可配置的音频缓冲区策略

#### 6.3.2 性能优化
1. **减少重渲染**: `interimRange` 计算在每次临时转录更新时触发，可使用 `useMemo` 优化
2. **防抖按键检测**: 当前 120ms 阈值是经验值，可根据用户打字习惯动态调整
3. **WebSocket连接池**: 焦点模式下频繁焦点切换可复用连接

#### 6.3.3 可维护性
1. **文档化魔法数字**: `HOLD_THRESHOLD=5`, `RAPID_KEY_GAP_MS=120` 等需要注释说明推导依据
2. **类型安全**: `voiceNs` 的条件导入使用 `any` 类型，建议完善类型定义
3. **测试覆盖**: 按键时序逻辑（warmup/activation）缺乏单元测试，建议添加模拟时钟测试

#### 6.3.4 用户体验
1. **可视化阈值**: 预热阶段可显示进度指示器（如 `■■□□□`）提示用户还需按住多久
2. **语音输入提示**: 首次使用时提示"按住空格说话"，降低发现成本
3. **错误恢复**: 网络错误后可自动重试，无需用户重新按住

---

## 7. 附录

### 7.1 相关研究文档
- `src/context/voice.tsx` - VoiceState 全局状态管理
- `src/hooks/useVoice.ts` - 核心语音录制逻辑
- `src/services/voiceStreamSTT.ts` - WebSocket STT 客户端
- `src/commands/voice/voice.ts` - /voice 命令实现

### 7.2 调试技巧
```typescript
// 启用语音调试日志
process.env.DEBUG_VOICE = '1'

// 关键日志标记
'[voice] Starting recording session'
'[voice] onTranscript'
'[voice] finishRecording'
'[voice_stream] Connecting to'
'[voice_stream] WebSocket connected'
```

### 7.3 性能指标（代码内注释）
- 首次按键到录音开始: ~0ms（同步状态更新）
- WebSocket 连接建立: ~1-2s（OAuth 刷新 + TLS 握手）
- 音频缓冲消除延迟: 连接期间音频零丢失
- 最终转录延迟: ~300ms（TranscriptEndpoint）到 ~5s（safety timeout）
