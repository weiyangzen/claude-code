# RemoteTriggerTool.ts 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`RemoteTriggerTool` 是 Claude Code CLI 中用于管理**远程定时触发器（Scheduled Remote Agent Triggers）**的核心工具。它允许用户通过 claude.ai 的 CCR (Claude Code Remote) API 来：
- 列出所有已配置的远程触发器
- 获取特定触发器的详细信息
- 创建新的触发器
- 更新现有触发器
- 手动运行触发器

### 1.2 业务场景
该工具主要服务于以下场景：
- **定时任务管理**：用户需要配置周期性执行的 Claude Code 任务（如定时代码审查、定时报告生成）
- **远程代理调度**：在云端调度和管理 Claude Code 代理实例
- **CI/CD 集成**：与 GitHub Actions 等 CI 系统集成，实现自动化工作流

### 1.3 安全设计
工具的核心安全特性是 **OAuth Token 进程内处理**：
- Token 在进程内部自动添加，不会暴露给 shell 或子进程
- 相比直接使用 curl，避免了 Token 在命令行历史或进程列表中泄露的风险

---

## 2. 功能点目的

### 2.1 核心功能

| 功能 | HTTP 方法 | API 端点 | 用途 |
|------|-----------|----------|------|
| list | GET | `/v1/code/triggers` | 列出所有触发器 |
| get | GET | `/v1/code/triggers/{trigger_id}` | 获取单个触发器详情 |
| create | POST | `/v1/code/triggers` | 创建新触发器 |
| update | POST | `/v1/code/triggers/{trigger_id}` | 部分更新触发器 |
| run | POST | `/v1/code/triggers/{trigger_id}/run` | 手动执行触发器 |

### 2.2 工具元数据

```typescript
const TRIGGERS_BETA = 'ccr-triggers-2026-01-30'  // Beta 版本标识
```

- **工具名称**: `RemoteTrigger`
- **搜索提示**: `manage scheduled remote agent triggers`
- **最大结果大小**: 100,000 字符
- **延迟加载**: 是 (`shouldDefer: true`)

---

## 3. 具体技术实现

### 3.1 输入输出 Schema

#### 输入 Schema (Zod)
```typescript
const inputSchema = z.strictObject({
  action: z.enum(['list', 'get', 'create', 'update', 'run']),
  trigger_id: z.string().regex(/^[\w-]+$/).optional(),
  body: z.record(z.string(), z.unknown()).optional(),
})
```

**验证规则**：
- `trigger_id` 必须符合 `^[\w-]+$` 正则（字母、数字、下划线、连字符）
- `body` 为 JSON 对象，用于 create/update 操作

#### 输出 Schema
```typescript
const outputSchema = z.object({
  status: z.number(),  // HTTP 状态码
  json: z.string(),    // API 返回的 JSON 字符串
})
```

### 3.2 核心调用流程

```
call(input, context)
  ├── 1. 检查并刷新 OAuth Token
  │     └── checkAndRefreshOAuthTokenIfNeeded()
  ├── 2. 获取 Access Token
  │     └── getClaudeAIOAuthTokens()?.accessToken
  ├── 3. 获取组织 UUID
  │     └── getOrganizationUUID()
  ├── 4. 构建请求参数
  │     ├── method: GET | POST
  │     ├── url: BASE_API_URL/v1/code/triggers[/{trigger_id}][/run]
  │     ├── headers: Auth + Content-Type + Beta + OrgUUID
  │     └── data: body (for create/update/run)
  ├── 5. 执行 HTTP 请求
  │     └── axios.request({ timeout: 20000, signal })
  └── 6. 返回结果
        └── { status, json: jsonStringify(res.data) }
```

### 3.3 HTTP 请求构造

