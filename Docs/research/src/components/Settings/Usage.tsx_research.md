# Usage.tsx 深度研究文档

## 场景与职责

`Usage.tsx` 是 Claude Code 设置面板的**用量标签页**组件，负责展示用户的 API 调用限额、用量统计和额外用量（Extra Usage）信息。它是用户了解当前配额使用情况和订阅状态的主要界面。

### 核心职责

1. **用量限额可视化**：以进度条形式展示各类限额的使用情况
2. **多层级限额支持**：
   - 5小时会话限额（`five_hour`）
   - 7天全模型限额（`seven_day`）
   - 7天 Sonnet 专用限额（`seven_day_sonnet`，仅 Max/Team 计划）
3. **额外用量管理**：展示 Pro/Max 用户的额外用量额度和消耗
4. **超限信用推广**：在符合条件时显示超限信用购买入口

### 架构定位

- 作为 `Settings.tsx` 的子组件，**不接收任何 props**，完全自包含
- 独立管理数据获取、状态更新和错误处理
- 支持键盘快捷键重试加载

## 功能点目的

### 1. 限额进度条展示（LimitBar 组件）

核心可视化组件，展示单个限额的使用情况：

**功能特性**：
- **自适应宽度**：根据终端宽度选择紧凑或展开布局（阈值 62 字符）
- **进度可视化**：使用 `ProgressBar` 组件显示百分比
- **重置时间**：显示限额重置的相对时间（如 "Resets in 2h"）
- **额外子文本**：支持显示额外信息（如额外用量的花费）

**响应式布局**：
- **宽终端（≥62 字符）**：标题、进度条、百分比水平排列
- **窄终端（<62 字符）**：垂直堆叠布局

### 2. 用量数据获取

通过 `fetchUtilization()` API 获取用量数据：

```typescript
interface Utilization {
  five_hour?: RateLimit | null;        // 5小时限额
  seven_day?: RateLimit | null;        // 7天全模型限额
  seven_day_oauth_apps?: RateLimit | null;
  seven_day_opus?: RateLimit | null;
  seven_day_sonnet?: RateLimit | null; // 7天Sonnet专用限额
  extra_usage?: ExtraUsage | null;     // 额外用量
}

interface RateLimit {
  utilization: number | null;  // 使用百分比 0-100
  resets_at: string | null;    // ISO 8601 重置时间
}
```

### 3. 额外用量展示（ExtraUsageSection 组件）

专为 Pro/Max 用户设计的额外用量信息：

**显示逻辑**：
- 仅对 `pro` 或 `max` 订阅类型显示
- 未启用时显示启用提示（"/extra-usage to enable"）
- 无限制时显示 "Unlimited"
- 有限额时显示进度条和花费统计

**花费计算**：
```typescript
const formattedUsedCredits = formatCost(extraUsage.used_credits / 100, 2);
const formattedMonthlyLimit = formatCost(extraUsage.monthly_limit / 100, 2);
```

### 4. Sonnet 限额条件显示

根据订阅类型智能显示 Sonnet 专用限额：

```typescript
const showSonnetBar = subscriptionType === 'max' || 
                      subscriptionType === 'team' || 
                      subscriptionType === null;  // 未知计划默认显示
```

原因：Max 和 Team 计划的 Sonnet 限额与周限额不同，其他计划两者相同，显示会造成冗余。

### 5. 错误处理与重试

完整的错误处理流程：
- **加载状态**：显示 "Loading usage data…"
- **错误状态**：显示错误信息和重试快捷键
- **重试机制**：绑定 `settings:retry` 动作，默认快捷键 "r"

## 具体技术实现

### 关键数据结构

```typescript
// LimitBar 组件 Props
interface LimitBarProps {
  title: string;                    // 限额名称
  limit: RateLimit;                 // 限额数据
  maxWidth: number;                 // 最大显示宽度
  showTimeInReset?: boolean;        // 是否显示重置时间（默认true）
  extraSubtext?: string;            // 额外子文本
}

// 组件状态
const [utilization, setUtilization] = useState<Utilization | null>(null);
const [error, setError] = useState<string | null>(null);
const [isLoading, setIsLoading] = useState(true);
```

### 核心流程

#### 数据加载（行 183-201）
```typescript
const loadUtilization = React.useCallback(async () => {
  setIsLoading(true);
  setError(null);
  try {
    const data = await fetchUtilization();
    setUtilization(data);
  } catch (err) {
    logError(err as Error);
    const axiosError = err as { response?: { data?: unknown } };
    const responseBody = axiosError.response?.data 
      ? jsonStringify(axiosError.response.data) 
      : undefined;
    setError(responseBody 
      ? `Failed to load usage data: ${responseBody}` 
      : 'Failed to load usage data'
    );
  } finally {
    setIsLoading(false);
  }
}, []);
```

