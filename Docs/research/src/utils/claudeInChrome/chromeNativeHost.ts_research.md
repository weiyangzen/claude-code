# chromeNativeHost.ts 深度研究文档

## 场景与职责

`chromeNativeHost.ts` 是 **Claude in Chrome** 功能的 **Chrome Native Messaging Host** 的纯 TypeScript 实现。它作为 Chrome 浏览器扩展与本地 Claude Code 进程之间的双向桥梁，负责：

1. **接收来自 Chrome 扩展的消息**（通过 Chrome Native Messaging 协议，即 `stdin` 上的 4 字节长度前缀 + JSON 载荷）。
2. **将消息转发给本地 MCP 客户端**（通过 Unix domain socket 或 Windows named pipe）。
3. **接收来自 MCP 客户端的工具请求**，并通过 Native Messaging 协议回写给 Chrome 扩展。
4. **生命周期管理**：启动 socket server、清理 stale socket、优雅关闭。

该模块此前由 Rust NAPI 绑定实现，现已完全迁移为纯 TypeScript/Node，降低了构建复杂度和跨平台维护成本。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `sendChromeMessage()` | 按 Chrome Native Messaging 协议（4 字节 little-endian 长度前缀 + UTF-8 JSON）向 `stdout` 写消息，供 Chrome 扩展读取。 |
| `runChromeNativeHost()` | 入口函数：初始化 `ChromeNativeHost` 与 `ChromeMessageReader`，进入消息循环直到 `stdin` 关闭。 |
| `ChromeNativeHost` | 核心类：管理 Unix socket / Windows named pipe server，维护 MCP 客户端连接池，处理 Chrome ↔ MCP 双向转发。 |
| `ChromeMessageReader` | 异步 `stdin` 读取器，解决同步读取在 Bun 下可能崩溃的问题，支持消息缓冲与分帧。 |

## 具体技术实现

### 1. Chrome Native Messaging 协议

```
[4 bytes: message length, uint32le][N bytes: JSON string, utf-8]
```

- `MAX_MESSAGE_SIZE = 1MB`：限制单条消息大小，防止内存溢出。
- `sendChromeMessage()` 使用 `process.stdout.write` 连续写入长度前缀和 JSON 字节。

### 2. ChromeNativeHost 类

#### Socket 创建流程 (`start()`)

1. **获取 socket 路径**：
   - Unix: `/tmp/claude-mcp-browser-bridge-<username>/<pid>.sock`
   - Windows: `\\.\pipe\claude-mcp-browser-bridge-<username>`
2. **迁移旧版 socket**：若 socket 目录路径存在但不是一个目录，则删除。
3. **创建安全目录**（Unix）：`mkdir(..., mode: 0o700)`，并 `chmod` 修复已存在目录的权限。
4. **清理 stale socket**：扫描目录下 `*.sock`，通过 `process.kill(pid, 0)` 检测进程是否存活，死进程对应的 socket 被删除。
5. **启动 `net.createServer`**，监听 socket 路径。
6. **设置 socket 权限**（Unix）：`chmod(socketPath, 0o600)`。

#### 消息处理 (`handleMessage()`)

支持的消息类型：

| 类型 | 行为 |
|------|------|
| `ping` | 回复 `pong` + 时间戳 |
| `get_status` | 回复版本号 `native_host_version` |
| `tool_response` | 提取 `type` 以外的字段，按 4 字节长度前缀协议转发给所有连接的 MCP 客户端 |
| `notification` | 同上，用于扩展通知（如配对成功） |
| 其他 | 返回 `error: Unknown message type` |

#### MCP 客户端管理 (`handleMcpClient()`)

- 每个新连接分配递增 `clientId`。
- 客户端数据使用同样的 4 字节长度前缀协议。
- 从 socket 读取到的完整 JSON 被解析为 `ToolRequest`，然后通过 `sendChromeMessage()` 以 `type: 'tool_request'` 转发给 Chrome。
- 客户端断开时发送 `mcp_disconnected` 通知给 Chrome。

#### 优雅关闭 (`stop()`)

- 销毁所有 MCP 客户端 socket。
- 关闭 server。
- 删除 socket 文件；若目录为空则删除目录。

### 3. ChromeMessageReader 类

