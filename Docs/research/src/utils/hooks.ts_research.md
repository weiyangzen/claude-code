# 研究文档：src/utils/hooks.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文。  
> 生成时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 1. 场景与职责

`src/utils/hooks.ts` 是 Claude Code 的**Hook 执行中枢**，负责把用户配置、插件、Skill、SDK 回调等不同来源的 "hooks" 在合适的生命周期节点调度执行。Hook 本质上是用户自定义的扩展点：可以是本地 shell 命令、HTTP 端点、LLM prompt/agent 调用，或是内存中的 TypeScript 回调函数。

### 1.1 核心场景

| 场景 | 说明 |
|------|------|
| **生命周期事件** | `SessionStart` / `Setup` / `SessionEnd` / `Stop` / `SubagentStart` / `SubagentStop` 等 |
| **工具拦截** | `PreToolUse`（工具执行前拦截/改参/授权）、`PostToolUse` / `PostToolUseFailure`（工具执行后注入上下文） |
| **权限决策** | `PermissionRequest` / `PermissionDenied`（用 hook 替代或辅助人工弹窗做 allow/deny/ask 决策） |
| **用户输入** | `UserPromptSubmit`（用户发送消息前拦截） |
| **环境变化** | `CwdChanged` / `FileChanged` / `ConfigChange` / `InstructionsLoaded` |
| **状态/任务** | `TeammateIdle` / `TaskCreated` / `TaskCompleted` / `Notification` |
| **MCP 相关** | `Elicitation` / `ElicitationResult`（MCP 表单交互前后） |
| **UI 辅助** | `StatusLine` / `FileSuggestion`（特殊命令，不走标准 hook 匹配器） |
| **Git 工作树** | `WorktreeCreate` / `WorktreeRemove` |

### 1.2 文件职责边界

- **不负责** hook 的持久化配置解析（由 `hooksConfigSnapshot.ts` / `settings.ts` 负责）。
- **不负责** hook 的注册（由 `bootstrap/state.ts` 的 `registerHookCallbacks`、插件系统、Agent 前 matter 等负责）。
- **负责**：
  1. 收集并合并多来源 hook 配置（snapshot + registered + session-derived）。
  2. 按 `matcher` 模式过滤出应执行的 hook。
  3. 按类型分发到 `command` / `prompt` / `agent` / `http` / `callback` / `function` 执行器。
  4. 解析 hook 输出（JSON schema 校验、plain text、exit code 语义）。
  5. 汇总执行结果（blocking error、permission decision、additional context、updated input 等）并返回给调用方。
  6. 安全控制（workspace trust、managed-hooks-only、disableAllHooks）。

---

## 2. 功能点目的

### 2.1 多来源 Hook 合并与去重

用户可以在以下位置定义 hook：
- `.claude/settings.json`（project / user / local / policy）
- 插件 manifest
- Skill 配置
- SDK `registerHookCallbacks`
- 运行时 session 注入（`addSessionHook` / `addFunctionHook`）

`getHooksConfig()` 与 `getMatchingHooks()` 把这些来源合并成统一的 `MatchedHook[]`，并按来源命名空间去重（`hookDedupKey`），防止不同插件的模板命令因展开前文本相同而被误删。

### 2.2 六种 Hook 执行器

| 类型 | 执行器 | 目的 |
|------|--------|------|
| `command` | `execCommandHook` | 派生子进程（bash / PowerShell），通过 stdin 传入 JSON input，读取 stdout/stderr 和 exit code。支持异步后台执行（`async` / `asyncRewake`）。 |
| `prompt` | `execPromptHook` | 调用轻量模型（默认 Haiku）做一次性 JSON 结构化判断，返回 `{ok, reason}`，用于快速条件拦截。 |
| `agent` | `execAgentHook` | 启动一个多轮 agent（使用 `query()`），可调用工具验证复杂条件，最终通过 `StructuredOutputTool` 返回结果。 |
| `http` | `execHttpHook` | 向指定 URL POST JSON，支持 header 环境变量插值、URL allowlist、SSRF 防护、sandbox 代理。 |
| `callback` | `executeHookCallback` | 执行内存中的 JS 回调（SDK 或内部使用），返回 `HookJSONOutput`。 |
| `function` | `executeFunctionHook` | 执行 session 作用域的验证函数（`FunctionHookCallback`），返回 boolean，用于结构化输出强制校验等。 |

