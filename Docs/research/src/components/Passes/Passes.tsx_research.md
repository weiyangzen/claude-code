# Passes.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位
`Passes.tsx` 是 Claude Code CLI 中 `/passes` 命令的核心 UI 组件，负责展示**Guest Passes（访客通行证）**功能界面。该功能允许 Claude Code 的付费订阅用户（Max 套餐）分享免费试用链接给好友，同时根据推荐计划获得额外使用额度奖励。

### 1.2 业务场景
- **目标用户**: Claude.ai 的 Max 订阅用户（`isClaudeAISubscriber() && getSubscriptionType() === 'max'`）
- **功能场景**: 
  - 展示用户可用的 Guest Pass 数量（默认最多3个）
  - 显示推荐链接供用户复制分享
  - 展示推荐奖励信息（v1 活动：如果被推荐人订阅，推荐人获得额外使用额度）
  - 追踪 Pass 的使用状态（已使用/未使用）

### 1.3 产品价值
- 用户增长：通过现有用户推荐获取新用户
- 用户留存：通过奖励机制激励现有用户持续使用
- 病毒式传播：利用社交关系链扩大产品影响力

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 | 用户交互 |
|--------|------|----------|
| **资格检查** | 验证用户是否为 Max 订阅用户，是否有权使用 Guest Passes | 自动进行，无感加载 |
| **Pass 状态展示** | 以 ASCII 票券形式直观展示可用/已使用的 Pass | 可视化票券，可用为彩色，已使用为灰色带斜杠 |
| **推荐链接复制** | 允许用户一键复制推荐链接到剪贴板 | 按 Enter 键复制 |
| **奖励信息展示** | 显示推荐成功后可获得的奖励金额 | 文案动态根据活动版本变化 |
| **取消/退出** | 支持 Esc 取消或 Ctrl+C/D 退出 | 标准退出交互 |

### 2.2 状态流转

```
[Loading] → [检查资格] → [不合格] → [展示不可用提示]
                    ↓
              [合格] → [获取 Redemptions 数据] → [构建 Pass 状态数组]
                    ↓
              [渲染票券 UI] → [等待用户操作]
                    ↓
              [Enter: 复制链接] / [Esc: 取消]
```

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 PassStatus（组件内部类型）
```typescript
type PassStatus = {
  passNumber: number;    // Pass 序号（1-3）
  isAvailable: boolean;  // 是否可用（未被使用）
};
```

#### 3.1.2 依赖的外部类型（来自 `services/oauth/types.js`）
```typescript
// 推荐活动响应
interface ReferralEligibilityResponse {
  eligible: boolean;
  referral_code_details?: {
    referral_link: string;
    campaign: string;
  };
  referrer_reward?: ReferrerRewardInfo;
  remaining_passes?: number;
}

// 兑换记录响应
interface ReferralRedemptionsResponse {
  redemptions: Array<{
    redeemed_at: string;
    // ... 其他字段
  }>;
  limit: number;
}

// 推荐人奖励信息
interface ReferrerRewardInfo {
  amount_minor_units: number;  // 金额（最小单位，如美分）
  currency: string;            // 货币代码（USD/EUR/GBP等）
}
```

### 3.2 核心流程

#### 3.2.1 数据加载流程（`loadPassesData`）

```typescript
async function loadPassesData() {
  // 1. 检查资格（使用缓存优先策略）
  const eligibilityData = await getCachedOrFetchPassesEligibility();
  if (!eligibilityData?.eligible) {
    setIsAvailable(false);
    return;
  }

  // 2. 提取推荐链接和奖励信息
  setReferralLink(eligibilityData.referral_code_details?.referral_link);
  setReferrerReward(eligibilityData.referrer_reward);

  // 3. 确定活动名称（默认 claude_code_guest_pass）
  const campaign = eligibilityData.referral_code_details?.campaign ?? 'claude_code_guest_pass';

  // 4. 获取兑换记录
  const redemptionsData = await fetchReferralRedemptions(campaign);

  // 5. 构建 Pass 状态数组
  const redemptions = redemptionsData.redemptions || [];
  const maxRedemptions = redemptionsData.limit || 3;
  const statuses: PassStatus[] = [];
  for (let i = 0; i < maxRedemptions; i++) {
    const redemption = redemptions[i];
    statuses.push({
      passNumber: i + 1,
      isAvailable: !redemption  // 无兑换记录 = 可用
    });
  }
  setPassStatuses(statuses);
}
```

#### 3.2.2 票券渲染逻辑

