# src/utils/queryHelpers.ts 研究文档

## 场景与职责

`queryHelpers.ts` 是 Claude Code 查询引擎的通用辅助函数集合，横跨消息规范化、孤立权限处理、文件状态缓存重建、Bash 工具提取等多个子领域。由于这些函数同时依赖 `context.ts`、`constants/prompts.ts` 等高依赖图节点，若把它们放在 `systemPrompt.ts` 或 `sideQuestion.ts` 中会导致 import cycle。因此它们被集中到此文件，供 `QueryEngine.ts`、`cli/print.ts`、`REPL.tsx` 等入口层调用。

## 功能点目的

### 1. 查询结果成功判定（`isResultSuccessful`）

判断一次查询/工具调用链的结果是否应视为成功，用于错误恢复和 UI 状态更新：
- 最后一条消息是 assistant 且最后内容块为 `text` / `thinking` / `redacted_thinking` → 成功。
- 最后一条消息是 user 且所有内容块都是 `tool_result` → 成功（模型只做了工具调用，用户消息全是结果）。
- `stopReason === 'end_turn'` 但 assistant 没有任何内容块 → 成功（模型决定无需回复，常见于 task_notification drain turns）。

### 2. 消息规范化生成器（`normalizeMessage`）

将内部 `Message` 类型转换为 SDK 所需的 `SDKMessage` 格式，按消息类型分支：
- **assistant**：通过 `normalizeMessages` 过滤空消息，附加 `session_id`、`uuid`、`parent_tool_use_id`。
- **user**：同样过滤并附加 `session_id`、`uuid`、`timestamp`、`isSynthetic`、`tool_use_result`；支持 MCP 元数据透传。
- **progress**：
  - `agent_progress` / `skill_progress`：递归规范化其内嵌消息。
  - `bash_progress` / `powershell_progress`：仅对 Remote/Container 环境做 30 秒节流，生成 `tool_progress` SDK 消息。

### 3. 孤立权限处理（`handleOrphanedPermission`）

当用户在查询间隙（如组件 unmount 后）对某个工具使用做出了权限决定，但该工具使用尚未被实际执行时，需要“追赶”执行：
- 在 assistant message 中定位对应的 `tool_use` block（通过 `toolUseID`）。
- 若权限为 allow，使用 `updatedInput` 覆盖原始输入。
- 构造一个永远返回该权限结果的 `canUseTool` 函数。
- 将 assistant message 推入 `mutableMessages`（若已存在则去重，防止 CCR resume 时重复）。
- 调用 `runTools` 执行工具，流式产出 `SDKMessage` 更新。

### 4. 从消息历史提取文件状态缓存（`extractReadFilesFromMessages`）

在 `--resume` 或冷启动时，根据历史消息重建 `FileStateCache`，避免重复读取磁盘：
- **第一遍扫描**：遍历 assistant message 中的 `tool_use`，收集 `FileReadTool`、`FileWriteTool`、`FileEditTool` 的 `tool_use_id` → 文件路径映射。
  - `FileReadTool` 仅缓存无 `offset`/`limit` 的全文件读取。
  - `FileWriteTool` 直接从 tool input 取内容。
  - `FileEditTool` 只记录路径，后续需要读磁盘获取编辑后内容。
- **第二遍扫描**：遍历 user message 中的 `tool_result`，按 `tool_use_id` 匹配：
  - 读取结果：去掉 `<system-reminder>` 块，去掉行号前缀，用消息时间戳作为缓存时间戳。
  - 写入结果：直接用 input 中的 content。
  - 编辑结果：调用 `readFileSyncWithMetadata` 读当前磁盘状态，用实际 `mtime` 作为时间戳。

### 5. 提取 Bash 工具 CLI 名称（`extractBashToolsFromMessages` / `extractCliName`）

从历史消息中收集顶层 CLI 命令名（如 `git`、`aws`、`vercel`），用于 analytics 或功能标记：
- 跳过环境变量赋值（`FOO=bar`）。
- 跳过 `sudo`。
- 返回去重后的 `Set<string>`。

## 具体技术实现

- `normalizeMessage` 是 `Generator<SDKMessage>`，允许调用方以流式方式消费转换后的消息，减少大消息数组的内存峰值。
- `handleOrphanedPermission` 的去重逻辑非常精细：不基于 `message.id`（因为流式响应可能把 `[text, tool_use]` 拆成两条共享同一 `message.id` 的记录），而是基于 `tool_use` block 的 `id` 字段，防止 `filterUnresolvedToolUses` 剥离 tool_use 后导致的误判。
- `extractReadFilesFromMessages` 使用 `ASK_READ_FILE_STATE_CACHE_SIZE = 10` 作为默认缓存大小，适合权限提示等轻量级场景；`QueryEngine` 等主路径会传入更大的 `maxSize`。

