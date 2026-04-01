# useTaskListWatcher.ts 深度研究文档

## 场景与职责

`useTaskListWatcher` 是一个 React Hook，用于监视任务列表目录并自动领取可执行的任务。它实现了"任务模式"（Tasks Mode），使 Claude 能够监视外部创建的任务并逐个处理它们。

### 核心职责

1. **任务目录监视**: 使用 `fs.watch` 监视任务目录变化
2. **可用任务发现**: 查找状态为 pending、无所有者、未被阻塞的任务
3. **任务领取**: 使用文件锁原子性地领取任务
4. **任务提交**: 将领取的任务作为提示词提交给 REPL
5. **任务状态跟踪**: 跟踪当前正在处理的任务

### 使用场景

- **任务模式**: Claude 作为任务处理器自动执行外部创建的任务
- **团队协作**: 多个 Claude 实例共享任务队列
- **自动化工作流**: 外部系统创建任务，Claude 自动处理

---

## 功能点目的

### 1. 任务目录监视

使用 `fs.watch` 监视任务目录：
- 防抖处理（1000ms）避免频繁触发
- 使用 `unref()` 防止阻止进程退出
- 优雅处理目录不存在的情况

### 2. 任务领取逻辑

查找可执行任务的标准：
- 状态为 `pending`
- 无所有者（`owner` 未设置）
- 未被未解决的任务阻塞（`blockedBy` 检查）

### 3. 原子性任务领取

使用文件锁确保原子性：
- 调用 `claimTask` 尝试领取
- 处理领取失败的各种原因
- 成功后提交任务

### 4. 任务提交与释放

将任务格式化为提示词提交：
- 格式化任务主题为提示词
- 调用 `onSubmitTask` 提交
- 提交失败时释放任务所有权

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface Props {
  taskListId?: string           // 任务列表 ID（也是 agent ID）
  isLoading: boolean            // 是否正在处理请求
  onSubmitTask: (prompt: string) => boolean  // 提交任务回调
}

// Task 类型（来自 ../utils/tasks.js）
interface Task {
  id: string
  subject: string
  description: string
  owner?: string                // agent ID
  status: 'pending' | 'in_progress' | 'completed'
  blocks: string[]              // 此任务阻塞的任务 ID
  blockedBy: string[]           // 阻塞此任务的任务 ID
  metadata?: Record<string, unknown>
}

// 领取结果
interface ClaimTaskResult {
  success: boolean
  reason?: 'task_not_found' | 'already_claimed' | 'already_resolved' | 'blocked' | 'agent_busy'
  task?: Task
  blockedByTasks?: string[]
}
```

### 核心流程

#### 1. Hook 初始化流程
```
useEffect 触发
  ↓
检查 enabled（taskListId 是否存在）
  ↓
确保任务目录存在 ensureTasksDir
  ↓
获取任务目录路径
  ↓
设置防抖检查函数
  ↓
创建 fs.watch 监视器
  ↓
执行初始检查
  ↓
返回清理函数（关闭监视器、清除定时器）
```

#### 2. 任务检查流程
```
checkForTasks 调用
  ↓
检查 enabled
  ↓
检查 isLoadingRef.current → true 则返回
  ↓
获取任务列表 listTasks
  ↓
检查当前任务状态:
  如果 currentTaskRef 不为 null:
    查找当前任务
    如果任务不存在或已完成 → 重置 currentTaskRef
    否则 → 返回（仍在处理中）
  ↓
查找可用任务 findAvailableTask
  ↓
无可用任务 → 返回
  ↓
记录日志
  ↓
领取任务 claimTask
  ↓
领取失败 → 记录日志，返回
  ↓
更新 currentTaskRef
  ↓
格式化任务为提示词 formatTaskAsPrompt
  ↓
提交任务 onSubmitTaskRef.current
  ↓
