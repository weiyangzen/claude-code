# SdkControlTransport.ts 研究文档

## 场景与职责

`SdkControlTransport.ts` 实现了 **SDK MCP 传输桥接层**，用于解决 CLI 进程与 SDK 进程之间的 MCP 通信问题。这是 Claude Code 特有的架构需求：

### 架构背景

```
┌─────────────────┐                      ┌─────────────────┐
│   CLI Process   │ ←── stdin/stdout ──→ │   SDK Process   │
│                 │                      │                 │
│  MCP Client ────┼── SdkControlClient ──┼──→ MCP Server   │
│  (Tool Caller)  │    Transport         │   (In-Process)  │
└─────────────────┘                      └─────────────────┘
```

与常规 MCP 服务器作为独立进程运行不同，SDK MCP 服务器运行在 SDK 进程内部。这需要特殊的传输机制来桥接两个进程之间的通信。

### 核心职责

1. **CLI 侧传输** (`SdkControlClientTransport`): 将 MCP 客户端的请求转换为控制消息发送到 SDK
2. **SDK 侧传输** (`SdkControlServerTransport`): 在 SDK 进程中接收控制消息并转发给 MCP 服务器
3. **请求-响应关联**: 维护消息 ID 确保请求和响应正确匹配

## 功能点目的

### 双端传输设计

| 类 | 运行位置 | 职责 |
|----|----------|------|
| `SdkControlClientTransport` | CLI 进程 | 将 MCP JSONRPC 消息包装为控制请求，通过 `sendMcpMessage` 回调发送到 SDK |
| `SdkControlServerTransport` | SDK 进程 | 接收控制请求，解包后转发给 MCP 服务器，通过回调返回响应 |

### 消息流

#### CLI → SDK (工具调用请求)
```
1. CLI's MCP Client 调用工具
2. 发送 JSONRPC request 到 SdkControlClientTransport
3. Transport 包装消息为 control request (含 server_name, request_id)
4. Control request 通过 stdout 发送到 SDK 进程
5. SDK's StructuredIO 接收并路由到 SdkControlServerTransport
6. Transport 解包并转发给 MCP Server
7. MCP Server 处理并返回响应
8. 响应通过 callback 返回给 CLI
```

#### SDK → CLI (通知/响应)
```
1. Query 接收 control request 含 MCP 消息
2. 调用 SdkControlServerTransport.onmessage
3. MCP Server 处理消息并调用 transport.send() 返回响应
4. Transport 调用 sendMcpMessage callback 发送响应
5. Query's callback 解析 pending promise
6. 响应返回给 CLI 完成 control request
```

## 具体技术实现

### 关键类型定义

```typescript
/**
 * 发送 MCP 消息并获取响应的回调函数
 */
export type SendMcpMessageCallback = (
  serverName: string,
  message: JSONRPCMessage,
) => Promise<JSONRPCMessage>
```

### SdkControlClientTransport (CLI 侧)

```typescript
export class SdkControlClientTransport implements Transport {
  private isClosed = false

  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void

  constructor(
    private serverName: string,
    private sendMcpMessage: SendMcpMessageCallback,
  ) {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) {
      throw new Error('Transport is closed')
    }
    // 发送消息并等待响应
    const response = await this.sendMcpMessage(this.serverName, message)
    // 将响应传回 MCP 客户端
    if (this.onmessage) {
      this.onmessage(response)
    }
  }
}
```

**关键设计点**:
- `send()` 方法阻塞等待响应，模拟同步请求-响应模式
- 响应通过 `onmessage` 回调返回给 MCP 客户端
- 支持多个 SDK MCP 服务器同时运行（通过 `serverName` 路由）

### SdkControlServerTransport (SDK 侧)

```typescript
export class SdkControlServerTransport implements Transport {
  private isClosed = false

  constructor(private sendMcpMessage: (message: JSONRPCMessage) => void) {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) {
      throw new Error('Transport is closed')
    }
    // 简单地将响应通过 callback 返回
    this.sendMcpMessage(message)
  }
}
```

**关键设计点**:
- 作为简单的透传层，将消息转发给 MCP 服务器
- 实际的请求-响应关联由 Query 层处理
- 消息 ID 在整个流程中保持不变以确保正确关联

## 关键代码路径与文件引用

### 本文件关键代码

| 行号 | 代码 | 说明 |
|------|------|------|
| 1-37 | 文件头注释 | 详细的架构和消息流文档 |
| 45-48 | `SendMcpMessageCallback` 类型 | 核心回调类型定义 |
| 60-95 | `SdkControlClientTransport` | CLI 侧传输实现 |
| 109-136 | `SdkControlServerTransport` | SDK 侧传输实现 |

### 调用方文件

| 文件路径 | 使用场景 |
|----------|----------|
| `src/services/mcp/client.ts:137` | 导入 `SdkControlClientTransport` |
| `src/services/mcp/client.ts:3278` | 创建 `SdkControlClientTransport` 实例 |

