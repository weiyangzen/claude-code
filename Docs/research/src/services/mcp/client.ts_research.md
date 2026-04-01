# 研究文档：src/services/mcp/client.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 文件规模：~3348 行（TypeScript）  
> 研究日期：2026-04-01  
> 执行器：kimi / model=k2p5

---

## 1. 场景与职责

`src/services/mcp/client.ts` 是 **Claude Code CLI 的 MCP（Model Context Protocol）客户端核心引擎**。它承担了所有与外部 MCP Server 建立连接、发现能力、调用工具、处理结果的生命周期管理职责。

### 1.1 业务场景
- **多源 MCP 配置聚合**：支持从 `.mcp.json`（project/local）、用户全局配置、claude.ai 云端连接器、插件动态注入、企业托管策略等多来源加载 MCP Server 配置。
- **异构传输协议统一接入**：同一套抽象层覆盖 stdio（子进程）、SSE（Server-Sent Events）、Streamable HTTP、WebSocket、IDE 专用通道（sse-ide/ws-ide）、SDK 内进程桥接（sdk）、claude.ai 代理（claudeai-proxy）等 8 种传输类型。
- **工具/资源/命令的动态发现**：在启动或重连时批量拉取各 Server 的 `tools/list`、`resources/list`、`prompts/list`，并映射为 Claude Code 内部统一的 `Tool[]` / `Command[]` / `ServerResource[]`。
- **运行时工具调用与结果治理**：实际调用 MCP Tool（`tools/call`），处理超大输出（持久化到磁盘或截断）、二进制内容（图片/音频/Blob 的落盘与压缩）、URL Elicitation（-32042 交互式授权重试）、会话过期自动重建等复杂运行时问题。

### 1.2 架构定位
在整体架构中，该文件位于 **Service Layer**（`src/services/mcp/`），向上为：
- `src/services/mcp/useManageMCPConnections.ts`（React Hook，负责状态同步、自动重连、通知监听）
- `src/cli/print.ts`（结构化 I/O / SDK 模式下的控制消息处理）
- `src/main.tsx`（CLI 入口，启动时批量预取 MCP 资源）
- `src/utils/api.ts`（API 层，在构建 tool context 时预加载 MCP 工具）

向下依赖：
- `@modelcontextprotocol/sdk`（官方 MCP SDK，提供 `Client`、各类 `Transport`、OAuth 辅助类）
- 系统级能力：子进程管理、文件 I/O、图片压缩、代理配置、TLS/mTLS、OAuth Token 存储与刷新。

---

## 2. 功能点目的

| 功能模块 | 目的说明 |
|---------|---------|
| **连接管理 (`connectToServer`)** | 根据配置类型创建对应 Transport，完成 MCP 握手（`client.connect`），设置超时、错误处理、关闭清理，并返回带能力声明的连接对象。 |
| **认证与授权缓存 (`McpAuthCache`)** | 对近期返回 401 的远程 Server 进行 15 分钟短路缓存，避免重复无意义的 OAuth 探测和连接尝试；同时提供 `clearMcpAuthCache` 供用户手动刷新。 |
| **claude.ai 代理 Fetch (`createClaudeAiProxyFetch`)** | 为 claude.ai 云端 MCP 连接器封装带 Bearer Token 的 fetch，支持一次 401 后的强制刷新重试，防止因单点 Token 过期导致所有连接器集体失效。 |
| **请求超时包装 (`wrapFetchWithTimeout`)** | 解决 `AbortSignal.timeout()` 在 Bun 中 GC 延迟导致内存泄漏的问题；为 Streamable HTTP 的 POST 请求附加规范要求的 `Accept: application/json, text/event-stream` 头。 |
| **工具发现与映射 (`fetchToolsForClient`)** | 调用 `tools/list`，将 MCP Tool 映射为 Claude Code 内部 `Tool` 对象（含权限检查、自动分类、前缀命名 `mcp__{server}__{tool}` 等）。 |
| **资源/命令发现 (`fetchResourcesForClient` / `fetchCommandsForClient`)** | 同理映射 `resources/list` 和 `prompts/list`；其中 prompts 被映射为 slash commands。 |
| **工具调用 (`callMCPTool` / `callMCPToolWithUrlElicitationRetry`)** | 执行 `tools/call`，处理进度回调、超时、401 认证失效、会话过期（404/-32001）、URL Elicitation（-32042）重试循环。 |
| **结果转换与治理 (`transformMCPResult` / `processMCPResult`)** | 将 MCP 返回的 `content`（text/image/audio/resource/resource_link）转换为 Anthropic API 的 `ContentBlockParam[]`；对超大结果进行文件持久化或截断。 |
| **SDK MCP 桥接 (`setupSdkMcpClients`)** | 为运行在 SDK 进程内的 MCP Server 建立 `SdkControlClientTransport`，通过控制消息通道（stdout/stdin）完成跨进程 JSON-RPC 转发。 |
| **批量预取 (`prefetchAllMcpResources` / `getMcpToolsCommandsAndResources`)** | 启动阶段按本地/远程分组并发连接，本地用低并发（默认 3）避免子进程资源争抢，远程用高并发（默认 20）加速网络握手。 |

