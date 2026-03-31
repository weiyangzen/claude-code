# extra-usage-core.ts 研究文档

## 场景与职责

`extra-usage-core.ts` 是 `/extra-usage` 命令的核心逻辑实现模块，负责处理用户请求额外使用额度（Extra Usage）的完整流程。该模块主要服务于以下场景：

1. **Team/Enterprise 用户无账单权限时**：当组织成员没有直接管理账单的权限，但需要申请增加使用额度或启用额外使用功能时，通过向管理员发送请求来实现
2. **有账单权限的用户**：直接打开浏览器访问 Claude.ai 的账单管理页面
3. **Consumer 计划用户（Pro/Max）**：直接打开浏览器访问个人使用设置页面

该模块是连接 CLI 用户与 Claude.ai Web 账单系统的桥梁，处理权限检查、请求创建、缓存管理等复杂逻辑。

## 功能点目的

### 1. 访问追踪 (`hasVisitedExtraUsage`)
- **目的**：记录用户是否首次访问 `/extra-usage` 命令
- **实现**：通过 `getGlobalConfig().hasVisitedExtraUsage` 检查并更新配置
- **用途**：用于控制相关 Upsell 提示的显示逻辑

### 2. 缓存失效 (`invalidateOverageCreditGrantCache`)
- **目的**：确保用户在多次运行命令时获取最新的额度授予状态
- **实现**：调用 `overageCreditGrant.ts` 中的缓存失效函数
- **时机**：每次命令执行开始时

### 3. 权限与订阅类型检查
- **订阅类型判断**：通过 `getSubscriptionType()` 区分 Team/Enterprise 与 Consumer 计划
- **账单权限检查**：通过 `hasClaudeAiBillingAccess()` 判断用户是否有直接管理权限

### 4. Team/Enterprise 无账单权限流程
当用户属于 Team/Enterprise 计划但没有账单权限时，执行以下检查链：

#### 4.1 无限额度检查
- 调用 `fetchUtilization()` 获取使用情况
- 如果 `extra_usage.is_enabled` 为 true 且 `monthly_limit` 为 null，说明已有无限额度，直接返回提示

#### 4.2 管理员请求资格检查
- 调用 `checkAdminRequestEligibility('limit_increase')`
- 如果 `is_allowed` 为 false，提示用户联系管理员

#### 4.3 待处理请求检查
- 调用 `getMyAdminRequests('limit_increase', ['pending', 'dismissed'])`
- 如果存在待处理或被驳回的请求，提示用户已有请求在处理中

#### 4.4 创建管理员请求
- 调用 `createAdminRequest({ request_type: 'limit_increase', details: null })`
- 根据 `extra_usage.is_enabled` 状态返回不同的成功消息

### 5. 浏览器打开流程
对于有账单权限的用户，根据订阅类型打开不同的 URL：
- **Team/Enterprise**: `https://claude.ai/admin-settings/usage`
- **Consumer (Pro/Max)**: `https://claude.ai/settings/usage`

## 具体技术实现

### 关键流程

```
runExtraUsage()
├── 更新 hasVisitedExtraUsage 配置
├── 失效 overage credit grant 缓存
├── 获取订阅类型和账单权限
├── 判断流程分支
│   ├── 无账单权限 + Team/Enterprise → 管理员请求流程
│   │   ├── 检查是否已有无限额度
│   │   ├── 检查请求资格
│   │   ├── 检查是否已有待处理请求
│   │   ├── 创建新请求
│   │   └── 返回相应消息
│   └── 有账单权限 → 浏览器打开流程
│       ├── 构建对应 URL
│       ├── 调用 openBrowser()
│       └── 返回浏览器打开结果
└── 错误处理（各环节失败后的降级处理）
```

### 数据结构

#### ExtraUsageResult 联合类型
```typescript
type ExtraUsageResult =
  | { type: 'message'; value: string }           // 纯文本消息结果
  | { type: 'browser-opened'; url: string; opened: boolean }  // 浏览器打开结果
```

### 关键代码路径

| 功能 | 代码位置 | 说明 |
|------|----------|------|
| 核心入口 | `runExtraUsage()` (line 18) | 主函数，协调整个流程 |
| 访问追踪 | line 19-21 | 更新 `hasVisitedExtraUsage` |
| 缓存失效 | line 25 | 调用 `invalidateOverageCreditGrantCache()` |
| 权限检查 | line 27-30 | 获取订阅类型和账单权限 |
| 无限额度检查 | line 36-50 | 检查 `extra_usage.is_enabled` 和 `monthly_limit` |
| 资格检查 | line 52-63 | 调用 `checkAdminRequestEligibility()` |
| 待处理请求检查 | line 65-80 | 调用 `getMyAdminRequests()` |
| 创建请求 | line 82-96 | 调用 `createAdminRequest()` |
| 浏览器打开 | line 104-117 | 调用 `openBrowser()` |

