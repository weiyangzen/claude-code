# oauth.ts 深度研究文档

## 场景与职责

`oauth.ts` 是 Claude Code CLI 中管理 OAuth 2.0 认证流程的核心配置模块。它定义了所有 OAuth 相关的端点、客户端 ID、作用域以及多环境（生产、预发、本地开发）配置，支持 Claude.ai 和 Console 两种认证路径。

### 核心使用场景
1. **用户登录**：引导用户完成 OAuth 授权流程
2. **Token 交换**：授权码换取访问令牌
3. **API 密钥创建**：通过 Console OAuth 创建 API 密钥
4. **MCP 代理连接**：使用 OAuth 令牌连接 MCP 代理服务
5. **多环境支持**：支持生产、预发、本地开发环境切换
6. **FedStart 支持**：支持政府/企业私有部署的自定义 OAuth 端点

---

## 功能点目的

### 1. OAuth 作用域定义

| 常量 | 值 | 用途 |
|------|-----|------|
| `CLAUDE_AI_INFERENCE_SCOPE` | `user:inference` | Claude.ai 推理访问 |
| `CLAUDE_AI_PROFILE_SCOPE` | `user:profile` | 用户资料读取 |
| `CONSOLE_SCOPE` | `org:create_api_key` | Console API 密钥创建 |
| `OAUTH_BETA_HEADER` | `oauth-2025-04-20` | OAuth Beta 功能标识 |

**作用域集合**：
- `CONSOLE_OAUTH_SCOPES`: Console 认证所需作用域
- `CLAUDE_AI_OAUTH_SCOPES`: Claude.ai 订阅者（Pro/Max/Team/Enterprise）所需作用域
- `ALL_OAUTH_SCOPES`: 所有作用域的并集，登录时请求以支持 Console → Claude.ai 重定向

### 2. 多环境配置

#### 生产环境 (`PROD_OAUTH_CONFIG`)

| 配置项 | 值 |
|--------|-----|
| `BASE_API_URL` | `https://api.anthropic.com` |
| `CONSOLE_AUTHORIZE_URL` | `https://platform.claude.com/oauth/authorize` |
| `CLAUDE_AI_AUTHORIZE_URL` | `https://claude.com/cai/oauth/authorize` |
| `CLAUDE_AI_ORIGIN` | `https://claude.ai` |
| `TOKEN_URL` | `https://platform.claude.com/v1/oauth/token` |
| `API_KEY_URL` | `https://api.anthropic.com/api/oauth/claude_cli/create_api_key` |
| `ROLES_URL` | `https://api.anthropic.com/api/oauth/claude_cli/roles` |
| `CONSOLE_SUCCESS_URL` | `https://platform.claude.com/buy_credits?...` |
| `CLAUDEAI_SUCCESS_URL` | `https://platform.claude.com/oauth/code/success?...` |
| `MANUAL_REDIRECT_URL` | `https://platform.claude.com/oauth/code/callback` |
| `CLIENT_ID` | `9d1c250a-e61b-44d9-88ed-5944d1962f5e` |
| `MCP_PROXY_URL` | `https://mcp-proxy.anthropic.com` |
| `MCP_PROXY_PATH` | `/v1/mcp/{server_id}` |

**Claude.ai 授权 URL 说明**：
- 实际通过 `claude.com/cai/*` 路由，用于归因追踪
- 307 重定向到 `claude.ai/oauth/authorize`
- `CLAUDE_AI_ORIGIN` 单独设置为 `https://claude.ai`，确保链接到 `/code`、`/settings/connectors` 等页面正确

#### 预发环境 (`STAGING_OAUTH_CONFIG`)

**仅内部 ant 构建包含**，通过 `process.env.USER_TYPE === 'ant'` 条件编译保护。

| 配置项 | 值 |
|--------|-----|
| `BASE_API_URL` | `https://api-staging.anthropic.com` |
| `CONSOLE_AUTHORIZE_URL` | `https://platform.staging.ant.dev/oauth/authorize` |
| `CLAUDE_AI_AUTHORIZE_URL` | `https://claude-ai.staging.ant.dev/oauth/authorize` |
| `CLIENT_ID` | `22422756-60c9-4084-8eb7-27705fd5cf9a` |

#### 本地开发环境 (`getLocalOauthConfig`)

**本地开发服务器配置**：
- API 代理：`:8000`（`api dev start -g ccr`）
- Claude.ai 前端：`:4000`
- Console 前端：`:3000`

