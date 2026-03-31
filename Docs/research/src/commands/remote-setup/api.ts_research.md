# Research: src/commands/remote-setup/api.ts

## 场景与职责

本文件是 Claude Code CLI 中 `/web-setup` 命令的核心 API 层，负责处理与 Claude.ai Web 平台相关的后端通信。主要场景包括：

1. **GitHub Token 导入**：将本地 GitHub CLI 的认证 Token 安全地导入到 Claude.ai 后端，使得 Web 端 Claude 能够代表用户执行 GitHub 操作（克隆、推送代码）
2. **默认环境创建**：为新用户自动创建默认的云端执行环境（anthropic_cloud），实现零配置上手
3. **登录状态检查**：验证用户是否已登录 Claude.ai OAuth

该模块是连接本地 CLI 与云端 Web 体验的桥梁，使用户能够在 claude.ai/code 网页端无缝使用本地 GitHub 凭证。

## 功能点目的

### 1. RedactedGithubToken - 安全令牌封装

**目的**：防止 GitHub Token 在日志、错误消息或调试输出中意外泄露。

**实现机制**：
- 使用 ES2022 私有字段 `#value` 存储原始令牌
- 重写 `toString()`、`toJSON()`、`[Symbol.for('nodejs.util.inspect.custom')]` 方法，统一返回 `[REDACTED:gh-token]`
- 仅在 HTTP 请求体中通过 `.reveal()` 方法暴露原始值

### 2. importGithubToken - Token 导入

**目的**：将本地 GitHub Token 安全传输到 Claude.ai 后端存储。

**流程**：
1. 调用 `prepareApiRequest()` 获取 OAuth access token 和 org UUID
2. 向 `/v1/code/github/import-token` 发送 POST 请求
3. 后端验证 Token 有效性（调用 GitHub /user API）
4. 后端使用 Fernet 加密存储 Token 到 `sync_user_tokens` 表
5. 返回 GitHub 用户名确认成功

**错误处理**：
- `not_signed_in`：用户未登录 Claude.ai
- `invalid_token`：GitHub 拒绝该 Token（400 响应）
- `server`：服务器错误（非 200/400/401 状态码）
- `network`：网络连接问题

### 3. createDefaultEnvironment - 默认环境创建

**目的**：为新用户自动创建默认云端执行环境，避免首次使用时进入环境配置流程。

**设计决策**：
- **Best-effort 语义**：失败不阻塞主流程，Web 端会回退到 env-setup 引导
- **幂等性检查**：先调用 `hasExistingEnvironment()` 检查，避免重复创建
- **镜像 Web 端配置**：与 Web onboarding 的 `DEFAULT_CLOUD_ENVIRONMENT_REQUEST` 保持一致

**环境配置**：
```typescript
{
  name: 'Default',
  kind: 'anthropic_cloud',
  description: 'Default - trusted network access',
  config: {
    environment_type: 'anthropic',
    cwd: '/home/user',
    init_script: null,
    environment: {},
    languages: [
      { name: 'python', version: '3.11' },
      { name: 'node', version: '20' },
    ],
    network_config: {
      allowed_hosts: [],
      allow_default_hosts: true,
    },
  },
}
```

### 4. isSignedIn - 登录状态检查

**目的**：快速验证用户是否拥有有效的 Claude.ai OAuth 凭证。

**实现**：通过 `prepareApiRequest()` 的成功与否判断。

### 5. getCodeWebUrl - Web 端 URL 生成

**目的**：生成指向 claude.ai/code 的链接，用于引导用户打开 Web 界面。

## 具体技术实现

### 关键数据结构

```typescript
// Token 导入结果
type ImportTokenResult = {
  github_username: string
}

// 错误类型联合
type ImportTokenError =
  | { kind: 'not_signed_in' }
  | { kind: 'invalid_token' }
  | { kind: 'server'; status: number }
  | { kind: 'network' }
```

### API 端点

| 功能 | 端点 | 方法 | 特殊 Header |
|------|------|------|-------------|
| 导入 GitHub Token | `/v1/code/github/import-token` | POST | `anthropic-beta: ccr-byoc-2025-07-29` |
| 创建默认环境 | `/v1/environment_providers/cloud/create` | POST | `x-organization-uuid` |

