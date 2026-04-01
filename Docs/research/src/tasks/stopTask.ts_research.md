# stopTask.ts 研究文档

## 场景与职责

stopTask.ts 是 Claude Code CLI 中处理**停止运行中任务**的共享逻辑模块。它提供了统一的任务停止接口，被以下两个入口使用：

1. **TaskStopTool**: LLM 调用的工具，用于停止指定的后台任务
2. **SDK stop_task 控制请求**: 外部 SDK 客户端发送的停止任务请求

### 核心使用场景

1. **用户通过 LLM 停止任务**: LLM 调用 `TaskStopTool` 停止指定 ID 的任务
2. **SDK 客户端停止任务**: 外部应用通过 SDK 发送控制请求停止任务
3. **批量停止任务**: 配合其他模块实现停止所有后台任务

---

## 功能点目的

### 1. 停止任务 (`stopTask`)

核心函数，执行停止任务的完整流程：

- **任务查找**: 通过任务 ID 在 AppState 中查找任务
- **状态验证**: 确保任务存在且处于运行状态
- **类型验证**: 确保任务类型支持停止操作
- **执行停止**: 调用任务类型的 `kill` 方法
- **通知处理**: 对 Shell 任务抑制退出码 137 的通知（避免噪音）
- **SDK 事件**: 确保 SDK 消费者能收到任务关闭事件

### 2. 错误处理 (`StopTaskError`)

自定义错误类，提供结构化的错误信息：

- `not_found`: 任务不存在
- `not_running`: 任务未在运行
- `unsupported_type`: 不支持的任务类型

---

## 具体技术实现

### 关键数据结构

```typescript
// 停止任务错误
export class StopTaskError extends Error {
  constructor(
    message: string,
    public readonly code: 'not_found' | 'not_running' | 'unsupported_type',
  ) {
    super(message)
    this.name = 'StopTaskError'
  }
}

// 停止任务上下文
 type StopTaskContext = {
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
}

// 停止任务结果
type StopTaskResult = {
  taskId: string
  taskType: string
  command: string | undefined  // Shell 任务返回 command，其他返回 description
}
```

### 核心算法

#### stopTask 函数流程

```typescript
export async function stopTask(
  taskId: string,
  context: StopTaskContext,
): Promise<StopTaskResult> {
  const { getAppState, setAppState } = context
  const appState = getAppState()
  const task = appState.tasks?.[taskId] as TaskStateBase | undefined

  // 1. 验证任务存在
  if (!task) {
    throw new StopTaskError(`No task found with ID: ${taskId}`, 'not_found')
  }

  // 2. 验证任务正在运行
  if (task.status !== 'running') {
    throw new StopTaskError(
      `Task ${taskId} is not running (status: ${task.status})`,
      'not_running',
    )
  }

  // 3. 获取任务实现
  const taskImpl = getTaskByType(task.type)
  if (!taskImpl) {
    throw new StopTaskError(
      `Unsupported task type: ${task.type}`,
      'unsupported_type',
    )
  }

  // 4. 执行停止
  await taskImpl.kill(taskId, setAppState)

  // 5. 处理 Shell 任务的通知抑制
  if (isLocalShellTask(task)) {
    let suppressed = false
    setAppState(prev => {
      const prevTask = prev.tasks[taskId]
      if (!prevTask || prevTask.notified) {
        return prev
      }
      suppressed = true
      return {
        ...prev,
        tasks: {
          ...prev.tasks,
          [taskId]: { ...prevTask, notified: true },
        },
      }
    })
    
    // 6. 发送 SDK 事件（如果抑制了通知）
    if (suppressed) {
      emitTaskTerminatedSdk(taskId, 'stopped', {
        toolUseId: task.toolUseId,
        summary: task.description,
      })
    }
  }

  // 7. 返回结果
  const command = isLocalShellTask(task) ? task.command : task.description
  return { taskId, taskType: task.type, command }
}
```

### 通知抑制逻辑

```typescript
// Bash: 抑制 "exit code 137" 通知（噪音）
// Agent 任务: 不抑制 — AbortError catch 会发送携带 extractPartialResult(agentMessages) 的通知
if (isLocalShellTask(task)) {
  let suppressed = false
  setAppState(prev => {
    const prevTask = prev.tasks[taskId]
    if (!prevTask || prevTask.notified) {
      return prev
    }
    suppressed = true
    return {
      ...prev,
      tasks: {
        ...prev.tasks,
        [taskId]: { ...prevTask, notified: true },
      },
    }
  })
  
  // 抑制 XML 通知也会抑制 print.ts 解析的 task_notification SDK 事件
  // 直接发送 SDK 事件确保 SDK 消费者能看到任务关闭
  if (suppressed) {
    emitTaskTerminatedSdk(taskId, 'stopped', {
      toolUseId: task.toolUseId,
      summary: task.description,
    })
  }
}
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件路径 | 用途 |
|---------|------|
| `src/tasks/stopTask.ts` | 本文件，停止任务的核心逻辑 |
| `src/tasks.ts` | `getTaskByType` 函数，获取任务类型实现 |
| `src/Task.ts` | `TaskStateBase` 类型定义 |
| `src/tasks/LocalShellTask/guards.ts` | `isLocalShellTask` 类型守卫 |

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/state/AppState.ts` | `AppState` 类型定义 |
| `src/utils/sdkEventQueue.ts` | `emitTaskTerminatedSdk` - SDK 事件发送 |