---

## 3. 具体技术实现

### 3.1 关键数据结构与类型

#### 3.1.1 连接对象（来自 `src/services/mcp/types.ts`）
```ts
export type ConnectedMCPServer = {
  client: Client              // MCP SDK Client 实例
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

#### 3.1.2 配置类型（`ScopedMcpServerConfig`）
支持 8 种传输变体：
- `stdio`（默认）：`{ command, args?, env? }`
- `sse`：`{ type: 'sse', url, headers?, headersHelper?, oauth? }`
- `http`：`{ type: 'http', url, headers?, headersHelper?, oauth? }`
- `ws`：`{ type: 'ws', url, headers?, headersHelper? }`
- `sse-ide` / `ws-ide`：IDE 扩展专用，带 `ideName` 和可选 `authToken`
- `sdk`：SDK 内进程占位，`{ type: 'sdk', name }`
- `claudeai-proxy`：claude.ai 云端连接器，`{ type: 'claudeai-proxy', url, id }`

### 3.2 核心流程：连接建立（`connectToServer`）

`connectToServer` 被 `lodash-es/memoize` 包裹，缓存键为 `getServerCacheKey(name, serverRef)`（即 `${name}-${jsonStringify(serverRef)}`）。这意味着**同一配置在同一进程内只会建立一次真实连接**，后续调用直接返回缓存对象。

#### 3.2.1 Transport 创建分支
| 配置类型 | Transport 实现 | 关键细节 |
|---------|---------------|---------|
| `sse` | `SSEClientTransport` | 使用 `ClaudeAuthProvider` 处理 OAuth；`eventSourceInit.fetch` 不套超时（SSE 长连接），但普通 API fetch 套 `wrapFetchWithTimeout` + `wrapFetchWithStepUpDetection` |
| `http` | `StreamableHTTPClientTransport` | 同上；若存在 OAuth Token 则不用 `sessionIngressToken`，否则注入 JWT；POST 请求通过 `wrapFetchWithTimeout` 附加规范 Accept 头 |
| `ws` / `ws-ide` | `WebSocketTransport`（自定义） | Bun 环境用 `globalThis.WebSocket`（支持 headers/proxy/tls），Node 环境动态引入 `ws` 包；`ws-ide` 带 `X-Claude-Code-Ide-Authorization` |
| `stdio` | `StdioClientTransport` | 子进程 stderr 被 pipe 并收集（上限 64MB），用于连接失败时诊断；支持 `CLAUDE_CODE_SHELL_PREFIX` 环境变量覆盖命令 |
| `claudeai-proxy` | `StreamableHTTPClientTransport` | 通过 `createClaudeAiProxyFetch` 注入 claude.ai OAuth Bearer Token |
| Chrome MCP / Computer Use | 自定义 `InProcessTransport` | 为避免 325MB 子进程开销，直接在主进程内启动 MCP Server（`@ant/claude-for-chrome-mcp` 等） |
| `sdk` | 不在此函数处理 | 由 `setupSdkMcpClients` 单独处理 |

#### 3.2.2 连接超时与错误处理
- 连接阶段设置独立超时（默认 30s，由 `MCP_TIMEOUT` 控制）。
- 对 `sse`/`http`/`claudeai-proxy` 的 `UnauthorizedError` 或 401 HTTP 错误，返回 `type: 'needs-auth'` 并写入 15 分钟缓存。
- 对 `sse-ide`/`ws-ide` 失败单独上报 `tengu_mcp_ide_server_connection_failed` 事件。

#### 3.2.3 连接存活监控（`onerror` / `onclose`）
连接成功后，文件为 `client.onerror` 和 `client.onclose` 注入了增强逻辑：
- **会话过期检测**：对 HTTP 传输，若错误为 404 且消息包含 `"code":-32001`，则主动调用 `client.close()` 触发重连。
- **终端错误计数**：对远程传输，连续 3 次终端错误（`ECONNRESET`、`ETIMEDOUT`、`EPIPE`、`SSE stream disconnected` 等）后强制关闭并触发重连。
- **缓存清理**：`onclose` 中清除 `connectToServer`、`fetchToolsForClient`、`fetchResourcesForClient`、`fetchCommandsForClient` 的缓存，确保下次调用重建连接。

#### 3.2.4 stdio 子进程优雅退出
`cleanup()` 中对 stdio 传输实现了 SIGINT → 100ms → SIGTERM → 400ms → SIGKILL 的渐进式杀进程逻辑，并配合 `process.kill(pid, 0)` 探测进程是否已退出，总清理时间控制在 ~600ms。

### 3.3 核心流程：工具调用（`callMCPTool`）

#### 3.3.1 超时与进度
- 工具调用默认超时 **100,000,000ms（约 27.8 小时）**，可通过 `MCP_TOOL_TIMEOUT` 覆盖。
- 每 30 秒记录一次进度日志。
- 使用 `Promise.race` 在 SDK 内部超时之外再套一层自定义超时，防止 SSE 断流导致 SDK 超时失效。

#### 3.3.2 错误分类与恢复
| 错误类型 | 处理行为 |
|---------|---------|
| `isError: true`（MCP 业务错误） | 抛出 `McpToolCallError`，携带 `_meta` |
| 401 / `UnauthorizedError` | 抛出 `McpAuthError`，触发上层将 Server 标记为 `needs-auth` |
| 404 + JSON-RPC `-32001`（Session not found） | 清除连接缓存，抛出 `McpSessionExpiredError`，上层可重试一次 |
| `-32000` "Connection closed" on HTTP | 同样视为会话过期，清除缓存后重试 |
| `AbortError` | 静默返回 `{ content: undefined }` |

#### 3.3.3 URL Elicitation 重试（`-32042`）
`callMCPToolWithUrlElicitationRetry` 实现了最多 3 次的 URL Elicitation 重试：
1. 工具返回 `McpError` 且 `code === ErrorCode.UrlElicitationRequired`。
2. 解析 `error.data.elicitations`，过滤出 `mode === 'url'` 的有效条目。
3. 先执行 `runElicitationHooks`（允许程序化自动处理）。
4. 若未解决，进入 UI 交互：
   - **Print/SDK 模式**：通过 `handleElicitation` 回调走结构化 I/O。
   - **REPL 模式**：将请求加入 `AppState.elicitation.queue`，由 `ElicitationDialog` 展示两阶段（同意 → 等待完成 → 重试）。
5. 用户完成 Elicitation 后，循环重试工具调用。

### 3.4 结果转换与超大输出治理

#### 3.4.1 `transformResultContent`
将 MCP `PromptMessage['content']` 映射为 Anthropic `ContentBlockParam[]`：
- `text` → `text` block
- `image` → 经 `maybeResizeAndDownsampleImageBuffer` 压缩后的 `image` block
- `audio` / 非图片 `blob` → 经 `persistBlobToTextBlock` 写入磁盘，返回文件路径文本
- `resource` → 文本或图片（视 `text`/`blob` 及 MIME 类型而定）
- `resource_link` → 文本摘要

#### 3.4.2 `processMCPResult`
在 `transformMCPResult` 之后进行大小治理：
1. 调用 `mcpContentNeedsTruncation` 判断内容是否超过 Token 上限。
2. 若未超限，直接返回。
3. 若 `ENABLE_MCP_LARGE_OUTPUT_FILES` 为 falsy，回退到旧截断行为。
4. 若内容包含图片，也回退截断（避免 JSON 持久化破坏图片压缩）。
5. 否则将内容字符串持久化到磁盘（`persistToolResult`），返回 `getLargeOutputInstructions` 引导模型读取文件。

### 3.5 批量连接与并发控制

`getMcpToolsCommandsAndResources` 是启动阶段的核心入口：
1. 将 Server 列表按 `isLocalMcpServer` 分为本地（stdio/sdk）和远程两组。
2. 本地使用 `getMcpServerConnectionBatchSize()`（默认 3）并发，避免子进程爆炸。
3. 远程使用 `getRemoteMcpServerConnectionBatchSize()`（默认 20）并发，加速网络 I/O。
4. 对 `needs-auth` 的 Server 先检查 15 分钟缓存或 `hasMcpDiscoveryButNoToken`，满足条件则直接跳过连接，注入 `createMcpAuthTool`。
5. 连接成功后并行拉取 `tools`、`commands`、`skills`（feature `MCP_SKILLS`）、`resources`。
6. 若某 Server 支持 resources 且全局尚未添加，则注入 `ListMcpResourcesTool` 和 `ReadMcpResourceTool`（仅一次）。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件对外暴露的关键 API

```ts
// 连接与缓存
export const connectToServer: (...)
export function clearServerCache(name, serverRef)
export async function ensureConnectedClient(client)
export function areMcpConfigsEqual(a, b)

