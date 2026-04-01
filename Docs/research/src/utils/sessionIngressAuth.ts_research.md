# 研究文档：src/utils/sessionIngressAuth.ts

## 场景与职责

`sessionIngressAuth.ts` 负责管理 Claude Code 在**远程/托管环境**（如 CCR - Claude Code Remote、Claude Desktop 的 local-agent 模式）中的**会话入口认证令牌（session ingress token）**。该令牌用于向 Anthropic 后端证明当前会话的身份，从而建立 WebSocket 或 SSE 连接。

模块需要处理多种令牌来源，因为远程环境的启动方式各异：
1. **环境变量**：父进程直接注入（最常见、最优先）。
2. **文件描述符（FD）**：CCR 的 Go env-manager 通过 `cmd.ExtraFiles` 传递管道（legacy 路径）。
3. **well-known 文件**：供无法继承 FD 的子进程读取的回退路径。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getSessionIngressAuthToken()` | 按优先级获取 session ingress token：env var → FD → well-known file。 |
| `getSessionIngressAuthHeaders()` | 根据 token 类型生成对应的 HTTP auth headers：session key 用 `Cookie` + `X-Organization-Uuid`；JWT 用 `Bearer`。 |
| `updateSessionIngressAuthToken(token)` | 在进程运行中更新 token（如 REPL bridge 重连后注入新 token），通过修改 `process.env` 实现。 |
| `getTokenFromFileDescriptor()` | 内部函数：从 FD 读取 token，失败时回退到 well-known file，结果缓存到全局状态。 |

---

## 具体技术实现

### 1. `getSessionIngressAuthToken`

优先级：
1. `process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN`
   - 若存在且非空，直接返回。
2. `getTokenFromFileDescriptor()`
   - 处理 legacy FD 路径与 well-known file 回退。

### 2. `getTokenFromFileDescriptor`

缓存机制：
- 使用 `../bootstrap/state.js` 中的 `getSessionIngressToken()` / `setSessionIngressToken()` 做全局状态缓存。
- FD 只能读取一次（pipe），因此缓存至关重要。

读取流程：
1. 检查全局缓存，若已尝试（包括成功、失败、null）则直接返回。
2. 读取 `process.env.CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`。
   - 若不存在，直接走 well-known file 回退。
3. 解析为整数 FD。
   - 解析失败 → `logForDebugging(level: 'error')` → 缓存 `null` → 返回 `null`。
4. 根据平台选择 FD 代理路径：
   - macOS / FreeBSD → `/dev/fd/${fd}`
   - Linux → `/proc/self/fd/${fd}`
5. 使用 `getFsImplementation().readFileSync(fdPath, { encoding: 'utf8' })` 读取并 `trim()`。
   - 空 token → 缓存 `null` → 返回 `null`。
   - 成功 → 缓存 token → `maybePersistTokenForSubprocesses(...)` 写入 well-known file → 返回 token。
   - 任何异常（如 ENXIO，子进程继承了 env var 但没继承 FD）→ 记录 error → 走 well-known file 回退。

### 3. `getSessionIngressAuthHeaders`

```ts
export function getSessionIngressAuthHeaders(): Record<string, string> {
  const token = getSessionIngressAuthToken()
  if (!token) return {}
  if (token.startsWith('sk-ant-sid')) {
    const headers: Record<string, string> = { Cookie: `sessionKey=${token}` }
    const orgUuid = process.env.CLAUDE_CODE_ORGANIZATION_UUID
    if (orgUuid) {
      headers['X-Organization-Uuid'] = orgUuid
    }
    return headers
  }
  return { Authorization: `Bearer ${token}` }
}
```

- `sk-ant-sid` 开头的 session key 使用 Cookie 认证。
- 其他 token（如 JWT）使用 Bearer 认证。

### 4. `updateSessionIngressAuthToken`

```ts
export function updateSessionIngressAuthToken(token: string): void {
  process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN = token
}
```

- 仅修改内存中的环境变量，不影响磁盘上的 well-known file。
- 由于 `getSessionIngressAuthToken` 优先检查环境变量，新 token 会立即生效。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sessionIngressAuth.ts:18-86` | `getTokenFromFileDescriptor` FD 读取与回退逻辑。 |
| `src/utils/sessionIngressAuth.ts:101-110` | `getSessionIngressAuthToken` 优先级入口。 |
| `src/utils/sessionIngressAuth.ts:117-131` | `getSessionIngressAuthHeaders` header 构建。 |
| `src/utils/sessionIngressAuth.ts:138-140` | `updateSessionIngressAuthToken` 运行时更新。 |
| `src/utils/authFileDescriptor.ts` | 被依赖：`maybePersistTokenForSubprocesses`, `readTokenFromWellKnownFile`, `CCR_SESSION_INGRESS_TOKEN_PATH`。 |
| `src/utils/fsOperations.ts` | 被依赖：`getFsImplementation`。 |
| `src/utils/debug.ts` | 被依赖：`logForDebugging`。 |
| `src/utils/errors.ts` | 被依赖：`errorMessage`。 |
| `src/bootstrap/state.ts` | 被依赖：`getSessionIngressToken`, `setSessionIngressToken`。 |
| `src/cli/transports/ccrClient.ts` | 调用方：CCR 客户端获取认证 token。 |
| `src/cli/transports/SSETransport.ts` | 调用方：SSE 传输层。 |
| `src/cli/transports/HybridTransport.ts` | 调用方：混合传输层。 |
| `src/services/api/sessionIngress.ts` | 调用方：session ingress API 请求。 |
| `src/bridge/replBridge.ts` | 调用方：REPL bridge 更新 token。 |
| `src/main.tsx` | 调用方：启动时获取 token 判断 client type。 |

