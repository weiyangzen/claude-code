# src/utils/proxy.ts 研究文档

## 场景与职责

`proxy.ts` 是 Claude Code 的网络代理基础设施，负责为 axios、undici（fetch/WebSocket）、AWS SDK 以及 Anthropic SDK 提供统一的代理、mTLS、CA 证书和 `NO_PROXY` 解析支持。该模块在企业网络、沙箱环境、SSH 隧道（`claude ssh`）等场景下至关重要，是几乎所有出站 HTTP/HTTPS/WebSocket 流量的必经之路。

## 功能点目的

1. **代理 URL 解析（`getProxyUrl`、`getNoProxy`）**
   - 优先读取小写环境变量（`https_proxy` > `HTTPS_PROXY` > `http_proxy` > `HTTP_PROXY`）。
   - `NO_PROXY` / `no_proxy` 同样优先小写。

2. **NO_PROXY 匹配（`shouldBypassProxy`）**
   - 支持精确主机名、带前导点的域名后缀（匹配子域和根域）、通配符 `*`、带端口匹配、IP 地址。
   - 按逗号或空格分割规则，大小写不敏感。

3. **代理 Agent 创建**
   - `createHttpsProxyAgent`：基于 `https-proxy-agent` 创建代理 agent，支持 mTLS 证书、CA 证书、可选的代理侧 DNS 解析（`CLAUDE_CODE_PROXY_RESOLVES_HOSTS`）。
   - `getProxyAgent`：基于 `undici` 的 `EnvHttpProxyAgent`，自动处理 `NO_PROXY`，支持 mTLS/CA 配置；使用 `lodash/memoize` 缓存，避免重复创建。
   - `getWebSocketProxyAgent` / `getWebSocketProxyUrl`：为 WebSocket 连接提供代理 agent 或代理 URL（Bun 原生 WebSocket 使用字符串 `proxy` 选项）。

4. **Axios 实例与全局配置**
   - `createAxiosInstance`：创建带有独立代理 interceptor 的 axios 实例，支持 `NO_PROXY` 和 mTLS。
   - `configureGlobalAgents`：为全局 `axios` 和 `undici` 设置代理 agent/mTLS agent，支持重复调用（会 eject 旧 interceptor、重置 defaults）。
   - `disableKeepAlive`：在检测到 stale-pool ECONNRESET 后禁用 fetch keep-alive，强制后续请求开新连接。

5. **Fetch 选项组装（`getProxyFetchOptions`）**
   - 为 Anthropic SDK 提供 fetch 选项（`dispatcher` / `proxy` / `unix` / `tls` / `keepalive`）。
   - 支持 `ANTHROPIC_UNIX_SOCKET`（`claude ssh` 的远程 Unix socket 隧道），但仅限 `forAnthropicAPI: true` 的调用方，防止 MCP/SSE 等流量被误路由到 Anthropic API。

6. **AWS SDK 代理配置（`getAWSClientProxyConfig`）**
   - 动态导入 `@smithy/node-http-handler` 和 `@aws-sdk/credential-provider-node`（延迟 ~929KB）。
   - 返回包含 `requestHandler` 和 `credentials` 的对象，可直接展开到 AWS 服务客户端构造函数中。

## 具体技术实现

- **延迟加载**：
  - `@aws-sdk/credential-provider-node` 和 `@smithy/node-http-handler` 在 `getAWSClientProxyConfig` 中动态 `import()`。
  - `undici` 在 `getProxyAgent`、`getProxyFetchOptions`（Node 路径）、`configureGlobalAgents` 中通过 `require('undici')` 延迟加载（~1.5MB）。
- **mTLS + CA 集成**：所有创建 agent 的路径都会读取 `getMTLSConfig()` 和 `getCACertificates()`，自动注入 `cert`/`key`/`passphrase`/`ca`。
- **Bun 兼容**：多处使用 `typeof Bun !== 'undefined'` 分支，为 Bun 提供 `proxy: proxyUrl` 或 `unix: socketPath` 选项，而不是 Node 的 `dispatcher`。
- **缓存管理**：`getProxyAgent` 使用 `memoize`；提供 `clearProxyCache()` 和 `_resetKeepAliveForTesting()` 供测试和配置热重载使用。

