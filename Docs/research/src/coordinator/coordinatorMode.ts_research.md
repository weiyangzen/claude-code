# Research Document: coordinatorMode.ts

## 1. 场景与职责

### 1.1 模块定位

`src/coordinator/coordinatorMode.ts` 是 Claude Code 中 **Coordinator Mode（协调者模式）** 的核心实现模块。该模式是一种特殊的 Agent 架构，将主会话转变为"协调者"角色，专门负责：

- **任务编排**：将复杂任务分解并分派给多个 Worker Agent 并行执行
- **结果汇总**：收集 Worker Agent 的执行结果并综合汇报给用户
- **工作流管理**：管理 Worker Agent 的生命周期（启动、停止、消息传递）

### 1.2 使用场景

| 场景 | 说明 |
|------|------|
| 大规模代码重构 | 将重构任务分派给多个 Worker 并行处理不同模块 |
| 多维度研究 | 同时启动多个 Research Worker 从不同角度调研问题 |
| 复杂验证流程 | 独立 Worker 执行测试验证，与实现 Worker 并行运行 |
| 异步任务管理 | 后台运行长时间任务，协调者继续响应用户交互 |

### 1.3 启用条件

Coordinator Mode 通过以下两个条件同时满足时启用：

1. **Feature Gate**: `feature('COORDINATOR_MODE')` 返回 `true`（通过 GrowthBook 控制）
2. **环境变量**: `CLAUDE_CODE_COORDINATOR_MODE` 设置为 truthy 值

```typescript
// src/coordinator/coordinatorMode.ts:36-41
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

---

## 2. 功能点目的

### 2.1 核心功能概览

| 功能 | 目的 | 关键导出 |
|------|------|----------|
| 模式检测 | 判断当前是否处于协调者模式 | `isCoordinatorMode()` |
| 会话模式匹配 | 恢复会话时同步协调者模式状态 | `matchSessionMode()` |
| Worker 工具上下文 | 向 Worker 提供可用工具信息 | `getCoordinatorUserContext()` |
| 协调者系统提示 | 定义协调者的行为准则和能力 | `getCoordinatorSystemPrompt()` |

### 2.2 功能详细说明

#### 2.2.1 模式检测 (`isCoordinatorMode`)

- **目的**：运行时检测当前会话是否以协调者模式运行
- **调用位置**：
  - `src/utils/toolPool.ts:73` - 工具池过滤
  - `src/utils/systemPrompt.ts:64` - 系统提示构建
  - `src/tools/AgentTool/AgentTool.tsx:252` - Agent 工具调用
  - `src/tools.ts:281` - 工具列表构建

#### 2.2.2 会话模式匹配 (`matchSessionMode`)

- **目的**：处理会话恢复时的模式切换
- **场景**：用户恢复一个之前以协调者模式运行的会话
- **行为**：
  - 检测当前模式与会话存储的模式是否一致
  - 不一致时自动切换环境变量
  - 记录分析事件 `tengu_coordinator_mode_switched`

#### 2.2.3 Worker 工具上下文 (`getCoordinatorUserContext`)

- **目的**：向 Worker Agent 注入可用工具信息
- **注入内容**：
  - Worker 可访问的工具列表（根据 `CLAUDE_CODE_SIMPLE` 模式区分）
  - MCP 服务器提供的工具列表
  - Scratchpad 目录路径（用于跨 Worker 持久化知识）
- **调用链**：`QueryEngine.ts:304-308` -> `getCoordinatorUserContext()`

#### 2.2.4 协调者系统提示 (`getCoordinatorSystemPrompt`)

- **目的**：定义协调者的核心行为准则
- **关键内容**：
  - **角色定义**：协调者是任务编排者，直接回答用户问题而非委托
  - **工具使用**：Agent、SendMessage、TaskStop、PR 活动订阅工具
  - **Worker 管理**：并行启动、结果汇总、错误处理
  - **提示编写规范**：如何编写高质量的 Worker 提示词

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 内部 Worker 工具集合

```typescript
// src/coordinator/coordinatorMode.ts:29-34
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,      // 'TeamCreate'
  TEAM_DELETE_TOOL_NAME,      // 'TeamDelete'
  SEND_MESSAGE_TOOL_NAME,     // 'SendMessage'
  SYNTHETIC_OUTPUT_TOOL_NAME, // 'StructuredOutput'
])
```

这些工具对 Worker 内部使用，不向用户暴露。

#### 3.1.2 协调者允许的工具集合

```typescript
// src/constants/tools.ts:107-112
export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  AGENT_TOOL_NAME,            // 'Agent'
  TASK_STOP_TOOL_NAME,        // 'TaskStop'
  SEND_MESSAGE_TOOL_NAME,     // 'SendMessage'
  SYNTHETIC_OUTPUT_TOOL_NAME, // 'StructuredOutput'
])
```

协调者模式下，主会话只能使用上述工具。

### 3.2 关键流程

#### 3.2.1 工具池过滤流程

```
getTools() / assembleToolPool()
    ↓
