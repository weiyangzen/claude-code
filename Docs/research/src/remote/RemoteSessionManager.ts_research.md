# RemoteSessionManager.ts 研究文档

## 场景与职责

`RemoteSessionManager` 是 Claude Code CLI 中用于管理远程 CCR（Claude Code Remote）会话的核心类。它充当本地 CLI 与远程 CCR 容器之间的通信桥梁，主要用于以下场景：

1. **远程会话连接**：通过 WebSocket 连接到 Anthropic 的远程会话服务
2. **双向消息通信**：
   - 接收：通过 WebSocket 订阅接收来自 CCR 的 SDK 消息（助手回复、流事件、系统消息等）
   - 发送：通过 HTTP POST 将用户消息发送到远程会话
3. **权限请求处理**：处理远程 CCR 发送的工具使用权限请求，支持用户允许/拒绝操作
4. **会话生命周期管理**：连接建立、重连、断开、中断信号发送等

该模块是 `useRemoteSession` hook 的底层依赖，为 REPL 界面提供远程会话能力。

## 功能点目的

### 1. 会话配置 (`RemoteSessionConfig`)
```typescript
type RemoteSessionConfig = {
  sessionId: string           // 远程会话唯一标识
  getAccessToken: () => string // OAuth 访问令牌获取函数
  orgUuid: string             // 组织 UUID
  hasInitialPrompt?: boolean  // 是否有初始提示（影响标题生成）
  viewerOnly?: boolean        // 纯查看模式（用于 claude assistant）
}
```

### 2. 回调接口 (`RemoteSessionCallbacks`)
定义了与上层 UI 交互的回调：
- `onMessage`: 接收到 SDK 消息时调用
- `onPermissionRequest`: 接收到权限请求时调用
- `onPermissionCancelled`: 权限请求被取消时调用
- `onConnected/onDisconnected/onReconnecting`: 连接状态变化
- `onError`: 错误处理

### 3. 权限响应类型 (`RemotePermissionResponse`)
简化的权限结果类型，支持两种行为：
- `allow`: 允许操作，可携带更新后的输入参数
- `deny`: 拒绝操作，附带拒绝原因消息

## 具体技术实现

### 关键流程

#### 1. 连接建立流程 (`connect()`)
```
1. 创建 SessionsWebSocket 实例
2. 配置 WebSocket 回调（消息、连接、关闭、错误）
3. 调用 websocket.connect() 建立连接
4. 连接成功后通过 onConnected 回调通知上层
```

#### 2. 消息处理流程 (`handleMessage()`)
消息类型分发逻辑：

| 消息类型 | 处理方式 |
|---------|---------|
| `control_request` | 调用 `handleControlRequest()` 处理权限请求 |
| `control_cancel_request` | 从 pending 列表移除，调用 `onPermissionCancelled` |
| `control_response` | 记录调试日志（确认消息） |
| SDKMessage (其他) | 通过 `isSDKMessage` 类型守卫检查后，调用 `onMessage` |

#### 3. 权限请求处理 (`handleControlRequest()`)
```
1. 提取 request_id 和内部请求数据
2. 检查 subtype 是否为 'can_use_tool'
3. 如果是：存储到 pendingPermissionRequests Map，调用 onPermissionRequest
4. 如果不是：返回错误响应（防止服务器挂起等待）
```

#### 4. 发送消息流程 (`sendMessage()`)
```
1. 记录调试日志
2. 调用 sendEventToRemoteSession() 发送 HTTP POST 请求
3. 返回发送成功/失败状态
```

#### 5. 权限响应流程 (`respondToPermissionRequest()`)
```
1. 从 pendingPermissionRequests 查找对应请求
2. 构造 SDKControlResponse 响应对象
3. 通过 WebSocket 发送响应
4. 从 pending 列表删除
```

### 数据结构

#### 内部状态
```typescript
private websocket: SessionsWebSocket | null = null
private pendingPermissionRequests: Map<string, SDKControlPermissionRequest> = new Map()
```

#### 控制消息类型守卫
```typescript
function isSDKMessage(message): message is SDKMessage {
  return (
    message.type !== 'control_request' &&
    message.type !== 'control_response' &&
    message.type !== 'control_cancel_request'
  )
}
```

### 协议与命令

#### 支持的 Control Request Subtypes
- `can_use_tool`: 工具使用权限请求

#### 支持的 Control Response 格式
```typescript
// 成功响应
{
  type: 'control_response',
  response: {
    subtype: 'success',
    request_id: string,
    response: {
      behavior: 'allow' | 'deny',
      // allow 时包含 updatedInput
      // deny 时包含 message
    }
  }
}

// 错误响应
{
  type: 'control_response',
  response: {
    subtype: 'error',
    request_id: string,
    error: string
  }
}
```

#### 中断命令
通过 `cancelSession()` 发送中断信号：
```typescript
this.websocket?.sendControlRequest({ subtype: 'interrupt' })
```

## 关键代码路径与文件引用

