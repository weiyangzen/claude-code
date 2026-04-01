# resumeAgent.ts 深度研究文档

## 1. 场景与职责

`resumeAgent.ts` 负责**恢复已停止（或已完成）后台 Agent 的执行**。当用户通过 UI 向一个处于 `completed`/`failed`/`killed` 状态的本地 Agent 任务发送新消息，或通过 `SendMessageTool` 向已停止的 Agent 发送消息时，系统不会创建新 Agent，而是调用 `resumeAgentBackground()` 从该 Agent 的历史 transcript 继续对话。

典型触发场景：
- **REPL 消息输入**：用户在主会话面板选中一个已停止的 background agent 并继续打字发送，`REPL.tsx` 检测到任务非 `running` 后调用恢复。
- **SendMessageTool 跨 Agent 通信**：`SendMessageTool.ts` 发现目标 Agent 没有活跃任务（`task.status !== 'running'`）时，自动恢复目标 Agent 并将消息作为新的 user prompt 注入。

职责边界：
- 只负责**恢复已有 Agent**，不负责首次创建。
- 只支持**异步（background）Agent**的恢复；同步 Agent 一旦结束即不存在恢复路径。
- 需要完整还原 Agent 的上下文：transcript、worktree 隔离目录、content replacement 状态、系统提示词、工具池等。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| **Transcript 恢复** | 从磁盘读取该 Agent 的 sidechain transcript，作为新一轮 API 请求的 message prefix，保证对话连续性。 |
| **消息清洗** | 过滤掉未完成的 `tool_use`、孤立的 `thinking` 块、仅含空白的 assistant 消息，避免 API 400 错误。 |
| **Content Replacement 重建** | 重建大工具结果被替换为文件引用的状态（`reconstructForSubagentResume`），确保 prompt cache 稳定。 |
| **Worktree 恢复** | 若原 Agent 使用了 `isolation: worktree`，恢复时重新进入该 worktree 目录；若目录已被外部删除则优雅回退到父 cwd。 |
| **Fork Agent 特化** | 对 `FORK_AGENT` 类型的子代理，恢复时必须复用父级的 system prompt 字节（cache-identical），并继承父级完整工具池。 |
| **任务再注册** | 调用 `registerAsyncAgent` 重新在 `AppState.tasks` 中注册任务，使用原 `agentId`，保持 UI 面板和输出文件路径不变。 |

---

## 3. 具体技术实现

### 3.1 入口函数 `resumeAgentBackground`

签名：
```typescript
export async function resumeAgentBackground({
  agentId,
  prompt,
  toolUseContext,
  canUseTool,
  invokingRequestId,
}: {
  agentId: string
  prompt: string
  toolUseContext: ToolUseContext
  canUseTool: CanUseToolFn
  invokingRequestId?: string
}): Promise<ResumeAgentResult>
```

执行流程：
1. **读取持久化数据**（并行）：
   - `getAgentTranscript(asAgentId(agentId))` → 获取 `transcript.messages` 和 `transcript.contentReplacements`。
   - `readAgentMetadata(asAgentId(agentId))` → 获取 `agentType`、`worktreePath`、`description`。
2. **消息清洗链**（顺序执行，由内到外）：
   - `filterUnresolvedToolUses(transcript.messages)`：移除 assistant 消息中所有 `tool_use` 都未收到 `tool_result` 的条目。
   - `filterOrphanedThinkingOnlyMessages(...)`：移除仅包含 `thinking`/`redacted_thinking` 且没有对应非 thinking 消息的 assistant 消息。
   - `filterWhitespaceOnlyAssistantMessages(...)`：移除仅含空白文本的 assistant 消息。
3. **Content Replacement 状态重建**：
   - `reconstructForSubagentResume(parentState, resumedMessages, sidechainRecords)`
   - 以 sidechain 持久化的 `ContentReplacementRecord[]` 为主，缺失条目用父级 `toolUseContext.contentReplacementState.replacements` gap-fill（对 fork 子代理尤为重要）。
4. **Worktree 路径校验**：
   - 若 `meta.worktreePath` 存在，执行 `fsp.stat` 确认目录仍有效；无效则打 debug log 并回退到 `undefined`。
   - 有效时调用 `fsp.utimes` 更新 mtime，防止 stale-worktree 清理任务误删刚恢复的目录（issue #22355）。
5. **Agent 定义解析**：
   - 若 `meta.agentType === FORK_AGENT.agentType`，直接选中 `FORK_AGENT`，标记 `isResumedFork = true`。
   - 否则在 `toolUseContext.options.agentDefinitions.activeAgents` 中按 `agentType` 查找；找不到则回退到 `GENERAL_PURPOSE_AGENT`。
   - **注意**：跳过 `filterDeniedAgents` 重新鉴权——原始 spawn 已通过权限检查。
