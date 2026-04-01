# auth.ts 深度研究文档

## 场景与职责

`auth.ts` 是 Claude Code 中 MCP (Model Context Protocol) 服务的核心 OAuth 认证模块，负责处理 MCP 服务器的完整认证生命周期。该模块实现了以下关键职责：

1. **OAuth 2.0 / OIDC 认证流程**：支持标准的授权码模式 (Authorization Code + PKCE) 进行用户认证
2. **跨应用访问 (XAA - Cross-App Access)**：实现 SEP-990 规范，允许企业用户通过单一 IdP 登录访问多个 MCP 服务器
3. **Token 生命周期管理**：包括 token 获取、刷新、撤销和存储
4. **动态客户端注册 (DCR)**：支持无预配置客户端的 OAuth 流程
5. **Step-up 认证**：处理权限提升场景，当现有 token 权限不足时触发重新授权

该模块是 MCP HTTP/SSE 传输层与授权服务器之间的桥梁，确保安全的 API 访问。

## 功能点目的

### 1. 标准 OAuth 认证流程 (`performMCPOAuthFlow`)

**目的**：为 MCP 服务器建立初始认证会话。

**流程**：
1. 启动本地 HTTP 回调服务器监听授权码
2. 打开浏览器引导用户完成 IdP 授权
3. 接收授权码并完成 token 交换
4. 安全存储 access_token 和 refresh_token

**关键特性**：
- 支持手动回调 URL 粘贴（用于远程/无浏览器环境）
- 5 分钟超时保护
- CSRF 状态验证
- 端口冲突自动检测

### 2. XAA (Cross-App Access) 认证 (`performMCPXaaAuth`)

**目的**：实现企业级无浏览器认证，用户只需登录一次 IdP 即可访问所有 XAA 配置的 MCP 服务器。

**技术实现**：
- **RFC 8693 Token Exchange**：id_token → ID-JAG (Identity Assertion Authorization Grant)
- **RFC 7523 JWT Bearer Grant**：ID-JAG → access_token
- **RFC 9728 Protected Resource Metadata (PRM)**：自动发现授权服务器

**优势**：
- 单一 IdP 登录，多服务器复用
- 无浏览器弹窗（silent exchange）
- 企业级安全合规

### 3. Token 刷新与维护

**目的**：确保长期运行的会话不会因 token 过期而中断。

**实现机制**：
- ** proactive refresh**：token 即将过期（5分钟内）时主动刷新
- **并发控制**：使用 `_refreshInProgress` 防止重复刷新
- **跨进程锁**：通过文件锁 (`lockfile`) 防止多实例竞争
- **指数退避重试**：网络错误时自动重试（最多3次）

### 4. Token 撤销 (`revokeServerTokens`)

**目的**：用户登出或删除服务器时，安全地使服务器端 token 失效。

**实现**：
- 遵循 RFC 7009 规范
- 先撤销 refresh_token（长期凭证），再撤销 access_token
- 支持多种认证方法（client_secret_basic / client_secret_post）
- 保留 step-up 状态以便重新认证

### 5. Step-up 认证检测

**目的**：处理权限提升场景，当服务器返回 403 insufficient_scope 时触发重新授权。

**实现**：
- `wrapFetchWithStepUpDetection`：包装 fetch 检测 403 响应
- `markStepUpPending`：标记需要权限提升
- tokens() 中省略 refresh_token 强制走完整授权流程

## 具体技术实现

### 关键数据结构

```typescript
// OAuth Token 存储结构 (SecureStorageData.mcpOAuth[serverKey])
interface McpOAuthEntry {
  serverName: string
  serverUrl: string
  accessToken: string
  refreshToken?: string
  expiresAt: number
  scope?: string
  clientId?: string
  clientSecret?: string
  discoveryState?: {
    authorizationServerUrl: string
    resourceMetadataUrl?: string
  }
  stepUpScope?: string  // 缓存的权限提升 scope
}

// XAA IdP 配置存储 (SecureStorageData.mcpXaaIdp[issuerKey])
interface XaaIdpEntry {
  idToken: string
  expiresAt: number
}
```

### 核心类：ClaudeAuthProvider

实现了 `OAuthClientProvider` 接口，是 MCP SDK 与 Claude Code 存储层的桥梁：

| 方法 | 职责 |
|------|------|
| `clientInformation()` | 提供 client_id/client_secret（从存储或配置获取） |
| `tokens()` | 返回当前 token，处理 XAA 静默交换和 proactive refresh |
| `saveTokens()` | 持久化新 token 到安全存储 |
| `redirectToAuthorization()` | 处理授权 URL，打开浏览器 |
| `refreshAuthorization()` | 使用 refresh_token 获取新 token |
| `invalidateCredentials()` | 按范围清除凭证 |
| `saveDiscoveryState()` | 缓存 OAuth 发现元数据（仅存储 URL，避免 keychain 溢出） |

