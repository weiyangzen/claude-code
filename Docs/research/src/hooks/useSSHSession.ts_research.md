# useSSHSession.ts 深度研究文档

## 场景与职责

`useSSHSession` 是一个 React Hook，用于在 Claude Code CLI 中实现 SSH 远程会话的 REPL 集成。它是 `claude ssh` 命令的核心组件，负责建立和管理与远程 SSH 服务器的双向通信通道。

与 `useDirectConnect`（WebSocket 连接）形成对比，该 Hook 专门处理 SSH 子进程的生命周期管理。关键区别在于：SSH 进程和认证代理在 Hook 运行之前就已经创建（在 `main.tsx` 启动阶段），而 `useDirectConnect` 是在 Effect 内部创建 WebSocket。

### 核心职责
1. **SSH 会话管理**：通过 `SSHSessionManager` 管理 SSH 连接生命周期
2. **消息收发**：发送用户消息到远程，接收并处理远程返回的 SDK 消息
3. **权限请求处理**：拦截远程工具使用权限请求，在本地显示权限确认对话框
4. **连接状态监控**：处理连接建立、重连、断开等状态变化
5. **优雅关闭**：在会话结束时执行清理操作

---

## 功能点目的

### 1. SSH 远程模式检测
```typescript
const isRemoteMode = !!session
```
通过检查 `session` 是否存在来判断是否处于 SSH 远程模式，控制相关 UI 和行为的启用。

### 2. 消息发送与取消
- `sendMessage`: 异步发送消息到远程 SSH 会话
- `cancelRequest`: 发送中断信号取消当前请求
- `disconnect`: 断开 SSH 连接

### 3. 权限请求桥接
将远程的权限请求转换为本地 `ToolUseConfirm` 对象，支持：
- 允许（`onAllow`）：传递可能的修改后输入
- 拒绝（`onReject`）：传递拒绝反馈
- 中止（`onAbort`）：用户主动取消

### 4. 连接状态管理
- **初始化消息去重**：避免 stream-json 模式下重复的 init 消息
- **重连提示**：SSH 断开时显示重连状态给用户
- **错误处理**：连接失败时显示远程 stderr 信息

---

## 具体技术实现

### 关键数据结构

```typescript
// Hook 返回类型
interface UseSSHSessionResult {
  isRemoteMode: boolean
  sendMessage: (content: RemoteMessageContent) => Promise<boolean>
  cancelRequest: () => void
  disconnect: () => void
}

// Props 定义
interface UseSSHSessionProps {
  session: SSHSession | undefined
  setMessages: React.Dispatch<React.SetStateAction<MessageType[]>>
  setIsLoading: (loading: boolean) => void
  setToolUseConfirmQueue: React.Dispatch<React.SetStateAction<ToolUseConfirm[]>>
  tools: Tool[]
}
```

### 核心流程

#### 1. 会话初始化流程
```
useEffect 触发
  ↓
创建 SSHSessionManager
  ↓
注册回调处理器:
  - onMessage: 处理 SDK 消息
  - onPermissionRequest: 处理权限请求
  - onConnected/onReconnecting/onDisconnected: 处理连接状态
  ↓
调用 manager.connect() 建立连接
```

#### 2. 权限请求处理流程
```
收到 onPermissionRequest 回调
  ↓
查找对应工具（本地或创建 stub）
  ↓
创建合成 AssistantMessage
  ↓
构建 PermissionAskDecision
  ↓
组装 ToolUseConfirm 对象
  ↓
添加到确认队列 setToolUseConfirmQueue
  ↓
用户交互后调用 manager.respondToPermissionRequest()
```

#### 3. 消息处理流程
```
收到 onMessage 回调
  ↓
检查是否为 session_end 消息 → 设置 loading=false
  ↓
检查是否为重复 init 消息 → 跳过
  ↓
转换 SDK 消息格式 convertSDKMessage
  ↓
添加到消息列表 setMessages
```

### 关键代码路径

