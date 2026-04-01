# Opus1mMergeNotice.tsx 深度研究文档

## 场景与职责

`Opus1mMergeNotice.tsx` 是 Claude Code CLI 中用于通知用户 Opus 模型默认上下文窗口升级的营销组件。当 Anthropic 将 Opus 模型的默认上下文从 200K 升级到 1M tokens 时，此组件负责向符合条件的用户展示这一产品更新通知。

### 核心职责
1. **产品更新通知**: 告知用户 Opus 模型现在默认支持 1M 上下文窗口
2. **展示频率控制**: 限制通知最多展示 6 次，避免过度打扰用户
3. **状态持久化**: 记录用户已查看通知的次数到全局配置
4. **视觉吸引**: 使用动画星号(↑)吸引用户注意

---

## 功能点目的

### 1. 展示条件判断
通知仅在以下**所有条件**满足时展示：
- `isOpus1mMergeEnabled()` 返回 true（功能开关启用）
- 用户查看次数 < `MAX_SHOW_COUNT` (6次)

### 2. 展示次数跟踪
- 使用 `opus1mMergeNoticeSeenCount` 字段记录展示次数
- 每次组件渲染且条件满足时，自动递增计数
- 计数达到上限后，组件永久不再展示（直到配置重置）

### 3. 视觉设计
- 使用 `UP_ARROW` (↑) 符号配合 `AnimatedAsterisk` 组件
- 动画效果：颜色渐变扫描（hue sweep），持续 1.5秒 × 2次 = 3秒
- 动画结束后 settled 为灰色

---

## 具体技术实现

### 关键常量
```typescript
const MAX_SHOW_COUNT = 6;  // 最大展示次数
```

### 展示条件判断函数
```typescript
export function shouldShowOpus1mMergeNotice(): boolean {
  return (
    isOpus1mMergeEnabled() &&
    (getGlobalConfig().opus1mMergeNoticeSeenCount ?? 0) < MAX_SHOW_COUNT
  );
}
```

### 状态更新机制
```typescript
useEffect(() => {
  if (!show) return;
  
  const newCount = (getGlobalConfig().opus1mMergeNoticeSeenCount ?? 0) + 1;
  
  saveGlobalConfig(prev => {
    // 防御性检查：如果其他进程已更新，避免重复计数
    if ((prev.opus1mMergeNoticeSeenCount ?? 0) >= newCount) {
      return prev;
    }
    return {
      ...prev,
      opus1mMergeNoticeSeenCount: newCount,
    };
  });
}, [show]);
```

