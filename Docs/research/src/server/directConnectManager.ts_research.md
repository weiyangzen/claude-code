# directConnectManager.ts 深度研究

## 场景与职责

本模块是 Claude Code CLI **直连服务器模式的核心 WebSocket 管理器**，负责在客户端与自托管 Claude 服务器之间建立和维护实时双向通信。它是 Direct Connect 功能的传输层实现，处理消息收发、权限请求、连接状态管理等。

**核心场景：**
1. **交互式直连模式**: `claude connect <url>` 启动 REPL 连接到远程服务器
2. **URL 协议直连**: 通过 `cc://host:port` 或 `cc+unix://socket` 自动连接
3. **权限协商**: 处理服务器发送的工具使用权限请求，向用户展示确认对话框
4. **消息中转**: 将用户输入转发到服务器，将服务器响应（assistant 消息、工具结果）展示给用户

## 功能点目的

### 1. WebSocket 连接管理 (DirectConnectSessionManager)

**核心职责：**
- 建立到服务器 WebSocket 端点的持久连接
- 处理连接生命周期事件（open/close/error）
- 自动重连逻辑（通过回调通知上层）

**关键方法：**
- `connect()`: 初始化 WebSocket 连接，设置事件监听器
- `disconnect()`: 优雅关闭连接
- `isConnected()`: 检查连接状态

### 2. 消息协议处理

**输入消息处理（服务器 → 客户端）：**
- **SDK 消息**: `assistant`, `result`, `system` 等类型，转发到 UI
- **权限请求**: `control_request` 类型，触发用户确认流程
- **控制响应**: `control_response` 类型，确认收到
- **心跳消息**: `keep_alive` 类型，忽略处理

**输出消息构建（客户端 → 服务器）：**
- **用户消息**: 包装为 `SDKUserMessage` 格式
- **权限响应**: 回复工具使用许可/拒绝
- **中断信号**: 发送 `interrupt` 请求取消当前操作

### 3. 权限请求处理

专门处理 `can_use_tool` 子类型的控制请求：
- 解析工具名称、输入参数、描述信息
- 通过回调通知 UI 层展示权限确认对话框
- 接收用户决策（允许/拒绝）并回复服务器

## 具体技术实现

### 关键流程

#### 连接建立流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    DirectConnectSessionManager                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ connect()                                                  │  │
│  │ ┌───────────────────────────────────────────────────────┐ │  │
│  │ │ 1. 创建 WebSocket 实例                                 │ │  │
│  │ │    - URL: config.wsUrl                                 │ │  │
│  │ │    - Headers: Authorization: Bearer <authToken>        │ │  │
│  │ ├───────────────────────────────────────────────────────┤ │  │
│  │ │ 2. 绑定事件处理器                                      │ │  │
│  │ │    - open → onConnected() 回调                         │ │  │
│  │ │    - message → 消息解析和路由                          │ │  │
│  │ │    - close → onDisconnected() 回调                     │ │  │
│  │ │    - error → onError() 回调                            │ │  │
│  │ └───────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### 消息接收处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      消息事件处理器                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ onMessage(event)                                           │  │
│  │ ┌───────────────────────────────────────────────────────┐ │  │
│  │ │ 1. 数据解码                                            │ │  │
│  │ │    - event.data 转为字符串                             │ │  │
│  │ │    - 按行分割，过滤空行                                │ │  │
│  │ ├───────────────────────────────────────────────────────┤ │  │
│  │ │ 2. 逐行解析 JSON                                       │ │  │
│  │ │    - 解析失败则跳过该行                                │ │  │
│  │ ├───────────────────────────────────────────────────────┤ │  │
│  │ │ 3. 类型守卫验证 (isStdoutMessage)                      │ │  │
│  │ ├───────────────────────────────────────────────────────┤ │  │
│  │ │ 4. 消息路由                                            │ │  │
│  │ │    ├─ control_request                                 │ │  │
│  │ │    │   ├─ can_use_tool → onPermissionRequest()        │ │  │
│  │ │    │   └─ 其他 → sendErrorResponse()                  │ │  │
│  │ │    ├─ control_response → 忽略                         │ │  │
│  │ │    ├─ keep_alive → 忽略                               │ │  │
│  │ │    ├─ streamlined_text → 忽略                         │ │  │
│  │ │    ├─ system/post_turn_summary → 忽略                 │ │  │
│  │ │    └─ 其他 SDK 消息 → onMessage() 回调                │ │  │
│  │ └───────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### 权限请求响应流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限请求处理时序                               │
│                                                                  │
│  服务器                    本模块                      UI 层     │
│    │                        │                           │       │
│    │ ──control_request────→ │                           │       │
│    │   (can_use_tool)       │                           │       │
│    │                        │ ──onPermissionRequest()──→│       │
│    │                        │   (request, requestId)    │       │
│    │                        │                           │       │
│    │                        │ ←────用户决策(allow/deny)──│       │
│    │                        │                           │       │
│    │ ←──control_response────│                           │       │
│    │   (success + behavior) │                           │       │
│    │                        │                           │       │
└─────────────────────────────────────────────────────────────────┘
```

### 数据结构

#### DirectConnectConfig
```typescript
{
  serverUrl: string    // 服务器基础 URL（用于 HTTP API）
  sessionId: string    // 会话唯一标识
  wsUrl: string        // WebSocket 连接地址
  authToken?: string   // 可选认证令牌
}
```

#### DirectConnectCallbacks
```typescript
{
  onMessage: (SDKMessage) => void              // SDK 消息回调
  onPermissionRequest: (SDKControlPermissionRequest, requestId) => void
  onConnected?: () => void                     // 连接成功回调
  onDisconnected?: () => void                  // 连接断开回调
  onError?: (Error) => void                    // 错误回调
}
```

#### 内部消息格式

**用户消息发送格式（匹配 StructuredIO）：**
```typescript
{
  type: 'user',
  message: {
    role: 'user',
    content: RemoteMessageContent  // 字符串或内容块数组
  },
  parent_tool_use_id: null,
  session_id: ''
}
```

**权限响应格式：**
```typescript
// 允许
{
  type: 'control_response',
  response: {
    subtype: 'success',
    request_id: string,
    response: {
      behavior: 'allow',
      updatedInput: Record<string, unknown>
    }
  }
}

