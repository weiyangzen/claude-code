# TaskGetTool.ts 研究文档

## 场景与职责

`TaskGetTool.ts` 定义了 **TaskGet** 工具，是 Claude Code 第二代任务管理（TodoV2）工具集的核心组件之一。该工具的主要职责是：**根据任务 ID 从当前任务列表中检索单个任务的完整详情**。

其设计目标服务于以下场景：
- **Agent Swarms（多代理协作）**：in-process 或 tmux  teammates 需要获取某个任务的具体描述、依赖关系后才能开始执行。
- **主线程任务查询**：用户在交互式会话中要求查看某个具体任务的详情时，主代理通过该工具读取任务文件。
- **任务依赖分析**：在 teammates 认领任务前，检查 `blockedBy` 列表是否为空，避免在阻塞条件未满足时开始工作。

该工具属于**只读工具**，不修改任何任务状态，也不会触发任务创建/完成时的 hooks。

## 功能点目的

1. **按 ID 精确查询**：输入一个 `taskId`（字符串），返回对应任务的完整元数据。
2. **返回任务依赖信息**：输出包含 `blocks`（该任务阻塞了哪些任务）和 `blockedBy`（该任务被哪些任务阻塞），帮助代理理解任务拓扑。
3. **人类可读的结果渲染**：通过 `mapToolResultToToolResultBlockParam` 将结构化数据格式化为自然语言文本，供模型在后续对话中直接消费。
4. **延迟加载（Deferred Tool）**：`shouldDefer: true` 表示该工具不会常驻在系统提示中，只有在模型通过 `ToolSearch` 显式请求后才会加载其完整 schema，减少长上下文开销。

## 具体技术实现

### 工具定义构建

文件使用 `buildTool`（来自 `src/Tool.ts`）构建完整的 `Tool` 对象，并满足 `ToolDef<InputSchema, Output>` 类型约束。`buildTool` 会自动填充默认值（如 `isEnabled`、`isConcurrencySafe`、`isReadOnly`、`checkPermissions` 等），使定义保持简洁。

### 输入/输出 Schema

```typescript
// 输入：严格对象，仅包含 taskId
const inputSchema = lazySchema(() =>
  z.strictObject({
    taskId: z.string().describe('The ID of the task to retrieve'),
  }),
)

// 输出：包含一个可为 null 的 task 对象
const outputSchema = lazySchema(() =>
  z.object({
    task: z.object({
      id: z.string(),
      subject: z.string(),
      description: z.string(),
      status: TaskStatusSchema(),  // 'pending' | 'in_progress' | 'completed'
      blocks: z.array(z.string()),
      blockedBy: z.array(z.string()),
    }).nullable(),
  }),
)
```

Schema 使用 `lazySchema` 包装（`src/utils/lazySchema.ts`），将 Zod schema 的构造从模块加载时延迟到首次访问时，降低启动开销。

### 核心调用流程 (`call`)

```typescript
async call({ taskId }) {
  const taskListId = getTaskListId()
  const task = await getTask(taskListId, taskId)
  if (!task) {
    return { data: { task: null } }
  }
  return {
    data: {
      task: {
        id: task.id,
        subject: task.subject,
        description: task.description,
        status: task.status,
        blocks: task.blocks,
        blockedBy: task.blockedBy,
      },
    },
  }
}
```

流程说明：
1. 调用 `getTaskListId()` 解析当前上下文对应的任务列表 ID（优先级：环境变量 `CLAUDE_CODE_TASK_LIST_ID` > teammate 上下文 teamName > `getTeamName()` > leader team name > session ID）。
2. 调用 `getTask(taskListId, taskId)` 从文件系统读取任务 JSON 文件（路径：`~/.claude/tasks/{taskListId}/{taskId}.json`）。
3. 若任务不存在或解析失败，返回 `{ task: null }`。
4. 若存在，提取并返回 `id`、`subject`、`description`、`status`、`blocks`、`blockedBy` 六个字段。

### 结果渲染 (`mapToolResultToToolResultBlockParam`)

将结构化输出转换为 Anthropic `tool_result` 块：

- 任务不存在时：返回 `"Task not found"`。
- 任务存在时：生成多行文本，格式如下：
  ```
  Task #{id}: {subject}
  Status: {status}
  Description: {description}
  Blocked by: #{id1}, #{id2}   // 仅在 blockedBy 非空时显示
  Blocks: #{id1}, #{id2}       // 仅在 blocks 非空时显示
  ```

### 工具元数据

| 属性 | 值 | 说明 |
|------|-----|------|
| `name` | `TaskGet` | 工具唯一标识 |
| `searchHint` | `retrieve a task by ID` | 供 ToolSearch 匹配的关键词 |
| `maxResultSizeChars` | `100_000` | 结果大小上限 |
| `shouldDefer` | `true` | 延迟加载工具 |
| `isEnabled` | `isTodoV2Enabled()` | 非交互式会话默认禁用，除非 `CLAUDE_CODE_ENABLE_TASKS` 为真 |
| `isConcurrencySafe` | `true` | 可并发执行 |
| `isReadOnly` | `true` | 只读，不触发写权限检查 |
| `userFacingName` | `TaskGet` | UI 显示名称 |
| `toAutoClassifierInput` | `input.taskId` | 自动分类器输入 |
| `renderToolUseMessage` | `null` | 不渲染自定义工具使用消息 |

