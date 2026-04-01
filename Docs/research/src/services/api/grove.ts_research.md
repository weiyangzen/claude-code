# Grove 服务研究文档

## 文件信息
- **路径**: `src/services/api/grove.ts`
- **大小**: 11,543 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Grove 服务负责管理 Claude Code 的 **Grove 隐私通知功能**，这是一个面向 Consumer 订阅用户的隐私政策更新通知系统。该服务处理以下核心场景：

1. **隐私政策更新通知**: 当 Anthropic 更新消费者条款和隐私政策时，向符合条件的用户展示通知对话框
2. **用户选择持久化**: 允许用户选择是否启用 Grove 功能（帮助改进 Claude）
3. **非交互式会话检查**: 在 `-p`/`--print` 模式下检查并提示用户查看更新的条款
4. **资格判定**: 基于用户订阅类型、域名排除规则和 Statsig 配置判断用户是否有资格看到通知

---

## 功能点目的

### 1. 用户设置管理 (`getGroveSettings`)
- 从 `/api/oauth/account/settings` 获取用户当前的 Grove 设置
- 使用 memoize 缓存避免重复请求
- 在更新操作后自动清除缓存

### 2. 配置获取 (`getGroveNoticeConfig`)
- 从 `/api/claude_code_grove` 获取 Statsig 配置的 Grove 配置
- 包含功能开关、域名排除、宽限期状态、提醒频率等
- 3 秒短超时，避免阻塞启动

### 3. 资格检查 (`isQualifiedForGrove`)
- **非阻塞设计**: 使用磁盘缓存，冷启动时返回 false 并在后台获取
- **双层缓存**: 24 小时磁盘缓存 + 会话级内存状态
- 仅对 Consumer 订阅者生效

### 4. 通知展示决策 (`calculateShouldShowGrove`)
- 综合 API 设置和配置判断是否应该展示 Grove 对话框
- 处理宽限期逻辑和提醒频率
- API 失败时保守地返回 false（不展示）

### 5. 非交互式检查 (`checkGroveForNonInteractive`)
- 在 `--print` 模式下检查条款更新
- 宽限期内显示信息性消息，宽限期结束后强制退出

---

## 具体技术实现

### 关键数据结构

```typescript
// 账户设置（来自 OAuth API）
type AccountSettings = {
  grove_enabled: boolean | null  // null 表示尚未选择
  grove_notice_viewed_at: string | null
}

// Grove 配置（来自 Statsig）
type GroveConfig = {
  grove_enabled: boolean
  domain_excluded: boolean        // 企业域名排除
  notice_is_grace_period: boolean // 是否处于宽限期
  notice_reminder_frequency: number | null // 提醒间隔（天）
}

// API 结果包装器（区分失败和成功）
type ApiResult<T> = 
  | { success: true; data: T }
  | { success: false }
```

### 关键流程

#### 资格检查流程
```
isQualifiedForGrove()
├── 检查 isConsumerSubscriber() → false 直接返回
├── 获取 accountId
├── 检查磁盘缓存
│   ├── 无缓存 → 后台获取，返回 false
│   ├── 缓存过期 → 返回缓存值 + 后台刷新
│   └── 缓存有效 → 直接返回
└── 缓存结构: { grove_enabled: boolean, timestamp: number }
```

#### 对话框展示决策流程
```
calculateShouldShowGrove(settings, config, showIfAlreadyViewed)
├── API 失败 → false
├── 已做出选择 (grove_enabled !== null) → false
├── showIfAlreadyViewed → true
├── 非宽限期 → true
└── 检查提醒频率和上次查看时间
```

### 缓存策略