// 拒绝
{
  type: 'control_response',
  response: {
    subtype: 'success',
    request_id: string,
    response: {
      behavior: 'deny',
      message: string
    }
  }
}
```

**中断请求格式：**
```typescript
{
  type: 'control_request',
  request_id: crypto.randomUUID(),
  request: {
    subtype: 'interrupt'
  }
}
```

### 协议细节

- **WebSocket 库**: 使用原生 `WebSocket` API（Bun 支持 headers 选项）
- **消息分隔**: 服务器发送的消息以 `\n` 分隔，支持批量处理
- **认证**: 通过 WebSocket 握手时的 `Authorization` Header 传递 Bearer Token
- **心跳**: 服务器发送 `keep_alive` 消息维持连接

## 关键代码路径与文件引用

### 本文件关键代码

| 行号 | 代码 | 说明 |
|------|------|------|
| 13-18 | `DirectConnectConfig` 类型 | 连接配置数据结构 |
| 20-29 | `DirectConnectCallbacks` 类型 | 回调接口定义 |
| 31-38 | `isStdoutMessage` 类型守卫 | 验证消息格式 |
| 40-48 | `DirectConnectSessionManager` 类定义 | 主管理器类 |
| 50-123 | `connect()` 方法 | WebSocket 连接建立和事件绑定 |
| 64-114 | `message` 事件处理器 | 消息解析和路由逻辑 |
| 82-99 | `control_request` 处理 | 权限请求特殊处理 |
| 125-142 | `sendMessage()` 方法 | 发送用户消息到服务器 |
| 144-167 | `respondToPermissionRequest()` | 回复权限请求 |
| 172-186 | `sendInterrupt()` 方法 | 发送中断信号 |
| 188-201 | `sendErrorResponse()` 私有方法 | 发送错误响应 |
| 203-208 | `disconnect()` 方法 | 关闭连接 |
| 210-212 | `isConnected()` 方法 | 状态检查 |

### 依赖文件

| 文件路径 | 导入内容 | 用途 |
|----------|----------|------|
| `../entrypoints/agentSdkTypes.js` | `SDKMessage` | SDK 消息类型 |
| `../entrypoints/sdk/controlTypes.js` | `SDKControlPermissionRequest`, `StdoutMessage` | 控制协议类型 |
| `../remote/RemoteSessionManager.js` | `RemotePermissionResponse` | 权限响应类型 |
| `../utils/debug.js` | `logForDebugging` | 调试日志 |
| `../utils/slowOperations.js` | `jsonParse`, `jsonStringify` | JSON 处理（带性能监控）|
| `../utils/teleport/api.js` | `RemoteMessageContent` | 消息内容类型 |

### 调用方

| 文件路径 | 使用方式 | 场景 |
|----------|----------|------|
| `src/hooks/useDirectConnect.ts` | 创建管理器实例 | React Hook 封装 |
| `src/screens/REPL.tsx` | 通过 useDirectConnect 使用 | REPL 主界面集成 |

## 依赖与外部交互

### 与 useDirectConnect Hook 的协作

```
┌─────────────────────────────────────────────────────────────────┐
│                      直连模式架构                                │
│                                                                  │
│  ┌─────────────┐    ┌──────────────────┐    ┌──────────────┐   │
│  │   REPL.tsx  │───→│ useDirectConnect │───→│ DirectConnect │   │
│  │             │    │     Hook         │    │ SessionManager│   │
│  │  UI 渲染    │←───│                  │←───│               │   │
│  │  消息展示   │    │  React 状态管理   │    │  WebSocket    │   │
│  │  权限对话框 │    │  生命周期管理     │    │  协议处理     │   │
│  └─────────────┘    └──────────────────┘    └──────────────┘   │
│                                                          │      │
│                              ┌───────────────────────────┘      │
│                              ↓                                   │
│                       ┌──────────────┐                          │
│                       │ Claude 服务器 │                          │
│                       │  (WebSocket) │                          │
│                       └──────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

