# tmuxSocket.ts 研究文档

## 场景与职责

`tmuxSocket.ts` 是 Claude Code CLI 的 tmux 隔离套接字管理模块，实现了 Claude 与用户 tmux 会话的完全隔离。这是确保 Bash 工具安全性和可预测性的关键基础设施。

主要使用场景：
1. **Tmux 工具隔离**：Claude 使用的 tmux 会话与用户自己的 tmux 会话完全分离
2. **Bash 命令隔离**：通过环境变量注入，确保用户在 Bash 工具中执行的 tmux 命令也使用隔离套接字
3. **生命周期管理**：延迟初始化、并发控制、优雅清理

## 功能点目的

### 1. 用户 Tmux 会话保护
- **问题**：如果 Claude 在用户的 tmux 会话中运行 `tmux kill-session`，会杀死用户当前会话
- **解决方案**：Claude 创建独立的 tmux server，通过专用 socket 通信

### 2. 环境隔离
- **机制**：通过 `TMUX` 环境变量指向 Claude 的 socket
- **效果**：所有子进程（包括 Bash 工具）自动使用隔离的 tmux server

### 3. Windows (WSL) 支持
- **特殊处理**：Windows 上 tmux 只在 WSL 中可用
- **Interop 修复**：固定 `WSL_INTEROP` 路径解决 WSL 会话断开问题

## 具体技术实现

### 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Claude Code Process                          │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    tmuxSocket.ts Module                       │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐  │  │
│  │  │ Socket Name │  │ Socket Path │  │ Server PID           │  │  │
│  │  │ claude-PID  │  │ /tmp/...    │  │ 12345                │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              │ spawn                                  │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              tmux -L claude-<PID> (isolated server)           │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  Session: base                                          │  │  │
│  │  │  - CLAUDE_CODE_SKIP_PROMPT_HISTORY=true (global env)   │  │  │
│  │  │  - WSL_INTEROP=/run/WSL/1_interop (Windows only)       │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ TMUX env var inheritance
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Bash Tool Child Processes                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  $ tmux list-sessions  # 自动使用 claude-<PID> socket         │  │
│  │  $ tmux new-session    # 在 Claude 的 server 中创建           │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 核心状态

```typescript
// Socket 状态（延迟初始化）
let socketName: string | null = null
let socketPath: string | null = null
let serverPid: number | null = null
let isInitializing = false
let initPromise: Promise<void> | null = null

// Tmux 可用性（一次性检查）
let tmuxAvailabilityChecked = false
let tmuxAvailable = false

// 延迟初始化标记
let tmuxToolUsed = false
```

### Socket 命名

```typescript
export function getClaudeSocketName(): string {
  if (!socketName) {
    socketName = `${CLAUDE_SOCKET_PREFIX}-${process.pid}`
  }
  return socketName
}
// 结果：claude-12345（基于进程 ID）
```

**命名策略**：
- 前缀：`claude`
- 标识符：当前进程 PID
- 唯一性：每个 Claude Code 实例有独立的 PID

### 延迟初始化机制

```typescript
export function markTmuxToolUsed(): void {
  tmuxToolUsed = true
}

export async function ensureSocketInitialized(): Promise<void>
```

**设计意图**：
- 避免在不需要 tmux 时初始化（性能优化）
- `TungstenTool`（Tmux 工具）首次使用时调用 `markTmuxToolUsed()`
- `Shell.ts` 在检测到命令包含 "tmux" 时也会触发初始化

### 初始化流程 (`doInitialize`)

```typescript
async function doInitialize(): Promise<void>
```

**步骤**：

1. **创建 Session**
   ```bash
   tmux -L claude-<PID> new-session -d -s base \
     -e CLAUDE_CODE_SKIP_PROMPT_HISTORY=true \
     -e WSL_INTEROP=/run/WSL/1_interop  # Windows only
   ```

2. **注册清理函数**
   ```typescript
   registerCleanup(killTmuxServer)
   ```

3. **设置全局环境变量**
   ```bash
   tmux -L claude-<PID> set-environment -g CLAUDE_CODE_SKIP_PROMPT_HISTORY true
   ```

4. **获取 Socket 信息**
   ```bash
   tmux -L claude-<PID> display-message -p '#{socket_path},#{pid}'
   # 输出：/tmp/tmux-1000/claude-12345,54321
   ```

5. **Fallback 处理**
   - 如果获取失败，构造标准路径：`$TMPDIR/tmux-<UID>/claude-<PID>`
   - 单独获取 PID

### Windows (WSL) 特殊处理

```typescript
async function execTmux(args: string[], opts?): Promise<ExecResult> {
  if (getPlatform() === 'windows') {
    // -e 直接执行 tmux，避免 bash 吃掉 # 字符
    return execFileNoThrow('wsl', ['-e', 'tmux', ...args], {
      env: { ...process.env, WSL_UTF8: '1' },
    })
  }
  return execFileNoThrow('tmux', args, opts)
}
```

**WSL_INTEROP 修复**：
```typescript
if (getPlatform() === 'windows') {
  await execTmux([
    '-L', socket,
    'set-environment', '-g', 'WSL_INTEROP', '/run/WSL/1_interop'
  ])
}
```

**问题背景**：
- WSL 的 tmux server 继承的 `WSL_INTEROP` 套接字在 spawning wsl.exe 退出后失效
- 固定到 `/run/WSL/1_interop`（WSL 维护的稳定符号链接）确保 interop 持续工作

### TMUX 环境变量格式

