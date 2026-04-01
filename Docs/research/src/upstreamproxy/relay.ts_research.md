# src/upstreamproxy/relay.ts 深度研究文档

## 场景与职责

### 部署场景

`relay.ts` 是 CCR (Claude Code Remote) 上游代理体系中的**本地传输中继层**，运行于容器内部的 CLI 进程内。其核心任务是在受限网络环境中，为 curl、gh、kubectl、Python SDK 等外部工具提供一条“HTTP CONNECT → WebSocket 隧道 → CCR 上游代理端点”的透明转发通道。

| 场景 | 说明 |
|------|------|
| **CCR 容器内网受限** | 容器无法直接出站，必须通过 CCR 提供的上游代理访问外部服务 |
| **GKE L7 Ingress** | CCR 入口为 GKE L7，不支持原生 TCP CONNECT，只能走 WebSocket（已有 sessions/tunnel 先例）|
| **凭证注入** | 上游代理端点 MITM TLS，并在转发前注入组织级凭证（如 DD-API-KEY）|
| **双运行时兼容** | 开发/测试使用 Bun，生产容器使用 Node.js，代码需同时兼容两者 |

### 核心职责

1. **本地 TCP 监听**：在 `127.0.0.1` 的随机 ephemeral 端口上启动 TCP 服务器，接受来自子进程的 HTTP CONNECT 请求。
2. **协议转换**：将 HTTP CONNECT 请求以及后续的双向裸字节流，封装为 `UpstreamProxyChunk` protobuf 消息，通过 WebSocket 二进制帧发送到服务端。
3. **双运行时适配**：
   - **Bun 路径**：使用 `Bun.listen()`，需手动处理 `sock.write()` 的部分写入（partial write）和 `drain` 刷新。
   - **Node 路径**：使用 `node:net.createServer()`，`sock.write()` 内部自带缓冲，无需手动队列。
4. **连接生命周期管理**：解析 CONNECT 请求头、建立 WS 隧道、处理握手期间的并发数据、发送心跳保活、清理关闭连接。

### 设计原则

> **零依赖热路径**：为避免在数据转发路径引入 `protobufjs` 等重型依赖，`UpstreamProxyChunk` 的编解码采用 20 行手写代码完成。
> **Fail-open 的下游表现**：如果 WebSocket 握手失败或隧道中断，在尚未发送 `200 Connection Established` 之前，向客户端返回 `502 Bad Gateway` 并关闭连接，避免子进程挂起。

---

## 功能点目的

### 功能总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  子进程 (curl / gh / python)                                                │
│        │                                                                    │
│        ▼ HTTP CONNECT                                                       │
│  ┌─────────────────┐                                                        │
│  │ 127.0.0.1:port  │  ◄── relay.ts 监听的本地 TCP 端口                      │
│  └─────────────────┘                                                        │
│        │                                                                    │
│        ▼ 解析 CONNECT 请求头                                                 │
│  ┌─────────────────┐                                                        │
│  │  WebSocket 升级  │  ◄── 携带 Bearer Token 的 WS 握手                      │
│  │ /v1/code/upstreamproxy/ws                                                │
│  └─────────────────┘                                                        │
│        │                                                                    │
│        ▼ Protobuf 二进制帧 (UpstreamProxyChunk)                              │
│  ┌─────────────────┐                                                        │
│  │  CCR UpstreamProxy Endpoint                                               │
│  │  (服务端终止隧道、MITM、注入凭证、转发到真实上游)                           │
│  └─────────────────┘                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 各功能点详细说明

#### 1. Protobuf 手工编解码 (`encodeChunk` / `decodeChunk`)

- **消息定义**：`message UpstreamProxyChunk { bytes data = 1; }`
- **Wire 格式**：`tag(0x0a) + varint(length) + raw_bytes`
- **目的**：避免引入 `protobufjs` 运行时依赖，减少 bundle 体积和启动开销。
- **容错**：`decodeChunk` 对零长度 chunk 返回空 `Uint8Array`，作为服务端可忽略的应用层 keepalive。

#### 2. CONNECT 请求解析 (`handleData` Phase 1)