### 渲染输出
```tsx
<Box paddingLeft={2}>
  <AnimatedAsterisk char={UP_ARROW} />
  <Text dimColor={true}>
    {" "}Opus now defaults to 1M context · 5x more room, same pricing
  </Text>
</Box>
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `react/compiler-runtime` | React Compiler自动记忆化 |
| `react` (useEffect, useState) | React hooks |
| `src/constants/figures.js` | `UP_ARROW` (↑) 符号常量 |
| `src/ink.js` (Box, Text) | Ink UI组件 |
| `src/utils/config.ts` | 全局配置读写 |
| `src/utils/model/model.ts` | `isOpus1mMergeEnabled()` 功能开关 |
| `src/components/LogoV2/AnimatedAsterisk.tsx` | 动画星号组件 |

### 依赖详解

#### `isOpus1mMergeEnabled()` (src/utils/model/model.ts)
```typescript
export function isOpus1mMergeEnabled(): boolean {
  // 1M上下文被禁用
  if (is1mContextDisabled()) return false;
  
  // Pro订阅用户不适用
  if (isProSubscriber()) return false;
  
  // 非第一方API provider
  if (getAPIProvider() !== 'firstParty') return false;
  
  // 订阅类型未知时保守处理（避免向VS Code子进程等泄露功能）
  if (isClaudeAISubscriber() && getSubscriptionType() === null) {
    return false;
  }
  
  return true;
}
```

#### `AnimatedAsterisk` (src/components/LogoV2/AnimatedAsterisk.tsx)
- **动画时长**: 1500ms × 2次扫描 = 3000ms 总时长
- **动画效果**: HSL色相环扫描（0° → 360°）
- **无障碍**: 检测 `prefersReducedMotion` 设置，如启用则跳过动画
- **性能**: 使用 `useAnimationFrame` 并在组件离开视口时自动暂停

---

## 依赖与外部交互

### 1. 配置系统
**读取字段**: `opus1mMergeNoticeSeenCount`
- 类型: `number | undefined`
- 默认值: 0 (通过 `?? 0` 处理)

**配置位置**: `GlobalConfig` 接口 (src/utils/config.ts:344)
```typescript
export type GlobalConfig = {
  // ...
  opus1mMergeNoticeSeenCount?: number; // Number of times the opus-1m-merge notice has been shown
  // ...
}
```

### 2. 功能开关系统
- 通过 `isOpus1mMergeEnabled()` 与模型选择逻辑集成
- 该开关同时控制：
  - 默认模型是否带 `[1m]` 后缀
  - 此通知的展示资格

### 3. 调用方
- **主调用**: `LogoV2.tsx` - 在精简模式和完整模式下都渲染
- **条件渲染**: 
  ```tsx
  // LogoV2.tsx 精简模式
  t13 = <Opus1mMergeNotice />;
  
  // LogoV2.tsx 紧凑/完整模式
  t15 = <Opus1mMergeNotice />;
  ```

---

## 风险、边界与改进建议

### 潜在风险

#### 1. 竞态条件
**问题**: 多个CLI实例同时启动时，可能同时读取和更新 `opus1mMergeNoticeSeenCount`

**当前缓解**: 
```typescript
// 使用函数式更新，基于prev而非外部读取
saveGlobalConfig(prev => {
  if ((prev.opus1mMergeNoticeSeenCount ?? 0) >= newCount) {
    return prev;  // 如果已更新则跳过
  }
  return { ...prev, opus1mMergeNoticeSeenCount: newCount };
});
```

**残余风险**: 配置文件的读写非原子操作，极端情况下仍可能丢失更新

#### 2. 硬编码展示次数
- `MAX_SHOW_COUNT = 6` 是硬编码常量
- 无法通过远程配置动态调整
- 建议：考虑使用 GrowthBook/Statsig 等配置系统

#### 3. 无时间衰减
- 6次展示计数是终身累计，不会随时间重置
- 如果用户数月后重新使用，通知可能已不再相关

### 边界条件

| 场景 | 处理逻辑 |
|-----|---------|
| 配置读取失败 | 组件不渲染（`shouldShowOpus1mMergeNotice` 依赖配置） |
| 动画组件加载失败 | 整个通知不渲染（AnimatedAsterisk是必要依赖） |
| 功能开关关闭 | 组件立即返回 null |
| 已达展示上限 | 组件立即返回 null |
| prefersReducedMotion | 动画立即完成，显示 settled 状态 |

### 改进建议

#### 1. 添加时间衰减
```typescript
const SHOW_COUNT_RESET_DAYS = 90; // 90天后重置计数

function shouldResetCount(lastShownTimestamp: number | undefined): boolean {
  if (!lastShownTimestamp) return false;
  const daysSinceLastShown = (Date.now() - lastShownTimestamp) / (1000 * 60 * 60 * 24);
  return daysSinceLastShown > SHOW_COUNT_RESET_DAYS;
}
```

#### 2. 远程配置支持
```typescript
// 建议: 从配置系统读取最大展示次数
const MAX_SHOW_COUNT = getGrowthBookFeature('opus_1m_notice_max_shows') ?? 6;
```

#### 3. 添加分析事件
```typescript
// 当前缺少展示时的分析追踪
useEffect(() => {
  if (!show) return;
  
  logEvent('opus_1m_merge_notice_shown', {
    seen_count: newCount,
    is_max_subscriber: isMaxSubscriber(),
  });
  // ...
}, [show]);
```

#### 4. 可关闭选项
```typescript
// 建议: 允许用户主动关闭通知
<Text dimColor>
  {" "}Opus now defaults to 1M context · 5x more room, same pricing
  {" "}[<Text color="claude" onPress={dismissPermanently}>Don&apos;t show again</Text>]
</Text>
```

#### 5. 测试覆盖
当前缺少的测试场景：
- 展示次数达到上限后的行为
- 多实例同时启动的竞态条件
- 配置持久化失败的处理
- 动画完成前后的状态变化

---

## 代码量统计
- **总行数**: 55行 (含source map)
- **有效代码**: ~35行
- **复杂度**: 低（单一职责，条件清晰）

## 相关文档
- `docs/feature-gating.md` - 功能开关设计文档
- `src/utils/model/model.ts` - 模型选择和功能开关实现
- `src/components/LogoV2/AnimatedAsterisk.tsx` - 动画组件实现