### Server Key 生成

使用 SHA-256 哈希确保不同配置的服务器有独立的存储槽：

```typescript
function getServerKey(serverName: string, serverConfig: McpSSEServerConfig | McpHTTPServerConfig): string {
  const configJson = jsonStringify({
    type: serverConfig.type,
    url: serverConfig.url,
    headers: serverConfig.headers || {},
  })
  const hash = createHash('sha256')
    .update(configJson)
    .digest('hex')
    .substring(0, 16)
  return `${serverName}|${hash}`
}
```

### OAuth 错误规范化

处理非标准 OAuth 错误（如 Slack 的 `invalid_refresh_token`）：

```typescript
const NONSTANDARD_INVALID_GRANT_ALIASES = new Set([
  'invalid_refresh_token',
  'expired_refresh_token',
  'token_expired',
])
```

`normalizeOAuthErrorBody` 函数将 HTTP 200 但包含错误响应体的情况转换为标准 400 响应，使 SDK 的错误处理逻辑正常工作。

### 发现流程 (Discovery)

支持多种 OAuth 发现机制：

1. **配置元数据 URL**：用户通过 `authServerMetadataUrl` 显式指定
2. **RFC 9728 PRM**：探测 `/.well-known/oauth-protected-resource`
3. **RFC 8414**：探测 `/.well-known/oauth-authorization-server`
4. **路径感知回退**：保留路径的遗留兼容探测

### XAA 静默刷新流程

```
tokens() 调用
  ↓
检查 XAA 启用且配置存在
  ↓
检查 id_token 缓存
  ↓
检查 access_token 是否存在或即将过期
  ↓
调用 xaaRefresh()
  ↓
  ├── 发现 IdP OIDC 元数据
  ├── 获取 AS 元数据（PRM → AS discovery）
  ├── RFC 8693 Token Exchange（id_token → ID-JAG）
  └── RFC 7523 JWT Bearer（ID-JAG → access_token）
  ↓
保存新 token
```

## 关键代码路径与文件引用

### 核心文件依赖关系

```
auth.ts
├── @modelcontextprotocol/sdk/client/auth.js  (SDK OAuth 实现)
├── @modelcontextprotocol/sdk/server/auth/errors.js  (OAuth 错误类)
├── @modelcontextprotocol/sdk/shared/auth.js  (类型定义)
├── ./xaa.ts  (XAA 核心实现)
├── ./xaaIdpLogin.ts  (IdP 登录管理)
├── ./oauthPort.ts  (回调端口管理)
├── ./types.ts  (配置类型定义)
├── ../../utils/secureStorage/index.ts  (安全存储)
├── ../../utils/lockfile.js  (跨进程锁)
└── ../../utils/browser.js  (浏览器打开)
```

### 关键调用路径

**初始认证**：
```
client.ts:connectToMcpServer()
  → performMCPOAuthFlow()
    → 创建 ClaudeAuthProvider
    → 启动回调服务器
    → sdkAuth() [SDK]
    → 浏览器授权
    → 接收授权码
    → sdkAuth() [带 authorizationCode]
    → saveTokens()
```

**Token 获取（每次请求）**：
```
SDK 内部
  → ClaudeAuthProvider.tokens()
    → 检查 XAA 条件 → xaaRefresh()
    → 或检查过期 → refreshAuthorization()
    → 返回当前 token
```

**Token 刷新**：
```
tokens() 检测到即将过期
  → refreshAuthorization()
    → 获取文件锁
    → 重新读取存储（检查其他进程是否已刷新）
    → _doRefresh()
      → sdkRefreshAuthorization() [SDK]
      → saveTokens()
    → 释放文件锁
```

### 配置类型定义 (types.ts)

```typescript
// OAuth 配置
interface McpOAuthConfig {
  clientId?: string           // 预配置客户端 ID
  callbackPort?: number       // 固定回调端口
  authServerMetadataUrl?: string  // 显式元数据 URL
  xaa?: boolean              // 启用 XAA
}

// SSE/HTTP 服务器配置
interface McpSSEServerConfig {
  type: 'sse'
  url: string
  headers?: Record<string, string>
  oauth?: McpOAuthConfig
}
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk` | MCP SDK，提供 OAuth 客户端实现 |
| `axios` | Token 撤销请求（需要更精细的错误处理） |
| `xss` | 回调页面 HTML 净化 |
| `crypto` | State/code_verifier 生成，server key 哈希 |
| `zod` | XAA 响应验证（通过 xaa.ts） |

