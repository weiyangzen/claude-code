# framework.ts 深度研究文档

## 场景与职责

`framework.ts` 是 Claude Code **任务管理框架的核心协调器**。它负责任务状态管理、任务生命周期事件处理、输出增量追踪和任务通知生成。该模块作为任务系统与 React 应用状态 (AppState) 之间的桥梁，确保任务状态的一致性和实时性。

### 核心职责

1. **任务状态管理**: 注册、更新、驱逐任务状态
2. **输出增量追踪**: 轮询运行中任务的新输出
3. **任务通知生成**: 创建任务状态变更通知
4. **任务生命周期**: 管理任务从 pending → running → completed/failed/killed 的完整流程
5. **SDK 事件集成**: 与 SDK 事件队列集成，支持 headless/streaming 模式

### 使用场景

| 场景 | 调用方 | 功能 |
|------|--------|------|
| 任务注册 | `LocalAgentTask.tsx`, `LocalShellTask.tsx` | `registerTask()` |
| 任务状态更新 | 各任务实现 | `updateTaskState()` |
| 任务轮询 | `App.tsx` useEffect | `pollTasks()` |
| 任务驱逐 | 任务完成/杀死 | `evictTerminalTask()` |
| 通知生成 | `print.ts` | `generateTaskAttachments()` |

---

## 功能点目的

### 1. 任务状态机

```
pending → running → completed
   ↓         ↓
        ┌──→ failed
        └──→ killed
```

**状态定义** (`Task.ts`):
```typescript
type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

### 2. 任务附件 (TaskAttachment)

```typescript
export type TaskAttachment = {
  type: 'task_status'
  taskId: string
  toolUseId?: string
  taskType: TaskType
  status: TaskStatus
  description: string
  deltaSummary: string | null  // 自上次附件以来的新输出
}
```

### 3. 轮询机制

- **轮询间隔**: 1000ms (`POLL_INTERVAL_MS`)
- **停止显示时间**: 3000ms (`STOPPED_DISPLAY_MS`)
- **面板宽限期**: 30000ms (`PANEL_GRACE_MS`) - 用于 local_agent 任务

### 4. 输出偏移量追踪

```typescript
// TaskStateBase (来自 Task.ts)
type TaskStateBase = {
  id: string
  outputFile: string
  outputOffset: number  // 上次读取的字节位置
  notified: boolean     // 是否已发送通知
  // ...
}
```

---

## 具体技术实现

### 关键数据结构

```typescript
// 任务附件类型
type TaskAttachment = {
  type: 'task_status'
  taskId: string
  toolUseId?: string
  taskType: TaskType
  status: TaskStatus
  description: string
  deltaSummary: string | null
}

