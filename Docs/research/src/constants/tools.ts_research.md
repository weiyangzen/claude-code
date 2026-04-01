# tools.ts 深度研究文档

## 场景与职责

`src/constants/tools.ts` 是 Claude Code CLI 的工具权限和可用性控制模块，定义了不同执行上下文（Agent、Coordinator、Async Agent 等）中允许或禁止使用的工具集合。该模块是 Agent 系统安全模型的核心组成部分，确保子 Agent 不会执行可能破坏系统状态或导致递归的操作。

**主要使用场景：**
1. **Agent 工具过滤**：决定子 Agent 可以使用哪些工具
2. **Coordinator 模式限制**：协调器模式只允许输出和 Agent 管理工具
3. **Async Agent 工具控制**：异步执行的 Agent 有专门的工具白名单
4. **进程内队友工具权限**：特殊的工具集用于进程内队友通信

**安全目标：**
- 防止递归调用（Agent 调用 Agent 调用 Agent...）
- 隔离子 Agent 的执行能力
- 区分不同运行模式下的权限级别

## 功能点目的

### 1. 通用 Agent 禁止工具 (`ALL_AGENT_DISALLOWED_TOOLS`)

所有 Agent（包括内置和自定义）都禁止使用的工具：

| 工具 | 禁止原因 |
|------|----------|
| `TaskOutputTool` | 防止递归，子 Agent 不应直接操作父任务输出 |
| `ExitPlanModeV2Tool` | Plan 模式是主线程抽象 |
| `EnterPlanModeTool` | Plan 模式是主线程抽象 |
| `AgentTool` | 防止递归（但 `ant` 用户可以启用嵌套 Agent） |
| `AskUserQuestionTool` | 子 Agent 不应直接与用户交互 |
| `TaskStopTool` | 需要访问主线程任务状态 |
| `WorkflowTool` | 防止工作流脚本内递归执行（当 WORKFLOW_SCRIPTS 特性启用时） |

**特殊规则**：当 `USER_TYPE === 'ant'` 时，允许使用 `AgentTool` 以支持嵌套 Agent。

### 2. 自定义 Agent 额外禁止工具 (`CUSTOM_AGENT_DISALLOWED_TOOLS`)

在通用禁止基础上，自定义 Agent 额外禁止的工具。当前与通用集合相同，为未来扩展预留。

### 3. 异步 Agent 允许工具 (`ASYNC_AGENT_ALLOWED_TOOLS`)

异步 Agent 可以使用的工具白名单：

**文件操作**：`FileReadTool`, `FileEditTool`, `FileWriteTool`, `GlobTool`, `GrepTool`
**Web 操作**：`WebSearchTool`, `WebFetchTool`
**Shell**：所有 Shell 工具（`SHELL_TOOL_NAMES`）
**Notebook**：`NotebookEditTool`
**任务管理**：`TodoWriteTool`
**技能**：`SkillTool`
**其他**：`SyntheticOutputTool`, `ToolSearchTool`, `EnterWorktreeTool`, `ExitWorktreeTool`

**明确禁止**（注释中说明）：
- `AgentTool`: 防止递归
- `TaskOutputTool`: 防止递归
- `ExitPlanModeTool`: Plan 模式是主线程抽象
- `TaskStopTool`: 需要主线程任务状态
- `TungstenTool`: 单例虚拟终端冲突
- MCP 相关工具: 待实现（TBD）

### 4. 进程内队友允许工具 (`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`)

仅对进程内队友（in-process teammates）可用的工具，不适用于普通异步 Agent：

**队友通信**：`SendMessageTool`
**任务管理**：`TaskCreateTool`, `TaskGetTool`, `TaskListTool`, `TaskUpdateTool`
**定时任务**（当 `AGENT_TRIGGERS` 特性启用时）：`CronCreateTool`, `CronDeleteTool`, `CronListTool`

这些工具由 `inProcessRunner.ts` 注入，通过 `isInProcessTeammate()` 检查放行。

### 5. Coordinator 模式允许工具 (`COORDINATOR_MODE_ALLOWED_TOOLS`)

Coordinator 模式仅允许输出和 Agent 管理工具：
- `AgentTool`: 创建和管理子 Agent
- `TaskStopTool`: 停止任务
- `SendMessageTool`: 发送消息
- `SyntheticOutputTool`: 合成输出

## 具体技术实现

### 导入结构

