# 研究文档：src/utils/processUserInput/processSlashCommand.tsx

> 研究范围：目标文件及其直接/间接依赖（调用链、类型定义、命令注册、子代理执行、权限系统、消息构造、进度 UI）。不含 README/Docs 等文档内容。

---

## 1. 场景与职责

`processSlashCommand.tsx` 是 **Claude Code 用户输入处理链路中负责“斜杠命令”解析与执行的核心调度器**。当用户在输入框键入以 `/` 开头的指令（如 `/commit`、`/compact`、`/config`）时，该模块决定：

1. **命令是否存在**（内置命令、插件命令、技能、MCP 命令）。
2. **命令类型**（`local-jsx`、`local`、`prompt`）以及对应的执行策略。
3. **执行环境**（主线程 inline 展开、同步 fork 子代理、后台异步 fork 子代理）。
4. **返回消息结构**（是否继续调用大模型 `shouldQuery`、附加 `allowedTools` / `model` / `effort` 覆盖、本地命令的 `resultText`）。

该文件处于 `processUserInput.ts` → `handlePromptSubmit.ts` 调用链的下游，所有斜杠命令最终都会收敛到这里。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| **命令解析与校验** | 通过 `parseSlashCommand` 提取 `commandName` + `args`，区分 MCP 命令；对未知命令进行容错（若像文件路径则降级为普通 prompt）。 |
| **三种命令类型的统一入口** | `local-jsx`（渲染 Ink UI）、`local`（执行本地 JS/TS 函数并返回文本/compact 结果）、`prompt`（展开 skill 内容到对话或 fork 子代理）。 |
| **Fork 子代理执行** | 对 `context: 'fork'` 的 prompt 命令，启动独立 agent（`runAgent`），支持同步（阻塞 UI，带进度条）和后台（KAIROS 模式 fire-and-forget）两种形态。 |
| **权限与工具注入** | 将命令声明的 `allowedTools` 解析后注入子代理的权限上下文；通过 `command_permissions` attachment 告知主模型。 |
| **遥测与埋点** | 对每条斜杠命令记录 `tengu_input_command`、`tengu_slash_command_forked`、`tengu_input_slash_invalid` 等事件，并附加插件/市场来源元数据。 |
| **消息标准化** | 统一构造 `synthetic caveat message`、`command input tags`、`local-command-stdout/stderr` 等消息格式，确保模型与 UI 层语义一致。 |

---

## 3. 具体技术实现

### 3.1 模块导出与类型

```ts
// 核心导出
export function processSlashCommand(...): Promise<ProcessUserInputBaseResult>
export function looksLikeCommand(commandName: string): boolean
export function formatSkillLoadingMetadata(skillName: string, ...): string
export function processPromptSlashCommand(...): Promise<SlashCommandResult>
```

- `SlashCommandResult` 在 `ProcessUserInputBaseResult` 基础上增加 `command: Command`，用于调用方识别实际触发的命令对象。
- `ProcessUserInputBaseResult` 定义于 `src/utils/processUserInput/processUserInput.ts`，包含 `messages`、`shouldQuery`、`allowedTools`、`model`、`effort`、`resultText`、`nextInput`、`submitNextInput`。

### 3.2 命令解析流程（processSlashCommand）

```
inputString
  └─ parseSlashCommand(inputString)
       ├─ null → 返回错误提示 "Commands are in the form `/command [args]`"
       └─ { commandName, args, isMcp }
            └─ hasCommand(commandName, context.options.commands)?
                 ├─ NO  → looksLikeCommand(commandName) && !isFilePath
                 │         ├─ true  → "Unknown skill: xxx"
                 │         └─ false → 降级为普通 user prompt（shouldQuery=true）
                 └─ YES → getMessagesForSlashCommand(...)
```

**关键细节**：
- `looksLikeCommand` 使用正则 `!/[a-zA-Z0-9:\-_]/.test(commandName)` 判断；若用户输入 `/tmp/foo`，会被识别为文件路径而降级为普通 prompt，避免误报“Unknown skill”。
- `sanitizedCommandName` 用于埋点：内置命令保留原名，非内置统一记为 `custom`，MCP 命令记为 `mcp`。

