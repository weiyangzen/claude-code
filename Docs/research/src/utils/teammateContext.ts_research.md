# teammateContext.ts 研究文档

## 场景与职责

`teammateContext.ts` 是 Claude Code 进程内 teammate（in-process teammate）系统的核心上下文管理模块。它基于 Node.js 的 `AsyncLocalStorage` 实现了**并发安全的执行上下文隔离**，使得多个 teammate 可以在同一个 Node.js 进程中并行运行而不会相互干扰。

### 核心职责

1. **执行上下文隔离**：使用 AsyncLocalStorage 为每个进程内 teammate 提供独立的执行上下文
2. **身份状态管理**：存储 teammate 的完整身份信息（agentId、agentName、teamName 等）
3. **生命周期管理**：通过 AbortController 支持 teammate 的取消/终止操作
4. **并发安全保证**：确保在多 teammate 并发执行时，身份状态不会相互覆盖

### 与其他身份机制的关系

 teammateContext.ts 提供的身份机制是 teammate.ts 三层身份识别体系中的**最高优先级**：

```
优先级 1: TeammateContext (AsyncLocalStorage) ← 本模块
优先级 2: dynamicTeamContext (teammate.ts)     ← 运行时加入
优先级 3: 环境变量                              ← tmux 进程隔离
```

## 功能点目的

### 1. AsyncLocalStorage 上下文管理

Node.js 的 `AsyncLocalStorage` 允许在异步调用链中存储和访问上下文数据，非常适合管理并发执行的 teammate：

```typescript
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()
```

### 2. TeammateContext 类型定义

完整的 teammate 身份和状态信息：

```typescript
export type TeammateContext = {
  agentId: string           // 完整 agent ID，如 "researcher@my-team"
  agentName: string         // 显示名称，如 "researcher"
  teamName: string          // 所属团队名称
  color?: string            // UI 分配颜色
  planModeRequired: boolean // 是否需要计划模式审批
  parentSessionId: string   // 领导会话 ID（用于 transcript 关联）
  isInProcess: true         // 标识：永远是 true（进程内 teammate）
  abortController: AbortController // 生命周期管理控制器
}
```

### 3. 上下文执行包装

`runWithTeammateContext()` 函数将 teammate 代码包装在 AsyncLocalStorage 上下文中执行：

```typescript
export function runWithTeammateContext<T>(
  context: TeammateContext,
  fn: () => T,
): T {
  return teammateContextStorage.run(context, fn)
}
```

### 4. 快速检测接口

`isInProcessTeammate()` 提供比 `getTeammateContext() !== undefined` 更快的检测方式。

## 具体技术实现

### AsyncLocalStorage 的工作原理

```typescript
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

// 存储上下文
export function runWithTeammateContext<T>(context: TeammateContext, fn: () => T): T {
  return teammateContextStorage.run(context, fn)
}

// 获取当前上下文
export function getTeammateContext(): TeammateContext | undefined {
  return teammateContextStorage.getStore()
}
```

当调用 `runWithTeammateContext(context, fn)` 时：
1. AsyncLocalStorage 将 `context` 与当前的异步资源关联
2. 在 `fn` 及其所有异步后代执行期间，`getTeammateContext()` 返回该 `context`
3. 不同的 teammate 调用拥有独立的 context，互不干扰

### 上下文创建

```typescript
export function createTeammateContext(config: {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
  abortController: AbortController
}): TeammateContext {
  return {
    ...config,
    isInProcess: true,  // 强制标记为进程内 teammate
  }
}
```

**关键设计**：`abortController` 由调用方传入，对于进程内 teammate，这通常是**独立的控制器**（不与父级关联），确保 teammate 在领导查询中断时仍能继续运行。

## 关键代码路径与文件引用

### 内部依赖

| 依赖 | 用途 |
|------|------|
| `async_hooks` (Node.js 内置) | 提供 AsyncLocalStorage |

### 外部调用方

| 文件 | 调用目的 |
|------|----------|
| `teammate.ts` | 重新导出所有功能，作为 teammate 系统的统一入口 |
| `inProcessRunner.ts` | 创建和运行进程内 teammate |
| `spawnInProcess.ts` | 创建 TeammateContext 并启动 teammate |
| `useInboxPoller.ts` | 检测是否为进程内 teammate 以跳过邮箱轮询 |
| `tasks.ts` | 检测 teammate 上下文以获取 teamName |

