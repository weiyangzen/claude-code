# src/utils/queryContext.ts 研究文档

## 场景与职责

`queryContext.ts` 负责为 `query()` 调用构建 API 缓存键前缀所需的三大上下文片段：`systemPrompt`、`userContext`、`systemContext`。该模块被设计为独立文件，专门用于打破 import cycle：如果把这部分逻辑放在 `systemPrompt.ts` 或 `sideQuestion.ts` 中，会因为它们被 `commands.ts` 引用而形成循环依赖。只有入口点层级的文件（`QueryEngine.ts`、`cli/print.ts`）会导入这里。

## 功能点目的

1. **获取系统 Prompt 片段（`fetchSystemPromptParts`）**
   - 并发拉取 `defaultSystemPrompt`（来自 `getSystemPrompt`）、`userContext`（来自 `getUserContext`）、`systemContext`（来自 `getSystemContext`）。
   - 若设置了 `customSystemPrompt`，则跳过默认的 `getSystemPrompt` 和 `getSystemContext`，因为自定义 prompt 会完全替换默认系统提示，`systemContext` 追加到默认提示上已无意义。

2. **构建 Side Question 回退参数（`buildSideQuestionFallbackParams`）**
   - 在 SDK `side_question` 恢复场景（`cli/print.ts`）中使用：此时当前 turn 尚未完成，没有 `stopHooks` 快照，因此 `getLastCacheSafeParams()` 可能为 `null`。
   - 该函数根据原始输入重建 `CacheSafeParams`，使其与主循环即将发送的 system prompt 前缀尽可能一致，从而最大化缓存命中率。
   - 显式说明：如果主循环追加了额外的 extras（如 coordinator 模式、memory-mechanics prompt），缓存仍可能 miss，但这比返回 null 导致 side question 完全失败要好。

## 具体技术实现

- `fetchSystemPromptParts` 使用 `Promise.all` 并发请求三个上下文源，减少 I/O 等待。
- `buildSideQuestionFallbackParams` 内部：
  - 调用 `getMainLoopModel()` 获取当前模型。
  - 从 `appState.toolPermissionContext.additionalWorkingDirectories` 提取附加工作目录。
  - 组装 `systemPrompt`：`customSystemPrompt` 或 `defaultSystemPrompt` + 可选的 `appendSystemPrompt`。
  - 过滤掉正在流式传输中的 assistant message（`stop_reason === null`），与 `btw.tsx` 保持一致。
  - 构造一个最小可用的 `ToolUseContext`，其中大量回调（`setInProgressToolUseIDs`、`setResponseLength` 等）使用空函数占位，因为 side question 路径不需要这些功能。

## 关键代码路径与文件引用

- **本文件**：`src/utils/queryContext.ts`
- **调用方**：
  - `src/QueryEngine.ts` — 主查询引擎在 `ask()` 中调用 `fetchSystemPromptParts`。
  - `src/cli/print.ts` — SDK `side_question` 恢复路径调用 `buildSideQuestionFallbackParams`。
- **依赖**：
  - `src/commands.js` — `Command` 类型。
  - `src/constants/prompts.js` — `getSystemPrompt`。
  - `src/context.js` — `getSystemContext`、`getUserContext`。
  - `src/services/mcp/types.js` — `MCPServerConnection` 类型。
  - `src/state/AppStateStore.js` — `AppState` 类型。
  - `src/Tool.js` — `Tools`、`ToolUseContext` 类型。
  - `src/tools/AgentTool/loadAgentsDir.js` — `AgentDefinition` 类型。
  - `src/types/message.js` — `Message` 类型。
  - `src/utils/abortController.js` — `createAbortController`。
  - `src/utils/fileStateCache.js` — `FileStateCache` 类型。
  - `src/utils/forkedAgent.js` — `CacheSafeParams` 类型。
  - `src/utils/model/model.js` — `getMainLoopModel`。
  - `src/utils/systemPromptType.js` — `asSystemPrompt`。
  - `src/utils/thinking.js` — `shouldEnableThinkingByDefault`、`ThinkingConfig`。

## 依赖与外部交互

- 依赖 `getSystemPrompt`、`getSystemContext`、`getUserContext`，这些函数可能读取本地配置、CLAUDE.md、MCP 服务器状态等，涉及文件 I/O。
- 无直接网络交互。

## 风险、边界与改进建议

1. **缓存命中率的近似性**：`buildSideQuestionFallbackParams` 明确承认自己是一个“尽力而为”的回退方案。如果主循环在 `ask()` 中追加了 coordinator 用户上下文或 memory-mechanics prompt，而这里无法获知，缓存就会 miss。当前注释已充分说明这是可接受的，但在高并发或长会话中，miss 可能导致额外的 API 调用和成本。
2. **ToolUseContext 的空函数占位**：`setInProgressToolUseIDs`、`setResponseLength`、`updateFileHistoryState`、`updateAttributionState` 等回调被设为空函数。如果未来 side question 路径需要支持工具调用或文件历史追踪，这些空函数会导致状态丢失或 UI 不同步。
3. **additionalWorkingDirectories 的序列化**：`Array.from(appState.toolPermissionContext.additionalWorkingDirectories.keys())` 假设该集合是 `Set` 或 `Map`，若数据结构变更，这里会出错。
4. **import cycle 的脆弱性**：该文件存在的唯一理由就是打破循环依赖。如果未来 `QueryEngine.ts` 或 `print.ts` 的导入关系发生变化，可能重新引入 cycle，需要持续维护。
5. **改进建议**：
   - 考虑在 `QueryEngine.ts` 中把 `systemPrompt` 组装逻辑抽成一个可复用的纯函数，供 `queryContext.ts` 直接调用，从而彻底消除“两边手动对齐”的风险。
   - 为 `buildSideQuestionFallbackParams` 增加一个 `cacheHit` 返回标志，或在上层增加 metric，监控回退方案的缓存命中率，以便评估是否需要进一步精确化。