### 3.3 三种命令类型的执行分支（getMessagesForSlashCommand）

#### 3.3.1 `local-jsx`

- 调用 `command.load()` 动态加载模块（懒加载），然后 `mod.call(onDone, context, args)`。
- `onDone` 回调接收 `result` + `options`（`display: 'skip' | 'system' | 'user'`，`shouldQuery`，`metaMessages`，`nextInput`，`submitNextInput`）。
- 若命令返回 JSX，则通过 `setToolJSX({ jsx, shouldHidePromptInput: true, isLocalJSXCommand: true, isImmediate })` 渲染 Ink 组件（如 `/config` 的配置面板）。
- **防死锁机制**：若 `load()/call()` 抛异常且 `onDone` 未被调用，代码会主动 `resolve({ messages: [], shouldQuery: false })` 并清理 `setToolJSX`，防止 `queryGuard` 永久卡在 `dispatching`。
- **Fullscreen 优化**：在 fullscreen 模式下，若结果以 ` dismissed` 结尾，则跳过 transcript 记录，仅保留 meta messages。

#### 3.3.2 `local`

- 调用 `mod.call(args, context)` 获取 `LocalCommandResult`：
  - `type: 'skip'` → 无消息，不查询。
  - `type: 'compact'` → 将 slash 命令产生的消息追加到 `compactionResult.messagesToKeep`，调用 `buildPostCompactMessages` 生成 compact 后的消息流；同时 `resetMicrocompactState()`。
  - `type: 'text'` → 返回 `[userMessage, systemMessage(local-command-stdout)]`。
- 异常捕获后返回 `<local-command-stderr>${String(e)}</local-command-stderr>`。
- 敏感参数：`command.isSensitive` 为 true 时，args 在消息中显示为 `***`。

#### 3.3.3 `prompt`

- **Fork 路径**：若 `command.context === 'fork'`，进入 `executeForkedSlashCommand`。
- **Inline 路径**：否则进入 `getMessagesForPromptSlashCommand`，将 skill 内容展开为 `ContentBlockParam[]` 并注入对话。

### 3.4 Fork 子代理执行（executeForkedSlashCommand）

这是本文件**技术复杂度最高**的区域，涉及两种运行模式：

#### A. 后台异步模式（KAIROS + kairosEnabled）

```ts
if (feature('KAIROS') && (await context.getAppState()).kairosEnabled) {
  // 创建独立 abortController
  const bgAbortController = createAbortController();
  const spawnTimeWorkload = getWorkload(); // AsyncLocalStorage 捕获

  const enqueueResult = (value: string) => enqueuePendingNotification({
    value,
    mode: 'prompt',
    priority: 'later',
    isMeta: true,
    skipSlashCommands: true,
    workload: spawnTimeWorkload
  });

  void (async () => {
    // 1. 等待 MCP settle（轮询 200ms，最多 10s）
    // 2. 刷新 tools
    // 3. runAgent({ isAsync: true, ... })
    // 4. 结果以 <scheduled-task-result> XML 入队
  })();

  return { messages: [], shouldQuery: false, command };
}
```

**设计意图**：
- 在 assistant 模式下，N 个定时任务若在启动时同时触发，同步执行会阻塞主代理轮次。后台模式让子代理并行运行，完成后以 `isMeta` prompt 重新入队，主代理在后续轮次中通过 `SendUserMessage` 决定是否通知用户。
- `getWorkload()` 通过 `AsyncLocalStorage` 捕获，确保后台任务与后续入队结果具有相同的 workload 标签（归因）。
- MCP settle 机制：因为任务可能在 MCP 连接完成前触发，代码会轮询 `mcp.clients` 直到没有 `pending` 状态，再调用 `context.options.refreshTools?.()` 获取最新工具列表。

#### B. 同步模式（默认）

- 通过 `runAgent({ isAsync: false, ... })` 阻塞执行。
- 使用 `setToolJSX` + `renderToolUseProgressMessage` 实时展示子代理进度（`Initializing…` → 工具调用摘要 → token 统计）。
- 每收到一条 `assistant` 消息，累加 `contentLength` 到 `context.setResponseLength`（用于 UI token 计数）。
- 执行结束后通过 `extractResultText(agentMessages, 'Command completed')` 提取最终结果，包装为：
  ```
  userMessage: "/command args"
  userMessage: "<local-command-stdout>...result...</local-command-stdout>"
  ```
