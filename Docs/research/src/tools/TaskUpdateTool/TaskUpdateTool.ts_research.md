# 研究报告：src/tools/TaskUpdateTool/TaskUpdateTool.ts

## 场景与职责

`TaskUpdateTool.ts` 是 Claude Code 交互式任务列表（Todo V2）体系中的核心变更工具，负责让模型/代理对已有任务执行状态流转、字段修改、依赖关系调整及所有权转移。它是任务生命周期管理的“写操作”入口之一，与 `TaskCreateTool`（创建）、`TaskGetTool`（读取）、`TaskListTool`（枚举）共同构成完整的任务管理工具链。该工具仅在交互式会话中启用（`isTodoV2Enabled()` 为真，即非 `CLAUDE_CODE_ENABLE_TASKS` 强制开启的非交互式场景外），并通过 `shouldDefer: true` 标记为延迟加载工具，意味着模型需先通过 `ToolSearch` 发现它，才会在后续轮次获得完整 schema。

在 Agent Swarm（多代理协作）场景下，`TaskUpdateTool` 还承担了 swarm 协调的附加职责：
- 当 teammate 将任务标记为 `in_progress` 时，若未显式指定 owner，自动将当前 agent 设为主人；
- 当任务 owner 发生变更时，通过 mailbox 向新 owner 发送任务分配通知；
- 当 teammate 完成任务时，在工具结果中追加“调用 TaskList 寻找下一个任务”的引导语。

## 功能点目的

1. **任务状态流转**：支持 `pending` → `in_progress` → `completed` 的三态流转，以及通过 `deleted` 永久删除任务。
2. **字段级更新**：可修改 `subject`（标题）、`description`（描述）、`activeForm`（进行时的 spinner 文案）、`owner`（负责人）、`metadata`（扩展元数据，支持 merge 及通过 `null` 删除键）。
3. **依赖关系管理**：通过 `addBlocks`（此任务阻塞哪些任务）和 `addBlockedBy`（此任务被哪些任务阻塞）增量追加任务依赖，底层调用 `blockTask` 双向维护 `blocks` / `blockedBy` 数组。
4. **TaskCompleted Hook 触发**：当任务被标记为 `completed` 时，先执行 `executeTaskCompletedHooks`，允许用户配置的 shell/HTTP/agent hook 拦截或阻塞完成动作（例如强制要求代码审查通过）。
5. **Swarm 自动认领**：在 `isAgentSwarmsEnabled()` 开启且 `status === 'in_progress'` 时，若未传 `owner` 且当前任务无主，则自动将 `getAgentName()` 写入 `owner`。
6. **Mailbox 通知**：当 `owner` 发生变更且处于 swarm 模式时，向新 owner 的 mailbox 写入 `task_assignment` 类型 JSON 消息，使其在后续轮次收到系统附件提醒。
7. **验证提示（Verification Nudge）**：当主线程代理一次性完成 3 个以上任务且没有任何任务标题匹配 `/verif/i` 时，在输出中追加提示，要求模型 spawn `verification` 子代理进行验证（仅对 V2 交互式 CLI 生效，作为 TodoWriteTool nudge 的镜像逻辑）。
8. **UI 状态联动**：调用时通过 `context.setAppState` 自动将左侧边栏展开到 `tasks` 视图，确保用户实时看到任务列表变化。

## 具体技术实现

### 工具定义模式

文件采用 `buildTool({ ... }) satisfies ToolDef<InputSchema, Output>` 模式构建。`buildTool` 位于 `src/Tool.ts`，会为缺失字段填充安全默认值（如 `isConcurrencySafe: false`、`checkPermissions: allow` 等）。`TaskUpdateTool` 显式覆盖了以下关键字段：