由于同步读取 `process.stdin` 在 Bun 环境下可能崩溃，该类采用**异步事件驱动**模式：

- 维护一个内部 `Buffer` 缓冲区。
- 监听 `process.stdin` 的 `data` / `end` / `error` 事件。
- `read()` 返回 `Promise<string | null>`：
  - 若缓冲区已有完整消息，立即解析返回。
  - 否则设置 `pendingResolve`，等待新数据到达后通过 `tryProcessMessage()` 解析。
- 对无效长度（`0` 或 `> MAX_MESSAGE_SIZE`）返回 `null`，触发上层退出循环。

## 关键代码路径与文件引用

```
src/entrypoints/cli.tsx:79-84
  └── --chrome-native-host 分支
      └── import('../utils/claudeInChrome/chromeNativeHost.js')
          └── runChromeNativeHost()
              ├── ChromeNativeHost.start()
              │   ├── getSecureSocketPath()        ← common.ts
              │   ├── getSocketDir()               ← common.ts
              │   └── net.createServer()
              │       └── handleMcpClient(socket)
              │           └── sendChromeMessage()  → stdout (to Chrome)
              └── ChromeMessageReader.read()
                  ← stdin (from Chrome)
                  └── handleMessage(json)
                      └── sendChromeMessage() / socket.write()
```

### 内部依赖

- `../lazySchema.js`：`lazySchema()` 用于延迟加载 Zod schema。
- `../slowOperations.js`：`jsonParse`, `jsonStringify`（可能是带错误处理或性能隔离的 JSON 工具）。
- `./common.js`：`getSecureSocketPath`, `getSocketDir`。

### 外部调用方

- `src/entrypoints/cli.tsx`：唯一直接调用 `runChromeNativeHost()` 的入口，由 Chrome 扩展通过 Native Host manifest 拉起子进程。

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `fs/promises` | 日志文件追加、目录/文件操作、权限设置 |
| `net` | `createServer`, `Socket` — Unix domain socket / named pipe 通信 |
| `os` | `homedir()`, `platform()` |
| `path` | `join()` |
| `zod` | 消息结构校验（`messageSchema`） |
| `@ant/claude-for-chrome-mcp`（间接） | MCP 客户端通过 socket 连接到此 server |

### 环境变量

- `USER_TYPE === 'ant'`：启用调试日志文件写入 `~/.claude/debug/chrome-native-host.txt`。

## 风险、边界与改进建议

### 风险

1. **Bun 兼容性**：注释明确指出同步读取 `stdin` 会 crash Bun，因此使用了异步缓冲读取。若 Bun 行为变化，需回归验证。
2. **Socket 竞争**：Windows named pipe 使用固定名称（不含 PID），多实例同时运行可能产生命名冲突；Unix socket 按 PID 隔离，相对安全。
3. **Stale socket 清理的竞态**：`process.kill(pid, 0)` 在极短时间内可能误判（进程刚死但文件系统尚未刷新），但影响有限。
4. **无消息 ACK 机制**：Chrome Native Messaging 本身无应用层 ACK，若 Chrome 扩展崩溃，MCP 客户端侧的请求会 hang 直到超时。

### 边界

- 单条消息硬上限 1MB；超大截图或 DOM 可能触及此限制。
- 仅支持单个 Chrome 扩展实例同时连接（Chrome 通常只维护一个 native host 进程）。
- 日志为 fire-and-forget，不阻塞主流程，但 `console.error` 可能污染 stderr（Chrome 扩展通常忽略 native host 的 stderr）。

### 改进建议

1. **动态消息上限**：根据实际场景评估是否将 1MB 上限配置化，或增加分片协议。
2. **Windows pipe 隔离**：考虑在 pipe 名中加入进程 PID 或会话标识，避免多 Claude Code 实例冲突。
3. **健康检查心跳**：在 `ping/pong` 基础上增加对 MCP 客户端的心跳，及时清理死连接。
4. **结构化日志**：当前日志为纯文本，可改用结构化 JSON 日志便于后续问题排查。
5. **测试覆盖**：该文件目前无单元测试，建议增加对 `ChromeMessageReader` 分帧逻辑和 `ChromeNativeHost` 消息路由的测试。
