# InProcessTransport.ts 研究文档

## 场景与职责

`InProcessTransport.ts` 提供了一种**进程内 MCP 传输机制**，用于在同一个 Node.js 进程中运行 MCP 服务器和客户端，而无需生成子进程。这在以下场景特别有用：

1. **Chrome MCP 服务器** (`claude-for-chrome-mcp`): 避免生成约 325MB 的子进程开销
2. **Computer Use MCP 服务器** (`CHICAGO_MCP` 特性): 在进程内运行计算机控制功能
3. **测试和开发**: 快速验证 MCP 服务器实现而无需进程间通信

## 功能点目的

### 核心功能

| 功能 | 目的 |
|------|------|
| `InProcessTransport` 类 | 实现 MCP SDK 的 `Transport` 接口，提供进程内消息传递 |
| `createLinkedTransportPair()` | 创建一对相互连接的传输对象，分别用于客户端和服务器端 |
| 双向消息传递 | 一方的 `send()` 调用会触发另一方的 `onmessage` 回调 |
| 同步关闭处理 | 一方 `close()` 会同时触发双方的 `onclose` 回调 |

### 设计特点

- **异步消息传递**: 使用 `queueMicrotask` 避免同步请求/响应循环导致的栈深度问题
- **状态管理**: 跟踪 `closed` 状态防止对已关闭传输进行操作
- **Peer 模式**: 通过 `_setPeer()` 方法建立双向连接关系

## 具体技术实现

### 关键数据结构

```typescript
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined  // 对端传输引用
  private closed = false                          // 关闭状态标志

  // Transport 接口必需的事件处理器
  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void
}
```

### 核心流程

#### 1. 消息发送流程 (`send`)
```
send(message) 
  → 检查 closed 状态
  → queueMicrotask(() => peer.onmessage?.(message))
  → 异步触发对端的 onmessage 回调
```

#### 2. 连接关闭流程 (`close`)
```
close()
  → 标记自身 closed = true
  → 调用自身 onclose?.()
  → 如果对端未关闭，标记对端 closed = true 并调用其 onclose?.()
```

#### 3. 配对创建流程
```
createLinkedTransportPair()
  → 创建 transport A
  → 创建 transport B  
  → A._setPeer(B)
  → B._setPeer(A)
  → 返回 [A, B] 作为 [clientTransport, serverTransport]
```

## 关键代码路径与文件引用

### 本文件关键代码

| 行号 | 代码 | 说明 |
|------|------|------|
| 11-49 | `InProcessTransport` 类 | 核心传输实现 |
| 26-35 | `send()` 方法 | 异步消息发送 |
| 37-48 | `close()` 方法 | 双向关闭处理 |
| 57-63 | `createLinkedTransportPair()` | 工厂函数创建配对 |

### 调用方文件

| 文件路径 | 使用场景 |
|----------|----------|
| `src/services/mcp/client.ts:916-923` | Chrome MCP 服务器进程内运行 |
| `src/services/mcp/client.ts:936-943` | Computer Use MCP 服务器进程内运行 |

### 调用代码示例

```typescript
// client.ts 中的使用方式
const { createLinkedTransportPair } = await import('./InProcessTransport.js')
const [clientTransport, serverTransport] = createLinkedTransportPair()
await inProcessServer.connect(serverTransport)
transport = clientTransport
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `Transport` 类型 | `@modelcontextprotocol/sdk/shared/transport.js` | MCP SDK 传输接口 |
| `JSONRPCMessage` 类型 | `@modelcontextprotocol/sdk/types.js` | MCP 协议消息类型 |

### 被依赖关系

- `client.ts` 动态导入此模块用于特殊 MCP 服务器的进程内运行
- 仅在检测到特定服务器类型时加载（Chrome MCP、Computer Use MCP）

## 风险、边界与改进建议

### 已知风险

1. **内存泄漏风险**: 
   - 如果 `close()` 未被正确调用，peer 引用可能阻止垃圾回收
   - 建议添加弱引用或显式清理机制

2. **栈溢出防护**:
   - 已实现 `queueMicrotask` 避免同步调用链过长
   - 但在极高频率消息场景下仍需注意

3. **错误处理**:
   - 当前 `onerror` 处理器未被主动调用
   - 消息发送失败仅抛出异常，无错误传播机制

### 边界情况

| 场景 | 行为 |
|------|------|
| 对已关闭传输调用 `send()` | 抛出 `'Transport is closed'` 错误 |
| 重复调用 `close()` | 第二次及以后调用直接返回，无副作用 |
| 单向关闭 | 关闭一方会自动触发对方的 `onclose` |
| 未配对使用 | `peer` 为 `undefined`，消息无法送达 |

### 改进建议

1. **增强错误处理**:
   ```typescript
   // 建议添加错误传播
   async send(message: JSONRPCMessage): Promise<void> {
     if (this.closed) {
       const error = new Error('Transport is closed')
       this.onerror?.(error)
       throw error
     }
     // ...
   }
   ```

2. **添加健康检查**:
   - 暴露 `isConnected` 或 `isClosed` 只读属性
   - 便于调用方检查传输状态

3. **性能优化**:
   - 考虑使用 `MessageChannel` API 替代 microtask 队列
   - 可获得更好的性能和更清晰的语义

4. **类型安全**:
   - `_setPeer` 标记为 `@internal` 是良好的做法
   - 可考虑使用 Symbol 进一步限制外部访问

### 测试建议

- 测试双向关闭的时序一致性
- 测试高频消息下的栈深度
- 测试未配对传输的错误行为
- 测试与真实 MCP 客户端/服务器的集成