- `name`: `TASK_UPDATE_TOOL_NAME`（来自 `constants.ts`，值为 `'TaskUpdate'`）
- `searchHint`: `'update a task'`
- `maxResultSizeChars`: `100_000`
- `shouldDefer`: `true`
- `isEnabled()`: 委托 `isTodoV2Enabled()`
- `isConcurrencySafe()`: 返回 `true`（文件级锁在 `utils/tasks.js` 中实现）
- `toAutoClassifierInput()`: 将 `taskId`、`status`、`subject` 拼接为分类器输入文本
- `renderToolUseMessage()`: 返回 `null`（不渲染特殊 UI）
- `call()`: 核心执行逻辑
- `mapToolResultToToolResultBlockParam()`: 将结构化输出转换为 Anthropic `tool_result` block

### Schema 设计

输入 schema 通过 `lazySchema` 延迟构造（避免模块加载时立即实例化 Zod 对象）：

```ts
const inputSchema = lazySchema(() => {
  const TaskUpdateStatusSchema = TaskStatusSchema().or(z.literal('deleted'))
  return z.strictObject({
    taskId: z.string(),
    subject: z.string().optional(),
    description: z.string().optional(),
    activeForm: z.string().optional(),
    status: TaskUpdateStatusSchema.optional(),
    addBlocks: z.array(z.string()).optional(),
    addBlockedBy: z.array(z.string()).optional(),
    owner: z.string().optional(),
    metadata: z.record(z.string(), z.unknown()).optional(),
  })
})
```

输出 schema：

```ts
z.object({
  success: z.boolean(),
  taskId: z.string(),
  updatedFields: z.array(z.string()),
  error: z.string().optional(),
  statusChange: z.object({ from: z.string(), to: z.string() }).optional(),
  verificationNudgeNeeded: z.boolean().optional(),
})
```

### `call()` 方法关键流程

1. **获取 taskListId**：调用 `getTaskListId()`，优先级为：环境变量 `CLAUDE_CODE_TASK_LIST_ID` > in-process teammate 上下文 > `CLAUDE_CODE_TEAM_NAME` / leader team name > session ID。
2. **展开任务视图**：`context.setAppState(prev => ({ ...prev, expandedView: 'tasks' }))`。
3. **存在性校验**：`const existingTask = await getTask(taskListId, taskId)`；若不存在，返回 `{ success: false, error: 'Task not found' }`。
4. **字段 diff 与合并**：
   - 对 `subject`、`description`、`activeForm`、`owner` 做值对比，仅当 `!== undefined` 且与现有值不同时才写入 `updates` 并记录 `updatedFields`。
   - `metadata` 采用浅合并：以现有 `metadata` 为基，遍历输入键值，值为 `null` 时删除键，否则覆盖/新增。
5. **Swarm 自动 owner**：在 `isAgentSwarmsEnabled()` 为真、`status === 'in_progress'`、`owner === undefined` 且任务当前无 owner 时，自动写入 `getAgentName()`。
6. **状态特殊处理——删除**：若 `status === 'deleted'`，直接调用 `deleteTask(taskListId, taskId)` 并立即返回，**跳过**后续所有字段更新、hook、mailbox 通知逻辑。
7. **状态特殊处理——完成**：若 `status === 'completed'` 且与当前状态不同：
   - 调用 `executeTaskCompletedHooks(taskId, existingTask.subject, existingTask.description, getAgentName(), getTeamName(), undefined, signal, undefined, context)` 获取异步生成器；
   - 遍历生成器结果，收集 `blockingError`；若有任何阻塞错误，立即返回 `{ success: false, updatedFields: [], error: blockingErrors.join('\n') }`，**阻止任务完成**。
8. **持久化更新**：若 `updates` 非空，调用 `updateTask(taskListId, taskId, updates)`。
9. **Mailbox 通知**：若 `updates.owner` 被设置且 swarm 启用，调用 `writeToMailbox(updates.owner, { from: senderName, text: assignmentMessage, timestamp, color }, taskListId)`。
10. **依赖追加**：
    - `addBlocks`：过滤掉已存在的 block ID，对每个新 ID 调用 `blockTask(taskListId, taskId, blockId)`（更新双向关系）。
    - `addBlockedBy`：同理，对每个新 ID 调用 `blockTask(taskListId, blockerId, taskId)`（注意参数顺序反转）。
