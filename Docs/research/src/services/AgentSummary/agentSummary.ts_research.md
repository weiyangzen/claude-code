# 研究文档：src/services/AgentSummary/agentSummary.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 生成时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 1. 场景与职责

`src/services/AgentSummary/agentSummary.ts` 是 Claude Code 中负责**后台子代理周期性进度摘要**的独立服务模块。它的核心职责是：

- **为谁服务**：Coordinator 模式下的后台子代理（sub-agent / local_agent task）。
- **做什么**：每隔约 30 秒，基于子代理当前的对话上下文，fork 出一个轻量级代理查询，生成一句 3-5 词的进度摘要（如 "Reading runAgent.ts"）。
- **结果去向**：
  1. 写入 `AppState.tasks[taskId].progress.summary`，供 UI 任务面板实时展示；
  2. 若 SDK 进度摘要开关开启，通过 `emitTaskProgress` 发送给 SDK 消费者（如 VS Code 子代理面板）。

该模块本身**不持有任何持久状态**，纯函数式地暴露一个 `startAgentSummarization(...)` 入口，返回 `{ stop: () => void }` 句柄，由调用方（`AgentTool.tsx`、`agentToolUtils.ts`）在代理生命周期结束时调用停止。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| **周期性摘要** | 让用户/消费者无需展开完整对话即可感知后台代理正在做什么 |
| **Prompt Cache 共享** | 复用主代理的 `CacheSafeParams`，使 fork 查询能命中父对话的 prompt cache，降低延迟与 token 成本 |
| **工具强制禁用** | 摘要只需要文本生成，不允许调用任何工具；通过 `canUseTool` 回调 deny，而非传空 `tools` 数组（后者会改变 cache key） |
| **去重/新鲜度控制** | 每次摘要 prompt 会带上前一次摘要内容，要求模型说点 "NEW"，避免重复 |
| **生命周期隔离** | 摘要失败或异常不影响主代理运行；`stopped` 标志 + `AbortController` 保证安全清理 |

---

## 3. 具体技术实现

### 3.1 模块导出与常量

```typescript
const SUMMARY_INTERVAL_MS = 30_000

export function startAgentSummarization(
  taskId: string,
  agentId: AgentId,
  cacheSafeParams: CacheSafeParams,
  setAppState: TaskContext['setAppState'],
): { stop: () => void }
```

- `taskId`：对应 `AppState.tasks` 中的任务 ID，用于状态更新。
- `agentId`：子代理的品牌化 ID (`AgentId`)，用于从 `sessionStorage` 读取当前对话 transcript。
- `cacheSafeParams`：来自 `runAgent.ts` 的 `onCacheSafeParams` 回调，包含 system prompt、user/system context、tool use context、fork context messages。
- `setAppState`：根级状态更新函数，直接操作 `AppState`。

### 3.2 Prompt 工程

```typescript
function buildSummaryPrompt(previousSummary: string | null): string
```

Prompt 要求模型：
- 用 **现在进行时**（-ing）描述最近动作；
- **3-5 个词**；
- 必须提到 **文件或函数名**，不能是分支名；
- **禁止调用工具**（通过 `canUseTool` 回调在运行时 enforce）。

若存在 `previousSummary`，会在 prompt 中附加：
```
Previous: "xxx" — say something NEW.
```
这通过闭包变量 `previousSummary` 在模块内逐轮传递实现。

### 3.3 核心执行循环 `runSummary()`

执行流程（agentSummary.ts:61-155）：

1. **检查 `stopped`**：若已停止立即返回。
2. **读取当前 transcript**：
   ```typescript
   const transcript = await getAgentTranscript(agentId)
   ```
   从磁盘 JSONL (`agent-${agentId}.jsonl`) 加载完整对话链。
3. **消息数量阈值**：若消息少于 3 条，跳过本次摘要（`return`），由 `finally` 重新调度下一轮。
4. **清理不完整工具调用**：
   ```typescript
   const cleanMessages = filterIncompleteToolCalls(transcript.messages)
   ```
   移除 assistant message 中存在未配对 `tool_result` 的 `tool_use` 块，防止 API 400 错误。