### 错误处理策略

模块采用**优雅降级**的错误处理策略：

1. **API 调用失败不阻断**：`fetchUtilization()`、`checkAdminRequestEligibility()`、`getMyAdminRequests()` 的失败都被捕获并记录，继续执行后续流程
2. **创建请求失败兜底**：如果 `createAdminRequest()` 失败，最终返回通用提示 "Please contact your admin..."
3. **浏览器打开失败处理**：`openBrowser()` 失败时返回包含 URL 的文本消息，引导用户手动访问

## 依赖与外部交互

### 导入依赖

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `../../services/api/adminRequests.js` | `checkAdminRequestEligibility`, `createAdminRequest`, `getMyAdminRequests` | 管理员请求 API |
| `../../services/api/overageCreditGrant.js` | `invalidateOverageCreditGrantCache` | 缓存失效 |
| `../../services/api/usage.js` | `ExtraUsage`, `fetchUtilization` | 使用情况查询 |
| `../../utils/auth.js` | `getSubscriptionType` | 订阅类型获取 |
| `../../utils/billing.js` | `hasClaudeAiBillingAccess` | 账单权限检查 |
| `../../utils/browser.js` | `openBrowser` | 浏览器打开 |
| `../../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 配置读写 |
| `../../utils/log.js` | `logError` | 错误日志 |

### API 调用详情

#### 1. fetchUtilization()
- **端点**: `GET /api/oauth/usage`
- **用途**: 获取用户使用情况，包括 extra_usage 状态
- **返回**: `Utilization` 对象，包含 `extra_usage` 字段

#### 2. checkAdminRequestEligibility('limit_increase')
- **端点**: `GET /api/oauth/organizations/{orgUUID}/admin_requests/eligibility?request_type=limit_increase`
- **用途**: 检查当前用户是否可以创建额度增加请求
- **返回**: `{ request_type: string; is_allowed: boolean }`

#### 3. getMyAdminRequests('limit_increase', ['pending', 'dismissed'])
- **端点**: `GET /api/oauth/organizations/{orgUUID}/admin_requests/me?request_type=limit_increase&statuses=pending&statuses=dismissed`
- **用途**: 获取当前用户的待处理或被驳回的请求
- **返回**: `AdminRequest[] | null`

#### 4. createAdminRequest({ request_type: 'limit_increase', details: null })
- **端点**: `POST /api/oauth/organizations/{orgUUID}/admin_requests`
- **用途**: 创建新的额度增加请求
- **请求体**: `{ request_type: 'limit_increase', details: null }`
- **返回**: `AdminRequest`

## 风险、边界与改进建议

### 潜在风险

1. **API 依赖风险**
   - 所有 API 调用都使用 try-catch 包裹，但连续失败可能导致用户体验不佳（多次看到降级提示）
   - 建议：增加更细粒度的错误分类，区分网络错误、权限错误、服务端错误

2. **状态一致性风险**
   - `hasVisitedExtraUsage` 在每次调用时都更新，即使后续流程失败
   - 建议：考虑在流程成功完成后再更新访问标记

3. **竞态条件**
   - 多次快速调用 `/extra-usage` 可能创建重复的管理员请求
   - 缓解：`getMyAdminRequests` 检查可以捕获大部分情况，但不是原子操作

### 边界情况

1. **订阅类型变更**：用户在中途变更订阅类型（如从 Pro 升级到 Team），下次调用时会正确路由
2. **权限变更**：用户被赋予/撤销账单权限，下次调用时会正确路由
3. **网络中断**：各环节都有降级处理，不会导致 CLI 崩溃
4. **浏览器不可用**：`openBrowser` 返回 `opened: false`，会显示 URL 让用户手动访问

### 改进建议

1. **增加重试机制**
   ```typescript
   // 建议：对关键 API 调用增加指数退避重试
   const eligibility = await withRetry(
     () => checkAdminRequestEligibility('limit_increase'),
     { maxRetries: 3, backoff: 'exponential' }
   )
   ```

2. **优化缓存策略**
   - 当前每次调用都失效缓存，可以考虑更智能的缓存策略
   - 例如：仅在确认有变更时才失效

3. **增加遥测**
   - 记录各步骤的成功率、失败原因
   - 帮助识别用户痛点和系统问题

4. **国际化支持**
   - 当前所有提示文本都是硬编码英文
   - 建议：使用 i18n 框架支持多语言

5. **UI 优化**
   - 对于 Team/Enterprise 用户，可以考虑在 CLI 中直接显示当前额度状态
   - 减少用户需要访问 Web 的频率
