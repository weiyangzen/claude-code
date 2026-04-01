# Grove.tsx 深度研究文档

## 1. 场景与职责

### 1.1 业务背景
Grove 是 Claude Code CLI 中的**隐私政策同意与数据使用授权**模块，用于处理用户对于"帮助改进 Claude"功能（即允许使用聊天记录训练 AI 模型）的授权同意。该模块涉及以下业务场景：

1. **消费者条款更新通知**（Consumer Terms Update）：2025年10月8日生效的新隐私政策需要用户明确同意
2. **数据保留策略选择**：用户可选择开启（5年保留）或关闭（30天保留）数据用于模型改进
3. **企业域排除**：对于使用企业邮箱域的用户，自动禁用数据收集功能

### 1.2 核心职责

| 组件 | 职责 |
|------|------|
| `GroveDialog` | 主对话框组件，用于首次展示隐私政策更新通知，收集用户同意决策 |
| `PrivacySettingsDialog` | 隐私设置子对话框，用于已同意用户后续修改设置 |
| `GracePeriodContentBody` | 宽限期内的内容展示（2025年10月8日前） |
| `PostGracePeriodContentBody` | 宽限期后的内容展示（2025年10月8日后） |

### 1.3 触发场景
- **Onboarding 流程**：新用户首次使用时的引导流程
- **Policy Update Modal**：政策更新时的弹窗通知
- **Settings 命令**：用户主动通过 `/privacy-settings` 命令查看/修改设置
- **Non-interactive 模式**：`--print` 模式下的命令行输出提示

---

## 2. 功能点目的

### 2.1 用户决策类型（GroveDecision）

```typescript
export type GroveDecision = 
  | 'accept_opt_in'   // 接受条款，开启数据收集
  | 'accept_opt_out'  // 接受条款，关闭数据收集
  | 'defer'           // 宽限期内推迟决定
  | 'escape'          // 宽限期后拒绝（将导致退出）
  | 'skip_rendering'; // 无需展示对话框
```

### 2.2 功能特性矩阵

| 功能 | 描述 | 实现位置 |
|------|------|----------|
| 宽限期检测 | 根据服务器配置判断当前是否处于宽限期 | `grove.ts: calculateShouldShowGrove` |
| 域排除检测 | 检测用户是否使用企业域邮箱，自动禁用数据收集 | `Grove.tsx: acceptOptions 动态生成` |
| 提醒频率控制 | 根据 `notice_reminder_frequency` 控制提醒间隔 | `grove.ts: calculateShouldShowGrove` |
| 已查看标记 | 记录用户已查看通知，避免重复展示 | `grove.ts: markGroveNoticeViewed` |
| 非交互式检查 | `--print` 模式下输出文本提示 | `grove.ts: checkGroveForNonInteractive` |

### 2.3 数据保留策略

| 用户选择 | 数据保留期 | 用途 |
|----------|-----------|------|
| ON (opt_in) | 5年 | 用于训练和改进 Anthropic AI 模型 |
| OFF (opt_out) | 30天 | 仅用于服务运营，不用于模型训练 |
| 企业域用户 | 强制 OFF | 企业域自动排除 |

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 AccountSettings（用户账户设置）
```typescript
// src/services/api/grove.ts:25-28
export type AccountSettings = {
  grove_enabled: boolean | null  // null = 尚未做出选择
  grove_notice_viewed_at: string | null  // ISO 8601 时间戳
}
```

#### 3.1.2 GroveConfig（服务器配置）
```typescript
// src/services/api/grove.ts:30-35
export type GroveConfig = {
  grove_enabled: boolean           // 功能是否启用
  domain_excluded: boolean         // 是否属于排除域
  notice_is_grace_period: boolean  // 是否处于宽限期
  notice_reminder_frequency: number | null  // 提醒频率（天）
}
```

#### 3.1.3 Props 接口
```typescript
// Grove.tsx:11-15
type Props = {
  showIfAlreadyViewed: boolean;                    // 是否对已查看用户展示
  location: 'settings' | 'policy_update_modal' | 'onboarding';  // 触发位置
  onDone(decision: GroveDecision): void;           // 完成回调
};
```

### 3.2 核心流程

#### 3.2.1 对话框展示决策流程

