# Voice Command Implementation Research Document

## 场景与职责

`src/commands/voice/voice.ts` 是 `/voice` 命令的实际实现模块，负责语音模式的切换逻辑。当用户执行 `/voice` 命令时，该模块：

1. **权限验证**：检查用户是否已登录 Claude.ai 账号
2. **环境检查**：验证录音设备可用性（麦克风权限、音频工具）
3. **状态切换**：开启或关闭语音模式设置
4. **用户引导**：提供快捷键提示和语言设置反馈

该模块是语音功能的控制中心，确保用户在合适的环境下才能启用语音输入。

## 功能点目的

### 核心功能

| 功能 | 描述 |
|------|------|
| Toggle OFF | 关闭语音模式，更新设置并通知变更 |
| Toggle ON | 预检通过后开启语音模式，提供操作指引 |
| 预检流程 | 录音可用性 → API 可用性 → 依赖检查 → 麦克风权限 |

### Toggle ON 预检流程

```
1. 检查录音可用性 (checkRecordingAvailability)
   └── 远程环境? → 拒绝
   └── 原生音频模块? → 通过
   └── arecord? → 通过 (Linux)
   └── SoX? → 通过 (macOS/Linux 回退)

2. 检查 API 可用性 (isVoiceStreamAvailable)
   └── OAuth 认证? → 通过
   └── 访问令牌有效? → 通过

3. 检查音频依赖 (checkVoiceDependencies)
   └── 原生音频模块可用? → 通过
   └── 提供安装命令提示

4. 请求麦克风权限 (requestMicrophonePermission)
   └── 触发系统权限对话框
   └── 被拒绝? → 提供平台特定指引
```

### 语言提示机制

- 首次启用或语言变更时显示当前听写语言
- 最多显示 2 次提示（`LANG_HINT_MAX_SHOWS = 2`）
- 不支持的语言回退到英语并提示用户

## 具体技术实现

### 关键常量

```typescript
const LANG_HINT_MAX_SHOWS = 2  // 语言提示最大显示次数
```

### 导入依赖分析

```typescript
// 语音相关
import { normalizeLanguageForSTT } from '../../hooks/useVoice.js'
import { isVoiceModeEnabled } from '../../voice/voiceModeEnabled.js'

// 快捷键显示
import { getShortcutDisplay } from '../../keybindings/shortcutFormat.js'

// 分析追踪
import { logEvent } from '../../services/analytics/index.js'

// 类型定义
import type { LocalCommandCall } from '../../types/command.js'

// 认证与配置
import { isAnthropicAuthEnabled } from '../../utils/auth.js'
import { getGlobalConfig, saveGlobalConfig } from '../../utils/config.js'

// 设置管理
import { settingsChangeDetector } from '../../utils/settings/changeDetector.js'
import { getInitialSettings, updateSettingsForSource } from '../../utils/settings/settings.js'

// 语音服务（动态导入）
const { isVoiceStreamAvailable } = await import('../../services/voiceStreamSTT.js')
const { checkRecordingAvailability, checkVoiceDependencies, requestMicrophonePermission } = await import('../../services/voice.js')
```

### 核心函数: `call`

```typescript
export const call: LocalCommandCall = async () => {
  // 1. 语音模式全局检查
  if (!isVoiceModeEnabled()) {
    if (!isAnthropicAuthEnabled()) {
      return { type: 'text', value: 'Voice mode requires a Claude.ai account...' }
    }
    return { type: 'text', value: 'Voice mode is not available.' }
  }

  // 2. 获取当前设置，判断是开启还是关闭
  const currentSettings = getInitialSettings()
  const isCurrentlyEnabled = currentSettings.voiceEnabled === true

  // 3. Toggle OFF 逻辑
  if (isCurrentlyEnabled) {
    const result = updateSettingsForSource('userSettings', { voiceEnabled: false })
    settingsChangeDetector.notifyChange('userSettings')
    logEvent('tengu_voice_toggled', { enabled: false })
    return { type: 'text', value: 'Voice mode disabled.' }
  }

  // 4. Toggle ON 预检流程
  // ... 录音可用性、API 可用性、依赖检查、麦克风权限

  // 5. 启用语音模式
  const result = updateSettingsForSource('userSettings', { voiceEnabled: true })
  settingsChangeDetector.notifyChange('userSettings')
  logEvent('tengu_voice_toggled', { enabled: true })
  
  // 6. 构建返回消息（含快捷键和语言提示）
  const key = getShortcutDisplay('voice:pushToTalk', 'Chat', 'Space')
  const stt = normalizeLanguageForSTT(currentSettings.language)
  // ... 语言提示逻辑
  
  return { type: 'text', value: `Voice mode enabled. Hold ${key} to record.${langNote}` }
}
```

