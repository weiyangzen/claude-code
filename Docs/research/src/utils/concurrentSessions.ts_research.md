# concurrentSessions.ts 研究文档

> 文件路径：`src/utils/concurrentSessions.ts`  
> 行数：204 行  
> 研究日期：2026-04-01

---

## 场景与职责

Claude Code 支持多种会话形态：交互式 CLI、`claude --bg` 后台会话、daemon 及其 worker。用户可能同时运行多个 Claude 进程。`concurrentSessions.ts` 提供了一个轻量级的**进程级会话注册表**，让系统能够：
1. 追踪当前机器上所有活跃的 Claude 会话。
2. 支持 `claude ps` 命令枚举并展示这些会话。
3. 在会话切换（如 `/resume`）时更新会话元数据。
4. 在进程退出时自动清理 PID 文件，避免僵尸记录。
5. 统计当前并发会话数，用于 telemetry 或功能限制（如提示用户已有其他会话在运行）。

---

## 功能点目的

| 导出项 | 签名 | 目的 |
|--------|------|------|
| `registerSession` | `() => Promise<boolean>` | 写入当前进程的 PID 文件到 `~/.claude/sessions/<pid>.json`，注册清理钩子 |
| `updateSessionName` | `(name?: string) => Promise<void>` | 更新 PID 文件中的会话名称（供 `claude ps` 展示） |
| `updateSessionBridgeId` | `(bridgeSessionId: string \| null) => Promise<void>` | 记录 Remote Control bridge session ID，用于去重 |
| `updateSessionActivity` | `(patch: { status?, waitingFor? }) => Promise<void>` | 推送实时活动状态（busy/idle/waiting） |
| `countConcurrentSessions` | `() => Promise<number>` | 扫描 `~/.claude/sessions/`，过滤掉已崩溃进程的 stale PID 文件，返回存活会话数 |
| `isBgSession` | `() => boolean` | 判断当前是否运行在 `claude --bg` tmux 会话中 |

---

## 具体技术实现

### 3.1 会话类型 `SessionKind`

```ts
export type SessionKind = 'interactive' | 'bg' | 'daemon' | 'daemon-worker'
```

- `interactive`：默认，用户直接交互的 CLI/SDK。
- `bg`：通过 `claude --bg` 启动的后台 tmux 会话。
- `daemon` / `daemon-worker`：守护进程及其工作进程。

### 3.2 会话种类覆盖 `envSessionKind`

```ts
function envSessionKind(): SessionKind | undefined {
  if (feature('BG_SESSIONS')) {
    const k = process.env.CLAUDE_CODE_SESSION_KIND
    if (k === 'bg' || k === 'daemon' || k === 'daemon-worker') return k
  }
  return undefined
}
```

由父进程（spawner）通过环境变量注入，这样子进程可以在没有父进程帮助写文件的情况下自行注册，实现“cleanup-on-exit 自动生效”。

### 3.3 PID 文件注册 `registerSession`

**跳过条件：**
```ts
if (getAgentId() != null) return false
```
- 子代理（teammate/subagent）不注册，防止把 swarm 代理的噪音混入真正的并发会话统计。

**写入内容：**
```json
{
  "pid": 12345,
  "sessionId": "uuid",
  "cwd": "/project/path",
  "startedAt": 1712345678901,
  "kind": "interactive",
  "entrypoint": "cli",
  "messagingSocketPath": "/tmp/...",
  "name": "my-session",
  "logPath": "/tmp/...",
  "agent": "claude"
}
```

**目录权限：**
```ts
await mkdir(dir, { recursive: true, mode: 0o700 })
await chmod(dir, 0o700)
```
- 使用 `0o700` 确保会话目录仅对所有者可读写，防止其他用户窥探会话元数据。

**退出清理：**
```ts
registerCleanup(async () => {
  try { await unlink(pidFile) } catch {}
})
```
- 通过 `cleanupRegistry.ts` 注册进程退出时的 PID 文件删除。