- ant-only 调试：会在 `resultText` 前追加 API dump 路径。

### 3.5 Prompt 命令 Inline 展开（getMessagesForPromptSlashCommand）

```ts
const result = await command.getPromptForCommand(args, context);

// 1. 注册 skill hooks（如果命令声明了 hooks）
if (command.hooks && hooksAllowedForThisSkill) {
  registerSkillHooks(context.setAppState, sessionId, command.hooks, command.name, command.skillRoot);
}

// 2. 记录 invoked skill（用于 compact 恢复）
addInvokedSkill(command.name, skillPath, skillContent, agentId);

// 3. 构造 metadata 消息（命令名/参数标签）
const metadata = formatCommandLoadingMetadata(command, args);

// 4. 提取附件（@-mentions、MCP resources、agent mentions）
const attachmentMessages = await toArray(getAttachmentMessages(..., { skipSkillDiscovery: true }));

// 5. 返回消息序列
[
  createUserMessage({ content: metadata, uuid }),
  createUserMessage({ content: mainMessageContent, isMeta: true }),
  ...attachmentMessages,
  createAttachmentMessage({ type: 'command_permissions', allowedTools: additionalAllowedTools, model: command.model })
]
```

**Coordinator 模式短路**：
- 若 `feature('COORDINATOR_MODE')` 且 `CLAUDE_CODE_COORDINATOR_MODE` 为真，并且当前不是子代理（`!context.agentId`），则不会真正加载 skill 内容，而是返回一段摘要，告诉 coordinator “将该 skill 委托给 worker 执行”。

### 3.6 关键数据结构

| 结构 | 来源文件 | 说明 |
|------|----------|------|
| `Command` / `PromptCommand` | `src/types/command.ts` | 命令基类型；`prompt` 类型包含 `getPromptForCommand`、`allowedTools`、`model`、`effort`、`context`、`agent`、`hooks` 等字段。 |
| `ProcessUserInputContext` | `src/utils/processUserInput/processUserInput.ts` | `ToolUseContext & LocalJSXCommandContext`，包含 `getAppState`、`setAppState`、`options`（tools、commands、model 等）。 |
| `SetToolJSXFn` | `src/Tool.ts` | 控制 Ink UI 层显示 JSX / Spinner / 隐藏输入框的回调。 |
| `CanUseToolFn` | `src/hooks/useCanUseTool.ts` | 权限检查函数签名，最终指向 `hasPermissionsToUseTool`。 |
| `CacheSafeParams` / `PreparedForkedContext` | `src/utils/forkedAgent.ts` | Fork 子代理所需的缓存安全参数与预计算上下文。 |
| `ProgressMessage<AgentProgress>` | `src/types/message.ts` | 子代理进度消息类型，被 `renderToolUseProgressMessage` 消费。 |

### 3.7 命令元数据格式化协议

文件内实现了三种元数据格式，用于 UI 显示“正在加载/执行某命令”：

1. **Slash Command 格式**（用户可直接调用的 skill）：
   ```xml
   <command-message>name</command-message>
   <command-name>/name</command-name>
   <command-args>args</command-args>
   ```
2. **Skill 格式**（仅模型可调用的 skill）：
   ```xml
   <command-message>name</command-message>
   <command-name>name</command-name>
   <skill-format>true</skill-format>
   ```
3. `formatCommandLoadingMetadata` 根据 `userInvocable` 和 `loadedFrom` 自动选择上述格式之一。

---

## 4. 关键代码路径与文件引用

### 4.1 调用链（上游 → 目标 → 下游）