- **累积缓冲**：客户端发送的 CONNECT 请求可能分片到达，使用 `connectBuf` 累积直到匹配 `\r\n\r\n`。
- **安全限制**：`connectBuf` 上限 8KB，超过则返回 `400 Bad Request` 并断开，防止恶意客户端无限堆积。
- **方法校验**：仅接受 `CONNECT host:port HTTP/1.x`，其他方法返回 `405 Method Not Allowed`。
- **尾部数据保护**：CONNECT 头之后可能紧跟着 TLS ClientHello（TCP 粘包），将尾部字节暂存到 `pending` 队列，待 WS `onopen` 后一并转发。

#### 3. WebSocket 隧道建立 (`openTunnel`)

- **升级地址**：由 `upstreamproxy.ts` 构造，形如 `wss://api.anthropic.com/v1/code/upstreamproxy/ws`。
- **认证头**：
  - WS 握手使用 `Authorization: Bearer <token>`（网关层鉴权，`proto authn: PRIVATE_API`）。
  - 隧道首帧携带 `Proxy-Authorization: Basic base64(sessionId:token)`（服务端用于隧道鉴权和目标路由）。
- **Content-Type 头**：必须设置 `application/proto`，否则服务端 `protojson.Unmarshal` 会静默失败（EOF）。
- **代理与 TLS**：Node 路径使用 `ws` 包并传入 `agent`（来自 `getWebSocketProxyAgent`）和 `tls` 选项（来自 `getWebSocketTLSOptions`）；Bun 路径使用原生 `WebSocket` 并传入 `proxy` 和 `tls` 扩展选项。

#### 4. 双向数据转发 (`handleData` Phase 2 / `forwardToWs` / `ws.onmessage`)

- **客户端 → WS**：数据按 512KB 切片，每片调用 `encodeChunk` 后 `ws.send()`。512KB 对应 Envoy 单请求缓冲区上限，为后续大 payload（如 Datadog）预留空间。
- **WS → 客户端**：收到 `ArrayBuffer` 或 `Blob` 后 `decodeChunk`，将 payload 写回 TCP socket。收到首个非空 payload 后设置 `established = true`，表示隧道已进入 TLS 透传阶段。
- **握手前数据缓冲**：如果 WS 尚未 `onopen`，新到达的客户端数据进入 `pending` 队列，`onopen` 时批量 flush。

#### 5. 心跳保活 (`sendKeepalive`)

- **间隔**：30 秒（`PING_INTERVAL_MS`）。
- **机制**：发送一个零长度的 `encodeChunk(new Uint8Array(0))`。
- **原因**：sidecar 空闲超时约为 50 秒，30 秒的心跳可保持隧道不被中间件切断。

#### 6. 连接清理 (`cleanupConn`)

- 清除 `setInterval` 定时器。
- 若 WS 仍处于 `OPEN` 或 `CONNECTING` 状态，调用 `ws.close()`。
- 将 `st.ws` 置为 `undefined`，协助 GC。

---

## 具体技术实现

### 关键数据结构

#### `UpstreamProxyRelay`（对外接口）

```typescript
// relay.ts:105-108
export type UpstreamProxyRelay = {
  port: number      // 实际绑定的本地端口号
  stop: () => void  // 关闭 TCP server 的清理函数
}
```

#### `ConnState`（单连接状态机）

```typescript
// relay.ts:110-127
type ConnState = {
  ws?: WebSocketLike           // 当前连接的 WebSocket 实例
  connectBuf: Buffer           // Phase 1 累积的 CONNECT 请求头
  pinger?: ReturnType<typeof setInterval>  // 30s 心跳定时器引用
  pending: Buffer[]            // WS 握手完成前到达的客户端数据
  wsOpen: boolean              // WS onopen 是否已触发
  established: boolean         // 是否已向客户端转发过服务端数据（即隧道已建立）
  closed: boolean              // 防止 onerror + onclose 重复清理的互斥标志
}
```

#### `WebSocketLike`（运行时抽象）

```typescript
// relay.ts:37-47
type WebSocketLike = Pick<
  WebSocket,
  | 'onopen' | 'onmessage' | 'onerror' | 'onclose'
  | 'send' | 'close' | 'readyState' | 'binaryType'
>
```

该类型同时兼容：
- Node 的 `ws` 包实例（通过属性风格 `onX` 回调）。
- Bun/浏览器的原生 `WebSocket`。

#### `ClientSocket`（写回客户端的抽象）

```typescript
// relay.ts:135-138
type ClientSocket = {
  write: (data: Uint8Array | string) => void
  end: () => void
}
```