5. **构建 fork 参数**：
   ```typescript
   const forkParams: CacheSafeParams = {
     ...baseParams,               // 显式丢弃原始 forkContextMessages
     forkContextMessages: cleanMessages,
   }
   ```
   注释强调：必须从闭包中**丢弃** `forkContextMessages`，否则原始消息会被定时器终身固定，导致摘要无法跟随主对话演进。
6. **创建 AbortController**：每轮摘要独立一个 `AbortController`，支持单轮取消。
7. **调用 `runForkedAgent`**：
   ```typescript
   const result = await runForkedAgent({
     promptMessages: [createUserMessage({ content: buildSummaryPrompt(previousSummary) })],
     cacheSafeParams: forkParams,
     canUseTool: async () => ({ behavior: 'deny', ... }),
     querySource: 'agent_summary',
     forkLabel: 'agent_summary',
     overrides: { abortController: summaryAbortController },
     skipTranscript: true,
   })
   ```
   - `querySource: 'agent_summary'`：在 `query.ts` 中被识别为 ephemeral 调用，**不持久化** `contentReplacement` 记录；在 `withRetry.ts` 中不属于 foreground 529 retry 源，遇到 529 会立即失败而非重试。
   - `skipTranscript: true`：不将摘要消息写入子代理的 sidechain JSONL，避免污染历史。
8. **解析结果**：遍历返回的 `assistant` 消息，跳过 `isApiErrorMessage`，提取第一个非空 `text` block。
9. **更新状态**：
   ```typescript
   previousSummary = summaryText
   updateAgentSummary(taskId, summaryText, setAppState)
   ```
10. ** finally 块**：
    - 清空 `summaryAbortController`；
    - 若未停止，调用 `scheduleNext()` 重新设置 30s 定时器。

### 3.4 停止与清理 `stop()`

```typescript
function stop(): void
```

- 设置 `stopped = true`；
- `clearTimeout(timeoutId)` 取消待执行的下一轮；
- `summaryAbortController?.abort()` 中断正在进行的摘要 API 调用；
- 所有操作幂等，重复调用安全。

### 3.5 关键数据结构

| 类型/变量 | 说明 |
|-----------|------|
| `CacheSafeParams` | `forkedAgent.ts:57` 定义，包含 `systemPrompt`, `userContext`, `systemContext`, `toolUseContext`, `forkContextMessages` |
| `AgentId` | `src/types/ids.ts` 品牌类型，格式 `a<label>-<16hex>` |
| `TaskContext['setAppState']` | `src/Task.ts:38`，即 `(f: (prev: AppState) => AppState) => void` |
| `previousSummary` | 模块级闭包字符串，用于去重提示 |

---

## 4. 关键代码路径与文件引用

### 4.1 本文件内部路径

```
startAgentSummarization()
  → scheduleNext()
    → setTimeout(runSummary, 30_000)
      → runSummary()
        → getAgentTranscript(agentId)          [sessionStorage.ts:4190]
        → filterIncompleteToolCalls(messages)  [runAgent.ts:866]
        → buildSummaryPrompt(previousSummary)  [agentSummary.ts:28]
        → runForkedAgent({...})                [forkedAgent.ts:489]
        → updateAgentSummary(taskId, ...)      [LocalAgentTask.tsx:359]
        → scheduleNext() // in finally
  → return { stop }
```

### 4.2 上游调用方（谁启动摘要）

| 文件 | 场景 | 代码位置 |
|------|------|----------|
| `src/tools/AgentTool/AgentTool.tsx` | **同步/前台代理**在 `runAgent()` 启动后，通过 `onCacheSafeParams` 回调启动；若代理被 background，前台摘要停止，后台重新启动新摘要 | 852-857 (foreground), 934-938 (backgrounded) |
| `src/tools/AgentTool/agentToolUtils.ts` | **纯异步代理生命周期**（`runAsyncAgentLifecycle`），在 `makeStream()` 启动前通过 `onCacheSafeParams` 启动 | 543-551 |
| `src/tools/AgentTool/resumeAgent.ts` | 恢复代理时，通过 `enableSummarization` 标志控制是否启用 | 250-253 |