// 发现
export const fetchToolsForClient: (...)
export const fetchResourcesForClient: (...)
export const fetchCommandsForClient: (...)

// 调用
export async function callIdeRpc(toolName, args, client)
export async function callMCPToolWithUrlElicitationRetry(...)

// 结果处理
export async function transformResultContent(...)
export async function transformMCPResult(...)
export async function processMCPResult(...)

// 批量入口
export async function getMcpToolsCommandsAndResources(...)
export function prefetchAllMcpResources(mcpConfigs)
export async function reconnectMcpServerImpl(name, config)

// SDK 桥接
export async function setupSdkMcpClients(sdkMcpConfigs, sendMcpMessage)

// 认证辅助
export function createClaudeAiProxyFetch(innerFetch)
export function wrapFetchWithTimeout(baseFetch)
export function clearMcpAuthCache()
```

### 4.2 调用方（上游）

| 文件 | 调用内容 | 场景 |
|-----|---------|------|
| `src/main.tsx` | `prefetchAllMcpResources`, `getMcpToolsCommandsAndResources` | CLI 启动时预加载 MCP 工具；`/clear` 后重连 |
| `src/utils/api.ts` | `prefetchAllMcpResources` | 构建 API tool context 时并行加载 |
| `src/services/mcp/useManageMCPConnections.ts` | `getMcpToolsCommandsAndResources`, `reconnectMcpServerImpl`, `clearServerCache`, `fetchToolsForClient`, `fetchResourcesForClient`, `fetchCommandsForClient` | React Hook 管理连接生命周期、自动重连、列表变更通知刷新 |
| `src/cli/print.ts` | `setupSdkMcpClients`, `connectToServer`, `clearServerCache`, `fetchToolsForClient`, `areMcpConfigsEqual`, `reconnectMcpServerImpl` | SDK/结构化 I/O 模式下的 MCP 控制消息处理 |
| `src/tools/McpAuthTool/McpAuthTool.ts` | `reconnectMcpServerImpl`, `clearMcpAuthCache` | 用户通过 `/mcp auth` 完成 OAuth 后重连 Server |
| `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts` | `ensureConnectedClient`, `fetchResourcesForClient` | 运行时列出 MCP 资源 |
| `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` | `ensureConnectedClient` | 运行时读取 MCP 资源 |
| `src/services/mcp/claudeai.ts` | `clearMcpAuthCache` | claude.ai 连接器登录状态变化时清除缓存 |

### 4.3 被调用方（下游依赖）

| 文件/模块 | 用途 |
|----------|------|
| `@modelcontextprotocol/sdk/client/index.js` | `Client` 实例化 |
| `@modelcontextprotocol/sdk/client/sse.js` | `SSEClientTransport` |
| `@modelcontextprotocol/sdk/client/stdio.js` | `StdioClientTransport` |
| `@modelcontextprotocol/sdk/client/streamableHttp.js` | `StreamableHTTPClientTransport` |
| `@modelcontextprotocol/sdk/shared/transport.js` | `createFetchWithInit`, `FetchLike`, `Transport` |
| `@modelcontextprotocol/sdk/types.js` | JSON-RPC Schema（`CallToolResultSchema`, `ListToolsResultSchema` 等） |
| `@modelcontextprotocol/sdk/client/auth.js` | `UnauthorizedError` |
| `src/services/mcp/auth.ts` | `ClaudeAuthProvider`, `wrapFetchWithStepUpDetection`, `hasMcpDiscoveryButNoToken` |
| `src/services/mcp/config.ts` | `getAllMcpConfigs`, `isMcpServerDisabled` |
| `src/services/mcp/headersHelper.ts` | `getMcpServerHeaders`（动态头获取） |
| `src/services/mcp/elicitationHandler.ts` | `registerElicitationHandler`, `runElicitationHooks`, `runElicitationResultHooks` |
| `src/services/mcp/SdkControlTransport.ts` | `SdkControlClientTransport` |
| `src/services/mcp/InProcessTransport.ts` | `createLinkedTransportPair`（Chrome/Computer Use 内进程） |
| `src/services/mcp/types.ts` | 所有 MCP 相关类型定义 |
| `src/utils/mcpWebSocketTransport.ts` | `WebSocketTransport`（自定义 WS Transport） |
| `src/utils/mcpOutputStorage.ts` | `persistBinaryContent`, `getBinaryBlobSavedMessage` |
| `src/utils/mcpValidation.ts` | `mcpContentNeedsTruncation`, `truncateMcpContentIfNeeded`, `getContentSizeEstimate` |
| `src/utils/toolResultStorage.ts` | `persistToolResult`, `isPersistError` |
| `src/utils/imageResizer.ts` | `maybeResizeAndDownsampleImageBuffer` |
| `src/utils/proxy.ts` | `getProxyFetchOptions`, `getWebSocketProxyAgent`, `getWebSocketProxyUrl` |
| `src/utils/mtls.ts` | `getWebSocketTLSOptions` |
| `src/utils/auth.ts` | `checkAndRefreshOAuthTokenIfNeeded`, `getClaudeAIOAuthTokens`, `handleOAuth401Error` |
| `src/utils/sessionIngressAuth.ts` | `getSessionIngressAuthToken` |
| `src/utils/subprocessEnv.ts` | `subprocessEnv()`（子进程环境变量） |
| `src/utils/cleanupRegistry.ts` | `registerCleanup`（进程退出清理） |
| `src/utils/http.ts` | `getMCPUserAgent` |
| `src/utils/ide.ts` | `maybeNotifyIDEConnected` |
| `src/utils/claudeInChrome/mcpServer.ts` | `createChromeContext` |
| `src/utils/computerUse/mcpServer.ts` | `createComputerUseMcpServerForCli` |
| `src/skills/mcpSkills.js` | `fetchMcpSkillsForClient`（feature `MCP_SKILLS`） |
| `src/tools/MCPTool/MCPTool.ts` | `MCPTool` 基型 |
| `src/tools/MCPTool/classifyForCollapse.ts` | `classifyMcpToolForCollapse` |
| `src/tools/ListMcpResourcesTool/...` | `ListMcpResourcesTool` |
| `src/tools/ReadMcpResourceTool/...` | `ReadMcpResourceTool` |
| `src/tools/McpAuthTool/McpAuthTool.ts` | `createMcpAuthTool` |

---

## 5. 依赖与外部交互

### 5.1 第三方依赖
- **`@modelcontextprotocol/sdk`**（核心）：提供 MCP 协议客户端、Transport 抽象、OAuth 辅助类、JSON-RPC Schema。
- **`lodash-es`**：`memoize`, `mapValues`, `zipObject`, `omit`, `reject` 等。
- **`p-map`**：批量连接的并发控制（替代早期手写的 sequential batch）。
- **`ws`**：Node 环境下的 WebSocket 客户端（Bun 环境直接用原生 `WebSocket`）。
- **`zod/v4`**：配置校验（主要在 `types.ts`）。

### 5.2 环境变量
| 变量 | 作用 |
|-----|------|
| `MCP_TIMEOUT` | 连接超时（默认 30000ms） |
| `MCP_TOOL_TIMEOUT` | 工具调用超时（默认 100_000_000ms） |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | 本地 Server 并发数（默认 3） |
| `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` | 远程 Server 并发数（默认 20） |
| `CLAUDE_CODE_SHELL_PREFIX` | 覆盖 stdio 命令前缀 |
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | SDK MCP 工具是否跳过 `mcp__` 前缀 |
| `ENABLE_MCP_LARGE_OUTPUT_FILES` | 超大输出是否持久化到文件 |
| `CLAUDE_CODE_ENABLE_XAA` | 是否启用 Cross-App Access (XAA) OAuth 流 |

### 5.3 外部系统交互
- **OAuth / OIDC 服务器**：通过 `ClaudeAuthProvider` 和 `src/services/mcp/auth.ts` 进行 RFC 8414 / RFC 9728 发现、PKCE 授权码流、Token 刷新、Token 撤销（RFC 7009）。
- **claude.ai API**：通过 `createClaudeAiProxyFetch` 访问 `MCP_PROXY_URL`，携带用户 OAuth Access Token。
- **Session Ingress（CCR 代理）**：远程会话中，部分 HTTP/WS MCP URL 被重写为 CCR 代理地址，通过 `getSessionIngressAuthToken()` 注入 JWT。
- **IDE 扩展**：通过 `sse-ide` / `ws-ide` 与 VS Code 等 IDE 的 Claude 扩展通信，支持 `mcp__ide__executeCode` 等受限工具。
- **操作系统子进程**：stdio 传输直接 `spawn` 外部命令，涉及信号管理、stderr 收集、环境变量注入。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险与边界

#### 6.1.1 连接缓存的“配置漂移”风险
`connectToServer` 使用 `memoize`，缓存键为 `name + jsonStringify(serverRef)`。若 Server 的 `env` 或 `headers` 在运行时被动态修改（如通过外部脚本），但配置对象引用未变，则**可能命中旧缓存**。虽然 `onclose` 会清除缓存，但在长连接未断开期间，配置变更不会生效。

#### 6.1.2 stdio 子进程僵尸进程风险
尽管 `cleanup()` 实现了 SIGINT → SIGTERM → SIGKILL 的渐进式退出，但：
- `StdioClientTransport.close()` 仅发送 abort 信号，部分 Docker 容器或异常 Server 可能忽略 SIGINT/SIGTERM。
- 若 `process.kill(pid, 0)` 在极短时间内连续调用，存在竞态窗口；且对 Windows 子进程，Node.js 的 `process.kill` 行为与 POSIX 不同（SIGINT 可能无法送达）。

#### 6.1.3 远程传输重连的“悬挂 Promise”风险
`client.onerror` 中检测到会话过期或最大重连次数耗尽时，调用 `client.close()`。这会触发 SDK 的 `_onclose()`，拒绝所有 pending request handlers。但**若 SDK 内部未正确 reject 某些边缘请求（如正在流式传输的 SSE 请求）**，`callMCPTool` 的 `Promise.race` 可能直到自定义超时（27.8 小时）才触发，导致工具调用长时间悬挂。

#### 6.1.4 401 缓存的“过度跳过”风险
`isMcpAuthCached` 使用 15 分钟 TTL。若用户在这 15 分钟内通过其他方式（如 Web UI）完成了授权，CLI 子进程仍会因缓存而跳过连接，直到 TTL 过期或用户手动触发 `/mcp auth`。

#### 6.1.5 大输出持久化的磁盘与隐私风险
`processMCPResult` 在内容超限时写入磁盘。虽然文件路径是临时目录，但：
- 未显式设置文件权限（如 `0o600`），多用户系统可能存在可读风险。
- 未对大输出文件设置自动清理策略，长期运行可能累积大量临时文件。

#### 6.1.6 `fetchToolsForClient` 的模型提示注入风险
MCP Tool 的 `description` 和 `prompt` 直接来自外部 Server。虽然代码对 `instructions` 和 `prompt()` 做了 2048 字符截断，但**未对内容进行语义过滤**。恶意 Server 可能通过超长描述进行提示注入（prompt injection）或上下文填充攻击。

#### 6.1.7 `headersHelper` 的命令执行风险
`getMcpHeadersFromHelper` 使用 `execFileNoThrowWithCwd(config.headersHelper, [], { shell: true, ... })`。虽然 `headersHelper` 路径来自受信任的配置文件，但 `shell: true` 意味着任何注入特殊字符的路径都可能导致命令注入。当前代码未对 `headersHelper` 路径做严格校验。

### 6.2 改进建议

#### 6.2.1 连接缓存：引入版本号或显式失效机制
建议在 `ScopedMcpServerConfig` 中增加一个可选的 `revision` 或 `updatedAt` 字段，纳入 `getServerCacheKey` 计算。这样外部配置热更新时无需等待连接断开即可生效。

#### 6.2.2 stdio 子进程：增加 `tree-kill` 或进程组终止
对 stdio Server，尤其是 Docker / npx 启动的进程，子进程可能再 fork 孙子进程。当前仅杀 `stdioTransport.pid` 可能导致孤儿进程。建议：
- 在 POSIX 系统上尝试 `process.kill(-pid, 'SIGTERM')`（杀进程组）。
- 或引入 `tree-kill` 依赖，确保整棵进程树被清理。

#### 6.2.3 悬挂 Promise：为 `client.callTool` 增加更严格的“连接健康”前置检查
在 `callMCPTool` 的 `Promise.race` 之前，增加一个快速检查：若 `transport` 已关闭或 `client` 的 `onclose` 已被触发，则立即 reject，避免将请求发送给已死亡的连接。

#### 6.2.4 401 缓存：提供缓存主动失效通道
当前 `clearMcpAuthCache()` 是全局清除。建议：
- 将缓存粒度细化到 `serverId`，允许按 Server 清除。
- 在 `McpAuthTool` 完成 OAuth 后，仅清除该 Server 的缓存条目，而非全部。

#### 6.2.5 大输出持久化：增加权限控制与自动清理
- 在 `persistToolResult` / `persistBinaryContent` 的实现中（`src/utils/mcpOutputStorage.ts`），建议写入文件时显式设置 `mode: 0o600`。
- 增加定期清理任务（如启动时删除超过 7 天的 `mcp-*` 临时文件）。

#### 6.2.6 描述截断：增加语义过滤层
在 `fetchToolsForClient` 的 `description()` / `prompt()` 中，除了长度截断，可增加一层基础过滤：
- 检测并移除常见的注入前缀（如 `Ignore previous instructions`）。
- 对包含大量重复字符或异常编码的描述进行降级处理（如替换为 `"[description truncated due to anomalies]"`）。

#### 6.2.7 `headersHelper`：禁用 `shell: true` 或严格白名单路径
`headersHelper` 的设计目的是执行用户配置的脚本。为降低命令注入面：
- 若脚本路径以常见可执行文件扩展名结尾（`.js`、`.py`、`.sh`），可直接用 `execFile`（`shell: false`）并传入解释器参数。
- 或至少对路径做 `path.normalize` 和禁止包含 `;|&$\`` 等 shell 元字符的校验。

#### 6.2.8 监控与可观测性
- 当前已有很多 `logEvent` 埋点，但缺少**连接池级别的聚合指标**（如各传输类型的平均连接时长、重连频率、工具调用 P99 延迟）。建议在 `connectToServer` 和 `callMCPTool` 中增加更细粒度的 histogram 类事件，便于后续 SLO 监控。

---

## 7. 附录：文件规模与变更敏感度

- **总行数**：~3348 行（项目内最大单体服务文件之一）
- **导出符号数**：20+ 个公开 API
- **变更敏感度**：极高。任何对 `connectToServer`、`callMCPTool`、`fetchToolsForClient` 的修改都会影响所有 MCP Server 的兼容性与稳定性。
- **测试覆盖建议**：该文件目前未找到同目录下的单元测试文件（`src/services/mcp/**/*.test.*` 无匹配）。鉴于其复杂度，强烈建议为核心函数补充单元测试或集成测试，尤其是：
  - `connectToServer` 的各 Transport 分支
  - `callMCPToolWithUrlElicitationRetry` 的 -32042 重试逻辑
  - `wrapFetchWithTimeout` 的内存泄漏防护
  - `processMCPResult` 的大输出截断与持久化分支
