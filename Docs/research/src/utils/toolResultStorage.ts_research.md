# 研究文档：src/utils/toolResultStorage.ts

## 场景与职责

本模块是 Claude Code 中**大型工具结果持久化与上下文预算控制**的核心基础设施。其职责分为两层：

1. **单工具级（per-tool）持久化**：当某个工具结果超过自身声明的 `maxResultSizeChars` 或系统默认 50K 字符时，将完整内容写入会话目录的磁盘文件，并向模型返回一个带预览的引用消息（`<persisted-output>`），避免超大结果直接挤占 API 上下文窗口。
2. **单消息级（per-message）聚合预算**：当一轮并行工具调用产生多个结果时，若它们在合并后的单条 user message 中总字符数超过阈值（默认 200K），则按从大到小的顺序把“新鲜”结果替换为磁盘引用，防止 N 个中等大小结果叠加导致上下文爆炸。

此外，模块还负责**跨轮次状态跟踪**（`ContentReplacementState`），确保同一工具结果在后续 query 中被一致处理（已替换的继续用缓存的预览字符串，已见过的未替换结果不再动），从而保护 prompt cache 前缀稳定。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `persistToolResult` | 将 `ToolResultBlockParam['content']` 异步写入 `projectDir/<sessionId>/tool-results/<tool_use_id>.{json\|txt}`，幂等（`wx` flag）。 |
| `maybePersistLargeToolResult` | 内部函数：按阈值判断是否需要持久化；空内容注入占位文本；含 image block 跳过。 |
| `processToolResultBlock` / `processPreMappedToolResultBlock` | 工具执行后的标准后处理入口，把原生工具输出映射为 API block 后再走持久化判断。 |
| `enforceToolResultBudget` | 核心聚合预算算法：扫描 message 历史，按 API-level user message group 评估，决定哪些 fresh tool results 需要替换。 |
| `applyToolResultBudget` | `query.ts` 的集成入口，负责 gate 检查、调用 enforcement、并将新替换记录写入 transcript。 |
| `ContentReplacementState` / `createContentReplacementState` / `cloneContentReplacementState` | 会话级状态容器，跟踪 `seenIds`（已见集合）和 `replacements`（预览字符串 Map）。 |
| `provisionContentReplacementState` | 冷启动或 resume 时初始化状态；若 feature flag 关闭返回 `undefined`。 |
| `reconstructContentReplacementState` / `reconstructForSubagentResume` | 从 transcript 的 `ContentReplacementRecord[]` 或父 subagent 状态重建替换决策，保证 resume/fork 后的 cache 一致性。 |
| `generatePreview` | 生成前 2000 字节的预览，优先在换行处截断，避免切断 mid-line。 |
| `isToolResultContentEmpty` | 判断工具结果是否为空/仅空白，用于注入空内容占位。 |

## 具体技术实现

### 1. 持久化文件布局与幂等写入

```
<projectDir>/<sessionId>/tool-results/<tool_use_id>.json   // array content
<projectDir>/<sessionId>/tool-results/<tool_use_id>.txt    // string content
```

- 目录通过 `ensureToolResultsDir()` 使用 `mkdir(..., { recursive: true })` 创建。
- 写入使用 `writeFile(..., { flag: 'wx' })`：因为同一 `tool_use_id` 的内容在确定性工具中是固定的，跳过重复写入可防止 microcompact 重放消息时反复落盘。
- 错误处理：仅 `EEXIST` 被忽略；其他文件系统错误记录日志并返回 `PersistToolResultError`，调用方回退到发送原始内容。

### 2. 单工具阈值与 GrowthBook 覆盖

`getPersistenceThreshold(toolName, declaredMaxResultSizeChars)` 逻辑：
1. 若 `declaredMaxResultSizeChars` 为 `Infinity`（如 `Read` 工具），直接返回 `Infinity`，跳过持久化。
2. 读取 GrowthBook flag `tengu_satin_quoll`（`Record<string, number>`），若该工具名存在且为正有限数，直接作为覆盖值。
3. 否则返回 `Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)`（50K）。

### 3. 空内容占位（inc-4586）

若工具结果为空字符串、空白字符串或空数组，`maybePersistLargeToolResult` 不持久化，而是将其替换为：

```
(${toolName} completed with no output)
```

这是为了防止空 `tool_result` 紧跟在 `</function_results>` 后出现 `\n\nHuman:` 的停止序列误匹配，导致模型零输出结束回合。

### 4. 聚合预算算法（per-message）

