# agentContext.ts 深度研究文档

## 场景与职责

`agentContext.ts` 是 Claude Code CLI 中用于**代理上下文管理**的核心基础设施模块。它解决了在多代理并发执行场景下的上下文隔离问题，确保每个代理（subagent/teammate）的异步操作链能够正确追踪自身的身份信息，而不会因为共享状态导致数据混淆。

### 核心场景

1. **Subagent 执行上下文追踪**：当使用 Agent 工具派生子代理时，需要追踪子代理的 ID、父会话 ID、调用请求 ID 等信息
2. **Swarm Teammate 协调**：在多代理团队（swarm）场景中，追踪队友代理的身份、团队名称、颜色标识等
3. **并发安全**：当多个代理被后台化（ctrl+b）时，它们可能在同一进程中并发执行，需要隔离各自的上下文
4. **遥测归因**：为分析事件（analytics events）提供准确的代理身份标识

### 为什么使用 AsyncLocalStorage

模块注释明确解释了设计决策：
- **AppState 的问题**：AppState 是单一共享状态，当多个代理并发执行时会被覆盖，导致代理 A 的事件错误地使用代理 B 的上下文
- **AsyncLocalStorage 的优势**：隔离每个异步执行链，确保并发代理不会相互干扰

---

## 功能点目的

### 1. 上下文类型定义

模块定义了两种主要的代理上下文类型：

| 类型 | 用途 | 关键字段 |
|------|------|----------|
| `SubagentContext` | Agent 工具派生的子代理 | `agentId`, `parentSessionId`, `subagentName`, `isBuiltIn`, `invokingRequestId` |
| `TeammateAgentContext` | Swarm 团队中的队友代理 | `agentId`, `agentName`, `teamName`, `agentColor`, `planModeRequired`, `isTeamLead` |

### 2. 调用边界追踪（Invocation Tracking）

用于追踪代理的调用生命周期：
- `invokingRequestId`: 触发此代理调用的请求 ID
- `invocationKind`: `'spawn'`（首次创建）或 `'resume'`（恢复执行）
- `invocationEmitted`: 标记此调用的边界事件是否已发送到遥测

### 3. 类型守卫函数

- `isSubagentContext()`: 判断上下文是否为子代理类型
- `isTeammateAgentContext()`: 判断上下文是否为队友类型（需同时检查功能开关）

### 4. 遥测安全函数

`getSubagentLogName()`: 返回适合分析日志记录的代理名称
- 内置代理返回实际名称（如 "Explore", "Bash"）
- 用户自定义代理统一返回 `"user-defined"`，避免泄露用户定义的代理名称

---

## 具体技术实现

### 核心数据结构

```typescript
// AsyncLocalStorage 实例 - 存储当前异步执行链的代理上下文
const agentContextStorage = new AsyncLocalStorage<AgentContext>()

// 子代理上下文
interface SubagentContext {
  agentId: string                    // 子代理 UUID
  parentSessionId?: string           // 团队领导的会话 ID
  agentType: 'subagent'              // 类型标识
  subagentName?: string              // 子代理类型名称
  isBuiltIn?: boolean                // 是否为内置代理
  invokingRequestId?: string         // 触发调用的请求 ID
  invocationKind?: 'spawn' | 'resume'
  invocationEmitted?: boolean        // 边界事件是否已发送
}

// 队友代理上下文
interface TeammateAgentContext {
  agentId: string                    // 完整代理 ID (name@team)
  agentName: string                  // 显示名称
  teamName: string                   // 所属团队
  agentColor?: string                // UI 颜色
  planModeRequired: boolean          // 是否需要计划模式
  parentSessionId: string            // 团队领导会话 ID
  isTeamLead: boolean                // 是否为团队领导
  agentType: 'teammate'
  invokingRequestId?: string
  invocationKind?: 'spawn' | 'resume'
  invocationEmitted?: boolean
}
```

### 关键流程

#### 1. 上下文获取
```typescript
export function getAgentContext(): AgentContext | undefined {
  return agentContextStorage.getStore()
}
```
- 使用 `AsyncLocalStorage.getStore()` 获取当前异步链的上下文
- 返回 `undefined` 表示不在代理上下文中（主线程执行）

#### 2. 上下文执行
```typescript
export function runWithAgentContext<T>(context: AgentContext, fn: () => T): T {
  return agentContextStorage.run(context, fn)
}
```
- 使用 `AsyncLocalStorage.run()` 在指定上下文中执行函数
- 该函数及其所有异步操作都能通过 `getAgentContext()` 获取相同上下文

