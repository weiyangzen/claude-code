# Research: src/hooks/usePromptsFromClaudeInChrome.tsx

## 场景与职责

`usePromptsFromClaudeInChrome` 是一个用于集成 **Claude for Chrome 浏览器扩展** 的 React Hook。它通过 MCP（Model Context Protocol）与浏览器扩展建立通信，实现两个核心能力：

1. **接收来自 Chrome 扩展的提示（Prompt）**：当用户在浏览器扩展中点击"发送到 Claude Code"时，扩展通过 MCP 的 JSON-RPC 通知将 prompt 发送到本 Hook，Hook 将其加入命令队列，最终作为用户输入提交到 REPL。
2. **同步权限模式到 Chrome 扩展**：当用户在 Claude Code 中切换工具权限模式（如 `ask` / `bypassPermissions`）时，Hook 通过 MCP RPC 将该模式同步给扩展，使扩展侧的自动执行行为与 CLI 保持一致。

该 Hook 在 `REPL.tsx` 中被调用，是 Claude Code 跨端体验（浏览器 ↔ 终端）的关键桥梁。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **MCP 客户端发现** | 在 `mcpClients` 数组中查找名为 `claude-in-chrome` 的已连接 MCP 服务器。 |
| **Prompt 通知监听** | 通过 `useEffect` 订阅 MCP 通知，解析符合 `ClaudeInChromePromptNotificationSchema` 的 JSON-RPC 消息。 |
| **Prompt 入队** | 将接收到的 prompt（可能包含文本 + base64 图片）封装为 `QueuedCommand`，通过 `enqueuePendingNotification` 加入统一命令队列。 |
| **Tab ID 追踪** | 对带有 `tabId` 的通知进行追踪，用于后续去重或关联。 |
| **权限模式同步** | 当 `toolPermissionMode` 变化时，调用 `callIdeRpc("set_permission_mode", ...)` 将模式映射为扩展可识别的格式。 |

## 具体技术实现

### MCP 客户端发现

```ts
function findChromeClient(clients: MCPServerConnection[]): ConnectedMCPServer | undefined {
  return clients.find(
    (client): client is ConnectedMCPServer =>
      client.type === 'connected' && client.name === CLAUDE_IN_CHROME_MCP_SERVER_NAME
  )
}
```

`CLAUDE_IN_CHROME_MCP_SERVER_NAME` 的值为 `'claude-in-chrome'`。

### Prompt 通知 Schema

```ts
const ClaudeInChromePromptNotificationSchema = lazySchema(() => z.object({
  method: z.literal('notifications/message'),
  params: z.object({
    prompt: z.string(),
    image: z.object({
      type: z.literal('base64'),
      media_type: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
      data: z.string()
    }).optional(),
    tabId: z.number().optional()
  })
}))
```

- 使用 `lazySchema` 延迟初始化 Zod schema，避免模块加载时的即时计算开销。
- 支持可选的 base64 图片块和 `tabId`。

### Prompt 入队逻辑（源码中被编译后的 effect）

虽然编译后的代码较抽象，但从上下文和注释可推断出第一个 `useEffect` 的大致行为：

```ts
useEffect(() => {
  const chromeClient = findChromeClient(mcpClients)
  if (!chromeClient) return

  // 设置通知监听器
  // 当收到 notifications/message 时：
  //   1. 验证 schema
  //   2. 若包含 tabId，调用 trackClaudeInChromeTabId(tabId)
  //   3. 构造 ContentBlockParam[]（text + 可选 image）
  //   4. 调用 enqueuePendingNotification({ value: contentBlocks, ... })
  // 返回清理函数
}, [mcpClients])
```

### 权限模式同步

```ts
useEffect(() => {
  const chromeClient = findChromeClient(mcpClients)
  if (!chromeClient) return

  const chromeMode =
    toolPermissionMode === "bypassPermissions"
      ? "skip_all_permission_checks"
      : "ask"

  callIdeRpc("set_permission_mode", { mode: chromeMode }, chromeClient)
}, [mcpClients, toolPermissionMode])
```