// AppState 更新函数类型
type SetAppState = (updater: (prev: AppState) => AppState) => void
```

### 核心流程

#### 1. 任务注册 (`registerTask`)

```typescript
export function registerTask(task: TaskState, setAppState: SetAppState): void {
  let isReplacement = false
  setAppState(prev => {
    const existing = prev.tasks[task.id]
    isReplacement = existing !== undefined
    
    // 合并状态：恢复 UI 持有的状态（如 retain、messages）
    const merged = existing && 'retain' in existing
      ? { ...task, retain: existing.retain, startTime: existing.startTime, ... }
      : task
    
    return { ...prev, tasks: { ...prev.tasks, [task.id]: merged } }
  })
  
  // 替换任务（resume）不发送 task_started 事件
  if (isReplacement) return
  
  // 发送 SDK task_started 事件
  enqueueSdkEvent({
    type: 'system',
    subtype: 'task_started',
    task_id: task.id,
    // ...
  })
}
```

**关键设计**: 任务替换（如 `resumeAgentBackground`）时保留 UI 状态，避免用户看到的面板状态重置。

#### 2. 任务状态更新 (`updateTaskState`)

```typescript
export function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T,
): void {
  setAppState(prev => {
    const task = prev.tasks?.[taskId] as T | undefined
    if (!task) return prev
    
    const updated = updater(task)
    if (updated === task) {
      // 引用相同，跳过更新（避免不必要的重渲染）
      return prev
    }
    
    return {
      ...prev,
      tasks: { ...prev.tasks, [taskId]: updated }
    }
  })
}
```

**性能优化**: 比较引用相等性，避免不必要的 React 重渲染。

#### 3. 生成任务附件 (`generateTaskAttachments`)

```typescript
export async function generateTaskAttachments(state: AppState): Promise<{
  attachments: TaskAttachment[]
  updatedTaskOffsets: Record<string, number>
  evictedTaskIds: string[]
}> {
  const attachments: TaskAttachment[] = []
  const updatedTaskOffsets: Record<string, number> = {}
  const evictedTaskIds: string[] = []
  
  for (const taskState of Object.values(tasks)) {
    if (taskState.notified) {
      switch (taskState.status) {
        case 'completed':
        case 'failed':
        case 'killed':
          // 终端状态任务可以被驱逐
          evictedTaskIds.push(taskState.id)
          continue
        case 'pending':
          continue
        case 'running':
          break
      }
    }
    
    if (taskState.status === 'running') {
      // 增量读取新输出
      const delta = await getTaskOutputDelta(
        taskState.id,
        taskState.outputOffset,
      )
      if (delta.content) {
        updatedTaskOffsets[taskState.id] = delta.newOffset
      }
    }
  }
  
  return { attachments, updatedTaskOffsets, evictedTaskIds }
}
```

**重要注释**: 完成任务的附件**不在这里生成**，由各任务类型自行处理，避免与 `enqueuePendingNotification()` 竞态导致重复通知。

#### 4. 应用偏移量和驱逐 (`applyTaskOffsetsAndEvictions`)

```typescript
export function applyTaskOffsetsAndEvictions(
  setAppState: SetAppState,
  updatedTaskOffsets: Record<string, number>,
  evictedTaskIds: string[],
): void {
  setAppState(prev => {
    const newTasks = { ...prev.tasks }
    
    for (const id of offsetIds) {
      const fresh = newTasks[id]
      // 重新检查状态：任务可能在 await 期间完成
      if (fresh?.status === 'running') {
        newTasks[id] = { ...fresh, outputOffset: updatedTaskOffsets[id]! }
      }
    }
    
    for (const id of evictedTaskIds) {
      const fresh = newTasks[id]
      // 重新检查终端状态 + notified（TOCTOU 防护）
      if (!fresh || !isTerminalTaskStatus(fresh.status) || !fresh.notified) {
        continue
      }
      // 面板宽限期检查（仅 local_agent）
      if ('retain' in fresh && (fresh.evictAfter ?? Infinity) > Date.now()) {
        continue
      }
      delete newTasks[id]
    }
    
    return changed ? { ...prev, tasks: newTasks } : prev
  })
}
```

**TOCTOU 防护**: 在 `generateTaskAttachments` 的 `await` 期间，任务状态可能已变更。使用 fresh state 重新验证避免竞态条件。

#### 5. 任务轮询 (`pollTasks`)

```typescript
export async function pollTasks(
  getAppState: () => AppState,
  setAppState: SetAppState,
): Promise<void> {
  const state = getAppState()
  const { attachments, updatedTaskOffsets, evictedTaskIds } =
    await generateTaskAttachments(state)
  
  applyTaskOffsetsAndEvictions(setAppState, updatedTaskOffsets, evictedTaskIds)
  
  // 发送任务通知
  for (const attachment of attachments) {
    enqueueTaskNotification(attachment)
  }
}
```

#### 6. 任务通知入队 (`enqueueTaskNotification`)

```typescript
function enqueueTaskNotification(attachment: TaskAttachment): void {
  const statusText = getStatusText(attachment.status)
  const outputPath = getTaskOutputPath(attachment.taskId)
  
  const message = `<${TASK_NOTIFICATION_TAG}>
<${TASK_ID_TAG}>${attachment.taskId}</${TASK_ID_TAG}>
<${TASK_TYPE_TAG}>${attachment.taskType}</${TASK_TYPE_TAG}>
<${OUTPUT_FILE_TAG}>${outputPath}</${OUTPUT_FILE_TAG}>
<${STATUS_TAG}>${attachment.status}</${STATUS_TAG}>
<${SUMMARY_TAG}>Task "${attachment.description}" ${statusText}</${SUMMARY_TAG}>
</${TASK_NOTIFICATION_TAG}>`
  
  enqueuePendingNotification({ value: message, mode: 'task-notification' })
}
```

**XML 格式**: 任务通知使用 XML 格式，由 `print.ts` 解析并转换为 SDK 事件。

#### 7. 终端任务驱逐 (`evictTerminalTask`)

```typescript
export function evictTerminalTask(
  taskId: string,
  setAppState: SetAppState,
): void {
  setAppState(prev => {
    const task = prev.tasks?.[taskId]
    if (!task) return prev
    if (!isTerminalTaskStatus(task.status)) return prev
    if (!task.notified) return prev
    
    // 面板宽限期（仅 local_agent）
    if ('retain' in task && (task.evictAfter ?? Infinity) > Date.now()) {
      return prev
    }
    
    const { [taskId]: _, ...remainingTasks } = prev.tasks
    return { ...prev, tasks: remainingTasks }
  })
}
```

---

## 关键代码路径与文件引用

### 核心调用链

```
App.tsx (useEffect)
  └── pollTasks(getAppState, setAppState)
      ├── generateTaskAttachments(state)
      │   ├── getRunningTasks(state)
      │   └── getTaskOutputDelta(taskId, offset)  [diskOutput.ts]
      ├── applyTaskOffsetsAndEvictions(setAppState, offsets, evictions)
      └── for each attachment: enqueueTaskNotification(attachment)
          └── enqueuePendingNotification({ value: xml, mode: 'task-notification' })
              [messageQueueManager.ts]

