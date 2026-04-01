# InProcessBackend.ts 深度研究文档

## 场景与职责

InProcessBackend.ts 实现了 **进程内队友执行后端**，是 Agent Swarm 系统的三大执行模式之一。与基于窗口的后端（tmux/iTerm2）不同，进程内队友在与 leader 相同的 Node.js 进程中运行，通过 AsyncLocalStorage 实现上下文隔离。

**核心定位：**
- 作为无 tmux/iTerm2 环境时的默认执行模式
- 适用于非交互式会话（如 `-p` 模式）
- 提供更轻量级的队友执行，无需外部依赖

**关键特性：**
- 共享 leader 的资源（API 客户端、MCP 连接）
- 通过文件邮箱进行通信（与窗口队友相同）
- 通过 AbortController 终止（而非 kill-pane）
- 支持 graceful shutdown 流程

---

## 功能点目的

### 1. 后端生命周期管理
- `isAvailable()`: 始终返回 true（无外部依赖）
- `setContext()`: 设置 ToolUseContext，提供 AppState 访问能力

### 2. 队友创建与启动
- `spawn()`: 创建并启动进程内队友
  - 创建 TeammateContext（AsyncLocalStorage 上下文）
  - 创建独立的 AbortController
  - 在 AppState.tasks 中注册任务
  - 启动 agent 执行循环

### 3. 消息传递
- `sendMessage()`: 通过文件邮箱向队友发送消息
- 使用统一的 mailbox 系统，与窗口队友兼容

### 4. 生命周期控制
- `terminate()`: 请求优雅关闭（发送 shutdown request）
- `kill()`: 强制终止（通过 AbortController）
- `isActive()`: 检查队友是否仍在运行

---

## 具体技术实现

### 关键数据结构

```typescript
class InProcessBackend implements TeammateExecutor {
  readonly type = 'in-process' as const
  private context: ToolUseContext | null = null  // AppState 访问入口
}

// TeammateSpawnConfig（来自 types.ts）
interface TeammateSpawnConfig {
  name: string                    // 队友名称
  teamName: string               // 所属团队
  prompt: string                 // 初始提示
  color?: string                 // UI 颜色
  cwd: string                    // 工作目录
  model?: string                 // 模型覆盖
  systemPrompt?: string          // 系统提示
  systemPromptMode?: 'default' | 'replace' | 'append'
  parentSessionId: string        // 父会话 ID
  permissions?: string[]         // 工具权限
  allowPermissionPrompts?: boolean
}

// TeammateSpawnResult（来自 types.ts）
interface TeammateSpawnResult {
  success: boolean
  agentId: string
  error?: string
  abortController?: AbortController
  taskId?: string
  paneId?: string  // in-process 模式下为 undefined
}
```

### 核心流程

#### 1. 队友创建流程（spawn）

```
1. 验证 context 已设置（必须调用 setContext()）
2. 调用 spawnInProcessTeammate() 创建基础设施：
   a. 生成 agentId（格式: name@teamName）
   b. 生成 taskId
   c. 创建独立 AbortController
   d. 创建 TeammateContext（AsyncLocalStorage）
   e. 在 AppState 中注册 InProcessTeammateTaskState
3. 如果创建成功，调用 startInProcessTeammate() 启动执行：
   a. 构建系统提示（包含队友附加说明）
   b. 设置工具权限
   c. 在 teammate context 中运行 runAgent()
4. 返回 spawn 结果
```

**关键代码（行 72-143）：**

```typescript
async spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult> {
  // 验证 context
  if (!this.context) {
    return { success: false, agentId, error: '...' }
  }

  // 创建基础设施
  const result = await spawnInProcessTeammate({...}, this.context)

  // 启动执行循环
  if (result.success) {
    startInProcessTeammate({
      identity: {...},
      taskId: result.taskId,
      prompt: config.prompt,
      teammateContext: result.teammateContext,
      toolUseContext: { ...this.context, messages: [] },  // 剥离父消息
      abortController: result.abortController,
      model: config.model,
      systemPrompt: config.systemPrompt,
      allowedTools: config.permissions,
      // ...
    })
  }

  return result
}
```

