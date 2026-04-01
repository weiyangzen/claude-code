# 研究文档：src/tools/TaskOutputTool/TaskOutputTool.tsx

## 场景与职责

`TaskOutputTool.tsx` 定义了 **TaskOutput** 工具（内部名称为 `TaskOutput`，即 `TASK_OUTPUT_TOOL_NAME`），用于让模型读取后台任务的输出内容。该工具面向所有需要"查询某个已启动的后台任务当前状态与输出"的场景，包括：
- 后台 Bash/Shell 任务（`local_bash`）
- 本地 Agent 子任务（`local_agent`）
- 远程 Agent/Review 任务（`remote_agent`）

**重要背景**：该工具已被标记为 **Deprecated**。代码注释与 `description()`/`prompt()` 均明确建议模型"优先直接对任务返回的 output file path 使用 Read 工具"，而不是调用 `TaskOutputTool`。它目前仍被注册在全局工具列表中（`src/tools.ts` 的 `getAllBaseTools()`），但主要作为向后兼容与过渡保留。

在权限与分类层面，`TaskOutputTool` 被归入：
- **只读工具**（`src/components/agents/ToolSelector.tsx` 的 `READ_ONLY` bucket）
- **Async Agent 禁止工具**（`src/constants/tools.ts` 的 `ALL_AGENT_DISALLOWED_TOOLS`），防止子 Agent 递归查询父任务输出
- **YOLO 安全白名单**（`src/utils/permissions/classifierDecision.ts` 的 `SAFE_YOLO_ALLOWLISTED_TOOLS`），在自动模式下可跳过额外分类检查

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `task_id` 输入 | 指定要查询的目标任务 ID |
| `block`（默认 `true`） | 是否阻塞等待任务完成；`true` 时轮询直到任务退出 `running`/`pending`，`false` 时立即返回当前状态 |
| `timeout`（默认 30000ms，上限 600000ms） | 阻塞模式下的最大等待时间 |
| 输出格式化与截断 | 通过 `formatTaskOutput()` 控制返回给模型的输出长度，避免超出 token 预算 |
| 任务类型差异化输出 | 对 bash 返回 `exitCode`；对 local_agent 返回 `prompt`/`result`/`error`；对 remote_agent 返回 `prompt`（即 `command`） |
| 进度渲染 | 阻塞等待期间向 UI 发送 `waiting_for_task` 进度消息，提升交互体验 |
| 结果渲染 | 使用 React/ink 组件在终端 UI 中差异化展示各任务类型的输出（`BashToolResultMessage`、`AgentPromptDisplay`、`AgentResponseDisplay` 等） |

---

## 具体技术实现

### 1. 输入 Schema（Zod）

```ts
const inputSchema = lazySchema(() => z.strictObject({
  task_id: z.string().describe('The task ID to get output from'),
  block: semanticBoolean(z.boolean().default(true)).describe('Whether to wait for completion'),
  timeout: z.number().min(0).max(600000).default(30000).describe('Max wait time in ms')
}));
```

- 使用 `lazySchema` 延迟构造，避免模块加载时即实例化 Zod schema。
- `semanticBoolean` 预处理字符串 `"true"`/`"false"`，兼容模型偶发的字符串化布尔值输入。

### 2. 核心数据类型

```ts
type TaskOutput = {
  task_id: string;
  task_type: TaskType;
  status: string;
  description: string;
  output: string;
  exitCode?: number | null;
  error?: string;
  prompt?: string;
  result?: string;
};

type TaskOutputToolOutput = {
  retrieval_status: 'success' | 'timeout' | 'not_ready';
  task: TaskOutput | null;
};
```

### 3. 获取任务输出：`getTaskOutputData(task)`

该函数统一处理三种任务类型的输出读取：

- **`local_bash`**：优先从内存中的 `shellCommand?.taskOutput` 读取 stdout/stderr；若不存在，则回退到磁盘文件 `getTaskOutput(task.id)`（`src/utils/task/diskOutput.ts`）。最终拼接并返回 `exitCode`。
- **`local_agent`**：优先使用内存中的 `agentTask.result`（通过 `extractTextContent` 提取纯文本），回退到磁盘输出。返回 `prompt`、`result`（与 `output` 同值）、`error`。
- **`remote_agent`**：返回 `prompt`（实际为 `remoteTask.command`）。

### 4. 阻塞等待：`waitForTaskCompletion(taskId, getAppState, timeoutMs, abortController?)`

简单的轮询实现：
- 每 100ms 调用 `getAppState()` 检查任务状态。
- 若任务状态不再是 `running` 或 `pending`，立即返回该任务。
- 若超时，返回当前状态（可能仍在运行）。
- 支持 `AbortController` 中断，抛出 `AbortError`。

### 5. `call()` 方法执行流程

