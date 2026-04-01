# mcpServer.ts 深度研究文档

## 场景与职责

`mcpServer.ts` 是 **Claude in Chrome** 功能的 **MCP 服务器入口与上下文工厂**。它承担两个核心角色：

1. **子进程模式入口**：通过 `runClaudeInChromeMcpServer()` 在 `--claude-in-chrome-mcp` 子进程中启动一个基于 `stdio` 传输的 MCP 服务器。
2. **同进程上下文工厂**：通过 `createChromeContext()` 为 `src/services/mcp/client.ts` 提供 `ClaudeForChromeContext`，支持将 MCP 服务器以内联（in-process）方式运行，避免每次连接都产生 ~325MB 的子进程开销。

此外，该文件还负责：
- 桥接（Bridge）模式与原生 socket 模式的动态切换
- OAuth 认证与设备配对持久化
- 浏览器自动化事件的受控匿名化分析上报
- **Ant 内部用户**的 `browser_task` 闪电模式代理循环（`callAnthropicMessages`）

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getChromeBridgeUrl()` | 根据环境变量和 GrowthBook feature flag 决定是否使用 WebSocket 桥接，以及桥接的端点 URL。 |
| `createChromeContext(env?)` | 构建供 `@ant/claude-for-chrome-mcp` 使用的完整上下文对象，包含 logger、socket 路径、认证回调、事件追踪、桥接配置等。 |
| `runClaudeInChromeMcpServer()` | 子进程入口：初始化配置、分析、创建上下文、连接 stdio transport、在 stdin 关闭时优雅退出并 flush 分析事件。 |
| `DebugLogger` | 将 `@ant/claude-for-chrome-mcp` 的日志桥接到 Claude Code 内部的 `logForDebugging` 系统。 |

## 具体技术实现

### 1. Bridge 模式判定 (`getChromeBridgeUrl()`)

启用条件：
- `process.env.USER_TYPE === 'ant'` **或**
- GrowthBook flag `tengu_copper_bridge` 为 `true`

URL 优先级：
1. `ws://localhost:8765`（`USE_LOCAL_OAUTH` 或 `LOCAL_BRIDGE` 为真）
2. `wss://bridge-staging.claudeusercontent.com`（`USE_STAGING_OAUTH` 为真）
3. `wss://bridge.claudeusercontent.com`（生产环境）

若 Bridge 未启用，则 `socketPath` / `getSocketPaths` 走原生 Unix socket / named pipe 模式。

### 2. ClaudeForChromeContext 构建 (`createChromeContext()`)

上下文对象字段解析：

| 字段 | 说明 |
|------|------|
| `serverName` | 固定为 `'Claude in Chrome'` |
| `logger` | `DebugLogger` 实例，桥接 mcp 包的日志级别 |
| `socketPath` | 当前进程的安全 socket 路径（`common.ts`） |
| `getSocketPaths` | 返回所有可能的 socket 路径（含 legacy fallback） |
| `clientTypeId` | `'claude-code'` |
| `onAuthenticationError` | 提示用户确保浏览器扩展与 Claude Code 使用同一 claude.ai 账号登录 |
| `onToolCallDisconnected` | 返回用户友好的重连/安装指导文本（含 `BUG_REPORT_URL`） |
| `onExtensionPaired` | 将设备 ID 与名称持久化到 `~/.claude.json` 的 `chromeExtension` 字段 |
| `getPersistedDeviceId` | 从全局配置读取已配对设备 ID |
| `bridgeConfig`（可选） | 包含桥接 URL、`getUserId`（OAuth account UUID）、`getOAuthToken`（access token） |
| `initialPermissionMode`（可选） | 从 `CLAUDE_CHROME_PERMISSION_MODE` 环境变量读取，`ask` / `skip_all_permission_checks` / `follow_a_plan` |
| `callAnthropicMessages`（Ant-only） | 为 `browser_task` 工具提供后台 LLM 调用能力 |
| `trackEvent` | 将 Chrome MCP 事件安全地转发到 Datadog/1P 分析系统 |

