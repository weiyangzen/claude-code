# runAgent.ts 深度研究文档

## 1. 场景与职责

`runAgent.ts` 是 Claude Code 中**子代理（subagent）实际执行推理循环的核心引擎**。它负责将一个 `AgentDefinition` 转换为可运行的 LLM query 迭代器，管理 Agent 生命周期内的所有状态初始化、工具过滤、MCP 服务器连接、skill 预加载、消息录制和最终清理。

该模块被以下场景调用：
- **AgentTool.tsx**：用户通过 `Agent` 工具首次 spawn 子代理（同步或异步）。
- **resumeAgent.ts**：恢复一个已停止的后台 Agent。
- **fork 子代理路径**：通过 `forkSubagent.ts` 构建的隐式 fork 子代理。
- **其他内部模块**：如 `MagicDocs`、`swarm/inProcessRunner` 等需要执行独立推理循环的组件。

职责边界：
- 不负责解析用户输入或选择 Agent 类型（由调用方完成）。
- 不负责后台任务进度追踪和通知（由 `runAsyncAgentLifecycle` 在 `agentToolUtils.ts` 中完成）。
- 专注于**单轮 Agent 执行**：从初始消息到 `query()` 迭代结束，返回所有产生的 `Message`。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| **Agent MCP 服务器初始化** | 为 Agent  frontmatter 中声明的 MCP 服务器建立连接，并在 Agent 结束时清理；支持引用已有服务器或内联定义新服务器。 |
| **工具池解析与过滤** | 根据 Agent 定义中的 `tools`/`disallowedTools` 以及同步/异步/内置/自定义属性，从可用工具池中解析出实际授予 Agent 的工具列表。 |
| **权限上下文重写** | 通过包装 `getAppState()` 动态调整 `toolPermissionContext.mode`、`shouldAvoidPermissionPrompts`、`awaitAutomatedChecksBeforeDialog` 和 `allowedTools`。 |
| **System Prompt 构建** | 调用 Agent 定义的 `getSystemPrompt()`，并附加环境详情（`enhanceSystemPromptWithEnvDetails`），生成最终发送给模型的 system prompt。 |
| **上下文消息 fork** | 支持 `forkContextMessages` 参数，使子代理继承父代理的完整对话前缀（用于 fork 子代理的 prompt cache 共享）。 |
| **Skill 预加载** | 解析 Agent frontmatter 中的 `skills` 列表，将对应的 prompt-based skill 内容作为 meta user message 注入到初始消息中。 |
| **Sidechain Transcript 录制** | 将 Agent 的每一轮消息实时写入 `subagents/<agentId>.jsonl`，供后续恢复、UI 查看和输出文件生成。 |
| **清理与资源释放** | 在 `finally` 中统一关闭 MCP 连接、清除 session hooks、释放文件状态缓存、停止 Perfetto tracing、清理 todo 条目、杀死后台 shell 任务。 |

---

## 3. 具体技术实现

### 3.1 入口函数 `runAgent`

签名（简化）：
```typescript
export async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  canUseTool,
  isAsync,
  forkContextMessages,
  querySource,
  override,
  model,
  maxTurns,
  availableTools,
  allowedTools,
  onCacheSafeParams,
  contentReplacementState,
  useExactTools,
  worktreePath,
  description,
  transcriptSubdir,
  onQueryProgress,
  ...
}: AgentRunParams): AsyncGenerator<Message, void>
```

核心执行流程：

#### 1) Agent ID 与 Transcript 子目录
- `agentId = override?.agentId ?? createAgentId()`
- 若 `transcriptSubdir` 存在（如 workflow 子代理），调用 `setAgentTranscriptSubdir(agentId, transcriptSubdir)` 将 transcript 写入 `subagents/<subdir>/<agentId>.jsonl`。

#### 2) Perfetto Tracing 注册
- 若 tracing 启用，注册 Agent 节点：`registerPerfettoAgent(agentId, agentDefinition.agentType, parentId)`。

#### 3) Fork 上下文消息处理
```typescript
const contextMessages = forkContextMessages
  ? filterIncompleteToolCalls(forkContextMessages)
  : []
const initialMessages = [...contextMessages, ...promptMessages]
```
- `filterIncompleteToolCalls`：遍历消息，收集所有 `tool_result` 对应的 `tool_use_id`，过滤掉包含未匹配 `tool_use` 的 assistant 消息，防止 API 400。

#### 4) 文件状态缓存
```typescript
const agentReadFileState = forkContextMessages !== undefined
  ? cloneFileStateCache(toolUseContext.readFileState)
  : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)
```
- Fork 子代理继承父级缓存以保持一致性；非 fork 创建全新缓存。