### 2.3 输出协议与语义

Hook 输出分三类：
1. **Async JSON**：首行输出 `{"async": true}`，进程被后台化，后续结果由 `AsyncHookRegistry` 在轮询中消费。
2. **Sync JSON**：经 Zod schema（`hookJSONOutputSchema`）校验，可携带 `decision`、`permissionDecision`、`additionalContext`、`updatedInput`、`systemMessage` 等字段。
3. **Plain text**：非 JSON 时，exit code 0 视为成功文本；exit code 2 视为 **blocking error**；其他非零视为 non-blocking error。

### 2.4 安全与治理

- **Workspace Trust**：所有 hook 在交互模式下必须等用户通过 trust dialog（`shouldSkipHookDueToTrust`）。这是为了防止在 trust 弹窗出现前执行恶意 hook（历史漏洞修复）。
- **Managed Hooks Only**：企业策略可设置 `allowManagedHooksOnly`，此时仅 policy settings 和内部 callback 可执行，插件/Skill/session hook 被跳过。
- **Disable All Hooks**：`shouldDisableAllHooksIncludingManaged()` 可全局关闭。
- **HTTP 限制**：`allowedHttpHookUrls` / `httpHookAllowedEnvVars` 控制 URL 和白名单环境变量；SSRF guard 阻止私有 IP 直连（代理环境除外）。

---

## 3. 具体技术实现

### 3.1 关键常量与时序

```ts
// src/utils/hooks.ts
const TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000        // 10 分钟
const SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500             // 1.5 秒
```

- 普通 hook 默认 10 分钟；`SessionEnd` 默认 1.5 秒（可通过 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 覆盖）。
- `StatusLine` / `FileSuggestion` 使用 5 秒短超时。

### 3.2 核心数据类型

#### 3.2.1 Hook 定义类型（来自 `src/utils/settings/types.ts` 与 `src/types/hooks.ts`）

- `HookCommand`：`{ type: 'command', command: string, shell?: 'bash' | 'powershell', timeout?: number, async?: boolean, asyncRewake?: boolean, if?: string }`
- `PromptHook` / `AgentHook` / `HttpHook`：分别对应 prompt/agent/http。
- `HookCallback`：内存回调，带 `internal?: boolean` 标记以排除在 `tengu_run_hook` 埋点之外。
- `FunctionHook`：session 专用，不可持久化到 settings。

#### 3.2.2 结果聚合类型

```ts
// src/utils/hooks.ts
export interface HookResult {
  message?: HookResultMessage
  systemMessage?: string
  blockingError?: HookBlockingError
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean
  stopReason?: string
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  hookPermissionDecisionReason?: string
  additionalContext?: string
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
  elicitationResponse?: ElicitationResponse
  watchPaths?: string[]
  hook: HookCommand | HookCallback | FunctionHook
}

export type AggregatedHookResult = { ... }  // 批量执行时对外 yield 的聚合结果
```

### 3.3 关键执行流程

#### 3.3.1 标准 REPL 内执行路径（`executeHooks`）

1. **前置过滤**：
   - `shouldDisableAllHooksIncludingManaged()` → 直接返回。
   - `isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)` → 直接返回。
   - `shouldSkipHookDueToTrust()` → 直接返回（交互模式未 trust 时）。
2. **匹配**：`getMatchingHooks(appState, sessionId, hookEvent, hookInput, tools?)`
   - 合并 snapshot + registered + session hooks。
   - 按 `matchQuery` 与 `matcher` 做模式匹配（`matchesPattern`）。
   - 支持 `if` 条件过滤（对 tool 事件调用 `prepareIfConditionMatcher`，利用 tool 的 `preparePermissionMatcher`）。
   - 去重（按来源 + 命令/prompt/url）。
3. **快速路径**：如果所有匹配 hook 都是 internal callback，直接串行执行并跳过 span/telemetry/abort 复杂逻辑（性能优化 ~70%）。
4. **批量执行**：所有 hook 并行（`all(hookPromises)`），每个 hook 单独包装 `createCombinedAbortSignal` 实现独立超时。
5. **按类型分发**：
   - `callback` → `executeHookCallback`
   - `function` → `executeFunctionHook`（需要 `messages`）
   - `prompt` → `execPromptHook`
   - `agent` → `execAgentHook`
   - `http` → `execHttpHook`
   - `command` → `execCommandHook`