**会话切换监听：**
```ts
onSessionSwitch(id => {
  void updatePidFile({ sessionId: id })
})
```
- 当用户执行 `/resume` 或 `--resume` 切换会话时，`getSessionId()` 会变化。如果不更新 PID 文件，`claude ps` 会读取到旧的 session ID，导致 sparkline 和 transcript 关联错误。

### 3.4 PID 文件更新 `updatePidFile`

最佳 effort 读写：
```ts
async function updatePidFile(patch: Record<string, unknown>): Promise<void> {
  const pidFile = join(getSessionsDir(), `${process.pid}.json`)
  try {
    const data = jsonParse(await readFile(pidFile, 'utf8')) as Record<string, unknown>
    await writeFile(pidFile, jsonStringify({ ...data, ...patch }))
  } catch (e) {
    logForDebugging(`[concurrentSessions] updatePidFile failed: ${errorMessage(e)}`)
  }
}
```
- 如果文件不存在（会话未注册）或读写失败，静默忽略并记录调试日志。

### 3.5 并发会话计数 `countConcurrentSessions`

```ts
export async function countConcurrentSessions(): Promise<number> {
  const dir = getSessionsDir()
  // 读取目录
  for (const file of files) {
    if (!/^\d+\.json$/.test(file)) continue   // 严格文件名过滤
    const pid = parseInt(file.slice(0, -5), 10)
    if (pid === process.pid) { count++; continue }
    if (isProcessRunning(pid)) {
      count++
    } else if (getPlatform() !== 'wsl') {
      void unlink(join(dir, file)).catch(() => {})
    }
  }
  return count
}
```

**关键设计：**

1. **严格文件名过滤 `/^\d+\.json$/`**
   - 早期版本使用 `parseInt` 的宽松前缀解析，导致用户放在 `~/.claude/sessions/` 下的笔记文件（如 `2026-03-14_notes.md`）被误解析为 PID 2026 并被删除。此修复参考了 issue #34210。

2. **进程存活探测 `isProcessRunning(pid)`**
   - 使用 `process.kill(pid, 0)` 信号 0 探测。若进程不存在或无权访问，返回 false。
   - PID ≤ 1 直接视为不存活。

3. **WSL 保守策略**
   - 在 WSL 下**不删除 stale 文件**。因为 `~/.claude/sessions/` 可能通过符号链接或 `CLAUDE_CONFIG_DIR` 与 Windows 原生 Claude 共享，Windows PID 在 WSL 中无法被 `process.kill` 探测，误删风险高。此处选择保守低计（undercount）。

