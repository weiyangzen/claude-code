# useBackgroundTaskNavigation.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useBackgroundTaskNavigation.ts` 是 Claude Code 的后台任务键盘导航 Hook，处理 Shift+Up/Down 在队友（teammates）和后台任务之间的导航，支持查看、终止等操作。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 队友导航 | Shift+Up/Down 在 Leader 和队友之间切换 |
| 查看转录 | 选中队友后按 'f' 查看完整对话 |
| 终止队友 | 选中运行中队友后按 'k' 终止 |
| 确认选择 | 按 Enter 进入选中队友的视图 |
| 后台任务对话框 | 无非队友后台任务时打开对话框 |

### 1.3 调用方
- `src/screens/REPL.tsx` - 主应用界面
- `src/components/PromptInput/PromptInput.tsx` - 输入组件
- `src/cli/print.ts` - CLI 打印
- `src/hooks/useCancelRequest.ts` - 取消请求
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` - 队友任务

---

## 2. 功能点目的

### 2.1 队友选择导航
- **目的**：在 Leader 和多个队友之间快速切换
- **实现**：`selectedIPAgentIndex` 状态，-1 表示 Leader
- **循环**：支持在 Leader(-1) → 队友(0..n-1) → 隐藏(n) 之间循环

### 2.2 视图模式切换
- **选择模式** (`selecting-agent`)：显示队友列表，可选择
- **查看模式** (`viewing-agent`)：查看选中队友的完整转录

### 2.3 键盘操作
| 按键 | 作用 |
|------|------|
| Shift+Up/Down | 导航队友选择 |
| Enter | 确认选择/进入视图 |
| f | 查看选中队友转录 |
| k | 终止选中队友 |
| Escape | 退出选择/视图模式 |

### 2.4 动态适应
- **目的**：队友数量变化时自动调整选择
- **实现**：`useEffect` 监听 `teammateCount`，钳制或重置索引

---

## 3. 具体技术实现

### 3.1 类型定义

```typescript
export function useBackgroundTaskNavigation(options?: {
  onOpenBackgroundTasks?: () => void  // 无非队友任务时回调
}): { handleKeyDown: (e: KeyboardEvent) => void }
```

### 3.2 队友选择步进

```typescript
function stepTeammateSelection(
  delta: 1 | -1,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  setAppState(prev => {
    const currentCount = getRunningTeammatesSorted(prev.tasks).length
    if (currentCount === 0) return prev

    // 首次从折叠树展开，定位到 Leader
    if (prev.expandedView !== 'teammates') {
      return {
        ...prev,
        expandedView: 'teammates',
        viewSelectionMode: 'selecting-agent',
        selectedIPAgentIndex: -1,
      }
    }

    const maxIdx = currentCount  // 包含 "hide" 行
    const cur = prev.selectedIPAgentIndex
    
    // 循环步进逻辑
    const next =
      delta === 1
        ? cur >= maxIdx ? -1 : cur + 1      // 向下
        : cur <= -1 ? maxIdx : cur - 1      // 向上

    return {
      ...prev,
      selectedIPAgentIndex: next,
      viewSelectionMode: 'selecting-agent',
    }
  })
}
```

### 3.3 动态索引调整

```typescript
useEffect(() => {
  const prevCount = prevTeammateCountRef.current
  prevTeammateCountRef.current = teammateCount

  setAppState(prev => {
    const currentTeammates = getRunningTeammatesSorted(prev.tasks)
    const currentCount = currentTeammates.length

    // 队友全部移除时重置
    if (
      currentCount === 0 &&
      prevCount > 0 &&
      prev.selectedIPAgentIndex !== -1
    ) {
      if (prev.viewSelectionMode === 'viewing-agent') {
        return { ...prev, selectedIPAgentIndex: -1 }
      }
      return {
        ...prev,
        selectedIPAgentIndex: -1,
        viewSelectionMode: 'none',
      }
    }

    // 索引越界时钳制
    const maxIndex =
      prev.expandedView === 'teammates' ? currentCount : currentCount - 1
    if (currentCount > 0 && prev.selectedIPAgentIndex > maxIndex) {
      return { ...prev, selectedIPAgentIndex: maxIndex }
    }

    return prev
  })
}, [teammateCount, setAppState])
```

### 3.4 键盘处理

```typescript
const handleKeyDown = (e: KeyboardEvent): void => {
  // Escape 在查看模式：中止当前工作或退出
  if (e.key === 'escape' && viewSelectionMode === 'viewing-agent') {
    e.preventDefault()
    const taskId = viewingAgentTaskId
    if (taskId) {
      const task = tasks[taskId]
      if (isInProcessTeammateTask(task) && task.status === 'running') {
        // 中止当前工作（非终止队友）
        task.currentWorkAbortController?.abort()
        return
      }
    }
    exitTeammateView(setAppState)
    return
  }

  // Escape 在选择模式：退出选择
  if (e.key === 'escape' && viewSelectionMode === 'selecting-agent') {
    e.preventDefault()
    setAppState(prev => ({
      ...prev,
      viewSelectionMode: 'none',
      selectedIPAgentIndex: -1,
    }))
    return
  }

  // Shift+Up/Down 导航
  if (e.shift && (e.key === 'up' || e.key === 'down')) {
    e.preventDefault()
    if (teammateCount > 0) {
      stepTeammateSelection(e.key === 'down' ? 1 : -1, setAppState)
    } else if (hasNonTeammateBackgroundTasks) {
      options?.onOpenBackgroundTasks?.()
    }
    return
  }

  // 'f' 查看转录
  if (e.key === 'f' && viewSelectionMode === 'selecting-agent' && teammateCount > 0) {
    e.preventDefault()
    const selected = getSelectedTeammate()
    if (selected) {
      enterTeammateView(selected.taskId, setAppState)
    }
    return
  }

  // Enter 确认选择
  if (e.key === 'return' && viewSelectionMode === 'selecting-agent') {
    e.preventDefault()
    if (selectedIPAgentIndex === -1) {
      exitTeammateView(setAppState)  // 选择 Leader，退出视图
    } else if (selectedIPAgentIndex >= teammateCount) {
      // 选择 "Hide"，折叠树
      setAppState(prev => ({
        ...prev,
        expandedView: 'none',
        viewSelectionMode: 'none',
        selectedIPAgentIndex: -1,
      }))
    } else {
      const selected = getSelectedTeammate()
      if (selected) {
        enterTeammateView(selected.taskId, setAppState)
      }
    }
    return
  }

  // 'k' 终止队友
  if (e.key === 'k' && viewSelectionMode === 'selecting-agent' && selectedIPAgentIndex >= 0) {
    e.preventDefault()
    const selected = getSelectedTeammate()
    if (selected && selected.task.status === 'running') {
      void InProcessTeammateTask.kill(selected.taskId, setAppState)
    }
    return
  }
}
```

### 3.5 获取选中队友

```typescript
const getSelectedTeammate = (): {
  taskId: string
  task: InProcessTeammateTaskState
} | null => {
  if (teammateCount === 0) return null
  const selectedIndex = selectedIPAgentIndex
  const task = teammateTasks[selectedIndex]
  if (!task) return null

  return { taskId: task.id, task }
}
```

### 3.6 向后兼容桥接

```typescript
// TODO(onKeyDown-migration): REPL.tsx 尚未将 handleKeyDown 绑定到 <Box onKeyDown>
// 临时通过 useInput 适配
useInput((_input, _key, event) => {
  handleKeyDown(new KeyboardEvent(event.keypress))
})
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
useBackgroundTaskNavigation.ts
├── react (useEffect, useRef)
├── ink/events/keyboard-event.js    (KeyboardEvent)
├── ink.js                          (useInput)
├── state/AppState.js
│   ├── useAppState
│   └── useSetAppState
├── state/teammateViewHelpers.js
│   ├── enterTeammateView
│   └── exitTeammateView
├── tasks/InProcessTeammateTask/InProcessTeammateTask.js
│   └── getRunningTeammatesSorted
├── tasks/InProcessTeammateTask/types.js
│   └── isInProcessTeammateTask
└── tasks/types.js                  (isBackgroundTask)
```

### 4.2 调用链

```
PromptInput.tsx / REPL.tsx
  └── useBackgroundTaskNavigation()
      ├── useEffect (动态索引调整)
      │   └── setAppState (钳制/重置)
      ├── handleKeyDown (键盘处理)
      │   ├── stepTeammateSelection()
      │   ├── enterTeammateView()
      │   ├── exitTeammateView()
      │   └── InProcessTeammateTask.kill()
      └── useInput (向后兼容)
