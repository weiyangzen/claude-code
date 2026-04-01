# useDirectConnect.ts 研究文档

## 场景与职责

`useDirectConnect` 是 Claude Code 中支持 **`claude connect` 直连模式** 的核心 Hook。在该模式下，本地 REPL 不通过常规的 CCR（Claude Code Runtime）后端，而是直接通过 WebSocket 连接到一个远程的 Claude 服务器实例。这个 Hook 负责：

- 建立并维护 WebSocket 连接（通过 `DirectConnectSessionManager`）
- 将远程服务器发来的 SDK 格式消息转换为本地 REPL 可渲染的 `Message` 类型
- 将远程的权限请求（tool use permission）桥接到本地的 `ToolUseConfirm` 队列
- 提供发送消息、取消请求、断开连接等操作接口

它是 `REPL.tsx` 中三种特殊会话模式之一（另外两种是 `remoteSessionConfig` 的 CCR 远程模式和 `sshSession` 的 SSH 模式）。

## 功能点目的

1. **WebSocket 会话生命周期管理**：根据传入的 `DirectConnectConfig` 创建 `DirectConnectSessionManager`，在 `useEffect` 中连接，在 cleanup 中断开。

2. **消息双向桥接**：
   - **下行**：将服务器发来的 SDK 消息（`SDKMessage`）通过 `convertSDKMessage` 转换为本地 `Message`，追加到 `messages` 状态。
   - **上行**：将用户输入（`RemoteMessageContent`）封装为 SDK `user` 消息格式发送给服务器。

3. **权限请求本地化处理**：远程服务器上的 tool use 需要权限时，服务器会发送 `control_request`（subtype `can_use_tool`）。Hook 将其转换为合成的 `AssistantMessage` 和 `ToolUseConfirm`，放入本地的权限确认队列，让用户在本地终端中做 allow/deny 决策，再通过 WebSocket 将决策回传给服务器。

4. **连接状态与错误处理**：
   - `onConnected` / `onDisconnected`：记录连接状态，断开时触发 `gracefulShutdown(1)`
   - `onError`：记录调试日志
   - 从未连接就断开（如认证失败）与连接后断开做了区分提示

5. **中断支持**：`cancelRequest` 通过发送 `control_request`（subtype `interrupt`）来取消远程当前请求。

## 具体技术实现

### 关键流程

#### 1. 初始化与连接建立
```ts
useEffect(() => {
  if (!config) return
  const manager = new DirectConnectSessionManager(config, callbacks)
  managerRef.current = manager
  manager.connect()
  return () => {
    manager.disconnect()
    managerRef.current = null
  }
}, [config, setMessages, setIsLoading, setToolUseConfirmQueue])
```
- `config` 来自 `REPL.tsx` 的 `directConnectConfig` prop
- `hasReceivedInitRef` 用于去重 `system/init` 消息（服务器每 turn 都会发一次）

#### 2. 消息接收与过滤（`DirectConnectSessionManager`）
`DirectConnectSessionManager`（`src/server/directConnectManager.ts`）的 `onMessage` 回调处理 WebSocket 收到的每一行 NDJSON：
- 跳过 `control_response`、`keep_alive`、`control_cancel_request`、`streamlined_text`、`streamlined_tool_use_summary`
- 跳过 `system/post_turn_summary`
- 其他消息通过 `convertSDKMessage` 转换

在 `useDirectConnect` 中：
```ts
onMessage: sdkMessage => {
  if (isSessionEndMessage(sdkMessage)) setIsLoading(false)
  if (sdkMessage.type === 'system' && sdkMessage.subtype === 'init') {
    if (hasReceivedInitRef.current) return
    hasReceivedInitRef.current = true
  }
  const converted = convertSDKMessage(sdkMessage, { convertToolResults: true })
  if (converted.type === 'message') {
    setMessages(prev => [...prev, converted.message])
  }
}
```

#### 3. 权限请求桥接
当收到 `control_request` / `can_use_tool` 时：
- 用 `createSyntheticAssistantMessage`（`src/remote/remotePermissionBridge.ts`）构造一个假的 `AssistantMessage`
- 用 `createToolStub` 为未知工具生成最小化的 `Tool` 对象（因为远程服务器可能有本地没有的 MCP 工具）
- 构造 `PermissionAskDecision`（behavior: 'ask'）
- 组装成 `ToolUseConfirm` 对象，其中：
  - `onAllow` → 发送 `control_response`（behavior: 'allow'，携带 `updatedInput`）
  - `onReject` → 发送 `control_response`（behavior: 'deny'，携带反馈消息）
  - `onAbort` → 发送 `control_response`（behavior: 'deny'，message: 'User aborted'）

#### 4. 发送消息
```ts
const sendMessage = useCallback(async (content: RemoteMessageContent): Promise<boolean> => {
  const manager = managerRef.current
  if (!manager) return false
  setIsLoading(true)
  return manager.sendMessage(content)
}, [setIsLoading])
```
- `sendMessage` 将内容序列化为 SDK `user` 消息格式：
```json
{
  "type": "user",
  "message": { "role": "user", "content": content },
  "parent_tool_use_id": null,
  "session_id": ""
}
```

### 数据结构

- **DirectConnectConfig**（来自 `src/server/directConnectManager.ts`）：
  - `serverUrl: string`
  - `sessionId: string`
  - `wsUrl: string`
  - `authToken?: string`

- **UseDirectConnectResult**：
  - `isRemoteMode: boolean`（即 `!!config`）
  - `sendMessage(content): Promise<boolean>`
  - `cancelRequest(): void`
  - `disconnect(): void`

