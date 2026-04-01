# GuestPassesUpsell.tsx 深度研究文档

## 场景与职责

GuestPassesUpsell.tsx 是 Claude Code 终端 UI 中的营销/推广组件，负责在欢迎屏幕中展示 Guest Passes（访客通行证）推广信息。该组件主要面向 Claude AI 的 Max 订阅用户，鼓励他们分享 Claude Code 给好友以获得额外使用额度。

**核心职责**:
1. **资格检查**: 判断当前用户是否应该看到 Guest Passes 推广
2. **状态管理**: 跟踪推广展示次数、用户是否已访问 `/passes` 命令等
3. **推广展示**: 在 CondensedLogo 和 Feed 中渲染推广内容
4. **分析追踪**: 记录推广展示事件用于数据分析

## 功能点目的

### 1. 资格检查系统

**目的**: 确定是否应该向用户展示 Guest Passes 推广。

**检查条件**（`shouldShowGuestPassesUpsell` 函数）:
1. 用户必须通过 `checkCachedPassesEligibility()` 检查
   - 需要是 Max 订阅用户
   - 需要有有效的 OAuth 组织信息
   - 需要缓存数据存在（不阻塞等待网络请求）
2. 推广展示次数必须少于 3 次
3. 用户不能已经访问过 `/passes` 命令

**特殊逻辑 - 重置机制**（`resetIfPassesRefreshed`）:
- 当检测到剩余通行证数量增加时（表示用户获得了新的通行证）
- 重置展示计数器和访问状态，允许再次展示推广

```typescript
function resetIfPassesRefreshed(): void {
  const remaining = getCachedRemainingPasses();
  if (remaining == null || remaining <= 0) return;
  
  const config = getGlobalConfig();
  const lastSeen = config.passesLastSeenRemaining ?? 0;
  
  if (remaining > lastSeen) {
    saveGlobalConfig(prev => ({
      ...prev,
      passesUpsellSeenCount: 0,
      hasVisitedPasses: false,
      passesLastSeenRemaining: remaining
    }));
  }
}
```

### 2. 展示计数管理

**目的**: 限制推广展示次数，避免过度打扰用户。

**实现**（`incrementGuestPassesSeenCount` 函数）:
1. 从全局配置读取当前计数
2. 递增计数并保存
3. 记录分析事件 `tengu_guest_passes_upsell_shown`

```typescript
export function incrementGuestPassesSeenCount(): void {
  let newCount = 0;
  saveGlobalConfig(prev => {
    newCount = (prev.passesUpsellSeenCount ?? 0) + 1;
    return { ...prev, passesUpsellSeenCount: newCount };
  });
  logEvent('tengu_guest_passes_upsell_shown', { seen_count: newCount });
}
```

### 3. 推广内容渲染

**目的**: 在终端 UI 中渲染吸引人的推广信息。

**CondensedLogo 模式**（`GuestPassesUpsell` 组件）:
- 单行紧凑布局
- 显示装饰性符号 `[✻] [✻] [✻]`
- 动态内容：如果有奖励信息则显示奖励金额，否则显示默认文案
- 指向 `/passes` 命令

```typescript
export function GuestPassesUpsell() {
  const $ = _c(1);
  let t0;
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    const reward = getCachedReferrerReward();
    t0 = <Text dimColor>
      <Text color="claude">[✻]</Text>{" "}
      <Text color="claude">[✻]</Text>{" "}
      <Text color="claude">[✻]</Text> · {" "}
      {reward 
        ? `Share Claude Code and earn ${formatCreditAmount(reward)} of extra usage · /passes`
        : "3 guest passes at /passes"
      }
    </Text>;
    $[0] = t0;
  } else {
    t0 = $[0];
  }
  return t0;
}
```

**Feed 模式**（通过 `createGuestPassesFeed` 在 `feedConfigs.tsx`）:
- 使用 Feed 组件的 `customContent` 渲染更丰富的内容
- 包含 Box 布局和装饰性元素
- 固定宽度 48 字符

## 具体技术实现

### 关键流程

1. **资格检查流程**:
   ```
   shouldShowGuestPassesUpsell()
   ├── checkCachedPassesEligibility()
   │   ├── shouldCheckForPasses() - 检查订阅类型
   │   └── 检查缓存状态和过期时间
   ├── resetIfPassesRefreshed() - 检查是否需要重置计数
   ├── 检查展示次数 < 3
   └── 检查是否已访问 /passes
   ```

2. **展示计数递增流程**:
   ```
   incrementGuestPassesSeenCount()
   ├── saveGlobalConfig() - 更新计数
   └── logEvent('tengu_guest_passes_upsell_shown')
   ```

3. **渲染流程**:
   ```
   useShowGuestPassesUpsell() 
   └── useState(shouldShowGuestPassesUpsell)
   
   GuestPassesUpsell()
   ├── getCachedReferrerReward() - 获取奖励信息
   ├── formatCreditAmount() - 格式化金额
   └── 返回 memoized JSX
   ```

### 数据结构