**步骤 1：候选收集** `collectCandidatesByMessage(messages)`
- 以 assistant message 为 group 边界（但同 ID 的 assistant fragment 不分割）。
- 与 `normalizeMessagesForAPI` 的合并语义保持一致：progress/attachment/system 不创建边界。
- 仅收集非空、非 image、且未带有 `<persisted-output>` 前缀的 `tool_result` 块。

**步骤 2：状态分区** `partitionByPriorDecision(candidates, state)`
- `mustReapply`：已在 `replacements` Map 中 → 直接复用缓存的预览字符串（零 I/O，字节级一致）。
- `frozen`：已在 `seenIds` 但未被替换 → 禁止再替换，否则会改变已缓存的 prompt 前缀。
- `fresh`：从未见过 → 本轮唯一可能触发新持久化的集合。

**步骤 3：选择替换** `selectFreshToReplace(fresh, frozenSize, limit)`
- 将 fresh 按 size 降序排列。
- 从大到小依次“选中”替换，直到 `frozenSize + remainingFreshSize <= limit`。
- 注意：扣除的是原始完整 size，而非替换后预览 size；注释说明这是可接受的近似（预览约 2K，远小于原始结果）。

**步骤 4：并发持久化与原子状态更新**
- 所有选中的候选通过 `Promise.all` 并发调用 `buildReplacement`（即 `persistToolResult` + `buildLargeToolResultMessage`）。
- 成功后在同一次循环中原子地 `state.seenIds.add(id)` + `state.replacements.set(id, preview)`，防止并发 reader（如 subagent）观察到 `seen` 但无 `replacement` 的不一致中间态。

### 5. 消息替换

`replaceToolResultContents(messages, replacementMap)` 遍历所有 user message，对匹配 `tool_use_id` 的 `tool_result` block 替换其 `content`。无替换需求的 message 按引用返回，避免不必要拷贝。

### 6. Resume / Fork 重建

- `reconstructContentReplacementState` 从 transcript 中的 `ContentReplacementRecord[]` 恢复 `replacements`，并将消息中所有候选 id 标记为 `seen`（无论当时是否被替换）。
- `reconstructForSubagentResume` 额外传入父状态的 `replacements` 作为 `inheritedReplacements`，用于填补 fork subagent 的 sidechain 记录缺口。

## 关键代码路径与文件引用

- **主实现**：`src/utils/toolResultStorage.ts`（~1040 行）
- **阈值常量**：`src/constants/toolLimits.ts`（`DEFAULT_MAX_RESULT_SIZE_CHARS`、`MAX_TOOL_RESULTS_PER_MESSAGE_CHARS`、`BYTES_PER_TOKEN`）
- **调用方（query 循环）**：`src/query.ts`（`applyToolResultBudget`）
- **调用方（REPL 状态管理）**：`src/screens/REPL.tsx`（`provisionContentReplacementState`、`reconstructContentReplacementState`）
- **调用方（subagent resume）**：`src/tools/AgentTool/resumeAgent.ts`（`reconstructForSubagentResume`）
- **调用方（工具执行）**：`src/services/tools/toolExecution.ts`、`src/utils/processUserInput/processBashCommand.tsx`、`src/utils/promptShellExecution.ts`（`processToolResultBlock`）
- **调用方（Bash/PowerShell 直接持久化）**：`src/tools/BashTool/BashTool.tsx`、`src/tools/PowerShellTool/PowerShellTool.tsx`（直接调用 `getToolResultPath`、`buildLargeToolResultMessage`）
- **调用方（MCP 输出存储）**：`src/utils/mcpOutputStorage.ts`（复用 `getToolResultsDir`）
- **调用方（文件系统权限）**：`src/utils/permissions/filesystem.ts`（复用 `getToolResultsDir`）
- **调用方（PDF 处理）**：`src/utils/pdf.ts`（复用 `getToolResultsDir`）
- **调用方（清理）**：`src/utils/cleanup.ts`（`TOOL_RESULTS_SUBDIR`）
- **调用方（会话恢复）**：`src/utils/sessionRestore.ts`、`src/utils/conversationRecovery.ts`（`ContentReplacementRecord` 类型）
- **调用方（日志类型）**：`src/types/logs.ts`（`ContentReplacementRecord` 类型）
- **调用方（ToolUseContext）**：`src/Tool.ts`（`ContentReplacementState` 类型）
- **调用方（forkedAgent）**：`src/utils/forkedAgent.ts`（`createContentReplacementState`、`cloneContentReplacementState`）
- **相关（microcompact）**：`src/services/compact/microCompact.ts` 内联引用了 `TIME_BASED_MC_CLEARED_MESSAGE`（`[Old tool result content cleared]`），并通过测试保证与主模块一致。

