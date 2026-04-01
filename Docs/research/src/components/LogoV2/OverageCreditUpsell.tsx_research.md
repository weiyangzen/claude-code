# OverageCreditUpsell.tsx 深度研究文档

## 场景与职责

`OverageCreditUpsell.tsx` 是 Claude Code CLI 中用于推广"额外使用额度"(Overage Credit Grant)功能的营销组件。当用户有资格获得免费的额外API使用额度时，此组件在欢迎界面展示相关推广信息，引导用户使用 `/extra-usage` 命令领取额度。

### 核心职责
1. **资格检查**: 根据后端API返回的数据判断用户是否有资格获得额外额度
2. **展示控制**: 限制最多展示3次，且用户访问过 `/extra-usage` 后不再展示
3. **多形态渲染**: 支持单行和双行两种展示模式，适应不同布局场景
4. **Feed集成**: 提供Feed配置，可集成到LogoV2的右侧Feed列
5. **分析追踪**: 记录展示次数用于产品分析

---

## 功能点目的

### 1. 资格判断体系
组件区分两个层面的资格：

| 函数 | 用途 | 使用场景 |
|-----|------|---------|
| `isEligibleForOverageCreditGrant()` | 纯后端资格判断 | `/usage` 等持久展示场景 |
| `shouldShowOverageCreditUpsell()` | 资格 + 展示限制 | 营销推广场景（欢迎界面） |

### 2. 展示限制策略
- **展示次数上限**: `MAX_IMPRESSIONS = 3`
- **访问后隐藏**: 用户访问 `/extra-usage` 后设置 `hasVisitedExtraUsage = true`
- **优先级**: 低于 GuestPassesUpsell（在 LogoV2 中后判断）

### 3. 缓存策略
- **缓存时长**: 1小时 (`CACHE_TTL_MS = 60 * 60 * 1000`)
- **缓存键**: 按组织ID (`organizationUuid`) 存储
- **懒加载**: `maybeRefreshOverageCreditCache()` 在组件挂载时触发后台刷新

---

## 具体技术实现

### 关键常量
```typescript
const MAX_IMPRESSIONS = 3;  // 最大展示次数
```

### 资格判断逻辑
```typescript
export function isEligibleForOverageCreditGrant(): boolean {
  const info = getCachedOverageCreditGrant();
  // 必须满足: 有缓存数据、available=true、granted=false、金额可格式化
  if (!info || !info.available || info.granted) return false;
  return formatGrantAmount(info) !== null;
}

export function shouldShowOverageCreditUpsell(): boolean {
  if (!isEligibleForOverageCreditGrant()) return false;
  const config = getGlobalConfig();
  if (config.hasVisitedExtraUsage) return false;
  if ((config.overageCreditUpsellSeenCount ?? 0) >= MAX_IMPRESSIONS) return false;
  return true;
}
```

### 数据类型定义
```typescript
// API返回的原始数据
export type OverageCreditGrantInfo = {
  available: boolean;        // 是否可以领取
  eligible: boolean;         // 是否有资格
  granted: boolean;          // 是否已领取
  amount_minor_units: number | null;  // 金额（最小货币单位，如美分）
  currency: string | null;   // 货币代码（如 "USD"）
}

// 缓存条目结构
type CachedGrantEntry = {
  info: OverageCreditGrantInfo;
  timestamp: number;  // 缓存时间戳
}
```

### 缓存刷新机制
```typescript
export function maybeRefreshOverageCreditCache(): void {
  // 缓存已存在则不刷新（避免重复请求）
  if (getCachedOverageCreditGrant() !== null) return;
  // 后台刷新，不阻塞UI
  void refreshOverageCreditGrantCache();
}

export async function refreshOverageCreditGrantCache(): Promise<void> {
  if (isEssentialTrafficOnly()) return;  // 隐私模式跳过
  
  const orgId = getOauthAccountInfo()?.organizationUuid;
  if (!orgId) return;
  
  const info = await fetchOverageCreditGrant();
  if (!info) return;
  
  saveGlobalConfig(prev => {
    // 数据未变化且缓存仍新鲜时跳过写入（inc-4552模式）
    const prevCached = prev.overageCreditGrantCache?.[orgId];
    const dataUnchanged = /* ... */;
    if (dataUnchanged && /* 缓存新鲜 */) {
      return prev;
    }
    // 更新缓存
    return {
      ...prev,
      overageCreditGrantCache: {
        ...prev.overageCreditGrantCache,
        [orgId]: { info: dataUnchanged ? existing : info, timestamp: Date.now() },
      },
    };
  });
}
```

