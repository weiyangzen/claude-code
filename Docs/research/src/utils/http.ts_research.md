# http.ts 研究文档

## 场景与职责

`http.ts` 是 Claude Code CLI 的**HTTP 工具层**，负责处理所有与 Anthropic API 通信相关的 HTTP 基础功能。该模块处于网络请求链的底层，为上层 API 客户端提供统一的认证头生成、User-Agent 构建和 OAuth 错误重试机制。

### 核心使用场景

1. **API 请求认证**：为所有 Anthropic API 请求生成正确的认证头（OAuth 或 API Key）
2. **User-Agent 标识**：构建包含版本、用户类型、入口点等信息的 User-Agent 字符串，用于日志过滤和分析
3. **OAuth 401 错误恢复**：处理时钟漂移导致的 OAuth token 过期问题，自动刷新并重试
4. **MCP 服务器通信**：为 Model Context Protocol 服务器提供专门的 User-Agent

---

## 功能点目的

### 1. User-Agent 生成 (`getUserAgent`)

**设计目标**：提供详细的客户端标识信息，支持日志过滤、QoS 路由和使用分析。

**包含信息**：
- 基础标识：`claude-cli/{VERSION}`
- 用户类型：`pro`, `free`, `ant` 等
- 入口点：`cli`, `claude-desktop`, `agent-sdk` 等
- SDK 版本（如果通过 Agent SDK 调用）
- 客户端应用标识（`CLADE_AGENT_SDK_CLIENT_APP`）
- 工作负载标签（用于 cron 发起的请求）

**重要警告**：User-Agent 中的 `claude-cli` 字符串被日志系统硬编码依赖，修改需同步更新日志过滤逻辑。

### 2. MCP User-Agent (`getMCPUserAgent`)

为 MCP（Model Context Protocol）服务器通信提供简化的 User-Agent：
- 格式：`claude-code/{VERSION} ({entrypoint}, agent-sdk/{version}, client-app/{app})`
- 用于 MCP 服务器识别客户端身份

### 3. WebFetch User-Agent (`getWebFetchUserAgent`)

用于访问外部网站的 User-Agent：
- 格式：`Claude-User (claude-code/{VERSION}; +https://support.anthropic.com/)`
- 遵循 Anthropic 公开的爬虫标识规范
- 允许网站运营者在 `robots.txt` 中识别和配置

### 4. 认证头生成 (`getAuthHeaders`)

**双模式认证支持**：

| 用户类型 | 认证方式 | 请求头 |
|---------|---------|--------|
| Claude AI 订阅者 (Max/Pro) | OAuth | `Authorization: Bearer {token}` + `anthropic-beta: oauth-2025...` |
| 普通用户 | API Key | `x-api-key: {apiKey}` |

**错误处理**：当认证信息缺失时返回错误对象而非抛出异常，允许调用方优雅处理。

### 5. OAuth 401 重试 (`withOAuth401Retry`)

**解决的问题**：
- 本地时钟漂移导致 token 在本地未过期但在服务器端已过期
- OAuth token 被撤销的场景（某些端点返回 403 而非 401）

**重试策略**：
- 首次请求失败（401 或特定 403）→ 强制刷新 token → 重试一次
- 请求闭包在重试时重新执行，确保获取最新 token

---

## 具体技术实现

### User-Agent 构建逻辑

```typescript
export function getUserAgent(): string {
  const agentSdkVersion = process.env.CLAUDE_AGENT_SDK_VERSION
    ? `, agent-sdk/${process.env.CLAUDE_AGENT_SDK_VERSION}`
    : ''
  const clientApp = process.env.CLAUDE_AGENT_SDK_CLIENT_APP
    ? `, client-app/${process.env.CLAUDE_AGENT_SDK_CLIENT_APP}`
    : ''
  const workload = getWorkload()
  const workloadSuffix = workload ? `, workload/${workload}` : ''
  
  return `claude-cli/${MACRO.VERSION} (${process.env.USER_TYPE}, ${process.env.CLAUDE_CODE_ENTRYPOINT ?? 'cli'}${agentSdkVersion}${clientApp}${workloadSuffix})`
}
```