LocalAgentTask.tsx / LocalShellTask.tsx
  └── registerTask(taskState, setAppState)
      └── enqueueSdkEvent({ type: 'system', subtype: 'task_started', ... })

任务状态更新
  └── updateTaskState(taskId, setAppState, updater)
      └── setAppState(prev => { ... })
```

### 外部调用方

| 文件 | 调用函数 | 用途 |
|------|----------|------|
| `src/App.tsx` | `pollTasks` | 主轮询循环 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `registerTask`, `updateTaskState` | 代理任务管理 |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | `registerTask`, `updateTaskState` | Shell 任务管理 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `registerTask` | 远程代理任务 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | `registerTask` | 进程内队友任务 |
| `src/tasks/DreamTask/DreamTask.ts` | `registerTask` | Dream 任务 |
| `src/components/CoordinatorAgentStatus.tsx` | `getRunningTasks` | 获取运行中任务 |
| `src/tools/AgentTool/resumeAgent.ts` | `evictTerminalTask` | 驱逐终端代理 |
| `src/cli/print.ts` | `generateTaskAttachments` | 生成任务附件 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/Task.ts` | `TaskStatus`, `TaskType`, `isTerminalTaskStatus`, `generateTaskId` |
| `src/state/AppState.ts` | `AppState` 类型 |
| `src/tasks/types.ts` | `TaskState` 类型 |
| `src/utils/task/diskOutput.ts` | `getTaskOutputDelta`, `getTaskOutputPath` |
| `src/utils/sdkEventQueue.ts` | `enqueueSdkEvent` |
| `src/utils/messageQueueManager.ts` | `enqueuePendingNotification` |
| `src/constants/xml.ts` | XML 标签常量 |

---

## 依赖与外部交互

### 模块依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                        framework.ts                             │
├─────────────────────────────────────────────────────────────────┤
│  Task State Management                                          │
│  ├── registerTask() → enqueueSdkEvent()                        │
│  ├── updateTaskState() → setAppState()                         │
│  └── evictTerminalTask() → setAppState()                       │
├─────────────────────────────────────────────────────────────────┤
│  Polling & Notifications                                        │
│  ├── pollTasks()                                                │
│  │   ├── generateTaskAttachments()                              │
│  │   │   └── getTaskOutputDelta()  [diskOutput.ts]             │
│  │   ├── applyTaskOffsetsAndEvictions()                         │
│  │   └── enqueueTaskNotification()                              │
│  │       └── enqueuePendingNotification() [messageQueueManager]│
│  └── getRunningTasks()                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 与 AppState 的交互