#### 键盘重试绑定（行 205-210）
```typescript
useKeybinding('settings:retry', () => {
  void loadUtilization();
}, {
  context: 'Settings',
  isActive: !!error && !isLoading
});
```

#### 宽终端布局（行 63-117）
```typescript
if (maxWidth >= 62) {
  return (
    <Box flexDirection="column">
      <Text bold>{title}</Text>
      <Box flexDirection="row" gap={1}>
        <ProgressBar 
          ratio={utilization / 100} 
          width={50} 
          fillColor="rate_limit_fill" 
          emptyColor="rate_limit_empty" 
        />
        <Text>{usedText}</Text>
      </Box>
      {subtext && <Text dimColor>{subtext}</Text>}
    </Box>
  );
}
```

#### 窄终端布局（行 118-172）
```typescript
// 紧凑布局：标题和子文本在同一行
<Text>
  <Text bold>{title}</Text>
  {subtext && <><Text> </Text><Text dimColor>· {subtext}</Text></>}
</Text>
<ProgressBar ratio={t5} width={maxWidth} ... />
```

### 额外用量渲染逻辑（ExtraUsageSection）

```typescript
function ExtraUsageSection({ extraUsage, maxWidth }: ExtraUsageSectionProps) {
  const subscriptionType = getSubscriptionType();
  const isProOrMax = subscriptionType === "pro" || subscriptionType === "max";
  
  if (!isProOrMax) return false;  // 非Pro/Max用户不显示
  
  if (!extraUsage.is_enabled) {
    if (extraUsageCommand.isEnabled()) {
      return <Text dimColor>Extra usage not enabled · /extra-usage to enable</Text>;
    }
    return null;
  }
  
  if (extraUsage.monthly_limit === null) {
    return <Text dimColor>Unlimited</Text>;  // 无限制计划
  }
  
  // 有限额：显示进度条
  return (
    <LimitBar
      title="Extra usage"
      limit={{ utilization: extraUsage.utilization, resets_at: oneMonthReset.toISOString() }}
      showTimeInReset={false}
      extraSubtext={`${formattedUsedCredits} / ${formattedMonthlyLimit} spent`}
      maxWidth={maxWidth}
    />
  );
}
```

## 关键代码路径与文件引用

### 直接依赖

| 导入路径 | 用途 |
|---------|------|
| `src/commands/extra-usage/index.js` | 额外用量命令状态检查 |
| `src/cost-tracker.js` | `formatCost()` 花费格式化 |
| `src/utils/auth.js` | `getSubscriptionType()` 订阅类型 |
| `../../hooks/useTerminalSize.js` | 终端尺寸监听 |
| `../../ink.js` | `Box`, `Text` 组件 |
| `../../keybindings/useKeybinding.js` | 键盘事件绑定 |
| `../../services/api/usage.js` | `fetchUtilization()` API |
| `../../utils/format.js` | `formatResetText()` 时间格式化 |
| `../../utils/log.js` | `logError()` 错误日志 |
| `../../utils/slowOperations.js` | `jsonStringify()` JSON序列化 |
| `../ConfigurableShortcutHint.js` | 快捷键提示 |
| `../design-system/Byline.js` | 行内分隔组件 |
| `../design-system/ProgressBar.js` | 进度条组件 |
| `../LogoV2/OverageCreditUpsell.js` | 超限信用推广 |

### API 调用流程

```
Usage.tsx
└── fetchUtilization() [src/services/api/usage.ts]
    ├── isClaudeAISubscriber()    # 检查订阅状态
    ├── hasProfileScope()         # 检查权限范围
    ├── isOAuthTokenExpired()     # 检查令牌过期
    ├── getAuthHeaders()          # 获取认证头
    └── axios.get()               # 调用 /api/oauth/usage
```

### 组件渲染流程

```
Usage
├── loadUtilization()           # 初始加载
├── useKeybinding('settings:retry')  # 错误重试
├── 错误状态渲染
│   ├── <Text color="error">Error: {error}</Text>
│   └── <Byline>重试/取消快捷键</Byline>
├── 加载状态渲染
│   └── <Text dimColor>Loading usage data…</Text>
├── 正常状态渲染
│   ├── 限额列表
│   │   ├── LimitBar (five_hour)
│   │   ├── LimitBar (seven_day)
│   │   └── LimitBar (seven_day_sonnet) [条件]
│   ├── ExtraUsageSection [Pro/Max]
│   │   └── LimitBar 或提示文本
│   ├── OverageCreditUpsell [条件]
│   └── <ConfigurableShortcutHint>取消快捷键
```

