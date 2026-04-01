# LocalAgentTask.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位

`LocalAgentTask` 是 Claude Code 中负责**本地后台 Agent 任务生命周期管理**的核心模块。它实现了 `Task` 接口，专门处理类型为 `'local_agent'` 的后台任务，是 Agent 工具（AgentTool）执行异步/后台子代理的基础设施。

### 1.2 主要使用场景

| 场景 | 描述 |
|------|------|
| **后台 Agent 执行** | 用户通过 `Agent` 工具启动 `run_in_background=true` 的子代理 |
| **前台 Agent 后台化** | 长时间运行的前台 Agent 超过阈值后自动转为后台执行 |
| **Agent 恢复执行** | 通过 `SendMessage` 工具向已停止的 Agent 发送消息触发恢复 |
| **Coordinator 模式** | 协调器模式下管理多个并行子代理任务 |
| **Fork 子代理** | Fork 路径的异步子代理生命周期管理 |

### 1.3 与相关模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        AgentTool.tsx                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  同步 Agent  │  │  异步 Agent  │  │  前台→后台转换           │  │
│  │  (直接返回)  │  │ (registerAsyncAgent)│ (registerAgentForeground)│  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
│         └─────────────────┴─────────────────────┘                │
│                         │                                       │
│                         ▼                                       │
│              LocalAgentTask.tsx (本模块)                         │
│         ┌──────────────────────────────┐                       │
│         │  • 任务注册/状态管理           │                       │
│         │  • 进度追踪                   │                       │
│         │  • 生命周期控制               │                       │
│         │  • 通知队列                   │                       │
│         └──────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│   AgentTool   │      │  SendMessage  │      │  Coordinator  │
│  (agentToolUtils)    │   (resume)    │      │   TaskPanel   │
└───────────────┘      └───────────────┘      └───────────────┘
```

---

## 2. 功能点目的

### 2.1 任务状态管理

**目的**：维护本地 Agent 任务的完整状态机

**状态流转**：
```
registerAsyncAgent/registerAgentForeground
         │
         ▼
    ┌─────────┐
    │ running │◄────────────────────────┐
    └────┬────┘                         │
         │                              │
    ┌────┴────┬─────────┐               │
    ▼         ▼         ▼               │
completed   failed    killed            │
    │         │         │               │
    └─────────┴─────────┘               │
              │                         │
              ▼                         │
    ┌─────────────────┐                 │
    │  evictAfter 设置 │                 │
    │ (PANEL_GRACE_MS) │                 │
    └────────┬────────┘                 │
             │                          │
             ▼                          │
    ┌─────────────────┐                 │
    │  从 AppState 驱逐 │                 │
    └─────────────────┘                 │
                                        │
              SendMessage 恢复 ──────────┘