### 导出函数清单

```typescript
// 类型定义
export type TeammateContext = { ... }

// 核心功能
export function getTeammateContext(): TeammateContext | undefined
export function runWithTeammateContext<T>(context: TeammateContext, fn: () => T): T
export function isInProcessTeammate(): boolean
export function createTeammateContext(config): TeammateContext
```

### 使用示例（来自 spawnInProcess.ts）

```typescript
import { createTeammateContext, runWithTeammateContext } from '../teammateContext.js'

const context = createTeammateContext({
  agentId: formatAgentId(config.name, config.teamName),
  agentName: config.name,
  teamName: config.teamName,
  color: config.color,
  planModeRequired: config.planModeRequired ?? false,
  parentSessionId: config.parentSessionId,
  abortController,  // 独立控制器
})

// 在隔离上下文中运行 teammate
runWithTeammateContext(context, () => {
  // teammate 代码执行期间，getTeammateContext() 返回此 context
  return runInProcessTeammate(config, abortController, taskId)
})
```

## 依赖与外部交互

### 与 teammate.ts 的关系

- `teammate.ts` 重新导出本模块的所有功能，作为统一的 teammate API 入口
- `teammate.ts` 中的函数优先检查 `getTeammateContext()`，然后回退到其他身份源
- 两者形成"优先级链"：AsyncLocalStorage → dynamicTeamContext → 环境变量

### 与进程内 teammate 执行系统的关系

```
spawnInProcess.ts
    ↓ 创建 TeammateContext
inProcessRunner.ts
    ↓ 调用 runWithTeammateContext
Teammate 代码执行
    ↓ 调用 getTeammateContext() 获取身份
teammate.ts 中的工具函数
```

### 与 AppState 的关系

- 进程内 teammate 的任务状态存储在 `AppState.tasks` 中（类型为 `in_process_teammate`）
- `abortController` 允许领导通过 AppState 操作取消 teammate

## 风险、边界与改进建议

### 已知风险

1. **AsyncLocalStorage 限制**：
   - 仅在异步调用链中有效
   - 同步代码块中的 `getTeammateContext()` 可能返回 `undefined`
   - 某些边界情况（如 Promise 微任务）可能有意外行为

2. **内存泄漏风险**：
   - AsyncLocalStorage 会保持对上下文的引用直到异步操作完成
   - 长时间运行的 teammate 可能占用较多内存

3. **调试复杂性**：
   - 异步上下文使得调试和日志追踪更困难
   - 错误堆栈可能不包含完整的上下文信息

### 边界情况

1. **嵌套 teammate**：如果在一个 teammate 中启动另一个 teammate，AsyncLocalStorage 会正确嵌套，但可能导致混淆
2. **同步代码块**：在 `runWithTeammateContext` 的同步代码中调用 `getTeammateContext()` 是安全的，但在某些复杂的异步边界情况下可能返回 `undefined`
3. **异常处理**：如果 `fn` 抛出异常，AsyncLocalStorage 会自动清理上下文

### 改进建议

1. **调试支持**：
   - 添加 `getTeammateContextDebugInfo()` 函数，返回更详细的上下文信息（如创建时间、调用链深度等）
   - 在日志中自动注入 teammate 身份信息

2. **性能监控**：
   - 监控 AsyncLocalStorage 的内存使用情况
   - 添加 teammate 生命周期指标（创建时间、活跃时间等）

3. **类型安全增强**：
   - 考虑使用 branded types 区分不同来源的 agentId
   - 添加编译时检查确保 `isInProcess: true` 不被遗漏

4. **文档和示例**：
   - 提供更多关于 AsyncLocalStorage 使用模式的文档
   - 添加单元测试示例展示并发场景下的正确用法

5. **错误处理**：
   - 在 `getTeammateContext()` 返回 `undefined` 时提供更友好的错误信息
   - 考虑添加 `requireTeammateContext()` 变体，在不存在时抛出错误
