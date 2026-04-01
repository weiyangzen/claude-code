# Referral 服务研究文档

## 文件信息
- **路径**: `src/services/api/referral.ts`
- **大小**: 7,985 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Referral 服务负责管理 Claude Code 的 **Guest Passes（访客通行证）推荐系统**，向符合条件的 Max 订阅用户展示推荐功能。该服务处理以下核心场景：

1. **资格检查**: 检查用户是否有资格参与 Guest Passes 推荐计划
2. **推荐记录查询**: 获取用户的推荐兑换记录
3. **缓存管理**: 24 小时 TTL 的按组织缓存，支持后台刷新
4. **奖励信息展示**: 提供格式化后的推荐奖励金额显示
5. **启动预加载**: 应用启动时预加载资格数据

---

## 功能点目的

### 1. 资格获取 (`fetchReferralEligibility`)
从后端 API 获取用户的推荐资格信息：
- 端点: `/api/oauth/organizations/{orgUUID}/referral/eligibility`
- 支持 campaign 参数（默认 `claude_code_guest_pass`）
- 5 秒超时，适合后台调用

### 2. 兑换记录获取 (`fetchReferralRedemptions`)
获取用户的推荐兑换记录：
- 端点: `/api/oauth/organizations/{orgUUID}/referral/redemptions`
- 10 秒超时
- 用于 `/passes` 命令展示详细记录

### 3. 缓存管理

#### 检查缓存状态 (`checkCachedPassesEligibility`)
同步检查缓存状态，返回：
- `eligible`: 是否有资格
- `needsRefresh`: 是否需要刷新
- `hasCache`: 是否有缓存

#### 获取或获取 (`getCachedOrFetchPassesEligibility`)
主入口函数，实现非阻塞缓存策略：
- 无缓存 → 后台获取，返回 null
- 缓存过期 → 返回旧值 + 后台刷新
- 缓存有效 → 直接返回

#### 获取并存储 (`fetchAndStorePassesEligibility`)
实际 API 调用和缓存更新：
- 使用 `fetchInProgress` 防止重复调用
- 存储完整响应 + 时间戳
- 按 `organizationUuid` 分区

### 4. 奖励信息获取

#### 获取奖励金额 (`getCachedReferrerReward`)
从缓存获取推荐人奖励信息（用于 v1 campaign）。

#### 获取剩余通行证数 (`getCachedRemainingPasses`)
从缓存获取剩余可推荐通行证数量。

#### 格式化金额 (`formatCreditAmount`)
支持多货币格式化（USD, EUR, GBP, BRL, CAD, AUD, NZD, SGD）。

### 5. 启动预加载 (`prefetchPassesEligibility`)
应用启动时触发预加载：
- 受 `isEssentialTrafficOnly()` 保护
- 使用 `void` 忽略 Promise，不阻塞启动

---

## 具体技术实现

### 关键数据结构

```typescript
// 来自 services/oauth/types.ts
type ReferralEligibilityResponse = {
  eligible: boolean
  remaining_passes: number
  referrer_reward?: ReferrerRewardInfo
  // ... 其他字段
}

type ReferralRedemptionsResponse = {
  redemptions: Array<{
    redeemed_by: string
    redeemed_at: string
    // ...
  }>
}

type ReferrerRewardInfo = {
  amount_minor_units: number
  currency: string
}

// 内部类型
type ReferralCampaign = 'claude_code_guest_pass' | string

// 缓存配置
const CACHE_EXPIRATION_MS = 24 * 60 * 60 * 1000  // 24 小时
```

### 关键流程

#### 非阻塞获取流程
```
getCachedOrFetchPassesEligibility()
├── 预检查 shouldCheckForPasses()
│   ├── 需要 organizationUuid
│   ├── 需要 isClaudeAISubscriber()
│   └── 需要 subscriptionType === 'max'
├── 检查缓存
│   ├── 无缓存 → 后台 fetch，返回 null
│   ├── 过期 → 返回旧值 + 后台 fetch
│   └── 有效 → 返回缓存值
└── 返回 ReferralEligibilityResponse | null
```

#### 防重复调用机制
```typescript
let fetchInProgress: Promise<ReferralEligibilityResponse | null> | null = null

async function fetchAndStorePassesEligibility() {
  if (fetchInProgress) {
    return fetchInProgress  // 复用进行中的请求
  }
  
  fetchInProgress = (async () => {
    try {
      const response = await fetchReferralEligibility()
      // ... 存储到缓存
      return response
    } finally {
      fetchInProgress = null  // 清理
    }
  })()
  
  return fetchInProgress
}
```

### 货币格式化

