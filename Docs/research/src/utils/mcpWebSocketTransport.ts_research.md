# mcpWebSocketTransport.ts 研究文档

## 场景与职责

本模块实现了 MCP (Model Context Protocol) SDK 的 WebSocket 传输层。核心职责包括：

1. **双运行时支持**：同时支持 Bun（原生 WebSocket）和 Node.js（ws 包）环境
2. **JSON-RPC 消息传输**：在 WebSocket 连接上发送和接收 JSON-RPC 消息
3. **连接生命周期管理**：处理连接建立、启动、关闭和错误恢复
4. **事件驱动架构**：通过回调函数（onmessage/onerror/onclose）与上层集成

该模块是 MCP 客户端与远程 MCP 服务器通过 WebSocket 通信的基础设施。

## 功能点目的

### 1. `WebSocketTransport` 类
- **目的**：实现 MCP SDK 的 `Transport` 接口
- **功能**：
  - 包装底层 WebSocket 连接
  - 处理 Bun 和 Node.js 的 API 差异
  - 提供统一的 JSON-RPC 消息接口

### 2. 运行时检测与适配
- **目的**：自动检测运行环境（Bun vs Node.js）并适配相应 API
- **检测方式**：`typeof Bun !== 'undefined'`
- **差异处理**：
  - Bun：使用 `addEventListener`/`removeEventListener`
  - Node.js：使用 `on`/`off` 方法

### 3. 连接状态管理
- **目的**：确保在连接就绪后才允许消息传输
- **状态常量**：
  - `WS_CONNECTING = 0`
  - `WS_OPEN = 1`
- **启动流程**：等待 `opened` Promise 完成，验证 `readyState`

### 4. 消息处理
- **目的**：解析和验证 JSON-RPC 消息
- **流程**：
  - 接收原始数据（Bun: `MessageEvent.data`，Node: `Buffer`）
  - 使用 `jsonParse` 解析 JSON
  - 使用 `JSONRPCMessageSchema` 验证结构
  - 调用 `onmessage` 回调

### 5. 错误处理与诊断
- **目的**：统一处理 WebSocket 错误并记录诊断信息
- **实现**：
  - 使用 `logForDiagnosticsNoPII` 记录错误事件
  - 通过 `toError` 统一错误格式
  - 调用 `onerror` 回调通知上层

## 具体技术实现

### 关键流程

```
创建 WebSocketTransport
    ↓
检测运行时（Bun/Node）
    ↓
设置连接就绪 Promise
    ↓
附加事件监听器
    ↓
等待 start() 调用
    ↓
验证连接状态
    ↓
开始接收/发送消息
```

### 数据结构

```typescript
// WebSocket 就绪状态常量
const WS_CONNECTING = 0
const WS_OPEN = 1

// 通用 WebSocket 接口（Bun 和 Node 的最小交集）
type WebSocketLike = {
  readonly readyState: number
  close(): void
  send(data: string): void
}

// 传输类公共回调
onclose?: () => void
onerror?: (error: Error) => void
onmessage?: (message: JSONRPCMessage) => void
```

### 运行时适配细节

| 特性 | Bun (原生) | Node.js (ws) |
|------|------------|--------------|
| 事件监听 | `addEventListener(event, handler)` | `on(event, handler)` |
| 事件移除 | `removeEventListener(event, handler)` | `off(event, handler)` |
| 消息接收 | `MessageEvent.data` | `Buffer` |
| 发送回调 | 同步（无回调） | 异步（回调函数） |
| 错误事件 | `Event` 对象 | `Error` 对象 |

### 关键代码路径

1. **构造函数**：
   ```typescript
   constructor(private ws: WebSocketLike) {
     this.opened = new Promise((resolve, reject) => {
       // 根据运行时设置 open/error 处理器
     })
     // 附加持久消息处理器
   }
   ```

2. **消息发送**：
   ```typescript
   async send(message: JSONRPCMessage): Promise<void> {
     if (this.ws.readyState !== WS_OPEN) throw Error
     const json = jsonStringify(message)
     if (this.isBun) {
       this.ws.send(json)  // 同步
     } else {
       await new Promise((resolve, reject) => {
         ws.send(json, error => error ? reject(error) : resolve())
       })
     }
   }
   ```