提交失败 → 释放任务所有权，重置 currentTaskRef
```

#### 3. 可用任务查找逻辑（行 197-207）
```typescript
function findAvailableTask(tasks: Task[]): Task | undefined {
  const unresolvedTaskIds = new Set(
    tasks.filter(t => t.status !== 'completed').map(t => t.id),
  )

  return tasks.find(task => {
    if (task.status !== 'pending') return false
    if (task.owner) return false
    // Check all blockers are completed
    return task.blockedBy.every(id => !unresolvedTaskIds.has(id))
  })
}
```

### 关键代码路径

#### Ref 稳定化模式（行 42-50）
```typescript
// Stabilize unstable props via refs so the watcher effect doesn't depend on
// them. isLoading flips every turn, and onSubmitTask's identity changes
// whenever onQuery's deps change. Without this, the watcher effect re-runs
// on every turn, calling watcher.close() + watch() each time — which is a
// trigger for Bun's PathWatcherManager deadlock (oven-sh/bun#27469).
const isLoadingRef = useRef(isLoading)
isLoadingRef.current = isLoading
const onSubmitTaskRef = useRef(onSubmitTask)
onSubmitTaskRef.current = onSubmitTask
```

关键注释解释了使用 Ref 的原因：避免 Bun 的 PathWatcherManager 死锁。

#### 任务提示词格式化（行 213-221）
```typescript
function formatTaskAsPrompt(task: Task): string {
  let prompt = `Complete all open tasks. Start with task #${task.id}: \n\n ${task.subject}`

  if (task.description) {
    prompt += `\n\n${task.description}`
  }

  return prompt
}
```

#### 空闲触发检查（行 184-188）
```typescript
useEffect(() => {
  if (!enabled) return
  if (isLoading) return
  scheduleCheckRef.current()
}, [enabled, isLoading])
```

当 `isLoading` 变为 false 时，安排一次检查以获取下一个任务。

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `fs` | `FSWatcher`, `watch` |
| `../state/AppState.js` | `useAppState`, `useSetAppState` |
| `../Task.js` | `isTerminalTaskStatus` |
| `../tasks/InProcessTeammateTask/InProcessTeammateTask.js` | 队友任务查找和消息注入 |
| `../tools/ScheduleCronTool/prompt.js` | `isKairosCronEnabled` |
| `../types/message.js` | `Message` 类型 |
| `../utils/cronJitterConfig.js` | `getCronJitterConfig` |
| `../utils/cronScheduler.js` | `createCronScheduler` |
| `../utils/cronTasks.js` | `removeCronTasks` |
| `../utils/debug.js` | `logForDebugging` |
| `../utils/messageQueueManager.js` | `enqueuePendingNotification` |
| `../utils/messages.js` | `createScheduledTaskFireMessage` |
| `../utils/workloadContext.js` | `WORKLOAD_CRON` |

### 外部交互

1. **任务系统**: 
   - `listTasks()`: 获取任务列表
   - `claimTask()`: 领取任务
   - `updateTask()`: 更新任务（释放所有权）
   - `ensureTasksDir()`: 确保目录存在
   - `getTasksDir()`: 获取任务目录路径

2. **文件系统**: 
   - `fs.watch()`: 监视目录变化

3. **父组件**: 
   - `onSubmitTask`: 提交任务提示词

---

## 风险、边界与改进建议

### 已知风险

1. **Bun 死锁**: 注释提到 `watcher.close() + watch()` 可能触发 Bun 的 PathWatcherManager 死锁
2. **任务丢失**: 如果进程崩溃，正在处理的任务状态可能不一致
3. **竞争条件**: 多个 Claude 实例可能同时尝试领取同一任务

### 边界情况

1. **任务列表 ID 变化**: `taskListId` 变化时会重新设置监视器
2. **目录不存在**: `fs.watch` 在目录不存在时抛出错误，已被捕获
3. **任务提交失败**: 提交失败时会尝试释放任务所有权
4. **快速连续变化**: 防抖处理避免频繁检查

### 改进建议

1. **持久化状态**: 将任务处理状态持久化，支持崩溃恢复
2. **心跳机制**: 添加任务处理心跳，检测僵死任务
3. **优先级支持**: 支持任务优先级，优先处理高优先级任务
4. **批量领取**: 支持一次领取多个无依赖的任务
5. **任务超时**: 为任务处理添加超时机制
6. **进度报告**: 定期报告任务处理进度

### 测试关注点

1. 任务目录变化时的正确检测
2. 任务领取的原子性
3. 任务阻塞关系的正确处理
4. 任务提交失败时的释放逻辑
5. 组件卸载时的资源清理
6. Bun 死锁问题的避免
