# apiPreconnect.ts 研究文档

## 场景与职责

`apiPreconnect.ts` 实现了 Anthropic API 的预连接功能，通过在实际 API 请求前发起一个 "fire-and-forget" 的 TCP+TLS 握手，来重叠网络延迟与启动初始化工作，从而减少用户感知的延迟。

## 功能点目的

### 1. 连接预热 (`preconnectAnthropicApi`)
- 在应用初始化阶段提前建立与 API 服务器的 TCP 连接
- 利用 Bun 的全局 keep-alive 连接池复用预热连接
- 减少首次 API 请求的延迟（约 100-200ms）

### 2. 条件跳过
- 云提供商（Bedrock/Vertex/Foundry）：不同端点和认证方式
- 代理配置（HTTPS_PROXY/HTTP_PROXY）：SDK 使用自定义 dispatcher
- mTLS/Unix Socket：不共享全局连接池

## 具体技术实现

### 关键流程

```typescript
export function preconnectAnthropicApi(): void {
  // 防止重复调用
  if (fired) return
  fired = true

  // 跳过云提供商
  if (
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
  ) {
    return
  }

  // 跳过代理/mTLS/Unix Socket
  if (
    process.env.HTTPS_PROXY ||
    process.env.https_proxy ||
    process.env.HTTP_PROXY ||
    process.env.http_proxy ||
    process.env.ANTHROPIC_UNIX_SOCKET ||
    process.env.CLAUDE_CODE_CLIENT_CERT ||
    process.env.CLAUDE_CODE_CLIENT_KEY
  ) {
    return
  }

  // 获取基础 URL（支持 staging/local/自定义网关）
  const baseUrl =
    process.env.ANTHROPIC_BASE_URL || getOauthConfig().BASE_API_URL

  // 发起 HEAD 请求（无响应体，连接可立即复用）
  void fetch(baseUrl, {
    method: 'HEAD',
    signal: AbortSignal.timeout(10_000),
  }).catch(() => {})
}
```

### 调用时机

根据代码注释，该函数必须在以下操作**之后**调用：
1. `applyExtraCACertsFromConfig()` - 应用额外 CA 证书
2. `configureGlobalAgents()` - 配置全局代理

早期在 `cli.tsx` 中调用被移除，因为那会发生在 `settings.json` 加载之前，导致 `ANTHROPIC_BASE_URL` 等配置不可见。

## 关键代码路径与文件引用

### 本文件导出
- `preconnectAnthropicApi(): void` - 预连接主函数

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/constants/oauth.ts` | 获取 OAuth 配置（`getOauthConfig`） |
| `src/utils/envUtils.ts` | 环境变量检查（`isEnvTruthy`） |

### 调用方
- `src/entrypoints/init.ts` - 初始化流程中调用

## 依赖与外部交互

### 外部依赖
- 使用 Bun 内置的 `fetch` API
- 依赖 Bun 的全局 keep-alive 连接池

### 环境变量检查
| 变量 | 作用 |
|------|------|
| `CLAUDE_CODE_USE_BEDROCK` | 使用 Bedrock 提供商 |
| `CLAUDE_CODE_USE_VERTEX` | 使用 Vertex 提供商 |
| `CLAUDE_CODE_USE_FOUNDRY` | 使用 Foundry 提供商 |
| `HTTPS_PROXY`/`https_proxy` | HTTPS 代理 |
| `HTTP_PROXY`/`http_proxy` | HTTP 代理 |
| `ANTHROPIC_UNIX_SOCKET` | Unix Socket 路径 |
| `CLAUDE_CODE_CLIENT_CERT` | mTLS 客户端证书 |
| `CLAUDE_CODE_CLIENT_KEY` | mTLS 客户端密钥 |
| `ANTHROPIC_BASE_URL` | 自定义 API 基础 URL |

## 风险、边界与改进建议

### 已知限制
1. **Bun 专属**：依赖 Bun 的 fetch 和连接池行为
2. **超时固定**：10 秒超时不可配置
3. **无反馈机制**：失败静默处理，调用方无法感知

### 边界条件
1. **重复调用保护**：使用 `fired` 标志确保只执行一次
2. **网络故障**：`catch(() => {})` 确保网络错误不影响启动
3. **配置变更**：不支持运行时配置变更（如切换代理）

### 安全风险
1. **信息泄露**：HEAD 请求可能暴露客户端 IP 和 User-Agent
2. **中间人攻击**：依赖已配置的 CA 证书和代理设置

### 改进建议
1. **可配置超时**：允许通过环境变量调整超时时间
2. **诊断信息**：在调试模式下记录预连接结果
3. **重试机制**：对特定错误码（如 502/503）进行有限重试
4. **连接健康检查**：定期验证连接池状态

### 测试建议
1. 测试各种代理配置下的跳过行为
2. 测试云提供商配置下的跳过行为
3. 测试重复调用不会触发额外请求
4. 测试网络故障不会导致崩溃