#### 2. 消息发送流程（sendMessage）

```
1. 解析 agentId 获取 agentName 和 teamName
2. 调用 writeToMailbox() 写入文件邮箱
3. 队友通过轮询读取消息
```

**关键代码（行 150-180）：**

```typescript
async sendMessage(agentId: string, message: TeammateMessage): Promise<void> {
  const parsed = parseAgentId(agentId)  // 解析 name@teamName
  await writeToMailbox(agentName, {
    text: message.text,
    from: message.from,
    color: message.color,
    timestamp: message.timestamp ?? new Date().toISOString(),
  }, teamName)
}
```

#### 3. 优雅关闭流程（terminate）

```
1. 从 AppState 查找对应任务
2. 检查是否已有待处理的 shutdown 请求
3. 生成确定性 request ID
4. 创建 shutdown request 消息
5. 写入队友邮箱
6. 标记任务状态为 shutdownRequested
7. 队友检测到请求后决定是否退出
```

**关键代码（行 192-253）：**

```typescript
async terminate(agentId: string, reason?: string): Promise<boolean> {
  const task = findTeammateTaskByAgentId(agentId, state.tasks)
  
  // 避免重复发送
  if (task.shutdownRequested) return true

  const requestId = `shutdown-${agentId}-${Date.now()}`
  const shutdownRequest = createShutdownRequestMessage({
    requestId,
    from: 'team-lead',
    reason,
  })

  // 发送到队友邮箱
  await writeToMailbox(teammateAgentName, {
    from: 'team-lead',
    text: jsonStringify(shutdownRequest),
    timestamp: new Date().toISOString(),
  }, task.identity.teamName)

  // 标记状态
  requestTeammateShutdown(task.id, this.context.setAppState)
  return true
}
```

#### 4. 强制终止流程（kill）

```
1. 从 AppState 查找对应任务
2. 调用 killInProcessTeammate() 执行终止：
   a. 调用 abortController.abort()
   b. 更新任务状态为 'killed'
   c. 从 teamContext.teammates 中移除
   d. 清理 Perfetto 追踪注册
   e. 触发 SDK 事件
```

**关键代码（行 261-290）：**

```typescript
async kill(agentId: string): Promise<boolean> {
  const state = this.context.getAppState()
  const task = findTeammateTaskByAgentId(agentId, state.tasks)
  
  if (!task) return false
  
  const killed = killInProcessTeammate(task.id, this.context.setAppState)
  return killed
}
```

#### 5. 活跃状态检查（isActive）

```
1. 从 AppState 查找对应任务
2. 检查条件：
   - 任务存在
   - 状态为 'running'
   - AbortController 未被中止
```

**关键代码（行 298-330）：**

```typescript
async isActive(agentId: string): Promise<boolean> {
  const task = findTeammateTaskByAgentId(agentId, state.tasks)
  if (!task) return false

  const isRunning = task.status === 'running'
  const isAborted = task.abortController?.signal.aborted ?? true
  return isRunning && !isAborted
}
```

### 与 PaneBackend 的关键差异

| 特性 | InProcessBackend | PaneBackend (tmux/iTerm2) |
|-----|------------------|--------------------------|
| 执行环境 | 同进程（AsyncLocalStorage） | 独立子进程 |
| 资源隔离 | 共享 leader 资源 | 独立资源 |
| 终止方式 | AbortController | kill-pane / it2 close |
| 通信方式 | 文件邮箱 | 文件邮箱 |
| UI 可见性 | 无独立窗口 | 独立窗口/分屏 |
| 启动延迟 | 低（无进程创建） | 较高 |
| 适用场景 | 后台任务、非交互式 | 可视化协作 |

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/swarm/backends/types.ts` | TeammateExecutor 接口定义 |
| `src/utils/swarm/inProcessRunner.ts` | `startInProcessTeammate()` - 启动执行循环 |
| `src/utils/swarm/spawnInProcess.ts` | `spawnInProcessTeammate()`, `killInProcessTeammate()` |
| `src/utils/teammateMailbox.ts` | `writeToMailbox()`, `createShutdownRequestMessage()` |
| `src/utils/agentId.ts` | `parseAgentId()`, `formatAgentId()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/slowOperations.ts` | `jsonStringify()` |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.ts` | `findTeammateTaskByAgentId()`, `requestTeammateShutdown()` |
| `src/Tool.ts` | `ToolUseContext` 类型 |