Bun 和 Node 的 socket 对象被分别适配成此接口，使 `handleData` / `openTunnel` 逻辑完全运行时无关。

### 关键流程

#### 1. 启动流程 (`startUpstreamProxyRelay`)

```
开始
  │
  ▼
构造 authHeader (Basic base64(sessionId:token))
构造 wsAuthHeader (Bearer token)
  │
  ▼
检测运行时 ──Bun──▶ startBunRelay(wsUrl, authHeader, wsAuthHeader)
  │Node
  ▼
await startNodeRelay(wsUrl, authHeader, wsAuthHeader)
  │
  ▼
返回 { port, stop }
```

#### 2. Bun 运行时 TCP 服务器 (`startBunRelay`)

```typescript
Bun.listen<BunState>({
  hostname: '127.0.0.1',
  port: 0,
  socket: {
    open(sock) {
      sock.data = { ...newConnState(), writeBuf: [] }
    },
    data(sock, data) {
      const adapter: ClientSocket = {
        write: payload => {
          const bytes = typeof payload === 'string' ? Buffer.from(payload, 'utf8') : payload
          if (st.writeBuf.length > 0) {
            st.writeBuf.push(bytes)  // 已有积压，直接入队
            return
          }
          const n = sock.write(bytes)
          if (n < bytes.length) st.writeBuf.push(bytes.subarray(n))  // 记录未写入尾部
        },
        end: () => sock.end(),
      }
      handleData(adapter, st, data, ...)
    },
    drain(sock) {
      // 内核缓冲区可写时，刷新 writeBuf 队列
      while (st.writeBuf.length > 0) {
        const chunk = st.writeBuf[0]!
        const n = sock.write(chunk)
        if (n < chunk.length) {
          st.writeBuf[0] = chunk.subarray(n)
          return  // 再次写满，等待下一次 drain
        }
        st.writeBuf.shift()
      }
    },
    close(sock) { cleanupConn(sock.data) },
    error(sock, err) { ... cleanupConn(sock.data) },
  },
})
```

**核心注意点**：Bun 的 `sock.write()` 返回实际写入字节数，剩余部分必须手动维护队列并在 `drain` 事件中补写，否则数据静默丢失。

#### 3. Node 运行时 TCP 服务器 (`startNodeRelay`)

```typescript
const server = createServer(sock => {
  const st = newConnState()
  states.set(sock, st)
  const adapter: ClientSocket = {
    write: payload => sock.write(typeof payload === 'string' ? payload : Buffer.from(payload)),
    end: () => sock.end(),
  }
  sock.on('data', data => handleData(adapter, st, data, ...))
  sock.on('close', () => cleanupConn(states.get(sock)))
  sock.on('error', err => { ... cleanupConn(...) })
})
```

Node 的 `net.Socket.write()` 内部由 libuv 缓冲，即使返回 `false` 也保证后续异步排空，因此无需像 Bun 那样手动维护 `writeBuf`。

#### 4. 共享数据处理 (`handleData`)

```
收到 data
  │
  ▼
ws 已存在？──否──▶ 追加到 connectBuf
  │                查找 \r\n\r\n
  │                未找到 ──▶ 检查 8KB 限制 ──▶ 超限返回 400
  │                找到 ──▶ 解析 CONNECT 行
  │                         非法方法 ──▶ 返回 405
  │                         合法 ──▶ 尾部数据入 pending ──▶ 调用 openTunnel
  │是
  ▼
wsOpen 为真？──否──▶ 数据入 pending 队列
  │是
  ▼
调用 forwardToWs(ws, data) 分片发送
```

#### 5. WebSocket 事件处理 (`openTunnel` 内部回调)

| 事件 | 行为 |
|------|------|
| `onopen` | 发送首帧（CONNECT 行 + Proxy-Authorization）；设置 `wsOpen = true`；flush `pending`；启动 30s 心跳 |
| `onmessage` | `decodeChunk` → 若 payload 非空则 `sock.write(payload)`，并设置 `established = true` |
| `onerror` | 记录 debug 日志；若 `closed` 则忽略；否则 `closed = true`；若 `!established` 则返回 `502 Bad Gateway`；`sock.end()`；`cleanupConn` |
| `onclose` | 同上，但不返回 502（隧道可能正常结束） |

### 协议细节：UpstreamProxyChunk

