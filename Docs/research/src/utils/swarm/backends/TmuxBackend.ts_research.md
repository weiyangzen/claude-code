# TmuxBackend.ts 深度研究文档

## 场景与职责

TmuxBackend.ts 实现了 **tmux 窗口管理后端**，是 Agent Swarm 系统中最功能完整的 PaneBackend 实现。它支持两种运行模式：在 tmux 内部运行（与 leader 共享窗口）和外部运行（创建独立的 swarm 会话）。

**核心定位：**
- 跨平台支持（macOS、Linux、WSL）
- 功能最完整的后端（支持 hide/show、颜色、标题等）
- 优先级最高的后端（在 tmux 内部运行时优先使用）

**两种运行模式：**

1. **Inside Tmux 模式**（用户在 tmux 中启动 Claude）
   - Leader 保持在左侧（30% 宽度）
   - 队友在右侧垂直堆叠（70% 宽度）
   - 使用用户的原始 tmux 会话

2. **External 模式**（用户在普通终端启动 Claude）
   - 创建独立的 `claude-swarm` 会话
   - 所有队友平均分布（平铺布局）
   - 使用独立的 socket 隔离

---

## 功能点目的

### 1. 后端可用性检测
- `isAvailable()`: 检测 tmux 是否已安装
- `isRunningInside()`: 检测当前是否在 tmux 会话中

### 2. 队友窗口创建
- `createTeammatePaneInSwarmView()`: 创建队友窗口
  - Inside 模式：从 leader 或现有队友分割
  - External 模式：在独立会话中创建
- 使用锁机制防止并行创建竞态

### 3. 命令发送
- `sendCommandToPane()`: 向指定窗口发送按键命令
- 支持外部会话 socket 切换

### 4. 视觉样式
- `setPaneBorderColor()`: 设置窗口边框颜色（tmux 3.2+）
- `setPaneTitle()`: 设置窗口标题
- `enablePaneBorderStatus()`: 启用边框状态显示

### 5. 布局管理
- `rebalancePanes()`: 重新平衡窗口布局
  - With Leader: main-vertical 布局，leader 占 30%
  - Without Leader: tiled 布局，平均分布

### 6. 窗口生命周期
- `killPane()`: 关闭窗口
- `hidePane()`: 隐藏窗口（移动到隐藏会话）
- `showPane()`: 显示窗口（从隐藏会话移回）

---

## 具体技术实现

### 关键数据结构

```typescript
// 模块级状态
let firstPaneUsedForExternal = false           // 外部模式首个队友标记
let cachedLeaderWindowTarget: string | null = null  // 缓存的窗口目标
let paneCreationLock: Promise<void> = Promise.resolve()  // 创建锁

const PANE_SHELL_INIT_DELAY_MS = 200           // shell 初始化等待时间

// 颜色映射
const tmuxColors: Record<AgentColorName, string> = {
  red: 'red',
  blue: 'blue',
  green: 'green',
  yellow: 'yellow',
  purple: 'magenta',
  orange: 'colour208',
  pink: 'colour205',
  cyan: 'cyan',
}
```

### 核心流程

#### 1. 窗口创建流程（createTeammatePaneInSwarmView）

```
1. 获取 paneCreationLock
2. 检测是否在 tmux 内部
3. 如果在内部：
   a. 调用 createTeammatePaneWithLeader()
4. 如果在外部：
   a. 调用 createTeammatePaneExternal()
5. 释放锁
```

#### 2. Inside Tmux 模式创建（createTeammatePaneWithLeader）

```
1. 获取当前 pane ID（getCurrentPaneId）
2. 获取当前窗口目标（getCurrentWindowTarget）
3. 获取窗口 pane 数量
4. 判断是否为首个队友（paneCount === 1）
5. 首个队友：
   a. 从 leader pane 水平分割（-h）
   b. 右侧占 70%（-l 70%）
   c. 命令：tmux split-window -t <pane> -h -l 70% -P -F #{pane_id}
6. 后续队友：
   a. 获取所有 pane ID 列表
   b. 计算分割策略（奇数垂直/偶数水平）
   c. 选择目标 pane（轮流选择）
   d. 执行分割命令
7. 设置边框颜色和标题
8. 重新平衡布局
9. 等待 shell 初始化（200ms）
10. 返回 pane ID
```

**分割策略算法：**

```typescript
const teammatePanes = panes.slice(1)  // 排除 leader
const teammateCount = teammatePanes.length

// 奇数时垂直分割，偶数时水平分割
const splitVertically = teammateCount % 2 === 1

// 选择目标 pane（轮流选择）
const targetPaneIndex = Math.floor((teammateCount - 1) / 2)
const targetPane = teammatePanes[targetPaneIndex] || teammatePanes[teammatePanes.length - 1]
```

