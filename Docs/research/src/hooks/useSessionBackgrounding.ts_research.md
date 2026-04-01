# useSessionBackgrounding.ts 深度研究文档

## 场景与职责

`useSessionBackgrounding` 是一个 React Hook，用于管理会话的后台化（Backgrounding）功能。它允许用户通过 `Ctrl+B` 将当前查询或前台任务切换到后台运行，同时也支持将后台任务重新前台化。

### 核心职责

1. **后台化当前查询**：当用户按下 `Ctrl+B` 时，将当前正在进行的查询转为后台任务
2. **重新后台化前台任务**：如果正在查看一个前台化的任务，将其重新放回后台
3. **同步前台任务状态**：将前台任务的消息和加载状态同步到主视图
4. **任务完成处理**：当前台任务完成时自动清理状态

### 使用场景

- **长时间运行的查询**：用户启动一个耗时操作后想继续其他工作
- **多任务并行**：同时运行多个独立的后台任务
- **任务监控**：前台查看后台任务的执行进度和消息
- **任务恢复**：将已完成的或正在运行的后台任务重新前台化查看

---

## 功能点目的

### 1. 后台化操作（Ctrl+B）

用户按下 `Ctrl+B` 时：
- 如果正在查看前台任务 → 将其重新后台化
- 否则 → 将当前查询转为后台任务

### 2. 前台任务状态同步

当 `foregroundedTaskId` 存在时：
- 同步任务消息到主视图
- 同步加载状态
- 同步中止控制器（支持 Escape 中止）

### 3. 任务完成自动清理

前台任务进入完成状态时：
- 自动将其标记为后台化
- 清空主视图消息
- 重置加载状态

### 4. 任务中止检测

检测前台任务是否被中止（用户按下 Escape）：
- 立即清理前台状态
- 将任务重新标记为后台化

---

## 具体技术实现

### 关键数据结构

```typescript
interface UseSessionBackgroundingProps {
  setMessages: (messages: Message[] | ((prev: Message[]) => Message[])) => void
  setIsLoading: (loading: boolean) => void
  resetLoadingState: () => void
  setAbortController: (controller: AbortController | null) => void
  onBackgroundQuery: () => void  // 后台化当前查询的回调
}

interface UseSessionBackgroundingResult {
  handleBackgroundSession: () => void  // Ctrl+B 处理器
}

// AppState 中的相关状态
interface AppState {
  foregroundedTaskId?: string           // 当前前台化的任务 ID
  tasks: Record<string, TaskState>     // 所有任务
}

interface TaskState {
  type: 'local_agent' | 'in_process_teammate' | ...
  status: 'running' | 'completed' | 'killed' | ...
  messages?: Message[]
  abortController?: AbortController
  isBackgrounded?: boolean
}
```

### 核心流程

#### 1. handleBackgroundSession 流程
```
handleBackgroundSession 调用
  ↓
检查 foregroundedTaskId 是否存在
  ├── 存在 → 重新后台化流程
  └── 不存在 → 后台化当前查询流程
```

**重新后台化前台任务**（行 41-64）：
```typescript
if (foregroundedTaskId) {
  setAppState(prev => {
    const taskId = prev.foregroundedTaskId
    if (!taskId) return prev
    const task = prev.tasks[taskId]
    if (!task) {
      return { ...prev, foregroundedTaskId: undefined }
    }
    return {
      ...prev,
      foregroundedTaskId: undefined,
      tasks: {
        ...prev.tasks,
        [taskId]: { ...task, isBackgrounded: true },
      },
    }
  })
  setMessages([])              // 清空主视图消息
  resetLoadingState()          // 重置加载状态
  setAbortController(null)     // 清除中止控制器
  return
}
```

**后台化当前查询**：
```typescript
onBackgroundQuery()  // 由父组件提供具体实现
```

#### 2. 前台任务同步 Effect 流程
```
foregroundedTaskId 或 foregroundedTask 变化
  ↓
无前台任务 → 重置 lastSyncedMessagesLengthRef
  ↓
任务不存在或类型不符 → 清理前台状态
  ↓
同步消息（仅当消息数量变化时）
  ↓
检查任务状态:
  ├── running → 检查中止状态，设置 loading=true
  └── 其他状态 → 自动清理前台状态
```