### 3. 权限模式

```typescript
const PERMISSION_MODES: readonly PermissionMode[] = [
  'ask',
  'skip_all_permission_checks',
  'follow_a_plan',
]
```

- 无效值会被记录 warn 日志并忽略。
- `skip_all_permission_checks` 在 `src/main.tsx` 中通过 `getSessionBypassPermissionsMode()` 注入到 `env.CLAUDE_CHROME_PERMISSION_MODE`。

### 4. Ant-only 闪电模式 (`callAnthropicMessages`)

该字段仅在 `USER_TYPE === 'ant'` 时存在，为 `browser_task` 工具提供在 Node 端运行的轻量级代理循环：

- 调用 `sideQuery()`（`src/utils/sideQuery.ts`）向 Anthropic API 发送请求。
- 参数特点：
  - `skipSystemPromptPrefix: true` — 避免 CLI 默认系统提示前缀稀释闪电模式的 batching 指令。
  - `tools: []` — **负载关键（load-bearing）**，若不加，Sonnet 会提前输出 `<function_calls>` XML。
  - `querySource: 'chrome_mcp'` — 用于分析归因。
- 返回结果过滤为纯 `text` 类型的 content blocks，并带上 token usage。

> 注释提到该功能依赖 `@ant/claude-for-chrome-mcp@0.4.0`，但 CI 安装的是 `0.3.0`。由于 TypeScript 结构类型匹配，额外字段在 spread 中不会导致 0.3.0 报错。

### 5. 安全的事件追踪 (`trackEvent`)

为避免用户页面内容或错误消息泄露到分析系统，实现了**严格的字段白名单**：

- 只允许 `boolean` 和 `number` 类型的元数据无条件通过。
- `string` 类型仅当 key 在 `SAFE_BRIDGE_STRING_KEYS` 中才通过：
  - `bridge_status`（原 `status` 被重命名，避开 Datadog 保留字段）
  - `error_type`
  - `tool_name`
- `error_message` 等可能包含用户数据的字符串被**显式丢弃**。

### 6. 子进程生命周期 (`runClaudeInChromeMcpServer()`)

```typescript
process.stdin.on('end', () => void shutdownAndExit())
process.stdin.on('error', () => void shutdownAndExit())
```

- 父进程（Claude Code）退出时 stdin pipe 关闭，触发 `shutdownAndExit()`。
- 退出前调用 `shutdown1PEventLogging()` 和 `shutdownDatadog()`，确保最后一批事件（如 disconnect）被 flush。
- 使用 `exiting` 标志防止重复退出。

## 关键代码路径与文件引用

```
src/entrypoints/cli.tsx:72-78
  └── --claude-in-chrome-mcp 分支
      └── import('../utils/claudeInChrome/mcpServer.js')
          └── runClaudeInChromeMcpServer()
              ├── enableConfigs()                 ← src/utils/config.js
              ├── initializeAnalyticsSink()       ← src/services/analytics/sink.js
              ├── createChromeContext()
              │   ├── getSecureSocketPath()       ← common.ts
              │   ├── getAllSocketPaths()         ← common.ts
              │   ├── getClaudeAIOAuthTokens()    ← src/utils/auth.js
              │   ├── getGlobalConfig()           ← src/utils/config.js
              │   ├── saveGlobalConfig()          ← src/utils/config.js
              │   ├── sideQuery()                 ← src/utils/sideQuery.ts
              │   └── logEvent()                  ← src/services/analytics/index.js
              └── createClaudeForChromeMcpServer(context)
                  ← @ant/claude-for-chrome-mcp
                  └── StdioServerTransport
                      ← @modelcontextprotocol/sdk
```

### 同进程路径（in-process）