### 环境变量

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_ENABLE_XAA` | 启用 XAA 功能 |
| `MCP_OAUTH_CLIENT_METADATA_URL` | 覆盖 CIMD URL |
| `MCP_OAUTH_CALLBACK_PORT` | 固定回调端口 |
| `MCP_CLIENT_SECRET` | 客户端密钥（环境变量方式） |

### 安全存储交互

通过 `getSecureStorage()` 访问平台特定的安全存储：

- **macOS**：Keychain（通过 `security` 命令）
- **其他平台**：明文文件存储（fallback）

存储键：
- `mcpOAuth[serverKey]`：服务器 token 数据
- `mcpOAuthClientConfig[serverKey]`：客户端密钥
- `mcpXaaIdp[issuerKey]`：IdP id_token 缓存
- `mcpXaaIdpConfig[issuerKey]`：IdP 客户端密钥

### 分析事件

| 事件 | 触发时机 |
|------|----------|
| `tengu_mcp_oauth_flow_start` | 开始 OAuth 流程 |
| `tengu_mcp_oauth_flow_success` | 认证成功 |
| `tengu_mcp_oauth_flow_failure` | 认证失败 |
| `tengu_mcp_oauth_refresh_success` | Token 刷新成功 |
| `tengu_mcp_oauth_refresh_failure` | Token 刷新失败 |

## 风险、边界与改进建议

### 已知风险

1. **Keychain 大小限制** (已处理)
   - macOS `security -i` 有 4096 字节 stdin 限制
   - 解决方案：只存储 discovery URL，不存储完整元数据
   - 相关代码：`saveDiscoveryState()` 中的注释说明

2. **跨进程 Token 竞争** (已处理)
   - 多实例同时刷新可能导致竞争
   - 解决方案：文件锁 (`lockfile`) + 刷新后重新读取存储
   - XAA 尚未实现跨进程锁（TODO 标记）

3. **敏感信息泄露风险** (已处理)
   - OAuth 参数（state, code, code_verifier）在日志中脱敏
   - `redactSensitiveUrlParams()` 函数处理
   - XAA token 在错误日志中脱敏

4. **Slack 非标准错误** (已处理)
   - Slack 返回 HTTP 200 但 JSON 中包含错误
   - `normalizeOAuthErrorBody()` 统一处理为 400 响应

### 边界条件

1. **Token 过期处理**
   - 5 分钟缓冲期（proactive refresh）
   - 过期后无 refresh_token → 返回 undefined → 触发重新认证

2. **Step-up 认证**
   - 403 insufficient_scope 检测
   - 省略 refresh_token 强制完整授权流程
   - RFC 6749 §6 禁止通过 refresh 提升权限

3. **XAA 回退**
   - 明确禁止：配置 XAA 后不降级到标准 OAuth
   - 需要显式清除配置才能切换

4. **并发刷新控制**
   - 进程内：`_refreshInProgress` Promise 去重
   - 跨进程：文件锁（标准 OAuth）/ 无锁（XAA，待改进）

### 改进建议

1. **XAA 跨进程锁**
   - 当前仅 `_refreshInProgress` 控制进程内并发
   - 建议添加文件锁，与标准 OAuth 保持一致
   - 代码位置：`xaaRefresh()` 方法

2. **Token 刷新退避策略**
   - 当前：固定指数退避（1s, 2s, 4s）
   - 建议：添加抖动 (jitter) 防止惊群效应

3. **元数据缓存 TTL**
   - 当前：内存缓存 `_metadata` 无 TTL
   - 建议：添加过期时间，避免长期运行的进程使用过时的端点

4. **更细粒度的错误分类**
   - 当前：网络错误统一归类为 `request_failed`
   - 建议：区分 DNS、TLS、超时等不同错误类型

5. **OAuth 配置验证**
   - 当前：配置错误在运行时暴露
   - 建议：添加配置验证模式，在添加服务器时检查

6. **刷新 Token 轮换**
   - 当前：保存 AS 返回的新 refresh_token
   - 注意：某些 AS 在每次刷新时轮换 refresh_token
   - 当前实现已正确处理，但需确保存储原子性

### 测试建议

1. **单元测试重点**：
   - `getServerKey` 哈希一致性
   - Token 过期计算逻辑
   - Step-up 检测正则表达式
   - 错误规范化逻辑

2. **集成测试场景**：
   - 多实例并发刷新
   - 网络中断恢复
   - IdP 会话过期
   - 权限提升流程

3. **安全测试**：
   - CSRF 状态验证
   - 回调 URL 参数注入
   - Token 存储加密