6. **结果解析**：
   - `parseHookOutput` / `parseHttpHookOutput` 区分 JSON / plain text。
   - `processHookJSONOutput` 把 JSON 映射到 `HookResult` 各字段。
   - exit code 2 映射为 `blocking` outcome。
7. **聚合 yield**：按优先级处理 `preventContinuation` → `blockingError` → `message` → `systemMessage` → `additionalContext` → `permissionBehavior`（deny > ask > allow）等。
8. **回调与清理**：若配置了 `onHookSuccess`（来自 session hook），在 success 时触发；`executeHooks` 末尾记录 `tengu_repl_hook_finished` 与 OTEL span。

#### 3.3.2 命令 Hook 的进程生命周期（`execCommandHook`）

```ts
// 伪代码
const shellType = hook.shell ?? DEFAULT_HOOK_SHELL  // 'bash'
const isPowerShell = shellType === 'powershell'

// Windows 路径转换（bash 用 Git Bash 需要 POSIX 路径）
const toHookPath = isWindows && !isPowerShell ? windowsPathToPosixPath : identity

// 变量替换：${CLAUDE_PLUGIN_ROOT} / ${CLAUDE_PLUGIN_DATA} / ${user_config.X}
command = substituteVariables(hook.command)

// 环境变量注入
envVars = {
  ...subprocessEnv(),
  CLAUDE_PROJECT_DIR: toHookPath(projectDir),
  CLAUDE_PLUGIN_ROOT?: toHookPath(pluginRoot),
  CLAUDE_PLUGIN_DATA?: toHookPath(dataDir),
  CLAUDE_PLUGIN_OPTION_*?: pluginOpts,
  CLAUDE_ENV_FILE?: await getHookEnvFilePath(hookEvent, hookIndex), // SessionStart/Setup/CwdChanged/FileChanged
}

// 子进程创建
if (isPowerShell) {
  child = spawn(pwshPath, buildPowerShellArgs(finalCommand), { env: envVars, cwd: safeCwd, windowsHide: true })
} else {
  child = spawn(finalCommand, [], { shell: isWindows ? findGitBashPath() : true, env: envVars, cwd: safeCwd, windowsHide: true })
}

// 包装为 ShellCommand（pipe 模式，因为 hooks 需要实时 stdout）
const hookTaskOutput = new TaskOutput(`hook_${child.pid}`, null)
const shellCommand = wrapSpawn(child, signal, hookTimeoutMs, hookTaskOutput)
```

**Async 协议**：
- 若 hook 声明 `async: true` 或 `asyncRewake: true`，在写入 stdin 后立刻调用 `executeInBackground`。
- `executeInBackground` 对普通 async 调用 `shellCommand.background(processId)` 把进程后台化，并将进程注册到 `AsyncHookRegistry`。
- 对 `asyncRewake`（目前仅 Stop 相关场景），**不**调用 `background()`（避免 `spillToDisk` 破坏内存 stdout 捕获），而是直接挂一个 `.then` 在 `shellCommand.result` 上，等进程结束后若 exit code 2 则通过 `enqueuePendingNotification` 唤醒模型。

**Prompt 请求协议**：
- 若 `requestPrompt` 被传入（如 `PreToolUse` 的某些路径），`execCommandHook` 会在 stdout 的流式输出中逐行检测 JSON 是否符合 `promptRequestSchema`。
- 检测到后通过 `requestPrompt` 向用户展示选项，用户选择结果通过 `child.stdin.write(jsonStringify(response))` 回写给 hook 进程。
- 所有 prompt 响应通过 `promptChain` Promise 串行化，防止竞态。

#### 3.3.3 Prompt Hook 实现（`execPromptHook.ts`）

- 使用 `queryModelWithoutStreaming` 调用轻量模型（默认 Haiku）。
- system prompt 强制要求返回 JSON `{ok: boolean, reason?: string}`。
- 若 `ok === false`，映射为 `blocking` outcome 并设置 `preventContinuation`。
- 传入 `messages` 时会把历史消息 prepend 到 prompt 前，使模型具备上下文。

#### 3.3.4 Agent Hook 实现（`execAgentHook.ts`）

