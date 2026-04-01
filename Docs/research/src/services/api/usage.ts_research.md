# Usage 服务研究文档

## 文件信息
- **路径**: `src/services/api/usage.ts`
- **大小**: 1,685 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Usage 服务负责管理 Claude Code 的 **用户使用情况查询**，用于展示用户的 API 使用率和配额信息。该服务处理以下核心场景：

1. **使用率查询**: 获取用户的 API 使用率（5小时、7天窗口）
2. **超额使用查询**: 获取 Extra Usage 功能的启用状态和额度
3. **OAuth 权限检查**: 需要 `profile` scope 才能访问
4. **Token 过期检查**: 避免使用过期 token 发送请求

---

## 功能点目的

### 1. 使用率获取 (`fetchUtilization`)
从后端 API 获取用户使用情况：
- 端点: `/api/oauth/usage`
- 需要 `isClaudeAISubscriber() && hasProfileScope()`
- Token 过期时返回 null
- 5 秒超时

### 2. 数据结构
返回的使用率信息包括：
- **5 小时窗口**: 短期使用率
- **7 天窗口**: 长期使用率（按模型细分：Opus、Sonnet、OAuth Apps）
- **Extra Usage**: 超额使用功能的配置和消耗

---

## 具体技术实现

### 关键数据结构

```typescript
// 使用率限制
type RateLimit = {
  utilization: number | null  // 使用率百分比 (0-100)
  resets_at: string | null    // ISO 8601 重置时间
}

// 超额使用配置
type ExtraUsage = {
  is_enabled: boolean         // 是否启用
  monthly_limit: number | null // 月度限额
  used_credits: number | null  // 已使用额度
  utilization: number | null   // 使用率
}

// 完整使用率响应
type Utilization = {
  five_hour?: RateLimit | null
  seven_day?: RateLimit | null
  seven_day_oauth_apps?: RateLimit | null
  seven_day_opus?: RateLimit | null
  seven_day_sonnet?: RateLimit | null
  extra_usage?: ExtraUsage | null
}
```

### 关键流程

