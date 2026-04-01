# 研究文档: src/services/api/adminRequests.ts

## 场景与职责

`adminRequests.ts` 是 Claude Code CLI 中用于处理**管理员请求**的 API 服务模块。它主要服务于 Team/Enterprise 组织成员，这些成员没有计费/管理员权限但需要请求额外的资源配额。

### 核心使用场景

1. **限额提升请求 (limit_increase)**: 当 Team/Enterprise 用户达到使用限额时，可以向组织管理员发起增加配额的请求
2. **席位升级请求 (seat_upgrade)**: 用户请求升级其席位等级

### 业务背景

- 在大型组织中，普通成员通常没有直接管理计费设置的权限
- 该模块提供了一个"向上申请"的通道，让普通成员能够发起资源请求
- 管理员可以在 Claude.ai 的管理后台查看并处理这些请求

---

## 功能点目的

### 1. 创建管理员请求 (`createAdminRequest`)

**目的**: 为当前用户创建一个新的管理员请求。

**关键行为**:
- 如果同类型请求已存在且处于 pending 状态，则返回现有请求（幂等性）
- 支持两种请求类型: `limit_increase` 和 `seat_upgrade`
- 需要 OAuth 认证和有效的组织 UUID

### 2. 获取我的管理员请求 (`getMyAdminRequests`)

**目的**: 查询当前用户的特定类型的待处理请求。

**使用场景**:
- 检查用户是否已经提交过请求（避免重复提交）
- 查看请求状态（pending/approved/dismissed）

### 3. 检查请求资格 (`checkAdminRequestEligibility`)

**目的**: 检查当前组织是否允许特定类型的管理员请求。

**使用场景**:
- 在显示请求 UI 之前预先检查资格
- 避免向没有权限的组织显示请求选项

---

## 具体技术实现

### 关键数据类型

```typescript
// 请求类型
export type AdminRequestType = 'limit_increase' | 'seat_upgrade'

// 请求状态
export type AdminRequestStatus = 'pending' | 'approved' | 'dismissed'

// 席位升级详情
export type AdminRequestSeatUpgradeDetails = {
  message?: string | null
  current_seat_tier?: string | null
}

// 创建请求参数（联合类型）
export type AdminRequestCreateParams =
  | { request_type: 'limit_increase'; details: null }
  | { request_type: 'seat_upgrade'; details: AdminRequestSeatUpgradeDetails }

// 管理员请求对象
export type AdminRequest = {
  uuid: string
  status: AdminRequestStatus
  requester_uuid?: string | null
  created_at: string
} & (
  | { request_type: 'limit_increase'; details: null }
  | { request_type: 'seat_upgrade'; details: AdminRequestSeatUpgradeDetails }
)
```

### API 端点

所有端点都基于 `${BASE_API_URL}/api/oauth/organizations/${orgUUID}/admin_requests`:

| 功能 | 方法 | 端点路径 | 参数 |
|------|------|----------|------|
| 创建请求 | POST | `/admin_requests` | `AdminRequestCreateParams` |
| 获取我的请求 | GET | `/admin_requests/me` | `request_type`, `statuses[]` |
| 检查资格 | GET | `/admin_requests/eligibility` | `request_type` |

### 认证机制

```typescript
// 请求头构造
const headers = {
  ...getOAuthHeaders(accessToken),  // Authorization: Bearer <token>
  'x-organization-uuid': orgUUID,   // 组织 UUID
}
```

**依赖的工具函数**:
- `prepareApiRequest()`: 从 `src/utils/teleport/api.ts` 导入，获取 accessToken 和 orgUUID
- `getOAuthHeaders()`: 从 `src/utils/teleport/api.ts` 导入，构造 OAuth 请求头

### 调用流程

```
用户执行 /extra-usage 命令
    ↓
extra-usage-core.ts 检查用户权限
    ↓
如果是 Team/Enterprise 且无计费权限:
    1. 调用 checkAdminRequestEligibility('limit_increase')
    2. 调用 getMyAdminRequests('limit_increase', ['pending', 'dismissed'])
    3. 如有必要，调用 createAdminRequest({ request_type: 'limit_increase', details: null })
    ↓
返回结果给用户
```

---

## 关键代码路径与文件引用

### 当前文件
- `src/services/api/adminRequests.ts` - 本模块，提供管理员请求 API 封装

