# useMcpConnectivityStatus.tsx 深度研究

## 场景与职责

`useMcpConnectivityStatus` 是一个 React Hook，用于监控 MCP（Model Context Protocol）服务器连接状态并在出现问题时显示通知。它检测本地 MCP 服务器和 Claude.ai 连接器的失败状态以及需要认证的情况，为用户提供及时的状态反馈。

### 核心场景
1. **本地 MCP 服务器失败**：当本地 MCP 服务器连接失败时通知用户
2. **Claude.ai 连接器失败**：当 Claude.ai MCP 连接器不可用时通知用户
3. **认证需求提示**：当 MCP 服务器需要认证时提示用户
4. **连接历史感知**：对于从未成功连接过的连接器，不显示失败通知（避免打扰）

## 功能点目的

### 1. MCP 客户端状态分类
将 MCP 客户端分为四类：
- **失败的本地客户端**：本地 MCP 服务器连接失败
- **失败的 Claude.ai 客户端**：Claude.ai 连接器失败且曾经成功连接过
- **需要认证的本地服务器**：本地 MCP 服务器需要认证
- **需要认证的 Claude.ai 服务器**：Claude.ai 连接器需要认证且曾经成功连接过

### 2. 智能通知策略
- 对于 Claude.ai 连接器，仅当曾经成功连接过 (`hasClaudeAiMcpEverConnected`) 才显示失败/认证通知
- 避免对从未使用过的组织配置连接器进行打扰
- 本地服务器始终显示通知（用户主动配置的）

### 3. 多类型通知
- **失败通知**：红色错误样式，显示失败的服务器数量
- **认证通知**：黄色警告样式，显示需要认证的服务器数量
- **区分本地和 Claude.ai**：分别显示不同类型的连接器

### 4. 复数处理
- 根据数量自动使用单数或复数形式（"server/servers", "connector/connectors"）

## 具体技术实现

### 关键数据结构

```typescript
// MCP 服务器连接类型
interface MCPServerConnection {
  name: string
  type: 'connected' | 'failed' | 'pending' | 'needs-auth'
  config: {
    type: 'sse-ide' | 'ws-ide' | 'claudeai-proxy' | string
    // ...
  }
}

// Props 接口
interface Props {
  mcpClients?: MCPServerConnection[]
}

// 空数组常量（避免重复创建）
const EMPTY_MCP_CLIENTS: MCPServerConnection[] = []
```

### 核心流程

```
useEffect 监听 mcpClients 变化
    ↓
检查远程模式 (getIsRemoteMode)
    ↓
分类过滤客户端：
    - failedLocalClients: 失败的本地客户端（排除 IDE 和 Claude.ai）
    - failedClaudeAiClients: 失败的 Claude.ai 客户端（需曾经连接过）
    - needsAuthLocalServers: 需要认证的本地服务器
    - needsAuthClaudeAiServers: 需要认证的 Claude.ai 服务器（需曾经连接过）
    ↓
如果所有类别都为空，返回
    ↓
显示相应的通知：
    - mcp-failed: 本地失败通知
    - mcp-claudeai-failed: Claude.ai 失败通知
    - mcp-needs-auth: 本地认证通知
    - mcp-claudeai-needs-auth: Claude.ai 认证通知
```

### 关键代码路径

