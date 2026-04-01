# 研究文档：src/utils/forkedAgent.ts

## 场景与职责

`forkedAgent.ts` 是 Claude Code 的**子代理（forked agent / subagent）运行基础设施**。它封装了一个完整的 `query()` 调用循环，用于在隔离的上下文中执行后台任务，同时通过与父代理共享**缓存安全参数（CacheSafeParams）**来最大化 Anthropic API 的 prompt cache 命中率。

典型使用场景包括：
- `/btw` 侧边问题（`sideQuestion.ts`）
- 会话记忆总结（`sessionMemory.ts`）
- 自动 compact 摘要（`compact.ts`）
- AgentTool 的异步子代理（`AgentTool.tsx`, `runAgent.ts`）
- 推测执行（`speculation.ts`）
- `autoDream`、`agentSummary`、`extractMemories` 等后台服务

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `CacheSafeParams` | 定义了必须与父请求完全一致才能共享 prompt cache 的参数集合。 |
| `saveCacheSafeParams` / `getLastCacheSafeParams` | 在 stop hooks 中保存/读取最后一次的缓存安全参数，供 post-turn fork 使用。 |
| `createCacheSafeParams(context)` | 从 `REPLHookContext` 构造 `CacheSafeParams`。 |
| `createSubagentContext(parentContext, overrides?)` | 创建隔离的 `ToolUseContext`，默认克隆可变状态、切断 UI 回调。 |
| `createGetAppStateWithAllowedTools` | 为子代理动态注入允许使用的工具白名单。 |
| `prepareForkedCommandContext` | 为 SkillTool / slash command 的 fork 执行准备通用上下文。 |
| `runForkedAgent(params)` | 运行子代理的 query 循环，累积 usage，记录 transcript，上报 analytics。 |
| `extractResultText` | 从子代理消息中提取最后 assistant 的文本结果。 |

## 具体技术实现

### CacheSafeParams 的构成

```ts
export type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]
}
```

- Anthropic 的 cache key 由 system prompt、tools、model、messages prefix、thinking config 组成。
- `CacheSafeParams` 携带前四项；thinking config 则继承自父上下文的 `toolUseContext.options.thinkingConfig`。
- 注释特别警告：**修改 `maxOutputTokens` 会通过 `claude.ts` 的 clamping 改变 `budget_tokens`，从而破坏 cache key**。

### 子代理上下文隔离（`createSubagentContext`）

默认隔离策略：

| 字段 | 默认行为 | 说明 |
|------|----------|------|
| `readFileState` | `cloneFileStateCache(parent)` | 防止子代理污染父代理的文件状态缓存。 |
| `abortController` | `createChildAbortController(parent)` | 父代理 abort 会级联到子代理，但子代理自己的 abort 不影响父代理。 |
| `getAppState` | 包装后返回 `shouldAvoidPermissionPrompts = true` | 非交互式子代理自动跳过权限弹窗。 |
| `setAppState` | no-op | 防止子代理修改父代理的 React state。 |
| `setAppStateForTasks` | 始终指向父代理 | 确保后台 bash 任务注册和清理能到达根 store。 |
| `localDenialTracking` | 新建独立状态 / 共享父状态 | 若 `shareSetAppState` 为 false，则新建，以在 no-op setAppState 时仍能累积拒绝计数。 |
| `contentReplacementState` | clone | 保证 cache-sharing fork 对 parent tool_use_id 的替换决策与父代理一致。 |
| `messages` | copy | 子代理从此数组继续。 |
| `agentId` | `createAgentId()` | 每个子代理拥有独立 ID。 |
| `queryTracking` | 新 `chainId` + `depth + 1` | 用于嵌套深度监控和 analytics。 |
| UI 回调 | `undefined` 或 no-op | 子代理不能控制父代理的 UI。 |

通过 `overrides` 可以显式覆盖上述行为，例如 `AgentTool` 会传入自定义 `options` 和 `agentId`。

### runForkedAgent 的执行流程

1. **构造隔离上下文**：`createSubagentContext(toolUseContext, overrides)`
2. **构造初始消息**：`[...forkContextMessages, ...promptMessages]`
   - 不调用 `filterIncompleteToolCalls`（注释说明这会破坏 tool pairing，由下游 `ensureToolResultPairing` 统一修复）。
3. **创建 agentId 并记录初始 transcript**（若 `skipTranscript` 为 false）。
4. **进入 `for await...of query()` 循环**：
   - `stream_event` 类型且 `message_delta` 含 usage → 累加到 `totalUsage`。
   - `stream_request_start` → 跳过。
   - 其他消息类型 → 推入 `outputMessages`，可选调用 `onMessage` 回调，并逐条记录到 sidechain transcript。
5. **finally 清理**：
   - `readFileState.clear()` 释放克隆缓存。
   - `initialMessages.length = 0` 释放消息数组引用。
6. **Analytics 上报**：计算 cache hit rate，发送 `tengu_fork_agent_query` 事件。

### prepareForkedCommandContext

为 SkillTool 和 slash command 提供统一的 fork 前准备：
1. 调用 `command.getPromptForCommand(args, context)` 获取 skill prompt。
2. 将 prompt block 拼接为纯文本 `skillContent`。
3. 解析 `command.allowedTools` 并通过 `createGetAppStateWithAllowedTools` 注入权限上下文。
4. 选择 agent（`command.agent` → `general-purpose` → 第一个可用 agent）。
5. 构造初始 user message。

## 关键代码路径与文件引用

