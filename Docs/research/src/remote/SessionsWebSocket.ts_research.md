# SessionsWebSocket.ts 研究文档

## 场景与职责

`SessionsWebSocket` 是 Claude Code CLI 中专门用于管理与 Anthropic 远程会话服务 WebSocket 连接的底层类。它是 `RemoteSessionManager` 的依赖组件，负责：

1. **WebSocket 连接管理**：建立、维护和关闭与远程会话的 WebSocket 连接
2. **双运行时支持**：同时支持 Bun（原生 WebSocket）和 Node.js（ws 包）
3. **自动重连机制**：实现智能重连策略，处理临时网络故障
4. **心跳保活**：定期发送 ping 消息维持连接
5. **控制消息传输**：发送控制请求和响应（权限、中断等）

该模块是远程会话通信的基础设施层，处理所有 WebSocket 相关的底层细节。

## 功能点目的

### 1. 连接配置常量
```typescript
const RECONNECT_DELAY_MS = 2000           // 基础重连延迟
const MAX_RECONNECT_ATTEMPTS = 5          // 最大重连次数
const PING_INTERVAL_MS = 30000            // 心跳间隔（30秒）
const MAX_SESSION_NOT_FOUND_RETRIES = 3   // 4001 错误特殊重试次数
```

### 2. 永久关闭代码
```typescript
const PERMANENT_CLOSE_CODES = new Set([4003]) // unauthorized - 不重连
```

### 3. WebSocket 状态
```typescript
type WebSocketState = 'connecting' | 'connected' | 'closed'
```

### 4. 回调接口 (`SessionsWebSocketCallbacks`)
- `onMessage`: 接收到消息时调用
- `onClose`: 连接永久关闭时调用
- `onError`: 发生错误时调用
- `onConnected`: 连接成功时调用
- `onReconnecting`: 开始重连时调用

## 具体技术实现

### 关键流程

#### 1. 连接建立流程 (`connect()`)

**Bun 运行时路径**:
```
1. 检查 state !== 'connecting'（防止重复连接）
2. 构造 WebSocket URL: wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe?organization_uuid={orgUuid}
3. 获取 OAuth token 构建 headers
4. 创建 Bun WebSocket: new globalThis.WebSocket(url, { headers, proxy, tls })
5. 绑定事件监听器: open, message, error, close, pong
6. 状态设置为 'connecting'
```

**Node.js 运行时路径**:
```
1-3. 同上
4. 动态导入 'ws' 包
5. 创建 WS 实例: new WS(url, { headers, agent, ...tlsOptions })
6. 绑定事件处理器: .on('open'), .on('message'), etc.
```

#### 2. 消息处理流程 (`handleMessage()`)
```
1. 使用 jsonParse() 解析 JSON 数据
2. 调用 isSessionsMessage() 类型守卫验证消息格式
3. 有效消息: 调用 callbacks.onMessage()
4. 无效消息: 记录调试日志，静默丢弃
5. 解析错误: 记录错误日志
```

#### 3. 连接关闭处理 (`handleClose()`)

关闭代码处理逻辑：

| 关闭代码 | 含义 | 处理方式 |
|---------|------|---------|
| 4003 | Unauthorized | 永久关闭，不重连 |
| 4001 | Session not found | 有限重试（最多3次），递增延迟 |
| 其他 | 网络/服务器错误 | 标准重连（最多5次），固定延迟 |

**重连决策流程**:
```
1. 停止 ping 定时器
2. 检查 state === 'closed'（已处理则返回）
3. 检查永久关闭代码 → 调用 onClose，返回
4. 检查 4001 代码 → 递增计数器，若超限则关闭，否则延迟重连
5. 检查 previousState === 'connected' 且重连次数 < 5 → 延迟重连
6. 否则 → 永久关闭
```

#### 4. 心跳机制 (`startPingInterval()` / `stopPingInterval()`)
```
每 30 秒:
  如果 state === 'connected':
    调用 ws.ping()（如果支持）
    忽略 ping 错误（由 close handler 处理）
```

#### 5. 控制消息发送

**发送控制响应** (`sendControlResponse`):
```typescript
// 检查连接状态
if (!ws || state !== 'connected') { logError; return }
// 序列化并发送
ws.send(jsonStringify(response))
```

