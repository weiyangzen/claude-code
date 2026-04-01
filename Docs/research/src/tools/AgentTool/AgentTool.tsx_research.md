# AgentTool.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位

`AgentTool` 是 Claude Code 中用于**启动和管理子代理(subagent)**的核心工具。它是整个多代理系统的入口点，负责：

1. **代理调度**：根据输入参数选择合适的代理类型（内置代理、自定义代理、Fork代理）
2. **执行模式管理**：支持同步执行、异步后台执行、远程执行等多种模式
3. **生命周期管理**：代理的创建、运行、监控、终止和清理
4. **团队协作**：支持多代理团队(teammate)的创建和管理

### 1.2 使用场景

| 场景 | 描述 |
|------|------|
| **简单任务委托** | 调用通用代理执行搜索、代码分析等任务 |
| **专业代理调用** | 调用特定类型的代理（如 Explore、Plan、Verification 等） |
| **Fork 子代理** | 当 `subagent_type` 省略时，创建继承父上下文的 Fork 子代理 |
| **后台任务** | 通过 `run_in_background` 启动长时间运行的后台代理 |
| **团队多代理** | 通过 `name` + `team_name` 创建 teammate 代理 |
| **隔离执行** | 通过 `isolation: worktree` 在独立 git worktree 中运行 |
| **远程执行** | 通过 `isolation: remote` 在远程 CCR 环境中运行 (ant-only) |

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                      Claude Code Main                        │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   REPL.tsx   │───▶│  QueryEngine │───▶│   AgentTool  │   │
│  └──────────────┘    └──────────────┘    └──────┬───────┘   │
│                                                  │          │
│                              ┌───────────────────┼──────┐   │
│                              ▼                   ▼       │   │
│                         ┌────────┐          ┌─────────┐  │   │
│                         │runAgent│          │Teammate │  │   │
│                         │        │          │ Spawn   │  │   │
│                         └───┬────┘          └─────────┘  │   │
│                             │                            │   │
│                    ┌────────┴────────┐                   │   │
│                    ▼                 ▼                   │   │
│              ┌──────────┐      ┌──────────┐              │   │
│              │ Sync Exec│      │Async Exec│              │   │
│              └──────────┘      └──────────┘              │   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 功能点目的

### 2.1 主要功能模块

#### 2.1.1 代理类型选择与路由

- **内置代理**：`general-purpose`, `Explore`, `Plan`, `verification`, `claudeCodeGuide`, `statusline-setup`
- **自定义代理**：从用户设置、项目设置、策略设置加载的代理
- **插件代理**：从插件加载的代理
- **Fork 代理**：当 `subagent_type` 省略且 `FORK_SUBAGENT` 功能开启时使用

#### 2.1.2 执行模式

| 模式 | 触发条件 | 特点 |
|------|----------|------|
| **同步执行** | `run_in_background: false` (默认) | 阻塞父代理，实时返回结果 |
| **异步后台** | `run_in_background: true` 或 `selectedAgent.background: true` | 非阻塞，通过通知返回结果 |
| **强制异步** | `isForkSubagentEnabled()` 返回 true | 所有代理都异步执行 |
| **协调器模式** | `COORDINATOR_MODE` 开启 | 强制异步，用于协调器工作流 |
| **远程执行** | `isolation: remote` (ant-only) | 在 CCR 远程环境执行 |

#### 2.1.3 隔离模式

- **Worktree 隔离**：创建临时 git worktree，代理在隔离的代码副本上工作
- **远程隔离**：在远程 CCR 沙箱中执行（仅 ant 内部版本）
- **CWD 覆盖**：通过 `cwd` 参数指定工作目录

#### 2.1.4 团队多代理 (Teammate)

当提供 `name` 和 `team_name` 参数时，创建 teammate 代理：
- **Out-of-process**：通过 tmux/iTerm2 创建独立进程
- **In-process**：在同一 Node.js 进程中使用 AsyncLocalStorage 隔离

### 2.2 关键数据结构

#### 2.2.1 输入参数 (AgentToolInput)