### 核心文件
- **本文件**: `src/remote/RemoteSessionManager.ts` (343 lines)

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/remote/SessionsWebSocket.ts` | WebSocket 连接管理 |
| `src/remote/sdkMessageAdapter.ts` | SDK 消息类型定义（间接） |
| `src/entrypoints/agentSdkTypes.ts` | SDKMessage 类型 |
| `src/entrypoints/sdk/controlTypes.ts` | 控制消息类型（SDKControlRequest 等） |
| `src/utils/teleport/api.ts` | `sendEventToRemoteSession` HTTP API |
| `src/utils/debug.ts` | `logForDebugging` 调试日志 |
| `src/utils/log.ts` | `logError` 错误日志 |

### 调用方文件
| 文件路径 | 用途 |
|---------|------|
| `src/hooks/useRemoteSession.ts` | React hook 封装，REPL 使用 |
| `src/hooks/useDirectConnect.ts` | 直接连接模式（部分类型引用） |

### 关键函数引用路径
```
connect()
  └── SessionsWebSocket.connect()
      └── WebSocket 连接到 wss://api.anthropic.com/v1/sessions/ws/{id}/subscribe

sendMessage()
  └── sendEventToRemoteSession() [src/utils/teleport/api.ts]
      └── axios.post() 到 /v1/sessions/{id}/events

respondToPermissionRequest()
  └── SessionsWebSocket.sendControlResponse()
      └── ws.send() 发送 JSON 响应

cancelSession()
  └── SessionsWebSocket.sendControlRequest()
      └── ws.send() 发送中断命令
```

## 依赖与外部交互

### 运行时依赖
- **WebSocket**: 通过 `SessionsWebSocket` 实现，支持 Bun 和 Node.js (ws 包)
- **HTTP API**: 通过 `sendEventToRemoteSession` 使用 axios 发送 POST 请求
- **OAuth**: 通过 `getAccessToken` 回调获取认证令牌

### 环境变量依赖（通过依赖传递）
- `CLAUDE_CODE_CLIENT_CERT/KEY`: mTLS 证书配置
- `HTTPS_PROXY/HTTP_PROXY`: 代理配置
- `DEBUG/DEBUG_SDK`: 调试模式

### 外部服务交互
1. **Anthropic API**:
   - WebSocket: `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`
   - HTTP: `https://api.anthropic.com/v1/sessions/{sessionId}/events`

2. **OAuth 认证**:
   - 通过回调函数 `getAccessToken()` 获取
   - 通过 `orgUuid` 标识组织

## 风险、边界与改进建议

### 已知风险

#### 1. 消息重复问题
- **风险**: 用户消息可能通过 WebSocket 回显多次（服务器广播 + worker 回显）
- **缓解**: 上层 `useRemoteSession` 使用 `BoundedUUIDSet` 进行去重过滤

#### 2. 权限请求状态同步
- **风险**: 网络断开期间可能错过 `control_cancel_request`，导致 UI 显示过期权限请求
- **缓解**: 重连时 `onReconnecting` 回调会清理状态

#### 3. 会话超时
- **风险**: 远程会话可能无响应（容器冷启动、compaction 等）
- **缓解**: 上层实现 60s/180s 超时检测和自动重连

### 边界条件

| 边界场景 | 行为 |
|---------|------|
| 重复连接调用 | 检查 `state === 'connecting'`，避免重复连接 |
| 未识别的 control subtype | 返回错误响应，防止服务器挂起 |
| 不存在的 permission request ID | 记录错误日志，静默返回 |
| 未连接时发送响应 | WebSocket 层检查，记录错误日志 |
| viewerOnly 模式 | 禁用中断信号、禁用重连超时、不更新标题 |

### 改进建议

#### 1. 连接状态机强化
当前状态管理分散在 `SessionsWebSocket` 和本类之间，建议：
- 统一状态机定义
- 添加状态转换事件日志
- 考虑添加 `connecting` 超时处理

#### 2. 权限请求过期机制
当前 pending 权限请求只会在以下情况清理：
- 用户响应
- 收到 cancel 请求
- 断开连接

建议添加：
- 权限请求超时机制（如 5 分钟）
- 超时后自动拒绝并清理

#### 3. 重连策略优化
当前重连逻辑在 `SessionsWebSocket` 中实现，但：
- 4001 (session not found) 有 3 次重试
- 其他情况最多 5 次重试

建议：
- 考虑指数退避策略
- 添加重连次数/间隔配置选项
- 区分可恢复错误和永久错误

#### 4. 错误处理增强
当前错误处理主要依赖日志记录，建议：
- 添加结构化错误类型
- 提供错误恢复建议
- 考虑添加错误上报机制

#### 5. 测试覆盖
建议添加：
- 单元测试：消息类型守卫、权限请求处理
- 集成测试：连接建立/断开、重连场景
- Mock 测试：模拟 WebSocket 和 HTTP 响应

### 代码质量观察

#### 优点
- 类型定义清晰，使用 TypeScript 严格类型
- 调试日志丰富，便于问题排查
- 职责分离明确（WebSocket vs HTTP）
- 支持 viewerOnly 模式，满足特殊场景

#### 潜在问题
- `pendingPermissionRequests` Map 可能内存泄漏（如果 cancel 消息丢失）
- `isSDKMessage` 类型守卫使用否定检查，可能不够健壮
- 部分错误处理仅记录日志，未向上层传播