```typescript
export function useMcpConnectivityStatus({
  mcpClients = EMPTY_MCP_CLIENTS
}: Props) {
  const { addNotification } = useNotifications()

  useEffect(() => {
    if (getIsRemoteMode()) return

    // 分类过滤
    const failedLocalClients = mcpClients.filter(
      client =>
        client.type === 'failed' &&
        client.config.type !== 'sse-ide' &&
        client.config.type !== 'ws-ide' &&
        client.config.type !== 'claudeai-proxy'
    )
    
    const failedClaudeAiClients = mcpClients.filter(
      client =>
        client.type === 'failed' &&
        client.config.type === 'claudeai-proxy' &&
        hasClaudeAiMcpEverConnected(client.name)
    )
    
    const needsAuthLocalServers = mcpClients.filter(
      client =>
        client.type === 'needs-auth' &&
        client.config.type !== 'claudeai-proxy'
    )
    
    const needsAuthClaudeAiServers = mcpClients.filter(
      client =>
        client.type === 'needs-auth' &&
        client.config.type === 'claudeai-proxy' &&
        hasClaudeAiMcpEverConnected(client.name)
    )

    // 如果没有问题，返回
    if (
      failedLocalClients.length === 0 &&
      failedClaudeAiClients.length === 0 &&
      needsAuthLocalServers.length === 0 &&
      needsAuthClaudeAiServers.length === 0
    ) {
      return
    }

    // 显示失败通知 - 本地
    if (failedLocalClients.length > 0) {
      addNotification({
        key: "mcp-failed",
        jsx: <>
          <Text color="error">
            {failedLocalClients.length} MCP{" "}
            {failedLocalClients.length === 1 ? "server" : "servers"} failed
          </Text>
          <Text dimColor={true}> · /mcp</Text>
        </>,
        priority: "medium"
      })
    }

    // 显示失败通知 - Claude.ai
    if (failedClaudeAiClients.length > 0) {
      addNotification({
        key: "mcp-claudeai-failed",
        jsx: <>
          <Text color="error">
            {failedClaudeAiClients.length} claude.ai{" "}
            {failedClaudeAiClients.length === 1 ? "connector" : "connectors"}{" "}
            unavailable
          </Text>
          <Text dimColor={true}> · /mcp</Text>
        </>,
        priority: "medium"
      })
    }

    // 显示认证通知 - 本地
    if (needsAuthLocalServers.length > 0) {
      addNotification({
        key: "mcp-needs-auth",
        jsx: <>
          <Text color="warning">
            {needsAuthLocalServers.length} MCP{" "}
            {needsAuthLocalServers.length === 1 ? "server needs" : "servers need"}{" "}
            auth
          </Text>
          <Text dimColor={true}> · /mcp</Text>
        </>,
        priority: "medium"
      })
    }

    // 显示认证通知 - Claude.ai
    if (needsAuthClaudeAiServers.length > 0) {
      addNotification({
        key: "mcp-claudeai-needs-auth",
        jsx: <>
          <Text color="warning">
            {needsAuthClaudeAiServers.length} claude.ai{" "}
            {needsAuthClaudeAiServers.length === 1 ? "connector needs" : "connectors need"}{" "}
            auth
          </Text>
          <Text dimColor={true}> · /mcp</Text>
        </>,
        priority: "medium"
      })
    }
  }, [addNotification, mcpClients])
}
```

### 过滤逻辑详解

```typescript
// 失败的本地客户端
const failedLocalClients = mcpClients.filter(client =>
  client.type === 'failed' &&
  client.config.type !== 'sse-ide' &&    // 排除 IDE SSE 连接
  client.config.type !== 'ws-ide' &&     // 排除 IDE WebSocket 连接
  client.config.type !== 'claudeai-proxy' // 排除 Claude.ai 连接器
)

// 失败的 Claude.ai 客户端（仅曾经连接过的）
const failedClaudeAiClients = mcpClients.filter(client =>
  client.type === 'failed' &&
  client.config.type === 'claudeai-proxy' &&
  hasClaudeAiMcpEverConnected(client.name) // 检查连接历史
)

// 需要认证的本地服务器
const needsAuthLocalServers = mcpClients.filter(client =>
  client.type === 'needs-auth' &&
  client.config.type !== 'claudeai-proxy'
)

// 需要认证的 Claude.ai 服务器（仅曾经连接过的）
const needsAuthClaudeAiServers = mcpClients.filter(client =>
  client.type === 'needs-auth' &&
  client.config.type === 'claudeai-proxy' &&
  hasClaudeAiMcpEverConnected(client.name)
)
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `React`, `useEffect` | `react` | React Hook API |
| `useNotifications` | `src/context/notifications.js` | 通知系统 |
| `getIsRemoteMode` | `src/bootstrap/state.js` | 远程模式检测 |
| `Text` | `src/ink.js` | Ink 文本组件 |
| `hasClaudeAiMcpEverConnected` | `src/services/mcp/claudeai.js` | 检查 Claude.ai 连接历史 |
| `MCPServerConnection` | `src/services/mcp/types.js` | MCP 连接类型 |

### 依赖模块详解

#### 1. hasClaudeAiMcpEverConnected (src/services/mcp/claudeai.js)
检查指定的 Claude.ai MCP 连接器是否曾经成功连接过：
```typescript
export function hasClaudeAiMcpEverConnected(name: string): boolean {
  const config = getGlobalConfig()
  return config.claudeAiMcpEverConnected?.includes(name) ?? false
}
```

此信息存储在全局配置的 `claudeAiMcpEverConnected` 数组中，用于区分：
- 用户曾经使用过的连接器（值得提醒）
- 组织配置但用户从未使用的连接器（避免打扰）

#### 2. MCPServerConnection (src/services/mcp/types.js)
MCP 服务器连接的类型定义：
```typescript
type MCPServerConnection = {
  name: string
  type: 'connected' | 'failed' | 'pending' | 'needs-auth'
  config: MCPServerConfig
}

