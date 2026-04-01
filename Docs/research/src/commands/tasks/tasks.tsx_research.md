# 研究文档: src/commands/tasks/tasks.tsx

## 场景与职责

`src/commands/tasks/tasks.tsx` 是 `/tasks` 命令的实际实现模块。作为 **local-jsx** 类型命令的执行入口，它负责渲染后台任务管理对话框。

### 使用场景
- 用户通过 `/tasks` 或 `/bashes` 命令触发
- 在 TUI 中弹出模态对话框，展示所有后台运行的任务
- 支持任务查看、选择、终止、前台化等交互操作

### 核心职责
1. **命令执行入口**: 实现 `LocalJSXCommandCall` 接口的 `call` 函数
2. **组件渲染**: 返回 `BackgroundTasksDialog` React 组件
3. **上下文传递**: 将 `toolUseContext` 和 `onDone` 回调传递给对话框

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `call()` 函数 | 命令系统的标准执行接口，接收回调和上下文 |
| `onDone` 回调 | 命令完成时通知系统，可附带结果消息和显示选项 |
| `context` 参数 | 提供工具使用上下文，包含 AppState、消息操作等 |
| JSX 返回值 | 使用 Ink 渲染 TUI 界面 |

---

## 具体技术实现

### 关键类型定义

```typescript
// 来自 src/types/command.ts
export type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>

export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay  // 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void
```

### 代码实现分析

```typescript
import * as React from 'react'
import type { LocalJSXCommandContext } from '../../commands.js'
import { BackgroundTasksDialog } from '../../components/tasks/BackgroundTasksDialog.js'
import type { LocalJSXCommandOnDone } from '../../types/command.js'

export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
): Promise<React.ReactNode> {
  return <BackgroundTasksDialog toolUseContext={context} onDone={onDone} />
}
```

### 实现要点

1. **异步函数**: `call` 标记为 `async`，尽管内部无 await，但符合接口定义
2. **Props 传递**: 将 `context` 作为 `toolUseContext` 传递给组件
3. **回调传递**: `onDone` 直接传递给组件，由组件在适当时机调用

---

## 关键代码路径与文件引用

### 直接依赖

| 路径 | 用途 |
|------|------|
| `react` | React 核心库 |
| `../../commands.js` | `LocalJSXCommandContext` 类型 |
| `../../components/tasks/BackgroundTasksDialog.js` | 核心 UI 组件 |
| `../../types/command.js` | `LocalJSXCommandOnDone` 类型 |

### 调用链

```
用户输入 /tasks
    ↓
src/commands/tasks/index.ts 的 load() 被调用
    ↓
动态导入 ./tasks.js (本文件编译产物)
    ↓
执行 call(onDone, context)
    ↓
渲染 <BackgroundTasksDialog toolUseContext={context} onDone={onDone} />
    ↓
用户交互（选择任务、查看详情、终止任务等）
    ↓
调用 onDone(result, options) 关闭对话框
```

### BackgroundTasksDialog 核心功能

`BackgroundTasksDialog` 是实际的任务管理 UI，支持：

1. **任务列表展示**: 
   - Teammates (队友任务)
   - Shells (本地 shell 任务)
   - Monitors (MCP 监控任务)
   - Remote agents (远程 agent)
   - Local agents (本地 agent)
   - Workflows (工作流)
   - Dreams (Dream 任务)

2. **交互操作**:
   - `↑/↓` 选择任务
   - `Enter` 查看详情
   - `x` 终止运行中任务
   - `f` 前台化队友任务
   - `←/Esc` 关闭对话框

3. **详情视图**: 根据任务类型渲染不同的详情对话框
   - `ShellDetailDialog` - Shell 任务详情
   - `AsyncAgentDetailDialog` - 本地 Agent 详情
   - `RemoteSessionDetailDialog` - 远程会话详情
   - `InProcessTeammateDetailDialog` - 队友任务详情
   - `WorkflowDetailDialog` - 工作流详情
   - `MonitorMcpDetailDialog` - MCP 监控详情
   - `DreamDetailDialog` - Dream 任务详情

---

## 依赖与外部交互

### 类型系统