### 消息类型映射

| 服务器消息类型 | 处理方式 | 目标 |
|---------------|----------|------|
| `assistant` | `onMessage` → `convertSDKMessage` | UI 消息列表 |
| `result` | `onMessage` → `convertSDKMessage` | UI 消息列表 |
| `system/init` | 去重后 → UI | 初始化标记 |
| `control_request` | `onPermissionRequest` | 权限对话框 |
| `control_response` | 忽略 | - |
| `keep_alive` | 忽略 | - |
| `streamlined_text` | 忽略 | - |
| `streamlined_tool_use_summary` | 忽略 | - |

## 风险、边界与改进建议

### 已知风险

1. **WebSocket Headers 兼容性**
   - 风险：代码注释提到 "Bun's WebSocket supports headers option but the DOM typings don't"
   - 缓解：使用 `as unknown as string[]` 类型断言绕过
   - 隐患：如果运行在非 Bun 环境可能失败

2. **消息解析失败静默处理**
   - 风险：第 72-74 行 `jsonParse` 失败直接 `continue`，可能丢失消息
   - 建议：增加调试日志记录解析失败的原始数据

3. **未支持的 control_request 子类型**
   - 风险：目前仅支持 `can_use_tool`，其他子类型返回错误
   - 未来扩展：可能需要支持 `interrupt`, `set_model` 等

4. **无内置重连机制**
   - 风险：连接断开后依赖上层（useDirectConnect）处理重连
   - 现状：通过 `onDisconnected` 回调通知，由 REPL 决定是否退出

### 边界情况

| 场景 | 行为 |
|------|------|
| WebSocket 未连接时发送消息 | `sendMessage()` 返回 `false` |
| 重复收到 `system/init` | 通过 `hasReceivedInitRef` 过滤（在 useDirectConnect 中）|
| 权限请求时连接断开 | 请求挂起，重新连接后服务器应重新发送 |
| 收到未知消息类型 | 作为 SDK 消息转发到 UI |
| 多行消息（含 `\n`）| 逐行解析，每行独立处理 |

### 改进建议

1. **添加消息序列号**
   - 实现消息去重和顺序保证
   - 检测消息丢失

2. **内置重连逻辑**
   ```typescript
   // 建议添加指数退避重连
   private reconnectAttempts = 0
   private maxReconnectDelay = 30000
   
   private scheduleReconnect() {
     const delay = Math.min(1000 * 2 ** this.reconnectAttempts, this.maxReconnectDelay)
     setTimeout(() => this.connect(), delay)
     this.reconnectAttempts++
   }
   ```

3. **心跳检测**
   - 客户端主动发送 ping，检测连接健康
   - 超时未收到 pong 判定为断开

4. **消息队列**
   - 连接断开时缓存待发送消息
   - 重连后按序重发

5. **更完善的错误分类**
   - 区分网络错误、协议错误、认证错误
   - 提供可操作的错误提示

6. **流量控制**
   - 实现背压机制，防止消息堆积
   - 监控待处理消息数量

### 测试建议

- 单元测试：mock WebSocket 验证消息路由逻辑
- 集成测试：使用真实服务器验证完整会话流程
- 边界测试：断网重连、消息乱序、大数据包等
- 性能测试：高频率消息收发、内存泄漏检测