### 使用代码示例 (client.ts)

```typescript
// 在 connectToSdkMcpServers 函数中
const transport = new SdkControlClientTransport(name, sendMcpMessage)

const client = new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version: MACRO.VERSION ?? 'unknown',
    description: "Anthropic's agentic coding tool",
    websiteUrl: PRODUCT_URL,
  },
  {
    capabilities: {},
  },
)

await client.connect(transport)
```

### 相关文件

| 文件路径 | 说明 |
|----------|------|
| `src/cli/print.ts:1259,1271` | 处理 SDK MCP 的 elicitation 流程 |
| `src/services/mcp/vscodeSdkMcp.ts` | VSCode SDK MCP 特殊处理 |

## 依赖与外部交互

### 外部依赖

```typescript
// MCP SDK 类型
import type { Transport } from '@modelcontextprotocol/sdk/shared/transport.js'
import type { JSONRPCMessage } from '@modelcontextprotocol/sdk/types.js'
```

### 架构依赖

| 组件 | 职责 |
|------|------|
| `StructuredIO` (SDK 侧) | 接收和路由控制消息 |
| `Query` (SDK 侧) | 跟踪 pending 请求，处理异步流 |
| `Client` (MCP SDK) | MCP 客户端实现 |

### 集成点

```
┌─────────────────────────────────────────────────────────────┐
│                         CLI Process                         │
│  ┌──────────────┐    ┌─────────────────────┐               │
│  │ MCP Client   │───→│ SdkControlClient    │               │
│  │ (SDK 提供)   │←───│ Transport           │               │
│  └──────────────┘    └──────────┬──────────┘               │
│                                 │ sendMcpMessage            │
│                                 ↓                           │
│                          ┌────────────┐                     │
│                          │  stdout    │                     │
│                          └────────────┘                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                         SDK Process                         │
│                          ┌────────────┐                     │
│                          │  stdin     │                     │
│                          └─────┬──────┘                     │
│                                │                            │
│  ┌─────────────────────┐      │ StructuredIO               │
│  │ SdkControlServer    │←─────┘                            │
│  │ Transport           │                                   │
│  └──────────┬──────────┘                                   │
│             │                                               │
│  ┌──────────↓──────────┐                                   │
│  │ MCP Server          │                                   │
│  │ (In-Process)        │                                   │
│  └─────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

## 风险、边界与改进建议

### 已知风险

1. **进程间通信失败**:
   - 如果 stdout/stdin 通道断开，所有 MCP 通信将失败
   - 需要检测连接中断并提供降级方案

2. **消息 ID 冲突**:
   - 多个 SDK MCP 服务器同时运行时，消息 ID 可能冲突
   - 当前通过 `serverName` 路由区分，但需要确保 ID 全局唯一

3. **超时处理**:
   - 当前实现没有内置超时机制
   - 长时间运行的 MCP 工具可能导致请求挂起

4. **背压处理**:
   - 高频消息场景下可能出现内存累积
   - 需要流量控制机制

### 边界情况

| 场景 | 行为 |
|------|------|
| 对已关闭传输调用 `send()` | 抛出 `'Transport is closed'` 错误 |
| `sendMcpMessage` 回调抛出错误 | 错误向上传播，MCP 客户端收到错误 |
| 响应在传输关闭后到达 | 响应被丢弃（无处理） |
| 多服务器并发请求 | 通过 `serverName` 正确路由 |

### 改进建议

1. **添加超时机制**:
   ```typescript
   async send(message: JSONRPCMessage, timeoutMs = 30000): Promise<void> {
     const timeout = setTimeout(() => {
       throw new Error('MCP request timeout')
     }, timeoutMs)
     
     try {
       const response = await this.sendMcpMessage(this.serverName, message)
       // ...
     } finally {
       clearTimeout(timeout)
     }
   }
   ```

2. **增强错误分类**:
   - 区分网络错误、协议错误、服务器错误
   - 提供更有针对性的错误信息和恢复建议

3. **添加连接健康检查**:
   ```typescript
   async healthCheck(): Promise<boolean> {
     try {
       await this.send({ method: 'ping', jsonrpc: '2.0', id: 'health' })
       return true
     } catch {
       return false
     }
   }
   ```

4. **性能优化**:
   - 实现消息批处理减少 IPC 开销
   - 添加消息压缩支持（大数据传输场景）

5. **可观测性**:
   - 添加消息计数和延迟指标
   - 记录消息流日志便于调试

6. **优雅降级**:
   - 当 SDK MCP 不可用时，提供本地备用方案
   - 支持 MCP 服务器的热切换

### 测试建议

- 测试进程间通信中断的恢复
- 测试多服务器并发场景
- 测试大消息传输性能
- 测试超时和错误处理
- 测试与真实 MCP 服务器的集成
- 测试长时间运行的工具调用