```typescript
const renderTicket = (pass: PassStatus) => {
  const isRedeemed = !pass.isAvailable;
  if (isRedeemed) {
    // 已使用：灰色 + 斜杠效果
    return (
      <Box flexDirection="column" marginRight={1}>
        <Text dimColor>{'┌─────────╱'}</Text>
        <Text dimColor>{` ) CC ${TEARDROP_ASTERISK} ┊╱`}</Text>
        <Text dimColor>{'└───────╱'}</Text>
      </Box>
    );
  }
  // 未使用：彩色 + 完整边框
  return (
    <Box flexDirection="column" marginRight={1}>
      <Text>{'┌──────────┐'}</Text>
      <Text>{' ) CC '}<Text color="claude">{TEARDROP_ASTERISK}</Text>{' ┊ ( '}</Text>
      <Text>{'└──────────┘'}</Text>
    </Box>
  );
};
```

#### 3.2.3 剪贴板复制流程

```typescript
useInput((_input, key) => {
  if (key.return && referralLink) {
    void setClipboard(referralLink).then(raw => {
      if (raw) process.stdout.write(raw);
      logEvent('tengu_guest_passes_link_copied', {});
      onDone(`Referral link copied to clipboard!`);
    });
  }
});
```

复制流程涉及：
1. **OSC 52 序列**: 通过终端控制序列写入剪贴板
2. **原生工具回退**: macOS 使用 `pbcopy`，Linux 使用 `wl-copy`/`xclip`/`xsel`，Windows 使用 `clip.exe`
3. **tmux 支持**: 在 tmux 会话中使用 `tmux load-buffer`

### 3.3 缓存策略

组件依赖 `referral.ts` 中的多层缓存机制：

| 缓存层级 | 位置 | 有效期 | 用途 |
|----------|------|--------|------|
| GlobalConfig | `~/.claude.json` | 24小时 | 持久化资格数据 |
| 内存缓存 | `fetchInProgress` | 单次请求 | 防止重复 API 调用 |
| 冷启动处理 | 无缓存时返回 null | N/A | 首次启动后台获取，下次可用 |

缓存数据结构：
```typescript
// GlobalConfig 中的缓存结构
passesEligibilityCache?: Record<string, ReferralEligibilityResponse & { timestamp: number }>
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

```
用户执行 /passes
    ↓
src/commands/passes/index.ts (命令定义)
    ↓
src/commands/passes/passes.tsx (命令实现)
    ↓
src/components/Passes/Passes.tsx (本组件)
    ↓
src/services/api/referral.ts (API 服务)
    ↓
