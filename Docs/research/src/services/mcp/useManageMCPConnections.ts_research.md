# useManageMCPConnections.ts 深度研究文档

## 场景与职责

`useManageMCPConnections.ts` 是 Claude Code 中 MCP (Model Context Protocol) 服务器连接管理的核心 React Hook。它负责：

1. **MCP 服务器生命周期管理**：初始化、连接、断开、重连
2. **状态同步**：将 MCP 客户端状态同步到 AppState
3. **自动重连机制**：为远程传输（SSE/HTTP/WebSocket）实现指数退避重连
4. **频道通知系统**（KAIROS/KAIROS_CHANNELS）：处理来自 MCP 服务器的频道消息推送
5. **列表变更通知**：处理 tools/prompts/resources 的 list_changed 通知
6. **服务器启用/禁用切换**：支持用户动态启用或禁用 MCP 服务器

该 Hook 在 `MCPConnectionManager.tsx` 中被调用，为整个应用提供 MCP 连接管理能力。

## 功能点目的

### 1. 批量状态更新系统

**目的**：避免频繁的 setAppState 调用导致的性能问题

**实现**：
- 使用 `pendingUpdatesRef` 收集待更新状态
- 16ms 时间窗口（`MCP_BATCH_FLUSH_MS`）合并多个更新
- 通过 `flushPendingUpdates` 一次性刷新到 AppState

### 2. 自动重连机制

**目的**：远程 MCP 服务器（SSE/HTTP/WebSocket）在网络波动后自动恢复连接

**参数**：
- `MAX_RECONNECT_ATTEMPTS = 5`：最大重试次数
- `INITIAL_BACKOFF_MS = 1000`：初始退避时间
- `MAX_BACKOFF_MS = 30000`：最大退避时间

**流程**：
1. 检测到连接断开（`onclose` 触发）
2. 检查服务器是否被禁用（`isMcpServerDisabled`）
3. 跳过 stdio/sdk 类型（不支持重连）
4. 指数退避重试，直到成功或达到最大次数

### 3. 频道通知系统（KAIROS）

**目的**：允许 MCP 服务器向对话推送消息（如 Discord/Slack 消息）

**门控检查**（`gateChannelServer`）：
- 能力检查：`capabilities.experimental['claude/channel']`
- 功能开关：`isChannelsEnabled()`
- 认证检查：需要 claude.ai OAuth
- 组织策略：Team/Enterprise 需要 `channelsEnabled: true`
- 会话白名单：必须在 `--channels` 列表中
- 插件市场验证：匹配插件来源

**通知类型**：
- `notifications/claude/channel`：普通频道消息
- `notifications/claude/channel/permission`：权限回复

### 4. 列表变更通知处理

**目的**：响应服务器端的工具/提示/资源变更

**处理流程**：
1. 注册 `ToolListChangedNotificationSchema` 处理器
2. 使缓存失效（`fetchToolsForClient.cache.delete`）
3. 重新获取数据
4. 更新 AppState
5. 记录分析事件 `tengu_mcp_list_changed`

### 5. 两阶段配置加载

**目的**：优化启动性能，先加载本地配置，再加载远程配置

**阶段**：
1. **Phase 1**：加载 Claude Code 配置（本地文件，快速）
2. **Phase 2**：加载 claude.ai 配置（网络请求，可能慢）
3. 去重处理：`dedupClaudeAiMcpServers` 避免重复连接

## 具体技术实现

### 关键数据结构

```typescript
// 待更新状态类型
type PendingUpdate = MCPServerConnection & {
  tools?: Tool[]
  commands?: Command[]
  resources?: ServerResource[]
}

// 重连定时器引用
const reconnectTimersRef = useRef<Map<string, NodeJS.Timeout>>(new Map())

// 频道警告去重
const channelWarnedKindsRef = useRef<Set<'disabled' | 'auth' | 'policy' | 'marketplace' | 'allowlist'>>(new Set())
```

### 关键流程

#### onConnectionAttempt 回调