## 依赖与外部交互

### 与用量 API 的交互

`fetchUtilization()` 在 `src/services/api/usage.ts` 中定义：

**请求配置**：
```typescript
const response = await axios.get<Utilization>(url, {
  headers: {
    'Content-Type': 'application/json',
    'User-Agent': getClaudeCodeUserAgent(),
    ...authResult.headers,
  },
  timeout: 5000,  // 5秒超时
});
```

**前置检查**：
1. 检查是否为 Claude AI 订阅者
2. 检查 OAuth 令牌是否过期
3. 获取认证头

### 与订阅系统的交互

通过 `getSubscriptionType()` 获取用户订阅类型：
- `null`：未订阅或无法获取
- `'pro'`：Pro 计划
- `'max'`：Max 计划
- `'team'`：Team 计划

用于决定：
- 是否显示 Sonnet 专用限额
- 是否显示额外用量部分

### 与额外用量命令的交互

通过 `extraUsageCommand.isEnabled()` 检查额外用量功能是否启用：
- 功能未启用时显示启用提示
- 功能已启用但未激活时显示状态

### 键盘事件系统

绑定 `settings:retry` 动作用于错误重试：
- 仅在出错且未加载时激活
- 默认快捷键为 "r"
- 上下文为 "Settings"

## 风险、边界与改进建议

### 已知风险

1. **硬编码的宽度阈值**：
   ```typescript
   if (maxWidth >= 62)  // 62 字符阈值无文档说明
   ```
   阈值选择缺乏依据，可能在特定终端尺寸下表现不佳。

2. **额外用量月份计算**：
   ```typescript
   const oneMonthReset = new Date(now.getFullYear(), now.getMonth() + 1, 1);
   ```
   使用本地时间而非 UTC，可能在时区边界处产生偏差。

3. **Sonnet 限额显示逻辑**：
   ```typescript
   const showSonnetBar = subscriptionType === 'max' || 
                         subscriptionType === 'team' || 
                         subscriptionType === null;
   ```
   对 `null`（未知计划）默认显示，可能与实际限额策略不一致。

4. **API 超时硬编码**：
   `timeout: 5000` 在慢网络环境下可能导致频繁超时。

### 边界情况

1. **utilization 为 null**：
   ```typescript
   if (utilization === null) return null;  // LimitBar 直接返回 null
   ```
   限额未设置时整个 LimitBar 不渲染。

2. **resets_at 为 null**：
   不显示重置时间，仅显示使用百分比。

3. **终端宽度极端变化**：
   `maxWidth` 计算为 `columns - 2`，在极窄终端（<10 列）下 ProgressBar 可能异常。

4. **额外用量数据不完整**：
   ```typescript
   if (typeof extraUsage.used_credits !== "number" || 
       typeof extraUsage.utilization !== "number") {
     return null;  // 静默忽略
   }
   ```

5. **非订阅用户**：
   显示 `/usage is only available for subscription plans.` 提示。

### 改进建议

1. **宽度阈值配置化**：
   ```typescript
   const COMPACT_LAYOUT_THRESHOLD = 62;  // 提取为常量
   ```

2. **添加加载超时提示**：
   当前 5 秒超时后只显示通用错误，建议区分超时和其他错误。

3. **支持手动刷新**：
   除了错误重试，添加正常状态下的刷新功能（如按 "r" 刷新）。

4. **限额预警**：
   当 utilization > 80% 时改变进度条颜色为警告色。

5. **历史趋势**：
   显示用量历史趋势，帮助用户了解使用模式。

6. **缓存机制**：
   添加本地缓存避免频繁 API 调用，同时显示缓存时间。

7. **时区处理**：
   ```typescript
   // 使用 UTC 计算月份重置
   const oneMonthReset = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth() + 1, 1));
   ```

8. **可访问性增强**：
   ```typescript
   <ProgressBar 
     ratio={utilization / 100} 
     aria-label={`${title}: ${Math.floor(utilization)}% used`}
   />
   ```

### 测试关注点

- 验证各种订阅类型（Pro/Max/Team/无）的正确显示
- 测试宽/窄终端布局切换
- 验证 API 超时和错误处理
- 测试额外用量的各种状态（未启用/无限制/有限额）
- 验证键盘重试功能
- 测试 utilization/resets_at 为 null 时的降级
- 验证时区边界处的重置时间显示
