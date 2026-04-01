# useVoiceEnabled.ts 研究文档

## 场景与职责

`useVoiceEnabled` 是一个轻量级的 React Hook，用于确定语音功能是否应该在当前会话中启用。它综合了三个因素：

1. **用户意图**：用户是否在设置中启用了语音功能
2. **认证状态**：用户是否有有效的 Anthropic OAuth 令牌
3. **功能开关**：GrowthBook 是否启用了语音功能（kill-switch）

该 Hook 被多个组件使用，包括 `useVoiceIntegration.tsx`、`PromptInputFooterLeftSide.tsx` 等，用于条件渲染语音相关的 UI 和功能。

## 功能点目的

### 1. 三因素综合判断

```typescript
export function useVoiceEnabled(): boolean {
  const userIntent = useAppState(s => s.settings.voiceEnabled === true)
  const authVersion = useAppState(s => s.authVersion)
  const authed = useMemo(hasVoiceAuth, [authVersion])
  return userIntent && authed && isVoiceGrowthBookEnabled()
}
```

- **userIntent**：用户主动开启语音功能（`settings.voiceEnabled === true`）
- **authed**：用户已通过 Anthropic OAuth 认证
- **isVoiceGrowthBookEnabled()**：GrowthBook 功能开关未关闭

### 2. 性能优化

```typescript
// eslint-disable-next-line react-hooks/exhaustive-deps
const authed = useMemo(hasVoiceAuth, [authVersion])
```

- `hasVoiceAuth` 调用 `getClaudeAIOAuthTokens`，后者可能触发同步的 `security` 子进程（约 60ms）
- 使用 `useMemo` 只在 `authVersion` 变化时重新计算
- `authVersion` 只在 `/login` 时递增，背景令牌刷新不递增
- GrowthBook 检查（`isVoiceGrowthBookEnabled()`）在 memo 之外，确保 kill-switch 能立即生效

## 具体技术实现

### 关键流程

1. **获取用户意图**：
   ```typescript
   const userIntent = useAppState(s => s.settings.voiceEnabled === true)
   ```
   使用 `useAppState` 从全局状态订阅 `settings.voiceEnabled`。

2. **获取认证版本**：
   ```typescript
   const authVersion = useAppState(s => s.authVersion)
   ```
   `authVersion` 是一个整数，每次用户登录时递增。

3. **计算认证状态**：
   ```typescript
   const authed = useMemo(hasVoiceAuth, [authVersion])
   ```
   只在 `authVersion` 变化时调用 `hasVoiceAuth()`。

4. **检查 GrowthBook**：
   ```typescript
   isVoiceGrowthBookEnabled()
   ```
   每次渲染都检查，确保 kill-switch 立即生效。

5. **返回综合结果**：
   ```typescript
   return userIntent && authed && isVoiceGrowthBookEnabled()
   ```

### 数据结构

- **输入**：无（Hook 无参数）
- **输出**：`boolean` - 语音功能是否启用
- **依赖状态**：
  - `AppState.settings.voiceEnabled`
  - `AppState.authVersion`
  - GrowthBook 功能开关

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useVoiceEnabled.ts` - Hook 实现

### 依赖文件
| 文件 | 用途 |
|------|------|
| `react` (useMemo) | React API |
| `../state/AppState.js` | 全局应用状态 |
| `../voice/voiceModeEnabled.js` | 语音启用检查函数 |

### 依赖的函数

#### hasVoiceAuth
来源：`../voice/voiceModeEnabled.js`

```typescript
export function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) {
    return false
  }
  const tokens = getClaudeAIOAuthTokens()
  return Boolean(tokens?.accessToken)
}
```

- 检查是否使用 Anthropic 认证（而非 API Key、Bedrock、Vertex 等）
- 检查是否有有效的访问令牌

#### isVoiceGrowthBookEnabled
来源：`../voice/voiceModeEnabled.js`

```typescript
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

- 检查编译时特性标志 `VOICE_MODE`
- 检查 GrowthBook 的 `tengu_amber_quartz_disabled` kill-switch
- 默认 `false` 确保新安装立即可用（无需等待 GrowthBook 初始化）

### 调用方

| 文件 | 用途 |
|------|------|
| `../hooks/useVoiceIntegration.tsx` | 决定是否启用语音 Hook |
| `../components/PromptInput/PromptInputFooterLeftSide.tsx` | 显示语音状态指示器 |
| `../components/PromptInput/Notifications.tsx` | 语音通知 |
| `../commands/voice/voice.ts` | `/voice` 命令 |
| `../tools/ConfigTool/ConfigTool.ts` | 配置工具 |

## 依赖与外部交互

### 全局状态

```typescript
type AppState = {
  settings: {
    voiceEnabled?: boolean
  }
  authVersion: number
}
```

### GrowthBook 集成

```typescript
// 特性标志（编译时）
feature('VOICE_MODE')  // 是否包含语音代码

// 功能开关（运行时）
getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
```

### 认证系统

```typescript
// 检查认证提供商
isAnthropicAuthEnabled(): boolean

// 获取 OAuth 令牌（可能触发 keychain 访问）
getClaudeAIOAuthTokens(): { accessToken: string } | null
```

## 风险、边界与改进建议

### 潜在风险

1. **性能问题**：
   - `hasVoiceAuth` 在冷缓存时可能触发同步子进程（~60ms）
   - 虽然使用 `useMemo`，但如果 `authVersion` 频繁变化，仍可能阻塞渲染

2. **GrowthBook 延迟**：
   - `getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期的缓存值
   - 首次启动时可能有一小段时间显示错误的启用状态

3. **状态不一致**：
   - `useVoiceEnabled` 返回 `true` 但后续 `connectVoiceStream` 可能失败
   - 因为 Hook 只检查令牌存在，不验证令牌有效性

### 边界情况

| 场景 | 行为 |
|------|------|
| 用户未登录 | `hasVoiceAuth()` 返回 `false`，语音禁用 |
| 用户使用 API Key | `isAnthropicAuthEnabled()` 返回 `false`，语音禁用 |
| GrowthBook 未初始化 | `isVoiceGrowthBookEnabled()` 默认返回 `true`（kill-switch 未触发）|
| 用户禁用语音设置 | `userIntent` 为 `false`，语音禁用 |
| 编译时未包含语音 | `feature('VOICE_MODE')` 为 `false`，语音禁用 |

### 改进建议

1. **异步认证检查**：
   - 当前 `hasVoiceAuth` 是同步的，可能阻塞
   - 可考虑添加异步版本，或使用 Suspense 模式

2. **令牌有效性验证**：
   - 当前只检查令牌存在
   - 可考虑添加轻量级的令牌有效性检查（如检查过期时间）

3. **缓存优化**：
   - `hasVoiceAuth` 内部有 memoize，但 `useMemo` 提供了额外的 React 层缓存
   - 可考虑移除一层缓存简化逻辑

4. **加载状态**：
   - 当前返回简单的 `boolean`
   - 可考虑返回 `{ enabled: boolean, loading: boolean }` 以便 UI 显示加载状态

5. **错误处理**：
   - 当前静默失败（返回 `false`）
   - 可考虑添加错误状态，用于调试和用户反馈

6. **测试覆盖**：
   - 添加单元测试覆盖各种组合情况
   - 测试 `authVersion` 变化时的重新计算行为