```typescript
const onConnectionAttempt = useCallback(({ client, tools, commands, resources }) => {
  // 1. 更新服务器状态
  updateServer({ ...client, tools, commands, resources })
  
  // 2. 根据客户端类型处理
  switch (client.type) {
    case 'connected':
      // 注册 Elicitation 处理器
      registerElicitationHandler(client.client, client.name, setAppState)
      
      // 设置 onclose 处理（触发重连）
      client.client.onclose = () => { ... }
      
      // 频道通知注册（如果满足门控条件）
      if (feature('KAIROS') || feature('KAIROS_CHANNELS')) {
        // gateChannelServer 检查...
        // 注册 ChannelMessageNotificationSchema 处理器
      }
      
      // 注册列表变更通知处理器
      if (client.capabilities?.tools?.listChanged) { ... }
      if (client.capabilities?.prompts?.listChanged) { ... }
      if (client.capabilities?.resources?.listChanged) { ... }
      break
  }
}, [updateServer])
```

#### 服务器初始化 Effect

```typescript
useEffect(() => {
  async function initializeServersAsPending() {
    // 1. 获取配置
    const { servers: existingConfigs, errors: mcpErrors } = await getClaudeCodeMcpConfigs(dynamicMcpConfig)
    
    // 2. 添加错误到 AppState
    addErrorsToAppState(setAppState, mcpErrors)
    
    // 3. 清理过期客户端
    const { stale, ...mcpWithoutStale } = excludeStalePluginClients(prevState.mcp, configs)
    
    // 4. 清理过期连接
    for (const s of stale) { ... }
    
    // 5. 添加新客户端为 pending 状态
    const newClients = Object.entries(configs)
      .filter(([name]) => !existingServerNames.has(name))
      .map(([name, config]) => ({
        name,
        type: isMcpServerDisabled(name) ? 'disabled' : 'pending',
        config,
      }))
  }
}, [isStrictMcpConfig, dynamicMcpConfig, setAppState, sessionId, _pluginReconnectKey])
```

#### 配置加载与连接 Effect

```typescript
useEffect(() => {
  async function loadAndConnectMcpConfigs() {
    // 1. 启动 claude.ai 配置获取（异步）
    clearClaudeAIMcpConfigsCache()
    const claudeaiPromise = fetchClaudeAIMcpConfigsIfEligible()
    
    // 2. Phase 1: 加载本地配置并连接
    const { servers: claudeCodeConfigs, errors: mcpErrors } = await getClaudeCodeMcpConfigs(dynamicMcpConfig, claudeaiPromise)
    getMcpToolsCommandsAndResources(onConnectionAttempt, enabledConfigs)
    
    // 3. Phase 2: 等待 claude.ai 配置并连接
    const claudeaiConfigs = filterMcpServersByPolicy(await claudeaiPromise).allowed
    const { servers: dedupedClaudeAi } = dedupClaudeAiMcpServers(claudeaiConfigs, configs)
    getMcpToolsCommandsAndResources(onConnectionAttempt, enabledClaudeaiConfigs)
    
    // 4. 记录统计信息
    logEvent('tengu_mcp_servers', { ...counts, stdio_commands: stdioCommands })
  }
}, [isStrictMcpConfig, dynamicMcpConfig, onConnectionAttempt, setAppState, _authVersion, sessionId, _pluginReconnectKey])
```

### 指数退避重连算法

```typescript
const reconnectWithBackoff = async () => {
  for (let attempt = 1; attempt <= MAX_RECONNECT_ATTEMPTS; attempt++) {
    // 检查是否被禁用
    if (isMcpServerDisabled(client.name)) return
    
    // 更新为 pending 状态
    updateServer({ ...client, type: 'pending', reconnectAttempt: attempt, maxReconnectAttempts: MAX_RECONNECT_ATTEMPTS })
    
    // 尝试重连
    const result = await reconnectMcpServerImpl(client.name, client.config)
    
    if (result.client.type === 'connected') {
      // 成功，退出循环
      onConnectionAttempt(result)
      return
    }
    
    // 计算退避时间
    const backoffMs = Math.min(INITIAL_BACKOFF_MS * Math.pow(2, attempt - 1), MAX_BACKOFF_MS)
    await new Promise<void>(resolve => {
      const timer = setTimeout(resolve, backoffMs)
      reconnectTimersRef.current.set(client.name, timer)
    })
  }
}
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/services/mcp/client.ts` | `getMcpToolsCommandsAndResources`, `reconnectMcpServerImpl`, `clearServerCache` |
| `src/services/mcp/types.ts` | `MCPServerConnection`, `ScopedMcpServerConfig`, `ServerResource` 类型定义 |
| `src/services/mcp/config.ts` | `getClaudeCodeMcpConfigs`, `isMcpServerDisabled`, `setMcpServerEnabled` |
| `src/services/mcp/claudeai.ts` | `fetchClaudeAIMcpConfigsIfEligible`, `clearClaudeAIMcpConfigsCache`, `dedupClaudeAiMcpServers` |
| `src/services/mcp/channelNotification.ts` | `ChannelMessageNotificationSchema`, `gateChannelServer`, `wrapChannelMessage` |
| `src/services/mcp/channelPermissions.ts` | `createChannelPermissionCallbacks`, `isChannelPermissionRelayEnabled` |
| `src/services/mcp/elicitationHandler.ts` | `registerElicitationHandler` |
| `src/services/mcp/utils.ts` | `commandBelongsToServer`, `excludeStalePluginClients` |
| `src/state/AppState.ts` | `useAppStateStore`, `useSetAppState` |
| `src/utils/messageQueueManager.ts` | `enqueue`（频道消息入队） |