#### 5) 用户/系统上下文解析
- 读取 `override?.userContext ?? getUserContext()` 和 `override?.systemContext ?? getSystemContext()`。
- **CLAUDE.md 省略优化**：对 `omitClaudeMd` 为 true 的 Agent（如 Explore、Plan），在 `tengu_slim_subagent_claudemd` 开关开启时从 `userContext` 中移除 `claudeMd`。
- **Git 状态省略优化**：对 Explore/Plan Agent，从 `systemContext` 中移除 `gitStatus`，避免传输 stale 数据。

#### 6) 权限上下文包装 (`agentGetAppState`)
通过闭包返回一个修改后的 `getAppState`：
- **Mode 覆盖**：若 Agent 定义了 `permissionMode` 且父级不是 `bypassPermissions`/`acceptEdits`/`auto`（在 classifier 开启时），则覆盖 mode。
- **避免权限弹窗**：异步 Agent 默认设置 `shouldAvoidPermissionPrompts: true`；`bubble` mode 或显式 `canShowPermissionPrompts=true` 时例外。
- **自动检查前置**：异步 Agent 且允许弹窗时，设置 `awaitAutomatedChecksBeforeDialog: true`。
- **AllowedTools 作用域隔离**：若调用方传入 `allowedTools`，将其设为 session-level 允许规则，同时保留 CLI 传入的 `cliArg` 规则。
- **Effort 覆盖**：若 Agent 定义了 `effort`，覆盖父级的 `effortValue`。

#### 7) 工具解析
```typescript
const resolvedTools = useExactTools
  ? availableTools
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
```
- `useExactTools=true` 时跳过 `resolveAgentTools`，直接使用调用方提供的工具池（fork 路径用于保证 cache-identical）。

#### 8) System Prompt 构建
```typescript
const agentSystemPrompt = override?.systemPrompt
  ? override.systemPrompt
  : asSystemPrompt(await getAgentSystemPrompt(agentDefinition, toolUseContext, ...))
```
- `getAgentSystemPrompt` 内部调用 `agentDefinition.getSystemPrompt({ toolUseContext })`，再经 `enhanceSystemPromptWithEnvDetails` 附加环境信息。
- 若 Agent 的 `getSystemPrompt` 抛异常，回退到 `DEFAULT_AGENT_PROMPT`。

#### 9) AbortController 决策
```typescript
const agentAbortController = override?.abortController
  ? override.abortController
  : isAsync
    ? new AbortController()
    : toolUseContext.abortController
```
- 异步 Agent 获得独立的 `AbortController`；同步 Agent 与父级共享。

#### 10) SubagentStart Hooks 执行
- 遍历 `executeSubagentStartHooks(agentId, agentDefinition.agentType, signal)`。
- 若 hook 返回 `additionalContexts`，构造 `attachment` 类型的 `hook_additional_context` 消息追加到 `initialMessages`。

#### 11) Frontmatter Hooks 注册
- 若 Agent 定义了 `hooks` 且未被 `strictPluginOnlyCustomization` 拦截（或 Agent 来源为 admin-trusted），调用 `registerFrontmatterHooks(rootSetAppState, agentId, agentDefinition.hooks, ..., true)`。
- `isAgent=true` 会将 `Stop` hook 自动映射为 `SubagentStop`。

#### 12) Skill 预加载
- 读取 `agentDefinition.skills`，通过 `getSkillToolCommands` 获取所有可用 skills。
- `resolveSkillName` 尝试三种匹配策略：
  1. 精确匹配（含别名）。
  2. 加上 Agent 的 plugin prefix（如 `my-plugin:my-skill`）。
  3. 后缀匹配（`:skillName`）。
- 仅加载 `type === 'prompt'` 的 skill，将内容作为 meta user message 注入。

#### 13) Agent MCP 服务器初始化
```typescript
const { clients: mergedMcpClients, tools: agentMcpTools, cleanup: mcpCleanup } =
  await initializeAgentMcpServers(agentDefinition, toolUseContext.options.mcpClients)
```
- 对 `agentDefinition.mcpServers` 中的每个 spec：
  - `string` 类型：通过 `getMcpConfigByName` 查找已有配置，调用 `connectToServer`（带 memoization，可能复用父级连接）。
  - `{ [name]: config }` 类型：视为内联定义，新建 `ScopedMcpServerConfig`，标记 `isNewlyCreated=true`。
