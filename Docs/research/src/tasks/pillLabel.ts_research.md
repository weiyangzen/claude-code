# pillLabel.ts 研究文档

## 场景与职责

pillLabel.ts 是 Claude Code CLI 中负责生成**后台任务状态指示器标签**的纯工具模块。它在 UI 的底部状态栏（footer pill）和轮次持续时间（turn-duration）转录行中显示后台任务的摘要信息。

### 核心使用场景

1. **底部状态栏显示**：在 `BackgroundTaskStatus` 组件中显示当前后台任务的摘要
2. **转录行显示**：在 `SystemTextMessage` 的轮次持续时间消息中显示后台任务状态
3. **统一术语**：确保 footer pill 和 transcript 行使用一致的术语描述后台任务

---

## 功能点目的

### 1. 生成 Pill 标签 (`getPillLabel`)

根据后台任务列表生成简洁的人类可读标签：

- **统一类型任务**：如果所有任务类型相同，使用类型特定的描述
  - `local_bash`: 区分 shell 和 monitor，如 "2 shells, 1 monitor"
  - `in_process_teammate`: 按团队分组，如 "1 team" / "2 teams"
  - `local_agent`: "1 local agent" / "3 local agents"
  - `remote_agent`: 支持 ultraplan 状态显示，使用菱形符号 ◇/◆
  - `local_workflow`: "1 background workflow" / "2 background workflows"
  - `monitor_mcp`: "1 monitor" / "2 monitors"
  - `dream`: 固定显示 "dreaming"
  
- **混合类型任务**：如果任务类型不同，使用通用描述如 "3 background tasks"

### 2. 判断是否需要 CTA (`pillNeedsCta`)

确定 pill 是否应该显示 "· ↓ to view" 的提示：

- 仅在单个远程 ultraplan 任务且处于 `needs_input` 或 `plan_ready` 状态时显示
- 这是为了引导用户查看需要交互的 ultraplan 任务

---

## 具体技术实现

### 关键数据结构

```typescript
// 从 types.ts 导入的后台任务状态联合类型
import type { BackgroundTaskState } from './types.js'

// BackgroundTaskState 包含以下类型：
// - LocalShellTaskState (type: 'local_bash')
// - LocalAgentTaskState (type: 'local_agent')  
// - RemoteAgentTaskState (type: 'remote_agent')
// - InProcessTeammateTaskState (type: 'in_process_teammate')
// - LocalWorkflowTaskState (type: 'local_workflow')
// - MonitorMcpTaskState (type: 'monitor_mcp')
// - DreamTaskState (type: 'dream')
```

### 核心算法

#### getPillLabel

```typescript
export function getPillLabel(tasks: BackgroundTaskState[]): string {
  const n = tasks.length
  const allSameType = tasks.every(t => t.type === tasks[0]!.type)

  if (allSameType) {
    switch (tasks[0]!.type) {
      case 'local_bash': {
        // 统计 monitor 和 shell 数量
        const monitors = count(tasks, t => t.type === 'local_bash' && t.kind === 'monitor')
        const shells = n - monitors
        // 组合显示，如 "2 shells, 1 monitor"
      }
      case 'in_process_teammate': {
        // 按 teamName 去重统计团队数
        const teamCount = new Set(tasks.map(t => t.identity.teamName)).size
      }
      case 'local_agent':
        return n === 1 ? '1 local agent' : `${n} local agents`
      case 'remote_agent': {
        // 特殊处理 ultraplan 状态，使用菱形符号
        if (n === 1 && first.isUltraplan) {
          switch (first.ultraplanPhase) {
            case 'plan_ready': return `${DIAMOND_FILLED} ultraplan ready`
            case 'needs_input': return `${DIAMOND_OPEN} ultraplan needs your input`
            default: return `${DIAMOND_OPEN} ultraplan`
          }
        }
        return n === 1 ? `${DIAMOND_OPEN} 1 cloud session` : `${DIAMOND_OPEN} ${n} cloud sessions`
      }
      // ... 其他类型处理
    }
  }

  // 混合类型使用通用描述
  return `${n} background ${n === 1 ? 'task' : 'tasks'}`
}
```

#### pillNeedsCta

```typescript
export function pillNeedsCta(tasks: BackgroundTaskState[]): boolean {
  if (tasks.length !== 1) return false
  const t = tasks[0]!
  return (
    t.type === 'remote_agent' &&
    t.isUltraplan === true &&
    t.ultraplanPhase !== undefined
  )
}
```