### 动态导入策略

语音相关服务使用动态导入，原因：
1. **避免启动时加载**：原生音频模块加载耗时（~1-8s）
2. **按需初始化**：仅在 Toggle ON 时加载
3. **错误隔离**：加载失败不会导致命令注册失败

```typescript
// Toggle ON 时才导入
const { isVoiceStreamAvailable } = await import('../../services/voiceStreamSTT.js')
const { checkRecordingAvailability } = await import('../../services/voice.js')
```

### 语言提示实现细节

```typescript
const stt = normalizeLanguageForSTT(currentSettings.language)
const cfg = getGlobalConfig()

// 检测语言是否变更
const langChanged = cfg.voiceLangHintLastLanguage !== stt.code
const priorCount = langChanged ? 0 : (cfg.voiceLangHintShownCount ?? 0)
const showHint = !stt.fellBackFrom && priorCount < LANG_HINT_MAX_SHOWS

// 构建提示文本
let langNote = ''
if (stt.fellBackFrom) {
  langNote = ` Note: "${stt.fellBackFrom}" is not a supported dictation language; using English.`
} else if (showHint) {
  langNote = ` Dictation language: ${stt.code} (/config to change).`
}

// 更新全局配置（提示计数器）
if (langChanged || showHint) {
  saveGlobalConfig(prev => ({
    ...prev,
    voiceLangHintShownCount: priorCount + (showHint ? 1 : 0),
    voiceLangHintLastLanguage: stt.code,
  }))
}
```

## 关键代码路径与文件引用

### 调用链

```
用户输入 /voice
    ↓
src/commands/voice/index.ts: load() → import('./voice.js')
    ↓
src/commands/voice/voice.ts: call()
    ↓
isVoiceModeEnabled() - 全局启用检查
    ↓
getInitialSettings() - 读取当前设置
    ↓
[Toggle OFF] updateSettingsForSource('userSettings', { voiceEnabled: false })
    ↓
settingsChangeDetector.notifyChange('userSettings')
    ↓
返回 "Voice mode disabled."

[Toggle ON] checkRecordingAvailability()
    ↓
isVoiceStreamAvailable()
    ↓
checkVoiceDependencies()
    ↓
requestMicrophonePermission()
    ↓
updateSettingsForSource('userSettings', { voiceEnabled: true })
    ↓
settingsChangeDetector.notifyChange('userSettings')
    ↓
返回 "Voice mode enabled. Hold <key> to record."
```

### 相关文件

| 文件路径 | 作用 |
|----------|------|
| `src/commands/voice/index.ts` | 命令入口，懒加载本模块 |
| `src/voice/voiceModeEnabled.ts` | 语音模式启用状态检查 |
| `src/hooks/useVoice.ts` | 语音功能 React Hook，含语言标准化 |
| `src/services/voiceStreamSTT.ts` | WebSocket STT 服务 |
| `src/services/voice.ts` | 录音功能服务 |
| `src/keybindings/shortcutFormat.ts` | 快捷键显示格式化 |
| `src/utils/settings/settings.ts` | 设置读写操作 |
| `src/utils/settings/changeDetector.ts` | 设置变更通知 |
| `src/utils/config.ts` | 全局配置管理（提示计数器） |
| `src/utils/auth.ts` | 认证状态检查 |

## 依赖与外部交互

### 内部服务依赖

1. **语音服务 (`src/services/voice.ts`)**
   - `checkRecordingAvailability()`: 检查录音环境
   - `checkVoiceDependencies()`: 检查音频工具依赖
   - `requestMicrophonePermission()`: 请求麦克风权限