### 依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/utils/teleport/api.ts` | 提供 `prepareApiRequest()` 和 `getOAuthHeaders()` |
| `src/constants/oauth.ts` | 提供 `getOauthConfig()` 获取 API 基础 URL |
| `src/services/oauth/client.ts` | `prepareApiRequest` 内部调用 `getOrganizationUUID()` |
| `src/utils/auth.ts` | `getClaudeAIOAuthTokens()` 获取 OAuth token |

### 调用方文件

| 文件路径 | 调用函数 | 用途 |
|----------|----------|------|
| `src/commands/extra-usage/extra-usage-core.ts` | `checkAdminRequestEligibility`, `getMyAdminRequests`, `createAdminRequest` | 处理 /extra-usage 命令 |

### 调用方代码示例

```typescript
// src/commands/extra-usage/extra-usage-core.ts
import {
  checkAdminRequestEligibility,
  createAdminRequest,
  getMyAdminRequests,
} from '../../services/api/adminRequests.js'

// 检查资格
const eligibility = await checkAdminRequestEligibility('limit_increase')
if (eligibility?.is_allowed === false) {
  return { type: 'message', value: 'Please contact your admin...' }
}

// 检查是否已有待处理请求
const pendingOrDismissedRequests = await getMyAdminRequests(
  'limit_increase',
  ['pending', 'dismissed'],
)
if (pendingOrDismissedRequests && pendingOrDismissedRequests.length > 0) {
  return { type: 'message', value: 'You have already submitted a request...' }
}

// 创建请求
await createAdminRequest({
  request_type: 'limit_increase',
  details: null,
})
```

---

## 依赖与外部交互

### 运行时依赖

```typescript
import axios from 'axios'
import { getOauthConfig } from '../../constants/oauth.js'
import { getOAuthHeaders, prepareApiRequest } from '../../utils/teleport/api.js'
```

### 外部 API 依赖

- **Anthropic OAuth API**: `https://api.anthropic.com/api/oauth/organizations/{orgUUID}/admin_requests`
- **认证要求**: 需要有效的 Claude.ai OAuth token（`user:inference` 或 `user:profile` scope）
- **组织要求**: 用户必须属于 Team 或 Enterprise 组织

### 配置依赖

- `BASE_API_URL`: 从 `getOauthConfig()` 获取，根据环境可能是:
  - 生产: `https://api.anthropic.com`
  - Staging: `https://api-staging.anthropic.com`
  - 本地: `http://localhost:8000`

---

## 风险、边界与改进建议

### 潜在风险

1. **认证失败风险**
   - 如果 OAuth token 过期或无效，API 调用会失败
   - 需要确保调用方正确处理 401 错误并引导用户重新登录

2. **组织 UUID 缺失**
   - `prepareApiRequest()` 在无法获取 orgUUID 时会抛出错误
   - 需要确保用户已完成 OAuth 流程并选择了组织

3. **网络错误处理**
   - 当前代码没有显式的重试机制
   - 建议添加指数退避重试以处理瞬态网络故障

### 边界条件

1. **请求去重机制**
   - 后端保证同类型 pending 请求的幂等性
   - 但客户端仍需检查 `getMyAdminRequests` 以避免不必要的 API 调用

2. **权限边界**
   - 只有 Team/Enterprise 组织成员可以使用此功能
   - Pro/Max 用户直接访问设置页面，不走此流程

3. **数据验证**
   - 使用 TypeScript 类型系统保证编译时类型安全
   - 但没有运行时 schema 验证（如 Zod），依赖后端验证

### 改进建议

1. **添加重试机制**
   ```typescript
   // 建议添加类似 axiosGetWithRetry 的包装
   export async function createAdminRequestWithRetry(...) {
     return withRetry(() => createAdminRequest(...), { maxRetries: 3 })
   }
   ```

2. **添加响应缓存**
   - `checkAdminRequestEligibility` 的结果可以短暂缓存（如 5 分钟）
   - 减少重复 API 调用

3. **增强错误处理**
   - 区分网络错误、认证错误和业务逻辑错误
   - 提供更友好的用户错误消息

4. **添加运行时验证**
   - 使用 Zod 等库验证 API 响应
   - 防止后端 API 变更导致的前端错误

5. **考虑添加取消机制**
   - 如果用户快速连续触发请求，应该能够取消之前的请求
   - 使用 `AbortController` 实现

### 测试建议

- 单元测试: 模拟 axios 和依赖函数，测试各种响应场景
- 集成测试: 使用 staging 环境测试完整的请求流程
- 边界测试: 测试 token 过期、网络超时、无效组织 UUID 等场景