### 调用方文件

| 文件路径 | 用途 |
|---------|------|
| `src/tools/TaskStopTool/TaskStopTool.ts` | LLM 调用的停止任务工具 |
| `src/cli/print.ts` | CLI 打印处理，可能调用停止任务 |
| `src/tasks/LocalMainSessionTask.ts` | 主会话任务，引用 stopTask |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | Shell 任务实现 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 远程 agent 任务实现 |

---

## 依赖与外部交互

### 类型依赖

```typescript
// 应用状态
import type { AppState } from '../state/AppState.js'

// 任务基础类型
import type { TaskStateBase } from '../Task.js'

// 任务类型获取
import { getTaskByType } from '../tasks.js'

// Shell 任务类型守卫
import { isLocalShellTask } from './LocalShellTask/guards.js'

// SDK 事件
import { emitTaskTerminatedSdk } from '../utils/sdkEventQueue.js'
```

### 外部服务交互

1. **任务类型系统** (`tasks.ts`)
   - `getTaskByType`: 根据任务类型获取对应的 Task 实现
   - 返回的 Task 对象包含 `kill` 方法用于停止任务

2. **Shell 任务守卫** (`LocalShellTask/guards.ts`)
   - `isLocalShellTask`: 类型守卫，判断任务是否是本地 Shell 任务
   - 用于决定是否抑制通知

3. **SDK 事件队列** (`utils/sdkEventQueue.ts`)
   - `emitTaskTerminatedSdk`: 发送任务终止 SDK 事件
   - 确保 SDK 消费者能收到任务关闭通知

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**
   - 检查任务状态和实际停止之间可能存在时间窗口
   - 任务可能在检查后被其他代码停止

2. **通知抑制副作用**
   - 抑制 Shell 任务的 XML 通知也会抑制 print.ts 解析的 SDK 事件
   - 需要手动发送 `emitTaskTerminatedSdk` 补偿

3. **类型断言**
   - 使用 `as TaskStateBase | undefined` 类型断言
   - 如果 AppState 结构变化可能导致运行时错误

### 边界情况

1. **任务不存在**
   - 抛出 `StopTaskError` 错误，code 为 `'not_found'`
   - 调用方可以据此提供用户友好的错误提示

2. **任务未运行**
   - 抛出 `StopTaskError` 错误，code 为 `'not_running'`
   - 包含当前状态信息，便于调试

3. **不支持的任务类型**
   - 抛出 `StopTaskError` 错误，code 为 `'unsupported_type'`
   - 可能发生在新增任务类型但未注册到 `getTaskByType` 时

4. **已通知的任务**
   - 如果任务已经被标记为 `notified`，跳过后续通知处理
   - 避免重复发送 SDK 事件

5. **Shell vs Agent 任务差异**
   - Shell 任务抑制通知（避免 exit code 137 噪音）
   - Agent 任务不抑制，AbortError catch 会发送包含部分结果的通知

### 改进建议

1. **原子性增强**
   - 考虑将状态检查和停止操作合并为原子操作
   - 减少竞态条件窗口

2. **重试机制**
   - 对于 `kill` 调用失败的情况，考虑添加重试逻辑
   - 特别是网络相关的远程任务

3. **更详细的返回信息**
   - 当前返回 `command` 或 `description`
   - 可以考虑增加停止时间、任务运行时长等信息

4. **批量停止支持**
   - 当前是单任务停止接口
   - 可以考虑增加批量停止接口，优化性能

5. **测试覆盖**
   - 建议增加单元测试：
     - 各种错误情况（not_found, not_running, unsupported_type）
     - Shell 任务通知抑制
     - Agent 任务通知不抑制
     - 已通知任务的跳过逻辑
     - 并发停止同一任务

6. **日志记录**
   - 当前没有显式的日志记录
   - 可以考虑添加调试日志，便于排查问题

7. **超时处理**
   - `taskImpl.kill` 是异步操作
   - 考虑添加超时机制，防止 kill 操作挂起