```
┌─────────────────────────────────────────────────────────────┐
│                    isQualifiedForGrove()                    │
│  1. 检查是否为 consumer subscriber                          │
│  2. 检查是否有有效的 account UUID                           │
│  3. 检查缓存的 groveConfigCache（24小时过期）               │
└──────────────────────────┬──────────────────────────────────┘
                           │ 是
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              calculateShouldShowGrove()                     │
│  1. API 调用失败 → 返回 false（隐藏对话框）                 │
│  2. grove_enabled !== null → 已选择 → 返回 false           │
│  3. showIfAlreadyViewed=true → 返回 true                   │
│  4. 非宽限期 → 返回 true（必须选择）                        │
│  5. 计算距上次查看天数 >= reminderFrequency → 返回 true    │
│  6. 从未查看 → 返回 true                                    │
└──────────────────────────┬──────────────────────────────────┘
                           │ 是
                           ▼
                    ┌──────────────┐
                    │ 展示 GroveDialog │
                    └──────────────┘
```

#### 3.2.2 用户选择处理流程

```typescript
// Grove.tsx:194-227
async function onChange(value: GroveDecision) {
  switch (value) {
    case 'accept_opt_in':
      await updateGroveSettings(true);   // 调用 API 开启
      logEvent('tengu_grove_policy_submitted', { state: true });
      break;
    case 'accept_opt_out':
      await updateGroveSettings(false);  // 调用 API 关闭
      logEvent('tengu_grove_policy_submitted', { state: false });
      break;
    case 'defer':
      logEvent('tengu_grove_policy_dismissed', { state: true });
      break;
    case 'escape':
      logEvent('tengu_grove_policy_escaped', {});
      break;
  }
  onDone(value);
}
```

#### 3.2.3 非交互式模式处理

```typescript
// src/services/api/grove.ts:323-357
export async function checkGroveForNonInteractive(): Promise<void> {
  const shouldShowGrove = calculateShouldShowGrove(...);
  if (shouldShowGrove) {
    if (config.notice_is_grace_period) {
      // 宽限期：仅输出提示，继续执行
      writeToStderr('...will take effect on October 8...');
      await markGroveNoticeViewed();
    } else {
      // 宽限期后：输出错误，强制退出
      writeToStderr('[ACTION REQUIRED]...');
      await gracefulShutdown(1);
    }
  }
}
```

### 3.3 API 接口协议

#### 3.3.1 获取用户设置
```
GET /api/oauth/account/settings
Authorization: Bearer {oauth_token}
User-Agent: {claude_code_user_agent}

Response: AccountSettings
```

#### 3.3.2 获取 Grove 配置
```
GET /api/claude_code_grove
Authorization: Bearer {oauth_token}
User-Agent: {claude_code_user_agent}
Timeout: 3000ms

Response: GroveConfig
```

#### 3.3.3 更新设置
```
PATCH /api/oauth/account/settings
Authorization: Bearer {oauth_token}
Content-Type: application/json

Body: { "grove_enabled": boolean }
```

#### 3.3.4 标记已查看
```
POST /api/oauth/account/grove_notice_viewed
Authorization: Bearer {oauth_token}
```

### 3.4 缓存机制

```typescript
// src/services/api/grove.ts:22-23
const GROVE_CACHE_EXPIRATION_MS = 24 * 60 * 60 * 1000; // 24小时

// src/utils/config.ts:307-311
groveConfigCache?: Record<string, {
  grove_enabled: boolean;
  timestamp: number;
}>
```

缓存策略：
1. **内存缓存**：`getGroveSettings` 和 `getGroveNoticeConfig` 使用 `lodash/memoize` 进行会话级缓存
2. **磁盘缓存**：`groveConfigCache` 存储在全局配置中，按 account UUID 索引
3. **缓存失效**：更新设置后调用 `getGroveSettings.cache.clear?.()`

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 职责 |
|----------|------|
| `src/components/grove/Grove.tsx` | UI 组件实现（Dialog、PrivacySettingsDialog） |
| `src/services/api/grove.ts` | API 调用、缓存逻辑、资格检查 |
| `src/utils/config.ts` | 全局配置类型定义（含 groveConfigCache） |
| `src/utils/privacyLevel.ts` | 隐私级别控制（essential-traffic 模式跳过 Grove） |

### 4.2 调用方文件