- 创建独立的 `hookAgentId`，构造新的 `ToolUseContext`（`agentToolUseContext`）。
- 过滤掉 `ALL_AGENT_DISALLOWED_TOOLS`（防止 stop hook agent 再 spawn 子 agent 或进入 plan mode）。
- 注入 `StructuredOutputTool` 并注册 session-level stop hook 强制 agent 最终必须调用该工具。
- 通过 `query({ ... })` 多轮执行，最多 `MAX_AGENT_TURNS = 50` 轮。
- 读取 transcript 的权限通过 `alwaysAllowRules` 自动放行。

#### 3.3.5 HTTP Hook 实现（`execHttpHook.ts`）

- 使用 `axios.post`，`validateStatus: () => true` 接收所有状态码。
- `maxRedirects: 0` 禁止重定向。
- 代理策略：
  - 若 sandbox 启用且网络代理就绪，走 sandbox proxy（`proxy: { host, port, protocol }`）。
  - 若环境变量代理（HTTP_PROXY/HTTPS_PROXY）生效，走 `configureGlobalAgents()` 安装的 interceptor（`proxy: false` 防止 axios 自身重复检测）。
  - 否则直接连接，并启用 `ssrfGuardedLookup` 阻止私有 IP。
- Header 支持 `$VAR_NAME` / `${VAR_NAME}` 环境变量插值，但仅允许 `allowedEnvVars` 白名单中的变量；同时用 `sanitizeHeaderValue` 去除 `\r\n\x00` 防止 CRLF 注入。

### 3.4 JSON 输出 Schema

定义在 `src/types/hooks.ts`：

```ts
export const syncHookResponseSchema = z.object({
  continue: z.boolean().optional(),
  suppressOutput: z.boolean().optional(),
  stopReason: z.string().optional(),
  decision: z.enum(['approve', 'block']).optional(),
  reason: z.string().optional(),
  systemMessage: z.string().optional(),
  hookSpecificOutput: z.union([...]).optional(),
})
```

`hookSpecificOutput` 按事件类型细分，例如：
- `PreToolUse`：`permissionDecision`, `permissionDecisionReason`, `updatedInput`, `additionalContext`
- `UserPromptSubmit`：`additionalContext`
- `SessionStart`：`additionalContext`, `initialUserMessage`, `watchPaths`
- `PostToolUse`：`additionalContext`, `updatedMCPToolOutput`
- `PermissionRequest`：`decision`（含 `behavior: allow/deny` 与 `updatedInput`）
- `Elicitation` / `ElicitationResult`：`action`（accept/decline/cancel）, `content`
- `WorktreeCreate`：`worktreePath`

### 3.5 事件广播（`hookEvents.ts`）

`emitHookStarted` / `emitHookProgress` / `emitHookResponse` 把 hook 执行状态广播到注册的事件处理器。默认只有 `SessionStart` 和 `Setup` 事件会被广播；当 SDK 设置 `includeHookEvents` 或远程模式时，`setAllHookEventsEnabled(true)` 打开全部事件。事件先进入 100 条上限的 `pendingEvents` 队列，等 handler 注册后批量消费。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件核心函数分布

| 函数/类型 | 行号 | 作用 |
|-----------|------|------|
| `getSessionEndHookTimeoutMs` | 176-182 | 环境变量覆盖的 session end 超时 |
| `shouldSkipHookDueToTrust` | 286-296 | workspace trust 安全门 |
| `createBaseHookInput` | 301-328 | 构造所有 hook 共享的基础 JSON input |
| `validateHookJson` / `parseHookOutput` / `parseHttpHookOutput` | 382-487 | JSON 校验与解析 |
| `processHookJSONOutput` | 489-737 | 把 JSON 映射为 HookResult |
| `execCommandHook` | 747-1335 | 命令型 hook 的子进程执行 |
| `matchesPattern` | 1346-1381 | matcher 模式匹配（含正则） |
| `prepareIfConditionMatcher` | 1390-1421 | `if` 条件预编译（tool 专用） |
| `getMatchingHooks` | 1603-1874 | 合并、过滤、去重 |
| `executeHooks` | 1952-2972 | REPL 内的标准批量执行器 |
| `executeHooksOutsideREPL` | 3003-3381 | 非 REPL 场景（通知、session end 等） |
| `executePreToolHooks` / `executePostToolHooks` / ... | 3394- | 各生命周期事件的便捷入口 |
| `executeStatusLineCommand` | 4584-4666 | StatusLine 特殊入口 |
| `executeFileSuggestionCommand` | 4675-4738 | FileSuggestion 特殊入口 |