```typescript
type AgentToolInput = {
  description: string;           // 任务描述（3-5词）
  prompt: string;               // 任务详细提示
  subagent_type?: string;       // 代理类型（省略则触发 Fork）
  model?: 'sonnet' | 'opus' | 'haiku';  // 模型覆盖
  run_in_background?: boolean;  // 是否后台运行
  name?: string;                // teammate 名称
  team_name?: string;           // 团队名称
  mode?: PermissionMode;        // 权限模式
  isolation?: 'worktree' | 'remote';  // 隔离模式
  cwd?: string;                 // 工作目录覆盖
}
```

#### 2.2.2 输出类型 (InternalOutput)

```typescript
type Output = 
  | { status: 'completed'; agentId; content; totalTokens; totalToolUseCount; totalDurationMs; usage }
  | { status: 'async_launched'; agentId; description; prompt; outputFile; canReadOutputFile }
  | TeammateSpawnedOutput  // teammate 创建成功
  | RemoteLaunchedOutput;  // 远程代理启动成功
```

---

## 3. 具体技术实现

### 3.1 核心调用流程 (call 方法)

```
call(input, toolUseContext, canUseTool, assistantMessage, onProgress)
│
├─▶ 1. 参数解析与验证
│   ├─ 解析 model, run_in_background, name, team_name 等参数
│   ├─ 检查 Agent Swarms 权限 (isAgentSwarmsEnabled)
│   └─ 验证 teammate 限制（teammate 不能 spawn teammate）
│
├─▶ 2. 代理类型选择
│   ├─ 如果 team_name + name → spawnTeammate()
│   ├─ 如果 subagent_type 省略且 fork 开启 → FORK_AGENT
│   └─ 否则查找匹配的 AgentDefinition
│
├─▶ 3. MCP 服务器检查
│   └─ 等待所需 MCP 服务器连接，检查工具可用性
│
├─▶ 4. 执行模式决策
│   ├─ shouldRunAsync = run_in_background || background || coordinator || forceAsync
│   └─ 分支：同步执行 vs 异步执行
│
├─▶ 5. Worktree 隔离设置（如需要）
│   └─ createAgentWorktree(slug)
│
├─▶ 6. 系统提示词构建
│   ├─ Fork 路径：继承父系统提示词
│   └─ 普通路径：构建代理特定系统提示词
│
└─▶ 7. 执行代理
    ├─ 异步：registerAsyncAgent() → runAsyncAgentLifecycle()
    └─ 同步：runAgent() 迭代器循环
```

### 3.2 Fork 子代理机制

#### 3.2.1 Fork 路径触发条件

```typescript
// AgentTool.tsx:322
const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType);
const isForkPath = effectiveType === undefined;
```

#### 3.2.2 Fork 消息构建

Fork 子代理继承父代理的完整对话上下文，通过 `buildForkedMessages()` 实现：

```typescript
// forkSubagent.ts:107-169
export function buildForkedMessages(directive: string, assistantMessage: AssistantMessage): MessageType[] {
  // 1. 克隆父代理的完整 assistant 消息（包含所有 tool_use 块）
  // 2. 为每个 tool_use 创建占位 tool_result
  // 3. 追加 per-child directive
  return [fullAssistantMessage, toolResultMessage];
}
```

**关键设计**：所有 Fork 子代理共享相同的 API 请求前缀（到 directive 之前），实现 prompt cache 共享。

#### 3.2.3 Fork 指令格式

```xml
<fork_boilerplate>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — that's for the parent.
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly: Bash, Read, Write, etc.
...
</fork_boilerplate>

FORK DIRECTIVE: {用户提供的 prompt}
```

### 3.3 异步代理生命周期

```
registerAsyncAgent()
    │
    ▼
runAsyncAgentLifecycle()
    │
    ├─▶ 创建 ProgressTracker
    ├─▶ 启动 AgentSummarization（如启用）
    │
    ├─▶ for await (message of runAgent())
    │       ├─ 收集消息
    │       ├─ 更新进度 (updateProgressFromMessage)
    │       └─ 发射进度事件 (emitTaskProgress)
    │
    ├─▶ finalizeAgentTool() → 提取结果
    ├─▶ completeAsyncAgent() → 标记任务完成
    ├─▶ classifyHandoffIfNeeded() → 安全检查
    └─▶ enqueueAgentNotification() → 发送通知
```

### 3.4 同步代理执行与后台化

同步代理支持运行中转为后台执行：