### 金额格式化
```typescript
export function formatGrantAmount(info: OverageCreditGrantInfo): string | null {
  if (info.amount_minor_units == null || !info.currency) return null;
  
  // 目前仅支持USD，后端可能扩展
  if (info.currency.toUpperCase() === 'USD') {
    const dollars = info.amount_minor_units / 100;
    return Number.isInteger(dollars) ? `$${dollars}` : `$${dollars.toFixed(2)}`;
  }
  return null;
}
```

### 展示计数递增
```typescript
export function incrementOverageCreditUpsellSeenCount(): void {
  let newCount = 0;
  saveGlobalConfig(prev => {
    newCount = (prev.overageCreditUpsellSeenCount ?? 0) + 1;
    return {
      ...prev,
      overageCreditUpsellSeenCount: newCount,
    };
  });
  // 发送分析事件
  logEvent('tengu_overage_credit_upsell_shown', { seen_count: newCount });
}
```

### 组件渲染模式

#### 双行模式 (`twoLine=true`)
用于Feed展示：
```
$XX in extra usage
On us. Works on third-party apps · /extra-usage
```

#### 单行模式 (`twoLine=false`)
用于紧凑布局：
```
$XX in extra usage for third-party apps · /extra-usage
```
（前半部分高亮显示）

### Feed配置生成
```typescript
export function createOverageCreditFeed(): FeedConfig {
  const info = getCachedOverageCreditGrant();
  const amount = info ? formatGrantAmount(info) : null;
  const title = amount ? getFeedTitle(amount) : 'extra usage credit';
  
  return {
    title,
    lines: [],
    customContent: {
      content: <Text dimColor>{FEED_SUBTITLE}</Text>,
      width: Math.max(title.length, FEED_SUBTITLE.length),
    },
  };
}
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `react/compiler-runtime` | React Compiler自动记忆化 |
| `react` (useState) | React hooks |
| `src/ink.js` (Text) | Ink UI组件 |
| `src/services/analytics/index.ts` | `logEvent` 分析追踪 |
| `src/services/api/overageCreditGrant.ts` | 后端API交互和缓存管理 |
| `src/utils/config.ts` | 全局配置读写 |
| `src/utils/format.ts` | `truncate` 字符串截断 |
| `src/components/LogoV2/Feed.tsx` | `FeedConfig` 类型定义 |

### 依赖详解

#### `overageCreditGrant.ts` (src/services/api/overageCreditGrant.ts)
核心服务端API交互模块：

**API端点**: 
```
GET {BASE_API_URL}/api/oauth/organizations/{orgUUID}/overage_credit_grant
```

**缓存结构** (GlobalConfig中):
```typescript
overageCreditGrantCache?: Record<string, {
  info: OverageCreditGrantInfo;
  timestamp: number;
}>;
```

**关键设计决策**:
1. **按组织缓存**: 支持多组织用户，各组织额度独立
2. **数据不变性优化**: 如果API返回数据与缓存相同，仅更新时间戳避免配置写入
3. **错误静默处理**: API失败返回null，不阻断用户体验

---

## 依赖与外部交互

### 1. 配置系统交互

**读取字段**:
- `overageCreditGrantCache[orgId]` - 缓存的额度信息
- `overageCreditUpsellSeenCount` - 展示次数
- `hasVisitedExtraUsage` - 是否访问过/extra-usage

**配置位置** (src/utils/config.ts):
```typescript
export type GlobalConfig = {
  // ...
  overageCreditGrantCache?: Record<string, {
    info: {
      available: boolean;
      eligible: boolean;
      granted: boolean;
      amount_minor_units: number | null;
      currency: string | null;
    };
    timestamp: number;
  }>;
  overageCreditUpsellSeenCount?: number;
  hasVisitedExtraUsage?: boolean;
  // ...
};
```

### 2. 后端API交互

**请求头**:
```typescript
{
  Authorization: `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
}
```

**错误处理**: 
- 网络错误: 记录到诊断日志，返回null
- 401/403: 同样返回null，由上层决定是否需要重新认证

### 3. 调用方

| 调用方 | 使用方式 |
|-------|---------|
| `LogoV2.tsx` | `useShowOverageCreditUpsell()`, `incrementOverageCreditUpsellSeenCount()`, `createOverageCreditFeed()` |
| `CondensedLogo.tsx` | `useShowOverageCreditUpsell()`, `incrementOverageCreditUpsellSeenCount()`, `<OverageCreditUpsell />` |
| `/extra-usage` 命令 | 用户访问后设置 `hasVisitedExtraUsage = true` |
| `/usage` 命令 | `isEligibleForOverageCreditGrant()` 用于持久展示 |

### 4. 分析事件

| 事件名 | 触发时机 | 参数 |
|-------|---------|------|
| `tengu_overage_credit_upsell_shown` | 每次展示时 | `seen_count: number` |

---

## 风险、边界与改进建议

### 潜在风险

#### 1. 多组织用户缓存冲突
**问题**: 用户切换组织时，缓存可能混淆

**当前处理**: 缓存键使用 `organizationUuid`，切换组织时会读取不同键
```typescript
const orgId = getOauthAccountInfo()?.organizationUuid;
const cached = getGlobalConfig().overageCreditGrantCache?.[orgId];
```

**残余风险**: 如果 `getOauthAccountInfo()` 返回null，整个功能失效

#### 2. 缓存过期与展示次数的竞态
**问题**: 缓存过期后重新获取数据，如果资格状态变化，展示次数是否重置？

**当前行为**: 展示次数是独立的，即使资格变化也会继续计数

**建议**: 当用户从有资格变为无资格时，考虑重置展示次数

#### 3. 硬编码的字符预算
```typescript
// Copy from "OC & Bulk Overages copy" doc (#4 — CLI Welcome screen).
// Char budgets: title ≤19, subtitle ≤48.
const FEED_SUBTITLE = 'On us. Works on third-party apps · /extra-usage';
```
- 如果文案变更，需要同步更新截断逻辑
- 不同语言的本地化会有问题

#### 4. 货币支持有限
```typescript
// For now only USD; backend may expand later
if (info.currency.toUpperCase() === 'USD') {
  // ...
}
```
- 仅支持USD格式化
- 其他货币返回null，导致组件不渲染

### 边界条件

| 场景 | 处理逻辑 |
|-----|---------|
| 缓存为空 | `maybeRefreshOverageCreditCache()` 触发后台刷新，当前渲染周期不展示 |
| API返回granted=true | `isEligibleForOverageCreditGrant()` 返回false，不展示 |
| amount_minor_units为null | `formatGrantAmount()` 返回null，不展示 |
| 非USD货币 | `formatGrantAmount()` 返回null，不展示 |
| maxWidth未提供 | 不进行截断，使用完整文案 |
| 用户已访问/extra-usage | `hasVisitedExtraUsage` 为true，不展示 |
| 已达展示上限(3次) | 不展示 |

### 改进建议

#### 1. 支持多货币
```typescript
const CURRENCY_FORMATTERS: Record<string, (amount: number) => string> = {
  USD: (amount) => `$${(amount / 100).toFixed(2)}`,
  EUR: (amount) => `€${(amount / 100).toFixed(2)}`,
  GBP: (amount) => `£${(amount / 100).toFixed(2)}`,
  // 默认使用货币代码
  default: (amount, currency) => `${currency} ${amount}`,
};
```

#### 2. 添加更多分析事件
```typescript
// 当前只有展示事件，建议添加：
logEvent('tengu_overage_credit_upsell_dismissed');  // 用户主动关闭
logEvent('tengu_overage_credit_upsell_clicked');    // 用户点击/extra-usage
logEvent('tengu_overage_credit_eligible', {         // 资格检测
  available: info.available,
  eligible: info.eligible,
  amount: info.amount_minor_units,
});
```

#### 3. 文案国际化准备
```typescript
// 建议: 提取文案到配置
const COPY = {
  en: {
    feedTitle: (amount: string) => `${amount} in extra usage`,
    feedSubtitle: 'On us. Works on third-party apps · /extra-usage',
    usageText: (amount: string) => `${amount} in extra usage for third-party apps · /extra-usage`,
  },
  // 未来扩展其他语言
};
```

#### 4. 缓存预热优化
```typescript
// 当前: 组件挂载时才触发刷新
// 建议: 在应用启动时预加载
export function prefetchOverageCreditGrant(): void {
  if (getCachedOverageCreditGrant() === null) {
    void refreshOverageCreditGrantCache();
  }
}
// 在 setup.ts 中调用
```

#### 5. 展示频率限制细化
```typescript
// 建议: 添加时间窗口限制，如每天最多展示1次
function shouldShowInTimeWindow(): boolean {
  const lastShown = getGlobalConfig().overageCreditUpsellLastShown;
  if (!lastShown) return true;
  const hoursSinceLastShown = (Date.now() - lastShown) / (1000 * 60 * 60);
  return hoursSinceLastShown >= 24;
}
```

#### 6. 测试覆盖建议
缺少的测试场景：
- 多组织用户的缓存隔离
- 缓存过期后的行为
- 货币格式化边界（极大/极小金额）
- 网络失败后的降级行为
- 配置持久化失败的处理

---

## 代码量统计
- **总行数**: 166行 (含source map)
- **有效代码**: ~100行
- **复杂度**: 中等（多渲染模式、缓存逻辑、资格判断）

## 相关文档
- "OC & Bulk Overages copy" doc - 产品文案规范
- `src/services/api/overageCreditGrant.ts` - 后端API交互
- `src/components/LogoV2/Feed.tsx` - Feed系统集成