| 文件路径 | 调用方式 | 用途 |
|----------|----------|------|
| `src/interactiveHelpers.tsx:191-201` | `isQualifiedForGrove()` + 动态导入 `GroveDialog` | Onboarding 流程展示 |
| `src/commands/privacy-settings/privacy-settings.tsx` | 导入 `GroveDialog` 和 `PrivacySettingsDialog` | `/privacy-settings` 命令 |
| `src/cli/print.ts:558-560` | `isQualifiedForGrove()` + `checkGroveForNonInteractive()` | `--print` 模式检查 |
| `src/commands/logout/logout.tsx:63-64` | 清除 `getGroveNoticeConfig.cache` 和 `getGroveSettings.cache` | 登出时清理缓存 |

### 4.3 依赖组件

| 文件路径 | 用途 |
|----------|------|
| `src/components/design-system/Dialog.tsx` | 对话框容器组件 |
| `src/components/design-system/Byline.tsx` | 键盘快捷键提示分隔符 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 键盘快捷键提示 |
| `src/components/CustomSelect/select.tsx` | 选项选择器组件 |
| `src/ink.js` | Ink React 渲染库（Box、Text、Link、useInput） |
| `src/services/analytics/index.js` | 事件上报（logEvent） |

### 4.4 关键代码路径图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        交互式模式入口                                │
│  src/interactiveHelpers.tsx:showSetupScreens()                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐      ┌─────────────────┐     ┌──────────────┐
│   Onboarding  │      │ Policy Update   │     │   Settings   │
│   (location)  │      │   Modal         │     │   Command    │
└───────┬───────┘      └────────┬────────┘     └──────┬───────┘
        │                       │                     │
        └───────────────────────┼─────────────────────┘
                                ▼
                  ┌─────────────────────────┐
                  │  isQualifiedForGrove()  │
                  │  (src/services/api/     │
                  │       grove.ts:157)     │
                  └───────────┬─────────────┘
                              │ 是
                              ▼
                  ┌─────────────────────────┐
                  │ calculateShouldShowGrove│
                  │ (grove.ts:284)          │
                  └───────────┬─────────────┘
                              │ 是
                              ▼
                  ┌─────────────────────────┐
                  │     GroveDialog         │
                  │ (Grove.tsx:144)         │
                  └───────────┬─────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │  Grace   │   │  Post    │   │   Privacy    │
        │  Period  │   │  Grace   │   │  Settings    │
        │  Content │   │  Period  │   │  Dialog      │
        └──────────┘   └──────────┘   └──────────────┘
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
// React 生态
import React, { useEffect, useState } from 'react';

// Ink TUI 库
import { Box, Link, Text, useInput } from '../../ink.js';

// 内部服务
import { logEvent } from 'src/services/analytics/index.js';
import { 
  getGroveNoticeConfig, 
  getGroveSettings, 
  markGroveNoticeViewed, 
  updateGroveSettings,
  calculateShouldShowGrove,
  type AccountSettings,
  type GroveConfig 
} from '../../services/api/grove.js';

// UI 组件
import { Select } from '../CustomSelect/index.js';
import { Byline } from '../design-system/Byline.js';
import { Dialog } from '../design-system/Dialog.js';
import { KeyboardShortcutHint } from '../design-system/KeyboardShortcutHint.js';
```

### 5.2 事件上报

| 事件名 | 触发时机 | 参数 |
|--------|----------|------|
| `tengu_grove_policy_viewed` | 对话框展示时 | `location`, `dismissable` |
| `tengu_grove_policy_submitted` | 用户提交选择时 | `state` (true/false), `dismissable` |
| `tengu_grove_policy_dismissed` | 用户推迟决定时 | `state` |
| `tengu_grove_policy_escaped` | 用户按 Esc 退出时 | - |
| `tengu_grove_policy_exited` | 用户选择 escape 导致退出时 | - |
| `tengu_grove_privacy_settings_viewed` | 隐私设置对话框展示时 | - |
| `tengu_grove_policy_toggled` | 设置被修改时 | `state`, `location` |
| `tengu_grove_print_viewed` | 非交互模式下提示展示时 | `dismissable` |

### 5.3 与 OAuth 的交互

所有 Grove API 调用都通过 `withOAuth401Retry` 包装，处理 OAuth token 过期重试：

```typescript
// src/services/api/grove.ts:59-72
const response = await withOAuth401Retry(() => {
  const authHeaders = getAuthHeaders();
  return axios.get<AccountSettings>(
    `${getOauthConfig().BASE_API_URL}/api/oauth/account/settings`,
    { headers: { ...authHeaders.headers, 'User-Agent': getClaudeCodeUserAgent() } }
  );
});
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 API 失败处理
- **风险**：Grove API 失败会导致对话框被隐藏，用户可能在不知情的情况下继续使用
- **代码位置**：`grove.ts:289-292`
- **缓解**：API 失败时保守处理（不阻断用户），但可能错过重要的政策更新通知

