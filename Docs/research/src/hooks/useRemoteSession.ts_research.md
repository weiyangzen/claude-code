# Research: src/hooks/useRemoteSession.ts

## 场景与职责

`useRemoteSession` 是 Claude Code REPL 中用于管理**远程 CCR（Claude Code Remote）会话**的核心 Hook。当用户通过 `--remote` 或 `--teleport` 模式启动时，本地 REPL 不再直接调用 Anthropic API，而是作为"查看器（viewer）"连接到一个运行在云端或远程容器中的 CCR 会话。

该 Hook 承担以下关键职责：
1. **WebSocket 连接管理**：建立并维护与远程会话的 WebSocket 订阅连接。
2. **消息协议转换**：将 CCR 后端发送的 SDK 格式消息转换为 REPL 内部可渲染的 `Message` 类型。
3. **用户输入转发**：通过 HTTP POST 将用户输入发送到远程会话。
4. **权限请求桥接**：将远程会话发来的工具权限请求映射到本地已有的 `ToolUseConfirm` 队列和 UI 流程。
5. **超时与重连控制**：检测会话无响应（60s 常规 / 3min compaction），自动触发重连；跟踪 compaction、后台任务、工具使用等状态。
6. **回声过滤**：防止本地已添加的用户消息在 WebSocket 回传时被重复渲染。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **WebSocket 生命周期** | 通过 `RemoteSessionManager` 建立 WS 连接，处理 `onConnected` / `onReconnecting` / `onDisconnected` / `onError` 事件。 |
| **SDK 消息适配** | 使用 `convertSDKMessage` 将 `SDKMessage` 转为 `Message` / `StreamEvent` / `ignored`，支持 `viewerOnly` 模式下的特殊转换规则。 |
| **回声过滤（Echo Deduplication）** | 使用 `BoundedUUIDSet(50)` 记录本地已发送消息的 UUID，丢弃 WebSocket 回传的相同 UUID 消息，避免重复显示。 |
| **后台任务计数** | 监听 `task_started` / `task_notification` 系统消息，维护 `remoteBackgroundTaskCount`，在底部状态栏显示"N in background"。 |
| **Compaction 感知** | 跟踪 `status='compacting'` 和 `compact_boundary` 消息，在 compaction 期间使用 180s 超长超时，避免误报无响应。 |
| **权限请求桥接** | 将远程 `can_use_tool` 控制请求转换为本地 `ToolUseConfirm` 对象，用户允许/拒绝后通过 HTTP 回传响应。 |
| **会话标题更新** | 对非 `viewerOnly` 且没有初始 prompt 的会话，在首次用户消息后自动生成并更新会话标题。 |
| **工具使用进度跟踪** | 在远程 assistant 消息中识别 `tool_use` 块并加入 `inProgressToolUseIDs`；在 `tool_result` 到达时移除，保证 Spinner 状态正确。 |

## 具体技术实现

### 初始化与连接

```ts
useEffect(() => {
  if (!config) return

  const manager = new RemoteSessionManager(config, { onMessage, onPermissionRequest, ... })
  managerRef.current = manager
  manager.connect()

  return () => {
    if (responseTimeoutRef.current) clearTimeout(...)
    manager.disconnect()
    managerRef.current = null
  }
}, [config, setMessages, setIsLoading, ...])
```

- `config` 为 `RemoteSessionConfig | undefined`，缺失时直接跳过，表示非远程模式。
- Effect 的清理函数会断开 WS 并清除超时定时器。

### 回声过滤机制

```ts
const sentUUIDsRef = useRef(new BoundedUUIDSet(50))
```

在 `sendMessage` 中：
```ts
if (opts?.uuid) sentUUIDsRef.current.add(opts.uuid)
```

在 `onMessage` 中：
```ts
if (
  sdkMessage.type === 'user' &&
  sdkMessage.uuid &&
  sentUUIDsRef.current.has(sdkMessage.uuid)
) {
  return // 丢弃回声
}
```

- 使用 `BoundedUUIDSet` 而非普通 `Set` 的原因是：同一个 POST 可能被服务器广播一次、又被 worker 回传一次，即同一 UUID 可能回声**多次**。若用 `Set.delete()` 会在第一次匹配后放行第二次回声；环形缓冲区（ring）则通过容量上限自然淘汰旧 UUID，同时保留对多次回声的过滤能力。