#### 3. 调用边界消费（稀疏边语义）
```typescript
export function consumeInvokingRequestId():
  | { invokingRequestId: string; invocationKind: 'spawn' | 'resume' | undefined }
  | undefined {
  const context = getAgentContext()
  if (!context?.invokingRequestId || context.invocationEmitted) {
    return undefined
  }
  context.invocationEmitted = true
  return {
    invokingRequestId: context.invokingRequestId,
    invocationKind: context.invocationKind,
  }
}
```

**设计要点**：
- 每个调用边界（spawn/resume）只在**第一个**终端 API 事件上报告
- `invocationEmitted` 标志确保每个边界只被消费一次
- 非空值下游标记 spawn/resume 边界，用于遥测中的调用链追踪

---

## 关键代码路径与文件引用

### 调用方（Consumers）

| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | 创建子代理时设置上下文 |
| `src/tools/AgentTool/resumeAgent.ts` | 恢复子代理时更新上下文 |
| `src/utils/swarm/inProcessRunner.ts` | 进程内队友代理执行 |
| `src/services/analytics/metadata.ts` | 获取代理上下文用于遥测 |
| `src/services/api/claude.ts` | API 调用时归因 |
| `src/services/api/logging.ts` | 日志记录代理信息 |
| `src/utils/sessionFileAccessHooks.ts` | 会话文件访问钩子 |
| `src/tools/SkillTool/SkillTool.ts` | 技能工具执行 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 处理斜杠命令 |
| `src/tasks/LocalMainSessionTask.ts` | 本地主会话任务 |

### 依赖方

| 文件 | 用途 |
|------|------|
| `src/utils/agentSwarmsEnabled.ts` | 检查 swarm 功能是否启用（类型守卫使用） |
| `src/services/analytics/index.ts` | 分析元数据类型 |

---

## 依赖与外部交互

### 直接依赖

```typescript
import { AsyncLocalStorage } from 'async_hooks'
import type { AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS } from '../services/analytics/index.js'
import { isAgentSwarmsEnabled } from './agentSwarmsEnabled.js'
```

### Node.js 原生模块

- **`async_hooks.AsyncLocalStorage`**: Node.js 提供的异步上下文存储机制，用于在异步调用链中保持上下文

### 项目内部依赖

| 模块 | 关系 |
|------|------|
| `analytics/index.ts` | 类型依赖 - 遥测元数据类型 |
| `agentSwarmsEnabled.ts` | 运行时依赖 - 功能开关检查 |

---

## 风险、边界与改进建议

### 已知风险

1. **跨进程边界丢失**
   - 注释明确指出：对于 tmux/iTerm2 中的 swarm 队友（跨进程），需要使用环境变量（`CLAUDE_CODE_AGENT_ID`, `CLAUDE_CODE_PARENT_SESSION_ID`）而非 AsyncLocalStorage
   - 风险：进程间通信时上下文无法自动传递

2. **内存泄漏风险**
   - AsyncLocalStorage 会在异步资源（如 Promise、回调）中保持对上下文的引用
   - 如果长时间运行的异步操作持有上下文，可能导致内存无法及时释放

3. **并发覆盖风险**
   - 虽然 AsyncLocalStorage 解决了并发问题，但如果代码错误地直接修改上下文对象（而非通过 run），仍可能导致问题

### 边界情况

1. **主线程执行**
   - `getAgentContext()` 返回 `undefined` 表示不在代理上下文中
   - 调用方需要正确处理这种情况

2. **嵌套代理调用**
   - 支持嵌套子代理（subagent 调用 subagent）
   - `invokingRequestId` 始终指向**直接**调用者，而非根代理
   - `session_id` 已经捆绑了整个调用树

3. **恢复场景**
   - 代理恢复时 `invocationKind` 设为 `'resume'`
   - `invokingRequestId` 在每次恢复时更新

### 改进建议

1. **类型安全增强**
   ```typescript
   // 建议：添加更严格的类型守卫，确保在编译期捕获类型错误
   export function assertSubagentContext(ctx: unknown): asserts ctx is SubagentContext {
     if (!isSubagentContext(ctx)) {
       throw new Error('Expected subagent context')
     }
   }
   ```

2. **上下文不可变性**
   - 当前上下文对象是可变的（如 `invocationEmitted` 被修改）
   - 建议：使用不可变模式，每次更新创建新上下文对象

3. **调试支持**
   - 建议添加调试工具函数，用于打印当前上下文状态
   - 有助于排查上下文相关问题

4. **文档完善**
   - 添加更多使用示例，特别是嵌套代理和恢复场景
   - 明确说明与进程外代理的交互方式

5. **性能监控**
   - 考虑添加钩子监控上下文切换频率
   - 有助于发现异常频繁的上下文切换