```
src/screens/REPL.tsx
  └─ src/utils/handlePromptSubmit.ts
       └─ executeUserInput
            └─ src/utils/processUserInput/processUserInput.ts
                 └─ processUserInputBase
                      └─ (dynamic import) processSlashCommand.tsx  <-- 目标文件
                           ├─ executeForkedSlashCommand
                           │    └─ src/utils/forkedAgent.ts (prepareForkedCommandContext)
                           │    └─ src/tools/AgentTool/runAgent.ts (runAgent)
                           │    └─ src/tools/AgentTool/UI.tsx (renderToolUseProgressMessage)
                           ├─ getMessagesForPromptSlashCommand
                           │    └─ src/utils/attachments.ts (getAttachmentMessages)
                           │    └─ src/utils/messages.ts (createUserMessage, formatCommandInputTags)
                           │    └─ src/utils/permissions/permissionSetup.ts (parseToolListFromCLI)
                           └─ getMessagesForSlashCommand (local-jsx / local 分支)
                                └─ src/types/command.ts (Command 类型定义)
```

### 4.2 直接依赖清单（按功能分组）

| 功能 | 依赖文件 |
|------|----------|
| 命令注册与查找 | `src/commands.ts` (`builtInCommandNames`, `findCommand`, `getCommand`, `hasCommand`, `getCommandName`) |
| 命令类型定义 | `src/types/command.ts` (`Command`, `PromptCommand`, `CommandBase`, `LocalJSXCommandOnDone`, `CommandResultDisplay`) |
| 斜杠解析 | `src/utils/slashCommandParsing.ts` (`parseSlashCommand`, `ParsedSlashCommand`) |
| 消息构造 | `src/utils/messages.ts` (`createUserMessage`, `createSyntheticUserCaveatMessage`, `createSystemMessage`, `createUserInterruptionMessage`, `createCommandInputMessage`, `prepareUserContent`, `formatCommandInputTags`, `normalizeMessages`, `isCompactBoundaryMessage`, `isSystemLocalCommandMessage`) |
| Fork 上下文准备 | `src/utils/forkedAgent.ts` (`prepareForkedCommandContext`, `extractResultText`, `createGetAppStateWithAllowedTools`) |
| 子代理执行 | `src/tools/AgentTool/runAgent.ts` (`runAgent`) |
| 进度 UI | `src/tools/AgentTool/UI.tsx` (`renderToolUseProgressMessage`) |
| 附件提取 | `src/utils/attachments.ts` (`createAttachmentMessage`, `getAttachmentMessages`) |
| 权限检查 | `src/utils/permissions/permissions.ts` (`hasPermissionsToUseTool`) |
| 工具列表解析 | `src/utils/permissions/permissionSetup.ts` (`parseToolListFromCLI`) |
| 队列/后台通知 | `src/utils/messageQueueManager.ts` (`enqueuePendingNotification`) |
| 状态与 Session | `src/bootstrap/state.ts` (`setPromptId`, `addInvokedSkill`, `getSessionId`) |
| 遥测 | `src/services/analytics/index.ts` (`logEvent`)<br>`src/utils/telemetry/events.ts` (`logOTelEvent`, `redactIfDisabled`)<br>`src/utils/telemetry/pluginTelemetry.ts` (`buildPluginCommandTelemetryFields`) |
| 插件标识 | `src/utils/plugins/pluginIdentifier.ts` (`parsePluginIdentifier`, `isOfficialMarketplaceName`) |
| 插件策略 | `src/utils/settings/pluginOnlyPolicy.ts` (`isRestrictedToPluginOnly`, `isSourceAdminTrusted`) |
| Hooks 注册 | `src/utils/hooks/registerSkillHooks.ts` (`registerSkillHooks`) |
| 其他工具函数 | `src/utils/abortController.ts` (`createAbortController`)<br>`src/utils/agentContext.ts` (`getAgentContext`)<br>`src/utils/fullscreen.ts` (`isFullscreenEnvEnabled`)<br>`src/utils/generators.ts` (`toArray`)<br>`src/utils/tokens.ts` (`getAssistantMessageContentLength`)<br>`src/utils/uuid.ts` (`createAgentId`)<br>`src/utils/workloadContext.ts` (`getWorkload`)<br>`src/utils/sleep.ts` (`sleep`)<br>`src/utils/suggestions/skillUsageTracking.ts` (`recordSkillUsage`)<br>`src/utils/envUtils.ts` (`isEnvTruthy`)<br>`src/utils/errors.ts` (`AbortError`, `MalformedCommandError`)<br>`src/utils/file.ts` (`getDisplayPath`)<br>`src/utils/fsOperations.ts` (`getFsImplementation`)<br>`src/utils/debug.ts` (`logForDebugging`)<br>`src/utils/log.ts` (`logError`) |