```
同步执行循环
    │
    ├─▶ 注册前景任务 (registerAgentForeground)
    │       └─ 设置 backgroundSignal Promise
    │
    ├─▶ while (true) 消息循环
    │       ├─ Promise.race([nextMessage, backgroundSignal])
    │       ├─ 如果 backgroundSignal 触发
    │       │       ├─ 停止前景迭代器
    │       │       ├─ 启动后台执行 (runAgent with isAsync: true)
    │       │       └─ 立即返回 async_launched 结果
    │       └─ 否则处理消息
    │
    └─▶ 完成或错误处理
```

### 3.5 工具过滤与权限

#### 3.5.1 代理工具过滤

```typescript
// agentToolUtils.ts:70-116
export function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync = false,
  permissionMode,
}): Tools {
  return tools.filter(tool => {
    // MCP 工具始终允许
    if (tool.name.startsWith('mcp__')) return true;
    
    // Plan 模式允许 ExitPlanModeV2Tool
    if (toolMatchesName(tool, EXIT_PLAN_MODE_V2_TOOL_NAME) && permissionMode === 'plan') {
      return true;
    }
    
    // 全局禁用工具检查
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;
    
    // 自定义代理额外限制
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;
    
    // 异步代理工具限制
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false;
    
    return true;
  });
}
```

#### 3.5.2 工具常量定义

```typescript
// constants/tools.ts
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  'Agent',           // 子代理不能再 spawn 子代理（Fork 路径除外）
  'Task',            // 旧版名称
  'TodoWrite',       // 子代理不能操作父的 todo
  'TeamCreate',      // 子代理不能创建团队
  'TeamDelete',      // 子代理不能删除团队
]);

export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  'Bash', 'Read', 'Edit', 'Glob', 'Grep', 
  'FileRead', 'FileEdit', 'FileWrite',
  'NotebookEdit', 'WebFetch', 'WebSearch',
  // ... 允许后台使用的工具
]);
```

### 3.6 远程代理执行 (ant-only)

```typescript
// AgentTool.tsx:435-482
if ("external" === 'ant' && effectiveIsolation === 'remote') {
  // 1. 检查远程执行资格
  const eligibility = await checkRemoteAgentEligibility();
  
  // 2. 创建远程会话
  const session = await teleportToRemote({ initialMessage: prompt, description });
  
  // 3. 注册远程任务
  const { taskId, sessionId } = registerRemoteAgentTask({ ... });
  
  // 4. 返回 remote_launched 结果
  return { data: { status: 'remote_launched', taskId, sessionUrl, ... } };
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件结构

```
src/tools/AgentTool/
├── AgentTool.tsx          # 主工具实现，包含 call() 方法
├── UI.tsx                 # React UI 渲染组件
├── runAgent.ts            # 代理执行引擎
├── agentToolUtils.ts      # 工具函数（finalizeAgentTool, filterToolsForAgent 等）
├── forkSubagent.ts        # Fork 子代理逻辑
├── loadAgentsDir.ts       # 代理定义加载
├── prompt.ts              # 工具提示词生成
├── constants.ts           # 常量定义
├── agentColorManager.ts   # 代理颜色管理
├── agentMemory.ts         # 代理内存功能
├── agentMemorySnapshot.ts # 内存快照
├── resumeAgent.ts         # 代理恢复逻辑
└── built-in/              # 内置代理定义
    ├── generalPurposeAgent.ts
    ├── exploreAgent.ts
    ├── planAgent.ts
    ├── verificationAgent.ts
    ├── claudeCodeGuideAgent.ts
    └── statuslineSetup.ts
```

### 4.2 关键调用链

#### 4.2.1 代理启动调用链

```
AgentTool.call()                          [AgentTool.tsx:239]
    ├── spawnTeammate()                   [spawnMultiAgent.ts:1028]
    │       ├── handleSpawnSplitPane()    [spawnMultiAgent.ts:305]
    │       ├── handleSpawnSeparateWindow() [spawnMultiAgent.ts:545]
    │       └── handleSpawnInProcess()    [spawnMultiAgent.ts:840]
    │
    ├── runAgent()                        [runAgent.ts:248]
    │       ├── initializeAgentMcpServers() [runAgent.ts:95]
    │       ├── buildAgentSystemPrompt()  [runAgent.ts:906]
    │       ├── createSubagentContext()   [forkedAgent.ts]
    │       └── query()                   [query.ts]
    │
    └── runAsyncAgentLifecycle()          [agentToolUtils.ts:508]