```typescript
// LocalJSXCommandContext 扩展自 ToolUseContext
interface LocalJSXCommandContext extends ToolUseContext {
  canUseTool?: CanUseToolFn
  setMessages: (updater: (prev: Message[]) => Message[]) => void
  options: {
    dynamicMcpConfig?: Record<string, ScopedMcpServerConfig>
    ideInstallationStatus: IDEExtensionInstallationStatus | null
    theme: ThemeName
  }
  onChangeAPIKey: () => void
  onChangeDynamicMcpConfig?: (config: Record<string, ScopedMcpServerConfig>) => void
  onInstallIDEExtension?: (ide: IdeType) => void
  resume?: (sessionId: UUID, log: LogOption, entrypoint: ResumeEntrypoint) => Promise<void>
}
```

### 状态交互

`BackgroundTasksDialog` 通过 `toolUseContext` 访问和修改 AppState：

| 状态字段 | 用途 |
|----------|------|
| `tasks` | 获取所有任务状态 |
| `foregroundedTaskId` | 当前前台任务 ID |
| `expandedView` | 视图展开模式 |
| `viewingAgentTaskId` | 正在查看的 Agent 任务 |

### 任务操作

组件支持对各类任务的终止操作：

```typescript
// 不同任务类型的终止方式
LocalShellTask.kill(taskId, setAppState)
LocalAgentTask.kill(taskId, setAppState)
InProcessTeammateTask.kill(taskId, setAppState)
DreamTask.kill(taskId, setAppState)
RemoteAgentTask.kill(taskId, setAppState)
killWorkflowTask(taskId, setAppState)  // 条件编译
killMonitorMcp(taskId, setAppState)    // 条件编译
stopUltraplan(taskId, sessionId, setAppState)  // Ultraplan 特殊处理
```

---

## 风险、边界与改进建议

### 潜在风险

| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 类型版本不匹配 | `LocalJSXCommandContext` 定义变更后此处未同步 | 保持类型定义稳定，使用 TypeScript 严格检查 |
| 循环依赖 | BackgroundTasksDialog 可能反向依赖命令系统 | 通过接口抽象避免直接依赖 |
| 内存泄漏 | 对话框关闭后回调引用未释放 | React 组件卸载时自动清理 |

### 边界情况

1. **无任务状态**: BackgroundTasksDialog 显示 "No tasks currently running"
2. **单任务优化**: 只有一个任务时直接进入详情视图
3. **Feature Flag**: Workflow 和 MonitorMcp 功能通过 `feature()` 条件编译控制
4. **权限控制**: 某些操作（如终止）需要任务处于运行状态

### 改进建议

1. **错误边界**: 添加 Error Boundary 捕获 BackgroundTasksDialog 渲染错误
   ```typescript
   export async function call(...): Promise<React.ReactNode> {
     return (
       <ErrorBoundary fallback={<Text>Error loading tasks</Text>}>
         <BackgroundTasksDialog ... />
       </ErrorBoundary>
     )
   }
   ```

2. **加载状态**: 若 BackgroundTasksDialog 支持异步加载，可添加 Suspense
   ```typescript
   return (
     <Suspense fallback={<Text>Loading tasks...</Text>}>
       <BackgroundTasksDialog ... />
     </Suspense>
   )
   ```

3. **参数支持**: 当前忽略 `args` 参数，可考虑支持任务 ID 直接跳转
   ```typescript
   export async function call(
     onDone: LocalJSXCommandOnDone,
     context: LocalJSXCommandContext,
     args: string,  // 可传入任务 ID
   ): Promise<React.ReactNode> {
     const initialTaskId = args.trim() || undefined
     return <BackgroundTasksDialog ... initialDetailTaskId={initialTaskId} />
   }
   ```

4. **类型导入优化**: 直接从 `../../types/command.js` 导入类型，避免通过 `commands.js` 间接导入

### 测试建议

- **单元测试**: 验证 `call()` 函数返回正确的 React 元素
- **集成测试**: 验证命令完整流程，从输入 `/tasks` 到对话框关闭
- **组件测试**: 测试 BackgroundTasksDialog 的各种交互场景
- **边界测试**: 无任务、单任务、多任务、任务状态变化等场景

### 性能考虑

1. **懒加载**: 通过 `index.ts` 的 `load()` 实现代码分割，减少启动时间
2. **Memoization**: BackgroundTasksDialog 内部使用 `useMemo` 优化任务列表计算
3. **选择器优化**: 使用 `useAppState` 选择器仅订阅必要状态片段