---

## 5. 依赖与外部交互

### 5.1 与权限系统的交互

- **子代理权限提升**：`prepareForkedCommandContext` 会调用 `createGetAppStateWithAllowedTools`，将命令声明的 `allowedTools` 合并到 `appState.toolPermissionContext.alwaysAllowRules.command` 中，使 fork 子代理拥有额外工具使用权。
- **Prompt 命令权限声明**：`getMessagesForPromptSlashCommand` 将 `allowedTools` 解析后通过 `createAttachmentMessage({ type: 'command_permissions', ... })` 附加到消息流，`query.ts` 在构造 API 请求时会读取该 attachment 并限制模型可用工具。
- **默认权限检查**：`executeForkedSlashCommand` 的同步路径将 `canUseTool ?? hasPermissionsToUseTool` 传给 `runAgent`。

### 5.2 与 MCP 系统的交互

- **MCP settle 机制**：后台 fork 路径在启动子代理前，会轮询等待所有 MCP client 脱离 `pending` 状态（`MCP_SETTLE_POLL_MS = 200ms`，`MCP_SETTLE_TIMEOUT_MS = 10s`），然后调用 `context.options.refreshTools?.()` 刷新工具列表。这解决了“启动时 N 个定时任务同时触发，MCP 尚未连接导致工具缺失”的竞态问题。
- **MCP 命令解析**：`parseSlashCommand` 对 `/mcp:tool (MCP) arg1 arg2` 进行特殊处理，`isMcp = true`，`commandName = "mcp:tool (MCP)"`。

### 5.3 与 Compact / 恢复系统的交互

- `local` 类型的 `compact` 结果会将 slash 命令消息追加到 `messagesToKeep`，并调用 `buildPostCompactMessages` 和 `resetMicrocompactState()`。这确保 compact 操作不会丢失用户刚执行的 slash 命令上下文。
- `--resume` 恢复逻辑依赖消息时间戳；代码在 compact 分支中人为将最后一条消息时间戳加 100ms，确保恢复时选中正确的叶子节点。

### 5.4 与遥测/分析的交互

- 所有有效命令都会触发 `tengu_input_command`，并附带：
  - `input`（脱敏后的命令名或 `custom`/`mcp`）
  - `invocation_trigger: 'user-slash'`
  - 插件命令额外带上 `_PROTO_plugin_name`、`_PROTO_marketplace_name`、`plugin_repository`、`plugin_name`、`plugin_version` 以及 `buildPluginCommandTelemetryFields` 产生的字段。
- ant-only 额外带上 `skill_name`、`skill_source`、`skill_loaded_from`、`skill_kind`。
- 未知命令触发 `tengu_input_slash_invalid`；无法解析触发 `tengu_input_slash_missing`。

### 5.5 与 Hooks 系统的交互

- `getMessagesForPromptSlashCommand` 在 skill 加载后，若命令声明了 `hooks` 且通过 `pluginOnlyPolicy` 检查，则调用 `registerSkillHooks` 注册 session 级 hooks。
- 这些 hooks 会在后续工具调用前后（`PreToolUse` / `PostToolUse`）生效，影响整个会话行为。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险点