```typescript
// biome-ignore-all assist/source/organizeImports: ANT-ONLY import markers must not be reordered
import { feature } from 'bun:bundle'
import { TASK_OUTPUT_TOOL_NAME } from '../tools/TaskOutputTool/constants.js'
import { EXIT_PLAN_MODE_V2_TOOL_NAME } from '../tools/ExitPlanModeTool/constants.js'
// ... 更多工具名称导入
import { SHELL_TOOL_NAMES } from '../utils/shell/shellToolUtils.js'
```

**注意**：`biome-ignore-all` 注释防止导入排序，因为某些导入有顺序依赖（如 ANT-ONLY 标记）。

### 工具集合定义

```typescript
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  TASK_OUTPUT_TOOL_NAME,
  EXIT_PLAN_MODE_V2_TOOL_NAME,
  ENTER_PLAN_MODE_TOOL_NAME,
  // Allow Agent tool for agents when user is ant (enables nested agents)
  ...(process.env.USER_TYPE === 'ant' ? [] : [AGENT_TOOL_NAME]),
  ASK_USER_QUESTION_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  // Prevent recursive workflow execution inside subagents.
  ...(feature('WORKFLOW_SCRIPTS') ? [WORKFLOW_TOOL_NAME] : []),
])

export const CUSTOM_AGENT_DISALLOWED_TOOLS = new Set([
  ...ALL_AGENT_DISALLOWED_TOOLS,
])

export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
])

export const IN_PROCESS_TEAMMATE_ALLOWED_TOOLS = new Set([
  TASK_CREATE_TOOL_NAME,
  TASK_GET_TOOL_NAME,
  TASK_LIST_TOOL_NAME,
  TASK_UPDATE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  ...(feature('AGENT_TRIGGERS')
    ? [CRON_CREATE_TOOL_NAME, CRON_DELETE_TOOL_NAME, CRON_LIST_TOOL_NAME]
    : []),
])

export const COORDINATOR_MODE_ALLOWED_TOOLS = new Set([
  AGENT_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

### 运行时特性检查

使用 `feature()` 函数进行特性开关检查：
- `WORKFLOW_SCRIPTS`: 控制是否在工作流中禁止递归
- `AGENT_TRIGGERS`: 控制队友是否可以使用定时任务工具

## 关键代码路径与文件引用

### 调用方分析

| 调用文件 | 调用内容 | 用途 |
|----------|----------|------|
| `src/tools.ts` | `ALL_AGENT_DISALLOWED_TOOLS` | 主工具注册和过滤 |
| `src/tools/AgentTool/agentToolUtils.ts` | 所有集合 | Agent 工具过滤核心逻辑 |
| `src/utils/toolPool.ts` | 相关集合 | 工具池管理 |
| `src/coordinator/coordinatorMode.ts` | `COORDINATOR_MODE_ALLOWED_TOOLS` | Coordinator 模式工具限制 |

### 核心消费代码示例

```typescript
// src/tools/AgentTool/agentToolUtils.ts
import {
  ALL_AGENT_DISALLOWED_TOOLS,
  ASYNC_AGENT_ALLOWED_TOOLS,
  CUSTOM_AGENT_DISALLOWED_TOOLS,
  IN_PROCESS_TEAMMATE_ALLOWED_TOOLS,
} from '../../constants/tools.js'

export function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync = false,
  permissionMode,
}: {
  tools: Tools
  isBuiltIn: boolean
  isAsync?: boolean
  permissionMode?: PermissionMode
}): Tools {
  return tools.filter(tool => {
    // Allow MCP tools for all agents
    if (tool.name.startsWith('mcp__')) {
      return true
    }
    // Allow ExitPlanMode for agents in plan mode
    if (
      toolMatchesName(tool, EXIT_PLAN_MODE_V2_TOOL_NAME) &&
      permissionMode === 'plan'
    ) {
      return true
    }
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) {
      return false
    }
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) {
      return false
    }
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) {
      return false
    }
    // ... 更多过滤逻辑
  })
}
```

### Coordinator 模式使用

```typescript
// src/coordinator/coordinatorMode.ts
import { COORDINATOR_MODE_ALLOWED_TOOLS } from '../constants/tools.js'