```

### 4.3 状态交互

```typescript
// AppState 相关字段
interface AppState {
  tasks: Record<string, TaskState>
  expandedView: 'none' | 'teammates' | ...
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  selectedIPAgentIndex: number  // -1 = leader
  viewingAgentTaskId: string | null
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `react` | Hooks API | npm 包 |

### 5.2 状态管理

| 函数/钩子 | 来源 | 用途 |
|-----------|------|------|
| `useAppState()` | `AppState.js` | 读取应用状态 |
| `useSetAppState()` | `AppState.js` | 更新应用状态 |
| `getRunningTeammatesSorted()` | `InProcessTeammateTask.js` | 获取运行中队友 |

### 5.3 视图助手

| 函数 | 来源 | 用途 |
|------|------|------|
| `enterTeammateView()` | `teammateViewHelpers.js` | 进入队友视图 |
| `exitTeammateView()` | `teammateViewHelpers.js` | 退出队友视图 |

### 5.4 任务操作

| 函数 | 来源 | 用途 |
|------|------|------|
| `InProcessTeammateTask.kill()` | `InProcessTeammateTask.js` | 终止队友任务 |
| `isInProcessTeammateTask()` | `types.js` | 类型守卫 |
| `isBackgroundTask()` | `types.js` | 后台任务检查 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 索引越界 | 队友移除后索引无效 | `useEffect` 动态钳制 |
| 竞态条件 | 快速按键导致状态混乱 | `e.preventDefault()` |
| 向后兼容 | useInput 桥接可能冲突 | TODO 标记待移除 |
| 任务状态 | kill 时任务可能已完成 | 状态检查前置 |

### 6.2 边界条件

1. **无队友**：Shift+Up/Down 打开后台任务对话框
2. **单个队友**：在 -1, 0, 1（hide）之间循环
3. **队友完成**：查看模式下 Escape 退出而非中止
4. **快速移除**：索引自动重置到 -1
5. **非交互模式**：键盘事件不会触发

### 6.3 改进建议

1. **键盘快捷键配置**：
   ```typescript
   const keybindings = useKeybindings({
     'teammate:navigateUp': 'shift+up',
     'teammate:navigateDown': 'shift+down',
     'teammate:view': 'f',
     'teammate:kill': 'k',
   })
   ```

2. **视觉反馈**：
   ```typescript
   // 选中队友时高亮显示
   const { highlightTeammate } = useTeammateHighlight()
   ```

3. **批量操作**：
   ```typescript
   // 支持选择多个队友批量终止
   const [selectedIds, setSelectedIds] = useState<string[]>([])
   ```

4. **撤销支持**：
   ```typescript
   // 终止后支持撤销
   const { kill, undoKill } = useKillWithUndo()
   ```

5. **完成迁移**：
   ```typescript
   // 移除 useInput 桥接
   // REPL.tsx: <Box onKeyDown={handleKeyDown}>
   ```

### 6.4 代码质量

- **优点**：
  - 清晰的导航逻辑和循环处理
  - 完善的动态索引调整
  - 区分中止工作和终止队友
  
- **潜在改进**：
  - 提取键盘映射为配置
  - 添加更多 JSDoc 说明索引约定
  - 考虑使用 reducer 管理复杂状态转换

### 6.5 相关文档

- `state/teammateViewHelpers.js` - 视图切换助手
- `tasks/InProcessTeammateTask/` - 队友任务实现
- `state/AppState.js` - 全局状态定义