**GlobalConfig 相关字段**（来自 `src/utils/config.ts`）:
```typescript
type GlobalConfig = {
  // Guest passes eligibility cache per org - key is org ID
  passesEligibilityCache?: Record<
    string,
    ReferralEligibilityResponse & { timestamp: number }
  >;
  
  // Guest passes upsell tracking
  passesUpsellSeenCount?: number;  // 展示次数
  hasVisitedPasses?: boolean;      // 是否访问过 /passes
  passesLastSeenRemaining?: number; // 上次看到的剩余通行证数
};
```

**ReferralEligibilityResponse**（来自 `src/services/api/referral.ts`）:
```typescript
type ReferralEligibilityResponse = {
  eligible: boolean;
  remaining_passes?: number;
  referrer_reward?: ReferrerRewardInfo;
  timestamp: number;  // 添加的缓存时间戳
};

type ReferrerRewardInfo = {
  amount_minor_units: number;
  currency: string;
};
```

### 协议/接口

**API 接口**（`src/services/api/referral.ts`）:
- `checkCachedPassesEligibility()`: 同步检查缓存的资格状态
- `getCachedReferrerReward()`: 获取缓存的推荐奖励信息
- `getCachedRemainingPasses()`: 获取缓存的剩余通行证数
- `fetchAndStorePassesEligibility()`: 异步获取并缓存资格信息

**分析事件**:
- 事件名: `tengu_guest_passes_upsell_shown`
- 元数据: `{ seen_count: number }`

## 关键代码路径与文件引用

### 直接依赖

| 文件 | 用途 |
|------|------|
| `src/ink.ts` | Ink 组件库（Text） |
| `src/services/analytics/index.ts` | 分析事件记录（logEvent） |
| `src/services/api/referral.ts` | 推荐系统 API（资格检查、奖励信息） |
| `src/utils/config.ts` | 全局配置读写（getGlobalConfig, saveGlobalConfig） |

### 调用方

| 文件 | 调用方式 |
|------|----------|
| `src/components/LogoV2/CondensedLogo.tsx` | 导入 `GuestPassesUpsell`, `useShowGuestPassesUpsell`, `incrementGuestPassesSeenCount` |
| `src/components/LogoV2/LogoV2.tsx` | 同上 |
| `src/components/LogoV2/feedConfigs.tsx` | 导入 `getCachedReferrerReward`, `formatCreditAmount` 用于 `createGuestPassesFeed` |

### 核心代码片段

**GuestPassesUpsell.tsx 第 22-35 行 - 资格检查**:
```typescript
function shouldShowGuestPassesUpsell(): boolean {
  const { eligible, hasCache } = checkCachedPassesEligibility();
  
  // Only show if eligible and cache exists (don't block on fetch)
  if (!eligible || !hasCache) return false;
  
  // Reset upsell counters if passes were refreshed
  resetIfPassesRefreshed();
  
  const config = getGlobalConfig();
  if ((config.passesUpsellSeenCount ?? 0) >= 3) return false;
  if (config.hasVisitedPasses) return false;
  
  return true;
}
```

**GuestPassesUpsell.tsx 第 36-42 行 - Hook 封装**:
```typescript
export function useShowGuestPassesUpsell() {
  const [show] = useState(_temp);
  return show;
}

function _temp() {
  return shouldShowGuestPassesUpsell();
}
```

**GuestPassesUpsell.tsx 第 43-55 行 - 计数递增**:
```typescript
export function incrementGuestPassesSeenCount(): void {
  let newCount = 0;
  saveGlobalConfig(prev => {
    newCount = (prev.passesUpsellSeenCount ?? 0) + 1;
    return {
      ...prev,
      passesUpsellSeenCount: newCount
    };
  });
  logEvent('tengu_guest_passes_upsell_shown', {
    seen_count: newCount
  });
}
```

**GuestPassesUpsell.tsx 第 58-69 行 - 组件渲染**:
```typescript
export function GuestPassesUpsell() {
  const $ = _c(1);
  let t0;
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    const reward = getCachedReferrerReward();
    t0 = <Text dimColor={true}>
      <Text color="claude">[✻]</Text>{" "}
      <Text color="claude">[✻]</Text>{" "}
      <Text color="claude">[✻]</Text> · {" "}
      {reward 
        ? `Share Claude Code and earn ${formatCreditAmount(reward)} of extra usage · /passes`
        : "3 guest passes at /passes"
      }
    </Text>;
    $[0] = t0;
  } else {
    t0 = $[0];
  }
  return t0;
}
```

**feedConfigs.tsx 第 74-91 行 - Feed 模式**:
```typescript
export function createGuestPassesFeed(): FeedConfig {
  const reward = getCachedReferrerReward();
  const subtitle = reward 
    ? `Share Claude Code and earn ${formatCreditAmount(reward)} of extra usage` 
    : 'Share Claude Code with friends';
  
  return {
    title: '3 guest passes',
    lines: [],
    customContent: {
      content: <>
        <Box marginY={1}>
          <Text color="claude">[✻] [✻] [✻]</Text>
        </Box>
        <Text dimColor>{subtitle}</Text>
      </>,
      width: 48
    },
    footer: '/passes'
  };
}
```

## 依赖与外部交互