### 超时与重连

```ts
const RESPONSE_TIMEOUT_MS = 60000
const COMPACTION_TIMEOUT_MS = 180000
```

在 `sendMessage` 成功后启动定时器：
```ts
if (!config?.viewerOnly) {
  const timeoutMs = isCompactingRef.current ? COMPACTION_TIMEOUT_MS : RESPONSE_TIMEOUT_MS
  responseTimeoutRef.current = setTimeout((setMessages, manager) => {
    const warningMessage = createSystemMessage('Remote session may be unresponsive. Attempting to reconnect…', 'warning')
    setMessages(prev => [...prev, warningMessage])
    manager.reconnect()
  }, timeoutMs, setMessages, manager)
}
```

- 任何 WS 消息到达时都会先清除超时（包括回声消息，因此回声本身也起到心跳作用）。
- `viewerOnly` 模式禁用超时，因为远端 agent 可能处于 idle-shut 状态，唤醒时间可能超过 60s。

### 权限请求桥接

```ts
onPermissionRequest: (request, requestId) => {
  const tool = findToolByName(toolsRef.current, request.tool_name) ?? createToolStub(request.tool_name)
  const syntheticMessage = createSyntheticAssistantMessage(request, requestId)

  const toolUseConfirm: ToolUseConfirm = {
    assistantMessage: syntheticMessage,
    tool,
    description: request.description ?? `${request.tool_name} requires permission`,
    input: request.input,
    toolUseContext: {} as ToolUseConfirm['toolUseContext'],
    toolUseID: request.tool_use_id,
    permissionResult: { behavior: 'ask', message: ..., suggestions: ... },
    onAllow(updatedInput) {
      manager.respondToPermissionRequest(requestId, { behavior: 'allow', updatedInput })
      setToolUseConfirmQueue(queue => queue.filter(...))
      setIsLoading(true)
    },
    onReject(feedback) {
      manager.respondToPermissionRequest(requestId, { behavior: 'deny', message: feedback })
      setToolUseConfirmQueue(queue => queue.filter(...))
    },
    onAbort() { ... },
    onUserInteraction() { /* no-op for remote */ },
    async recheckPermission() { /* no-op for remote */ },
  }

  setToolUseConfirmQueue(queue => [...queue, toolUseConfirm])
  setIsLoading(false) // 暂停加载指示器，等待用户决策
}
```

### 会话标题更新

