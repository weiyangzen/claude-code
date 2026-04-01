# Overage Credit Grant 服务研究文档

## 文件信息
- **路径**: `src/services/api/overageCreditGrant.ts`
- **大小**: 4,913 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Overage Credit Grant 服务负责管理 **超额信用额度授予功能**，向符合条件的组织展示信用额度 upsell。该服务处理以下核心场景：

1. **信用额度查询**: 获取用户组织的超额信用额度资格和可用金额
2. **缓存管理**: 1 小时 TTL 的按组织缓存，避免重复 API 调用
3. **Upsell 展示**: 为 UI 组件提供格式化后的信用额度显示
4. **缓存失效**: 在特定操作后（如领取信用额度）使缓存失效

---

## 功能点目的

### 1. 信用额度获取 (`fetchOverageCreditGrant`)
从后端 API 获取当前用户的超额信用额度信息：
- 端点: `/api/oauth/organizations/{orgUUID}/overage_credit_grant`
- 使用 `prepareApiRequest` 获取认证信息
- 错误时返回 null（不阻塞 UI）

### 2. 缓存读取 (`getCachedOverageCreditGrant`)
非阻塞式缓存读取：
- 按 `organizationUuid` 分区存储
- 检查缓存存在性和 TTL（1 小时）
- 返回 null 时调用方应静默处理（不展示 upsell）

### 3. 缓存刷新 (`refreshOverageCreditGrantCache`)
懒加载式缓存刷新：
- 在 upsell 表面即将渲染且缓存为空时调用
- 使用 `saveGlobalConfig` 的函数式更新避免竞态
- **数据不变性优化**: 如果数据未变化且时间戳新鲜，跳过写入

### 4. 缓存失效 (`invalidateOverageCreditGrantCache`)
精确失效当前组织的缓存条目：
- 保留其他组织的缓存（支持多组织用户）
- 使用函数式更新安全删除

### 5. 金额格式化 (`formatGrantAmount`)
将后端返回的次要单位金额格式化为用户友好的字符串：
- 支持 USD（目前唯一支持的货币）
- 智能处理整数和小数金额（如 "$50" vs "$49.99"）

---

## 具体技术实现

### 关键数据结构

```typescript
// API 响应类型
export type OverageCreditGrantInfo = {
  available: boolean      // 信用额度是否可用
  eligible: boolean       // 用户是否有资格
  granted: boolean        // 是否已授予
  amount_minor_units: number | null  // 金额（次要单位，如美分）
  currency: string | null // 货币代码（如 "USD"）
}

// 缓存条目类型
type CachedGrantEntry = {
  info: OverageCreditGrantInfo
  timestamp: number
}

// 缓存 TTL
const CACHE_TTL_MS = 60 * 60 * 1000  // 1 小时
```

### 关键流程

#### 缓存刷新流程（含数据不变性优化）
```
refreshOverageCreditGrantCache()
├── 检查 isEssentialTrafficOnly() → 直接返回
├── 获取 orgId
├── 调用 fetchOverageCreditGrant()
├── saveGlobalConfig(prev => {
│   ├── 从 prev 读取现有缓存（锁安全）
│   ├── 比较数据字段（available, eligible, granted, amount, currency）
│   ├── 如果数据未变且时间戳新鲜 → 返回 prev（跳过写入）
│   └── 否则返回新配置对象
│ })
└── 完成
```

#### 数据不变性检查
```typescript
const dataUnchanged =
  existing &&
  existing.available === info.available &&
  existing.eligible === info.eligible &&
  existing.granted === info.granted &&
  existing.amount_minor_units === info.amount_minor_units &&
  existing.currency === info.currency

if (dataUnchanged && prevCached && Date.now() - prevCached.timestamp <= CACHE_TTL_MS) {
  return prev  // 跳过写入
}
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/overageCreditGrant.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/components/LogoV2/OverageCreditUpsell.tsx` | `getCachedOverageCreditGrant`, `refreshOverageCreditGrantCache`, `formatGrantAmount` | Logo 旁 upsell 展示 |
| `src/commands/extra-usage/extra-usage-core.ts` | `invalidateOverageCreditGrantCache` | 领取信用额度后失效缓存 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/auth.ts` | `getOauthAccountInfo` |
| `src/utils/config.ts` | `getGlobalConfig`, `saveGlobalConfig` |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly` |
| `src/utils/teleport/api.ts` | `getOAuthHeaders`, `prepareApiRequest` |
| `src/constants/oauth.ts` | `getOauthConfig` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 认证 | 用途 |
|------|------|------|------|
| `/api/oauth/organizations/{orgUUID}/overage_credit_grant` | GET | OAuth Bearer | 获取信用额度信息 |

