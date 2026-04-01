# ITermBackend.ts 深度研究文档

## 场景与职责

ITermBackend.ts 实现了 **iTerm2 原生分屏后端**，用于在 macOS iTerm2 终端中创建和管理队友（teammate）窗口分屏。它是 Agent Swarm 系统的三大后端实现之一（另两个是 TmuxBackend 和 InProcessBackend）。

**核心定位：**
- 专为 macOS iTerm2 用户设计，利用 iTerm2 的原生分屏功能
- 通过 `it2` CLI 工具与 iTerm2 的 Python API 交互
- 提供与 tmux 类似的窗口分屏体验，但使用 iTerm2 原生机制

**适用场景：**
- 用户在 macOS 上使用 iTerm2 作为终端
- 已安装并配置 `it2` CLI 工具（`pip install it2`）
- 已在 iTerm2 设置中启用 Python API

---

## 功能点目的

### 1. 后端可用性检测
- `isAvailable()`: 检测当前是否在 iTerm2 环境中且 `it2` CLI 可用
- `isRunningInside()`: 检测当前进程是否运行在 iTerm2 终端内

### 2. 队友窗口创建与管理
- `createTeammatePaneInSwarmView()`: 创建新的队友分屏窗口
  - 首个队友：从 leader 窗口垂直分割（-v），左侧 leader（30%），右侧队友（70%）
  - 后续队友：从最后一个队友窗口水平分割（-h），垂直堆叠
- 使用锁机制（`paneCreationLock`）防止并行创建时的竞态条件

### 3. 命令发送与执行
- `sendCommandToPane()`: 向指定分屏发送命令执行
- 使用 `it2 session run -s <session-id> <command>` 格式

### 4. 窗口生命周期管理
- `killPane()`: 强制关闭分屏（使用 `-f` 强制标志绕过确认对话框）
- `hidePane()` / `showPane()`: iTerm2 不支持，返回 false
- 自动清理死亡会话 ID，防止从已关闭窗口分割

### 5. 视觉样式（性能优化后禁用）
- `setPaneBorderColor()`: 空实现（性能考虑，每次 it2 调用都会启动 Python 进程）
- `setPaneTitle()`: 空实现
- `enablePaneBorderStatus()`: 空实现
- `rebalancePanes()`: 空实现（iTerm2 自动处理布局）

---

## 具体技术实现

### 关键数据结构

```typescript
// 模块级状态跟踪
teammateSessionIds: string[]           // 跟踪所有队友会话 ID
firstPaneUsed: boolean                 // 是否已创建首个队友窗口
paneCreationLock: Promise<void>        // 创建锁，防止竞态条件
```

### 核心流程

#### 1. 窗口创建流程（createTeammatePaneInSwarmView）

```
1. 获取 paneCreationLock
2. 判断是否为首个队友（!firstPaneUsed）
3. 首个队友：
   - 从 ITERM_SESSION_ID 提取 leader 会话 ID
   - 执行：it2 session split -v -s <leader-session-id>
4. 后续队友：
   - 获取最后一个队友的 session ID
   - 执行：it2 session split -s <teammate-session-id>
5. 解析输出获取新 pane ID："Created new pane: <uuid>"
6. 更新 teammateSessionIds 数组
7. 释放锁
```

#### 2. 故障恢复机制（At-fault Recovery）

当分割命令失败时，系统会检测目标会话是否已死亡：

```typescript
// 伪代码流程
if (splitResult.code !== 0 && targetedTeammateId) {
  // 检查目标会话是否仍然存在
  const listResult = await runIt2(['session', 'list'])
  if (!listResult.stdout.includes(targetedTeammateId)) {
    // 确认死亡，从跟踪数组中移除
    teammateSessionIds.splice(idx, 1)
    // 重试创建
    continue
  }
}
```

**复杂度保证：** 每次失败都会减少 teammateSessionIds 长度，最多 O(N+1) 次迭代。

#### 3. 会话 ID 解析

```typescript
function parseSplitOutput(output: string): string {
  const match = output.match(/Created new pane:\s*(.+)/)
  return match?.[1]?.trim() ?? ''
}
```

#### 4. Leader 会话 ID 提取

```typescript
function getLeaderSessionId(): string | null {
  // ITERM_SESSION_ID 格式: "wXtYpZ:UUID"
  // 提取冒号后的 UUID 部分
  const itermSessionId = process.env.ITERM_SESSION_ID
  const colonIndex = itermSessionId.indexOf(':')
  return itermSessionId.slice(colonIndex + 1)
}
```