```ts
if (
  !hasUpdatedTitleRef.current &&
  config &&
  !config.hasInitialPrompt &&
  !config.viewerOnly
) {
  hasUpdatedTitleRef.current = true
  const description = typeof content === 'string' ? content : extractTextContent(content, ' ')
  void generateSessionTitle(description, new AbortController().signal)
    .then(title => void updateSessionTitle(sessionId, title ?? truncateToWidth(description, 75)))
}
```

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/useRemoteSession.ts` | 本 Hook：远程会话的完整生命周期管理。 |
| `src/screens/REPL.tsx` | 主要调用方：传入 `config`（来自 `remoteSessionConfig` prop）和所有状态 setter。 |
| `src/hooks/useDirectConnect.ts` | 次要调用方：直接连接模式也复用该 Hook。 |
| `src/remote/RemoteSessionManager.ts` | `RemoteSessionManager` 类：封装 WebSocket 连接、HTTP POST、控制消息处理。 |
| `src/remote/sdkMessageAdapter.ts` | `convertSDKMessage`、`isSessionEndMessage`：SDK 消息到 REPL 消息的转换器。 |
| `src/remote/remotePermissionBridge.ts` | `createSyntheticAssistantMessage`、`createToolStub`：权限桥接辅助函数。 |
| `src/bridge/bridgeMessaging.ts` | `BoundedUUIDSet`：有界 UUID 环形集合，用于回声去重。 |
| `src/utils/messages.ts` | `createSystemMessage`、`extractTextContent`、`handleMessageFromStream` 等消息工厂函数。 |
| `src/utils/sessionTitle.ts` | `generateSessionTitle`：基于用户输入生成会话标题。 |
| `src/utils/teleport/api.ts` | `updateSessionTitle`、`sendEventToRemoteSession`：远程 API 调用。 |
| `src/state/AppState.js` | `useSetAppState`：更新 `remoteConnectionStatus`、`remoteBackgroundTaskCount`。 |

## 依赖与外部交互

### 运行时依赖
- **React**：`useCallback`、`useEffect`、`useMemo`、`useRef`。
- **RemoteSessionManager**：管理 WebSocket（`SessionsWebSocket.ts`）和 HTTP POST（`teleport/api.ts`）。
- **工具查找**：`findToolByName` 来自 `src/Tool.js`，用于将远程权限请求中的工具名映射到本地 Tool 对象。

### 与远程后端的协议
- **WebSocket**：订阅会话事件流，接收 `SDKMessage` 和 `SDKControlRequest`。
- **HTTP POST**：`sendEventToRemoteSession` 将用户消息发送到远程会话的 REST API。
- **控制消息**：`control_request`（权限询问）、`control_response`（ack）、`control_cancel_request`（取消权限询问）。

### 与本地 UI 的集成
- `setMessages`：将转换后的消息追加到本地消息列表。
- `setToolUseConfirmQueue`：将远程权限请求加入本地权限确认队列，复用与本地会话完全相同的 `PermissionRequest` UI。
- `setIsLoading`：控制 Spinner 的显示/隐藏，在权限等待期间暂停加载指示。
- `setStreamingToolUses` / `setStreamMode`：处理远程流式事件，更新实时 UI。

## 风险、边界与改进建议

### 风险与边界
1. **回声过滤不覆盖历史重叠**：注释明确说明 `BoundedUUIDSet` "does NOT dedup history-vs-live overlap at attach time"，即如果用户重新 attach 到一个已有历史的远程会话，历史中的用户消息 UUID 不会被预填充到集合中，理论上可能重复。不过实际场景中 attach 时的历史消息通常通过其他路径加载。
2. **`viewerOnly` 与 `!viewerOnly` 的行为分叉**：代码中存在大量 `config.viewerOnly` 条件分支（禁用超时、禁用中断、不同的消息转换选项）。随着功能演进，这两种模式的分支可能继续增加，维护复杂度上升。
3. **WebSocket 间隙期间的状态漂移**：`onReconnecting` 中会清空 `runningTaskIdsRef` 和 `inProgressToolUseIDs`，因为间隙期间可能丢失 `task_notification` 或 `tool_result`。这会导致重连后短暂低估后台任务数或工具状态，但这是"可接受的低估"（注释明确说明 undercounts are accepted）。
4. **`toolsRef` 的同步更新**：通过 `useEffect(() => { toolsRef.current = tools }, [tools])` 保证 WebSocket 回调中读取到最新工具列表，但若工具列表在回调执行间隙发生变化，仍可能有一帧的延迟。
5. **权限请求的 `recheckPermission` 为空操作**：远程模式下权限状态在远端容器上，本地无法重新检查。这意味着如果用户通过其他途径（如 Web 界面）修改了权限，本地 UI 不会自动同步更新。
6. **`generateSessionTitle` 的 AbortController 未被清理**：`new AbortController().signal` 创建的 controller 没有保存引用，无法在外部取消。虽然 `generateSessionTitle` 内部有容错，但在组件卸载时仍可能产生悬空 promise。

### 改进建议
1. **抽象 viewer / full-remote 两种模式**：当前大量 `if (config?.viewerOnly)` 分支可考虑拆分为两个策略对象（Strategy Pattern），如 `ViewerRemoteSessionStrategy` 和 `FullRemoteSessionStrategy`，减少单文件中的条件复杂度。
2. **预填充历史 UUID 到 BoundedUUIDSet**：在 attach 远程会话时，将历史用户消息的 UUID 批量加入 `sentUUIDsRef`，彻底消除历史-实时重叠导致的重复消息风险。
3. **增加重连后的状态同步握手**：在 `onConnected` 后向远程会话请求一次当前任务列表和工具使用状态的快照，弥补 WS 间隙期间的信息丢失。
4. **统一 AbortController 管理**：对 `generateSessionTitle` 等异步副作用使用 ref 保存 AbortController，在 effect 清理或 disconnect 时统一 abort，避免内存泄漏和竞态写入。
5. **远程权限模式的双向同步**：考虑在 WebSocket 控制通道中增加权限状态推送，使本地 UI 能感知远端权限变化，从而支持 `recheckPermission` 的非空实现。