#### 3. External 模式创建（createTeammatePaneExternal）

```
1. 创建或获取外部 swarm 会话（createExternalSwarmSession）
   a. 检查会话是否存在（hasSessionInSwarm）
   b. 不存在则创建：tmux -L <socket> new-session -d -s claude-swarm -n swarm-view
   c. 存在则获取或创建 swarm-view 窗口
2. 获取窗口 pane 数量
3. 判断是否为首个队友（!firstPaneUsedForExternal && paneCount === 1）
4. 首个队友：
   a. 使用会话创建时的初始 pane
   b. 标记 firstPaneUsedForExternal = true
5. 后续队友：
   a. 与 Inside 模式类似的分割逻辑
   b. 使用 swarm socket 执行命令
6. 设置边框颜色和标题
7. 应用平铺布局（tiled）
8. 等待 shell 初始化
9. 返回 pane ID
```

#### 4. 命令执行封装

**用户会话命令（runTmuxInUserSession）：**
```typescript
function runTmuxInUserSession(args: string[]) {
  return execFileNoThrow('tmux', args)
}
```

**Swarm 会话命令（runTmuxInSwarm）：**
```typescript
function runTmuxInSwarm(args: string[]) {
  return execFileNoThrow('tmux', ['-L', getSwarmSocketName(), ...args])
}
```

**Socket 命名：**
```typescript
function getSwarmSocketName(): string {
  return `claude-swarm-${process.pid}`
}
```

使用 PID 确保多个 Claude 实例不冲突。

#### 5. 隐藏/显示窗口实现

**隐藏窗口（hidePane）：**
```typescript
async hidePane(paneId: PaneId, useExternalSession = false): Promise<boolean> {
  // 1. 创建隐藏会话（如果不存在）
  await runTmux(['new-session', '-d', '-s', 'claude-hidden'])
  
  // 2. 移动 pane 到隐藏会话
  const result = await runTmux([
    'break-pane', '-d', '-s', paneId, '-t', 'claude-hidden:'
  ])
  
  return result.code === 0
}
```

**显示窗口（showPane）：**
```typescript
async showPane(paneId: PaneId, targetWindowOrPane: string): Promise<boolean> {
  // 1. 将 pane 从隐藏会话移回目标窗口
  const result = await runTmux([
    'join-pane', '-h', '-s', paneId, '-t', targetWindowOrPane
  ])
  
  if (result.code !== 0) return false
  
  // 2. 重新应用布局
  await runTmux(['select-layout', '-t', targetWindowOrPane, 'main-vertical'])
  
  // 3. 调整 leader 大小为 30%
  const panes = await getPanesList(targetWindowOrPane)
  if (panes[0]) {
    await runTmux(['resize-pane', '-t', panes[0], '-x', '30%'])
  }
  
  return true
}
```

#### 6. 布局重新平衡

**With Leader 布局（rebalancePanesWithLeader）：**
```
1. 获取所有 pane ID
2. 如果只有 2 个 pane（leader + 1 队友），无需调整
3. 应用 main-vertical 布局
4. 调整 leader pane 宽度为 30%
```

**Without Leader 布局（rebalancePanesTiled）：**
```
1. 获取所有 pane ID
2. 如果只有 1 个 pane，无需调整
3. 应用 tiled 布局（自动平均分布）
```

### 颜色设置实现