### 4.3 下游依赖（谁被调用）

| 文件 | 被调用符号 | 作用 |
|------|-----------|------|
| `src/utils/forkedAgent.ts` | `runForkedAgent`, `CacheSafeParams` | 执行 fork 查询，隔离状态，跟踪 usage |
| `src/utils/sessionStorage.ts` | `getAgentTranscript` | 从磁盘读取子代理对话历史 |
| `src/tools/AgentTool/runAgent.ts` | `filterIncompleteToolCalls` | 清理未完成的 tool_use/tool_result 对 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `updateAgentSummary` | 将摘要写入 AppState 并可选 emit SDK 事件 |
| `src/utils/messages.ts` | `createUserMessage` | 构造摘要 prompt 消息 |
| `src/utils/debug.ts` | `logForDebugging` | 调试日志（带 `[AgentSummary]` 前缀） |
| `src/utils/log.ts` | `logError` | 异常日志 |

---

## 5. 依赖与外部交互

### 5.1 与 `forkedAgent.ts` 的交互

`runForkedAgent` 是本模块的核心执行引擎。关键约定：

- **Cache 共享**：`agentSummary.ts` 显式注释说明**不能**设置 `maxOutputTokens`，因为 `claude.ts` 会根据该值 clamp `budget_tokens`，从而改变 thinking config，导致 cache key 不匹配、cache miss。
- **工具保留**：`tools` 仍保留在请求中（用于 cache key 匹配），但运行时通过 `canUseTool` 回调全部 deny。
- **状态隔离**：`runForkedAgent` 内部调用 `createSubagentContext()` 克隆 `readFileState`、`contentReplacementState` 等可变状态，确保摘要查询不会污染主代理。
- **无 transcript 记录**：`skipTranscript: true` 使得 `runForkedAgent` 不会为摘要生成独立的 sidechain agentId 和 JSONL 记录。

### 5.2 与 `sessionStorage.ts` 的交互

`getAgentTranscript(agentId)` 直接读取子代理的 sidechain JSONL 文件（路径：`projects/<sanitizedCwd>/<sessionId>/subagents/agent-<agentId>.jsonl`）。该函数：

- 调用 `loadTranscriptFile` 解析全部消息；
- 按 `agentId` 和 `isSidechain` 过滤；
- 找到最新 leaf message 后通过 `buildConversationChain` 重建完整链；
- 返回 `{ messages, contentReplacements }`（摘要服务只使用 `messages`）。

**注意**：由于 transcript 是异步写入磁盘的，`getAgentTranscript` 读取到的消息可能略滞后于主代理内存中的最新状态，这是设计上的可接受延迟。

### 5.3 与 `AgentTool.tsx` / `agentToolUtils.ts` 的生命周期配合

- **前台代理** (`AgentTool.tsx` 同步路径)：
  - `stopForegroundSummarization` 在代理完成或被 background 时调用；
  - 若用户将代理 background，前台摘要立即停止，后台闭包内新建 `stopBackgroundedSummarization`。
- **纯异步代理** (`agentToolUtils.ts`):
  - `stopSummarization` 在 `try` 正常完成、`catch` 异常、`finally` 之前显式调用，确保摘要服务与代理生命周期同步结束。

### 5.4 与 SDK 的交互