## 关键代码路径与文件引用

- **本文件**：`src/utils/proxy.ts`
- **调用方**：
  - `src/services/api/claude.ts` — Anthropic API 调用。
  - `src/services/api/grove.ts` — Grove 服务调用。
  - 各种 WebSocket/MCP/SSE 传输层。
  - `src/utils/mtls.ts` / `src/utils/caCerts.ts` — 被本模块依赖。
- **依赖**：
  - `axios`、`https-proxy-agent` — HTTP 代理。
  - `undici` — 现代 fetch/代理（延迟加载）。
  - `lodash-es/memoize.js` — agent 缓存。
  - `src/utils/caCerts.js` — `getCACertificates`。
  - `src/utils/debug.js` — `logForDebugging`。
  - `src/utils/envUtils.js` — `isEnvTruthy`。
  - `src/utils/mtls.js` — `getMTLSAgent`、`getMTLSConfig`、`getTLSFetchOptions`、`TLSConfig`。

## 依赖与外部交互

- 读取大量环境变量：`https_proxy`、`HTTPS_PROXY`、`http_proxy`、`HTTP_PROXY`、`no_proxy`、`NO_PROXY`、`CLAUDE_CODE_PROXY_RESOLVES_HOSTS`、`CLAUDE_CODE_CLIENT_CERT`、`CLAUDE_CODE_CLIENT_KEY`、`CLAUDE_CODE_CLIENT_KEY_PASSPHRASE`、`NODE_EXTRA_CA_CERTS`、`ANTHROPIC_UNIX_SOCKET`。
- 可能触发网络 DNS 解析（除非启用 `CLAUDE_CODE_PROXY_RESOLVES_HOSTS`）。
- 动态加载 AWS SDK 和 undici，涉及磁盘 I/O。

## 风险、边界与改进建议

1. **环境变量大小写混合**：虽然模块优先小写，但不同操作系统/Shell 对环境变量大小写的处理不同（Windows 不区分大小写，POSIX 区分）。若用户同时设置 `https_proxy` 和 `HTTPS_PROXY` 为不同值，行为可能不符合预期。
2. **`shouldBypassProxy` 的 URL 解析失败**：若 `urlString` 不是合法 URL，`new URL(urlString)` 会抛错，函数返回 `false`（不走代理）。这在某些只传 host:port 的调用方中可能导致意外走代理。
3. **`getProxyAgent` 的 memoize 缓存键**：只缓存 `uri` 字符串，不缓存 `mtlsConfig` 或 `caCerts` 的变化。如果运行时用户修改了证书配置并调用 `clearProxyCache()`，缓存会被清空；但如果调用方忘记清空，agent 将持有旧证书。
4. **`configureGlobalAgents` 的副作用**：修改全局 `axios.defaults` 和 undici 的 `setGlobalDispatcher`，在多实例测试或并发环境中可能产生交叉污染。`proxyInterceptorId` 的 eject 逻辑只能处理同一进程内的重复调用。
5. **Bun 的 `proxy` 字符串不支持 mTLS**：在 `getProxyFetchOptions` 的 Bun 分支中，返回 `{ proxy: proxyUrl, ...getTLSFetchOptions() }`，但 Bun 的 `proxy` 选项本身不传递 TLS 证书，mTLS 可能无法通过 HTTP CONNECT 隧道生效。这一点在代码注释中没有明确说明。
6. **改进建议**：
   - 为 `shouldBypassProxy` 增加对裸 host:port 字符串的兼容，或在上游调用方统一做 URL 格式化。
   - 考虑将 `configureGlobalAgents` 拆分为“实例级配置”和“全局配置”两个 API，减少测试中的副作用。
   - 增加对 `PAC`（Proxy Auto-Config）文件的支持，满足大型企业网络的复杂代理规则需求。
   - 在 `getProxyFetchOptions` 的 Bun 分支中增加 debug 日志，明确提示当前 proxy + mTLS 组合在 Bun 下的已知限制。