```typescript
const base = `${getOauthConfig().BASE_API_URL}/v1/code/triggers`
const headers = {
  Authorization: `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': TRIGGERS_BETA,
  'x-organization-uuid': orgUUID,
}
```

### 3.4 工具特性实现

#### 启用检查
```typescript
isEnabled() {
  return (
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali', false) &&
    isPolicyAllowed('allow_remote_sessions')
  )
}
```

- 依赖 GrowthBook Feature Flag: `tengu_surreal_dali`
- 依赖策略限制: `allow_remote_sessions`

#### 并发安全
```typescript
isConcurrencySafe() {
  return true  // 支持并发调用
}
```

#### 只读判断
```typescript
isReadOnly(input: Input) {
  return input.action === 'list' || input.action === 'get'
}
```

#### 自动分类器输入
```typescript
toAutoClassifierInput(input: Input) {
  return `RemoteTrigger ${input.action}${input.trigger_id ? ` ${input.trigger_id}` : ''}`
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件结构
```
src/tools/RemoteTriggerTool/
├── RemoteTriggerTool.ts    # 主工具实现 (161 行)
├── UI.tsx                  # UI 渲染组件
├── prompt.ts               # 提示词和常量定义
```

### 4.2 核心代码路径

| 功能 | 代码位置 | 说明 |
|------|----------|------|
| 工具构建 | `RemoteTriggerTool.ts:46-161` | `buildTool()` 调用 |
| Schema 定义 | `RemoteTriggerTool.ts:18-42` | lazySchema 包装 |
| HTTP 请求 | `RemoteTriggerTool.ts:135-143` | axios.request |
| Action 路由 | `RemoteTriggerTool.ts:104-133` | switch-case 路由 |
| 结果映射 | `RemoteTriggerTool.ts:152-158` | mapToolResultToToolResultBlockParam |

### 4.3 关键依赖文件

```typescript
// 核心依赖
import { buildTool, type ToolDef } from '../../Tool.js'
import type { ToolUseContext } from '../../Tool.js'

// OAuth 和认证
import { getOauthConfig } from '../../constants/oauth.js'
import { checkAndRefreshOAuthTokenIfNeeded, getClaudeAIOAuthTokens } from '../../utils/auth.js'
import { getOrganizationUUID } from '../../services/oauth/client.js'

// 权限和特性
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../../services/analytics/growthbook.js'
import { isPolicyAllowed } from '../../services/policyLimits/index.js'

// 工具函数
import { lazySchema } from '../../utils/lazySchema.js'
import { jsonStringify } from '../../utils/slowOperations.js'

// UI 和提示词
import { DESCRIPTION, PROMPT, REMOTE_TRIGGER_TOOL_NAME } from './prompt.js'
import { renderToolResultMessage, renderToolUseMessage } from './UI.js'
```

---

## 5. 依赖与外部交互

### 5.1 外部 API 依赖

#### CCR API 端点
- **基础 URL**: `https://api.anthropic.com/v1/code/triggers`
- **认证方式**: OAuth 2.0 Bearer Token
- **Beta Header**: `ccr-triggers-2026-01-30`

#### API 请求头
```
Authorization: Bearer {access_token}
Content-Type: application/json
anthropic-version: 2023-06-01
anthropic-beta: ccr-triggers-2026-01-30
x-organization-uuid: {org_uuid}
```

### 5.2 内部服务依赖

| 服务 | 文件路径 | 用途 |
|------|----------|------|
| OAuth 配置 | `src/constants/oauth.ts` | API URL、客户端配置 |
| 认证工具 | `src/utils/auth.ts` | Token 获取与刷新 |
| OAuth 客户端 | `src/services/oauth/client.ts` | 组织 UUID 获取 |
| GrowthBook | `src/services/analytics/growthbook.ts` | Feature Flag 检查 |
| 策略限制 | `src/services/policyLimits/index.ts` | 权限策略验证 |

### 5.3 依赖关系图

```
RemoteTriggerTool
├── 认证层
│   ├── getClaudeAIOAuthTokens() ──→ SecureStorage
│   ├── checkAndRefreshOAuthTokenIfNeeded() ──→ Token 刷新流程
│   └── getOrganizationUUID() ──→ OAuth Profile API
├── 权限层
│   ├── getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali')
│   └── isPolicyAllowed('allow_remote_sessions')
└── 执行层
    └── axios.request ──→ Anthropic API
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险 1: Token 过期
- **场景**: OAuth Token 在调用期间过期
- **缓解**: `checkAndRefreshOAuthTokenIfNeeded()` 在每次调用前检查并刷新 Token
- **残余风险**: 如果刷新失败，调用将抛出认证错误

#### 风险 2: 网络超时
- **配置**: 20 秒超时 (`timeout: 20_000`)
- **影响**: 在慢网络环境下可能导致操作失败
- **建议**: 考虑根据操作类型设置不同的超时（如 list 可以更长）

#### 风险 3: Feature Flag 依赖
- **场景**: 工具启用依赖 `tengu_surreal_dali` flag
- **影响**: Flag 变更可能导致工具突然不可用
- **建议**: 添加工具不可用时的友好提示

### 6.2 边界情况

| 场景 | 行为 | 建议 |
|------|------|------|
| 未登录用户 | 抛出错误: "Not authenticated with a claude.ai account" | 错误消息中包含 `/login` 提示 |
| 无组织 UUID | 抛出错误: "Unable to resolve organization UUID" | 需要检查 OAuth scope 是否包含 profile |
| 缺少 trigger_id (get/update/run) | 抛出参数验证错误 | 在 Zod schema 中已标记为 required |
| 缺少 body (create/update) | 抛出参数验证错误 | 同上 |
| API 返回非 2xx | 返回实际状态码和响应体 | `validateStatus: () => true` 允许所有状态码 |

### 6.3 改进建议

#### 建议 1: 添加重试机制
```typescript
// 当前: 单次请求
const res = await axios.request({...})

// 建议: 添加指数退避重试
const res = await withRetry(() => axios.request({...}), {
  maxRetries: 3,
  retryableStatuses: [429, 500, 502, 503, 504]
})
```

#### 建议 2: 响应缓存
对于 `list` 和 `get` 操作，可以考虑添加短时间缓存（如 5 秒），避免频繁调用 API。

#### 建议 3: 更好的错误分类
当前所有 API 错误都直接返回，建议根据状态码提供更具体的错误信息：
```typescript
if (res.status === 403) {
  throw new Error('Permission denied. Check your organization policy.')
}
if (res.status === 404) {
  throw new Error(`Trigger ${trigger_id} not found.`)
}
```

#### 建议 4: 分页支持
如果触发器列表可能很长，建议添加分页参数支持：
```typescript
inputSchema: {
  // ...
  limit: z.number().optional(),
  offset: z.number().optional(),
}
```

#### 建议 5: 响应格式化
当前返回原始 JSON 字符串，可以考虑根据 action 类型返回结构化的 Markdown 表格，提升可读性。

### 6.4 测试建议

1. **单元测试**: 模拟 axios 测试各种 action 的路由逻辑
2. **集成测试**: 使用测试 OAuth Token 验证与 staging API 的集成
3. **错误场景**: 测试 Token 过期、网络超时、API 错误响应等场景
4. **并发测试**: 验证 `isConcurrencySafe: true` 的正确性

---

## 7. 附录

### 7.1 API 响应示例

```json
// list 响应
{
  "triggers": [
    {
      "id": "trigger-123",
      "name": "Daily Report",
      "schedule": "0 9 * * *",
      "status": "active"
    }
  ]
}

// run 响应
{
  "execution_id": "exec-456",
  "status": "started"
}
```

### 7.2 相关文档

- [Claude Code Remote API 文档](https://docs.anthropic.com/claude-code/api)
- [OAuth 认证流程](../services/oauth/client.ts)
- [GrowthBook Feature Flag 系统](../services/analytics/growthbook.ts)