### OAuth 401 重试实现

```typescript
export async function withOAuth401Retry<T>(
  request: () => Promise<T>,
  opts?: { also403Revoked?: boolean },
): Promise<T> {
  try {
    return await request()
  } catch (err) {
    if (!axios.isAxiosError(err)) throw err
    
    const status = err.response?.status
    const isAuthError =
      status === 401 ||
      (opts?.also403Revoked &&
        status === 403 &&
        typeof err.response?.data === 'string' &&
        err.response.data.includes('OAuth token has been revoked'))
    
    if (!isAuthError) throw err
    
    const failedAccessToken = getClaudeAIOAuthTokens()?.accessToken
    if (!failedAccessToken) throw err
    
    await handleOAuth401Error(failedAccessToken)
    return await request()  // 重试，闭包应重新读取 auth headers
  }
}
```

### 认证头生成逻辑

```typescript
export function getAuthHeaders(): AuthHeaders {
  // OAuth 路径（Claude AI 订阅者）
  if (isClaudeAISubscriber()) {
    const oauthTokens = getClaudeAIOAuthTokens()
    if (!oauthTokens?.accessToken) {
      return { headers: {}, error: 'No OAuth token available' }
    }
    return {
      headers: {
        Authorization: `Bearer ${oauthTokens.accessToken}`,
        'anthropic-beta': OAUTH_BETA_HEADER,
      },
    }
  }
  
  // API Key 路径（普通用户）
  const apiKey = getAnthropicApiKey()
  if (!apiKey) {
    return { headers: {}, error: 'No API key available' }
  }
  return {
    headers: { 'x-api-key': apiKey },
  }
}
```

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/http.ts`
- **导出函数**：
  - `getUserAgent()` - 主 User-Agent 生成
  - `getMCPUserAgent()` - MCP 专用 User-Agent
  - `getWebFetchUserAgent()` - Web 抓取 User-Agent
  - `getAuthHeaders()` - 认证头生成
  - `withOAuth401Retry()` - OAuth 401 重试包装器
- **导出类型**：`AuthHeaders`

### 调用方分布

| 文件路径 | 使用函数 | 使用场景 |
|---------|---------|---------|
| `src/services/api/client.ts` | `getUserAgent` | Anthropic API 客户端初始化 |
| `src/services/voiceStreamSTT.ts` | `getUserAgent` | 语音流语音识别 API 请求 |
| `src/services/analytics/growthbook.ts` | `getUserAgent` | GrowthBook 特性开关请求 |
| `src/services/analytics/firstPartyEventLoggingExporter.ts` | `getUserAgent` | 第一方事件日志导出 |
| `src/services/api/bootstrap.ts` | `getUserAgent` | 启动时 API 请求 |
| `src/services/api/metricsOptOut.ts` | `getUserAgent` | 指标退出请求 |
| `src/services/api/grove.ts` | `getUserAgent` | Grove 服务请求 |
| `src/services/api/firstTokenDate.ts` | `getUserAgent` | 首 token 日期查询 |
| `src/services/api/usage.ts` | `getUserAgent` | 使用量查询 |
| `src/tools/WebFetchTool/utils.ts` | `getWebFetchUserAgent` | 网页抓取请求 |
| `src/services/mcp/client.ts` | `getMCPUserAgent` | MCP 服务器通信 |
| `src/utils/preflightChecks.tsx` | `getUserAgent` | 预检检查请求 |
| `src/components/Feedback.tsx` | `getUserAgent` | 反馈提交 |
| `src/components/FeedbackSurvey/submitTranscriptShare.ts` | `getUserAgent` | 转录分享提交 |

### 依赖导入

```typescript
import axios from 'axios'                                    // HTTP 客户端
import { OAUTH_BETA_HEADER } from '../constants/oauth.js'    // OAuth beta 头常量
import {
  getAnthropicApiKey,
  getClaudeAIOAuthTokens,
  handleOAuth401Error,
  isClaudeAISubscriber,
} from './auth.js'                                          // 认证工具
import { getClaudeCodeUserAgent } from './userAgent.js'      // 基础 User-Agent
import { getWorkload } from './workloadContext.js'           // 工作负载上下文
```

---

## 依赖与外部交互

### 外部依赖

| 包名 | 用途 |
|------|------|
| `axios` | HTTP 错误类型检测 (`axios.isAxiosError`) |

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `constants/oauth.js` | `OAUTH_BETA_HEADER` | OAuth 请求的 beta 头 |
| `utils/auth.js` | `getAnthropicApiKey`, `getClaudeAIOAuthTokens`, `handleOAuth401Error`, `isClaudeAISubscriber` | 认证状态检查和 token 刷新 |
| `utils/userAgent.js` | `getClaudeCodeUserAgent` | 基础 User-Agent 构建 |
| `utils/workloadContext.js` | `getWorkload` | 获取工作负载标签 |

### 环境变量依赖

| 环境变量 | 用途 |
|---------|------|
| `CLAUDE_AGENT_SDK_VERSION` | Agent SDK 版本标识 |
| `CLAUDE_AGENT_SDK_CLIENT_APP` | 客户端应用标识 |
| `CLAUDE_CODE_ENTRYPOINT` | 入口点标识（cli, claude-desktop 等） |
| `USER_TYPE` | 用户类型（pro, free, ant 等） |
| `MACRO.VERSION` | 编译时注入的版本号 |

---

## 风险、边界与改进建议

### 已知风险

1. **User-Agent 字符串硬编码依赖**
   - 代码注释明确警告：`claude-cli` 字符串用于日志过滤
   - 修改可能导致日志系统失效
   - **缓解**：任何修改需同步更新日志过滤配置

2. **OAuth 重试的竞态条件**
   - `handleOAuth401Error` 可能被并发调用
   - 如果多个请求同时遇到 401，可能导致重复刷新 token
   - **缓解**：`auth.js` 中的 `handleOAuth401Error` 实现了 token 级别的去重

3. **API Key 与 LLM Gateway 的兼容性问题**
   - 代码注释提到：当 API Key 设置为 LLM Gateway key 时可能失败
   - 当前实现未处理这种场景
   - **建议**：添加检测逻辑或配置选项

### 边界情况

| 场景 | 处理 |
|------|------|
| OAuth token 缺失 | 返回 `error: 'No OAuth token available'` |
| API Key 缺失 | 返回 `error: 'No API key available'` |
| 非 Axios 错误 | 直接抛出，不重试 |
| 403 非撤销错误 | 直接抛出，不重试 |
| 重试后仍失败 | 抛出最后一次错误 |

### 改进建议

1. **添加请求追踪 ID**
   ```typescript
   // 在 User-Agent 中添加请求追踪标识
   const traceId = generateTraceId()
   return `claude-cli/${MACRO.VERSION}... (trace/${traceId})`
   ```

2. **OAuth 重试指数退避**
   - 当前实现仅重试一次
   - 对于间歇性网络问题，可考虑指数退避策略
   - 需平衡用户体验和 API 负载

3. **认证头缓存**
   - 当前每次请求都重新生成认证头
   - OAuth token 有效期内可缓存结果
   - 需注意 token 刷新的原子性

4. **更详细的错误分类**
   ```typescript
   export type AuthError = 
     | { type: 'oauth_token_missing' }
     | { type: 'oauth_token_expired' }
     | { type: 'oauth_token_revoked' }
     | { type: 'api_key_missing' }
     | { type: 'api_key_invalid' }
   ```

5. **单元测试覆盖**
   - User-Agent 格式验证测试
   - OAuth 重试逻辑测试（模拟 401/403）
   - 认证头生成边界条件测试
