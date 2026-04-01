# sdkProgress.ts 深度研究文档

## 场景与职责

`sdkProgress.ts` 是 Claude Code 中 **SDK 任务进度事件的发射器**。它提供了一个统一的接口，用于向 SDK 消费者（如 Scuttle、VS Code 扩展、headless 模式客户端）报告任务进度信息。

### 核心职责

1. **进度事件标准化**: 统一封装任务进度数据为 SDK 事件格式
2. **后台代理进度**: 支持后台代理（background agents）的进度报告
3. **工作流进度**: 支持工作流（workflows）的批量进度刷新
4. **使用统计**: 报告 token 使用量、工具调用次数、执行时长等指标

### 使用场景

| 场景 | 调用方 | 功能 |
|------|--------|------|
| 后台代理生命周期 | `runAsyncAgentLifecycle` | 每 tool_use 报告进度 |
| 工作流进度刷新 | `flushProgress` | 批量报告工作流状态变化 |
| 任务进度追踪 | 各任务实现 | 向 SDK 消费者报告进度 |

---

## 功能点目的

### 1. SDK 进度事件标准化

统一封装为 `task_progress` 类型的 SDK 事件:

```typescript
type TaskProgressEvent = {
  type: 'system'
  subtype: 'task_progress'
  task_id: string
  tool_use_id?: string
  description: string
  usage: {
    total_tokens: number
    tool_uses: number
    duration_ms: number
  }
  last_tool_name?: string
  summary?: string
  workflow_progress?: SdkWorkflowProgress[]
}
```

### 2. 解耦调用方与事件队列

`emitTaskProgress` 接受原始参数，内部处理为 SDK 事件并入队:

```typescript
export function emitTaskProgress(params: {
  taskId: string
  toolUseId: string | undefined
  description: string
  startTime: number
  totalTokens: number
  toolUses: number
  lastToolName?: string
  summary?: string
  workflowProgress?: SdkWorkflowProgress[]
}): void
```

### 3. 工作流进度支持

支持批量报告工作流状态变化，用于复杂的分阶段工作流:

```typescript
type SdkWorkflowProgress = {
  type: string
  index: number
  phaseIndex: number
  // ... 其他字段
}
```

---

## 具体技术实现

### 完整代码

```typescript
import type { SdkWorkflowProgress } from '../../types/tools.js'
import { enqueueSdkEvent } from '../sdkEventQueue.js'

/**
 * Emit a `task_progress` SDK event. Shared by background agents (per tool_use
 * in runAsyncAgentLifecycle) and workflows (per flushProgress batch). Accepts
 * already-computed primitives so callers can derive them from their own state
 * shapes (ProgressTracker for agents, LocalWorkflowTaskState for workflows).
 */
export function emitTaskProgress(params: {
  taskId: string
  toolUseId: string | undefined
  description: string
  startTime: number
  totalTokens: number
  toolUses: number
  lastToolName?: string
  summary?: string
  workflowProgress?: SdkWorkflowProgress[]
}): void {
  enqueueSdkEvent({
    type: 'system',
    subtype: 'task_progress',
    task_id: params.taskId,
    tool_use_id: params.toolUseId,
    description: params.description,
    usage: {
      total_tokens: params.totalTokens,
      tool_uses: params.toolUses,
      duration_ms: Date.now() - params.startTime,
    },
    last_tool_name: params.lastToolName,
    summary: params.summary,
    workflow_progress: params.workflowProgress,
  })
}
```

### 关键设计决策

#### 1. 接受原始值而非对象

```typescript
// 当前设计：接受原始值
emitTaskProgress({
  taskId: string
  startTime: number
  totalTokens: number
  // ...
})

// 替代设计（未采用）：接受状态对象
emitTaskProgress(taskState: LocalWorkflowTaskState)
```

**理由**: 注释中说明 "Accepts already-computed primitives so callers can derive them from their own state shapes"
- 允许调用方从不同状态形状派生值
- `ProgressTracker` (agents) 和 `LocalWorkflowTaskState` (workflows) 可以使用同一接口
- 减少模块间的类型耦合

#### 2. 时长计算

```typescript
duration_ms: Date.now() - params.startTime
```

- 调用方提供 `startTime` (时间戳)
- 函数内部计算持续时间
- 避免调用方重复计算

#### 3. 可选字段

| 字段 | 可选 | 使用场景 |
|------|------|----------|
| `toolUseId` | 是 | 关联具体 tool_use |
| `lastToolName` | 是 | 显示最后使用的工具 |
| `summary` | 是 | 进度摘要描述 |
| `workflowProgress` | 是 | 工作流阶段进度 |

---

## 关键代码路径与文件引用

### 调用链

```
后台代理生命周期 / 工作流刷新
  └── emitTaskProgress(params)
      └── enqueueSdkEvent({
            type: 'system',
            subtype: 'task_progress',
            ...
          })
              └── sdkEventQueue.ts
                  └── queue.push(event)  (仅 headless/streaming 模式)
```