**环境变量覆盖**：
- `CLAUDE_LOCAL_OAUTH_API_BASE`
- `CLAUDE_LOCAL_OAUTH_APPS_BASE`
- `CLAUDE_LOCAL_OAUTH_CONSOLE_BASE`

### 3. MCP OAuth 客户端元数据

```typescript
export const MCP_CLIENT_METADATA_URL =
  'https://claude.ai/oauth/claude-code-client-metadata'
```

**用途**：实现 CIMD (Client ID Metadata Document / SEP-991) 规范，允许 MCP 认证服务器使用托管的客户端元数据而非动态客户端注册。

### 4. FedStart 自定义端点支持

**允许的自定义 OAuth 基础 URL**：
- `https://beacon.claude-ai.staging.ant.dev`
- `https://claude.fedstart.com`
- `https://claude-staging.fedstart.com`

**安全机制**：
- 白名单机制，仅允许特定端点
- 防止 OAuth 令牌被发送到任意端点
- 通过 `CLAUDE_CODE_CUSTOM_OAUTH_URL` 环境变量启用

### 5. 配置选择逻辑

```
getOauthConfig()
    ↓
检查 CLAUDE_CODE_CUSTOM_OAUTH_URL
    ↓
[设置] 验证白名单 → 应用自定义配置
[未设置] 检查配置类型
    ↓
getOauthConfigType()
    ↓
检查 USE_LOCAL_OAUTH → local
检查 USE_STAGING_OAUTH → staging
默认 → prod
```

---

## 具体技术实现

### 数据结构

```typescript
// OAuth 配置类型
type OauthConfig = {
  BASE_API_URL: string
  CONSOLE_AUTHORIZE_URL: string
  CLAUDE_AI_AUTHORIZE_URL: string
  CLAUDE_AI_ORIGIN: string
  TOKEN_URL: string
  API_KEY_URL: string
  ROLES_URL: string
  CONSOLE_SUCCESS_URL: string
  CLAUDEAI_SUCCESS_URL: string
  MANUAL_REDIRECT_URL: string
  CLIENT_ID: string
  OAUTH_FILE_SUFFIX: string
  MCP_PROXY_URL: string
  MCP_PROXY_PATH: string
}

// 作用域常量
export const CLAUDE_AI_INFERENCE_SCOPE = 'user:inference' as const
export const CLAUDE_AI_PROFILE_SCOPE = 'user:profile' as const
export const OAUTH_BETA_HEADER = 'oauth-2025-04-20' as const

// 作用域集合
export const CONSOLE_OAUTH_SCOPES: readonly string[]
export const CLAUDE_AI_OAUTH_SCOPES: readonly string[]
export const ALL_OAUTH_SCOPES: readonly string[]

// 配置获取函数
export function fileSuffixForOauthConfig(): string
export function getOauthConfig(): OauthConfig
```

### 关键代码路径

#### 1. OAuth 登录流程

```
用户执行登录命令
    ↓
src/services/oauth/auth-code-listener.ts
    ↓
调用 getOauthConfig() 获取配置
    ↓
构造授权 URL（CLAUDE_AI_AUTHORIZE_URL 或 CONSOLE_AUTHORIZE_URL）
    ↓
打开浏览器，用户授权
    ↓
回调到 MANUAL_REDIRECT_URL
    ↓
交换授权码获取 Token（TOKEN_URL）
```

**关键文件引用**：
- `src/services/oauth/auth-code-listener.ts`: 授权码监听
- `src/services/oauth/client.ts`: OAuth 客户端
- `src/services/oauth/getOauthprofile.ts`: 获取用户资料

#### 2. API 密钥创建流程

```
用户请求创建 API 密钥
    ↓
src/services/api/adminRequests.ts
    ↓
使用 API_KEY_URL 创建密钥
    ↓
返回密钥信息
```

**关键文件引用**：
- `src/services/api/adminRequests.ts`: API 密钥管理

#### 3. MCP 代理连接

```
连接 MCP 服务器
    ↓
src/services/mcp/claudeai.ts
    ↓
使用 MCP_PROXY_URL 和 MCP_PROXY_PATH
    ↓
建立代理连接
```

**关键文件引用**：
- `src/services/mcp/claudeai.ts`: MCP Claude.ai 集成
- `src/services/mcp/auth.ts`: MCP 认证
- `src/services/mcp/client.ts`: MCP 客户端

