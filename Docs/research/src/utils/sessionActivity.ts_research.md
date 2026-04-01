# 研究文档：src/utils/sessionActivity.ts

## 场景与职责

`sessionActivity.ts` 实现了 Claude Code 的**会话活跃度心跳机制**。在远程会话（如 CCR / cloud container）中，长时间空闲可能导致容器或连接被中间层（如负载均衡器、网关）回收。该模块通过跟踪“当前有多少工作在进行”，在活跃期间周期性发送 keep-alive 信号，空闲后停止心跳并记录诊断日志。

核心使用场景：
- API streaming 期间（`api_call`）
- 工具执行期间（`tool_exec`）
- 任何需要告诉基础设施“我还在工作”的长耗时操作

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `registerSessionActivityCallback(cb)` | 由传输层（如 WebSocketTransport）注册实际发送 keep-alive 的回调函数。 |
| `unregisterSessionActivityCallback()` | 移除回调并停止所有计时器，用于会话关闭或传输层切换。 |
| `startSessionActivity(reason)` | 增加活跃度引用计数；从 0→1 时启动 30s 周期心跳定时器。 |
| `stopSessionActivity(reason)` | 减少引用计数；降到 0 时停止心跳并启动 30s 空闲日志定时器。 |
| `sendSessionActivitySignal()` | 立即手动触发一次 keep-alive（受环境变量开关控制）。 |
| `isSessionActivityTrackingActive()` | 查询是否已注册回调。 |

---

## 具体技术实现

### 1. 状态变量

```ts
let activityCallback: (() => void) | null = null
let refcount = 0
const activeReasons = new Map<SessionActivityReason, number>()
let oldestActivityStartedAt: number | null = null
let heartbeatTimer: ReturnType<typeof setInterval> | null = null
let idleTimer: ReturnType<typeof setTimeout> | null = null
let cleanupRegistered = false
```

- `refcount`：总活跃计数。
- `activeReasons`：按原因分类的计数（`api_call` / `tool_exec`），用于诊断日志。
- `oldestActivityStartedAt`：记录最近一次从 0→1 的时间戳，用于计算最长进行中的活动持续时间。

### 2. 心跳定时器

```ts
const SESSION_ACTIVITY_INTERVAL_MS = 30_000

function startHeartbeatTimer(): void {
  clearIdleTimer()
  heartbeatTimer = setInterval(() => {
    logForDiagnosticsNoPII('debug', 'session_keepalive_heartbeat', { refcount })
    if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE_SEND_KEEPALIVES)) {
      activityCallback?.()
    }
  }, SESSION_ACTIVITY_INTERVAL_MS)
}
```

- 每 30 秒记录一次诊断日志。
- 实际发送 keep-alive 受 `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` 环境变量控制，默认不发送（避免在非远程环境中产生噪音）。

### 3. 空闲定时器

```ts
function startIdleTimer(): void {
  clearIdleTimer()
  if (activityCallback === null) return
  idleTimer = setTimeout(() => {
    logForDiagnosticsNoPII('info', 'session_idle_30s')
    idleTimer = null
  }, SESSION_ACTIVITY_INTERVAL_MS)
}
```

- 当 refcount 降到 0 时，启动一个 30s 的 one-shot timer。
- 若在此期间又有新活动开始，会被 `startHeartbeatTimer` 中的 `clearIdleTimer()` 取消。

### 4. 清理注册

```ts
if (!cleanupRegistered) {
  cleanupRegistered = true
  registerCleanup(async () => {
    logForDiagnosticsNoPII('info', 'session_activity_at_shutdown', {
      refcount,
      active: Object.fromEntries(activeReasons),
      oldest_activity_ms:
        refcount > 0 && oldestActivityStartedAt !== null
          ? Date.now() - oldestActivityStartedAt
          : null,
    })
  })
}
```

- 在 `startSessionActivity` 中懒注册一次进程退出清理钩子。
- 退出时输出当前 refcount、各原因计数、最老活动持续时间，用于排查“进程退出时是否还有未完成工作”。

### 5. 引用计数边界保护

```ts
export function stopSessionActivity(reason: SessionActivityReason): void {
  if (refcount > 0) {
    refcount--
  }
  const n = (activeReasons.get(reason) ?? 0) - 1
  if (n > 0) activeReasons.set(reason, n)
  else activeReasons.delete(reason)
  // ...
}
```

