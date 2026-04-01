# mtls.ts 研究文档

## 场景与职责

本模块提供 mTLS（双向 TLS）配置和 HTTPS 代理管理功能。核心职责包括：

1. **mTLS 配置加载**：从环境变量读取客户端证书、私钥和密钥口令
2. **HTTPS 代理创建**：创建配置 mTLS 和自定义 CA 证书的 HTTPS Agent
3. **WebSocket TLS 选项**：为 WebSocket 连接提供 TLS 配置
4. **Fetch 选项生成**：为 undici/Bun fetch 生成 TLS 配置选项
5. **全局配置**：配置 Node.js 全局 TLS 设置

该模块是企业环境安全通信的基础设施，支持客户端证书认证和自定义 CA 证书链。

## 功能点目的

### 1. `getMTLSConfig()` - mTLS 配置获取
- **目的**：从环境变量加载 mTLS 配置
- **环境变量**：
  - `CLAUDE_CODE_CLIENT_CERT`：客户端证书文件路径
  - `CLAUDE_CODE_CLIENT_KEY`：客户端私钥文件路径
  - `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE`：私钥口令
- **特性**：使用 `lodash-es/memoize` 缓存结果
- **注意**：`NODE_EXTRA_CA_CERTS` 由 Node.js 运行时自动处理

### 2. `getMTLSAgent()` - HTTPS Agent 创建
- **目的**：创建配置 mTLS 和 CA 证书的 HTTPS Agent
- **配置来源**：
  - mTLS 配置（证书、私钥、口令）
  - CA 证书（来自 `getCACertificates()`）
- **特性**：启用 `keepAlive` 提升性能
- **返回值**：`HttpsAgent` 或 `undefined`（无配置时）

### 3. `getWebSocketTLSOptions()` - WebSocket TLS 选项
- **目的**：为 WebSocket 连接生成 TLS 配置选项
- **返回值**：`tls.ConnectionOptions` 或 `undefined`
- **用途**：MCP WebSocket 传输层 TLS 配置

### 4. `getTLSFetchOptions()` - Fetch TLS 选项
- **目的**：为 undici/Bun fetch 生成 TLS 配置
- **运行时适配**：
  - Bun：返回 `{ tls: TLSConfig }`
  - Node.js：创建 undici Agent 返回 `{ dispatcher: Agent }`
- **延迟加载**：undici 仅在需要时动态 require（~1.5MB）

### 5. `clearMTLSCache()` - 缓存清除
- **目的**：清除 mTLS 配置和 Agent 缓存
- **场景**：配置变更后（如信任对话框接受设置）

### 6. `configureGlobalMTLS()` - 全局 TLS 配置
- **目的**：配置全局 Node.js TLS 设置
- **当前实现**：主要记录 `NODE_EXTRA_CA_CERTS` 检测日志

## 具体技术实现

### 关键流程

```
请求 TLS 配置
    ↓
getMTLSConfig() (memoized)
    ↓
检查环境变量
    ↓
读取证书文件
    ↓
返回 MTLSConfig
    ↓
getMTLSAgent()
    ↓
合并 CA 证书
    ↓
创建 HttpsAgent
```

### 数据结构

```typescript
// mTLS 配置
export type MTLSConfig = {
  cert?: string    // 客户端证书（PEM 格式）
  key?: string     // 客户端私钥（PEM 格式）
  passphrase?: string  // 私钥口令
}

// 完整 TLS 配置（含 CA）
export type TLSConfig = MTLSConfig & {
  ca?: string | string[] | Buffer  // CA 证书
}
```

### 环境变量映射

| 环境变量 | 用途 | 处理方式 |
|----------|------|----------|
| `CLAUDE_CODE_CLIENT_CERT` | 客户端证书路径 | `readFileSync` 读取内容 |
| `CLAUDE_CODE_CLIENT_KEY` | 客户端私钥路径 | `readFileSync` 读取内容 |
| `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | 私钥口令 | 直接使用 |
| `NODE_EXTRA_CA_CERTS` | 额外 CA 证书 | Node.js 自动处理 |

### Fetch 运行时适配

```typescript
if (typeof Bun !== 'undefined') {
  return { tls: tlsConfig }  // Bun 原生支持
}