Claude.ai API (/api/oauth/organizations/{orgUUID}/referral/*)
```

### 4.2 关键文件清单

| 文件路径 | 职责 | 与本组件关系 |
|----------|------|--------------|
| `src/components/Passes/Passes.tsx` | Guest Passes UI 组件 | **核心文件** |
| `src/commands/passes/index.ts` | 命令定义与懒加载配置 | 调用方 |
| `src/commands/passes/passes.tsx` | 命令实现，包装组件 | 直接调用本组件 |
| `src/services/api/referral.ts` | API 调用与缓存逻辑 | 数据层依赖 |
| `src/services/oauth/types.js` | OAuth/Referral 类型定义 | 类型依赖 |
| `src/utils/config.ts` | GlobalConfig 管理 | 缓存存储 |
| `src/components/LogoV2/GuestPassesUpsell.tsx` | Upsell 提示组件 | 相关功能 |
| `src/services/tips/tipRegistry.ts` | 提示注册表 | 包含 guest-passes 提示 |
| `src/ink/termio/osc.ts` | 剪贴板 OSC 序列 | 复制功能依赖 |
| `src/hooks/useExitOnCtrlCDWithKeybindings.ts` | 退出快捷键处理 | 交互依赖 |
| `src/components/design-system/Pane.tsx` | 面板容器组件 | UI 布局依赖 |

### 4.3 API 端点

```typescript
// 资格检查
GET /api/oauth/organizations/{orgUUID}/referral/eligibility?campaign=claude_code_guest_pass

// 兑换记录
GET /api/oauth/organizations/{orgUUID}/referral/redemptions?campaign=claude_code_guest_pass
```

---

## 5. 依赖与外部交互

### 5.1 直接依赖

```typescript
// React 核心
import * as React from 'react';
import { useCallback, useEffect, useState } from 'react';

// 类型
import type { CommandResultDisplay } from '../../commands.js';
import type { ReferralRedemptionsResponse, ReferrerRewardInfo } from '../../services/oauth/types.js';

// 常量
import { TEARDROP_ASTERISK } from '../../constants/figures.js';  // ✻ 符号

// Hooks
import { useExitOnCtrlCDWithKeybindings } from '../../hooks/useExitOnCtrlCDWithKeybindings.js';
import { useKeybinding } from '../../keybindings/useKeybinding.js';

// 工具函数
import { setClipboard } from '../../ink/termio/osc.js';  // 剪贴板操作
import { logEvent } from '../../services/analytics/index.js';  // 埋点
import { count } from '../../utils/array.js';  // 数组计数
import { logError } from '../../utils/log.js';  // 错误日志

// API 服务
import { 
  fetchReferralRedemptions, 
  formatCreditAmount, 
  getCachedOrFetchPassesEligibility 
} from '../../services/api/referral.js';

// UI 组件
import { Box, Link, Text, useInput } from '../../ink.js';
import { Pane } from '../design-system/Pane.js';
```

### 5.2 配置依赖

| 配置项 | 位置 | 说明 |
|--------|------|------|
| `passesEligibilityCache` | `GlobalConfig` | 资格数据缓存 |
| `hasVisitedPasses` | `GlobalConfig` | 是否访问过 /passes |
| `passesUpsellSeenCount` | `GlobalConfig` | Upsell 展示次数 |
| `passesLastSeenRemaining` | `GlobalConfig` | 上次看到的剩余 Pass 数 |

### 5.3 埋点事件

| 事件名 | 触发时机 | 元数据 |
|--------|----------|--------|
| `tengu_guest_passes_link_copied` | 用户复制推荐链接 | `{}` |
| `tengu_guest_passes_visited` | 访问 /passes 命令 | `{ is_first_visit: boolean }` |
| `tengu_guest_passes_upsell_shown` | Upsell 提示展示 | `{ seen_count: number }` |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 类型定义缺失风险
- **问题**: `ReferralEligibilityResponse`、`ReferralRedemptionsResponse`、`ReferrerRewardInfo` 等类型从 `services/oauth/types.js` 导入，但该文件在源码中不存在（可能是构建时生成或来自外部包）
- **影响**: 类型检查可能失败，IDE 跳转受限
- **缓解**: 确保构建流程正确生成或链接类型定义

#### 6.1.2 缓存失效风险
- **问题**: 缓存有效期为 24 小时，用户订阅状态变更后可能延迟感知
- **影响**: 用户升级/降级后，资格判断可能暂时不准确
- **缓解**: 后台刷新机制（`fetchAndStorePassesEligibility`）在缓存过期时自动更新

#### 6.1.3 API 失败降级
- **问题**: `fetchReferralRedemptions` 失败时，组件会降级为 "不可用" 状态
- **代码位置**: 
  ```typescript
  try {
    redemptionsData = await fetchReferralRedemptions(campaign);
  } catch (err_0) {
    logError(err_0 as Error);
    setIsAvailable(false);  // 降级处理
    setLoading(false);
    return;
  }
  ```

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 冷启动无缓存 | 返回 null，命令不可用，后台获取数据 |
| 缓存过期 | 返回过期数据，后台刷新 |
| 无推荐链接 | 不显示链接区域 |
| 无奖励信息 | 显示默认文案（无奖励金额） |
| 所有 Pass 已使用 | 显示 0 left，所有票券为灰色 |

### 6.3 改进建议

#### 6.3.1 代码质量
1. **类型安全**: 为 `services/oauth/types.js` 添加明确的类型定义文件，或确认其生成来源
2. **错误处理**: 当前 API 错误仅记录日志，可考虑向用户展示更友好的错误提示
3. **加载状态**: 加载期间显示 "Loading guest pass information…"，可考虑添加进度指示

#### 6.3.2 用户体验
1. **重试机制**: API 失败后可提供手动重试按钮
2. **链接预览**: 复制前显示链接预览，确认内容正确
3. **分享渠道**: 除复制链接外，可考虑集成直接分享（如邮件、Slack）

#### 6.3.3 性能优化
1. **预加载**: 已在 `main.tsx` 中通过 `prefetchPassesEligibility()` 实现启动时预加载
2. **防抖**: 考虑在快速重复打开 /passes 时添加防抖

#### 6.3.4 可维护性
1. **常量提取**: ASCII 票券的硬编码字符串可提取为常量
2. **文案国际化**: 当前文案为硬编码英文，可考虑 i18n 支持
3. **测试覆盖**: 建议添加单元测试覆盖资格检查、状态构建、错误处理等逻辑

### 6.4 相关 TODO/FIXME

- 代码中无显式 TODO，但存在 ESLint 忽略注释：
  ```typescript
  // eslint-disable-next-line custom-rules/prefer-use-keybindings -- enter to copy link
  import { Box, Link, Text, useInput } from '../../ink.js';
  ```
  说明 Enter 键复制链接使用 `useInput` 而非 `useKeybinding` 是有意为之。

---

## 7. 附录

### 7.1 ASCII 票券样式

**可用状态：**
```
┌──────────┐
 ) CC ✻ ┊ ( 
└──────────┘
```

**已使用状态：**
```
┌─────────╱
 ) CC ✻ ┊╱
└───────╱
```

### 7.2 货币符号映射

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
};
```

### 7.3 活动版本

- **默认活动**: `claude_code_guest_pass`
- **v1 活动**: 包含 `referrer_reward` 字段，显示奖励金额
- **活动切换**: 通过 `eligibilityData.referral_code_details?.campaign` 动态获取