## 关键代码路径与文件引用

- **本文件**：`src/utils/queryHelpers.ts`
- **调用方**：
  - `src/cli/print.ts` — `extractReadFilesFromMessages` 用于 resume 时重建缓存；`buildSideQuestionFallbackParams` 所在文件（`queryContext.ts`）不直接调用这里，但 `normalizeMessage` 被 `QueryEngine` / `print.ts` 广泛使用。
  - `src/screens/REPL.tsx` — `extractBashToolsFromMessages` 用于 analytics。
  - `src/services/PromptSuggestion/speculation.ts` — 推测性提示相关。
- **依赖**：
  - `@anthropic-ai/sdk/resources/index.mjs` — `ToolUseBlock`。
  - `lodash-es/last.js` — 取数组最后一项。
  - `src/bootstrap/state.js` — `getSessionId`、`isSessionPersistenceDisabled`。
  - `src/entrypoints/agentSdkTypes.js` — `SDKMessage`。
  - `src/hooks/useCanUseTool.js` — `CanUseToolFn`。
  - `src/services/tools/toolOrchestration.js` — `runTools`。
  - `src/Tool.js` — `Tool`、`Tools`、`toolMatchesName`。
  - `src/tools/BashTool/toolName.js`、`src/tools/FileEditTool/constants.js`、`src/tools/FileReadTool/FileReadTool.js`、`src/tools/FileReadTool/prompt.js`、`src/tools/FileWriteTool/prompt.js` — 工具名和提示常量。
  - `src/types/message.js` — `Message`。
  - `src/types/textInputTypes.js` — `OrphanedPermission`。
  - `src/utils/debug.js`、`src/utils/envUtils.js`、`src/utils/errors.js`、`src/utils/file.js`、`src/utils/fileRead.js`、`src/utils/fileStateCache.js`、`src/utils/messages.js`、`src/utils/path.js`、`src/utils/permissions/PermissionPromptToolResultSchema.js`、`src/utils/processUserInput/processUserInput.js`、`src/utils/sessionStorage.js`。

## 依赖与外部交互

- `handleOrphanedPermission` 调用 `runTools`，可能触发子进程、网络请求、文件操作。
- `extractReadFilesFromMessages` 在 `FileEditTool` 路径上会同步读取磁盘（`readFileSyncWithMetadata`）。
- `normalizeMessage` 中的 `bash_progress` 分支依赖 `process.env.CLAUDE_CODE_REMOTE` 和 `CLAUDE_CODE_CONTAINER_ID`。
- 无直接网络输出（除了 `recordTranscript` 的本地文件写入）。

## 风险、边界与改进建议

1. **`normalizeMessage` 的 progress 节流逻辑硬编码**：`TOOL_PROGRESS_THROTTLE_MS = 30000` 且仅对 Remote/Container 生效，普通本地用户完全看不到 bash progress 的 `tool_progress` 消息。如果未来需要向本地 UI 暴露长时间运行命令的进度，需要修改此处。
2. **`extractReadFilesFromMessages` 的同步磁盘读取**：处理 `FileEditTool` 时，如果历史消息很长且包含大量编辑操作，会连续多次调用 `readFileSyncWithMetadata`，阻塞事件循环。虽然 `ASK_READ_FILE_STATE_CACHE_SIZE` 限制了缓存大小，但读取次数不受限制。
3. **行号前缀剥离的脆弱性**：`stripLineNumberPrefix` 被用于从 tool result 中恢复原始文件内容。如果 tool result 的格式发生变化（如行号前缀样式调整），缓存内容将包含污染数据，导致后续 diff 或上下文计算错误。
4. **`handleOrphanedPermission` 的 `canUseTool` 永远返回同一结果**：虽然这在孤立权限场景下是正确的，但如果 `runTools` 内部有子工具调用（nested tool use），子调用的权限决策也会继承同一 `allow` 结果，可能绕过某些本应二次确认的安全检查。
5. **`extractCliName` 的解析过于简单**：基于空白分割和正则，无法处理引号包裹的命令、管道、子 shell 等复杂 bash 语法。例如 `"my command"` 会被错误地识别为 `"my`。
6. **改进建议**：
   - 将 `extractReadFilesFromMessages` 中的 `FileEditTool` 磁盘读取改为异步批量读取，或引入 debounce，避免阻塞主线程。
   - 为 `extractCliName` 引入一个轻量级 shell parser（或复用项目内已有的 bash parser），提高命令提取准确率。
   - 考虑把 `TOOL_PROGRESS_THROTTLE_MS` 提取为环境变量或配置项，方便不同部署环境调整。
   - 在 `handleOrphanedPermission` 中增加 metric/logging，追踪孤立权限的发生频率和平均处理耗时。