## 关键代码路径与文件引用

### 本文件
- `src/tools/TaskGetTool/TaskGetTool.ts` — 工具定义与实现。

### 直接依赖
- `src/Tool.ts` — `buildTool`、`ToolDef`、完整的 `Tool` 类型系统。
- `src/utils/lazySchema.ts` — `lazySchema`，延迟 schema 构造。
- `src/utils/tasks.ts` — `getTask`、`getTaskListId`、`isTodoV2Enabled`、`TaskStatusSchema`、任务文件读写逻辑。
- `src/tools/TaskGetTool/constants.ts` — `TASK_GET_TOOL_NAME`。
- `src/tools/TaskGetTool/prompt.ts` — `DESCRIPTION`、`PROMPT`。

### 调用方（注册与权限）
- `src/tools.ts` — 在 `getAllBaseTools()` 中条件注册：
  ```typescript
  ...(isTodoV2Enabled()
    ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool]
    : []),
  ```
- `src/constants/tools.ts` — `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` 包含 `TASK_GET_TOOL_NAME`，允许 in-process teammates 使用该工具。
- `src/utils/permissions/classifierDecision.ts` — `SAFE_YOLO_ALLOWLISTED_TOOLS` 包含 `TASK_GET_TOOL_NAME`，在 YOLO 自动模式下可跳过分类器检查。
- `src/utils/swarm/inProcessRunner.ts` — 为 in-process teammates 构建 `CustomAgentDefinition` 时，将 `TASK_GET_TOOL_NAME` 强制注入工具白名单（即使 agent 定义显式指定了工具列表）。

### 底层存储路径
- 任务文件：`~/.claude/tasks/{taskListId}/{taskId}.json`
- 锁文件：`~/.claude/tasks/{taskListId}/.lock`
- 高水位标记：`~/.claude/tasks/{taskListId}/.highwatermark`

## 依赖与外部交互

### 运行时依赖
- **Zod v4**：输入/输出 schema 校验（`z.strictObject`、`z.object`、`z.array`、`z.string`、`z.nullable`）。
- **`src/utils/tasks.ts`**：
  - `getTaskListId()`：根据当前上下文（环境变量、teammate 上下文、session ID）解析任务列表 ID。
  - `getTask(taskListId, taskId)`：异步读取任务 JSON 文件，包含旧状态迁移逻辑（仅 `USER_TYPE === 'ant'` 时生效），并做 schema 校验。
  - `isTodoV2Enabled()`：判断任务工具是否启用。
  - `TaskStatusSchema()`：返回 `z.enum(['pending', 'in_progress', 'completed'])`。

### 无外部网络/服务交互
TaskGetTool 是纯本地文件系统读取工具，不涉及网络请求、MCP 服务器、分析上报或用户交互 UI。

## 风险、边界与改进建议

### 风险

1. **任务文件损坏或 schema 不匹配**：`getTask` 内部使用 `TaskSchema().safeParse(data)`，若解析失败会记录调试日志并返回 `null`。此时 TaskGetTool 会返回 `"Task not found"`，模型可能误判为任务不存在，而非文件损坏。
2. **任务列表 ID 解析歧义**：`getTaskListId()` 的解析链较长（环境变量 → teammate 上下文 → team name → session ID）。若用户手动切换 team 或环境变量残留，可能导致读取到错误的任务列表。
3. **信息截断导致代理决策不足**：TaskGetTool 的输出**不包含** `owner`、`activeForm`、`metadata` 字段。在 swarm 场景中，teammate 无法通过该工具判断任务是否已被他人认领，或获取任务的元数据（如优先级、标签）。

### 边界

- **只读边界**：该工具不修改任务状态，也不更新 `owner` 或 `status`。若代理在获取任务后直接开始工作，需配合 `TaskUpdateTool` 将任务标记为 `in_progress` 并设置 `owner`。
- **延迟加载边界**：由于 `shouldDefer: true`，模型在首次对话中可能看不到 TaskGet 的完整 schema，需要先通过 `ToolSearch` 或系统提示中的 `searchHint` 触发加载。若 ToolSearch 阈值未达或模型未主动搜索，可能遗漏该工具。
- **非交互式禁用边界**：默认情况下，非交互式会话（如 SDK）中 `isTodoV2Enabled()` 返回 `false`，TaskGetTool 不可用。需显式设置 `CLAUDE_CODE_ENABLE_TASKS=true` 才能启用。

### 改进建议

1. **扩展输出字段**：建议在 `outputSchema` 中增加 `owner`、`activeForm`、`metadata` 的暴露，使 teammate 能更全面地评估任务状态，减少额外的 `TaskListTool` 调用。
2. **增强错误区分**：当 `getTask` 返回 `null` 时，可区分 "文件不存在"（ENOENT）与 "schema 解析失败"，并在 `mapToolResultToToolResultBlockParam` 中返回更明确的错误信息，帮助模型诊断。
3. **缓存优化**：对于高频查询同一任务的场景（如 teammate 轮询检查依赖），可考虑在 `getTask` 层引入短期内存缓存，减少重复的文件系统读取。需注意多进程并发下的缓存一致性。
4. **权限检查细化**：当前 `isReadOnly: true` 且 `checkPermissions` 使用默认 `allow`。虽然任务数据通常不敏感，但在多租户或共享机器上，可考虑增加对任务列表目录的访问权限校验，防止跨 session 读取。