6. **Fork 父级 system prompt 重建**（仅 fork）：
   - 优先使用 `toolUseContext.renderedSystemPrompt`（父级在 turn start 冻结的字节）。
   - 若不存在，则重新调用 `getSystemPrompt` + `buildEffectiveSystemPrompt` 计算；若仍失败则抛错。
7. **工具池组装**：
   - Fork resume：直接继承 `toolUseContext.options.tools`（保证 cache-identical）。
   - 非 fork：调用 `assembleToolPool(workerPermissionContext, appState.mcp.tools)` 按 Agent 自己的 `permissionMode` 重新组装。
8. **构造 `runAgent` 参数**：
   - `promptMessages = [...resumedMessages, createUserMessage({ content: prompt })]`
   - `override.systemPrompt`：fork 时传入父级 system prompt；非 fork 时 `undefined`（让 `runAgent` 在 `wrapWithCwd` 下重新计算）。
   - `useExactTools: isResumedFork ? true : undefined`
   - `worktreePath`、`description`、`contentReplacementState` 透传。
9. **任务注册与生命周期启动**：
   - `registerAsyncAgent({ agentId, description, prompt, selectedAgent, setAppState: rootSetAppState, toolUseId })`
   - 使用 `runWithAgentContext(asyncAgentContext, () => wrapWithCwd(() => runAsyncAgentLifecycle(...)))` 包裹执行。
   - `asyncAgentContext` 中 `invocationKind: 'resume'`，用于 telemetry 区分 spawn/resume。
10. **返回结果**：`{ agentId, description, outputFile: getTaskOutputPath(agentId) }`

### 3.2 关键数据结构

```typescript
export type ResumeAgentResult = {
  agentId: string
  description: string
  outputFile: string
}
```

- `agentId` 保持不变，因此输出文件路径、transcript 目录、metadata 文件均复用原路径。
- `outputFile` 指向 `~/.claude/projects/<project>/tasks/<agentId>.jsonl`（通过 `getTaskOutputPath` 解析），供调用方告知用户查看进度。

### 3.3 CWD 包装器

```typescript
const wrapWithCwd = <T>(fn: () => T): T =>
  resumedWorktreePath ? runWithCwdOverride(resumedWorktreePath, fn) : fn()
```

- 使用 `src/utils/cwd.ts` 中的 `runWithCwdOverride`，基于 `AsyncLocalStorage` 在恢复期间覆盖 `process.cwd()` 的返回值。
- 确保 `runAgent` 内部调用 `getCwd()` 时看到的是 worktree 路径，从而文件读写落在隔离目录。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件
- `src/tools/AgentTool/resumeAgent.ts`（265 行）

### 4.2 直接调用方
- `src/screens/REPL.tsx:3556` — 用户继续向已停止 Agent 发消息时恢复。
- `src/tools/SendMessageTool/SendMessageTool.ts:824,851` — SendMessage 目标 Agent 停止时自动恢复。

### 4.3 核心依赖文件
| 文件 | 作用 |
|------|------|
| `src/utils/sessionStorage.ts` | `getAgentTranscript`、`readAgentMetadata` — 读取 sidechain 持久化数据。 |
| `src/utils/messages.ts` | `filterUnresolvedToolUses`、`filterOrphanedThinkingOnlyMessages`、`filterWhitespaceOnlyAssistantMessages`、`createUserMessage` — 消息清洗与构造。 |
| `src/utils/toolResultStorage.ts` | `reconstructForSubagentResume` — 重建 content replacement 状态。 |
| `src/utils/cwd.ts` | `runWithCwdOverride` — worktree cwd 恢复。 |
| `src/utils/systemPrompt.ts` | `buildEffectiveSystemPrompt` — fork 路径重建父级 system prompt。 |
| `src/utils/model/agent.ts` | `getAgentModel` — 解析 Agent 实际使用的模型。 |
| `src/utils/forkedAgent.ts` | `createSubagentContext`（间接通过 `runAgent`）— 子代理上下文隔离。 |
| `src/utils/agentContext.ts` | `runWithAgentContext` — ALS 上下文包裹，用于 telemetry 归因。 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `registerAsyncAgent`、`runAsyncAgentLifecycle`（定义在 `agentToolUtils.ts`）— 任务注册与后台生命周期驱动。 |
| `src/tools/AgentTool/runAgent.ts` | `runAgent` — 实际执行 query 循环。 |
| `src/tools/AgentTool/forkSubagent.ts` | `FORK_AGENT`、`isForkSubagentEnabled` — fork 子代理定义与特性开关。 |
| `src/tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition`、`isBuiltInAgent` — Agent 类型定义。 |
| `src/tools/AgentTool/built-in/generalPurposeAgent.ts` | `GENERAL_PURPOSE_AGENT` — 默认回退 Agent。 |
| `src/tools/AgentTool/agentToolUtils.ts` | `runAsyncAgentLifecycle` — 后台 Agent 生命周期（进度追踪、通知、异常处理）。 |