- **内部 Refs**：
  - `managerRef`：持有 `DirectConnectSessionManager` 实例
  - `hasReceivedInitRef`：防止重复处理 `system/init`
  - `isConnectedRef`：区分“从未连接”和“连接后断开”
  - `toolsRef`：保存最新 tools 列表，避免 WebSocket 回调闭包 stale

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useDirectConnect.ts` | 本 Hook 实现 |
| `src/screens/REPL.tsx` | 主要调用方，集成到 REPL 主循环 |
| `src/hooks/useSSHSession.ts` | 另一个调用方（SSH 模式可能也复用或参考） |
| `src/server/directConnectManager.ts` | `DirectConnectSessionManager`：WebSocket 连接、消息序列化/反序列化、控制请求处理 |
| `src/remote/sdkMessageAdapter.ts` | `convertSDKMessage`、`isSessionEndMessage`：SDK 消息格式 → 本地 Message 格式 |
| `src/remote/remotePermissionBridge.ts` | `createSyntheticAssistantMessage`、`createToolStub`：远程权限请求的本地桥接 |
| `src/remote/RemoteSessionManager.ts` | `RemotePermissionResponse` 类型定义 |
| `src/utils/teleport/api.ts` | `RemoteMessageContent` 类型 |
| `src/utils/gracefulShutdown.ts` | 断开连接时调用 `gracefulShutdown(1)` |
| `src/Tool.ts` | `Tool` 类型、`findToolByName` |
| `src/types/message.ts` | `Message`、`AssistantMessage` 等类型 |
| `src/types/permissions.ts` | `PermissionAskDecision` 类型 |

## 依赖与外部交互

### 内部依赖
- **React**：`useCallback`、`useEffect`、`useMemo`、`useRef`
- **WebSocket**：由 `DirectConnectSessionManager` 使用原生 `WebSocket`（Bun 环境支持 headers）

### 外部交互
- **远程 Claude 服务器**：通过 WebSocket 连接，协议为自定义 NDJSON 流，包含：
  - 普通 SDK 消息（assistant、user、system、tool_progress 等）
  - 控制消息（`control_request` / `control_response`）
- **认证**：若 `config.authToken` 存在，通过 WebSocket `headers.authorization: Bearer <token>` 发送

### 与本地权限系统的交互
- 将远程权限请求映射为本地 `ToolUseConfirm`，复用本地终端的权限确认 UI（`PermissionRequest` 组件体系）
- 用户决策后通过 WebSocket 回传，远程服务器继续执行

## 风险、边界与改进建议

### 风险与边界

1. **工具缺失时的降级处理**：当远程服务器请求一个本地不存在的工具时，`findToolByName` 返回 `undefined`，此时用 `createToolStub` 生成一个最小化 `Tool`。这个 stub 的 `renderToolUseMessage` 只能展示前 3 个输入字段，用户体验较差，且 `description` 为空，可能导致权限对话框信息不足。

2. **`toolUseContext` 为空对象**：在构造 `ToolUseConfirm` 时，`toolUseContext` 被硬编码为 `{} as ToolUseConfirm['toolUseContext']`。这意味着任何依赖 `toolUseContext` 的权限 UI 行为（如 diff 支持、特定上下文提示）在直连模式下都会失效。

3. **`recheckPermission` 和 `onUserInteraction` 为 No-op**：直连模式下不支持重新检查权限，也不支持用户交互过程中的实时回调。如果未来本地权限系统增强这些能力，直连模式会落后。

4. **WebSocket 错误处理粗糙**：`onError` 仅记录调试日志，没有向用户展示连接错误的具体原因（如 401 认证失败 vs 网络不可达）。`onDisconnected` 虽然区分了“从未连接”和“断开”，但信息仅通过 `process.stderr.write` 输出，没有进入通知系统。

5. **`sendMessage` 的返回值语义不清**：返回 `boolean` 表示 WebSocket 是否处于 `OPEN` 状态，但调用方（如 `REPL.tsx`）通常不处理 `false`，可能导致用户输入静默丢失。

6. **`system/init` 去重依赖消息内容**：`hasReceivedInitRef` 是进程级/session 级的，如果服务器行为变更（如不再每 turn 发送 init），这个逻辑无害；但如果 init 消息确实需要在某些场景下重复（如重连后），会被错误过滤。

### 改进建议

1. **增强工具 stub 的信息展示**：考虑在 `control_request` 中携带更完整的工具描述信息，或让远程服务器在请求中附带 `tool_description`，填充到 `createToolStub` 中。

2. **传递真实的 `toolUseContext`**：如果远程服务器能提供 `mcpClients` 或文件路径上下文，应尝试填充 `toolUseContext`，使 IDE diff 等功能在直连模式下也能工作。

3. **连接错误分类与重试**：为 `DirectConnectSessionManager` 增加错误码解析（如 HTTP 401/403/500、DNS 失败、TLS 错误），并向用户展示可操作的提示（如“认证失败，请检查 token”）。

4. **`sendMessage` 失败时回退**：当 `sendMessage` 返回 `false` 时，应在 UI 层给出反馈（如通知“消息发送失败，连接已断开”），而不是静默丢弃。

5. **支持 `recheckPermission`**：如果远程协议允许，可以设计一个 `control_request`（subtype `recheck_permission`）来支持权限重检查，使直连模式与本地模式功能对齐。

6. **心跳与断线重连**：当前实现没有自动重连机制。对于长时间运行的会话，建议增加指数退避重连和心跳检测，提升稳定性。