#### 4. 会话创建

```
创建远程会话
    ↓
src/bridge/createSession.ts
    ↓
使用 getOauthConfig() 获取角色信息（ROLES_URL）
    ↓
创建会话
```

**关键文件引用**：
- `src/bridge/createSession.ts`: 会话创建

#### 5. 预检检查

```
应用启动预检
    ↓
src/utils/preflightChecks.tsx
    ↓
验证 OAuth 配置可访问性
    ↓
报告连接状态
```

**关键文件引用**：
- `src/utils/preflightChecks.tsx`: 预检检查

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `src/utils/envUtils.js` 的 `isEnvTruthy` | 解析环境变量布尔值 |

### 被依赖方

| 文件 | 使用的常量/函数 | 用途 |
|------|----------------|------|
| `src/services/oauth/auth-code-listener.ts` | `getOauthConfig` | 授权流程 |
| `src/services/oauth/client.ts` | `getOauthConfig`, `ALL_OAUTH_SCOPES` | OAuth 客户端 |
| `src/services/oauth/getOauthProfile.ts` | `getOauthConfig` | 用户资料 |
| `src/services/api/adminRequests.ts` | `getOauthConfig` | API 密钥管理 |
| `src/services/mcp/claudeai.ts` | `getOauthConfig`, `MCP_CLIENT_METADATA_URL` | MCP 代理 |
| `src/services/mcp/auth.ts` | `getOauthConfig` | MCP 认证 |
| `src/services/mcp/client.ts` | `getOauthConfig` | MCP 客户端 |
| `src/bridge/createSession.ts` | `getOauthConfig` | 会话创建 |
| `src/bridge/bridgeConfig.ts` | `getOauthConfig` | 桥接配置 |
| `src/bridge/trustedDevice.ts` | `getOauthConfig` | 受信任设备 |
| `src/utils/preflightChecks.tsx` | `getOauthConfig` | 预检检查 |
| `src/utils/auth.ts` | `getOauthConfig` | 认证工具 |
| `src/utils/fastMode.ts` | `getOauthConfig` | 快速模式 |
| `src/utils/http.ts` | `getOauthConfig` | HTTP 工具 |
| `src/utils/background/remote/preconditions.ts` | `getOauthConfig` | 远程前提条件 |
| `src/utils/betas.ts` | `getOauthConfig` | Beta 功能 |
| `src/utils/teleport/*.ts` | `getOauthConfig` | Teleport 功能 |
| `src/utils/apiPreconnect.ts` | `getOauthConfig` | API 预连接 |
| `src/utils/model/modelCapabilities.ts` | `getOauthConfig` | 模型能力 |
| `src/main.tsx` | `getOauthConfig` | 应用入口 |
| `src/remote/SessionsWebSocket.ts` | `getOauthConfig` | WebSocket 会话 |
| `src/assistant/sessionHistory.ts` | `getOauthConfig` | 会话历史 |
| `src/services/settingsSync/index.ts` | `getOauthConfig` | 设置同步 |
| `src/services/remoteManagedSettings/*.ts` | `getOauthConfig` | 远程管理设置 |
| `src/services/voiceStreamSTT.ts` | `getOauthConfig` | 语音流 STT |
| `src/services/policyLimits/index.ts` | `getOauthConfig` | 策略限制 |
| `src/services/teamMemorySync/index.ts` | `getOauthConfig` | 团队内存同步 |
| `src/services/api/*.ts` | `getOauthConfig` | 各类 API 服务 |
| `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts` | `getOauthConfig` | 远程触发工具 |
| `src/tools/BriefTool/upload.ts` | `getOauthConfig` | Brief 上传 |
| `src/commands/remote-setup/api.ts` | `getOauthConfig` | 远程设置 API |
| `src/upstreamproxy/upstreamproxy.ts` | `getOauthConfig` | 上游代理 |
| `src/components/mcp/MCPRemoteServerMenu.tsx` | `getOauthConfig` | MCP 菜单 |

### 外部服务交互

| 服务 | 交互方式 | 说明 |
|------|---------|------|
| **Anthropic API** | OAuth 2.0 | 用户认证和授权 |
| **Claude.ai** | OAuth 2.0 | 用户登录和资料 |
| **Console** | OAuth 2.0 | API 密钥管理 |
| **MCP Proxy** | OAuth Token | MCP 服务器代理 |

### 环境变量