### 4.2 直接依赖文件（被调用方）

| 文件 | 依赖内容 |
|------|----------|
| `src/utils/ShellCommand.ts` | `wrapSpawn` — 把 `ChildProcess` 包装成带超时、abort、后台化能力的 `ShellCommand` |
| `src/utils/task/TaskOutput.ts` | `TaskOutput` — hook 使用 pipe 模式（`stdoutToFile = false`），通过 `writeStdout` / `writeStderr` 收集输出 |
| `src/utils/hooks/execPromptHook.ts` | `execPromptHook` — prompt 型 hook 的模型调用 |
| `src/utils/hooks/execAgentHook.ts` | `execAgentHook` — agent 型 hook 的多轮执行 |
| `src/utils/hooks/execHttpHook.ts` | `execHttpHook` — HTTP 型 hook 的请求发送 |
| `src/utils/hooks/AsyncHookRegistry.ts` | `registerPendingAsyncHook` / `getPendingAsyncHooks` / `checkForAsyncHookResponses` — 异步后台 hook 的全局注册表 |
| `src/utils/hooks/hookEvents.ts` | `emitHookStarted` / `emitHookResponse` / `startHookProgressInterval` — 事件广播 |
| `src/utils/hooks/sessionHooks.ts` | `getSessionHooks` / `getSessionFunctionHooks` / `getSessionHookCallback` / `clearSessionHooks` — session 作用域 hook |
| `src/utils/hooks/hooksConfigSnapshot.ts` | `getHooksConfigFromSnapshot` / `shouldAllowManagedHooksOnly` / `shouldDisableAllHooksIncludingManaged` — 配置快照与策略门 |
| `src/utils/sessionEnvironment.js` | `getHookEnvFilePath` / `invalidateSessionEnvCache` — SessionStart 等事件的 env 文件路径 |
| `src/utils/settings/settings.ts` | `getSettings_DEPRECATED` / `getSettingsForSource` — 读取 statusLine / fileSuggestion |
| `src/utils/attachments.ts` | `createAttachmentMessage` — 把 hook 结果转为消息附件 |
| `src/utils/combinedAbortSignal.ts` | `createCombinedAbortSignal` — 组合外部 abort 与独立超时 |
| `src/utils/telemetry/sessionTracing.ts` | `startHookSpan` / `endHookSpan` / `isBetaTracingEnabled` — OTEL 追踪 |
| `src/services/analytics/index.ts` | `logEvent` — 埋点（`tengu_run_hook`, `tengu_repl_hook_finished` 等） |

### 4.3 主要调用方（谁在用本文件）