| 缓存层级 | TTL | 用途 |
|---------|-----|------|
| `getGroveSettings.cache` | 会话级 | 避免同一会话内重复请求 |
| `getGroveNoticeConfig.cache` | 会话级 | Statsig 配置缓存 |
| `globalConfig.groveConfigCache` | 24 小时 | 跨进程资格判定 |

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/grove.ts` - 本文件，所有 Grove API 逻辑

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/components/grove/Grove.tsx` | `getGroveSettings`, `getGroveNoticeConfig`, `updateGroveSettings`, `markGroveNoticeViewed` | Grove 对话框 UI |
| `src/interactiveHelpers.tsx` | `isQualifiedForGrove` | 启动时资格检查 |
| `src/cli/print.ts` | `checkGroveForNonInteractive`, `getGroveSettings`, `getGroveNoticeConfig` | 非交互式模式检查 |
| `src/commands/privacy-settings/privacy-settings.tsx` | `getGroveSettings`, `getGroveNoticeConfig`, `isQualifiedForGrove` | 隐私设置页面 |
| `src/commands/logout/logout.tsx` | `getGroveSettings`, `getGroveNoticeConfig` | 登出时清理 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/auth.ts` | `isConsumerSubscriber`, `getOauthAccountInfo` |
| `src/utils/http.ts` | `getAuthHeaders`, `withOAuth401Retry` |
| `src/utils/config.ts` | `getGlobalConfig`, `saveGlobalConfig` |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly` |
| `src/constants/oauth.ts` | `getOauthConfig` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/oauth/account/settings` | GET | 获取用户 Grove 设置 |
| `/api/oauth/account/settings` | PATCH | 更新 grove_enabled |
| `/api/oauth/account/grove_notice_viewed` | POST | 标记通知已查看 |
| `/api/claude_code_grove` | GET | 获取 Statsig 配置 |

### 外部依赖
- **axios**: HTTP 请求
- **lodash-es/memoize**: 函数级缓存
- **@anthropic-ai/sdk**: 间接依赖（通过 auth 模块）

### 配置存储
缓存存储在 `~/.claude/config.json` 中的 `groveConfigCache` 字段：
```json
{
  "groveConfigCache": {
    "account-uuid": {
      "grove_enabled": true,
      "timestamp": 1712345678901
    }
  }
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **缓存失效风险**
   - `getGroveSettings.cache.clear?.()` 使用可选链，某些 lodash 版本可能不支持
   - 失败时清除缓存可能导致竞态条件

2. **非阻塞设计的副作用**
   - 冷启动时 `isQualifiedForGrove()` 返回 false，用户可能错过首次通知
   - 这是有意设计，但可能导致通知延迟

3. **硬编码日期**
   - `checkGroveForNonInteractive` 中硬编码了 "October 8, 2025" 宽限期结束日期
   - 未来政策更新需要代码变更

### 边界情况

| 场景 | 行为 |
|------|------|
| `isEssentialTrafficOnly()` | 所有 API 调用返回 `{ success: false }` |
| 非 Consumer 订阅者 | `isQualifiedForGrove()` 立即返回 false |
| 无 accountId | 立即返回 false |
| API 401 错误 | `withOAuth401Retry` 自动刷新 token 后重试 |
| API 超时 (3s) | `getGroveNoticeConfig` 返回失败，对话框不展示 |

### 改进建议

1. **动态宽限期日期**
   ```typescript
   // 建议从 API 获取宽限期结束日期
   const GRACE_PERIOD_END = config.grace_period_end_date 
     ? new Date(config.grace_period_end_date)
     : new Date('2025-10-08')
   ```

2. **缓存持久化优化**
   - 考虑在 `saveGlobalConfig` 失败时添加降级逻辑
   - 当前 `fetchAndStoreGroveConfig` 在错误时静默失败

3. **测试覆盖**
   - 添加针对 `calculateShouldShowGrove` 的单元测试
   - 测试各种提醒频率组合

4. **监控增强**
   - 添加 Grove 通知展示率的 analytics 事件
   - 目前只有 `tengu_grove_print_viewed` 用于非交互式模式

### 相关 Issue 模式
- 缓存写入放大: 参考 `overageCreditGrant.ts` 中的数据不变性检查模式
- OAuth 401 处理: 与 `metricsOptOut.ts` 使用相同的 `withOAuth401Retry` 模式
