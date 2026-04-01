# teammate.ts 研究文档

## 场景与职责

`teammate.ts` 是 Claude Code 多智能体协作系统（Agent Swarm）的核心身份识别与状态管理模块。它负责：

1. ** teammate 身份识别**：判断当前 Claude 实例是否以 teammate 身份运行在 swarm 中
2. ** 多层级身份解析**：支持三种 teammate 运行模式的身份识别优先级
3. ** 团队领导检测**：判断当前会话是否为团队领导（team lead）
4. ** 进程内 teammate 状态管理**：检测和等待进程内 teammate 的空闲状态

该模块是 teammate 系统的入口点，被 `teammateContext.ts`（进程内 teammate）、mailbox 系统、权限同步、UI 组件等广泛依赖。

## 功能点目的

### 1. 多层级身份识别系统

 teammate.ts 实现了三层身份识别机制，按优先级排序：

| 优先级 | 机制 | 适用场景 |
|--------|------|----------|
| 1 | AsyncLocalStorage (TeammateContext) | 进程内 teammate（in-process） |
| 2 | dynamicTeamContext | 运行时加入团队的 teammate |
| 3 | 环境变量 (CLI args) | tmux 进程隔离的 teammate |

### 2. 核心身份属性获取

- `getAgentId()`: 获取 agent ID（格式：`agentName@teamName`）
- `getAgentName()`: 获取 agent 显示名称
- `getTeamName()`: 获取团队名称（支持 leader 传入 teamContext）
- `getParentSessionId()`: 获取父会话 ID（用于 transcript 关联）
- `getTeammateColor()`: 获取分配的 UI 颜色
- `isPlanModeRequired()`: 检查是否需要计划模式审批

### 3. 团队领导检测

`isTeamLead()` 函数通过以下逻辑判断是否为团队领导：
1. 检查 teamContext 是否存在且包含 leadAgentId
2. 比较当前 agent ID 与 leadAgentId
3. 向后兼容：无 agent ID 但有 teamContext 的原始会话视为领导

### 4. 进程内 teammate 状态管理

- `hasActiveInProcessTeammates()`: 检查是否有运行的进程内 teammate
- `hasWorkingInProcessTeammates()`: 检查是否有正在工作的 teammate（非空闲）
- `waitForTeammatesToBecomeIdle()`: 等待所有工作中的 teammate 变为空闲状态

## 具体技术实现

### 关键数据结构

```typescript
// 动态团队上下文（运行时加入团队）
let dynamicTeamContext: {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId?: string
} | null = null
```

### 身份解析优先级实现

```typescript
export function getAgentId(): string | undefined {
  // 1. 首先检查 AsyncLocalStorage（进程内 teammate）
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.agentId
  // 2. 回退到 dynamicTeamContext（tmux teammate）
  return dynamicTeamContext?.agentId
}
```

### 等待 teammate 空闲的实现

```typescript
export function waitForTeammatesToBecomeIdle(
  setAppState: (f: (prev: AppState) => AppState) => void,
  appState: AppState,
): Promise<void> {
  // 1. 收集所有工作中的 teammate
  const workingTaskIds: string[] = []
  for (const [taskId, task] of Object.entries(appState.tasks)) {
    if (task.type === 'in_process_teammate' && task.status === 'running' && !task.isIdle) {
      workingTaskIds.push(taskId)
    }
  }
  
  // 2. 创建 Promise，通过 onIdleCallbacks 机制解析
  return new Promise<void>(resolve => {
    let remaining = workingTaskIds.length
    const onIdle = (): void => {
      remaining--
      if (remaining === 0) resolve()
    }
    
    // 3. 注册回调到每个 working teammate
    setAppState(prev => {
      for (const taskId of workingTaskIds) {
        const task = newTasks[taskId]
        if (task.isIdle) {
          onIdle() // 已空闲，立即调用
        } else {
          newTasks[taskId] = {
            ...task,
            onIdleCallbacks: [...(task.onIdleCallbacks ?? []), onIdle],
          }
        }
      }
      return { ...prev, tasks: newTasks }
    })
  })
}
```

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `teammateContext.ts` | 进程内 teammate 的 AsyncLocalStorage 上下文 |
| `AppState.ts` | 应用状态类型定义（Task 类型） |
| `envUtils.ts` | `isEnvTruthy()` 环境变量解析 |