```

### 2.2 进度追踪 (Progress Tracking)

**目的**：实时追踪 Agent 执行进度，为 UI 和 SDK 提供数据

**追踪指标**：
- `toolUseCount`: 工具调用次数
- `tokenCount`: 输入 + 输出 token 总数
- `lastActivity`: 最近活动描述
- `recentActivities`: 最近 5 个活动（循环队列）
- `summary`: 后台总结（由 AgentSummary 服务生成）

### 2.3 前台/后台模式切换

**目的**：支持 Agent 从前台执行转为后台执行

**关键设计**：
- **前台模式**：`isBackgrounded: false`，阻塞主会话，显示 BackgroundHint UI
- **后台模式**：`isBackgrounded: true`，非阻塞，通过 `task_notification` 通知结果
- **自动后台化**：超过 `PROGRESS_THRESHOLD_MS` (2秒) 后显示提示，可配置 `autoBackgroundMs` (默认 120秒)

### 2.4 任务保留 (Retain) 机制

**目的**：支持 UI 查看 Agent 完整对话记录

**机制**：
- `retain: true`：UI 正在查看该 Agent，阻止驱逐，启用流式追加
- `retain: false`：终端状态任务进入驱逐倒计时 (`evictAfter`)
- `diskLoaded: true`：已从磁盘加载历史消息到内存

### 2.5 挂起消息队列

**目的**：处理 Agent 在工具轮次间接收到的消息

**流程**：
1. `SendMessage` 调用 `queuePendingMessage` 将消息加入 `pendingMessages`
2. Agent 完成当前工具调用后，`drainPendingMessages` 取出消息
3. 消息作为用户输入注入 Agent 对话

---

## 3. 具体技术实现

### 3.1 核心数据结构

#### 3.1.1 LocalAgentTaskState

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string                    // Agent 唯一标识
  prompt: string                     // 初始提示词
  selectedAgent?: AgentDefinition    // Agent 定义
  agentType: string                  // Agent 类型标识
  model?: string                     // 使用的模型
  abortController?: AbortController  // 取消控制器
  unregisterCleanup?: () => void     // 清理函数注销器
  error?: string                     // 错误信息
  result?: AgentToolResult           // 执行结果
  progress?: AgentProgress           // 进度信息
  retrieved: boolean                 // 是否已检索
  messages?: Message[]               // 对话消息（仅 retain=true 时存在）
  lastReportedToolCount: number      // 上次报告的工具数
  lastReportedTokenCount: number     // 上次报告的 token 数
  isBackgrounded: boolean            // 是否已后台化
  pendingMessages: string[]          // 挂起消息队列
  retain: boolean                    // UI 是否持有
  diskLoaded: boolean                // 是否从磁盘加载
  evictAfter?: number                // 驱逐时间戳
}
```

#### 3.1.2 AgentProgress

```typescript
type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity?: ToolActivity        // 最近活动
  recentActivities?: ToolActivity[]  // 最近活动列表
  summary?: string                   // 后台总结
}
```

#### 3.1.3 ProgressTracker

```typescript
type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number          // 最新输入 token（API 返回累计值）
  cumulativeOutputTokens: number     // 累计输出 token
  recentActivities: ToolActivity[]   // 最近活动（最多 5 个）
}
```

### 3.2 关键流程

#### 3.2.1 注册异步 Agent (registerAsyncAgent)

```typescript
export function registerAsyncAgent({
  agentId,
  description,
  prompt,
  selectedAgent,
  setAppState,
  parentAbortController,  // 可选父控制器
  toolUseId
}): LocalAgentTaskState
```

**执行步骤**：
1. 初始化任务输出文件（创建指向 Agent 对话记录的符号链接）
2. 创建 AbortController（支持父子控制器链）
3. 构建 `LocalAgentTaskState` 初始状态
4. 注册清理处理器（应用退出时自动 kill）
5. 调用 `registerTask` 注册到 AppState

**代码路径**：`src/tasks/LocalAgentTask/LocalAgentTask.tsx:466-515`

#### 3.2.2 注册前台 Agent (registerAgentForeground)

```typescript
export function registerAgentForeground({
  agentId,
  description,
  prompt,
  selectedAgent,
  setAppState,
  autoBackgroundMs,      // 自动后台化超时
  toolUseId
}): { taskId, backgroundSignal, cancelAutoBackground }
```

**特殊机制**：
- 创建 `backgroundSignal` Promise，用于前台→后台转换信号
- 支持 `autoBackgroundMs` 自动超时后台化
- 返回 `cancelAutoBackground` 用于取消自动后台化

**代码路径**：`src/tasks/LocalAgentTask/LocalAgentTask.tsx:526-614`

#### 3.2.3 后台化 Agent (backgroundAgentTask)

```typescript
export function backgroundAgentTask(
  taskId: string,
  getAppState: () => AppState,
  setAppState: SetAppState
): boolean
```

**执行步骤**：
1. 验证任务存在且未后台化
2. 更新 `isBackgrounded: true`
3. 解析 `backgroundSignalResolvers` 中的 Promise，触发后台化流程

**调用方**：
- `useSessionBackgrounding` hook（用户主动后台化）
- `autoBackgroundMs` 超时自动触发

#### 3.2.4 进度更新流程