**发送控制请求** (`sendControlRequest`):
```typescript
// 构造完整请求对象
const controlRequest = {
  type: 'control_request',
  request_id: randomUUID(),  // 生成唯一请求ID
  request,                   // 内部请求数据
}
ws.send(jsonStringify(controlRequest))
```

### 数据结构

#### 内部状态
```typescript
private ws: WebSocketLike | null = null
private state: WebSocketState = 'closed'
private reconnectAttempts = 0           // 通用重连计数器
private sessionNotFoundRetries = 0      // 4001 专用计数器
private pingInterval: NodeJS.Timeout | null = null
private reconnectTimer: NodeJS.Timeout | null = null
```

#### WebSocket 抽象接口
```typescript
type WebSocketLike = {
  close(): void
  send(data: string): void
  ping?(): void  // Bun & ws 都支持
}
```

#### 消息类型守卫
```typescript
function isSessionsMessage(value: unknown): value is SessionsMessage {
  if (typeof value !== 'object' || value === null || !('type' in value)) {
    return false
  }
  // 接受任何带有 string type 字段的消息
  // 下游处理器决定如何处理未知类型
  return typeof value.type === 'string'
}
```

### 协议与命令

#### WebSocket 连接协议
```
URL: wss://{BASE_API_URL}/v1/sessions/ws/{sessionId}/subscribe?organization_uuid={orgUuid}
Headers:
  Authorization: Bearer {accessToken}
  anthropic-version: 2023-06-01
```

#### 控制请求格式
```typescript
{
  type: 'control_request',
  request_id: string,      // UUID v4
  request: {
    subtype: 'interrupt' | 'can_use_tool' | ...,
    // ...  subtype-specific fields
  }
}
```

#### 控制响应格式
```typescript
{
  type: 'control_response',
  response: {
    subtype: 'success' | 'error',
    request_id: string,
    // success: response?: object
    // error: error: string
  }
}
```

## 关键代码路径与文件引用

### 核心文件
- **本文件**: `src/remote/SessionsWebSocket.ts` (404 lines)

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/constants/oauth.ts` | `getOauthConfig()` 获取 API 基础 URL |
| `src/entrypoints/agentSdkTypes.ts` | `SDKMessage` 类型 |
| `src/entrypoints/sdk/controlTypes.ts` | 控制消息类型 |
| `src/utils/debug.ts` | `logForDebugging` 调试日志 |
| `src/utils/errors.ts` | `errorMessage` 错误格式化 |
| `src/utils/log.ts` | `logError` 错误日志 |
| `src/utils/mtls.ts` | `getWebSocketTLSOptions()` mTLS 配置 |
| `src/utils/proxy.ts` | `getWebSocketProxyAgent/Url()` 代理配置 |
| `src/utils/slowOperations.ts` | `jsonParse/jsonStringify` JSON 操作 |

### 调用方文件
| 文件路径 | 用途 |
|---------|------|
| `src/remote/RemoteSessionManager.ts` | 主要调用方，管理远程会话 |

### 关键函数引用路径
```
connect()
  ├── getOauthConfig() [src/constants/oauth.ts]
  ├── getAccessToken() (构造函数传入)
  ├── getWebSocketProxyUrl() / getWebSocketProxyAgent() [src/utils/proxy.ts]
  ├── getWebSocketTLSOptions() [src/utils/mtls.ts]
  └── Bun WebSocket 或 ws 包

handleClose(code)
  ├── 检查 PERMANENT_CLOSE_CODES
  ├── scheduleReconnect() → setTimeout() → connect()
  └── callbacks.onClose() / onReconnecting()

sendControlRequest/Response()
  └── jsonStringify() [src/utils/slowOperations.ts]
      └── ws.send()