### 外部调用方

通过 Grep 搜索，以下文件可能调用 `emitTaskProgress`:

| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | 后台代理进度报告 |
| `src/tasks/LocalWorkflowTask/*.tsx` | 工作流进度刷新 |
| `src/utils/swarm/*.ts` | 多代理协调进度 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/types/tools.ts` | `SdkWorkflowProgress` 类型 |
| `src/utils/sdkEventQueue.ts` | `enqueueSdkEvent` 函数 |

---

## 依赖与外部交互

### 模块依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                       sdkProgress.ts                            │
├─────────────────────────────────────────────────────────────────┤
│  emitTaskProgress(params)                                       │
│  └── enqueueSdkEvent(TaskProgressEvent)                         │
│      └── sdkEventQueue.ts                                       │
│          └── queue: SdkEvent[]                                  │
│          └── MAX_QUEUE_SIZE = 1000                              │
└─────────────────────────────────────────────────────────────────┘
```

### 与 sdkEventQueue 的交互

```typescript
// sdkEventQueue.ts
const MAX_QUEUE_SIZE = 1000
const queue: SdkEvent[] = []

export function enqueueSdkEvent(event: SdkEvent): void {
  // 仅在非交互式会话中入队
  if (!getIsNonInteractiveSession()) {
    return
  }
  if (queue.length >= MAX_QUEUE_SIZE) {
    queue.shift()  // 队列满时丢弃最旧事件
  }
  queue.push(event)
}
```

**重要限制**: SDK 事件仅在 **headless/streaming 模式** 下入队。在 TUI 模式下，事件会累积到上限然后被丢弃。

---

## 风险、边界与改进建议

### 已知风险

#### 1. TUI 模式事件丢失

**问题**: 在 TUI 模式下，`enqueueSdkEvent` 直接返回，事件不入队。

**设计意图**: SDK 事件消费者只在 headless/streaming 模式下存在。

**风险**: 如果未来 TUI 模式也需要进度事件，需要重构。

### 边界情况

| 场景 | 行为 |
|------|------|
| `startTime` 是未来时间 | `duration_ms` 为负数 |
| `startTime` 是 0 | `duration_ms` 等于当前时间戳（约 50 年） |
| 队列为满 | 最旧事件被丢弃 |
| TUI 模式 | 事件不入队，函数无操作返回 |
| `workflowProgress` 为空数组 | 字段仍包含在事件中（空数组） |

### 改进建议

#### 1. 时长计算验证

添加对负时长的防护:
```typescript
const durationMs = Math.max(0, Date.now() - params.startTime)
```

#### 2. 事件队列满警告

当队列满时记录警告:
```typescript
export function enqueueSdkEvent(event: SdkEvent): void {
  if (!getIsNonInteractiveSession()) return
  
  if (queue.length >= MAX_QUEUE_SIZE) {
    const dropped = queue.shift()
    logForDebugging(`SDK event queue full, dropped: ${dropped?.type}`)
  }
  queue.push(event)
}
```

#### 3. 批量进度事件

对于高频进度更新，考虑批量发送:
```typescript
let pendingProgress: TaskProgressEvent[] = []
let flushTimeout: NodeJS.Timeout | null = null

export function emitTaskProgressBuffered(params: {...}): void {
  pendingProgress.push(createEvent(params))
  
  if (!flushTimeout) {
    flushTimeout = setTimeout(() => {
      enqueueSdkEvent({
        type: 'system',
        subtype: 'task_progress_batch',
        events: pendingProgress,
      })
      pendingProgress = []
      flushTimeout = null
    }, 100)
  }
}
```

#### 4. 进度事件节流

防止过于频繁的进度更新:
```typescript
const lastProgressTime = new Map<string, number>()
const MIN_PROGRESS_INTERVAL_MS = 500

export function emitTaskProgressThrottled(params: {...}): void {
  const lastTime = lastProgressTime.get(params.taskId) ?? 0
  const now = Date.now()
  
  if (now - lastTime < MIN_PROGRESS_INTERVAL_MS) {
    return  // 跳过，过于频繁
  }
  
  lastProgressTime.set(params.taskId, now)
  emitTaskProgress(params)
}
```

#### 5. 类型安全增强

当前 `SdkWorkflowProgress` 来自 `types/tools.ts`，可考虑更精确的类型:
```typescript
// 当前
workflow_progress?: SdkWorkflowProgress[]

// 改进：使用 branded type 确保类型安全
type WorkflowProgress = SdkWorkflowProgress & { __brand: 'workflow' }
```

### 测试覆盖建议

当前未发现专门的测试文件，建议添加:

1. **单元测试**:
   - 事件格式正确性
   - 时长计算边界
   - 可选字段省略

2. **集成测试**:
   - 与 `sdkEventQueue.ts` 的集成
   - 队列满时的行为
   - TUI vs headless 模式差异

3. **性能测试**:
   - 高频调用性能
   - 内存使用模式