```typescript
// 从消息更新进度
export function updateProgressFromMessage(
  tracker: ProgressTracker,
  message: Message,
  resolveActivityDescription?: ActivityDescriptionResolver,
  tools?: Tools
): void

// 更新 Agent 进度到 AppState
export function updateAgentProgress(
  taskId: string,
  progress: AgentProgress,
  setAppState: SetAppState
): void

// 更新 Agent 总结
export function updateAgentSummary(
  taskId: string,
  summary: string,
  setAppState: SetAppState
): void
```

**进度计算逻辑**：
- 输入 token：取最新值（API 返回累计值）
- 输出 token：累加每轮输出
- 活动分类：通过 `getToolSearchOrReadInfo` 判断搜索/读取操作

#### 3.2.5 任务完成/失败/终止

```typescript
// 正常完成
export function completeAgentTask(result: AgentToolResult, setAppState: SetAppState): void

// 失败
export function failAgentTask(taskId: string, error: string, setAppState: SetAppState): void

// 终止
export function killAsyncAgent(taskId: string, setAppState: SetAppState): void
```

**共同行为**：
1. 设置终端状态（`completed`/`failed`/`killed`）
2. 设置 `endTime`
3. 如未 `retain`，设置 `evictAfter = Date.now() + PANEL_GRACE_MS` (30秒)
4. 清理 `abortController` 和 `unregisterCleanup`
5. 调用 `evictTaskOutput` 刷新并清理内存中的输出缓存

#### 3.2.6 通知入队 (enqueueAgentNotification)

```typescript
export function enqueueAgentNotification({
  taskId,
  description,
  status,              // 'completed' | 'failed' | 'killed'
  error,
  setAppState,
  finalMessage,
  usage,               // { totalTokens, toolUses, durationMs }
  toolUseId,
  worktreePath,
  worktreeBranch
}): void
```

**功能**：
1. 原子性检查并设置 `notified` 标志，防止重复通知
2. 中止任何活跃的推测（`abortSpeculation`）
3. 构建 XML 格式的 `task_notification` 消息
4. 调用 `enqueuePendingNotification` 入队

**XML 格式**：
```xml
<task-notification>
  <task-id>{taskId}</task-id>
  <tool-use-id>{toolUseId}</tool-use-id>
  <output-file>{outputPath}</output-file>
  <status>{status}</status>
  <summary>{summary}</summary>
  <result>{finalMessage}</result>
  <usage>
    <total_tokens>{totalTokens}</total_tokens>
    <tool_uses>{toolUses}</tool_uses>
    <duration_ms>{durationMs}</duration_ms>
  </usage>
  <worktree>
    <worktreePath>{worktreePath}</worktreePath>
    <worktreeBranch>{worktreeBranch}</worktreeBranch>
  </worktree>
</task-notification>
```

### 3.3 协议与接口

#### 3.3.1 Task 接口实现

```typescript
export const LocalAgentTask: Task = {
  name: 'LocalAgentTask',
  type: 'local_agent',
  async kill(taskId, setAppState) {
    killAsyncAgent(taskId, setAppState)
  }
}
```

#### 3.3.2 类型守卫