- 仅对 `newlyCreatedClients` 在清理时调用 `client.cleanup()`；共享连接不清理。
- `agentMcpTools` 通过 `uniqBy([...resolvedTools, ...agentMcpTools], 'name')` 去重合并。

#### 14) 子代理上下文创建
```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  readFileState: agentReadFileState,
  abortController: agentAbortController,
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,
  shareSetResponseLength: true,
  criticalSystemReminder_EXPERIMENTAL: agentDefinition.criticalSystemReminder_EXPERIMENTAL,
  contentReplacementState,
})
```
- `shareSetAppState`：同步 Agent 与父级共享状态更新；异步 Agent 隔离（`setAppState` 为 no-op）。
- `shareSetResponseLength`：无论同步异步，都向父级上报响应长度指标。

#### 15) CacheSafeParams 暴露
- 若调用方提供了 `onCacheSafeParams`，在子代理上下文构建完成后回调，供后台 summarization 服务 fork 对话生成进度摘要。

#### 16) 初始持久化（fire-and-forget）
```typescript
void recordSidechainTranscript(initialMessages, agentId).catch(...)
void writeAgentMetadata(agentId, { agentType, worktreePath, description }).catch(...)
```
- 写入失败仅打 debug log，不阻塞 Agent 启动。

#### 17) Query 循环
```typescript
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns: maxTurns ?? agentDefinition.maxTurns,
})) {
  // ...
}
```
- 对每条消息：
  - `stream_event` + `message_start` 且含 `ttftMs`：通过 `toolUseContext.pushApiMetricsEntry` 上报到父级 metrics。
  - `attachment` 类型（如 `max_turns_reached`）：直接 `yield`，不录制到 transcript。
  - `isRecordableMessage`（assistant/user/progress/system compact_boundary）：调用 `recordSidechainTranscript([message], agentId, lastRecordedUuid)` 增量写入，并更新 `lastRecordedUuid`。

#### 18) 正常结束与回调
- 若循环正常结束且未被 abort，对内置 Agent 调用 `agentDefinition.callback()`（如某些 one-shot agent 的收尾动作）。

#### 19) Finally 清理
`finally` 块执行以下清理（顺序重要）：
1. `mcpCleanup()` — 关闭 Agent 专属 MCP 连接。
2. `clearSessionHooks(rootSetAppState, agentId)` — 清除 frontmatter hooks。
3. `cleanupAgentTracking(agentId)` — 清除 prompt cache break 检测状态。
4. `agentToolUseContext.readFileState.clear()` — 释放文件缓存内存。
5. `initialMessages.length = 0` — 释放 fork context messages 引用。
6. `unregisterPerfettoAgent(agentId)` — 注销 tracing。
7. `clearAgentTranscriptSubdir(agentId)` — 清除 transcript 子目录映射。
8. 从 `AppState.todos` 中删除该 `agentId` 的 key，防止内存泄漏。
9. `killShellTasksForAgent(agentId, ...)` — 杀死后台 shell 任务，防止 PPID=1 僵尸进程。
10. 若 `feature('MONITOR_TOOL')` 开启，同时调用 `killMonitorMcpTasksForAgent`。

### 3.2 辅助函数

#### `initializeAgentMcpServers`
- 位置：`runAgent.ts:95-218`
- 负责将 Agent frontmatter 中的 `mcpServers` 解析为实际连接，区分共享连接（不清理）与内联连接（需清理）。
- 受 `strictPluginOnlyCustomization` 保护：非 admin-trusted 的 user-controlled agent 在 plugin-only 模式下会被跳过。

#### `filterIncompleteToolCalls`
- 位置：`runAgent.ts:866-904`
- 遍历所有消息，收集 `tool_result` 的 `tool_use_id`，然后过滤掉 assistant 消息中**所有** `tool_use` 都未找到对应 `tool_result` 的消息。
- 这是 fork 路径的关键前置步骤，因为 fork 会继承父级的完整 assistant message（含多个 `tool_use`），但某些 `tool_use` 的结果可能尚未返回。

#### `getAgentSystemPrompt`
- 位置：`runAgent.ts:906-932`
- 调用 `agentDefinition.getSystemPrompt({ toolUseContext })`，捕获异常后回退到 `DEFAULT_AGENT_PROMPT`。
- 最终通过 `enhanceSystemPromptWithEnvDetails` 附加当前环境信息（如 cwd、附加工作目录、启用的工具列表 emoji 提示等）。

#### `resolveSkillName`
- 位置：`runAgent.ts:945-973`
- 三级回退策略解析 frontmatter 中声明的 skill 名称到实际注册的 command name。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件
- `src/tools/AgentTool/runAgent.ts`（973 行）