// 过滤工具列表只保留 Coordinator 允许的工具
const coordinatorTools = allTools.filter(tool =>
  COORDINATOR_MODE_ALLOWED_TOOLS.has(tool.name)
)
```

## 依赖与外部交互

### 外部依赖

| 模块 | 用途 |
|------|------|
| `bun:bundle` | 特性标志检查（`feature()`） |
| `../tools/*/constants.js` | 工具名称常量 |
| `../tools/*/prompt.js` | 部分工具名称定义在 prompt 文件中 |
| `../utils/shell/shellToolUtils.js` | Shell 工具名称集合 |

### 权限层级

```
┌─────────────────────────────────────────────────────────────┐
│                    工具权限层级架构                          │
├─────────────────────────────────────────────────────────────┤
│  全功能模式 (Main Thread)                                    │
│  └── 所有工具可用                                            │
├─────────────────────────────────────────────────────────────┤
│  Coordinator 模式                                            │
│  └── AGENT_TOOL, TASK_STOP, SEND_MESSAGE, SYNTHETIC_OUTPUT  │
├─────────────────────────────────────────────────────────────┤
│  进程内队友 (In-Process Teammate)                            │
│  └── ASYNC_AGENT_ALLOWED_TOOLS + 队友专用工具               │
├─────────────────────────────────────────────────────────────┤
│  异步 Agent (Async Agent)                                    │
│  └── ASYNC_AGENT_ALLOWED_TOOLS                               │
├─────────────────────────────────────────────────────────────┤
│  内置 Agent (Built-in Agent)                                 │
│  └── 全部 - ALL_AGENT_DISALLOWED_TOOLS                      │
├─────────────────────────────────────────────────────────────┤
│  自定义 Agent (Custom Agent)                                 │
│  └── 全部 - CUSTOM_AGENT_DISALLOWED_TOOLS                   │
└─────────────────────────────────────────────────────────────┘
```

### 特殊处理

1. **MCP 工具**：所有 Agent 都可以使用 `mcp__` 前缀的工具
2. **Plan 模式例外**：处于 plan 模式的 Agent 可以使用 `ExitPlanModeV2Tool`
3. **Ant 用户特权**：`USER_TYPE=ant` 可以启用嵌套 Agent

## 风险、边界与改进建议

### 潜在风险

1. **权限提升漏洞**
   - 如果工具过滤逻辑有漏洞，Agent 可能获得不应有的权限
   - 需要确保所有工具调用路径都经过 `filterToolsForAgent`

2. **特性开关不一致**
   - `feature()` 在构建时求值，运行时可能不一致
   - 环境变量 `USER_TYPE` 检查在模块加载时执行

3. **递归风险**
   - 即使禁止了 `AgentTool`，仍可能通过其他方式实现递归
   - 例如：通过 Shell 工具调用 CLI

4. **工具名称硬编码**
   - 工具名称分散在多个文件中
   - 重命名工具时需要同步更新此模块

### 边界情况

1. **空工具集**：如果所有工具都被过滤，Agent 将无法执行任何操作
2. **未知工具**：新添加的工具默认不在任何集合中，需要显式配置
3. **动态工具**：MCP 工具动态加载，通过前缀匹配放行

### 改进建议

1. **集中式工具注册**
   ```typescript
   // 建议：每个工具声明自己的 Agent 权限
   interface ToolDefinition {
     name: string
     allowedFor: {
       asyncAgent: boolean
       inProcessTeammate: boolean
       coordinator: boolean
       customAgent: boolean
     }
   }
   ```

2. **运行时权限检查**
   ```typescript
   // 建议：将权限检查从静态集合改为运行时函数
   export function isToolAllowedForAgent(
     toolName: string,
     context: AgentContext,
   ): boolean {
     // 动态权限决策
   }
   ```

3. **权限审计日志**
   - 记录每个 Agent 的工具使用尝试
   - 标记被过滤的工具调用尝试

4. **测试覆盖**
   - 为每个工具集合添加单元测试
   - 验证新工具默认被拒绝
   - 测试特性开关切换

5. **文档生成**
   ```typescript
   // 从代码生成权限文档
   export function generateToolPermissionsDoc(): string {
     // 返回 Markdown 格式的权限表
   }
   ```

6. **类型安全增强**
   ```typescript
   // 使用品牌类型确保工具名称正确
   type ToolName = string & { __brand: 'ToolName' }
   
   // 编译时验证集合内容
   const ALL_AGENT_DISALLOWED_TOOLS = new Set<ToolName>([
     // ...
   ])
   ```

7. **动态权限配置**
   - 支持通过 GrowthBook 动态调整工具权限
   - 紧急情况下可以远程禁用有问题的工具