### 调用方

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/utils/sideQuestion.ts` | `runForkedAgent` | `/btw` 侧边问题。 |
| `src/services/compact/compact.ts` | `runForkedAgent` | 上下文压缩摘要。 |
| `src/services/SessionMemory/sessionMemory.ts` | `runForkedAgent` | 会话记忆生成。 |
| `src/services/PromptSuggestion/speculation.ts` | `runForkedAgent` | 推测执行。 |
| `src/services/PromptSuggestion/promptSuggestion.ts` | `runForkedAgent` | 提示建议。 |
| `src/services/autoDream/autoDream.ts` | `runForkedAgent` | 自动 dream。 |
| `src/services/extractMemories/extractMemories.ts` | `runForkedAgent` | 记忆提取。 |
| `src/services/AgentSummary/agentSummary.ts` | `runForkedAgent` | Agent 摘要。 |
| `src/commands/btw/btw.tsx` | `runForkedAgent` | `/btw` 命令入口。 |
| `src/tools/AgentTool/AgentTool.tsx` | `createSubagentContext`, `runForkedAgent` | 异步 Agent 子代理。 |
| `src/tools/AgentTool/runAgent.ts` | `createSubagentContext`, `runForkedAgent` | Agent 运行器。 |
| `src/tools/SkillTool/SkillTool.ts` | `prepareForkedCommandContext`, `runForkedAgent` | Skill 执行。 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `prepareForkedCommandContext` | Slash 命令 fork。 |
| `src/utils/swarm/inProcessRunner.ts` | `createSubagentContext` | Swarm 进程内运行器。 |
| `src/query/stopHooks.ts` | `saveCacheSafeParams` | 每轮结束后保存 cache 参数。 |
| `src/utils/queryContext.ts` | `CacheSafeParams` (type) | 构建 fallback cache 参数。 |

### 被调用方

- `src/query.js`：`query`（核心对话循环）
- `src/services/analytics/index.js`：`logEvent`
- `src/services/api/claude.js`：`accumulateUsage`, `updateUsage`
- `src/services/api/logging.js`：`EMPTY_USAGE`, `NonNullableUsage`
- `src/utils/sessionStorage.js`：`recordSidechainTranscript`
- `src/utils/fileStateCache.js`：`cloneFileStateCache`
- `src/utils/toolResultStorage.js`：`cloneContentReplacementState`
- `src/utils/permissions/denialTracking.js`：`createDenialTrackingState`
- `src/utils/permissions/permissionSetup.js`：`parseToolListFromCLI`
- `src/utils/abortController.js`：`createChildAbortController`
- `src/utils/messages.js`：`createUserMessage`, `extractTextContent`, `getLastAssistantMessage`
- `src/utils/uuid.js`：`createAgentId`
- `src/utils/debug.js`：`logForDebugging`

## 依赖与外部交互

- **Analytics**：每个 fork 完成后上报 `tengu_fork_agent_query`，包含完整的 `NonNullableUsage` 和 cache hit rate。
- **Session Storage**：通过 `recordSidechainTranscript` 将子代理对话写入独立的 sidechain transcript（以 `agentId` 标识），便于 resume 和调试。
- **API Cache**：与 Anthropic prompt cache 机制深度耦合，任何破坏 `CacheSafeParams` 一致性的修改都会直接降低子代理的缓存命中率。

## 风险、边界与改进建议

### 风险

1. **Cache 失效陷阱**：`maxOutputTokens` 会间接改变 thinking budget，从而破坏 cache key。虽然代码中有大段注释警告，但调用方仍可能误用。
2. **消息泄漏**：`forkContextMessages` 直接 spread 到 `initialMessages`，若父消息包含敏感内容（如用户粘贴的密码），子代理默认会全部看到。`overrides.messages` 可部分缓解，但无自动过滤机制。
3. **Transcript 记录失败**：`recordSidechainTranscript` 的调用都是 fire-and-forget（`void ... .catch`），若磁盘满或权限问题导致持续失败，用户无感知。
4. **深度递归**：`queryTracking.depth` 只记录不限制，理论上可能出现极深层嵌套 fork，导致消息链指数增长。
5. **UI 回调隔离的副作用**：`setToolJSX: undefined` 意味着子代理中的工具无法展示进度 JSX，某些需要用户确认的长运行工具（如 SleepTool 的进度条）在子代理中可能表现异常。

### 边界

- `skipTranscript` 为 true 时，不创建 `agentId`，也不写入 sidechain，适用于一次性推测执行。
- `skipCacheWrite` 为 true 时，最后一轮消息不写入 prompt cache，适用于 fire-and-forget 场景（如 `/btw`）。
- `maxTurns` 默认未设置，由调用方控制；`sideQuestion` 设为 1 以强制单轮。

### 改进建议

1. **maxOutputTokens 自动校验**：在 `runForkedAgent` 入口处增加运行时断言，若 `maxOutputTokens` 与父上下文的 `max_tokens` 不一致且 `skipCacheWrite` 为 false，则发出 `logForDebugging` 警告。
2. **深度限制**：增加 `MAX_FORK_DEPTH` 常量，在 `depth` 超过阈值时抛出错误或降级为直接 API 调用，防止无限嵌套。
3. **敏感信息过滤**：为 `forkContextMessages` 提供可选的 redaction hook，自动剥离或替换 messages 中的高敏感内容块（如 tool_result 中的环境变量）。
4. **Transcript 失败可观测性**：将 `recordSidechainTranscript` 的 catch 日志级别从 debug 提升到 warn，并增加 analytics 事件。
5. **测试覆盖**：当前无直接单元测试。建议 mock `query()` 和 `recordSidechainTranscript`，对 `createSubagentContext` 的隔离行为、`runForkedAgent` 的 usage 累积逻辑做单元测试。
