# useTeammateViewAutoExit.ts 深度研究文档

## 场景与职责

`useTeammateViewAutoExit` 是一个 React Hook，用于在查看队友（Teammate）时自动退出查看模式。当被查看的队友被终止、遇到错误或进入非活跃状态时，自动将用户带回主视图。

### 核心职责

1. **队友状态监控**: 监控被查看队友的任务状态
2. **自动退出**: 在特定条件下自动退出队友查看模式
3. **完成状态保留**: 允许用户继续查看已完成的队友以审阅完整对话

### 使用场景

- **队友终止**: 队友进程被终止时自动退出
- **错误发生**: 队友遇到错误时自动退出
- **状态变化**: 队友进入非运行状态时自动退出
- **任务删除**: 队友任务从任务列表中移除时自动退出

---

## 功能点目的

### 1. 队友终止检测

当被查看的队友被终止时：
- 立即退出查看模式
- 返回主视图

### 2. 错误检测

当队友任务出现错误时：
- 自动退出查看模式
- 允许用户查看错误信息

### 3. 非活跃状态检测

当队友进入非运行状态时（非 running/completed/pending）：
- 自动退出查看模式

### 4. 任务删除检测

当被查看的任务从任务列表中移除时：
- 自动退出查看模式

---

## 具体技术实现

### 关键数据结构

```typescript
// AppState 相关类型
interface AppState {
  viewingAgentTaskId?: string           // 当前正在查看的队友任务 ID
  tasks: Record<string, TaskState>     // 所有任务
}

interface TaskState {
  type: 'local_agent' | 'in_process_teammate' | ...
  status: 'running' | 'completed' | 'killed' | 'failed' | 'pending' | ...
  error?: Error | string
}

// 队友任务类型（narrowed）
interface InProcessTeammateTaskState extends TaskState {
  type: 'in_process_teammate'
  status: 'running' | 'completed' | 'killed' | 'failed' | 'pending'
}
```

### 核心流程

#### 1. Effect 执行流程
```
useEffect 触发
  ↓
检查 viewingAgentTaskId 是否存在
  ↓
不存在 → 直接返回
  ↓
检查 taskExists（原始任务是否存在）
  ↓
不存在 → 调用 exitTeammateView 退出
  ↓
检查 viewedTask（队友类型 narrow）
  ↓
不是队友类型 → 返回（不退出，允许查看 local_agent）
  ↓
检查状态:
  - status === 'killed' → 退出
  - status === 'failed' → 退出
  - viewedError 存在 → 退出
  - status 不是 running/completed/pending → 退出
  ↓
其他情况 → 保持查看状态
```

### 关键代码路径

#### 状态选择优化（行 13-18）
```typescript
const setAppState = useSetAppState()
const viewingAgentTaskId = useAppState(s => s.viewingAgentTaskId)
// Select only the viewed task, not the full tasks map — otherwise every
// streaming update from any teammate re-renders this hook.
const task = useAppState(s =>
  s.viewingAgentTaskId ? s.tasks[s.viewingAgentTaskId] : undefined,
)
```

关键优化：只选择被查看的任务，而不是整个任务映射，避免任何队友的流式更新都触发重渲染。

#### 任务类型 Narrow（行 20）
```typescript
const viewedTask = task && isInProcessTeammateTask(task) ? task : undefined
```

使用类型守卫 `isInProcessTeammateTask` 将任务 narrow 为队友类型。

#### 退出条件检查（行 31-54）
```typescript
useEffect(() => {
  // Not viewing any teammate
  if (!viewingAgentTaskId) {
    return
  }

  // Task no longer exists in the map — evicted out from under us.
  // Check raw `task` not teammate-narrowed `viewedTask`; local_agent
  // tasks exist but narrow to undefined, which would eject immediately.
  if (!taskExists) {
    exitTeammateView(setAppState)
    return
  }
  // Status checks below are teammate-only (viewedTask is teammate-narrowed).
  // For local_agent, viewedStatus is undefined → all checks falsy → no eject.
  if (!viewedTask) return

  // Auto-exit if teammate is killed, stopped, has error, or is no longer running
  // This handles shutdown scenarios where teammate becomes inactive
  if (
    viewedStatus === 'killed' ||
    viewedStatus === 'failed' ||
    viewedError ||
    (viewedStatus !== 'running' &&
      viewedStatus !== 'completed' &&
      viewedStatus !== 'pending')
  ) {
    exitTeammateView(setAppState)
    return
  }
}, [viewingAgentTaskId, taskExists, viewedTask, viewedStatus, viewedError, setAppState])
```

关键注释解释了检查逻辑：
1. 首先检查 `taskExists`（原始任务）而不是 narrow 后的 `viewedTask`
2. `local_agent` 任务存在但 narrow 为 undefined，如果检查 `viewedTask` 会立即退出
3. 对于 `local_agent`，`viewedStatus` 是 undefined，所有检查都为 false，不会退出

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `react` | `useEffect` |
| `../state/AppState.js` | `useAppState`, `useSetAppState` |
| `../state/teammateViewHelpers.js` | `exitTeammateView` |
| `../tasks/InProcessTeammateTask/types.js` | `isInProcessTeammateTask` |

### 外部交互

1. **AppState**: 
   - 读取 `viewingAgentTaskId` 和任务状态
   - 调用 `exitTeammateView` 退出查看模式

2. **队友任务系统**: 
   - `isInProcessTeammateTask()`: 类型守卫，narrow 任务类型

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**: 任务状态变化和 Effect 执行之间可能有竞态
2. **过度渲染**: 虽然已优化选择器，但任务对象引用变化仍可能触发重渲染

### 边界情况

1. **快速状态切换**: 队友状态快速变化时的处理
2. **completed 状态**: 特意不退出 completed 状态，允许用户审阅
3. **local_agent 任务**: 通过类型 narrow 排除，不自动退出

### 改进建议

1. **延迟退出**: 添加短暂延迟，避免状态瞬变导致的闪烁
2. **退出确认**: 对于重要操作，添加退出确认提示
3. **退出原因提示**: 退出时显示原因（如"队友已终止"）
4. **历史查看**: 支持查看已终止队友的历史记录
5. **手动退出**: 添加显式的退出按钮

### 测试关注点

1. 队友终止时的自动退出
2. 队友错误时的自动退出
3. 队友完成时不退出
4. local_agent 任务不触发退出
5. 任务删除时的退出
6. 快速状态变化的稳定性