## 依赖与外部交互

- **`@anthropic-ai/sdk/resources/index.mjs`**：`ToolResultBlockParam` 类型。
- **`fs/promises`**、`**path**`（`join`）：文件系统操作。
- **`src/bootstrap/state.ts`**：`getOriginalCwd`、`getSessionId`。
- **`src/utils/sessionStorage.ts`**：`getProjectDir`。
- **`src/utils/slowOperations.ts`**：`jsonStringify`（用于 array content 序列化）。
- **`src/services/analytics/index.ts`**：`logEvent`（持久化事件、预算执行事件）。
- **`src/services/analytics/metadata.ts`**：`sanitizeToolNameForAnalytics`。
- **`src/services/analytics/growthbook.ts`**：`getFeatureValue_CACHED_MAY_BE_STALE`（feature flag）。
- **`src/utils/format.ts`**：`formatFileSize`。
- **`src/utils/errors.ts`**：`getErrnoCode`、`toError`。
- **`src/utils/debug.ts`**：`logForDebugging`。
- **`src/utils/log.ts`**：`logError`。
- **`src/types/message.ts`**：`Message` 类型。

## 风险、边界与改进建议

### 风险

1. **幂等写入假设**：`wx` flag 跳过重复写入的前提是“同一 `tool_use_id` 的内容确定性不变”。若某工具在相同输入下输出非确定性内容（如含时间戳），首次写入后后续重放将使用旧文件，模型可能看到 stale 数据。当前代码库中大多数工具输出是确定性的，但需警惕。
2. **并发状态观察窗口**：`enforceToolResultBudget` 在 `await Promise.all(...)` 前后分两步更新 `seenIds`。虽然注释声称“post-await 原子更新”，但在单线程事件循环中，若其他微任务在 await 解析后、循环继续前读取状态，理论上仍可能观察到不完整。实际影响有限，因为 Node.js 单线程且当前无显式并发 query。
3. **聚合预算近似误差**：`selectFreshToReplace` 用原始 size 而非替换后预览 size 估算剩余空间，可能导致“多替换一个结果”或“少替换一个结果”。由于预览约 2K，而触发替换的结果通常远大于此，误差方向是保守的（多替换），不会导致上下文超限。

### 边界

- **Image block 豁免**：任何含 `type: 'image'` 的 `tool_result` 内容完全跳过持久化，必须原样发往 API。这是模型视觉能力的必要代价。
- **Infinity 阈值豁免**：`maxResultSizeChars === Infinity` 的工具（如 `Read`）不仅不持久化，在聚合预算中也被 `skipToolNames` 排除，其大小不计入 fresh size。若 frozen + 剩余 fresh 仍超限，则接受超限（由 `Read` 自身的 `maxTokens` 约束）。
- **空内容注入**：空/空白结果被强制替换为 `(${toolName} completed with no output)`，长度通常 < 100 字符，不会触发持久化。
- **文件系统错误回退**：任何持久化失败（磁盘满、权限不足等）都静默回退到发送原始内容，保证可用性优先。

### 改进建议

1. **精确预算计算**：在 `selectFreshToReplace` 中预计算替换后的预览长度（可基于 `generatePreview` + `buildLargeToolResultMessage` 的模板长度快速估算），减少近似误差。
2. **持久化文件 TTL / 清理**：当前 `tool-results` 目录依赖会话级 `cleanup.ts` 清理。对于长期运行的远程会话或异常退出，可能留下大量文件。可考虑在 `enforceToolResultBudget` 中记录最后访问时间，并在 compaction 时清理老旧文件。
3. **状态序列化健壮性**：`ContentReplacementRecord` 目前只存储 `kind`、`toolUseId`、`replacement`。若未来预览模板格式变更，重建时直接复用旧字符串可保证 cache 稳定，但这也意味着旧格式会永久留在 resume 后的对话中。可考虑增加版本号以便渐进迁移。
4. **测试覆盖**：模块逻辑复杂（预算、分区、并发、resume），但项目中未找到对应的 `.test.ts` 文件。建议补充单元测试，尤其是 `collectCandidatesByMessage` 的边界分组和 `partitionByPriorDecision` 的状态机行为。