```
1. 校验 task 存在（已在 validateInput 中完成，call 中再次防御性检查）
2. 若 block=false：
   - 任务已结束 → 标记 notified=true，返回 retrieval_status='success'
   - 任务仍在运行 → 返回 retrieval_status='not_ready'
3. 若 block=true：
   - 发送 onProgress({ type: 'waiting_for_task', taskDescription, taskType })
   - 调用 waitForTaskCompletion()
   - 任务消失 → 'timeout' + task=null
   - 超时但仍在运行 → 'timeout' + 当前 task
   - 正常结束 → 标记 notified=true，返回 'success'
```

标记 `notified=true` 的作用：让任务框架（`src/utils/task/framework.ts` 的 `generateTaskAttachments` / `evictTerminalTask`）知道该任务已被消费，可以在合适的时机从 `AppState.tasks` 中驱逐（GC），释放内存。

### 6. 结果序列化：`mapToolResultToToolResultBlockParam()`

将结构化输出序列化为 XML 标签文本，供后续 LLM 消费：

```xml
<retrieval_status>success</retrieval_status>
<task_id>bxxxxx</task_id>
<task_type>local_bash</task_type>
<status>completed</status>
<exit_code>0</exit_code>
<output>
...（经 formatTaskOutput 截断后的内容）
</output>
<error>...</error>
```

`formatTaskOutput()`（`src/utils/task/outputFormatting.ts`）默认限制输出长度为 32,000 字符（可通过环境变量 `TASK_MAX_OUTPUT_LENGTH` 调整，上限 160,000），超限时保留尾部并在头部附加完整文件路径提示。

### 7. UI 渲染：`TaskOutputResultDisplay`

React 函数组件（经过 React Compiler 编译，代码中出现 `_c(54)` 等 memo cache 模式），根据 `task_type` 与 `verbose` 标志差异化渲染：

- **`local_bash`**：复用 `BashToolResultMessage`，将 `task.output` 作为 stdout、`task.error` 作为 `returnCodeInterpretation` 传入。
- **`local_agent`**：
  - `verbose=true`：展示描述、行数、prompt（`AgentPromptDisplay`）、result（`AgentResponseDisplay`）、error。
  - `verbose=false`：仅显示 "Read output (ctrl+o to expand)" 的折叠提示。
  - 若 `retrieval_status` 为 `timeout`/`not_ready`：显示 "Task is still running…"。
- **`remote_agent`**：展示描述与状态，verbose 时展开输出。
- **其他类型**：兜底展示描述、状态、输出前 500 字符。

---

## 关键代码路径与文件引用

| 本文件内符号 | 职责 | 引用的外部文件 |
|-------------|------|---------------|
| `getTaskOutputData` | 按任务类型读取输出 | `src/utils/task/diskOutput.ts` (`getTaskOutput`)<br>`src/utils/messages.ts` (`extractTextContent`) |
| `waitForTaskCompletion` | 轮询等待任务结束 | `src/utils/sleep.ts` (`sleep`)<br>`src/utils/errors.ts` (`AbortError`) |
| `call` | 工具调用主入口 | `src/utils/task/framework.ts` (`updateTaskState`)<br>`src/utils/task/outputFormatting.ts` (`formatTaskOutput`) |
| `mapToolResultToToolResultBlockParam` | 序列化为 LLM 文本块 | `src/utils/task/outputFormatting.ts` (`formatTaskOutput`) |
| `TaskOutputResultDisplay` | 终端 UI 渲染 | `src/components/FallbackToolUseErrorMessage.js`<br>`src/components/FallbackToolUseRejectedMessage.js`<br>`src/components/MessageResponse.js`<br>`src/ink.js` (`Box`, `Text`)<br>`src/keybindings/useShortcutDisplay.js`<br>`src/tools/AgentTool/UI.js` (`AgentPromptDisplay`, `AgentResponseDisplay`)<br>`src/tools/BashTool/BashToolResultMessage.js` |
| 输入 Schema | 参数校验 | `src/utils/lazySchema.ts`<br>`src/utils/semanticBoolean.ts` |
| 类型定义 | 任务状态类型 | `src/tasks/LocalAgentTask/LocalAgentTask.tsx`<br>`src/tasks/LocalShellTask/guards.ts`<br>`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`<br>`src/tasks/types.ts`<br>`src/Task.ts` (`TaskType`, `TaskStateBase`, `TaskStatus`) |

**注册与集成路径**：
- `src/tools.ts` 第 54 行导入并第 196 行将其加入 `getAllBaseTools()`。
- `src/components/agents/ToolSelector.tsx` 第 19、53 行将其归入只读工具 bucket。
- `src/constants/tools.ts` 第 3、36、93 行将其列入禁止递归工具与注释说明。
- `src/utils/permissions/classifierDecision.ts` 第 15、73 行将其加入 YOLO 安全白名单。
- `src/utils/permissions/permissionRuleParser.ts` 第 3、24 行维护旧名别名映射（`AgentOutputTool` → `TaskOutput`，`BashOutputTool` → `TaskOutput`）。
- `src/utils/messages.ts` 第 144 行导入工具名，用于消息处理中的工具识别。
- `src/utils/api.ts` 第 36 行导入工具名，用于系统提示或 schema 构建中的特殊处理（如 swarm 字段过滤不涉及此工具）。