### 4.2 直接调用方
- `src/tools/AgentTool/AgentTool.tsx` — Agent 工具 spawn 同步/异步 Agent。
- `src/tools/AgentTool/resumeAgent.ts` — 恢复已停止的后台 Agent。
- `src/services/MagicDocs/magicDocs.ts` — MagicDocs 内部 Agent 执行。
- `src/utils/swarm/inProcessRunner.ts` — In-process teammate 的推理循环。
- `src/utils/processUserInput/processSlashCommand.tsx` — 部分 slash command 的 fork 执行。

### 4.3 核心依赖文件
| 文件 | 作用 |
|------|------|
| `src/query.ts` | `query()` — 核心 LLM 推理循环，处理 tool use、streaming、compact 等。 |
| `src/Tool.ts` | `ToolUseContext`、`Tools`、`toolMatchesName` 类型与工具匹配。 |
| `src/tools/AgentTool/agentToolUtils.ts` | `resolveAgentTools` — 工具过滤与解析。 |
| `src/tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition`、`isBuiltInAgent` — Agent 类型定义。 |
| `src/utils/forkedAgent.ts` | `createSubagentContext`、`CacheSafeParams` — 子代理上下文隔离。 |
| `src/utils/messages.ts` | `createUserMessage`、`filterIncompleteToolCalls`（被引用逻辑类似）等消息工具。 |
| `src/utils/sessionStorage.ts` | `recordSidechainTranscript`、`writeAgentMetadata`、`clearAgentTranscriptSubdir` — transcript 与元数据持久化。 |
| `src/utils/toolResultStorage.ts` | `ContentReplacementState`、`applyToolResultBudget` — 大工具结果替换状态。 |
| `src/utils/fileStateCache.ts` | `cloneFileStateCache`、`createFileStateCacheWithSizeLimit` — 文件读取状态缓存。 |
| `src/utils/model/agent.ts` | `getAgentModel` — 模型解析。 |
| `src/utils/permissions/permissionRuleParser.ts` | `permissionRuleValueFromString` — 工具规则解析（间接通过 `resolveAgentTools`）。 |
| `src/utils/hooks/registerFrontmatterHooks.ts` | `registerFrontmatterHooks` — Agent 生命周期 hooks 注册。 |
| `src/utils/hooks.ts` | `executeSubagentStartHooks` — SubagentStart hooks 执行。 |
| `src/services/mcp/client.ts` | `connectToServer`、`fetchToolsForClient` — MCP 连接与工具获取。 |
| `src/services/mcp/config.ts` | `getMcpConfigByName` — MCP 配置查找。 |
| `src/tasks/LocalShellTask/killShellTasks.ts` | `killShellTasksForAgent` — 杀死后台 shell 任务。 |
| `src/utils/telemetry/perfettoTracing.ts` | `registerPerfettoAgent`、`unregisterPerfettoAgent` — Tracing 注册。 |
| `src/tasks/MonitorMcpTask/MonitorMcpTask.js` | `killMonitorMcpTasksForAgent` — Monitor MCP 任务清理（require 动态加载）。 |

---

## 5. 依赖与外部交互

### 5.1 文件系统交互
- **读取/写入**：`~/.claude/projects/<sanitized_cwd>/<sessionId>/subagents/[<subdir>/]<agentId>.jsonl`
- **写入**：`~/.claude/projects/<sanitized_cwd>/<sessionId>/subagents/<agentId>-metadata.json`
- **工具结果持久化**：通过 `toolUseContext` 间接使用 `src/utils/toolResultStorage.ts` 写入 `tool-results/` 子目录。

### 5.2 网络/API 交互
- 通过 `query()` 与 Anthropic/Bedrock/Vertex 等 LLM API 建立长连接（streaming）。
- MCP 服务器连接：通过 `connectToServer` 启动 stdio/SSE 子进程或网络连接。

### 5.3 进程/环境交互
- `initializeAgentMcpServers` 可能启动新的子进程（MCP server）。
- `finally` 块调用 `killShellTasksForAgent`，向该 Agent 启动的后台 bash 任务发送终止信号。
- `process.env.CLAUDE_CODE_SUBAGENT_MODEL` 可全局覆盖子代理模型选择。

### 5.4 AppState 交互
- 通过 `rootSetAppState`（即 `toolUseContext.setAppStateForTasks ?? toolUseContext.setAppState`）写入：
  - 注册/清理 frontmatter hooks。
  - 删除 `todos[agentId]` 防止内存泄漏。