- `bypassPermissions` 映射为 `skip_all_permission_checks`（扩展侧自动执行所有工具）。
- 其他所有模式映射为 `ask`（扩展侧每次请求用户确认）。

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/usePromptsFromClaudeInChrome.tsx` | 本 Hook：MCP 通知监听、Prompt 入队、权限同步。 |
| `src/screens/REPL.tsx` | 调用方：传入 `mcpClients` 和 `toolPermissionContext.mode`。 |
| `src/services/mcp/client.ts` | `callIdeRpc` 来源：向 MCP 服务器发送 JSON-RPC 请求的封装。 |
| `src/services/mcp/types.ts` | `ConnectedMCPServer`、`MCPServerConnection` 类型定义。 |
| `src/utils/claudeInChrome/common.ts` | `CLAUDE_IN_CHROME_MCP_SERVER_NAME`、`isTrackedClaudeInChromeTabId`、`trackClaudeInChromeTabId` 等常量与工具函数。 |
| `src/utils/messageQueueManager.ts` | `enqueuePendingNotification` 来源：统一命令队列的入队接口。 |
| `src/utils/lazySchema.js` | `lazySchema` 工具：延迟初始化 Zod schema。 |

## 依赖与外部交互

### 运行时依赖
- **React**：`useEffect`、`useRef`。
- **React Compiler Runtime**：`_c` 缓存优化。
- **Zod v4**：`ClaudeInChromePromptNotificationSchema` 的消息验证。
- **MCP SDK**：通过 `callIdeRpc` 与 `@ant/claude-for-chrome-mcp` 服务器通信。

### 与 MCP 服务器的交互
- Chrome 扩展侧运行一个 in-process MCP 服务器（`@ant/claude-for-chrome-mcp`），通过本地 Unix socket / named pipe 与 Claude Code 通信。
- `callIdeRpc` 发送的是标准 JSON-RPC 2.0 请求，`notifications/message` 则是服务器向客户端推送的通知。

### 与命令队列的交互
- 接收到的 prompt 以 `QueuedCommand` 形式入队，优先级为 `later`（`enqueuePendingNotification` 的默认行为），确保不会打断用户当前正在输入的内容。
- 队列由 `useQueueProcessor.ts` 在合适的时机（无活跃查询、无本地 JSX UI 阻塞）消费并执行。

## 风险、边界与改进建议

### 风险与边界
1. **编译后代码的可读性下降**：该文件经过 React Compiler 编译，原始的第一个 `useEffect`（通知监听）被压缩为 `_temp` 空函数和隐式的 effect 注册，直接阅读源码难以追踪具体的通知处理逻辑。需要结合 source map 或未编译的 TypeScript 源来理解完整行为。
2. **Schema 验证失败静默丢弃**：若扩展发送了格式不符的通知，Zod 解析失败后的行为取决于 effect 内部的错误处理。从现有代码结构看，没有显式的 `.catch` 或 `try/catch` 包裹验证逻辑，可能导致未捕获异常或通知被静默丢弃。
3. **权限模式映射过于粗粒度**：所有非 `bypassPermissions` 模式（包括 `acceptEdits`、`plan` 等）都被映射为 `ask`，扩展侧无法感知更细粒度的权限策略差异。
4. **MCP 客户端断开后的状态丢失**：当 Chrome 扩展关闭或 MCP 连接断开时，`findChromeClient` 返回 `undefined`，此时权限同步 effect 不再执行。若用户在此期间切换了权限模式，重新连接后不会自动补发同步请求。
5. **图片大小无上限校验**：通知 schema 仅验证了 `data` 是字符串，未限制 base64 图片的大小。若扩展发送了超大图片，可能超出 API 限制或导致队列处理卡顿。

### 改进建议
1. **恢复未编译源码的可维护性**：考虑将 React Compiler 编译后的文件与源码分离（如 build 产物放入 `dist/`），或在仓库中保留原始 `.tsx` 作为 primary source，便于后续研究和调试。
2. **增加通知解析失败的日志与容错**：在 schema 验证处增加 `try/catch` 和 `logError`，对格式错误的消息记录扩展版本和原始 payload，便于排查兼容性问题。
3. **细化权限模式映射**：与扩展团队协商，将 `acceptEdits`、`plan` 等模式也映射到扩展侧对应的语义，避免用户在不同端看到不一致的行为。
4. **连接恢复后的状态同步**：在 `findChromeClient` 从 `undefined` 变为有效客户端时，主动触发一次权限模式同步，确保重连后状态一致。
5. **图片预处理与大小限制**：在入队前对 base64 图片进行大小检查，必要时调用 `maybeResizeAndDownsampleImageBuffer` 压缩，或拒绝超大图片并给出用户提示。