### 调用方

| 文件 | 调用方式 |
|------|----------|
| `src/services/mcp/MCPConnectionManager.tsx` | 主要调用方，通过 Hook 获取 `reconnectMcpServer` 和 `toggleMcpServer` |

### 被调用方

| 函数 | 被调用文件 |
|------|------------|
| `useManageMCPConnections` | `MCPConnectionManager.tsx` |

## 依赖与外部交互

### 外部系统交互

1. **MCP SDK**：通过 `@modelcontextprotocol/sdk` 进行通信
2. **GrowthBook**：功能开关检查（`feature('KAIROS')`, `feature('KAIROS_CHANNELS')`）
3. **分析系统**：`logEvent` 记录各类 MCP 事件
4. **消息队列**：`enqueue` 将频道消息加入处理队列
5. **安全存储**：通过 `getSecureStorage` 读取 OAuth 状态

### 配置依赖

- `CLAUDE_CODE_ENABLE_XAA`：XAA 功能开关
- `MCP_BATCH_FLUSH_MS`：批量更新间隔（默认 16ms）
- `MAX_RECONNECT_ATTEMPTS`：最大重连次数
- `INITIAL_BACKOFF_MS`/`MAX_BACKOFF_MS`：退避时间范围

## 风险、边界与改进建议

### 风险点

1. **内存泄漏风险**
   - `reconnectTimersRef` 中的定时器需要在组件卸载时清理
   - `channelWarnedKindsRef` 的警告去重是 session 级别的，不会自动重置

2. **竞态条件**
   - 配置加载和连接是异步的，如果用户在加载过程中禁用服务器，可能出现竞态
   - `claudeaiPromise` 在 Phase 1 和 Phase 2 之间共享，需要确保不会重复消费

3. **错误处理**
   - `initializeServersAsPending` 中的错误被捕获但仅记录日志，用户可能感知不到
   - 重连失败后的状态更新可能覆盖用户手动操作

4. **性能问题**
   - 大量 MCP 服务器同时连接时，`getMcpToolsCommandsAndResources` 可能阻塞
   - 批量更新使用 setTimeout，在极端情况下可能延迟 16ms

### 边界情况

1. **服务器快速切换启用/禁用状态**
   - 代码通过检查 `isMcpServerDisabled` 来避免不必要的重连
   - 但磁盘状态检查可能存在延迟

2. **网络波动**
   - 指数退避算法可以处理短暂的网络问题
   - 长时间断网会导致所有服务器进入 `failed` 状态

3. **配置变更**
   - `/reload-plugins` 触发 `_pluginReconnectKey` 变化，重新初始化
   - 配置哈希变化会识别为过期客户端并断开连接

### 改进建议

1. **添加连接池限制**
   ```typescript
   // 建议添加并发连接数限制，避免同时连接过多服务器
   const MAX_CONCURRENT_CONNECTIONS = 10
   ```

2. **优化重连策略**
   - 添加网络状态检测，在网络恢复时立即尝试重连
   - 支持用户手动触发重连（已支持 `reconnectMcpServer`）

3. **增强错误报告**
   - 将初始化错误显示在 UI 中，而不仅仅是日志
   - 为重连失败添加更详细的错误分类

4. **代码重构**
   - 将 `onConnectionAttempt` 拆分为更小的函数，提高可读性
   - 将频道通知逻辑提取到单独的 Hook 中

5. **测试覆盖**
   - 添加单元测试验证重连逻辑
   - 模拟网络波动测试状态一致性