```
0x0a                    // tag: field_number=1, wire_type=2 (length-delimited)
<varint length>         // 数据长度，小端序 7-bit 编码
<length bytes of data>  // 原始字节
```

编码示例（10 字节数据）：
```
0a 0a [10 bytes payload]
```

编码示例（700 字节数据，需要 2 字节 varint）：
```
0a bc 05 [700 bytes payload]   // 700 = 0xbc + (0x05 << 7)
```

---

## 关键代码路径与文件引用

### 核心函数位置

| 函数 | 文件:行 | 说明 |
|------|---------|------|
| `encodeChunk` | `relay.ts:66-81` | Protobuf 手工编码 |
| `decodeChunk` | `relay.ts:87-103` | Protobuf 手工解码 |
| `startUpstreamProxyRelay` | `relay.ts:155-174` | 对外入口，运行时分发 |
| `startBunRelay` | `relay.ts:176-241` | Bun TCP 服务器 |
| `startNodeRelay` | `relay.ts:245-289` | Node TCP 服务器（测试可直接调用） |
| `handleData` | `relay.ts:295-342` | 共享的 CONNECT 解析与数据转发 |
| `openTunnel` | `relay.ts:344-428` | WebSocket 建立与事件绑定 |
| `forwardToWs` | `relay.ts:436-442` | 分片发送数据到 WS |
| `cleanupConn` | `relay.ts:444-455` | 清理连接资源 |

### 调用方

- **`src/upstreamproxy/upstreamproxy.ts:134`**：`await startUpstreamProxyRelay({ wsUrl, sessionId, token })`

### 被调用方 / 依赖方

- **`src/utils/proxy.ts`**：提供 `getWebSocketProxyAgent()`（Node 代理）和 `getWebSocketProxyUrl()`（Bun 代理）。
- **`src/utils/mtls.ts`**：提供 `getWebSocketTLSOptions()`，注入自定义 CA / 客户端证书。
- **`src/utils/debug.ts`**：提供 `logForDebugging()`，用于错误和诊断日志。
- **`ws` (npm)**：Node 路径动态导入的 WebSocket 实现。

---

## 依赖与外部交互

### 内部依赖

| 模块 | 导入符号 | 用途 |
|------|----------|------|
| `src/utils/debug.js` | `logForDebugging` | 错误日志、调试信息 |
| `src/utils/mtls.js` | `getWebSocketTLSOptions` | WS TLS 配置（mTLS / CA） |
| `src/utils/proxy.js` | `getWebSocketProxyAgent`, `getWebSocketProxyUrl` | 容器出向 HTTP CONNECT 代理 |

### 外部依赖

| 依赖 | 用途 | 加载方式 |
|------|------|----------|
| `node:net` | Node 的 `createServer` | 静态导入 |
| `ws` | Node 的 WebSocket 客户端 | `await import('ws')`，仅在 `startNodeRelay` 时加载 |
| `Bun.listen` / `globalThis.WebSocket` | Bun 的 TCP / WS 原生 API | 运行时全局对象 |

### 外部 API 交互

| 端点 | 协议 | 说明 |
|------|------|------|
| `wss://<baseUrl>/v1/code/upstreamproxy/ws` | WebSocket (binary) | 隧道端点，首帧携带 CONNECT + Proxy-Authorization |

### 环境变量（间接使用）

通过 `proxy.ts` / `mtls.ts` 间接影响本模块行为：
- `HTTPS_PROXY` / `https_proxy`：决定 WS 升级是否走 egress 代理。
- `CLAUDE_CODE_CLIENT_CERT` / `CLAUDE_CODE_CLIENT_KEY`：mTLS 客户端证书。
- `NODE_EXTRA_CA_CERTS` / 配置 CA：影响 WS TLS 信任链。

---

## 风险、边界与改进建议

### 已知风险