### 认证机制

- 使用 `prepareApiRequest()` 获取 OAuth access token
- 通过 `getOAuthHeaders()` 生成标准 OAuth 请求头
- 组织 UUID 通过 `x-organization-uuid` header 传递（而非 URL 路径），避免 CLI OAuth token 被错误路由拒绝

## 关键代码路径与文件引用

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/utils/teleport/api.ts` | `prepareApiRequest()`, `getOAuthHeaders()` - OAuth 请求准备 |
| `src/utils/teleport/environments.ts` | `fetchEnvironments()` - 查询现有环境 |
| `src/constants/oauth.ts` | `getOauthConfig()` - OAuth 配置（BASE_API_URL 等） |
| `src/utils/debug.ts` | `logForDebugging()` - 调试日志 |

### 被调用方

| 文件 | 用途 |
|------|------|
| `src/commands/remote-setup/remote-setup.tsx` | UI 层调用 `importGithubToken()`, `createDefaultEnvironment()`, `isSignedIn()`, `getCodeWebUrl()` |

## 依赖与外部交互

### 外部服务

1. **Claude.ai API** (`api.anthropic.com`)
   - `/v1/code/github/import-token`：Token 导入
   - `/v1/environment_providers/cloud/create`：环境创建
   - 需要有效的 OAuth access token

2. **GitHub API**（后端调用）
   - `/user` 端点验证 Token 有效性
   - 不直接由本模块调用，由后端服务代理

### 内部依赖

```
api.ts
├── axios (HTTP 客户端)
├── ../../constants/oauth.js
├── ../../utils/debug.js
├── ../../utils/teleport/api.js
└── ../../utils/teleport/environments.js
```

## 风险、边界与改进建议

### 安全风险

1. **Token 泄露风险**
   - **缓解**：`RedactedGithubToken` 类多重防护，确保原始值仅在 HTTP body 中暴露
   - **边界**：错误日志中仍可能通过 `err.config.data` 泄露，代码中已显式避免记录该字段

2. **中间人攻击**
   - **缓解**：所有 API 调用使用 HTTPS，依赖系统 CA 证书
   - **建议**：考虑证书固定（certificate pinning）增强安全性

### 功能边界

1. **Best-effort 环境创建**
   - 环境创建失败不阻塞主流程，用户会被引导至 Web 端的 env-setup
   - 可能导致用户多一次点击，但不影响核心功能

2. **重复环境检查**
   - `hasExistingEnvironment()` 在并发场景下可能存在竞态条件
   - 后端实际负责去重，前端检查仅为减少不必要的 API 调用

3. **超时设置**
   - Token 导入：15 秒超时
   - 环境创建：15 秒超时
   - 在慢网络环境下可能超时，但会返回明确的 `network` 错误类型

### 改进建议

1. **增强错误信息**
   - 当前网络错误仅返回 "Couldn't reach the server"
   - 建议：区分 DNS 解析失败、连接超时、TLS 握手失败等具体场景

2. **重试机制**
   - 当前无自动重试逻辑
   - 建议：对 `server` 错误类型（5xx）实现指数退避重试

3. **遥测增强**
   - 当前仅记录错误级别日志
   - 建议：添加成功/失败率指标，用于监控导入流程健康度

4. **Token 验证前置**
   - 当前依赖后端验证 GitHub Token
   - 建议：在本地先调用 `gh auth status` 验证，减少无效 API 调用

5. **环境配置可定制**
   - 当前环境配置硬编码（Python 3.11, Node 20）
   - 建议：支持通过配置或参数自定义环境规格

### 测试建议

1. **单元测试**
   - Mock `prepareApiRequest()` 测试各种错误场景
   - 验证 `RedactedGithubToken` 的序列化行为

2. **集成测试**
   - 使用测试 GitHub Token 验证完整导入流程
   - 验证环境创建幂等性

3. **安全测试**
   - 验证 Token 不会出现在日志、错误堆栈、调试输出中
   - 测试网络拦截场景下的错误处理