---

## 5. 依赖与外部交互

### 5.1 文件系统交互
- **读取**：`~/.claude/projects/<sanitized_cwd>/<sessionId>/subagents/<agentId>.jsonl`（transcript）
- **读取**：`~/.claude/projects/<sanitized_cwd>/<sessionId>/subagents/<agentId>-metadata.json`（metadata）
- **修改时间**：`fsp.utimes(resumedWorktreePath, now, now)` — 触碰 worktree 目录 mtime。

### 5.2 AppState 交互
- 通过 `rootSetAppState = toolUseContext.setAppStateForTasks ?? toolUseContext.setAppState` 写入任务状态。
- `registerAsyncAgent` 会在 `AppState.tasks[agentId]` 中创建/覆盖 `LocalAgentTaskState`。
- `utils/task/framework.ts:82` 特别说明：重新注册时会保留旧任务的 `retain`、`startTime`、`messages`、`diskLoaded`，避免 UI 状态闪烁。

### 5.3 网络/API 交互
- 不直接发起网络请求；所有 API 调用委托给 `runAgent` → `query()`。

### 5.4 进程/环境交互
- 使用 `runWithCwdOverride` 基于 `AsyncLocalStorage` 临时修改 cwd 语义。
- 使用 `runWithAgentContext` 基于 `AsyncLocalStorage` 设置 telemetry 上下文。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 代码位置 |
|------|------|----------|
| **Worktree 外部删除** | 若用户在 Agent 停止期间手动删除了 worktree，恢复时依赖 `fsp.stat` 回退；但 `runAgent` 内部的 `writeAgentMetadata` 仍可能尝试写入已不存在的路径结构。 | `resumeAgent.ts:82-92` |
| **Fork system prompt 漂移** | 若 `toolUseContext.renderedSystemPrompt` 缺失，重新计算 `buildEffectiveSystemPrompt` 可能因 GrowthBook 状态变化导致字节不一致，破坏 prompt cache。 | `resumeAgent.ts:117-148` |
| **Agent 定义变更** | 若 Agent 的 `.md` 文件在停止期间被修改（尤其是 custom/plugin agent），恢复时使用的是新定义，但 transcript 是基于旧定义产生的，可能导致行为不一致。 | `resumeAgent.ts:106-112` |
| **权限回退** | `filterDeniedAgents` 被显式跳过（注释 "Skip filterDeniedAgents re-gating"）。若用户在停止期间新增了 deny 规则，已停止的 Agent 恢复时不会生效。 | `resumeAgent.ts:99` |
| **No transcript 硬抛错** | 若 transcript 文件损坏或被清理，`getAgentTranscript` 返回空，直接 `throw new Error`，调用方（REPL/SendMessage）会展示错误通知。 | `resumeAgent.ts:67-69` |

### 6.2 边界行为

- **同步 Agent 无恢复路径**：`resumeAgentBackground` 只注册 async task；若原 Agent 是同步的，停止后不会在 UI 中保留可交互的任务卡片，自然无法触发恢复。
- **Fork 子代理的递归禁止**：fork 子代理保留 `AgentTool`，但 `AgentTool.tsx` 在 spawn 时通过 `isInForkChild()` 拦截；恢复时此检查在 `AgentTool.call()` 中生效，若恢复的 fork Agent 再次尝试 spawn 自己会被拒绝。
- **Content Replacement 特征关闭**：若父级 `toolUseContext.contentReplacementState` 为 `undefined`（功能关闭），`reconstructForSubagentResume` 返回 `undefined`，恢复后的 Agent 不再做结果替换。

### 6.3 改进建议

| 优先级 | 建议 | 理由 |
|--------|------|------|
| **中** | 在 `readAgentMetadata` 失败后增加降级逻辑 | 当前 metadata 读取失败仅导致 `meta` 为 `undefined`，全部回退到 `GENERAL_PURPOSE_AGENT`，可能丢失 `worktreePath` 和 `description`。 |
| **中** | 对非 fork 恢复也支持 `renderedSystemPrompt` 缓存 | 当前只有 fork 路径显式传递 `override.systemPrompt`；非 fork 恢复每次重新计算 system prompt，存在重复开销和漂移风险。 |
| **低** | 考虑在恢复前校验 Agent 定义哈希 | 在 metadata 中写入 Agent 定义的内容哈希，恢复时比对，若变更则向用户发出警告或强制使用新定义重开。 |
| **低** | 将 `resumeAgentBackground` 的异常处理统一收敛 | 当前 REPL 和 SendMessageTool 各自捕获错误并展示不同格式的通知，可提取为共享的 `handleResumeError` 工具函数。 |