### 外部调用方

| 文件 | 调用目的 |
|------|----------|
| `useInboxPoller.ts` | 获取 agentName 进行邮箱轮询 |
| `teammateMailbox.ts` | 获取 teamName、agentName、color |
| `permissionSync.ts` | 获取 agent ID 和名称进行权限请求 |
| `inProcessTeammateHelpers.ts` | 检测是否为进程内 teammate |
| `TeamCreateTool.ts` | 设置 dynamicTeamContext |
| `AgentTool.tsx` | 判断是否为 teammate 以调整行为 |
| `useSwarmPermissionPoller.ts` | 检测 teammate 身份 |
| `print.ts` | 检测是否有活动的进程内 teammate |

### 导出函数清单

```typescript
// 从 teammateContext.ts 重新导出
export { createTeammateContext, getTeammateContext, isInProcessTeammate, runWithTeammateContext, type TeammateContext }

// 本模块定义
export function getParentSessionId(): string | undefined
export function setDynamicTeamContext(context): void
export function clearDynamicTeamContext(): void
export function getDynamicTeamContext(): typeof dynamicTeamContext
export function getAgentId(): string | undefined
export function getAgentName(): string | undefined
export function getTeamName(teamContext?): string | undefined
export function isTeammate(): boolean
export function getTeammateColor(): string | undefined
export function isPlanModeRequired(): boolean
export function isTeamLead(teamContext): boolean
export function hasActiveInProcessTeammates(appState): boolean
export function hasWorkingInProcessTeammates(appState): boolean
export function waitForTeammatesToBecomeIdle(setAppState, appState): Promise<void>
```

## 依赖与外部交互

### 环境变量

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | 全局计划模式要求配置 |
| `CLAUDE_CODE_AGENT_ID` | tmux teammate 的 agent ID |
| `CLAUDE_CODE_AGENT_NAME` | tmux teammate 的 agent 名称 |
| `CLAUDE_CODE_TEAM_NAME` | 团队名称 |

### 与 teammateContext.ts 的关系

- `teammateContext.ts` 提供进程内 teammate 的 AsyncLocalStorage 实现
- `teammate.ts` 将其作为最高优先级身份源，同时提供统一的 API 接口
- 两者共同构成完整的 teammate 身份识别体系

### 与 AppState 的交互

- 通过 `AppState.tasks` 访问进程内 teammate 任务状态
- 使用 `setAppState` 注册 onIdle 回调
- 依赖 Task 类型定义中的 `in_process_teammate` 类型

## 风险、边界与改进建议

### 已知风险

1. **并发竞态条件**：`waitForTeammatesToBecomeIdle` 在注册回调和检查 isIdle 状态之间存在竞态窗口，代码已通过双重检查缓解

2. **dynamicTeamContext 全局状态**：模块级变量在并发场景下可能被覆盖，但进程内 teammate 使用 AsyncLocalStorage 避免了此问题

3. **向后兼容复杂性**：`isTeamLead` 中的无 agent ID 回退逻辑增加了理解成本

### 边界情况

1. **混合模式**：当同时存在 tmux teammate 和进程内 teammate 时，身份识别可能产生歧义
2. **团队切换**：`dynamicTeamContext` 需要显式清除，否则可能导致身份残留
3. **Leader 的 teamName 获取**：Leader 需要通过传入 teamContext 获取 teamName，与其他身份属性获取方式不一致

### 改进建议

1. **统一身份源**：考虑将三层身份识别合并为单一的身份服务，减少调用方的理解成本

2. **类型安全增强**：`dynamicTeamContext` 的类型定义可以提取为具名接口，便于复用

3. **回调清理机制**：`waitForTeammatesToBecomeIdle` 中的回调在 teammate 被终止时可能永远不会被调用，需要超时或清理机制

4. **文档完善**：建议添加更多关于三种身份识别机制何时使用的场景说明

5. **测试覆盖**：边界情况（如 teamContext 变化时的行为）需要更全面的单元测试