### 运行时依赖

1. **react/compiler-runtime**: React Compiler 缓存机制
2. **react**: useState Hook
3. **ink**: 终端 UI 渲染

### 外部服务依赖

1. **推荐系统 API** (`src/services/api/referral.ts`):
   - 缓存资格信息（24小时过期）
   - 提供奖励金额信息
   - 跟踪剩余通行证数量

2. **分析系统** (`src/services/analytics/index.ts`):
   - 记录推广展示事件
   - 支持展示次数追踪

3. **配置系统** (`src/utils/config.ts`):
   - 持久化展示计数
   - 跟踪用户访问状态

### 订阅类型检查

```typescript
// from src/services/api/referral.ts
function shouldCheckForPasses(): boolean {
  return !!(
    getOauthAccountInfo()?.organizationUuid &&
    isClaudeAISubscriber() &&
    getSubscriptionType() === 'max'
  );
}
```

只有满足以下条件的用户才会看到推广：
- 已登录并有组织 UUID
- 是 Claude AI 订阅者
- 订阅类型为 Max

## 风险、边界与改进建议

### 已知风险

1. **Hook 限制问题**:
   ```typescript
   export function useShowGuestPassesUpsell() {
     const [show] = useState(_temp);  // 只在挂载时计算
     return show;
   }
   ```
   - 使用 `useState` 只在组件挂载时计算一次
   - 如果资格状态在会话期间变化（如用户升级订阅），不会自动更新
   - 需要重新启动 Claude Code 才能看到变化

2. **缓存过期处理**:
   - `checkCachedPassesEligibility` 返回 `needsRefresh` 标志
   - 但当前实现忽略此标志，只检查 `eligible` 和 `hasCache`
   - 可能向用户展示过期的推广信息

3. **并发计数更新**:
   - `incrementGuestPassesSeenCount` 使用闭包变量 `newCount`
   - 在并发场景下可能导致计数不准确

### 边界情况

1. **奖励信息缺失**:
   - `getCachedReferrerReward()` 可能返回 null
   - 组件已处理，显示默认文案 "3 guest passes at /passes"

2. **配置字段缺失**:
   - 使用空值合并运算符 `??` 处理可能缺失的配置字段
   - `passesUpsellSeenCount ?? 0`, `passesLastSeenRemaining ?? 0`

3. **组织切换**:
   - 缓存按组织 ID 存储（`passesEligibilityCache[orgId]`）
   - 切换组织后展示计数不会重置（全局配置）

### 改进建议

1. **响应式资格检查**:
   ```typescript
   // 当前：一次性计算
   export function useShowGuestPassesUpsell() {
     const [show] = useState(shouldShowGuestPassesUpsell);
     return show;
   }
   
   // 建议：定期刷新或使用事件监听
   export function useShowGuestPassesUpsell() {
     const [show, setShow] = useState(shouldShowGuestPassesUpsell);
     
     useEffect(() => {
       const interval = setInterval(() => {
         setShow(shouldShowGuestPassesUpsell());
       }, 60000); // 每分钟检查
       return () => clearInterval(interval);
     }, []);
     
     return show;
   }
   ```

2. **缓存刷新触发**:
   ```typescript
   function shouldShowGuestPassesUpsell(): boolean {
     const { eligible, hasCache, needsRefresh } = checkCachedPassesEligibility();
     
     if (needsRefresh) {
       // 触发后台刷新，但不阻塞
       void fetchAndStorePassesEligibility();
     }
     
     // ... rest of logic
   }
   ```

3. **原子计数更新**:
   ```typescript
   export function incrementGuestPassesSeenCount(): number {
     let finalCount = 0;
     saveGlobalConfig(prev => {
       finalCount = (prev.passesUpsellSeenCount ?? 0) + 1;
       return { ...prev, passesUpsellSeenCount: finalCount };
     });
     logEvent('tengu_guest_passes_upsell_shown', { seen_count: finalCount });
     return finalCount;
   }
   ```

4. **组织隔离计数**:
   ```typescript
   type GlobalConfig = {
     passesUpsellTracking?: Record<string, {
       seenCount: number;
       hasVisited: boolean;
       lastSeenRemaining: number;
     }>;  // key: orgId
   };
   ```

5. **A/B 测试支持**:
   ```typescript
   type UpsellVariant = 'default' | 'emphasized' | 'minimal';
   
   export function GuestPassesUpsell({ variant }: { variant?: UpsellVariant }) {
     // 根据 variant 渲染不同样式
   }
   ```

6. **可访问性改进**:
   - 添加键盘快捷键支持（如按 `p` 直接跳转到 `/passes`）
   - 为装饰符号添加屏幕阅读器文本

### 测试建议

1. **单元测试**:
   - 各种资格状态的组合测试
   - 展示次数边界（0, 2, 3, 4 次）
   - 重置逻辑（remaining > lastSeen）

2. **集成测试**:
   - 与 GlobalConfig 的读写集成
   - 与 referral API 的集成
   - 分析事件正确发送

3. **端到端测试**:
   - 完整用户流程：看到推广 → 访问 /passes → 不再显示
   - 订阅变更后的行为
