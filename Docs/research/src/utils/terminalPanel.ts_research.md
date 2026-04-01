# terminalPanel.ts 研究文档

## 场景与职责

`terminalPanel.ts` 是 Claude Code CLI 的内置终端面板管理模块，实现了通过 `Meta+J`（Alt+J）快捷键快速切换到一个持久化的 shell 会话。这是 CLI 工具中的一项高级功能，允许用户在不退出 Claude Code 的情况下执行 shell 命令。

主要使用场景：
1. **快速 shell 访问**：用户需要临时执行 shell 命令而不想退出 Claude Code
2. **持久化会话**：使用 tmux 保持 shell 状态，切换回来后命令历史、环境变量等保持不变
3. **无缝切换**：按 `Meta+J` 进入终端面板，再按 `Meta+J` 返回 Claude Code

## 功能点目的

### 1. 基于 tmux 的持久化终端
- **问题**：直接在子进程中 spawn shell 是非持久的，切换回来会丢失状态
- **解决方案**：使用 tmux 作为后端，每个 Claude Code 实例有独立的 tmux server 和 socket
- **隔离性**：通过唯一的 socket 名称确保多实例之间互不干扰

### 2. 优雅降级
- **场景**：用户系统未安装 tmux
- **行为**：自动回退到非持久的直接 shell 执行

### 3. 生命周期管理
- **创建**：首次切换时自动创建 tmux session
- **绑定**：配置 `Meta+J` 为 detach-client 快捷键
- **清理**：应用退出时自动 kill tmux server

## 具体技术实现