| 变量 | 类型 | 用途 |
|------|------|------|
| `USER_TYPE` | 构建时 | 区分内部/外部用户 |
| `USE_LOCAL_OAUTH` | 运行时 | 启用本地 OAuth |
| `USE_STAGING_OAUTH` | 运行时 | 启用预发 OAuth |
| `CLAUDE_LOCAL_OAUTH_API_BASE` | 运行时 | 本地 API 基础 URL |
| `CLAUDE_LOCAL_OAUTH_APPS_BASE` | 运行时 | 本地 Apps 基础 URL |
| `CLAUDE_LOCAL_OAUTH_CONSOLE_BASE` | 运行时 | 本地 Console 基础 URL |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | 运行时 | 自定义 OAuth URL（FedStart） |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | 运行时 | 覆盖客户端 ID |

---

## 风险、边界与改进建议

### 当前风险

1. **硬编码端点**
   - URL 硬编码在源代码中
   - 端点变更需要代码更新和重新发布

2. **客户端 ID 暴露**
   - 客户端 ID 是公开信息，但大量硬编码增加维护复杂度

3. **环境切换复杂性**
   - 多个环境变量控制环境切换
   - 可能产生意外的组合（如同时设置 LOCAL 和 STAGING）

4. **FedStart 白名单限制**
   - 白名单需要代码更新才能添加新端点
   - 阻碍客户自助部署

5. **死代码消除依赖**
   - 预发配置依赖 `process.env.USER_TYPE === 'ant'` 条件
   - 如果构建系统变更，可能导致配置泄露

### 边界情况

| 场景 | 行为 |
|------|------|
| 同时设置 LOCAL 和 STAGING | LOCAL 优先（按代码顺序） |
| 自定义 URL 不在白名单 | 抛出错误 |
| 客户端 ID 覆盖 | 应用于最终配置 |
| 环境变量 URL 格式错误 | 可能产生无效 URL |
| 网络不可达 | 运行时错误，非配置错误 |

### 改进建议

1. **配置外部化**
   ```typescript
   // 建议从配置文件或远程获取
   export async function getOauthConfig(): Promise<OauthConfig> {
     // 优先从远程配置服务获取
     if (process.env.CLAUDE_CODE_CONFIG_URL) {
       return fetchRemoteConfig()
     }
     // 回退到本地配置
     return getLocalOauthConfig()
   }
   ```

2. **配置验证**
   ```typescript
   // 建议添加配置验证
   export function validateOauthConfig(config: OauthConfig): void {
     const required = ['BASE_API_URL', 'TOKEN_URL', 'CLIENT_ID']
     for (const key of required) {
       if (!config[key]) {
         throw new Error(`Missing required OAuth config: ${key}`)
       }
     }
     // URL 格式验证
     new URL(config.BASE_API_URL)
   }
   ```

3. **动态 FedStart 支持**
   ```typescript
   // 建议支持动态添加白名单（需安全审核）
   export function addAllowedOauthBaseUrl(url: string, signature: string): void {
     // 验证签名（由 Anthropic 签发）
     if (!verifyAnthropicSignature(url, signature)) {
       throw new Error('Invalid signature')
     }
     ALLOWED_OAUTH_BASE_URLS.push(url)
   }
   ```

4. **配置缓存**
   ```typescript
   // 建议缓存配置避免重复计算
   let cachedConfig: OauthConfig | undefined
   
   export function getOauthConfig(): OauthConfig {
     if (!cachedConfig) {
       cachedConfig = computeOauthConfig()
     }
     return cachedConfig
   }
   
   export function clearOauthConfigCache(): void {
     cachedConfig = undefined
   }
   ```

5. **增强日志**
   ```typescript
   // 建议记录配置来源
   export function getOauthConfig(): OauthConfig {
     const config = computeOauthConfig()
     logForDebugging(`OAuth config source: ${config.source}`)
     return config
   }
   ```

6. **自动化测试**
   - 各环境配置验证
   - 白名单机制测试
   - 环境变量优先级测试

### 与认证系统的关系

```
oauth.ts (配置定义)
    ↓ 被使用
services/oauth/*.ts (OAuth 实现)
    ↓ 调用
GitHub/浏览器 (用户授权)
    ↓ 回调
CLI (Token 获取)
    ↓ 使用
Anthropic API (认证请求)
```

OAuth 配置是整个认证流程的基础，任何配置错误都会导致用户无法登录或使用服务。