```

## 依赖与外部交互

### 运行时依赖

#### Bun 运行时
- **原生 WebSocket**: `globalThis.WebSocket`
- **特性**: 原生支持 headers、proxy、tls 选项

#### Node.js 运行时
- **ws 包**: 动态导入 `import('ws')`
- **特性**: 通过 `agent` 选项支持代理

### 网络配置依赖

#### mTLS 支持（通过 `mtls.ts`）
- `CLAUDE_CODE_CLIENT_CERT`: 客户端证书路径
- `CLAUDE_CODE_CLIENT_KEY`: 客户端密钥路径
- `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE`: 密钥密码

#### 代理支持（通过 `proxy.ts`）
- `HTTPS_PROXY/HTTP_PROXY`: 代理 URL
- `NO_PROXY`: 绕过代理的主机列表
- `CLAUDE_CODE_PROXY_RESOLVES_HOSTS`: 代理解析主机名

### 外部服务交互

#### Anthropic WebSocket 服务
```
Endpoint: wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe
Query: organization_uuid={orgUuid}
Auth: Bearer token via header
```

#### 连接生命周期
1. **建立**: TLS 握手 → WebSocket 升级 → 认证
2. **保活**: 30s 间隔 ping/pong
3. **重连**: 根据关闭代码决定是否重连
4. **终止**: 永久关闭代码或重试耗尽

## 风险、边界与改进建议

### 已知风险

#### 1. 4001 (Session Not Found) 竞态条件
- **风险**: 会话 compaction 期间服务器可能短暂认为会话不存在
- **缓解**: 专门的重试逻辑（最多3次，递增延迟）
- **潜在问题**: 如果 compaction 超过 3*2s=6s，仍会导致连接失败

#### 2. 重连风暴
- **风险**: 大量客户端同时重连可能压垮服务器
- **缓解**: 固定 2s 延迟，无抖动（jitter）
- **潜在问题**: 缺乏指数退避和随机化

#### 3. 消息丢失
- **风险**: 重连期间的消息可能丢失
- **缓解**: 上层 `useRemoteSession` 有超时检测
- **潜在问题**: 无消息队列/缓冲机制

#### 4. Bun/Node 行为差异
- **风险**: 两种运行时的 WebSocket 实现有细微差异
- **缓解**: 统一的 `WebSocketLike` 接口抽象
- **潜在问题**: 边缘场景测试覆盖可能不足

### 边界条件

| 边界场景 | 行为 |
|---------|------|
| 重复调用 connect() | 检查 state，如果 connecting 则直接返回 |
| 发送消息时未连接 | 记录错误日志，静默丢弃 |
| 收到非 JSON 消息 | 记录错误，不中断连接 |
| 收到无 type 字段的消息 | 记录调试日志，静默丢弃 |
| close() 后事件触发 | state 检查防止重复处理 |
| reconnectTimer 存在时调用 reconnect() | clearTimeout 后重新调度 |

### 改进建议

#### 1. 重连策略增强
当前实现：
```typescript
// 固定延迟，无抖动
scheduleReconnect(RECONNECT_DELAY_MS, label)  // 2000ms
```

建议改进：
```typescript
// 指数退避 + 随机抖动
const delay = Math.min(
  RECONNECT_DELAY_MS * Math.pow(2, attempts),
  MAX_RECONNECT_DELAY_MS
) + Math.random() * JITTER_MS
```

#### 2. 消息序列号/确认机制
- 添加消息序列号跟踪
- 实现消息确认（ACK）机制
- 重连后请求丢失的消息

#### 3. 连接健康检查
- 添加应用层心跳（除了 WebSocket ping）
- 检测半开连接（half-open connections）
- 实现连接质量指标收集

#### 4. 错误分类优化
当前仅区分 4001/4003/其他，建议：
- 分类更多关闭代码
- 根据错误类型调整重连策略
- 提供用户友好的错误提示

#### 5. 配置外部化
当前常量为硬编码：
```typescript
const RECONNECT_DELAY_MS = 2000
const MAX_RECONNECT_ATTEMPTS = 5
```

建议：
- 支持环境变量覆盖
- 根据网络条件动态调整
- 提供配置接口给上层

#### 6. 测试覆盖
建议添加：
- 模拟不同关闭代码的单元测试
- Bun/Node 双运行时的集成测试
- 重连策略的压力测试
- 代理/mTLS 配置测试

### 代码质量观察

#### 优点
- 双运行时支持良好，抽象层清晰
- 4001 特殊处理体现了对业务场景的理解
- 调试日志丰富，便于问题排查
- 类型定义完整，TypeScript 支持良好

#### 潜在问题
- `isSessionsMessage` 过于宽松，可能放过无效消息
- 重连逻辑分散在多个条件分支中，可读性一般
- 无连接质量指标（延迟、重连频率等）
- `close()` 方法的 event handler 清理注释提到 race condition

#### 性能考虑
- `jsonParse/jsonStringify` 使用包装版本（带性能监控）
- 消息处理是同步的，可能阻塞事件循环
- ping 间隔 30s 是合理的平衡点