mergeAndFilterTools() [src/utils/toolPool.ts:55]
    ↓
检查 isCoordinatorMode()
    ↓
是 → applyCoordinatorToolFilter() → 仅保留 COORDINATOR_MODE_ALLOWED_TOOLS
    ↓
否 → 返回完整工具集
```

#### 3.2.2 系统提示构建流程

```
buildEffectiveSystemPrompt() [src/utils/systemPrompt.ts:41]
    ↓
检查 feature('COORDINATOR_MODE') && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
    ↓
是 → 调用 getCoordinatorSystemPrompt() 返回协调者提示
    ↓
否 → 使用默认系统提示或 Agent 特定提示
```

#### 3.2.3 Worker 上下文注入流程

```
QueryEngine.submitMessage() [src/QueryEngine.ts:302-308]
    ↓
调用 getCoordinatorUserContext(mcpClients, scratchpadDir)
    ↓
构建 workerToolsContext 字符串
    ↓
合并到 userContext
    ↓
传递给 query() 作为上下文
```

### 3.3 协议与规范

#### 3.3.1 Worker 结果通知格式

Worker 完成后，结果以 `<task-notification>` XML 格式发送给协调者：

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{human-readable status summary}</summary>
  <result>{agent's final text response}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

协调者通过识别 `<task-notification>` 标签区分 Worker 结果与普通用户消息。

#### 3.3.2 Worker 工具权限

| 模式 | Worker 可用工具 |
|------|----------------|
| `CLAUDE_CODE_SIMPLE=1` | Bash, FileRead, FileEdit |
| 普通模式 | ASYNC_AGENT_ALLOWED_TOOLS 中定义的标准工具集 |

Worker **禁止**使用的工具（`ALL_AGENT_DISALLOWED_TOOLS`）：
- TaskOutput（防止递归）
- ExitPlanModeV2 / EnterPlanMode（主线程抽象）
- TaskStop（需要主线程任务状态）
- Agent（防止嵌套，除非 USER_TYPE='ant'）
- Workflow（防止递归执行）

### 3.4 命令/环境变量

| 变量/命令 | 说明 |
|-----------|------|
| `CLAUDE_CODE_COORDINATOR_MODE=1` | 启用协调者模式 |
| `CLAUDE_CODE_SIMPLE=1` | 简化模式，Worker 仅使用基础工具 |
| `tengu_scratch` (GrowthBook) | 控制 Scratchpad 功能开关 |
| `tengu_agent_list_attach` (GrowthBook) | 控制 Agent 列表注入方式 |

---

## 4. 关键代码路径与文件引用

### 4.1 核心实现文件

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `src/coordinator/coordinatorMode.ts` | 协调者模式核心逻辑 | `isCoordinatorMode`, `getCoordinatorSystemPrompt`, `getCoordinatorUserContext`, `matchSessionMode` |
| `src/constants/tools.ts` | 工具权限常量定义 | `COORDINATOR_MODE_ALLOWED_TOOLS`, `ASYNC_AGENT_ALLOWED_TOOLS`, `ALL_AGENT_DISALLOWED_TOOLS` |

### 4.2 调用方文件

| 文件 | 调用点 | 用途 |
|------|--------|------|
| `src/utils/toolPool.ts` | `mergeAndFilterTools()` | 根据协调者模式过滤工具池 |
| `src/utils/systemPrompt.ts` | `buildEffectiveSystemPrompt()` | 构建协调者系统提示 |
| `src/QueryEngine.ts` | `submitMessage()` | 注入 Worker 工具上下文 |
| `src/tools/AgentTool/AgentTool.tsx` | `call()`, `prompt()` | 协调者模式下禁用 model 参数 |
| `src/tools.ts` | `getTools()` | 简化模式下包含协调者工具 |
| `src/utils/sessionRestore.ts` | `refreshAgentDefinitionsForModeSwitch()` | 模式切换时刷新 Agent 定义 |
| `src/main.tsx` | 多处 | 条件导入协调者模块 |

### 4.3 依赖文件

| 文件 | 依赖关系 |
|------|----------|
| `src/services/analytics/growthbook.ts` | 调用 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_scratch')` |
| `src/constants/tools.ts` | 导入 `ASYNC_AGENT_ALLOWED_TOOLS` |
| `src/tools/AgentTool/constants.ts` | 导入 `AGENT_TOOL_NAME` |
| `src/tools/BashTool/toolName.js` | 导入 `BASH_TOOL_NAME` |
| `src/tools/FileEditTool/constants.js` | 导入 `FILE_EDIT_TOOL_NAME` |
| `src/tools/FileReadTool/prompt.js` | 导入 `FILE_READ_TOOL_NAME` |
| `src/tools/SendMessageTool/constants.js` | 导入 `SEND_MESSAGE_TOOL_NAME` |
| `src/tools/SyntheticOutputTool/SyntheticOutputTool.js` | 导入 `SYNTHETIC_OUTPUT_TOOL_NAME` |
| `src/tools/TaskStopTool/prompt.js` | 导入 `TASK_STOP_TOOL_NAME` |
| `src/tools/TeamCreateTool/constants.js` | 导入 `TEAM_CREATE_TOOL_NAME` |
| `src/tools/TeamDeleteTool/constants.js` | 导入 `TEAM_DELETE_TOOL_NAME` |
| `src/utils/envUtils.js` | 导入 `isEnvTruthy` |