```typescript
// 判断是否为 LocalAgentTask
export function isLocalAgentTask(task: unknown): task is LocalAgentTaskState

// 判断是否为面板管理的 Agent 任务（排除 main-session）
export function isPanelAgentTask(t: unknown): t is LocalAgentTaskState
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件 | 职责 |
|------|------|
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 本模块，Agent 任务生命周期管理 |
| `src/tasks/types.ts` | TaskState 联合类型定义 |
| `src/Task.ts` | Task 接口和基础类型定义 |
| `src/utils/task/framework.ts` | 任务框架基础设施（registerTask, updateTaskState, PANEL_GRACE_MS） |
| `src/utils/task/diskOutput.ts` | 任务输出文件管理（DiskTaskOutput, evictTaskOutput） |
| `src/utils/task/sdkProgress.ts` | SDK 进度事件发射 |

### 4.2 调用方文件

| 文件 | 调用函数 | 用途 |
|------|----------|------|
| `src/tools/AgentTool/AgentTool.tsx` | `registerAsyncAgent`, `registerAgentForeground`, `backgroundAgentTask`, `completeAgentTask`, `failAgentTask`, `killAsyncAgent`, `enqueueAgentNotification` | Agent 工具执行和生命周期管理 |
| `src/tools/AgentTool/agentToolUtils.ts` | `createProgressTracker`, `updateProgressFromMessage`, `getProgressUpdate`, `getTokenCountFromTracker`, `createActivityDescriptionResolver`, `isLocalAgentTask` | 进度追踪工具函数 |
| `src/tools/AgentTool/resumeAgent.ts` | `registerAsyncAgent`, `updateAgentProgress` | Agent 恢复执行 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `isLocalAgentTask`, `queuePendingMessage` | 向 Agent 发送消息 |
| `src/services/AgentSummary/agentSummary.ts` | `updateAgentSummary` | 更新 Agent 进度总结 |
| `src/state/teammateViewHelpers.ts` | `isLocalAgentTask` (内联实现避免循环依赖) | 进入/退出 Agent 视图 |
| `src/components/CoordinatorAgentStatus.tsx` | `isPanelAgentTask`, `getVisibleAgentTasks` | Coordinator 面板渲染 |

### 4.3 关键代码片段

#### 4.3.1 进度追踪器创建与更新

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:50-96
export function createProgressTracker(): ProgressTracker {
  return {
    toolUseCount: 0,
    latestInputTokens: 0,
    cumulativeOutputTokens: 0,
    recentActivities: []
  }
}

export function updateProgressFromMessage(
  tracker: ProgressTracker,
  message: Message,
  resolveActivityDescription?: ActivityDescriptionResolver,
  tools?: Tools
): void {
  if (message.type !== 'assistant') return
  const usage = message.message.usage
  // 输入 token 取最新（API 返回累计值）
  tracker.latestInputTokens = usage.input_tokens + 
    (usage.cache_creation_input_tokens ?? 0) + 
    (usage.cache_read_input_tokens ?? 0)
  // 输出 token 累加
  tracker.cumulativeOutputTokens += usage.output_tokens
  
  for (const content of message.message.content) {
    if (content.type === 'tool_use') {
      tracker.toolUseCount++
      if (content.name !== SYNTHETIC_OUTPUT_TOOL_NAME) {
        const classification = tools ? 
          getToolSearchOrReadInfo(content.name, content.input, tools) : 
          undefined
        tracker.recentActivities.push({
          toolName: content.name,
          input: content.input as Record<string, unknown>,
          activityDescription: resolveActivityDescription?.(content.name, content.input),
          isSearch: classification?.isSearch,
          isRead: classification?.isRead
        })
      }
    }
  }
  // 保持最近 5 个活动
  while (tracker.recentActivities.length > MAX_RECENT_ACTIVITIES) {
    tracker.recentActivities.shift()
  }
}
```

#### 4.3.2 前台→后台转换信号机制

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:519-577
const backgroundSignalResolvers = new Map<string, () => void>()