```typescript
export function getClaudeTmuxEnv(): string | null {
  if (!socketPath || serverPid === null) {
    return null
  }
  return `${socketPath},${serverPid},0`
  // 示例：/tmp/tmux-1000/claude-12345,54321,0
}
```

**格式说明**（与标准 tmux 的 `TMUX` 变量一致）：
- `socket_path`：Unix domain socket 路径
- `server_pid`：tmux server 进程 ID
- `pane_index`：当前 pane 索引（固定为 0）

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/utils/shell/bashProvider.ts` | Bash 工具获取 TMUX 环境变量 |

### 被本模块使用的依赖

| 模块 | 用途 |
|-----|------|
| `path` (posix) | 跨平台路径处理 |
| `./cleanupRegistry.js` | 注册退出清理 |
| `./debug.js` | 调试日志 |
| `./errors.js` | 错误处理 |
| `./execFileNoThrow.js` | 执行 tmux 命令 |
| `./log.js` | 错误日志 |
| `./platform.js` | 平台检测（Windows） |

## 依赖与外部交互

### 与 Shell.ts 的集成

`Shell.ts` 通过 `getClaudeTmuxEnv()` 获取 TMUX 环境变量，注入到所有 Bash 子进程：

```typescript
// Shell.ts 伪代码
const tmuxEnv = getClaudeTmuxEnv()
if (tmuxEnv) {
  childEnv.TMUX = tmuxEnv
}
```

**关键效果**：
- 用户在 Bash 工具中执行 `tmux` 命令自动使用隔离 socket
- 无需用户知道 socket 名称

### 与 TungstenTool 的集成

Tmux 工具显式标记使用：
```typescript
import { markTmuxToolUsed } from '../utils/tmuxSocket.js'

export class TungstenTool {
  constructor() {
    markTmuxToolUsed()
  }
}
```

### 与 CleanupRegistry 的集成

```typescript
async function killTmuxServer(): Promise<void> {
  const socket = getClaudeSocketName()
  const result = await execTmux(['-L', socket, 'kill-server'])
  
  if (result.code === 0) {
    logForDebugging(`[Socket] Successfully killed tmux server`)
  } else {
    // Server 可能已死亡，这是正常的
    logForDebugging(`[Socket] Failed to kill tmux server (exit ${result.code})`)
  }
}
```

**设计要点**：
- 使用 `kill-server` 而非 `kill-session` 确保完全清理
- 忽略错误（server 可能已提前退出）

## 风险、边界与改进建议

### 潜在风险

1. **PID 重用**
   - Socket 名称基于 PID，如果 Claude 异常退出后 PID 被重用
   - 新实例可能连接到旧的 tmux server

2. **Socket 文件残留**
   - 如果清理未执行（如 `kill -9`），socket 文件可能残留
   - 下次初始化时可能检测到 "已存在" 的 session

3. **并发初始化竞态**
   - `isInitializing` + `initPromise` 模式防止并发
   - 但错误处理中 `catch { return }` 可能隐藏问题

4. **WSL 版本差异**
   - WSL1 和 WSL2 的 interop 行为不同
   - `/run/WSL/1_interop` 假设可能不适用于所有版本

### 边界条件

| 场景 | 行为 |
|-----|------|
| tmux 未安装 | `checkTmuxAvailable()` 返回 false，socket 不初始化 |
| 初始化过程中出错 | 记录错误，静默失败（graceful degradation） |
| Socket 信息获取失败 | 使用 fallback 路径构造 |
| PID 解析失败 | 抛出错误，初始化失败 |
| 重复调用 `ensureSocketInitialized` | 已初始化时立即返回，初始化中时等待 Promise |

### 改进建议

1. **更强的 Socket 唯一性**
   ```typescript
   // 建议：加入时间戳或随机组件
   socketName = `claude-${process.pid}-${Date.now().toString(36)}`
   ```

2. **Stale Socket 检测**
   ```typescript
   async function isSocketStale(socketPath: string): Promise<boolean> {
     try {
       const stat = await fs.stat(socketPath)
       const age = Date.now() - stat.mtimeMs
       return age > 24 * 60 * 60 * 1000  // 24 hours
     } catch {
       return false
     }
   }
   ```

3. **健康检查**
   ```typescript
   export async function isTmuxServerHealthy(): Promise<boolean> {
     const result = await execTmux(['-L', getClaudeSocketName(), 'list-sessions'])
     return result.code === 0
   }
   ```

4. **配置暴露**
   ```typescript
   // 允许高级用户自定义 socket 路径
   export function setCustomSocketPath(path: string): void
   ```

5. **更好的错误报告**
   ```typescript
   // 当前：错误仅记录到调试日志
   // 建议：向用户显示友好的错误提示
   if (!tmuxAvailable) {
     throw new UserVisibleError(
       'Tmux is not installed. The Tmux tool and persistent Bash sessions are unavailable.'
     )
   }
   ```

6. **Socket 权限检查**
   ```typescript
   // 确保 socket 文件权限正确（仅当前用户可访问）
   async function ensureSocketPermissions(socketPath: string): Promise<void> {
     await fs.chmod(socketPath, 0o600)
   }
   ```

### 测试建议

应覆盖以下场景：
- 有/无 tmux 的环境
- 首次初始化和重复初始化
- 并发初始化调用
- 初始化失败后的行为
- 清理执行（正常退出和异常退出）
- Windows (WSL) 特殊路径
- Socket 信息获取失败后的 fallback