### 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                     Claude Code Instance                     │
│  ┌─────────────────┐         ┌──────────────────────────┐  │
│  │  TerminalPanel  │◄────────│  Singleton (lazy init)   │  │
│  │    (class)      │         │   getTerminalPanel()     │  │
│  └────────┬────────┘         └──────────────────────────┘  │
│           │                                                  │
│           │  spawnSync / spawn                               │
│           ▼                                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              tmux -L claude-panel-<sessionId>         │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  Session: panel                                 │  │  │
│  │  │  ┌──────────────────────────────────────────┐  │  │  │
│  │  │  │  Shell (user's $SHELL or /bin/bash)     │  │  │  │
│  │  │  │  - Meta+J bound to detach-client         │  │  │  │
│  │  │  │  - Status bar shows "Alt+J to return"    │  │  │  │
│  │  │  └──────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 核心类：TerminalPanel

```typescript
class TerminalPanel {
  private hasTmux: boolean | undefined    // tmux 可用性缓存
  private cleanupRegistered = false       // 防止重复注册清理函数

  toggle(): void                          // 公共 API：切换显示
  private checkTmux(): boolean            // 检查 tmux 是否可用
  private hasSession(): boolean           // 检查 session 是否存在
  private createSession(): boolean        // 创建 tmux session
  private attachSession(): void           // 附加到 session
  private showShell(): void               // 主显示逻辑
  private runShellDirect(): void          // 无 tmux 时的回退
}
```

### Socket 命名策略

```typescript
export function getTerminalPanelSocket(): string {
  const sessionId = getSessionId()
  return `claude-panel-${sessionId.slice(0, 8)}`
}
```

- 使用 `getSessionId()` 获取会话唯一标识
- 取前 8 字符平衡唯一性和可读性
- 格式：`claude-panel-a1b2c3d4`

### tmux 配置命令

创建 session 时执行的配置（通过链式命令优化为单次 spawn）：

```bash
tmux -L <socket> \
  bind-key -n M-j detach-client ; \
  set-option -g status-style bg=default ; \
  set-option -g status-left '' ; \
  set-option -g status-right ' Alt+J to return to Claude ' ; \
  set-option -g status-right-style fg=brightblack
```

**配置说明**：
- `bind-key -n M-j detach-client`：全局绑定 Alt+J 为返回 Claude Code
- `status-style bg=default`：透明状态栏背景
- `status-right`：显示返回提示

### 与 Ink 的集成

```typescript
private showShell(): void {
  const inkInstance = instances.get(process.stdout)
  if (!inkInstance) return

  inkInstance.enterAlternateScreen()  // 进入备用屏幕（保留 Claude UI）
  try {
    if (this.checkTmux() && this.ensureSession()) {
      this.attachSession()
    } else {
      this.runShellDirect()
    }
  } finally {
    inkInstance.exitAlternateScreen()  // 返回主屏幕
  }
}
```

使用 Ink 的备用屏幕模式（alternate screen）实现无缝切换：
- `enterAlternateScreen()`：切换到终端的备用缓冲区
- `exitAlternateScreen()`：返回主缓冲区，Claude UI 保持不变

### 清理机制

```typescript
registerCleanup(async () => {
  spawn('tmux', ['-L', socket, 'kill-server'], {
    detached: true,
    stdio: 'ignore',
  })
    .on('error', () => {})  // 忽略 ENOENT（tmux 已消失）
    .unref()                // 不阻塞事件循环
})
```

**设计要点**：
- 使用 `spawn`（非 `spawnSync`）避免阻塞 graceful shutdown 的 Promise.all
- `detached: true` 和 `unref()` 确保不阻塞进程退出
- `.on('error', () => {})` 防止 tmux 提前消失导致的 uncaughtException

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/hooks/useGlobalKeybindings.tsx` | 全局快捷键 `Meta+J` 绑定 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `child_process` (Node.js) | spawn/spawnSync 执行 tmux |
| `../bootstrap/state.js` | `getSessionId()` 获取会话 ID |
| `../ink/instances.js` | 获取 Ink 实例进行屏幕切换 |
| `./cleanupRegistry.js` | 注册退出清理函数 |
| `./cwd.js` | `pwd()` 获取当前工作目录 |
| `./debug.js` | `logForDebugging()` 调试日志 |

## 依赖与外部交互

### 与 tmux 的交互

**命令序列**：
1. **检查可用性**：`tmux -V` → exit code 0 表示可用
2. **检查 session**：`tmux -L <socket> has-session -t panel`
3. **创建 session**：`tmux -L <socket> new-session -d -s panel -c <cwd> <shell> -l`
4. **附加 session**：`tmux -L <socket> attach-session -t panel` (stdio: 'inherit')
5. **清理**：`tmux -L <socket> kill-server` (detached, 忽略错误)

**tmux 参数说明**：
- `-L <socket>`：指定 Unix domain socket 名称（隔离关键）
- `-d`：后台创建（不立即附加）
- `-s panel`：session 名称固定为 "panel"
- `-c <cwd>`：设置初始工作目录
- `-l`：作为 login shell 启动

### 与 Ink 的屏幕管理

使用终端的 alternate screen buffer 特性：
- 主缓冲区：Claude Code TUI
- 备用缓冲区：tmux shell
- 切换是瞬时的，无需重新渲染 Claude UI

### 环境继承

```typescript
const shell = process.env.SHELL || '/bin/bash'
const cwd = pwd()
```
- 使用用户首选 shell（尊重 `$SHELL`）
- 继承 Claude Code 的当前工作目录（通过 `pwd()` 获取，支持 AsyncLocalStorage 覆盖）

## 风险、边界与改进建议

### 潜在风险

1. **tmux 版本兼容性**
   - 不同 tmux 版本的配置语法可能有差异
   - 当前未检查 tmux 版本

2. **Socket 名称冲突**
   - 8 字符 session ID 在极端情况下可能冲突
   - 如果用户同时运行多个 Claude Code 实例且 session ID 前 8 字符相同

3. **僵尸进程风险**
   - `spawn(...).unref()` 可能导致 tmux server 成为孤儿进程
   - 如果 Claude Code 异常退出且 cleanup 未执行

4. **Windows 不支持**
   - tmux 是 Unix 特有工具，Windows 上始终回退到直接 shell
   - 回退模式下无持久化能力

### 边界条件

| 场景 | 行为 |
|-----|------|
| tmux 未安装 | 回退到 `runShellDirect()`，非持久化 |
| session 创建失败 | 记录调试日志，返回 false，调用方应处理 |
| 附加时 session 已不存在 | tmux 会报错，用户回到 Claude Code（需测试） |
| 工作目录不存在 | `new-session -c` 可能失败 |
| `$SHELL` 未设置 | 回退到 `/bin/bash` |
| 用户按 Ctrl+C 退出 shell | 行为取决于 shell 和 tmux 配置 |

### 改进建议

1. **添加 tmux 版本检查**
   ```typescript
   private checkTmuxVersion(): number {
     const result = spawnSync('tmux', ['-V'], { encoding: 'utf-8' })
     // 解析 "tmux 3.2a" → 3.2
   }
   ```

2. **更可靠的 socket 命名**
   ```typescript
   // 建议：使用完整 session ID + 进程 ID
   return `claude-panel-${sessionId}-${process.pid}`
   ```

3. **session 健康检查**
   ```typescript
   // 在 attach 前验证 session 仍然健康
   private isSessionHealthy(): boolean {
     return this.hasSession() && this.isServerResponsive()
   }
   ```

4. **Windows 支持**
   - 考虑使用 Windows Terminal 的 session 管理
   - 或使用 ConPTY 实现类似功能

5. **配置暴露**
   ```typescript
   // 允许用户自定义
   interface TerminalPanelConfig {
     shell?: string           // 覆盖 $SHELL
     shellArgs?: string[]     // 自定义 shell 参数
     tmuxConfig?: string[]    // 额外的 tmux 配置
   }
   ```

6. **更好的错误处理**
   - 当前创建失败仅记录调试日志
   - 建议向用户显示友好的错误提示（如 "tmux session 创建失败，请检查权限"）

### 测试建议

应覆盖以下场景：
- 有/无 tmux 的环境
- session 首次创建和重复附加
- 多实例并发（验证 socket 隔离）
- Claude Code 退出后的 tmux 清理
- 工作目录切换后的行为
- 各种 shell（bash/zsh/fish）的兼容性
- 终端大小变化的处理