#### 权限请求处理（行 91-157）
```typescript
onPermissionRequest: (request, requestId) => {
  // 1. 查找或创建工具
  const tool = findToolByName(toolsRef.current, request.tool_name) ?? 
               createToolStub(request.tool_name)
  
  // 2. 创建合成消息
  const syntheticMessage = createSyntheticAssistantMessage(request, requestId)
  
  // 3. 构建权限结果
  const permissionResult: PermissionAskDecision = {
    behavior: 'ask',
    message: request.description ?? `${request.tool_name} requires permission`,
    suggestions: request.permission_suggestions,
    blockedPath: request.blocked_path,
  }
  
  // 4. 组装 ToolUseConfirm
  const toolUseConfirm: ToolUseConfirm = {
    assistantMessage: syntheticMessage,
    tool,
    // ... 其他字段
    onAllow(updatedInput) {
      manager.respondToPermissionRequest(requestId, {
        behavior: 'allow',
        updatedInput,
      })
    },
    // ... onReject, onAbort
  }
  
  setToolUseConfirmQueue(q => [...q, toolUseConfirm])
}
```

#### 连接断开处理（行 182-199）
```typescript
onDisconnected: () => {
  const stderr = session.getStderrTail().trim()
  const connected = isConnectedRef.current
  const exitCode = session.proc.exitCode
  
  let msg = connected 
    ? 'Remote session ended.' 
    : 'SSH session failed before connecting.'
    
  // 连接前失败或非正常退出时显示 stderr
  if (stderr && (!connected || exitCode !== 0)) {
    msg += `\nRemote stderr (exit ${exitCode}):\n${stderr}`
  }
  
  void gracefulShutdown(1, 'other', { finalMessage: msg })
}
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `../ssh/createSSHSession.js` | `SSHSession` 类型定义 |
| `../ssh/SSHSessionManager.js` | 会话管理器 |
| `../remote/remotePermissionBridge.js` | 权限桥接工具 |
| `../remote/sdkMessageAdapter.js` | SDK 消息转换 |
| `../Tool.js` | 工具类型和查找 |
| `../utils/gracefulShutdown.js` | 优雅关闭 |

### 外部交互

1. **SSHSessionManager**: 通过 `session.createManager()` 创建，提供：
   - `connect()`: 建立连接
   - `disconnect()`: 断开连接
   - `sendMessage()`: 发送消息
   - `sendInterrupt()`: 发送中断
   - `respondToPermissionRequest()`: 响应权限请求

2. **权限系统**: 与 `ToolUseConfirm` 类型集成，复用本地权限确认 UI

3. **消息系统**: 通过 `convertSDKMessage` 将远程 SDK 消息转换为本地消息格式

---

## 风险、边界与改进建议

### 已知风险

1. **重复 init 消息**: stream-json 模式下每个 turn 都会发送 init，使用 `hasReceivedInitRef` 去重
2. **连接状态竞争**: `isConnectedRef` 用于跟踪连接状态，但在快速重连场景下可能存在竞态
3. **stderr 截断**: `session.getStderrTail()` 可能截断重要错误信息

### 边界情况

1. **权限回调未注册**: 如果用户快速关闭权限对话框，回调可能未正确清理
2. **工具 stub 限制**: 远程 MCP 工具在本地显示为 stub，功能受限
3. **重连消息丢失**: 重连期间的 in-flight 请求会丢失，依赖远程 `--continue` 恢复

### 改进建议

1. **连接状态持久化**: 考虑将连接状态持久化到磁盘，支持跨进程恢复
2. **权限请求超时**: 添加权限请求超时机制，避免无限等待
3. **更丰富的错误信息**: 区分 SSH 连接失败、认证失败、远程进程崩溃等不同错误类型
4. **工具缓存**: 缓存远程工具定义，减少 stub 的使用场景
5. **重连恢复**: 实现请求队列，在重连后自动重发未完成的请求

### 测试关注点

1. SSH 连接失败时的错误处理
2. 权限请求的各种交互路径（允许/拒绝/中止）
3. 重连场景下的消息连续性
4. 并发权限请求的处理
5. 会话结束时的资源清理