| 风险 | 等级 | 说明 |
|------|------|------|
| **Bun 部分写入丢失** | 高（已缓解） | 若未正确维护 `writeBuf` + `drain`，Bun 环境下回写客户端的数据会静默截断。当前代码已显式处理，但属于易错点。 |
| **无自动重连** | 中 | WebSocket 断开后直接关闭 TCP 连接，子进程需自行重试或重新建立 CONNECT。对于长连接场景（如 LSP）可能造成中断。 |
| **512KB 分块未经验证** | 中 | 虽然按 Envoy 缓冲区设计，但实际未在集成环境中测试超大 payload（如大文件上传）的端到端稳定性。 |
| **单帧 Protobuf 解析失败** | 低 | `decodeChunk` 对非法帧返回 `null`，当前实现直接忽略，不会触发连接关闭，可能导致后续数据错位（但服务端受控，概率低）。 |
| `ws` 包动态加载失败 | 低 | Node 路径依赖 `ws` 包，若打包工具 tree-shake 或运行时缺失会抛异常。当前由 `upstreamproxy.ts` 的 try/catch 兜底为 fail-open。 |

### 边界情况

#### 已处理的边界

1. **CONNECT 分片**：`connectBuf` 累积 + 8KB 上限，兼容慢速或分片发送的客户端。
2. **粘包（CONNECT + ClientHello）**：解析完 header 后，将 `\r\n\r\n` 之后的尾部数据存入 `pending`，WS `onopen` 后自动 flush。
3. **WS 握手期间的并发数据**：`wsOpen` 标志 + `pending` 队列，避免握手完成前数据丢失。
4. **重复关闭竞争**：`closed` 布尔标志确保 `onerror` 和 `onclose` 不会双重 `sock.end()`。
5. **未建立隧道前的错误**：在 `established === false` 时返回 `HTTP/1.1 502 Bad Gateway`，使 curl 等子进程能立即感知失败。
6. **心跳与 sidecar 超时**：30s 心跳低于 50s sidecar idle timeout，防止中间件切断空闲连接。

#### 未处理或潜在的边界

1. **并发连接数无限制**：TCP server 未设置 `maxConnections`，恶意或异常子进程可能耗尽文件描述符。
2. **IPv6 仅回环未监听**：当前硬编码 `127.0.0.1`，若子进程通过 `::1` 访问会失败。
3. **HTTP/2 CONNECT 不支持**：仅支持 HTTP/1.1 CONNECT，部分现代工具可能尝试 HTTP/2 Extended CONNECT。
4. **WS 二进制帧与 JSON 混用**：服务端若误发 JSON，decodeChunk 返回 `null` 且静默丢弃，缺少诊断日志。

### 改进建议

#### 高优先级

1. **连接数限制与熔断**
   - 当前：无并发限制。
   - 建议：增加 `maxConnections` 配置（默认 128 或 256），超限直接拒绝新连接，防止 FD 耗尽。

2. **异常帧诊断日志**
   - 当前：`decodeChunk` 返回 `null` 时完全静默。
   - 建议：在 `onmessage` 中增加 `else if (payload === null)` 分支，记录 `warn` 级别日志，便于排查服务端/客户端版本不匹配。

3. **IPv6 双栈监听**
   - 当前：硬编码 `127.0.0.1`。
   - 建议：同时尝试监听 `::1`（Bun/Node 均支持 `::1` 双栈回退），或至少让 `NO_PROXY` 中的 `::1` 与监听地址保持一致。

#### 中优先级

4. **优雅重连（会话级）**
   - 当前：WS 断开即终止连接。
   - 建议：在 `upstreamproxy.ts` 层增加可选的重启策略（指数退避重连 + 重新 `startUpstreamProxyRelay`），而非在 `relay.ts` 单连接层实现，以保持 relay 的简单性。

5. **流量指标埋点**
   - 当前：仅有 debug 日志。
   - 建议：增加 OpenTelemetry Counter，记录 `upstreamproxy.connections.opened`、`upstreamproxy.bytes.sent`、`upstreamproxy.bytes.received`、`upstreamproxy.errors`。

6. **超大分块端到端测试**
   - 当前：512KB 分块逻辑基于 Envoy 文档，但无实际测试覆盖。
   - 建议：在 CI 中增加 >10MB 文件通过 relay 上传/下载的稳定性测试。

#### 低优先级

7. **protobuf 编码泛化**
   - 当前：手写编码仅支持单字段 `bytes data = 1`。
   - 建议：若未来需要扩展字段（如添加 `metadata`），应迁移到 `protobufjs` 或 `protobuf-es`，并评估 bundle 体积影响。

8. **HTTP/2 Extended CONNECT 评估**
   - 当前：仅 HTTP/1.1。
   - 建议：随着 Envoy/GKE 对 Extended CONNECT 的支持成熟，评估是否能减少 WebSocket 帧封装开销。
