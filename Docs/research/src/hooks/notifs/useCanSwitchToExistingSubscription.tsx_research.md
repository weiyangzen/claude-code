# useCanSwitchToExistingSubscription.tsx 深度研究

## 场景与职责

`useCanSwitchToExistingSubscription` 是一个 React Hook，用于检测用户是否拥有 Claude AI 订阅（Claude Pro 或 Claude Max）但当前未使用该订阅登录。如果检测到这种情况，会向用户显示提示通知，建议运行 `/login` 命令来激活订阅。

### 核心场景
1. **订阅未激活提示**：用户通过 API Key 或其他方式使用 Claude Code，但账号实际上有付费订阅
2. **引导登录**：提示用户使用 `/login` 命令切换到订阅账号，以获得更好的服务体验
3. **频率控制**：限制通知显示次数（最多 3 次），避免过度打扰

## 功能点目的

### 1. 订阅状态检测
- 检查用户当前是否已使用订阅认证 (`isClaudeAISubscriber()`)
- 通过 OAuth Profile API 获取用户账号的订阅信息
- 检测账号是否拥有 `has_claude_max` 或 `has_claude_pro`

### 2. 智能频率控制
- 使用 `MAX_SHOW_COUNT = 3` 限制通知显示次数
- 通过 `subscriptionNoticeCount` 在全局配置中持久化计数
- 达到上限后不再显示

### 3. 用户引导
- 通知中包含 `/login` 命令提示
- 显示具体的订阅类型（Pro 或 Max）
- 使用 JSX 渲染带样式的通知内容

## 具体技术实现

### 关键数据结构

```typescript
// OAuth Profile 响应类型
interface OAuthProfileResponse {
  account: {
    has_claude_max: boolean
    has_claude_pro: boolean
    // ... 其他字段
  }
}

// 全局配置中的订阅通知计数
type GlobalConfig = {
  subscriptionNoticeCount?: number
  // ...
}
```

### 核心流程

```
useStartupNotification 初始化
    ↓
检查显示次数是否已达上限 (MAX_SHOW_COUNT)
    ↓
获取现有订阅类型 (getExistingClaudeSubscription)
    - 检查是否已是订阅用户 (isClaudeAISubscriber)
    - 获取 OAuth Profile
    - 检查 has_claude_max / has_claude_pro
    ↓
如果检测到订阅
    ↓
增加显示计数 (saveGlobalConfig)
    ↓
记录分析事件 (logEvent)
    ↓
返回通知对象
```

### 关键代码路径