- 异步 Agent 的 `setAppState` 在 `createSubagentContext` 中被替换为 no-op，但 `setAppStateForTasks` 始终指向根 store，保证任务注册和清理可达。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 代码位置 |
|------|------|----------|
| **MCP 连接泄漏** | `mcpCleanup` 仅清理 `newlyCreatedClients`；若 `connectToServer` 对 string 引用返回了非 memoized 的新连接，则不会清理。 | `runAgent.ts:131-210` |
| **Skill 加载循环依赖** | `formatSkillLoadingMetadata` 通过动态 `import()` 加载 `processSlashCommand.js`，在大量并发 spawn 时可能触发多次模块加载。 | `runAgent.ts:618-645` |
| **Query 异常穿透** | `query()` 抛出的异常（如网络超时、API 限流）会中断 `for await...of` 并直接进入 `finally`；虽然会清理资源，但调用方（如 `runAsyncAgentLifecycle`）需要正确区分 `AbortError` 与其他错误。 | `runAgent.ts:747-810` |
| **Frontmatter Hooks 权限绕过** | `hooksAllowedForThisAgent` 的判断仅基于 `isRestrictedToPluginOnly('hooks')` 和 `isSourceAdminTrusted`；若未来增加更细粒度的 hook 权限模型，此处逻辑需要同步更新。 | `runAgent.ts:564-575` |
| **ToolResult 预算未在 runAgent 内应用** | `applyToolResultBudget` 在 `query.ts` 中调用，不在 `runAgent.ts` 中；若未来需要按 Agent 粒度设置预算，修改点分散。 | 间接依赖 `query.ts` |
| **MonitorMcpTask 动态 require** | `finally` 中对 `MonitorMcpTask.js` 使用 `require()` 动态加载，若模块不存在或加载失败会抛错（虽然被包裹在条件中）。 | `runAgent.ts:849-858` |

### 6.2 边界行为

- **`useExactTools=true` 的 cache 保证**：fork 子代理通过此标志跳过 `resolveAgentTools`，并继承父级的 `thinkingConfig` 和 `isNonInteractiveSession`，确保 API 请求前缀字节级一致。
- **`preserveToolUseResults`**：当调用方设置此标志（如 in-process teammates），`agentToolUseContext.preserveToolUseResults = true`，`query()` 会保留 `toolUseResult` 块在消息中，使 transcript 可被外部查看。
- **`maxTurns` 终止信号**：当 `query()` 返回 `attachment` 类型且 `attachment.type === 'max_turns_reached'` 时，`runAgent` 会 `break` 退出循环，不再继续迭代。
- **同步 Agent 的 `finally` 清理**：同步 Agent 与父级共享 `abortController`，但 `finally` 仍会尝试 `unregisterPerfettoAgent` 和 `clearAgentTranscriptSubdir`。若同步 Agent 被嵌套多次调用，Perfetto 注册/注销需保证幂等。

### 6.3 改进建议

| 优先级 | 建议 | 理由 |
|--------|------|------|
| **高** | 将 `initializeAgentMcpServers` 提取为独立模块 | 当前 `runAgent.ts` 已接近 1000 行，MCP 初始化逻辑占 120+ 行，独立后可降低主文件复杂度并便于单元测试。 |
| **高** | 为 `filterIncompleteToolCalls` 增加单元测试 | 该函数是 fork 路径正确性的关键，但当前无直接测试覆盖各种 `tool_use`/`tool_result` 排列组合。 |
| **中** | 统一 `finally` 中的清理顺序文档化 | 当前 10 项清理散落在 `finally` 中，建议提取为 `cleanupAgentResources(agentId, ...)` 并内联注释说明顺序依赖（如必须先 `mcpCleanup` 再 `clearSessionHooks`）。 |
| **中** | 将 `resolveSkillName` 提取到 `src/commands.ts` 或独立工具模块 | 该函数不仅 Agent 使用，slash command 也可能需要类似的 skill 名称解析逻辑，避免重复实现。 |
| **低** | 考虑在 `runAgent` 返回的 generator 上暴露 `agentId` | 调用方有时需要知道最终使用的 `agentId`（尤其是 `override.agentId` 未传时由 `createAgentId()` 生成），目前只能预先计算或事后推断。 |
| **低** | 对 `agentDefinition.getSystemPrompt()` 的异常增加结构化日志 | 当前仅 `logForDebugging` 打印字符串，建议增加 `logEvent` 埋点以便监控特定 Agent 的 system prompt 构建失败率。 |