### 关键代码位置

- **类定义**: 行 38-331
- **工厂函数**: 行 337-339
- **spawn 方法**: 行 72-143
- **sendMessage 方法**: 行 150-180
- **terminate 方法**: 行 192-253
- **kill 方法**: 行 261-290
- **isActive 方法**: 行 298-330

---

## 依赖与外部交互

### 与 InProcessTeammateTask 的交互

InProcessBackend 与 `InProcessTeammateTask` 组件紧密协作：

1. **spawnInProcessTeammate** 创建任务状态并注册到 AppState
2. **startInProcessTeammate** 在任务上下文中运行 agent 循环
3. **killInProcessTeammate** 更新任务状态并清理资源

### 与 Mailbox 系统的交互

所有队友（无论进程内还是窗口）使用统一的文件邮箱系统：

```
Mailbox 路径: ~/.claude/mailbox/<team-name>/<agent-name>.jsonl
```

### 与 Registry 的交互

```typescript
// 通过工厂函数创建实例
export function createInProcessBackend(): InProcessBackend {
  return new InProcessBackend()
}
```

与 PaneBackend 不同，InProcessBackend 不自注册，而是由 `registry.ts` 直接调用工厂函数创建。

### 与 Leader Permission Bridge 的交互

进程内队友通过 `leaderPermissionBridge.ts` 访问 leader 的权限确认队列：

```typescript
// 在 inProcessRunner.ts 中
const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()
if (setToolUseConfirmQueue) {
  // 使用 leader 的 UI 显示权限请求
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **Context 未设置风险**
   - `spawn()` 必须在 `setContext()` 之后调用
   - 否则返回错误：`"InProcessBackend not initialized..."`
   - **缓解**: 调用方（TeammateTool）确保顺序

2. **资源竞争**
   - 进程内队友共享 leader 的 API 客户端和 MCP 连接
   - 高并发时可能导致资源争用
   - **缓解**: 当前限制并行队友数量

3. **内存泄漏风险**
   - 任务状态长期保留在 AppState 中
   - 消息历史可能无限增长
   - **缓解**: 定期 compact 和任务清理机制

4. **错误隔离不足**
   - 队友错误可能影响 leader（同进程）
   - 未捕获的异常可能导致整个进程崩溃
   - **缓解**: 使用 AsyncLocalStorage 和 try/catch 包装

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| Context 未设置 | 返回错误，不创建队友 |
| 任务不存在（terminate/kill） | 返回 false |
| 重复 terminate 请求 | 检查 shutdownRequested 标志，避免重复发送 |
| AbortController 已中止 | isActive 返回 false |
| 无效的 agentId 格式 | 抛出错误 |

### 改进建议

1. **资源配额管理**
   - 实现 CPU/内存使用限制
   - 防止单个队友耗尽资源

2. **更好的错误隔离**
   - 使用 Worker Threads 替代 AsyncLocalStorage
   - 真正的进程隔离，同时保持低延迟

3. **自动清理策略**
   - 根据消息数量或时间自动清理旧任务
   - 防止 AppState 无限增长

4. **性能监控**
   - 添加队友执行时间、内存使用监控
   - 帮助识别资源密集型队友

5. **优雅关闭增强**
   - 当前仅发送请求，不强制超时
   - 可考虑添加强制终止超时机制

6. **与 PaneBackend 功能对等**
   - 当前缺少一些可视化功能
   - 可考虑添加虚拟 UI 表示