---

## 依赖与外部交互

- **Node.js 内置模块**：`process.env`。
- **内部依赖**：
  - `../bootstrap/state.js`：`getSessionIngressToken`, `setSessionIngressToken`
  - `./authFileDescriptor.js`：`CCR_SESSION_INGRESS_TOKEN_PATH`, `maybePersistTokenForSubprocesses`, `readTokenFromWellKnownFile`
  - `./debug.js`：`logForDebugging`
  - `./errors.js`：`errorMessage`
  - `./fsOperations.js`：`getFsImplementation`
- **外部文件系统路径**：
  - `/dev/fd/${fd}` 或 `/proc/self/fd/${fd}`
  - `/home/claude/.claude/remote/.session_ingress_token`
- **调用方**：CCR/SSE/Hybrid 传输层、API 层、REPL bridge、主入口。

---

## 风险、边界与改进建议

### 风险与边界

1. **FD 读取的竞态与一次性**：pipe FD 是字节流，读取一次后内容即被消费。若多个模块同时尝试读取（如 `getSessionIngressAuthToken` 和另一个独立调用 `getTokenFromFileDescriptor` 的代码），后者会读到空内容。当前通过全局状态缓存缓解了这一问题，但缓存是在首次读取后才设置的，首次读取本身的并发仍可能出问题。

2. **`readFileSync` 的同步 I/O**：`getTokenFromFileDescriptor` 在 FD 路径上使用同步文件读取。这在启动路径中通常可接受，但在某些事件循环敏感的场景中可能产生微阻塞。

3. **`maybePersistTokenForSubprocesses` 的权限问题**：将 token 写入 `/home/claude/.claude/remote/.session_ingress_token` 时，若目录不存在或权限不足，会静默失败（仅记录 debug log）。子进程随后可能因找不到 well-known file 而认证失败，排查困难。

4. **环境变量更新的副作用**：`updateSessionIngressAuthToken` 直接修改 `process.env`，这会影响到当前进程内所有后续创建的子进程。如果某些子进程不应继承新 token（如沙箱中的不可信进程），可能存在最小权限原则上的泄露。不过当前架构下子进程通常都在同一信任域内。

5. **无 token 过期检查**：模块本身不解析 JWT 的 `exp` 字段，也不做 token 有效性校验。过期处理完全由上层网络层（如 WebSocketTransport 收到 401/403 后重连）负责。

### 改进建议

1. **启动时原子化 FD 读取**：在进程最早期的初始化代码（如 `main.tsx` 开头）一次性读取 FD 并将结果写入全局状态，确保后续任何模块都不会再触及 FD 读取逻辑，彻底消除并发竞态。

2. **增加 token 格式校验**：在 `getSessionIngressAuthToken` 返回前，对 token 做基础格式检查（如 JWT 应包含两个点号、session key 应以 `sk-ant-sid` 开头）。格式异常时记录 error 并返回 `null`，帮助快速定位配置问题。

3. **JWT 过期预警（可选）**：若 token 是 JWT，可解析 payload 中的 `exp`，在接近过期时通过 `logForDebugging` 或 telemetry 发出预警，让上层有机会提前刷新。

4. **well-known file 写入失败时提升日志级别**：当前 `maybePersistTokenForSubprocesses` 失败只记录 debug。由于这直接影响子进程认证，建议提升到 `warn` 或 `error` 级别。

5. **增加 `_resetSessionIngressAuthForTesting()`**：方便单元测试在 case 之间清理全局缓存和 `process.env`，避免测试间状态污染。