```typescript
const CURRENCY_SYMBOLS: Record<string, string> = {
  USD: '$',
  EUR: '€',
  GBP: '£',
  BRL: 'R$',
  CAD: 'CA$',
  AUD: 'A$',
  NZD: 'NZ$',
  SGD: 'S$',
}

export function formatCreditAmount(reward: ReferrerRewardInfo): string {
  const symbol = CURRENCY_SYMBOLS[reward.currency] ?? `${reward.currency} `
  const amount = reward.amount_minor_units / 100
  const formatted = amount % 1 === 0 ? amount.toString() : amount.toFixed(2)
  return `${symbol}${formatted}`
}
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/referral.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/main.tsx` | `prefetchPassesEligibility` | 启动预加载 |
| `src/components/Passes/Passes.tsx` | `fetchReferralRedemptions`, `formatCreditAmount`, `getCachedOrFetchPassesEligibility` | /passes 命令页面 |
| `src/components/LogoV2/GuestPassesUpsell.tsx` | `checkCachedPassesEligibility`, `formatCreditAmount`, `getCachedReferrerReward`, `getCachedRemainingPasses` | Logo 旁 upsell |
| `src/components/LogoV2/feedConfigs.tsx` | `formatCreditAmount`, `getCachedReferrerReward` | Feed 配置 |
| `src/commands/passes/index.ts` | 类型导入 | Passes 命令入口 |
| `src/commands/passes/passes.tsx` | `getCachedRemainingPasses` | Passes 命令实现 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/services/oauth/types.ts` | `ReferralEligibilityResponse`, `ReferrerRewardInfo` 等类型 |
| `src/utils/auth.ts` | `getOauthAccountInfo`, `isClaudeAISubscriber`, `getSubscriptionType` |
| `src/utils/config.ts` | `getGlobalConfig`, `saveGlobalConfig` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/log.ts` | `logError` |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly` |
| `src/utils/teleport/api.ts` | `getOAuthHeaders`, `prepareApiRequest` |
| `src/constants/oauth.ts` | `getOauthConfig` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 认证 | 超时 | 用途 |
|------|------|------|------|------|
| `/api/oauth/organizations/{orgUUID}/referral/eligibility` | GET | OAuth + x-organization-uuid | 5s | 获取资格 |
| `/api/oauth/organizations/{orgUUID}/referral/redemptions` | GET | OAuth + x-organization-uuid | 10s | 获取兑换记录 |

### 请求头
```typescript
{
  'Authorization': 'Bearer {accessToken}',
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'x-organization-uuid': orgUUID
}
```

### 配置存储

缓存存储在 `~/.claude/config.json`：
```json
{
  "passesEligibilityCache": {
    "org-uuid-1": {
      "eligible": true,
      "remaining_passes": 3,
      "referrer_reward": {
        "amount_minor_units": 2000,
        "currency": "USD"
      },
      "timestamp": 1712345678901
    }
  }
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **订阅类型硬编码**
   ```typescript
   function shouldCheckForPasses(): boolean {
     return getSubscriptionType() === 'max'
   }
   ```
   - 仅 Max 订阅者可使用，Pro 或其他 tier 被排除
   - 如果后端扩展资格标准，前端需要同步更新

2. **无数据不变性检查**
   - 与 `overageCreditGrant.ts` 不同，直接覆盖缓存
   - 可能导致不必要的磁盘写入

3. **全局 fetchInProgress**
   - 单例模式，不区分 campaign 或组织
   - 多组织用户快速切换时可能有问题

### 边界情况

| 场景 | 行为 |
|------|------|
| `isEssentialTrafficOnly()` | `prefetchPassesEligibility` 直接返回 |
| 非 Max 订阅者 | `shouldCheckForPasses()` 返回 false，所有函数返回 null/false |
| 无 orgId | 返回 null/false |
| API 错误 | `fetchAndStorePassesEligibility` 返回 null，不更新缓存 |
| 并发调用 | `fetchInProgress` 确保只发出一个请求 |
| 未知货币 | `formatCreditAmount` 使用货币代码前缀（如 "JPY 100"）|

### 改进建议

1. **数据不变性检查**
   ```typescript
   // 参考 overageCreditGrant.ts 模式
   const dataUnchanged = 
     cached.eligible === response.eligible &&
     cached.remaining_passes === response.remaining_passes
   
   if (dataUnchanged && Date.now() - cached.timestamp <= CACHE_EXPIRATION_MS) {
     return prev  // 跳过写入
   }
   ```

2. **按 campaign 分区**
   ```typescript
   // 当前缓存键仅按 orgId
   passesEligibilityCache?: Record<string, CacheEntry>
   
   // 建议支持多 campaign
   passesEligibilityCache?: Record<string, Record<string, CacheEntry>>
   // 外层 key: orgId, 内层 key: campaign
   ```

3. **错误重试**
   - 当前无重试逻辑
   - 建议添加指数退避重试（参考 `sessionIngress.ts`）

4. **Analytics 事件**
   ```typescript
   // 建议添加事件
   logEvent('tengu_guest_passes_shown', {
     eligible,
     remaining_passes,
     has_reward: !!referrer_reward
   })
   ```

5. **订阅类型动态获取**
   - 当前硬编码 'max'
   - 建议从后端 eligibility 响应中读取资格标准

6. **测试覆盖**
   - 添加 `fetchInProgress` 竞态条件测试
   - 测试多货币格式化
   - 测试缓存过期和后台刷新逻辑

### 相关模式对比

| 服务 | 缓存策略 | 防重复调用 | 数据不变性检查 |
|------|---------|-----------|--------------|
| `referral.ts` | 磁盘，24h，按组织 | ✅ fetchInProgress | ❌ |
| `overageCreditGrant.ts` | 磁盘，1h，按组织 | ❌ | ✅ |
| `grove.ts` | 内存+磁盘，24h，按账户 | ✅ memoize | ❌ |

`referral.ts` 的 `fetchInProgress` 模式适合防止并发请求，但缺乏数据不变性检查可能导致不必要的写入。