type MCPServerConfig = {
  type: 'sse-ide' | 'ws-ide' | 'claudeai-proxy' | 'stdio' | 'sse'
  // ... 其他配置
}
```

## 风险、边界与改进建议

### 潜在风险

1. **频繁通知**
   - 如果 MCP 客户端状态频繁变化，可能导致通知闪烁
   - 当前没有防抖或节流处理

2. **通知堆积**
   - 如果多个 MCP 服务器同时出现问题，会显示多个通知
   - 可能淹没用户界面

3. **历史状态依赖**
   - `hasClaudeAiMcpEverConnected` 依赖全局配置
   - 如果配置被清除，行为会改变

4. **IDE 连接排除**
   - 明确排除了 `sse-ide` 和 `ws-ide` 类型的连接
   - 这些由 `useIDEStatusIndicator` 单独处理

### 边界情况

1. **空客户端数组**
   - 使用 `EMPTY_MCP_CLIENTS` 作为默认值
   - 避免不必要的重新渲染

2. **远程模式**
   - 在远程模式下完全禁用
   - 这是预期行为

3. **Claude.ai 连接器首次失败**
   - 如果连接器从未成功连接过，不显示失败通知
   - 这是设计决策，避免对新用户造成打扰

4. **通知 Key 冲突**
   - 使用固定的 key（如 `"mcp-failed"`）
   - 新通知会替换旧通知（如果通知系统支持）

### 改进建议

1. **增加防抖处理**
   ```typescript
   const debouncedMcpClients = useDebounce(mcpClients, 500)
   useEffect(() => {
     // 使用 debouncedMcpClients
   }, [debouncedMcpClients])
   ```

2. **消息聚合**
   ```typescript
   // 如果多个服务器失败，考虑合并为一条通知
   const totalFailed = failedLocalClients.length + failedClaudeAiClients.length
   if (totalFailed > 3) {
     // 显示汇总通知
   }
   ```

3. **添加分析事件**
   ```typescript
   if (failedLocalClients.length > 0) {
     logEvent('tengu_mcp_failed', {
       count: failedLocalClients.length,
       servers: failedLocalClients.map(c => c.name)
     })
   }
   ```

4. **考虑通知优先级调整**
   ```typescript
   // 对于曾经工作过的连接器失败，可以提高优先级
   const priority = hasClaudeAiMcpEverConnected(name) ? 'high' : 'medium'
   ```

5. **添加重试提示**
   ```typescript
   // 在通知中添加重试建议
   <Text dimColor={true}> · /mcp retry</Text>
   ```

6. **错误详情展示**
   ```typescript
   // 考虑在通知中显示简短的错误信息
   <Text dimColor={true}> ({client.lastError?.message})</Text>
   ```

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useMcpConnectivityStatus.tsx`
- **通知系统**: `src/context/notifications.tsx`
- **Claude.ai MCP**: `src/services/mcp/claudeai.js`
- **MCP 类型**: `src/services/mcp/types.js`
- **启动状态**: `src/bootstrap/state.ts`
- **Ink 组件**: `src/ink.js`