3. **关闭清理**：
   ```typescript
   private handleCloseCleanup(): void {
     this.onclose?.()
     // 移除所有事件监听器（根据运行时选择方法）
   }
   ```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk/shared/transport.js` | Transport 接口定义 |
| `@modelcontextprotocol/sdk/types.js` | JSON-RPC 类型和验证 |
| `ws` (类型) | Node.js WebSocket 类型 |
| `./diagLogs.js` | 诊断日志（无 PII） |
| `./errors.js` | 错误转换工具 |
| `./slowOperations.js` | JSON 解析/序列化 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/services/mcp/client.ts` | MCP 客户端创建 WebSocket 传输 |

### 外部依赖

- `@modelcontextprotocol/sdk`：MCP 官方 SDK
- `ws`：Node.js WebSocket 库（运行时依赖）

## 风险、边界与改进建议

### 已知风险

1. **运行时检测可靠性**
   - 检测方式：`typeof Bun !== 'undefined'`
   - 风险：未来 Bun 版本可能改变全局变量
   - 缓解：使用官方推荐的检测方式

2. **WebSocket 状态竞争**
   - 风险：`send()` 和 `close()` 之间可能出现状态不一致
   - 现状：每次操作前检查 `readyState`
   - 潜在问题：异步操作间隙状态可能变化

3. **消息解析失败**
   - 风险：收到非 JSON-RPC 格式的消息
   - 处理：解析失败调用 `handleError`，不抛出
   - 潜在问题：恶意服务器可能发送无效消息导致错误循环

4. **内存泄漏**
   - 风险：事件监听器未正确移除
   - 缓解：`handleCloseCleanup` 统一移除监听器
   - 边界：外部传入的 WebSocket 对象不受控制

### 边界情况

| 场景 | 行为 |
|------|------|
| 连接未就绪时调用 `start()` | 等待 `opened` Promise |
| 连接未就绪时调用 `send()` | 抛出错误 "WebSocket is not open" |
| 重复调用 `start()` | 抛出错误 "Start can only be called once" |
| 收到无效 JSON | 解析错误，调用 `onerror` |
| 收到非 JSON-RPC 格式消息 | 验证失败，调用 `onerror` |
| WebSocket 异常关闭 | 调用 `onclose`，清理监听器 |
| Bun 发送失败 | 抛出错误，调用 `handleError` |
| Node 发送回调错误 | Promise reject，调用 `handleError` |

### 改进建议

1. **连接健康检查**
   - 当前：依赖 WebSocket 的 `readyState`
   - 建议：添加心跳机制（ping/pong）检测连接活性
   - 实现：定期发送心跳，超时未响应则标记为断开

2. **重连机制**
   - 当前：连接断开即通知上层，无自动重连
   - 建议：添加指数退避重连策略
   - 考虑：重连后需要重新初始化 MCP 会话

3. **消息队列**
   - 当前：`send()` 在连接未就绪时直接报错
   - 建议：添加发送队列，连接就绪后自动发送
   - 限制：设置队列大小上限，防止内存溢出

4. **更精确的运行时检测**
   - 当前：简单的 `typeof Bun` 检查
   - 建议：多特征检测（`process.versions.bun` 等）
   - 备选：通过构建时注入 `__RUNTIME__` 常量

5. **性能优化**
   - 当前：每个消息都进行 JSON 验证
   - 建议：
     - 生产环境可跳过验证（信任 SDK）
     - 使用更快的 JSON 解析库（如 `simdjson`）

6. **可观测性增强**
   - 当前：仅记录错误事件
   - 建议：
     - 记录消息流量统计（发送/接收字节数）
     - 记录连接持续时间
     - 记录消息延迟（发送→确认）

7. **流式消息支持**
   - 当前：假设每个 WebSocket 消息是一个完整的 JSON-RPC 消息
   - 建议：处理消息分片（TCP 粘包/拆包）
   - 实现：添加消息边界解析器