### 4.4 代码调用图

```
coordinatorMode.ts
├── isCoordinatorMode()
│   ├── src/utils/toolPool.ts:73
│   ├── src/utils/systemPrompt.ts:64
│   ├── src/tools/AgentTool/AgentTool.tsx:252
│   └── src/tools.ts:281
├── getCoordinatorSystemPrompt()
│   └── src/utils/systemPrompt.ts:68-74
├── getCoordinatorUserContext()
│   └── src/QueryEngine.ts:112-117 (via conditional require)
└── matchSessionMode()
    └── src/utils/sessionRestore.ts (via conditional require)
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `bun:bundle` | Feature flag 检查 (`feature('COORDINATOR_MODE')`) |
| GrowthBook | Feature gate 控制 (`tengu_scratch`) |
| 环境变量 | `CLAUDE_CODE_COORDINATOR_MODE`, `CLAUDE_CODE_SIMPLE` |

### 5.2 模块间交互

#### 5.2.1 与 QueryEngine 的交互

```typescript
// src/QueryEngine.ts:110-118
const getCoordinatorUserContext: (...) => { [k: string]: string } = 
  feature('COORDINATOR_MODE')
    ? require('./coordinator/coordinatorMode.js').getCoordinatorUserContext
    : () => ({})
```

使用条件导入避免非协调者模式的代码膨胀。

#### 5.2.2 与 ToolPool 的交互

```typescript
// src/utils/toolPool.ts:72-76
if (feature('COORDINATOR_MODE') && coordinatorModeModule) {
  if (coordinatorModeModule.isCoordinatorMode()) {
    return applyCoordinatorToolFilter(tools)
  }
}
```

#### 5.2.3 与 AgentTool 的交互

协调者模式下禁用 `model` 参数（Worker 使用默认模型）：

```typescript
// src/tools/AgentTool/AgentTool.tsx:252
const model = isCoordinatorMode() ? undefined : modelParam;
```

### 5.3 与 Worker Agent 的交互

协调者通过以下机制与 Worker 交互：

1. **启动 Worker**: 使用 `Agent` 工具调用 `runAgent()`
2. **发送消息**: 使用 `SendMessage` 工具向运行中的 Worker 发送指令
3. **停止 Worker**: 使用 `TaskStop` 工具终止 Worker
4. **接收结果**: 通过 `<task-notification>` XML 消息接收 Worker 完成通知

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 循环依赖风险

```typescript
// coordinatorMode.ts:19-27 注释说明
// Checks the same gate as isScratchpadEnabled() in utils/permissions/filesystem.ts. 
// Duplicated here because importing filesystem.ts creates a circular dependency 
// (filesystem -> permissions -> ... -> coordinatorMode).
```

**缓解措施**：通过 GrowthBook 直接检查 `tengu_scratch`，而非导入 `isScratchpadEnabled()`。

#### 6.1.2 会话恢复模式不匹配

当用户恢复一个以协调者模式运行的会话，但当前环境未启用协调者模式时，可能导致行为不一致。

**缓解措施**：`matchSessionMode()` 函数自动检测并切换模式，记录分析事件。

#### 6.1.3 工具权限泄漏

Worker Agent 理论上可能访问到协调者才能使用的工具。

**缓解措施**：
- `resolveAgentTools()` 在 `runAgent.ts` 中过滤工具
- `ASYNC_AGENT_ALLOWED_TOOLS` 明确定义 Worker 可用工具
- `ALL_AGENT_DISALLOWED_TOOLS` 禁止递归和敏感操作

### 6.2 边界情况

| 边界情况 | 行为 |
|----------|------|
| 协调者模式 + SIMPLE 模式 | Worker 仅获得 Bash/Read/Edit 工具 |
| 协调者模式 + 无 MCP | Worker 上下文不显示 MCP 工具 |
| 协调者模式 + Scratchpad 关闭 | Worker 上下文不显示 Scratchpad 路径 |
| 恢复会话模式切换 | 自动同步模式，刷新 Agent 定义 |
| Worker 嵌套 | 禁止（Agent 工具在 Worker 中禁用） |

### 6.3 改进建议

#### 6.3.1 代码结构优化

1. **提取 Magic String**: 将 `'tengu_scratch'` 等 GrowthBook key 提取为常量
2. **统一条件导入**: 考虑使用 DI 模式替代多处条件 `require()`
3. **类型安全**: 为 `getCoordinatorUserContext` 返回值添加更精确的类型定义

#### 6.3.2 功能增强

1. **Worker 资源限制**: 当前 Worker 没有显式的资源（token/时间）限制
2. **Worker 间通信**: 当前 Worker 之间不能直接通信，需通过协调者中转
3. **动态工具调整**: 运行时动态调整 Worker 可用工具集

#### 6.3.3 可观测性

1. **协调者指标**: 添加 Worker 启动/完成/失败的计数指标
2. **延迟追踪**: 测量 Worker 任务排队和执行延迟
3. **上下文大小**: 监控注入 Worker 上下文的 token 开销

### 6.4 测试建议

| 测试场景 | 验证点 |
|----------|--------|
| 模式切换 | `matchSessionMode` 正确同步环境变量 |
| 工具过滤 | 协调者模式下仅允许指定工具 |
| 上下文注入 | `getCoordinatorUserContext` 正确包含工具列表和 MCP 信息 |
| 系统提示 | `getCoordinatorSystemPrompt` 返回非空字符串且包含关键章节 |
| SIMPLE 模式 | Worker 工具上下文仅包含基础工具 |
| 会话恢复 | 模式切换后 Agent 定义正确刷新 |

---

## 7. 附录

### 7.1 相关 Issue/PR 参考

- 功能实现基于 `COORDINATOR_MODE` feature flag
- Scratchpad 功能通过 `tengu_scratch` gate 控制
- Agent 列表附件化通过 `tengu_agent_list_attach` gate 控制

### 7.2 术语表

| 术语 | 说明 |
|------|------|
| Coordinator | 协调者，主会话角色，负责任务编排 |
| Worker | 工作单元，由协调者启动的 Agent |
| Scratchpad | 跨 Worker 持久化的共享目录 |
| Feature Gate | 通过 GrowthBook 控制的功能开关 |
| SIMPLE Mode | 简化模式，仅启用基础工具 |

---

*文档生成时间: 2026-04-01*
*研究范围: 代码、配置、测试及实现上下文*
*排除范围: README、docs、Docs、markdown 等文档文件*