```
fetchUtilization()
├── 权限检查
│   ├── !isClaudeAISubscriber() → 返回 {}
│   └── !hasProfileScope() → 返回 {}
├── Token 过期检查
│   ├── 获取 tokens
│   └── isOAuthTokenExpired(expiresAt) → true 返回 null
├── 构建认证头
│   └── getAuthHeaders()
├── 发送 GET 请求
│   ├── 端点: /api/oauth/usage
│   ├── 认证: OAuth Bearer
│   └── 超时: 5s
└── 返回 Utilization
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/usage.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/commands/review/reviewRemote.ts` | `fetchUtilization` | 审查前检查使用率 |
| `src/commands/extra-usage/extra-usage-core.ts` | `fetchUtilization`, `ExtraUsage` | /extra-usage 命令 |
| `src/components/Settings/Usage.tsx` | `fetchUtilization`, `RateLimit`, `Utilization`, `ExtraUsage` | 设置页面使用率展示 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/auth.ts` | `getClaudeAIOAuthTokens`, `hasProfileScope`, `isClaudeAISubscriber`, `getAuthHeaders` |
| `src/utils/http.ts` | `getClaudeCodeUserAgent` |
| `src/services/oauth/client.ts` | `isOAuthTokenExpired` |
| `src/constants/oauth.ts` | `getOauthConfig` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 认证 | 超时 | 用途 |
|------|------|------|------|------|
| `/api/oauth/usage` | GET | OAuth Bearer | 5s | 获取使用率信息 |

### 请求头
```typescript
{
  'Content-Type': 'application/json',
  'User-Agent': getClaudeCodeUserAgent(),
  'Authorization': 'Bearer {accessToken}'
}
```

### 权限要求

| 条件 | 不满足时的行为 |
|------|---------------|
| `isClaudeAISubscriber()` | 返回 `{}`（空对象）|
| `hasProfileScope()` | 返回 `{}`（空对象）|
| Token 未过期 | 正常调用 API |
| Token 已过期 | 返回 `null` |

---

## 风险、边界与改进建议

### 已知风险

1. **权限检查分散**
   - `isClaudeAISubscriber()` 和 `hasProfileScope()` 检查在函数开头
   - 但 `getAuthHeaders()` 内部也有类似检查
   - 可能导致重复逻辑

2. **Token 过期检查时机**
   ```typescript
   if (tokens && isOAuthTokenExpired(tokens.expiresAt)) {
     return null
   }
   ```
   - 在 `getAuthHeaders()` 之前检查
   - 但 `getAuthHeaders()` 内部可能获取新的 token

3. **无缓存机制**
   - 每次调用都请求 API
   - 设置页面可能频繁刷新

4. **错误处理不一致**
   - 权限不足返回 `{}`
   - Token 过期返回 `null`
   - 其他错误抛出异常

### 边界情况

| 场景 | 行为 |
|------|------|
| 非订阅者 | 返回 `{}` |
| 无 profile scope | 返回 `{}` |
| Token 过期 | 返回 `null` |
| `getAuthHeaders()` 失败 | 抛出错误 |
| API 超时（5s） | 抛出错误 |
| API 401 | 抛出错误（axios 默认行为）|
| API 其他错误 | 抛出错误 |

### 改进建议

1. **统一返回类型**
   ```typescript
   // 当前：Utilization | null | {}
   // 建议：统一为 Utilization | null
   export async function fetchUtilization(): Promise<Utilization | null> {
     if (!isClaudeAISubscriber() || !hasProfileScope()) {
       return null  // 或返回默认空对象
     }
     // ...
   }
   ```

2. **添加缓存机制**
   ```typescript
   const CACHE_TTL_MS = 60 * 1000  // 1 分钟
   
   let cachedUtilization: { data: Utilization; timestamp: number } | null = null
   
   export async function fetchUtilization(): Promise<Utilization | null> {
     if (cachedUtilization && Date.now() - cachedUtilization.timestamp < CACHE_TTL_MS) {
       return cachedUtilization.data
     }
     // ... 获取并缓存
   }
   ```

3. **Token 刷新集成**
   ```typescript
   // 当前仅检查过期，不尝试刷新
   // 建议：
   if (isOAuthTokenExpired(tokens.expiresAt)) {
     const refreshed = await refreshOAuthToken()
     if (!refreshed) return null
   }
   ```

4. **Analytics 事件**
   ```typescript
   logEvent('tengu_usage_fetched', {
     has_five_hour: !!data.five_hour,
     has_seven_day: !!data.seven_day,
     has_extra_usage: !!data.extra_usage,
     extra_usage_enabled: data.extra_usage?.is_enabled
   })
   ```

5. **错误分类**
   ```typescript
   try {
     return await axios.get(...)
   } catch (error) {
     if (axios.isAxiosError(error)) {
       if (error.response?.status === 401) {
         return null  // 认证失败
       }
       if (error.code === 'ECONNABORTED') {
         return null  // 超时
       }
     }
     throw error  // 其他错误
   }
   ```

6. **使用 `withOAuth401Retry`**
   - 当前未使用 `withOAuth401Retry`
   - 建议添加以处理时钟漂移导致的 401

### 相关模式对比

| 服务 | 权限检查 | Token 过期处理 | 缓存 | 重试 |
|------|---------|---------------|------|------|
| `usage.ts` | 前置检查 | 返回 null | ❌ | ❌ |
| `metricsOptOut.ts` | 前置检查 | `withOAuth401Retry` | ✅ | ✅ |
| `ultrareviewQuota.ts` | 前置检查 | 抛出错误 | ❌ | ❌ |
| `referral.ts` | 前置检查 | 返回 null | ✅ | ❌ |

`usage.ts` 的实现相对简单，缺乏缓存和重试机制，适合低频调用场景（如设置页面加载）。