### 符号常量

```typescript
// 从 figures.ts 导入
import { DIAMOND_FILLED, DIAMOND_OPEN } from '../constants/figures.js'

// DIAMOND_OPEN = '◇' (\u25c7) - 运行中
// DIAMOND_FILLED = '◆' (\u25c6) - 完成/失败/需要输入
```

### 辅助函数

```typescript
// 从 array.ts 导入的计数函数
import { count } from '../utils/array.js'

// count 实现：
export function count<T>(arr: readonly T[], pred: (x: T) => unknown): number {
  let n = 0
  for (const x of arr) n += +!!pred(x)
  return n
}
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件路径 | 用途 |
|---------|------|
| `src/tasks/pillLabel.ts` | 本文件，pill 标签生成逻辑 |
| `src/tasks/types.ts` | `BackgroundTaskState` 类型定义 |
| `src/constants/figures.ts` | 菱形符号常量定义 |
| `src/utils/array.ts` | `count` 辅助函数 |

### 调用方文件

| 文件路径 | 用途 |
|---------|------|
| `src/components/tasks/BackgroundTaskStatus.tsx` | 底部状态栏组件，显示后台任务 pill |
| `src/components/messages/SystemTextMessage.tsx` | 系统消息组件，在 turn_duration 消息中显示 |

---

## 依赖与外部交互

### 类型依赖

```typescript
// 后台任务状态类型
import type { BackgroundTaskState } from './types.js'

// BackgroundTaskState 是以下类型的联合：
// - LocalShellTaskState
// - LocalAgentTaskState  
// - RemoteAgentTaskState
// - InProcessTeammateTaskState
// - LocalWorkflowTaskState
// - MonitorMcpTaskState
// - DreamTaskState
```

### 外部依赖

1. **图形常量** (`constants/figures.ts`)
   - `DIAMOND_FILLED`: 填充菱形 ◆
   - `DIAMOND_OPEN`: 空心菱形 ◇

2. **数组工具** (`utils/array.ts`)
   - `count`: 统计满足条件的元素数量

---

## 风险、边界与改进建议

### 已知风险

1. **空数组风险**
   - `getPillLabel` 假设 `tasks` 非空，直接访问 `tasks[0]!`
   - 如果传入空数组会导致运行时错误

2. **类型安全**
   - 使用非空断言 `tasks[0]!` 和 `first!`
   - 虽然调用方通常确保非空，但缺乏运行时检查

3. **远程 agent 状态依赖**
   - ultraplan 状态显示依赖 `isUltraplan` 和 `ultraplanPhase` 字段
   - 这些字段可能在某些 RemoteAgentTaskState 中不存在

### 边界情况

1. **混合类型任务**
   - 当任务类型混合时，使用通用描述 "N background tasks"
   - 这可能丢失重要的状态信息（如 ultraplan 需要输入）

2. **单个任务的精确显示**
   - 对于单个远程 agent 任务，显示详细的 ultraplan 状态
   - 对于多个任务，即使包含 ultraplan 也只显示 "N cloud sessions"

3. **InProcessTeammate 团队统计**
   - 使用 `Set` 去重统计团队数
   - 空 teamName 会被计入（使用 `''` 作为默认值）

### 改进建议

1. **空数组保护**
   ```typescript
   export function getPillLabel(tasks: BackgroundTaskState[]): string {
     if (tasks.length === 0) return 'No background tasks'
     // ...
   }
   ```

2. **混合类型时的重要状态提示**
   - 考虑在混合类型时，如果有需要关注的任务（如 ultraplan needs_input），优先显示该状态

3. **类型守卫增强**
   - 添加运行时类型检查，确保字段存在后再访问

4. **国际化支持**
   - 当前标签是硬编码的英文
   - 考虑支持多语言

5. **测试覆盖**
   - 建议增加单元测试：
     - 各种任务类型的单数和复数形式
     - 混合类型任务
     - ultraplan 各种状态
     - 空数组边界情况
     - `pillNeedsCta` 的各种条件组合

6. **性能优化**
   - 当前实现遍历数组多次（`every`, `count`, `map`, `filter`）
   - 对于大量任务，可以考虑单次遍历优化
   - 但实际场景中后台任务数量通常较少（<10），当前实现足够高效