```
src/services/mcp/client.ts:905-924
  └── isClaudeInChromeMCPServer(name)
      └── import('../../utils/claudeInChrome/mcpServer.js')
          ├── createChromeContext(serverRef.env)
          └── createClaudeForChromeMcpServer(context)
              └── createLinkedTransportPair()     ← InProcessTransport.ts
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `@ant/claude-for-chrome-mcp` | `createClaudeForChromeMcpServer`, `ClaudeForChromeContext`, `Logger`, `PermissionMode` |
| `@modelcontextprotocol/sdk/server/stdio.js` | `StdioServerTransport` |
| `util` | `format()` 用于日志格式化 |
| `src/services/analytics/*` | Datadog、1P 事件、GrowthBook feature flags |
| `src/utils/auth.js` | OAuth token 获取 |
| `src/utils/config.js` | 全局配置的读写 |
| `src/utils/debug.js` | `logForDebugging` |
| `src/utils/envUtils.js` | `isEnvTruthy` |
| `src/utils/sideQuery.js` | Ant-only 闪电模式代理的 LLM 调用 |
| `./common.js` | Socket 路径相关函数 |

### 环境变量

- `USER_TYPE`：决定 Bridge 是否强制启用、是否暴露 dev extension IDs、是否启用 `callAnthropicMessages`。
- `USE_LOCAL_OAUTH` / `LOCAL_BRIDGE` / `USE_STAGING_OAUTH`：Bridge URL 覆盖。
- `CLAUDE_CHROME_PERMISSION_MODE`：初始权限模式。

## 风险、边界与改进建议

### 风险

1. **Bridge 与 Socket 的竞态**：当 Bridge feature flag 动态变化时，已缓存的 `getChromeBridgeUrl()` 结果可能在同一会话内不一致。注释已标注 `CACHED_MAY_BE_STALE`。
2. **`callAnthropicMessages` 的类型脆弱性**：依赖未发布的 `@ant/claude-for-chrome-mcp@0.4.0` 类型，当前通过内联类型和 spread 绕过编译检查。一旦包接口发生不兼容变更，可能在运行时暴露问题。
3. **OAuth Token 过期**：`getOAuthToken` 返回的是内存中缓存的 access token，若 token 在会话期间过期，桥接模式可能遭遇认证失败，依赖 `onAuthenticationError` 回调做用户提示，但无自动刷新逻辑。
4. **`sideQuery` 的同步阻塞**：`callAnthropicMessages` 是 async 函数，但在 `browser_task` 的代理循环中可能被频繁调用，若网络延迟高会影响浏览器自动化响应速度。

### 边界

- `runClaudeInChromeMcpServer()` 仅作为子进程入口使用，不能在同一进程中多次调用（`process.stdin` 是全局单例）。
- `trackEvent` 的字符串白名单非常严格，这意味着大量对调试有用的字符串上下文被丢弃，问题排查时信息可能不足。
- Ant-only 的 `callAnthropicMessages` 在 public build 中通过 `process.env.USER_TYPE === 'ant'` 运行时判断存在，但 build-time 是否完全 DCE 取决于打包器能力。

### 改进建议

1. **Bridge URL 缓存刷新机制**：在 `createChromeContext` 被调用时重新计算 Bridge URL，而不是依赖可能过期的全局缓存。
2. **OAuth Token 自动刷新**：在 `bridgeConfig.getOAuthToken` 中集成 token 刷新逻辑（或至少检查过期时间），减少桥接断连。
3. **结构化错误追踪**：在 `trackEvent` 中允许经过哈希/截断处理的错误消息，在保护隐私的同时保留更多调试信号。
4. **`callAnthropicMessages` 的模型与参数配置化**：当前 `skipSystemPromptPrefix` 和 `tools: []` 是硬编码经验值，可考虑暴露为环境变量或配置项，便于 A/B 实验。
5. **增加子进程异常处理**：`runClaudeInChromeMcpServer` 中未捕获 `server.connect()` 的异常，建议增加 `try/catch` 并输出结构化错误日志。