```typescript
// 主 Hook 实现
export function useCanSwitchToExistingSubscription() {
  useStartupNotification(async () => {
    // 频率检查
    if ((getGlobalConfig().subscriptionNoticeCount ?? 0) >= MAX_SHOW_COUNT) {
      return null
    }
    
    // 获取订阅类型
    const subscriptionType = await getExistingClaudeSubscription()
    if (subscriptionType === null) {
      return null
    }
    
    // 更新计数
    saveGlobalConfig(current => ({
      ...current,
      subscriptionNoticeCount: (current.subscriptionNoticeCount ?? 0) + 1
    }))
    
    // 记录事件
    logEvent("tengu_switch_to_subscription_notice_shown", {})
    
    // 返回通知
    return {
      key: "switch-to-subscription",
      jsx: <Text color="suggestion">
        Use your existing Claude {subscriptionType} plan with Claude Code
        <Text color="text" dimColor={true}> · /login to activate</Text>
      </Text>,
      priority: "low"
    }
  })
}

// 订阅检测逻辑
async function getExistingClaudeSubscription(): Promise<'Max' | 'Pro' | null> {
  // 如果已经是订阅用户，无需提示
  if (isClaudeAISubscriber()) {
    return null
  }
  
  const profile = await getOauthProfileFromApiKey()
  if (!profile) {
    return null
  }
  
  if (profile.account.has_claude_max) {
    return 'Max'
  }
  if (profile.account.has_claude_pro) {
    return 'Pro'
  }
  return null
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `React` | `react` | JSX 渲染 |
| `getOauthProfileFromApiKey` | `src/services/oauth/getOauthProfile.js` | 获取 OAuth Profile |
| `isClaudeAISubscriber` | `src/utils/auth.js` | 检查当前是否为订阅用户 |
| `Text` | `src/ink.js` | Ink 文本组件 |
| `logEvent` | `src/services/analytics/index.js` | 分析事件记录 |
| `getGlobalConfig`, `saveGlobalConfig` | `src/utils/config.js` | 全局配置读写 |
| `useStartupNotification` | `./useStartupNotification.js` | 启动通知基类 |

### 依赖模块详解

#### 1. getOauthProfileFromApiKey (src/services/oauth/getOauthProfile.ts)
```typescript
export async function getOauthProfileFromApiKey(): Promise<OAuthProfileResponse | undefined> {
  const config = getGlobalConfig()
  const accountUuid = config.oauthAccount?.accountUuid
  const apiKey = getAnthropicApiKey()
  
  if (!accountUuid || !apiKey) {
    return
  }
  
  const endpoint = `${getOauthConfig().BASE_API_URL}/api/claude_cli_profile`
  const response = await axios.get<OAuthProfileResponse>(endpoint, {
    headers: {
      'x-api-key': apiKey,
      'anthropic-beta': OAUTH_BETA_HEADER,
    },
    params: { account_uuid: accountUuid },
    timeout: 10000,
  })
  return response.data
}
```

#### 2. isClaudeAISubscriber (src/utils/auth.ts)
检查当前认证状态是否为 Claude AI 订阅用户，通过检查 OAuth Token 的作用域来判断。

#### 3. useStartupNotification (src/hooks/notifs/useStartupNotification.ts)
提供启动时一次性通知的基础设施：
- 远程模式检查
- 单次执行保证（useRef）
- 异步计算支持
- 错误处理

## 风险、边界与改进建议

### 潜在风险

1. **API 调用失败**
   - `getOauthProfileFromApiKey` 是网络请求，可能超时或失败
   - 当前实现静默失败（返回 null），用户可能错过重要提示
   - 建议：增加重试机制或离线缓存

2. **隐私考虑**
   - 每次启动都会调用 OAuth Profile API
   - 即使用户没有订阅，也会产生 API 调用
   - 建议：增加缓存机制，减少不必要的调用

3. **计数器持久化**
   - `subscriptionNoticeCount` 存储在全局配置中
   - 如果用户删除配置或在新设备上使用，计数会重置

### 边界情况

1. **API Key 用户**
   - 如果用户使用 API Key 而非 OAuth，可能无法获取 Profile
   - 这种情况下不会显示通知（预期行为）

2. **网络不可用**
   - 离线状态下无法检测订阅状态
   - 静默失败，不显示通知

3. **多账号场景**
   - 如果用户有多个 Claude 账号，可能在一个账号有订阅，另一个没有
   - 检测基于当前使用的 API Key

### 改进建议

1. **增加缓存机制**
   ```typescript
   // 建议：缓存订阅检测结果
   const SUBSCRIPTION_CACHE_TTL = 24 * 60 * 60 * 1000 // 24小时
   
   function getCachedSubscriptionStatus() {
     const cached = getGlobalConfig().subscriptionCheckCache
     if (cached && Date.now() - cached.timestamp < SUBSCRIPTION_CACHE_TTL) {
       return cached.result
     }
     return null
   }
   ```

2. **增加错误日志**
   ```typescript
   // 建议添加调试日志
   logForDebugging(`[SubscriptionCheck] API call failed: ${error.message}`)
   ```

3. **考虑添加"不再提示"选项**
   - 当前只能显示 3 次
   - 可以添加用户主动关闭的选项

4. **优化通知时机**
   - 当前在启动时立即检查
   - 可以考虑延迟到用户完成首次交互后再显示

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useCanSwitchToExistingSubscription.tsx`
- **启动通知基类**: `src/hooks/notifs/useStartupNotification.ts`
- **OAuth Profile 服务**: `src/services/oauth/getOauthProfile.ts`
- **认证工具**: `src/utils/auth.ts`
- **全局配置**: `src/utils/config.ts`
- **分析服务**: `src/services/analytics/index.ts`
- **Ink 组件**: `src/ink.js`