---

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.js` | `getOriginalCwd`, `getSessionId`, `onSessionSwitch` |
| `src/utils/cleanupRegistry.js` | `registerCleanup` — 进程退出时清理 PID 文件 |
| `src/utils/debug.js` | `logForDebugging` |
| `src/utils/envUtils.js` | `getClaudeConfigHomeDir` |
| `src/utils/errors.js` | `errorMessage`, `isFsInaccessible` |
| `src/utils/genericProcessUtils.js` | `isProcessRunning` |
| `src/utils/platform.js` | `getPlatform` |
| `src/utils/slowOperations.js` | `jsonParse`, `jsonStringify` |
| `src/utils/teammate.js` | `getAgentId` — 判断是否为子代理 |

### 调用方
| 文件 | 调用场景 |
|------|----------|
| `src/main.tsx` | 应用启动时调用 `registerSession` |
| `src/screens/REPL.tsx` | REPL 状态变化时调用 `updateSessionActivity` |
| `src/screens/ResumeConversation.tsx` | 恢复会话 |
| `src/bootstrap/state.ts` | 会话切换状态管理 |
| `src/bridge/replBridge.ts` / `replBridgeHandle.ts` | Remote Control bridge 更新 `updateSessionBridgeId` |
| `src/utils/sessionStorage.ts` | 会话存储 |
| `src/utils/sessionRestore.ts` | 会话恢复 |
| `src/commands/exit/exit.tsx` | 退出时可能涉及 bg session 判断 |
| `src/services/tips/tipRegistry.ts` | 根据并发会话数决定是否展示某些提示 |

---

## 依赖与外部交互

### 外部系统交互
- **文件系统**：在 `~/.claude/sessions/` 目录下读写 `<pid>.json` 文件。
- **进程管理**：通过 `process.kill(pid, 0)` 探测其他进程是否存活。
- **环境变量**：读取 `CLAUDE_CODE_SESSION_KIND`、`CLAUDE_CODE_ENTRYPOINT`、`CLAUDE_CODE_MESSAGING_SOCKET`、`CLAUDE_CODE_SESSION_NAME`、`CLAUDE_CODE_SESSION_LOG`、`CLAUDE_CODE_AGENT` 等。

### 与 feature gate 的交互
- `feature('BG_SESSIONS')`：控制后台会话相关字段是否写入 PID 文件，以及 `envSessionKind` 是否生效。
- `feature('UDS_INBOX')`：控制是否写入 `messagingSocketPath`。

---

## 风险、边界与改进建议

### 风险与边界

1. **PID 文件目录共享风险**
   - 如果用户通过 `CLAUDE_CONFIG_DIR` 或符号链接让多个操作系统/架构的 Claude 共享同一个 `sessions` 目录，`isProcessRunning` 可能无法识别其他平台的 PID（如 Windows PID 在 WSL 中），导致误删或误计。WSL 的保守策略缓解了部分问题，但非 WSL 的跨平台共享仍存在风险。

2. **`process.kill(pid, 0)` 的权限限制**
   - 当进程存在但属于其他用户时，`kill(0)` 会抛出 `EPERM`，被捕获后返回 `false`。这意味着多用户系统上其他用户的 Claude 会话会被视为不存在并从计数中排除，也可能被删除（非 WSL 下）。

3. **`updateSessionActivity` 的 feature gate 提前返回**
   - 若 `feature('BG_SESSIONS')` 关闭，`updateSessionActivity` 直接返回，不写入任何状态。这导致 `claude ps` 在 non-bg 构建中无法看到实时活动，只能依赖 transcript 推导。

4. **JSON 读写无锁**
   - `updatePidFile` 采用读-改-写模式，但没有文件锁。在极端情况下（如进程崩溃与并发更新同时发生），可能读到半写文件或丢失更新。

5. **子代理过滤的边界**
   - 通过 `getAgentId() != null` 过滤子代理，但如果某些非子代理的辅助进程（如 LSP server、plugin host）也调用了 `registerSession`，会污染会话列表。

### 改进建议

1. **增加文件锁或原子写**
   使用 `writeFile` 的临时文件+重命名模式（`writeFile(pidFile.tmp)` → `rename(pidFile.tmp, pidFile)`），减少半写 JSON 的概率。

2. **跨平台 PID 命名空间隔离**
   在 PID 文件名中附加平台标识（如 `12345-darwin.json`、`12345-win32.json`），避免不同操作系统进程共享目录时的冲突。

3. **更细粒度的进程类型过滤**
   除了 `getAgentId`，还可以检查 `process.title` 或传入的 `kind` 是否为已知类型，拒绝未知类型的注册。

4. **增加心跳机制**
   目前仅靠 PID 文件存在性和 `process.kill` 判断存活。如果进程进入死锁或无限等待但尚未退出，PID 文件仍存在，会被计为存活。可增加一个 `updatedAt` 时间戳（`updateSessionActivity` 已写入），在 `countConcurrentSessions` 中过滤掉长时间未心跳的进程。

5. **单元测试建议**
   - `registerSession` 在子代理下的跳过行为
   - `countConcurrentSessions` 对非法文件名、当前进程、存活进程、已死进程、WSL 环境的处理
   - `onSessionSwitch` 触发后 PID 文件内容是否正确更新
   - `isBgSession` 对各种 `CLAUDE_CODE_SESSION_KIND` 值的返回