### 请求头
```typescript
{
  'Authorization': 'Bearer {accessToken}',
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01'
}
```

### 配置存储

缓存存储在 `~/.claude/config.json`：
```json
{
  "overageCreditGrantCache": {
    "org-uuid-1": {
      "info": {
        "available": true,
        "eligible": true,
        "granted": false,
        "amount_minor_units": 5000,
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

1. **单货币支持**
   - `formatGrantAmount` 仅支持 USD
   - 后端若扩展其他货币，前端显示可能不正确

2. **缓存分区粒度**
   - 按 `organizationUuid` 分区是正确的
   - 但 `getOauthAccountInfo()` 可能返回 null，导致无法读取缓存

3. **竞态条件**
   - `invalidateOverageCreditGrantCache` 和 `refreshOverageCreditGrantCache` 可能并发执行
   - 使用 `saveGlobalConfig` 的函数式更新可缓解，但仍存在理论上的竞态

### 边界情况

| 场景 | 行为 |
|------|------|
| `isEssentialTrafficOnly()` | `refreshOverageCreditGrantCache` 直接返回，不调用 API |
| 无 orgId | 所有操作直接返回 null/void |
| API 错误 | `fetchOverageCreditGrant` 返回 null，不更新缓存 |
| 缓存过期 | `getCachedOverageCreditGrant` 返回 null，触发方应调用 refresh |
| 多组织用户 | 各组织缓存独立，切换组织时显示对应额度 |
| 已授予额度 (`granted: true`) | UI 通常隐藏 upsell |

### 改进建议

1. **多货币支持**
   ```typescript
   const CURRENCY_FORMATTERS: Record<string, Intl.NumberFormat> = {
     'USD': new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }),
     'EUR': new Intl.NumberFormat('de-DE', { style: 'currency', currency: 'EUR' }),
     // ...
   }
   
   export function formatGrantAmount(info: OverageCreditGrantInfo): string | null {
     if (info.amount_minor_units == null || !info.currency) return null
     const formatter = CURRENCY_FORMATTERS[info.currency.toUpperCase()]
     if (!formatter) return null
     return formatter.format(info.amount_minor_units / 100)
   }
   ```

2. **缓存预热**
   - 当前仅在 upsell 即将渲染时刷新
   - 建议在登录或组织切换时主动预热缓存

3. **错误重试**
   - 当前 `fetchOverageCreditGrant` 无重试逻辑
   - 建议添加指数退避重试（参考 `referral.ts` 的 `fetchInProgress` 模式）

4. **Analytics 事件**
   ```typescript
   // 建议添加事件跟踪
   logEvent('tengu_overage_credit_shown', {
     eligible: info.eligible,
     available: info.available,
     amount: info.amount_minor_units,
     currency: info.currency
   })
   ```

5. **类型导出优化**
   - `OverageCreditGrantCacheEntry` 通过类型别名导出
   - 建议直接导出 `CachedGrantEntry` 的命名

### 相关模式对比

| 服务 | 缓存策略 | 数据不变性检查 |
|------|---------|---------------|
| `overageCreditGrant.ts` | 磁盘，1h，按组织 | ✅ 完整字段比较 |
| `referral.ts` | 磁盘，24h，按组织 | ❌ 无（直接覆盖） |
| `grove.ts` | 内存+磁盘，24h，按账户 | ❌ 无（直接覆盖） |

`overageCreditGrant.ts` 的数据不变性检查模式（inc-4552）值得在其他缓存服务中推广，可显著减少配置写入放大。
