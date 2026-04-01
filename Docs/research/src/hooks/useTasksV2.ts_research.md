# useTasksV2.ts 深度研究文档

## 场景与职责

`useTasksV2` 是一个 React Hook，用于获取当前任务列表以进行持久化 UI 显示。它通过单例存储模式实现多个组件实例共享同一个文件监视器，避免每个组件都创建独立的 `fs.watch`。

### 核心职责

1. **任务列表获取**: 从文件系统读取任务列表
2. **单例存储**: 所有 Hook 实例共享一个 `TasksV2Store`
3. **文件监视**: 使用 `fs.watch` 监视任务目录变化
4. **自动隐藏**: 所有任务完成后 5 秒自动隐藏任务列表
5. **回退轮询**: 当 `fs.watch` 遗漏事件时使用轮询作为后备

### 使用场景

- **REPL 界面**: 显示当前任务列表
- **Spinner 组件**: 显示任务加载状态
- **PromptInputFooterLeftSide**: 在输入框旁显示任务状态
- **任何需要显示任务列表的组件**

---

## 功能点目的

### 1. 单例存储模式

`TasksV2Store` 类实现单例模式：
- 所有 Hook 实例共享同一个存储
- 第一个订阅者启动存储，最后一个取消订阅者停止存储
- 避免多个 `fs.watch` 实例（特别是 Spinner 每 turn 挂载/卸载）

### 2. 自动隐藏机制

任务列表自动隐藏逻辑：
- 所有任务完成后启动 5 秒定时器
- 定时器触发后重置任务列表
- 有新任务时立即显示

### 3. 回退轮询

当存在未完成任务时：
- 每 5 秒轮询一次任务列表
- 作为 `fs.watch` 遗漏事件的保险
- 所有任务完成后停止轮询

### 4. 团队上下文感知

根据团队角色控制启用：
- 仅团队领导显示任务列表
- 队友不显示（他们有自己的任务视图）

---

## 具体技术实现

### 关键数据结构

```typescript
// TasksV2Store 私有字段
class TasksV2Store {
  #tasks: Task[] | undefined = undefined    // 缓存的任务列表
  #hidden = false                           // 是否隐藏
  #watcher: FSWatcher | null = null         // 文件监视器
  #watchedDir: string | null = null         // 当前监视的目录
  #hideTimer: Timeout | null = null         // 隐藏定时器
  #debounceTimer: Timeout | null = null     // 防抖定时器
  #pollTimer: Timeout | null = null         // 轮询定时器
  #unsubscribeTasksUpdated: (() => void) | null = null
  #changed = createSignal()                 // 变更信号
  #subscriberCount = 0                      // 订阅者计数
  #started = false                          // 是否已启动
}

// Hook 返回类型
function useTasksV2(): Task[] | undefined

// 带折叠效果的版本
function useTasksV2WithCollapseEffect(): Task[] | undefined
```

### 核心流程

#### 1. 存储启动流程
```
第一个订阅者调用 subscribe
  ↓
订阅者计数增加
  ↓
检查 #started
  ↓
未启动 → 启动存储:
  1. 订阅 onTasksUpdated
  2. 调用 #fetch() 获取初始数据
  3. 设置 #started = true
```

#### 2. 数据获取流程（#fetch）
```
#fetch 调用
  ↓
获取当前 taskListId
  ↓
调用 #rewatch(getTasksDir(taskListId)) 更新监视目录
  ↓
调用 listTasks(taskListId) 获取任务列表
  ↓
过滤掉内部任务（metadata._internal）
  ↓
更新 #tasks
  ↓
检查是否有未完成任务
  ↓
有未完成任务或列表为空:
  - 设置 #hidden = (列表长度 === 0)
  - 清除隐藏定时器
  ↓
所有任务刚完成且未隐藏:
  - 启动 5 秒隐藏定时器
  ↓
触发变更通知 #notify()
  ↓
设置/清除轮询定时器
```

#### 3. 隐藏定时器触发流程（#onHideTimerFired）
```
定时器触发
  ↓
清除定时器引用
  ↓
检查 taskListId 是否变化（防止重置错误的列表）
  ↓
验证所有任务仍已完成
  ↓
调用 resetTaskList(currentId) 重置任务列表
  ↓
更新 #tasks = []
  ↓
设置 #hidden = true
  ↓
触发变更通知
```