**消息同步逻辑**（行 91-97）：
```typescript
const taskMessages = foregroundedTask.messages ?? []
if (taskMessages.length !== lastSyncedMessagesLengthRef.current) {
  lastSyncedMessagesLengthRef.current = taskMessages.length
  setMessages([...taskMessages])  // 创建新数组触发更新
}
```

**中止检测**（行 99-121）：
```typescript
if (foregroundedTask.status === 'running') {
  const taskAbortController = foregroundedTask.abortController
  if (taskAbortController?.signal.aborted) {
    // 任务被中止 - 立即清理前台状态
    setAppState(prev => { /* ... */ })
    resetLoadingState()
    setAbortController(null)
    lastSyncedMessagesLengthRef.current = 0
    return
  }
  setIsLoading(true)
  if (taskAbortController) {
    setAbortController(taskAbortController)
  }
}
```

### 关键代码路径

#### 任务状态检查（行 84-89）
```typescript
if (!foregroundedTask || foregroundedTask.type !== 'local_agent') {
  setAppState(prev => ({ ...prev, foregroundedTaskId: undefined }))
  resetLoadingState()
  lastSyncedMessagesLengthRef.current = 0
  return
}
```

#### 任务完成清理（行 129-144）
```typescript
} else {
  // Task completed - restore to background and clear foregrounded view
  setAppState(prev => {
    const taskId = prev.foregroundedTaskId
    if (!taskId) return prev
    const task = prev.tasks[taskId]
    if (!task) return { ...prev, foregroundedTaskId: undefined }
    return {
      ...prev,
      foregroundedTaskId: undefined,
      tasks: { ...prev.tasks, [taskId]: { ...task, isBackgrounded: true } },
    }
  })
  resetLoadingState()
  setAbortController(null)
  lastSyncedMessagesLengthRef.current = 0
}
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `../state/AppState.js` | `useAppState`, `useSetAppState` |
| `../types/message.js` | `Message` 类型 |

### 外部交互

1. **AppState**: 
   - 读取 `foregroundedTaskId` 和任务状态
   - 更新 `foregroundedTaskId` 和 `tasks[taskId].isBackgrounded`

2. **父组件回调**:
   - `setMessages`: 同步消息到主视图
   - `setIsLoading`: 控制加载状态
   - `resetLoadingState`: 重置加载状态
   - `setAbortController`: 设置/清除中止控制器
   - `onBackgroundQuery`: 执行实际的后台化操作

---

## 风险、边界与改进建议

### 已知风险

1. **消息同步延迟**: 使用 `lastSyncedMessagesLengthRef` 优化，但可能导致极端情况下的不同步
2. **状态竞争**: `foregroundedTask` 从 AppState 读取时可能已经过期
3. **内存泄漏**: 如果任务被删除但 `foregroundedTaskId` 未清理，可能导致悬空引用

### 边界情况

1. **任务类型过滤**: 只处理 `type === 'local_agent'` 的任务，其他类型会被清理
2. **快速切换**: 用户快速前后台切换时的状态一致性
3. **任务删除**: 前台任务被删除时的处理
4. **中止状态检测**: 依赖 `abortController.signal.aborted`，需要确保控制器正确传递

### 改进建议

1. **消息内容对比**: 不仅比较长度，还比较内容哈希确保完全同步
2. **增量同步**: 只同步新增的消息，而不是整个数组
3. **任务恢复提示**: 任务完成时显示通知，提示用户查看结果
4. **多前台任务**: 支持同时前台化多个任务（分屏/标签页）
5. **任务状态持久化**: 刷新页面后恢复前台任务状态
6. **自动前台化**: 后台任务出错时自动前台化提示用户

### 测试关注点

1. Ctrl+B 后台化和重新后台化的状态转换
2. 前台任务消息同步的准确性
3. 任务中止时的状态清理
4. 任务完成后的自动清理
5. 任务删除时的处理
6. 快速前后台切换的稳定性