---

## 依赖与外部交互

### 编译/构建依赖
- **React Compiler**：源码已被编译，出现大量 `_c(n)` memo cache 代码，直接修改需留意编译后产物与原始 TSX 的对应关系。
- **Zod v4**：输入校验与 JSON Schema 生成的基础库。

### 运行时依赖
- **AppState 任务存储**：通过 `toolUseContext.getAppState()` / `setAppState()` 访问任务状态，与整个任务生命周期系统（`registerTask`、`updateTaskState`、`evictTerminalTask`）紧耦合。
- **磁盘输出系统**：`src/utils/task/diskOutput.ts` 提供 `DiskTaskOutput` 类、文件路径解析、tail 读取、delta 读取等能力。
- **消息/通知队列**：`notified` 标记影响 `src/utils/task/framework.ts` 中的任务驱逐逻辑，进而影响 `<task-notification>` 的推送时机。
- **Ink UI 框架**：渲染组件依赖 `src/ink.js` 提供的 `Box`、`Text` 等终端 UI 基元。

### 向后兼容
- 保留别名 `AgentOutputTool`、`BashOutputTool`，在 `permissionRuleParser.ts` 与 `aliases` 字段中均有体现，确保旧会话或旧权限规则仍能解析。

---

## 风险、边界与改进建议

### 风险

1. **已弃用但仍在注册表**：工具明确 deprecated，但仍在 `getAllBaseTools()` 中暴露给模型。模型可能继续调用它，而非遵循建议直接 Read 输出文件。
2. **轮询阻塞的精度与成本**：`waitForTaskCompletion` 每 100ms 轮询一次 `getAppState()`，对极短任务会造成少量空转；对长任务则持续占用事件循环切片。虽然单次开销小，但大量并发阻塞调用时可能累积。
3. **超时后仍返回部分数据**：当 `block=true` 且超时时，若任务仍在运行，返回 `retrieval_status='timeout'` 并附带当前 `task` 数据。调用方（模型）需正确理解"timeout 不代表失败"，否则可能误判任务状态。
4. **输出截断导致上下文丢失**：`formatTaskOutput` 默认只保留尾部 32K 字符，对于产生大量日志的 bash 任务，模型只能看到末尾输出，可能丢失早期错误信息。虽然头部提示了完整文件路径，但模型未必会主动 Read。
5. **递归风险**：`TaskOutputTool` 被明确禁止给 Async Agent 使用（`ALL_AGENT_DISALLOWED_TOOLS`），防止子 Agent 查询父 Agent 的任务输出造成递归或信息泄露。若该禁止列表被意外修改，将引入架构风险。
6. **编译后代码的可维护性**：文件为 React Compiler 输出产物，包含大量手动 memo cache 索引（`$[0]`、`$[1]` 等），直接人工修改极易破坏缓存一致性，导致渲染 bug。

### 边界

- **最大超时**：`timeout` 上限硬编码为 600,000ms（10 分钟），超过该值会在 Zod 校验阶段被拒绝。
- **最大结果字符数**：`maxResultSizeChars: 100_000`（`buildTool` 配置），由工具框架在更外层进行二次截断保护。
- **任务类型覆盖**：当前仅对 `local_bash`、`local_agent`、`remote_agent` 做了差异化字段处理；`in_process_teammate`、`local_workflow`、`monitor_mcp`、`dream` 等类型走兜底分支，可能丢失类型特有信息。
- **磁盘输出 vs 内存输出**：`local_bash` 优先读内存 `shellCommand?.taskOutput`，但该对象仅在任务由当前进程启动时才存在；对于 `--resume` 恢复的任务，只能读磁盘文件，可能因文件已被清理（`cleanupTaskOutput`）而得到空字符串。

### 改进建议

1. **彻底移除或降级暴露**：既然已 deprecated，可考虑：
   - 从 `getAllBaseTools()` 中移除，仅保留别名映射用于旧会话兼容；或
   - 将 `isEnabled()` 设为返回 `false`（外部构建），逐步淘汰。
2. **将轮询改为事件驱动**：`waitForTaskCompletion` 的轮询可替换为基于 `updateTaskState` 事件或 Promise 的等待机制，减少空转。任务完成时由任务框架主动 resolve 一个 Promise，避免 100ms 轮询。
3. **增强 timeout 场景的指导**：在 `prompt()` 和 `description()` 中进一步强调"timeout 时应再次调用或改用 Read 工具"，降低模型误判概率。
4. **输出截断提示优化**：在 `formatTaskOutput` 的截断头部增加更明确的结构化提示（例如 `<truncated reason="length">`），帮助模型识别并决定是否需要 Read 完整文件。
5. **扩展任务类型支持**：若未来 `monitor_mcp` 或 `dream` 任务也需要被查询，应在 `getTaskOutputData` 中补充对应分支，避免信息丢失。
6. **源码与编译产物分离**：建议将 React Compiler 作为构建步骤在 CI 中运行，仓库中保留可读的人类编写源码，降低后续维护与审查成本。