#### 4. Hook 使用流程
```
组件调用 useTasksV2()
  ↓
检查 isTodoV2Enabled()
  ↓
检查团队上下文（仅领导显示）
  ↓
获取/创建单例 store
  ↓
调用 useSyncExternalStore:
  - subscribe: store.subscribe
  - getSnapshot: store.getSnapshot
  ↓
返回任务列表或 undefined
```

### 关键代码路径

#### 单例实现（行 201-204）
```typescript
let _store: TasksV2Store | null = null
function getStore(): TasksV2Store {
  return (_store ??= new TasksV2Store())
}
```

#### 订阅管理（行 57-79）
```typescript
subscribe = (fn: () => void): (() => void) => {
  const unsubscribe = this.#changed.subscribe(fn)
  this.#subscriberCount++
  if (!this.#started) {
    this.#started = true
    this.#unsubscribeTasksUpdated = onTasksUpdated(this.#debouncedFetch)
    void this.#fetch()
  }
  let unsubscribed = false
  return () => {
    if (unsubscribed) return
    unsubscribed = true
    unsubscribe()
    this.#subscriberCount--
    if (this.#subscriberCount === 0) this.#stop()
  }
}
```

#### 目录重新监视（行 90-105）
```typescript
#rewatch(dir: string): void {
  if (dir === this.#watchedDir && this.#watcher !== null) return
  this.#watcher?.close()
  this.#watcher = null
  this.#watchedDir = dir
  try {
    this.#watcher = watch(dir, this.#debouncedFetch)
    this.#watcher.unref()
  } catch {
    // Directory may not exist yet
  }
}
```

#### 隐藏效果 Hook（行 236-250）
```typescript
export function useTasksV2WithCollapseEffect(): Task[] | undefined {
  const tasks = useTasksV2()
  const setAppState = useSetAppState()

  const hidden = tasks === undefined
  useEffect(() => {
    if (!hidden) return
    setAppState(prev => {
      if (prev.expandedView !== 'tasks') return prev
      return { ...prev, expandedView: 'none' as const }
    })
  }, [hidden, setAppState])

  return tasks
}
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `fs` | `FSWatcher`, `watch` |
| `react` | `useSyncExternalStore`, `useEffect` |
| `../state/AppState.js` | `useAppState`, `useSetAppState` |
| `../utils/signal.js` | `createSignal` |
| `../utils/tasks.js` | `Task`, `getTaskListId`, `getTasksDir`, `isTodoV2Enabled`, `listTasks`, `onTasksUpdated`, `resetTaskList` |
| `../utils/teammate.js` | `isTeamLead` |

### 外部交互

1. **任务系统**: 
   - `listTasks()`: 获取任务列表
   - `resetTaskList()`: 重置任务列表
   - `onTasksUpdated()`: 订阅任务更新
   - `getTaskListId()`: 获取当前任务列表 ID
   - `getTasksDir()`: 获取任务目录

2. **文件系统**: 
   - `fs.watch()`: 监视目录变化

3. **AppState**: 
   - `useAppState`: 获取团队上下文
   - `useSetAppState`: 更新 expandedView

4. **React**: 
   - `useSyncExternalStore`: 订阅外部存储

---

## 风险、边界与改进建议

### 已知风险

1. **内存泄漏**: 如果订阅者未正确取消订阅，存储可能永远不会停止
2. **竞态条件**: 任务列表 ID 变化时可能有短暂的过时数据
3. **定时器累积**: 快速切换隐藏状态时可能有定时器累积

### 边界情况

1. **空任务列表**: 列表为空时立即隐藏
2. **任务列表 ID 变化**: 团队创建/删除时正确切换监视目录
3. **文件系统错误**: 目录不存在时优雅降级到轮询
4. **组件快速挂载/卸载**: Spinner 每 turn 挂载/卸载不会导致问题

### 改进建议

1. **错误重试**: 任务获取失败时添加重试机制
2. **增量更新**: 只获取变化的任务而不是整个列表
3. **虚拟列表**: 大量任务时使用虚拟列表优化渲染
4. **任务排序**: 支持按优先级、截止日期等排序
5. **过滤功能**: 支持按状态、所有者等过滤任务
6. **持久化展开状态**: 记住用户的展开/折叠偏好

### 测试关注点

1. 单例存储的正确共享
2. 第一个订阅者启动、最后一个停止
3. 任务列表 ID 变化时的目录切换
4. 5 秒隐藏定时器的正确触发
5. 回退轮询的启停
6. 团队角色对显示的影响