```typescript
// AppState 中的任务状态
type AppState = {
  tasks?: Record<string, TaskState>
  // ...
}

// TaskState 结构 (来自 tasks/types.ts)
type TaskState = 
  | LocalAgentTaskState
  | LocalShellTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | DreamTaskState
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 竞态条件 (已缓解)

**问题**: `generateTaskAttachments` 中的 `await getTaskOutputDelta` 期间，任务状态可能变更。

**缓解**: `applyTaskOffsetsAndEvictions` 使用 fresh state 重新验证:
```typescript
// 重新检查状态
if (fresh?.status === 'running') {
  newTasks[id] = { ...fresh, outputOffset: updatedTaskOffsets[id]! }
}
```

#### 2. 重复通知风险 (已缓解)

**问题**: 如果 `generateTaskAttachments` 和任务完成回调都发送通知，会导致重复。

**缓解**: 完成任务的附件**不在这里生成**，由各任务类型自行处理:
```typescript
// Completed tasks are NOT notified here
// each task type handles its own completion notification
```

#### 3. 内存泄漏 (已缓解)

**问题**: 长期运行的会话可能积累大量已完成任务。

**缓解**: 
- 终端任务自动驱逐机制
- `evictedTaskIds` 从 AppState 中删除
- `notified` 标记确保通知已发送后才驱逐

### 边界情况

| 场景 | 行为 |
|------|------|
| 任务在 `await` 期间完成 | 使用 fresh state 重新验证，跳过偏移量更新 |
| 任务在 `await` 期间被替换 | `evictTerminalTask` 检查 fresh state |
| 面板宽限期内 | `local_agent` 任务在 `evictAfter` 前不被驱逐 |
| 重复注册同一任务 | `registerTask` 合并现有状态，不重复发送 `task_started` |
| 空输出增量 | `delta.content` 为空时，不更新 `outputOffset` |

### 改进建议

#### 1. 指数退避轮询

当前固定 1000ms 轮询，对于空闲任务可优化:
```typescript
const POLL_INTERVAL_MS = 1000
const IDLE_POLL_INTERVAL_MS = 5000

function getPollInterval(tasks: TaskState[]): number {
  const hasRecentOutput = tasks.some(t => 
    Date.now() - t.lastOutputTime < 30000
  )
  return hasRecentOutput ? POLL_INTERVAL_MS : IDLE_POLL_INTERVAL_MS
}
```

#### 2. 批量偏移量更新

当前每个任务单独调用 `getTaskOutputDelta`，可优化为批量:
```typescript
// 使用 Promise.all 并行读取
const deltas = await Promise.all(
  runningTasks.map(t => 
    getTaskOutputDelta(t.id, t.outputOffset)
  )
)
```

#### 3. 智能驱逐策略

当前基于时间和状态驱逐，可考虑内存压力:
```typescript
function shouldEvict(task: TaskState): boolean {
  if (!isTerminalTaskStatus(task.status)) return false
  
  // 内存压力下更激进地驱逐
  if (process.memoryUsage().heapUsed > MEMORY_THRESHOLD) {
    return task.notified
  }
  
  return task.notified && (task.evictAfter ?? Infinity) <= Date.now()
}
```

#### 4. 任务输出缓存

对于频繁读取的任务输出，考虑缓存:
```typescript
const outputCache = new Map<string, { content: string; offset: number }>()

async function getCachedTaskOutputDelta(
  taskId: string,
  fromOffset: number,
): Promise<{ content: string; newOffset: number }> {
  const cached = outputCache.get(taskId)
  if (cached && cached.offset === fromOffset) {
    return { content: '', newOffset: fromOffset }  // 无新内容
  }
  // ...
}
```

#### 5. 可观测性增强

添加 OpenTelemetry 指标:
```typescript
// 任务状态转换计数
// 轮询延迟分布
// 驱逐任务数量
// 平均输出增量大小
```

### 测试覆盖建议

当前未发现专门的 `framework.test.ts`，建议添加:

1. **单元测试**:
   - `updateTaskState` 引用相等性优化
   - `applyTaskOffsetsAndEvictions` TOCTOU 防护
   - `evictTerminalTask` 宽限期逻辑

2. **集成测试**:
   - 完整轮询流程
   - 任务状态机转换
   - 与 `diskOutput.ts` 的集成

3. **竞态测试**:
   - 任务在 `await` 期间完成
   - 任务在 `await` 期间被替换
   - 并发状态更新