| 风险 | 位置 | 说明 |
|------|------|------|
| **后台 fork 异常吞没** | `executeForkedSlashCommand` 的 `void (async () => { ... })().catch(...)` | 若 `runAgent` 内部抛非 Error 对象或 `enqueuePendingNotification` 失败，错误仅通过 `logError` 输出，用户可能永远看不到失败结果。 |
| **MCP settle 硬超时** | `MCP_SETTLE_TIMEOUT_MS = 10_000` | 若某 MCP server 10s 内仍未连接，后台 fork 会在工具未就绪的情况下启动，可能导致子代理执行时缺少预期工具。 |
| **local-jsx 死锁残留** | `local-jsx` 分支的 `catch` | 虽然已有 `doneWasCalled` 防护，但如果 `mod.call()` 既抛异常又未调用 `onDone`，且异常被外层吞掉，仍可能留下 `isLocalJSXCommand = true` 的状态（不过当前代码已做 `clearLocalJSX: true` 清理）。 |
| **文件路径误判** | `processSlashCommand` 的 `isFilePath` 检查 | 使用 `getFsImplementation().stat("/${commandName}")` 判断；若用户输入 `/etc/passwd` 且文件存在，会被误判为文件路径而降级为普通 prompt，可能让用户困惑。 |
| **Coordinator 模式 skill 内容缺失** | `getMessagesForPromptSlashCommand` | Coordinator 模式下直接跳过 `getPromptForCommand`，返回摘要。若 worker 未正确执行 SkillTool，可能导致用户请求被挂起或忽略。 |
| **Workload ALS 泄漏** | `getWorkload()` 捕获 | 虽然 ALS 能正确隔离后台任务，但如果 `runAgent` 内部又启动新的 detached Promise 且未在 `runWithWorkload` 内执行，可能丢失 workload 标签。 |

### 6.2 边界行为

- **空消息命令**：若 `newMessages.length === 0`（如 `/model` 切换模型后只改状态不生成消息），`processSlashCommand` 返回 `shouldQuery: false`，`executeUserInput` 中不会调用 `onQuery`，而是直接清理 UI。
- **Compact 结果的消息顺序**：`isCompactResult` 会检查首条消息是否为 `compact_boundary`；若是，则**不**前置 `createSyntheticUserCaveatMessage`，因为 compact 结果自带 caveat 排序控制。
- **未知命令参数保留**：当命令不存在且看起来像命令名时，返回的错误消息中会附带 `createSystemMessage("Args from unknown skill: ${parsedArgs}", 'warning')`，让用户可以直接复制重试（gh-32591）。
- **userInvocable = false 的拦截**：模型专用 skill 若被用户直接输入，会返回提示 "This skill can only be invoked by Claude..."，同时仍记录 `recordSkillUsage`（因为拦截发生在查找之后）。

### 6.3 改进建议

1. **MCP settle 可配置化**
   - 当前 10s 硬编码对慢网络环境可能不足。建议将 `MCP_SETTLE_TIMEOUT_MS` 改为从环境变量或 GrowthBook 动态读取，并增加 settle 失败时的显式告警/埋点。

2. **后台 fork 结果可观测性**
   - 建议在 `enqueueResult` 的 `<scheduled-task-result>` 外增加一个内部状态通道（如 AppState 中的 `forkedCommandStatus`），让 UI 层能够展示“N 个后台任务运行中 / 失败”的指示器，而不是完全静默。

3. **文件路径判断更精确**
   - `stat("/${commandName}")` 会触发真实的文件系统调用，且仅检查根路径。建议改为检查 `path.resolve(getCwd(), commandName)`，并限制只在命令名包含 `.` 或 `/` 时才走文件路径判断，减少误伤。

4. **local-jsx 异常类型细化**
   - 当前 `catch(e)` 对所有异常统一处理。建议区分 `AbortError`（用户取消）与业务异常，给用户更准确的反馈。

5. **减少 executeForkedSlashCommand 的代码重复**
   - 同步 fork 与后台 fork 的 `agentDefinition` 构造、`runAgent` 参数组装逻辑有大量重复。可提取一个 `buildRunAgentParams(command, args, context, ...)` 辅助函数，降低维护成本。

6. **测试覆盖**
   - 当前仓库中未找到针对 `processSlashCommand.tsx` 的单元测试（`*.test.*` / `*.spec.*` 搜索无结果）。建议补充：
     - `looksLikeCommand` 的边界测试。
     - `processSlashCommand` 对未知命令、文件路径降级、MCP 命令的解析测试。
     - `getMessagesForSlashCommand` 三种命令类型的 mock 测试。
     - `executeForkedSlashCommand` 在 KAIROS / 非 KAIROS 模式下的行为测试（需 mock `runAgent` 和 `enqueuePendingNotification`）。