```typescript
async setPaneBorderColor(paneId: PaneId, color: AgentColorName, useExternalSession = false) {
  const tmuxColor = getTmuxColorName(color)
  
  // 设置 pane 特定样式（tmux 3.2+）
  await runTmux(['select-pane', '-t', paneId, '-P', `bg=default,fg=${tmuxColor}`])
  
  // 设置边框样式
  await runTmux([
    'set-option', '-p', '-t', paneId,
    'pane-border-style', `fg=${tmuxColor}`
  ])
  
  // 设置激活边框样式
  await runTmux([
    'set-option', '-p', '-t', paneId,
    'pane-active-border-style', `fg=${tmuxColor}`
  ])
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/swarm/backends/types.ts` | `PaneBackend`, `CreatePaneResult`, `PaneId` 类型 |
| `src/utils/swarm/backends/detection.ts` | `getLeaderPaneId()`, `isInsideTmux()`, `isTmuxAvailable()` |
| `src/utils/swarm/backends/registry.ts` | `registerTmuxBackend()` |
| `src/utils/swarm/constants.ts` | `TMUX_COMMAND`, `SWARM_SESSION_NAME`, `SWARM_VIEW_WINDOW_NAME`, `HIDDEN_SESSION_NAME`, `getSwarmSocketName()` |
| `src/utils/execFileNoThrow.ts` | 安全执行 tmux 命令 |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/log.ts` | `logError()` |
| `src/utils/array.ts` | `count()` |
| `src/utils/sleep.ts` | `sleep()` |
| `src/tools/AgentTool/agentColorManager.ts` | `AgentColorName` 类型 |

### 外部命令

| 命令 | 用途 |
|-----|------|
| `tmux split-window -t <pane> -h -l 70%` | 水平分割创建队友窗口 |
| `tmux split-window -t <pane> -v` | 垂直分割 |
| `tmux send-keys -t <pane> <cmd> Enter` | 发送命令 |
| `tmux kill-pane -t <pane>` | 关闭窗口 |
| `tmux break-pane -d -s <pane> -t <session>:` | 隐藏窗口 |
| `tmux join-pane -h -s <pane> -t <window>` | 显示窗口 |
| `tmux select-layout -t <window> main-vertical/tiled` | 设置布局 |
| `tmux resize-pane -t <pane> -x 30%` | 调整大小 |
| `tmux new-session -d -s <name> -n <window>` | 创建会话 |
| `tmux -L <socket>` | 使用指定 socket |

### 关键代码位置

- **类定义**: 行 104-759
- **锁获取**: 行 43-53
- **窗口创建（主入口）**: 行 129-146
- **Inside 模式创建**: 行 551-630
- **External 模式创建**: 行 635-702
- **外部会话创建**: 行 467-546
- **布局重新平衡**: 行 707-758
- **隐藏/显示窗口**: 行 281-361
- **颜色设置**: 行 169-203
- **后端注册**: 行 764

---

## 依赖与外部交互

### 运行时依赖

1. **tmux**: 必须已安装且在 PATH 中
2. **shell**: 队友窗口运行 shell 初始化（~/.bashrc, ~/.zshrc 等）

### 环境变量

| 变量 | 说明 |
|-----|------|
| `TMUX` | tmux 环境变量（模块加载时捕获） |
| `TMUX_PANE` | 当前 pane ID（模块加载时捕获） |

### 与 Registry 的交互

```typescript
// 模块加载时自动注册
registerTmuxBackend(TmuxBackend)
```

### 与 PaneBackendExecutor 的关系

TmuxBackend 被 `PaneBackendExecutor` 包装：

```typescript
const backend = new TmuxBackend()
const executor = new PaneBackendExecutor(backend)
```

### 常量定义

```typescript
// src/utils/swarm/constants.ts
export const SWARM_SESSION_NAME = 'claude-swarm'
export const SWARM_VIEW_WINDOW_NAME = 'swarm-view'
export const HIDDEN_SESSION_NAME = 'claude-hidden'
export const TMUX_COMMAND = 'tmux'
```

---

## 风险、边界与改进建议

### 已知风险

1. **tmux 版本兼容性**
   - Pane 特定选项（`-p` 标志）需要 tmux 3.2+
   - 旧版本可能不支持边框颜色设置
   - **缓解**: 命令失败不会阻止队友创建

2. **Shell 初始化延迟**
   - 固定 200ms 等待可能不足（如使用 starship/oh-my-zsh 的慢配置）
   - **影响**: 命令可能在 shell 就绪前发送

3. **Socket 冲突**
   - 使用 PID 命名 socket，但 PID 可能复用
   - **影响**: 极低概率的 socket 冲突

4. **隐藏会话累积**
   - 隐藏的 pane 如果不手动清理会一直存在
   - **影响**: 资源泄漏

5. **布局状态不一致**
   - 用户手动调整布局后，rebalance 会重置
   - **影响**: 用户体验不一致

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| 无法获取 pane/window ID | 抛出错误 |
| 分割命令失败 | 抛出错误，包含 stderr |
| 隐藏会话已存在 | 复用现有会话 |
| Swarm 会话已存在 | 复用现有会话，检查窗口 |
| Leader 窗口切换 | 使用缓存的窗口目标 |
| 用户切换 pane | 使用模块加载时捕获的 TMUX_PANE |

### 改进建议

1. **动态 Shell 就绪检测**
   - 替代固定 200ms 等待
   - 检测 shell 提示符出现后再发送命令

2. **tmux 版本检测**
   - 运行时检测 tmux 版本
   - 对旧版本禁用不支持的特性

3. **隐藏会话自动清理**
   - 定期清理空的隐藏会话
   - 或提供手动清理命令

4. **布局持久化**
   - 记住用户手动调整后的布局
   - 只在新增队友时调整

5. **错误恢复增强**
   - 对临时性 tmux 错误实现重试
   - 更好的错误消息（区分权限问题、tmux 未运行等）

6. **性能优化**
   - 批量执行 tmux 命令
   - 减少子进程调用次数

7. **配置选项**
   - 允许用户自定义布局比例（非固定 30/70）
   - 允许禁用颜色设置