- `refcount` 有下界保护（不会降到负数）。
- `activeReasons` 的计数在归零时删除 key，保持 Map 整洁。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sessionActivity.ts:22-28` | 状态变量与类型定义。 |
| `src/utils/sessionActivity.ts:30-39` | 心跳定时器实现。 |
| `src/utils/sessionActivity.ts:42-51` | 空闲定时器实现。 |
| `src/utils/sessionActivity.ts:60-75` | 回调注册与注销。 |
| `src/utils/sessionActivity.ts:92-115` | `startSessionActivity` 与清理注册。 |
| `src/utils/sessionActivity.ts:121-133` | `stopSessionActivity` 与 refcount 管理。 |
| `src/cli/transports/WebSocketTransport.ts` | 调用方：注册 keep-alive 发送回调。 |
| `src/cli/transports/ccrClient.ts` | 调用方：CCR 客户端可能涉及活动跟踪。 |
| `src/services/tools/toolExecution.ts` | 调用方：工具执行前后 bracket 调用。 |
| `src/services/api/claude.ts` | 调用方：API 调用前后 bracket 调用。 |
| `src/services/compact/compact.ts` | 调用方：compact 操作期间保持活跃。 |

---

## 依赖与外部交互

- **内部依赖**：
  - `./cleanupRegistry.js`：`registerCleanup`
  - `./diagLogs.js`：`logForDiagnosticsNoPII`
  - `./envUtils.js`：`isEnvTruthy`
- **无第三方依赖**。
- **调用方**：
  - 传输层：`WebSocketTransport.ts`, `ccrClient.ts`
  - API 层：`claude.ts`
  - 工具层：`toolExecution.ts`
  - 其他：`compact.ts`

---

## 风险、边界与改进建议

### 风险与边界

1. **`startSessionActivity` / `stopSessionActivity` 必须成对调用**：若某条代码路径调用了 `start` 但忘记在 finally/exit 中调用 `stop`，`refcount` 将永远大于 0，心跳持续发送，导致：
   - 容器不会被回收（资源浪费）
   - 退出日志中的 `oldest_activity_ms` 极大，掩盖真正的问题

2. **`activeReasons` 的 key 在 `stop` 时可能不匹配**：若 `start` 和 `stop` 传入的 `reason` 不一致（如 `api_call` 开始，`tool_exec` 结束），`activeReasons` 的统计会失真。当前依赖调用方的正确性，代码层面没有断言检查。

3. **`activityCallback` 的同步性假设**：`activityCallback` 被同步调用。若回调内部执行了较重的同步 I/O（如阻塞式网络请求），会阻塞事件循环 30ms 或更久。虽然 keep-alive 通常是轻量 ping，但模块本身不做任何异步化或超时保护。

4. **`registerCleanup` 的单例性**：`cleanupRegistered` 确保只注册一次清理钩子。若测试需要重置模块状态，没有暴露 `_resetForTesting` 接口，可能需要通过 `jest.resetModules()` 实现。

5. **环境变量控制粒度粗**：`CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` 是全局开关，无法针对特定会话或传输层单独控制。在某些混合模式（本地 + 远程桥接）中可能不够灵活。

### 改进建议

1. **增加 `start`/`stop` 的调用栈追踪（debug 模式）**：在 `logForDiagnosticsNoPII` 中记录每次 `start`/`stop` 的 reason 和当前 refcount，帮助排查不成对调用。或者使用 `AsyncLocalStorage` 追踪调用来源。

2. **引入 `withSessionActivity(reason, fn)` 高阶辅助**：
   ```ts
   export async function withSessionActivity<T>(
     reason: SessionActivityReason,
     fn: () => Promise<T>,
   ): Promise<T> {
     startSessionActivity(reason)
     try {
       return await fn()
     } finally {
       stopSessionActivity(reason)
     }
   }
   ```
   这能显著降低调用方忘记 `stop` 的风险，是更 robust 的 API。

3. **增加 `activityCallback` 执行时间监控**：在调用 `activityCallback()` 前后计时，若超过阈值（如 100ms）则记录 warning，帮助发现阻塞型回调。

4. **按会话隔离状态**：当前模块是全局单例。若未来支持一个进程内同时管理多个会话（如多 tab 守护进程），需要将状态封装为 `SessionActivityTracker` 实例。

5. **暴露测试重置接口**：增加 `__resetSessionActivityForTesting()` 函数，方便单元测试在 case 之间清理定时器和状态，避免测试间污染。