11. **Verification Nudge 计算**：
    - 条件：`feature('VERIFICATION_AGENT')` 为真、`tengu_hive_evidence` GrowthBook 开关为真、`!context.agentId`（主线程代理）、`updates.status === 'completed'`。
    - 调用 `listTasks(taskListId)` 获取全部任务；若全部已完成、总数 ≥3、且没有任何任务标题匹配 `/verif/i`，则设置 `verificationNudgeNeeded = true`。
12. **返回结果**：包含 `success`、`taskId`、`updatedFields`、`statusChange`、`verificationNudgeNeeded`。

### `mapToolResultToToolResultBlockParam()` 渲染逻辑

- 若 `success === false`，返回 `tool_result` 的 `content` 为 `error || 'Task #${taskId} not found'`。注释特别说明返回为非 error 类型，以避免在 `StreamingToolExecutor` 中触发同级工具取消（“Task not found”被视为良性条件，例如任务列表已被清理）。
- 成功时，基础文本为 `Updated task #${taskId} ${updatedFields.join(', ')}`。
- 若 `statusChange.to === 'completed'` 且当前是 swarm teammate（`getAgentId()` 存在），追加提示语引导其调用 `TaskList` 寻找下一个任务。
- 若 `verificationNudgeNeeded` 为真，追加一段强提示语，要求 spawn `subagent_type="verification"` 的验证代理，并强调只有 verifier 才能给出 verdict，模型不能通过在 summary 中列 caveat 来自我判定 `PARTIAL`。

## 关键代码路径与文件引用

| 功能 | 依赖文件 | 关键导出/函数 |
|------|----------|---------------|
| 工具框架 | `src/Tool.ts` | `buildTool`, `ToolDef`, `ToolUseContext` |
| 任务 CRUD | `src/utils/tasks.ts` | `getTask`, `updateTask`, `deleteTask`, `listTasks`, `blockTask`, `getTaskListId`, `TaskStatus`, `TaskStatusSchema` |
| TaskCompleted Hook | `src/utils/hooks.ts` | `executeTaskCompletedHooks`, `getTaskCompletedHookMessage` |
| Swarm 判定 | `src/utils/agentSwarmsEnabled.ts` | `isAgentSwarmsEnabled` |
| 代理身份 | `src/utils/teammate.ts` | `getAgentId`, `getAgentName`, `getTeammateColor`, `getTeamName` |
| Mailbox 写入 | `src/utils/teammateMailbox.ts` | `writeToMailbox` |
| 验证子代理类型 | `src/tools/AgentTool/constants.ts` | `VERIFICATION_AGENT_TYPE` |
| 特性开关 | `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` |
| 延迟 Schema | `src/utils/lazySchema.ts` | `lazySchema` |
| 工具注册 | `src/tools.ts` | `getAllBaseTools` 在 `isTodoV2Enabled()` 为真时注入 `TaskUpdateTool` |
| 常量 | `src/tools/TaskUpdateTool/constants.ts` | `TASK_UPDATE_TOOL_NAME` |
| 提示文本 | `src/tools/TaskUpdateTool/prompt.ts` | `DESCRIPTION`, `PROMPT` |

## 依赖与外部交互

### 上游调用方

- `src/tools.ts`：在 `getAllBaseTools()` 中条件性注册 `TaskUpdateTool`，并通过 `getTools()` / `assembleToolPool()` 暴露给模型。
- `src/utils/permissions/classifierDecision.ts`：将 `TASK_UPDATE_TOOL_NAME` 列入自动分类器可识别的工具白名单。
- `src/utils/messages.ts`：在任务提醒（task reminder）附件逻辑中引用 `TASK_UPDATE_TOOL_NAME`，用于生成“请使用 TaskUpdate 更新任务状态”的系统提示。
- `src/utils/attachments.ts`：在计算 `turnsSinceLastTaskManagement` 时，将 `TaskUpdate` 与 `TaskCreate` 一同视为任务管理行为，用于决定何时向模型追加任务提醒附件。
- `src/utils/swarm/inProcessRunner.ts`：为 in-process teammate 构建工具白名单时，强制将 `TASK_UPDATE_TOOL_NAME` 加入允许列表，确保 teammate 始终可以更新任务。
- `src/constants/tools.ts`：在 `ASYNC_AGENT_ALLOWED_TOOLS`、`CUSTOM_AGENT_DISALLOWED_TOOLS` 等常量集合中引用该工具名。