// Node.js: 使用 undici
const undiciMod = require('undici') as typeof undici
const agent = new undiciMod.Agent({
  connect: {
    cert: tlsConfig.cert,
    key: tlsConfig.key,
    passphrase: tlsConfig.passphrase,
    ...(tlsConfig.ca && { ca: tlsConfig.ca }),
  },
  pipelining: 1,
})
return { dispatcher: agent }
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `https` | HTTPS Agent 类型 |
| `tls` | TLS 选项类型 |
| `undici` (类型) | Node.js fetch 库类型 |
| `lodash-es/memoize.js` | 配置缓存 |
| `./caCerts.js` | CA 证书获取 |
| `./debug.js` | 调试日志 |
| `./fsOperations.js` | 文件系统操作 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/upstreamproxy/relay.ts` | 上游代理 TLS 配置 |
| `src/services/voiceStreamSTT.ts` | 语音流 STT TLS |
| `src/cli/transports/WebSocketTransport.ts` | CLI WebSocket TLS |
| `src/utils/status.tsx` | 状态检查 TLS |
| `src/utils/telemetry/instrumentation.ts` | 遥测 TLS |
| `src/entrypoints/init.ts` | 初始化全局 TLS |
| `src/utils/proxy.ts` | 代理 TLS 配置 |
| `src/utils/managedEnv.ts` | 托管环境 TLS |
| `src/services/mcp/client.ts` | MCP 客户端 TLS |
| `src/remote/SessionsWebSocket.ts` | 远程会话 WebSocket TLS |

### 相关模块

- `./caCerts.ts`：CA 证书加载和管理
- `./proxy.ts`：代理配置（使用本模块的 TLS 选项）

## 风险、边界与改进建议

### 已知风险

1. **证书文件权限**
   - 风险：私钥文件权限可能过于宽松
   - 现状：未检查文件权限
   - 建议：添加权限检查（如 Unix 模式应 600）

2. **证书过期**
   - 风险：缓存的证书可能过期
   - 现状：memoize 缓存无过期机制
   - 建议：添加证书过期检查或定期刷新

3. **密钥口令安全**
   - 风险：口令以明文形式存在于内存
   - 现状：Node.js TLS 接口要求
   - 缓解：依赖进程内存保护

4. **undici 延迟加载**
   - 风险：动态 require 可能失败
   - 现状：try-catch 未包裹
   - 建议：添加错误处理

### 边界情况

| 场景 | 行为 |
|------|------|
| 证书文件不存在 | 捕获错误，记录日志，不包含该字段 |
| 部分配置（仅有证书无私钥） | 返回部分配置，可能连接失败 |
| 空配置（无环境变量） | 返回 `undefined` |
| CA 证书加载失败 | 记录错误，继续无 CA 配置 |
| Bun 环境 | 使用原生 `tls` 选项 |
| Node.js 环境 | 创建 undici Agent |

### 改进建议

1. **证书验证**
   - 当前：简单文件读取
   - 建议：
     - 验证证书格式（PEM 解析）
     - 检查证书有效期
     - 验证证书与私钥匹配

2. **热重载支持**
   - 当前：需手动调用 `clearMTLSCache()`
   - 建议：
     - 文件系统监听证书文件变更
     - 自动刷新配置

3. **更细粒度的配置**
   - 当前：全局配置
   - 建议：
     - 按目标主机配置不同证书
     - 支持 SNI（Server Name Indication）

4. **安全增强**
   - 建议：
     - 支持硬件安全模块（HSM）
     - 支持 PKCS#11 接口
     - 证书固定（Certificate Pinning）

5. **可观测性**
   - 当前：仅调试日志
   - 建议：
     - 记录 TLS 握手事件
     - 记录证书使用统计
     - 证书过期预警

6. **错误处理优化**
   - 当前：简单错误记录
   - 建议：
     - 分类 TLS 错误类型
     - 提供用户友好的错误消息
     - 建议修复步骤

7. **配置来源扩展**
   - 当前：仅环境变量
   - 建议：
     - 支持 settings.json 配置
     - 支持密钥管理系统（KMS）
     - 支持云提供商证书服务

8. **性能优化**
   - 当前：每次创建新 Agent
   - 建议：
     - Agent 池化复用
     - 连接预热
     - TLS 会话恢复