export function registerAgentForeground({...}): {
  taskId: string
  backgroundSignal: Promise<void>
  cancelAutoBackground?: () => void
} {
  // ... 创建任务状态 ...
  
  // 创建后台信号 Promise
  let resolveBackgroundSignal: () => void
  const backgroundSignal = new Promise<void>(resolve => {
    resolveBackgroundSignal = resolve
  })
  backgroundSignalResolvers.set(agentId, resolveBackgroundSignal!)
  
  // 自动后台化定时器
  let cancelAutoBackground: (() => void) | undefined
  if (autoBackgroundMs !== undefined && autoBackgroundMs > 0) {
    const timer = setTimeout((setAppState, agentId) => {
      // 标记为后台化并解析信号
      setAppState(prev => {
        const prevTask = prev.tasks[agentId]
        if (!isLocalAgentTask(prevTask) || prevTask.isBackgrounded) return prev
        return {
          ...prev,
          tasks: {
            ...prev.tasks,
            [agentId]: { ...prevTask, isBackgrounded: true }
          }
        }
      })
      const resolver = backgroundSignalResolvers.get(agentId)
      if (resolver) {
        resolver()
        backgroundSignalResolvers.delete(agentId)
      }
    }, autoBackgroundMs, setAppState, agentId)
    cancelAutoBackground = () => clearTimeout(timer)
  }
  
  return { taskId: agentId, backgroundSignal, cancelAutoBackground }
}
```

---

## 5. 依赖与外部交互

### 5.1 直接依赖模块

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `../../bootstrap/state.js` | `getSdkAgentProgressSummariesEnabled` | SDK 进度总结开关 |
| `../../constants/xml.js` | XML 标签常量 | 构建 task_notification XML |
| `../../services/PromptSuggestion/speculation.js` | `abortSpeculation` | 后台任务变化时中止推测 |
| `../../state/AppState.js` | `AppState` 类型 | 状态类型定义 |
| `../../Task.js` | `SetAppState`, `Task`, `TaskStateBase`, `createTaskStateBase` | 任务基础设施 |
| `../../Tool.js` | `Tools`, `findToolByName` | 工具查找和活动描述 |
| `../../tools/AgentTool/agentToolUtils.js` | `AgentToolResult` 类型 | 结果类型 |
| `../../tools/AgentTool/loadAgentsDir.js` | `AgentDefinition` 类型 | Agent 定义类型 |
| `../../tools/SyntheticOutputTool/SyntheticOutputTool.js` | `SYNTHETIC_OUTPUT_TOOL_NAME` | 排除内部工具 |
| `../../types/ids.js` | `asAgentId` | ID 转换 |
| `../../types/message.js` | `Message` 类型 | 消息类型 |
| `../../utils/abortController.js` | `createAbortController`, `createChildAbortController` | 取消控制 |
| `../../utils/cleanupRegistry.js` | `registerCleanup` | 清理注册 |
| `../../utils/collapseReadSearch.js` | `getToolSearchOrReadInfo` | 工具活动分类 |
| `../../utils/messageQueueManager.js` | `enqueuePendingNotification` | 通知入队 |
| `../../utils/sessionStorage.js` | `getAgentTranscriptPath` | Agent 对话记录路径 |
| `../../utils/task/diskOutput.js` | `evictTaskOutput`, `getTaskOutputPath`, `initTaskOutputAsSymlink` | 输出文件管理 |
| `../../utils/task/framework.js` | `PANEL_GRACE_MS`, `registerTask`, `updateTaskState` | 任务框架 |
| `../../utils/task/sdkProgress.js` | `emitTaskProgress` | SDK 进度事件 |
| `../types.js` | `TaskState` 类型 | 任务状态联合类型 |

### 5.2 循环依赖处理

**问题**：`teammateViewHelpers.ts` 需要判断 `isLocalAgentTask`，但直接导入会导致循环依赖（通过 `BackgroundTasksDialog`）。

**解决方案**：在 `teammateViewHelpers.ts` 中内联类型检查函数：

```typescript
// src/state/teammateViewHelpers.ts:14-21
function isLocalAgent(task: unknown): task is LocalAgentTaskState {
  return (
    typeof task === 'object' &&
    task !== null &&
    'type' in task &&
    task.type === 'local_agent'
  )
}
```

同时内联 `PANEL_GRACE_MS` 常量以保持同步。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 内存泄漏风险

**风险点**：`backgroundSignalResolvers` Map 可能积累未清理的 resolver

**缓解措施**：
- `backgroundAgentTask` 成功后台化后会 `delete` resolver
- `unregisterAgentForeground` 清理 resolver

**潜在问题**：如果前台 Agent 完成前未被后台化且未被注销，resolver 可能残留。

#### 6.1.2 状态竞争

**风险点**：`enqueueAgentNotification` 的原子性检查依赖 `updateTaskState` 的同步执行

**代码**：
```typescript
let shouldEnqueue = false
updateTaskState<LocalAgentTaskState>(taskId, setAppState, task => {
  if (task.notified) return task
  shouldEnqueue = true
  return { ...task, notified: true }
})
if (!shouldEnqueue) return
```

**分析**：虽然 `setAppState` 的 updater 函数是同步执行的，但 React 的状态更新是批量的。在极端并发情况下，可能存在竞态条件。

#### 6.1.3 消息队列溢出

**风险点**：`pendingMessages` 数组无上限，如果 Agent 长时间不取消息，可能无限增长

**缓解**：实际场景下 SendMessage 调用频率有限，且 Agent 通常会在工具轮次边界处理。

### 6.2 边界条件

| 边界条件 | 行为 |
|----------|------|
| Agent 在 retain 状态下完成 | `evictAfter` 不设置，等待用户退出视图后设置 |
| Agent 被 kill 时正在工具调用中 | `abortController.abort()` 触发，Agent 工具框架处理中断 |
| 重复后台化调用 | `backgroundAgentTask` 返回 `false`，无操作 |
| 完成通知时 `notified` 已设置 | 跳过通知，防止重复 |
| 进度更新时任务已非 running | `updateAgentProgress` 返回原任务，无更新 |

### 6.3 改进建议

#### 6.3.1 添加 pendingMessages 上限

```typescript
const MAX_PENDING_MESSAGES = 100