2. **STT 服务 (`src/services/voiceStreamSTT.ts`)**
   - `isVoiceStreamAvailable()`: 检查 OAuth 和访问令牌

3. **设置系统 (`src/utils/settings/`)**
   - `getInitialSettings()`: 读取合并后的设置
   - `updateSettingsForSource()`: 更新指定源设置
   - `settingsChangeDetector.notifyChange()`: 通知设置变更

4. **分析追踪 (`src/services/analytics/index.ts`)**
   - `logEvent('tengu_voice_toggled', { enabled })`: 追踪开关事件

### 外部系统交互

| 系统 | 交互方式 | 目的 |
|------|----------|------|
| 操作系统 | 麦克风权限 API | 触发权限对话框 |
| GrowthBook | `isVoiceGrowthBookEnabled()` | 特性开关 |
| Claude.ai OAuth | `isVoiceStreamAvailable()` | 验证用户身份 |
| 全局配置存储 | `getGlobalConfig/saveGlobalConfig` | 提示计数持久化 |

## 风险、边界与改进建议

### 风险点

1. **动态导入失败处理**
   ```typescript
   // 当前代码未处理导入失败
   const { isVoiceStreamAvailable } = await import('../../services/voiceStreamSTT.js')
   // 如果模块加载失败，会抛出未捕获的异常
   ```
   **建议**: 添加 try-catch 包裹动态导入

2. **设置更新竞态条件**
   ```typescript
   const result = updateSettingsForSource('userSettings', { voiceEnabled: true })
   if (result.error) { /* 处理错误 */ }
   settingsChangeDetector.notifyChange('userSettings')
   ```
   如果更新失败，通知仍会发送，可能导致状态不一致。

3. **全局配置读写**
   ```typescript
   const cfg = getGlobalConfig()
   saveGlobalConfig(prev => ({ ... }))
   ```
   非原子操作，并发调用可能丢失更新。

### 边界情况

| 场景 | 当前行为 | 潜在问题 |
|------|----------|----------|
| 快速连续调用 /voice | 每次调用独立执行 | 可能产生竞态条件 |
| 设置文件损坏 | `updateSettingsForSource` 返回错误 | 用户看到通用错误信息 |
| 麦克风权限被拒绝 | 返回平台特定指引 | Windows/Linux/macOS 指引可能过时 |
| 网络断开时启用 | `isVoiceStreamAvailable()` 可能返回 false | 用户可能困惑为何需要网络 |

### 改进建议

1. **增强错误处理**
   ```typescript
   try {
     const voiceModule = await import('../../services/voice.js')
     // ...
   } catch (error) {
     logError(error)
     return {
       type: 'text',
       value: 'Failed to load voice module. Please try again or restart Claude Code.',
     }
   }
   ```

2. **添加操作防抖**
   ```typescript
   let isToggling = false
   
   export const call: LocalCommandCall = async () => {
     if (isToggling) {
       return { type: 'text', value: 'Voice mode toggle in progress...' }
     }
     isToggling = true
     try {
       // ... 原有逻辑
     } finally {
       isToggling = false
     }
   }
   ```

3. **细化错误信息**
   ```typescript
   if (result.error) {
     if (result.error.message.includes('JSON syntax')) {
       return {
         type: 'text',
         value: 'Failed to update settings: Your settings.json has a syntax error. Please fix it and try again.',
       }
     }
     // ...
   }
   ```

4. **提取预检逻辑**
   当前预检逻辑较长，可提取为独立函数：
   ```typescript
   async function runPreflightChecks(): Promise<
     | { success: true }
     | { success: false; reason: string }
   > {
     // ... 预检逻辑
   }
   ```

5. **国际化支持**
   当前错误信息和提示均为英文，考虑添加多语言支持：
   ```typescript
   const messages = {
     en: { voiceEnabled: 'Voice mode enabled...' },
     zh: { voiceEnabled: '语音模式已开启...' },
     // ...
   }
   ```

6. **配置持久化优化**
   考虑使用事务性更新避免竞态：
   ```typescript
   await saveGlobalConfigAtomic(prev => ({
     ...prev,
     voiceLangHintShownCount: priorCount + (showHint ? 1 : 0),
     voiceLangHintLastLanguage: stt.code,
   }))
   ```