### 锁机制实现

```typescript
let paneCreationLock: Promise<void> = Promise.resolve()

function acquirePaneCreationLock(): Promise<() => void> {
  let release: () => void
  const newLock = new Promise<void>(resolve => {
    release = resolve
  })
  const previousLock = paneCreationLock
  paneCreationLock = newLock
  return previousLock.then(() => release!)
}
```

**特点：**
- 使用 Promise 链实现 FIFO 队列
- 返回 release 函数，支持 try/finally 模式
- 确保即使发生异常也能释放锁

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/swarm/backends/detection.ts` | 环境检测（`isInITerm2`, `isIt2CliAvailable`） |
| `src/utils/swarm/backends/registry.ts` | 后端注册（`registerITermBackend`） |
| `src/utils/swarm/backends/types.ts` | 类型定义（`PaneBackend`, `CreatePaneResult`） |
| `src/utils/execFileNoThrow.ts` | 安全执行外部命令 |
| `src/utils/debug.ts` | 调试日志（`logForDebugging`） |
| `src/tools/AgentTool/agentColorManager.ts` | 颜色类型定义 |

### 外部命令

| 命令 | 用途 |
|-----|------|
| `it2 session split -v -s <id>` | 垂直分割创建新窗口 |
| `it2 session split -s <id>` | 水平分割创建新窗口 |
| `it2 session run -s <id> <cmd>` | 在指定窗口执行命令 |
| `it2 session close -f -s <id>` | 强制关闭窗口 |
| `it2 session list` | 列出所有会话 |

### 关键代码位置

- **类定义**: 行 79-365
- **锁获取**: 行 21-31
- **窗口创建**: 行 114-240
- **命令发送**: 行 245-264
- **窗口关闭**: 行 320-339
- **后端注册**: 行 370

---

## 依赖与外部交互

### 运行时依赖

1. **iTerm2 终端**: 必须在 iTerm2 中运行
2. **it2 CLI**: Python 包 `it2` 必须已安装
3. **Python API**: iTerm2 设置中必须启用 Python API

### 环境变量

| 变量 | 说明 |
|-----|------|
| `ITERM_SESSION_ID` | iTerm2 会话标识，格式 `wXtYpZ:UUID` |

### 与 Registry 的交互

```typescript
// 模块加载时自动注册
registerITermBackend(ITermBackend)
```

这种自注册模式避免了循环依赖问题。

### 与 PaneBackendExecutor 的关系

ITermBackend 实现 `PaneBackend` 接口，被 `PaneBackendExecutor` 包装后提供给上层使用。`PaneBackendExecutor` 负责：
- 构建启动命令
- 管理队友映射关系
- 提供统一的 `TeammateExecutor` 接口

---

## 风险、边界与改进建议

### 已知风险

1. **性能问题**
   - 每次 `it2` 调用都会启动 Python 进程，延迟较高
   - 因此跳过了颜色和标题设置等视觉功能
   - **影响**: 用户体验略逊于 tmux 后端

2. **Python API 依赖**
   - 需要用户在 iTerm2 设置中手动启用 Python API
   - 未启用时后端不可用，会回退到 tmux 或 in-process 模式

3. **会话 ID 失效**
   - 用户手动关闭窗口（Cmd+W）会导致会话 ID 失效
   - 已实现故障恢复，但首次创建会失败

4. **平台限制**
   - 仅支持 macOS（iTerm2 是 macOS 专用）
   - 不支持 hide/show 功能（iTerm2 无等效机制）

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| 无法获取 leader session ID | 回退到 active session 分割 |
| 目标队友会话已死亡 | 检测并清理，重试创建 |
| 并行创建请求 | 使用锁机制串行化 |
| it2 命令失败 | 抛出错误，包含 stderr |

### 改进建议

1. **批量操作优化**
   - 当前每个操作都是独立的 it2 进程调用
   - 可考虑批量发送命令减少进程启动开销

2. **视觉反馈增强**
   - 使用 iTerm2 的 escape sequences 替代 it2 CLI 设置颜色
   - 可减少 Python 进程启动次数

3. **错误恢复增强**
   - 当前仅处理死亡会话的情况
   - 可增加重试机制和指数退避

4. **功能对等**
   - 研究 iTerm2 是否支持类似 tmux break-pane/join-pane 的功能
   - 实现 hide/show 功能以与 tmux 后端功能对等

5. **配置持久化**
   - 考虑记住用户对 it2 设置提示的选择
   - 避免每次启动都提示安装 it2