`updateAgentSummary` (LocalAgentTask.tsx:359) 在写入 `AppState` 后，若 `getSdkAgentProgressSummariesEnabled()` 为 true，会调用 `emitTaskProgress` 发送 `task_progress` 系统事件。该开关由 `src/bootstrap/state.ts` 维护，通常在 SDK/Tungsten 模式下启用。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 代码位置 |
|------|------|----------|
| **Cache 失效陷阱** | 若未来某人在 `runForkedAgent` 调用中加上 `maxOutputTokens`，会改变 `budget_tokens`，导致 thinking config 不匹配，prompt cache 失效。文件内已有大写注释警告。 | agentSummary.ts:100-104 |
| **重叠摘要** | 若某次 `runSummary` 执行时间超过 30s（如 API 极慢），下一次不会启动，因为 `scheduleNext` 在 `finally` 中才调用。这是有意设计的防重叠机制，但极端情况下摘要更新会滞后。 | agentSummary.ts:148-154 |
| **Transcript 读取失败静默跳过** | `getAgentTranscript` 若文件不存在返回 `null`，此时消息数视为 0，仅记录 debug log 即跳过。无告警。 | agentSummary.ts:68-75 |
| **API 错误消息过滤** | 若模型返回 `isApiErrorMessage` 的 assistant message，会被跳过并继续扫描下一条；若全部消息都是 API 错误，则本轮无摘要产出，但 `previousSummary` 不会被更新。 | agentSummary.ts:126-132 |
| **无单元测试覆盖** | 仓库中未找到针对 `agentSummary.ts` 或 `startAgentSummarization` 的测试文件。 | — |

### 6.2 边界行为

- **消息数 < 3 不摘要**：认为对话上下文不足，直接跳过，30s 后再试。
- **工具调用未完成时不摘要**：`filterIncompleteToolCalls` 会移除含未完成 `tool_use` 的 assistant message。若清理后消息链变短，仍正常传给 `runForkedAgent`。
- **停止后异常静默**：`catch` 块中若 `stopped` 为 true，则错误被吞掉不打印，避免停止时的竞态报错污染日志。

### 6.3 改进建议

1. **增加测试覆盖**
   - 建议补充单元测试：模拟 `getAgentTranscript` 返回不同消息数、模拟 `runForkedAgent` 返回摘要/错误/空结果、验证 `updateAgentSummary` 调用次数与参数、验证 `stop()` 的幂等性。

2. **摘要失败退避**
   - 当前无论 `runForkedAgent` 失败多少次，都是固定 30s 重试。可考虑连续失败时指数退避（如 30s → 60s → 120s），减少 API 压力与无意义开销。

3. **摘要质量监控**
   - 可记录摘要长度、模型是否遵守 "3-5 词" 约束的 telemetry，用于后续 prompt 调优。

4. **明确 `querySource` 类型安全**
   - `'agent_summary'` 是字符串字面量，未在 `QuerySource` 联合类型中显式声明（`QuerySource` 实际为宽松类型）。建议将其加入类型定义，以获得编译期检查和 IDE 补全。

5. **考虑 transcript 读取的内存上限**
   - `getAgentTranscript` 底层 `loadTranscriptFile` 对大文件有 `MAX_TRANSCRIPT_READ_BYTES = 50MB` 保护，但摘要服务本身未对消息数量做上限截断。若子代理运行极长时间，传给 `runForkedAgent` 的消息前缀可能过长。可考虑只取最近 N 条消息用于摘要。

---

## 7. 附录：相关文件完整清单

- `src/services/AgentSummary/agentSummary.ts` — 本研究目标文件
- `src/utils/forkedAgent.ts` — `runForkedAgent`, `CacheSafeParams`, `createSubagentContext`
- `src/utils/sessionStorage.ts` — `getAgentTranscript`, `getAgentTranscriptPath`, `loadTranscriptFile`
- `src/tools/AgentTool/runAgent.ts` — `filterIncompleteToolCalls`, `runAgent`
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx` — `updateAgentSummary`, `AgentProgress`, `emitTaskProgress`
- `src/tools/AgentTool/AgentTool.tsx` — 前台/后台摘要调用方
- `src/tools/AgentTool/agentToolUtils.ts` — 纯异步代理摘要调用方
- `src/tools/AgentTool/resumeAgent.ts` — 恢复代理时的摘要开关判断
- `src/bootstrap/state.ts` — `getSdkAgentProgressSummariesEnabled`
- `src/utils/task/sdkProgress.ts` — `emitTaskProgress`
- `src/utils/messages.ts` — `createUserMessage`
- `src/utils/debug.ts` — `logForDebugging`
- `src/utils/log.ts` — `logError`
- `src/types/ids.ts` — `AgentId`, `asAgentId`
- `src/Task.ts` — `TaskContext`, `SetAppState`
- `src/query.ts` — `querySource` 相关的 `persistReplacements` 判断