### 下游被调用方

- `src/utils/tasks.ts`：所有任务持久化操作均委托给此模块。任务以 JSON 文件形式存储在 `~/.claude/tasks/{taskListId}/{taskId}.json`，并通过 `proper-lockfile` 实现文件级并发锁。
- `src/utils/hooks.ts`：任务完成前触发 `TaskCompleted` 事件 hook，允许外部脚本拦截或修改行为。
- `src/utils/teammateMailbox.ts`：所有权变更时向新 owner 的 inbox 文件写入 JSON 消息，实现异步通知。

## 风险、边界与改进建议

### 风险与边界

1. **删除操作的短路行为**：当 `status === 'deleted'` 时，`call()` 立即返回，**不会**处理同一次调用中同时传入的 `subject`、`description`、`metadata` 等其他字段更新。这意味着模型若在一次调用中既改字段又删任务，字段修改会被静默丢弃。虽然模型通常不会这样做，但 schema 并未禁止这种组合。
2. **Hook 阻塞的 UX 成本**：`executeTaskCompletedHooks` 是同步等待的生成器遍历，若用户配置了耗时较长的 shell hook（默认超时 10 分钟），任务完成操作会被长时间挂起，且当前没有进度 UI 反馈给最终用户。
3. **Verification Nudge 的脆弱启发式**：判断条件依赖任务标题正则 `/verif/i`，若用户将验证任务命名为 "Check quality" 而非包含 "verify" 的词汇，nudge 会误触发；反之，若普通任务标题恰好包含 "verification" 字样，则会漏触发。
4. **Metadata 合并的浅层限制**：`metadata` 只支持一级键值 merge，无法对嵌套对象做深度合并，也无法原子性地“替换整个 metadata 对象”（只能逐键设置或删除）。
5. **依赖关系只能增不能删**：当前 schema 仅提供 `addBlocks` / `addBlockedBy`，没有 `removeBlocks` / `removeBlockedBy`。若模型需要解除任务依赖，只能删除并重建任务，或通过其他工具/手动修改文件实现。
6. **并发安全声明与实际锁粒度**：`isConcurrencySafe()` 返回 `true`，但真正的并发控制下沉在 `utils/tasks.ts` 的 `proper-lockfile` 中。若 `blockTask` 内部锁与 `updateTask` 锁在极端并发下产生死锁风险（目前代码通过统一的 `LOCK_OPTIONS` 和任务文件锁机制已规避，但仍需关注）。

### 改进建议

1. **明确拒绝非法组合输入**：在 `status === 'deleted'` 且同时存在其他字段更新时，返回更明确的错误信息（如 "Cannot update fields and delete in the same call"），或改为先应用字段更新再删除，以符合最小惊讶原则。
2. **为 TaskCompleted Hook 增加进度反馈**：在 hook 执行期间向 `onProgress` 发送进度事件，让 UI 显示 "Running task-completed hooks..."，改善长耗时 hook 的用户体验。
3. **引入结构化验证标记**：在 `TaskSchema` 中增加显式的 `verification` 布尔字段或 `tags` 数组，取代基于标题正则的启发式判断，使 verification nudge 更可靠。
4. **支持依赖关系删除**：扩展输入 schema，增加 `removeBlocks` / `removeBlockedBy` 字段，并在 `utils/tasks.ts` 中提供对应的 `unblockTask` 函数，完善任务依赖管理能力。
5. **Metadata 操作增强**：考虑增加 `replaceMetadata` 布尔开关，允许模型选择是 merge 还是全量替换 metadata，以支持更复杂的元数据使用场景。