export function queuePendingMessage(...): void {
  updateTaskState<LocalAgentTaskState>(taskId, setAppState, task => {
    if (task.pendingMessages.length >= MAX_PENDING_MESSAGES) {
      // 丢弃最旧的消息或报错
      logWarning('Pending messages overflow for agent', taskId)
      return task
    }
    return { ...task, pendingMessages: [...task.pendingMessages, msg] }
  })
}
```

#### 6.3.2 统一状态更新接口

当前状态更新分散在多个函数中，建议提供统一的事务性更新接口：

```typescript
interface TaskStateTransition {
  from: TaskStatus
  to: TaskStatus
  onEnter?: (task: LocalAgentTaskState) => Partial<LocalAgentTaskState>
}
```

#### 6.3.3 增强可观测性

添加更多调试日志和指标：
- 任务状态转换追踪
- pendingMessages 队列深度监控
- retain/diskLoaded 状态变化追踪

#### 6.3.4 清理残留 Resolver

在 `registerAgentForeground` 中添加定时清理机制：

```typescript
// 在 registerAgentForeground 中
const cleanupTimer = setTimeout(() => {
  if (backgroundSignalResolvers.has(agentId)) {
    logForDebugging(`Cleaning up stale background resolver for ${agentId}`)
    backgroundSignalResolvers.delete(agentId)
  }
}, MAX_FOREGROUND_DURATION_MS)

// 在返回的 cancelAutoBackground 中合并清理
cancelAutoBackground = () => {
  clearTimeout(timer)
  clearTimeout(cleanupTimer)
}
```

#### 6.3.5 类型安全增强

`isPanelAgentTask` 的注释说明它是"所有 pill/panel 过滤器必须同意的单一谓词"，建议：
1. 将其提取到共享的 type-guard 模块
2. 添加单元测试确保所有过滤器行为一致
3. 考虑使用 branded types 增强类型安全

---

## 7. 附录

### 7.1 相关测试文件

| 文件 | 测试内容 |
|------|----------|
| `src/tasks/__tests__/LocalAgentTask.test.tsx` | LocalAgentTask 单元测试（如有） |
| `src/tools/AgentTool/__tests__/AgentTool.test.tsx` | AgentTool 集成测试 |

### 7.2 配置项

| 配置 | 默认值 | 说明 |
|------|--------|------|
| `PANEL_GRACE_MS` | 30,000ms | 终端任务在面板中保留时间 |
| `MAX_RECENT_ACTIVITIES` | 5 | 最近活动列表长度 |
| `PROGRESS_THRESHOLD_MS` | 2,000ms | 显示后台提示的阈值 |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | - | 启用自动后台化 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | - | 禁用后台任务 |

### 7.3 版本历史

- **初始实现**：替换 `src/tools/AgentTool/asyncAgentUtils.ts` 中的 AsyncAgent
- **近期变更**：添加了 `retain`/`diskLoaded`/`evictAfter` 支持 Coordinator 面板