#### 6.1.2 缓存一致性问题
- **风险**：`getGroveSettings` 使用 memoize 缓存，但在 `updateGroveSettings` 后手动清除缓存
- **代码位置**：`grove.ts:144`
- **潜在问题**：如果清除失败，后续读取可能获取旧值

#### 6.1.3 时间边界问题
- **风险**：宽限期判断依赖服务器时间，但客户端使用 `Date.now()` 计算提醒间隔
- **代码位置**：`grove.ts:311-314`
- **潜在问题**：客户端时间不准确可能导致提醒频率异常

#### 6.1.4 非交互模式强制退出
- **风险**：宽限期后非交互模式会强制退出（exit code 1）
- **代码位置**：`grove.ts:351-354`
- **影响**：CI/CD 流程可能被中断

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 用户从未登录 | `isQualifiedForGrove()` 返回 false，不展示对话框 |
| 企业域用户 | `domain_excluded=true`，仅展示 "Accept terms · Help improve Claude: OFF (for emails with your domain)" 选项 |
| 已选择用户再次访问 | `grove_enabled !== null`，展示 `PrivacySettingsDialog` 而非 `GroveDialog` |
| 宽限期内推迟 | 允许继续，记录 `tengu_grove_policy_dismissed` |
| 宽限期后按 Esc | 调用 `gracefulShutdownSync(0)` 退出应用 |
| 隐私级别为 essential-traffic | `isEssentialTrafficOnly()` 返回 true，跳过所有 Grove 相关请求 |

### 6.3 改进建议

#### 6.3.1 缓存策略优化
```typescript
// 建议：使用更可靠的缓存失效机制
// 当前：依赖手动调用 cache.clear()
// 改进：使用 TTL 缓存或版本号机制
```

#### 6.3.2 错误处理增强
```typescript
// 建议：API 失败时提供降级 UI 而非完全隐藏
// 当前：calculateShouldShowGrove 返回 false 隐藏对话框
// 改进：展示简化版提示，引导用户访问网页设置
```

#### 6.3.3 时间同步
```typescript
// 建议：使用服务器时间计算提醒间隔
// 当前：使用客户端 Date.now()
// 改进：API 返回服务器时间戳，或完全依赖服务器判断提醒时机
```

#### 6.3.4 测试覆盖
- 当前未发现针对 Grove 的单元测试文件
- 建议添加：
  - `calculateShouldShowGrove` 的各种边界条件测试
  - 缓存失效逻辑测试
  - 企业域用户场景测试
  - 宽限期前后行为测试

#### 6.3.5 代码组织
- `Grove.tsx` 文件包含 463 行，职责较集中
- 建议考虑将 `GracePeriodContentBody` 和 `PostGracePeriodContentBody` 拆分为独立文件
- 建议将 ASCII 艺术（NEW_TERMS_ASCII）移至常量文件

### 6.4 监控建议

| 指标 | 用途 |
|------|------|
| `tengu_grove_policy_viewed` 触发率 | 监控对话框展示频率 |
| `tengu_grove_policy_submitted` 转化率 | 监控用户决策完成率 |
| `tengu_grove_policy_dismissed` 频率 | 监控推迟决策的用户比例 |
| API 失败率 | 监控服务端可用性 |
| 缓存命中率 | 优化缓存策略 |

---

## 7. 附录

### 7.1 相关链接

- 消费者条款：https://anthropic.com/legal/terms
- 隐私政策：https://anthropic.com/legal/privacy
- 数据隐私控制：https://claude.ai/settings/data-privacy-controls
- 政策更新公告：https://www.anthropic.com/news/updates-to-our-consumer-terms

### 7.2 版本信息

- 研究日期：2026-04-01
- 目标文件版本：基于当前工作目录 HEAD
- 政策生效日期：2025-10-08（代码中硬编码）