| 文件 | 调用内容 |
|------|----------|
| `src/services/tools/toolHooks.ts` | `executePreToolHooks`, `executePostToolHooks`, `executePostToolUseFailureHooks`, `getPreToolHookBlockingMessage` |
| `src/services/tools/toolExecution.ts` | `executePermissionDeniedHooks` |
| `src/query/stopHooks.ts` | `executeStopHooks`, `executeTeammateIdleHooks`, `executeTaskCompletedHooks`, `getStopHookMessage` 等 |
| `src/query.ts` | `executeStopFailureHooks` |
| `src/screens/REPL.tsx` | `executeSessionEndHooks`, `getSessionEndHookTimeoutMs` |
| `src/services/compact/compact.ts` | `executePreCompactHooks`, `executePostCompactHooks` |
| `src/components/StatusLine.tsx` | `createBaseHookInput`, `executeStatusLineCommand` |
| `src/hooks/fileSuggestions.ts` | `executeFileSuggestionCommand` |
| `src/hooks/toolPermission/PermissionContext.ts` | `executePermissionRequestHooks` |
| `src/cli/structuredIO.ts` | `executePermissionRequestHooks` |
| `src/cli/print.ts` | `executeNotificationHooks` |
| `src/services/notifier.ts` | `executeNotificationHooks` |
| `src/tools/AgentTool/runAgent.ts` | `executeSubagentStartHooks` |
| `src/tools/TaskCreateTool/TaskCreateTool.ts` | `executeTaskCreatedHooks` |
| `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | `executeTaskCreatedHooks`, `executeTaskCompletedHooks` |
| `src/commands/compact/compact.ts` | `executePreCompactHooks` |
| `src/commands/clear/conversation.ts` | `executeSessionEndHooks` |
| `src/setup.ts` | `hasWorktreeCreateHook` |
| `src/bridge/bridgeMain.ts` | `hasWorktreeCreateHook`（动态 import） |
| `src/services/mcp/elicitationHandler.ts` | `executeElicitationHooks`, `executeElicitationResultHooks` |
| `src/utils/skills/skillChangeDetector.ts` | `executeConfigChangeHooks`, `hasBlockingResult` |
| `src/utils/sessionStart.ts` | `executeSessionStartHooks`, `executeSetupHooks` |
| `src/utils/permissions/permissions.ts` | `executePermissionRequestHooks` |

---

## 5. 依赖与外部交互

### 5.1 进程与环境交互

- **子进程创建**：使用 Node.js `child_process.spawn`。
  - bash 路径通过 `findGitBashPath()` 在 Windows 上显式定位 Git Bash。
  - PowerShell 路径通过 `getCachedPowerShellPath()` 缓存 `pwsh` / `powershell`。
- **环境变量**：
  - `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` — 覆盖 session end 超时。
  - `CLAUDE_CODE_SHELL_PREFIX` — 为 bash hook 包装前缀命令（PowerShell 当前忽略）。
  - `CLAUDE_CODE_SIMPLE` — 若 truthy，则跳过所有 hook。
- **stdin 写入**：所有 command hook 通过 `child.stdin.write(jsonInput + '\n', 'utf8')` 传递输入；prompt 协议下 stdin 保持开放以支持多轮交互。

### 5.2 与状态管理的交互

- `AppState.sessionHooks`（`Map<string, SessionStore>`）存储运行时注入的 session hook 和 function hook。
- `getRegisteredHooks()` 来自 `bootstrap/state.ts`，存储 SDK/插件注册的 callback hook。
- `getHooksConfigFromSnapshot()` 返回 settings.json 中解析出的静态 hook 配置快照。

### 5.3 与权限系统的交互

- `PreToolUse` hook 可返回 `permissionBehavior`（allow/deny/ask/passthrough），在 `src/services/tools/toolHooks.ts` 的 `resolveHookPermissionDecision` 中与 `checkRuleBasedPermissions` 的结果合并：
  - hook 的 **deny** 直接生效。
  - hook 的 **allow** 仍需过 settings.json 的 deny/ask 规则（不能绕过用户显式规则）。
  - hook 的 **ask** 会强制弹出权限对话框，并携带 hook 提供的 `updatedInput`。

### 5.4 与消息系统的交互

- hook 执行过程中产生的 `ProgressMessage<HookProgress>` 会实时 yield 到调用方，最终进入 transcript。
- `createAttachmentMessage` 生成的附件类型包括：`hook_success`、`hook_non_blocking_error`、`hook_blocking_error`、`hook_cancelled`、`hook_stopped_continuation`、`hook_additional_context`、`hook_permission_decision`、`hook_system_message`、`hook_error_during_execution`。

### 5.5 与测试的交互

- 经检索，**本仓库没有针对 `src/utils/hooks.ts` 的独立单元测试文件**（无 `hooks.test.ts` / `hooks.spec.ts`）。
- 相关逻辑主要通过以下方式覆盖：
  - `AsyncHookRegistry.ts` 自带 `clearAllAsyncHooks()` 测试辅助函数。
  - `execHttpHook.ts` 的测试可能存在于更上层的集成测试中（未在代码树内发现直接测试文件）。
  - 工具链 `toolHooks.ts` 和 `toolExecution.ts` 的复杂交互更多依赖端到端或手动测试。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 代码位置 |
|------|------|----------|
| **Trust 绕过（历史漏洞）** | 若 `shouldSkipHookDueToTrust` 被绕过，未 trust 的工作空间可在弹窗前执行任意命令。当前已在所有入口统一检查。 | `executeHooks:1994`, `executeHooksOutsideREPL:3031`, `executeStatusLineCommand:4597` |
| **Exit code 2 的歧义** | 命令 hook 中 exit code 2 表示 blocking。但某些工具（如 `python3 <missing-file>`）也会返回 2，导致误拦截。代码在 spawn 前增加了 `pathExists(pluginRoot)` 检查以缓解。 | `execCommandHook:831-836` |
| **AsyncRewake 的内存泄漏** | `asyncRewake` 路径故意不调用 `shellCommand.background()`，因此不触发 `spillToDisk`，但也没有显式清理 `shellCommand` 的流监听器（依赖 `.then` 中的 `shellCommand.cleanup()`）。若进程长期挂起，可能持有引用。 | `executeInBackground:218-245` |
| **Prompt 请求的 EPIPE** | 当 hook 进程提前退出而后续还有 prompt 响应待写入时，可能触发 `EPIPE`。代码已做错误捕获并销毁 stdin，但 TODO 标注缺少 Bun/Node 差异测试。 | `execCommandHook:1198-1207` |
| **HTTP Hook 的 URL 允许列表空数组** | `allowedHttpHookUrls` 若为 `[]`，会阻止所有 HTTP hook（符合设计），但用户可能误配导致静默失败。 | `execHttpHook.ts:138-144` |
| **Agent Hook 的无限循环** | `MAX_AGENT_TURNS = 50` 是硬限制，但 agent 可能在达到限制前进入无意义循环消耗 token。 | `execAgentHook.ts:119` |

### 6.2 边界行为

- **空 matcher**：`matcher` 为空字符串或 `*` 时，`matchesPattern` 返回 `true`，即匹配所有查询。
- **非工具事件的 `if` 条件**：`prepareIfConditionMatcher` 对非 `PreToolUse` / `PostToolUse` / `PostToolUseFailure` / `PermissionRequest` 返回 `undefined`，因此这些事件的 `if` 条件会被静默跳过（`logForDebugging` 记录）。
- **HTTP hooks 在 SessionStart/Setup 被禁用**：因为 headless 模式下 sandbox ask callback 可能死锁。代码在 `getMatchingHooks` 中显式过滤。
- **Function hooks 不支持 REPL 外执行**：`executeHooksOutsideREPL` 遇到 `function` 类型会记录 error 并返回失败结果。
- **StatusLine / FileSuggestion 不受 `hasHookForEvent` 优化**：它们直接读取 settings，不走 `getMatchingHooks` 的批量路径。

### 6.3 改进建议

1. **增加单元测试覆盖**
   - 当前 `hooks.ts` 缺乏独立测试。建议为 `matchesPattern`、`processHookJSONOutput`、`getMatchingHooks` 的 dedup 逻辑、`shouldSkipHookDueToTrust` 的各分支增加单元测试。
   - `execCommandHook` 的 async 检测、prompt 请求协议、EPIPE 处理可用 mock `ChildProcess` + `TaskOutput` 做集成测试。

2. **统一超时配置**
   - 目前 `TOOL_HOOK_EXECUTION_TIMEOUT_MS` 是硬编码的 10 分钟，建议允许通过环境变量或 settings.json 全局覆盖，而不是仅 `SessionEnd` 支持环境变量。

3. **细化 asyncRewake 生命周期管理**
   - 建议为 `asyncRewake` 进程增加一个显式的超时清理器（类似 `AsyncHookRegistry` 的 `timeout`），防止进程异常挂起时 `shellCommand` 长期不被释放。

4. **改进 `if` 条件的错误可见性**
   - 当前非工具事件的 `if` 条件被静默丢弃。建议在 settings 校验阶段（`hooksConfigManager.ts` 或 schema）就报错或警告，而不是在运行时忽略。

5. **PowerShell 前缀支持**
   - `CLAUDE_CODE_SHELL_PREFIX` 当前对 PowerShell hook 完全忽略。若用户有统一的 hook 包装需求（如审计、日志），应补充 `CLAUDE_CODE_PS_SHELL_PREFIX` 或 shell-aware 前缀机制（设计文档 §8.1 已提及）。

6. **减少 `executeHooks` 的函数长度与认知负担**
   - `executeHooks` 接近 1000 行（含内部 generator），职责过重。可考虑把结果聚合逻辑（`for await (const result of all(hookPromises))` 的大段 switch/yield）提取为独立的 `aggregateHookResults` 模块，提升可维护性。

7. **HTTP Hook 响应体大小限制**
   - 当前 `execHttpHook` 未对 `body` 做长度截断。若服务端返回超大响应，可能导致 `parseHttpHookOutput` 中的 JSON parse 占用大量内存。建议在 axios 层增加 `maxContentLength` 或读取后截断。

---

*文档结束*