```

#### 4.2.2 任务状态管理调用链

```
registerAsyncAgent()                      [LocalAgentTask.tsx:466]
registerAgentForeground()                 [LocalAgentTask.tsx:526]
backgroundAgentTask()                     [LocalAgentTask.tsx:620]
completeAgentTask()                       [LocalAgentTask.tsx:412]
failAgentTask()                           [LocalAgentTask.tsx:437]
killAsyncAgent()                          [LocalAgentTask.tsx:281]
```

### 4.3 关键代码片段

#### 4.3.1 代理选择逻辑

```typescript
// AgentTool.tsx:318-356
// Fork subagent experiment routing:
const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType);
const isForkPath = effectiveType === undefined;
let selectedAgent: AgentDefinition;

if (isForkPath) {
  // 递归 Fork 防护
  if (toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}` || 
      isInForkChild(toolUseContext.messages)) {
    throw new Error('Fork is not available inside a forked worker.');
  }
  selectedAgent = FORK_AGENT;
} else {
  // 查找匹配的代理定义
  const agents = filterDeniedAgents(...);
  const found = agents.find(agent => agent.agentType === effectiveType);
  selectedAgent = found;
}
```

#### 4.3.2 异步执行决策

```typescript
// AgentTool.tsx:556-567
const forceAsync = isForkSubagentEnabled();
const assistantForceAsync = feature('KAIROS') ? appState.kairosEnabled : false;
const shouldRunAsync = (
  run_in_background === true || 
  selectedAgent.background === true || 
  isCoordinator || 
  forceAsync || 
  assistantForceAsync || 
  (proactiveModule?.isProactiveActive() ?? false)
) && !isBackgroundTasksDisabled;
```

#### 4.3.3 同步转后台逻辑

```typescript
// AgentTool.tsx:897-1051
if (raceResult.type === 'background' && foregroundTaskId) {
  wasBackgrounded = true;
  stopForegroundSummarization?.();
  
  // 启动后台执行
  void runWithAgentContext(syncAgentContext, async () => {
    // 清理前景迭代器
    await Promise.race([agentIterator.return(undefined), sleep(1000)]);
    
    // 后台执行
    for await (const msg of runAgent({ ...runAgentParams, isAsync: true })) {
      agentMessages.push(msg);
      updateProgressFromMessage(tracker, msg, ...);
      updateAsyncAgentProgress(backgroundedTaskId, getProgressUpdate(tracker), rootSetAppState);
    }
    
    // 完成处理
    const agentResult = finalizeAgentTool(agentMessages, backgroundedTaskId, metadata);
    completeAsyncAgent(agentResult, rootSetAppState);
    enqueueAgentNotification({ ... });
  });
  
  // 立即返回 async_launched
  return { data: { status: 'async_launched', agentId: backgroundedTaskId, ... } };
}
```

---

## 5. 依赖与外部交互

### 5.1 主要依赖模块

| 模块 | 用途 |
|------|------|
| `src/Tool.ts` | Tool 类型定义、buildTool 工厂函数 |
| `src/tools.ts` | 工具池组装 (assembleToolPool) |
| `src/query.ts` | 核心查询循环 |
| `src/state/AppState.ts` | 全局状态管理 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 本地代理任务管理 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.ts` | 远程代理任务管理 |
| `src/utils/forkedAgent.ts` | 子代理上下文创建 |
| `src/utils/worktree.ts` | Git worktree 管理 |
| `src/utils/swarm/` | 多代理团队管理 |
| `src/services/AgentSummary/agentSummary.ts` | 代理进度摘要 |

### 5.2 外部服务交互

| 服务 | 交互方式 | 用途 |
|------|----------|------|
| MCP Servers | `initializeAgentMcpServers()` | 代理特定的 MCP 工具 |
| GrowthBook | `getFeatureValue_CACHED_MAY_BE_STALE()` | 功能开关 |
| Analytics | `logEvent()` | 事件追踪 |
| CCR Remote | `teleportToRemote()` | 远程代理执行 |
| tmux/iTerm2 | `spawnMultiAgent.ts` | teammate 进程管理 |

### 5.3 环境变量依赖

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务 |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | 启用自动后台化 |
| `CLAUDE_CODE_COORDINATOR_MODE` | 协调器模式 |
| `CLAUDE_CODE_SIMPLE` | 简化模式（仅内置代理） |
| `USER_TYPE=ant` | 启用 ant-only 功能 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 递归 Fork 风险

**问题**：Fork 子代理理论上可以继续 Fork，导致无限递归。

**现有防护**：
- 通过 `querySource` 检查（AgentTool.tsx:332）
- 通过消息历史扫描 `isInForkChild()`（forkSubagent.ts:78）

**风险**：autocompact 可能重写消息，导致 message-scan  fallback 失效。

#### 6.1.2 后台任务泄漏

**问题**：异步代理可能因异常或取消而未能正确清理。

**现有防护**：
- `registerCleanup()` 注册清理回调
- `finally` 块中的 `clearInvokedSkillsForAgent`, `clearDumpState`

**风险**：MCP 连接、worktree 等可能泄漏。

#### 6.1.3 Worktree 清理风险

**问题**：Worktree 清理依赖 git 命令，如果 git 挂起会阻塞。

**现有防护**：
- 状态先更新，清理后执行（AgentTool.tsx:957-977）
- `cleanupWorktreeIfNeeded()` 在 finally 中调用

**风险**：极端情况下 worktree 可能残留。

#### 6.1.4 工具权限绕过

**问题**：子代理可能通过特定方式绕过工具限制。

**现有防护**：
- `filterToolsForAgent()` 过滤工具列表
- `resolveAgentTools()` 解析并验证工具

**风险**：Fork 路径使用 `useExactTools: true`，继承父工具池。

### 6.2 边界条件

| 边界 | 行为 |
|------|------|
| `subagent_type` 不存在 | 如果 Fork 开启 → Fork 路径；否则 → general-purpose |
| `name` + `team_name` | 创建 teammate，忽略其他代理逻辑 |
| `isolation: worktree` + `cwd` | `cwd` 优先（互斥） |
| `run_in_background: true` + teammate | In-process teammate 禁止后台代理 |
| MCP 服务器未就绪 | 等待最多 30 秒，超时则报错 |
| 代理执行超时 | 通过 `maxTurns` 限制，默认 200 |

### 6.3 改进建议

#### 6.3.1 架构层面

1. **统一执行模型**：当前同步/异步/远程执行有重复代码，可抽象为统一的 `AgentExecutor` 接口
2. **状态机重构**：代理生命周期状态转换复杂，建议引入显式状态机
3. **错误分类**：当前错误处理较粗粒度，建议区分可恢复/不可恢复错误

#### 6.3.2 性能优化

1. **Prompt Cache 优化**：Fork 路径的 cache 共享机制可扩展到更多场景
2. **工具过滤缓存**：`filterToolsForAgent` 可缓存结果避免重复计算
3. **MCP 连接池**：代理特定的 MCP 连接可复用而非每次都新建

#### 6.3.3 可观测性

1. **执行追踪**：增加 OpenTelemetry 风格的 span 追踪
2. **性能指标**：收集代理启动时间、工具调用延迟等指标
3. **调试工具**：提供 `/debug agents` 命令查看代理状态

#### 6.3.4 安全性

1. **沙箱强化**：Worktree 隔离可进一步增强（如只读挂载）
2. **权限审计**：记录子代理的工具调用权限变更
3. **资源限制**：增加代理的 CPU/内存资源限制

---

## 7. 附录

### 7.1 相关 Issue 参考

- gh-20236: 任务状态转换不应被清理操作阻塞
- gh-31069: 'inherit' 模型值被错误传递

### 7.2 测试要点

1. **Fork 路径**：验证 cache 共享、递归防护
2. **后台化**：验证同步→异步转换、通知发送
3. **Worktree**：验证创建、清理、变更检测
4. **Teammate**：验证 in-process 和 out-of-process 模式
5. **MCP**：验证代理特定的 MCP 服务器生命周期

### 7.3 文档更新记录

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-01 | 1.0 | 初始研究文档 |
