# 研究文档：src/utils/toolSearch.ts

## 场景与职责

本模块实现 **Tool Search（动态工具发现）** 的决策逻辑与辅助工具。当 MCP 工具或标记为 `shouldDefer` 的内置工具数量庞大时，若全部预加载到 prompt 中，会严重挤占上下文窗口。Tool Search 机制允许将这些工具标记为 `defer_loading: true`，模型通过调用 `ToolSearchTool` 按需拉取具体 schema，从而：

- 减少初始 prompt 的 token 占用；
- 解除对 MCP 工具总数的硬性限制；
- 通过 `tool_reference` beta 内容块实现工具动态发现。

本模块不负责 `ToolSearchTool` 本身的实现（位于 `src/tools/ToolSearchTool/`），而是提供**是否启用 Tool Search、如何启用、以及从消息历史中提取已发现工具**的全局判断逻辑。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `getToolSearchMode()` | 根据环境变量 `ENABLE_TOOL_SEARCH` 解析当前模式：`tst`（始终启用）、`tst-auto`（超阈值自动启用）、`standard`（禁用）。 |
| `isToolSearchEnabledOptimistic()` | 乐观检查：在不具备完整请求上下文时快速判断 Tool Search 是否“可能”启用，用于决定要不要把 `ToolSearchTool` 加入 base tools 列表。 |
| `isToolSearchEnabled()` | 确定性检查：综合模型是否支持 `tool_reference`、ToolSearchTool 是否在可用工具列表中、以及 `tst-auto` 模式下的阈值计算，给出最终是否启用的布尔值。 |
| `modelSupportsToolReference(model)` | 判断指定模型是否支持 `tool_reference`。采用负向列表（默认仅 `haiku` 不支持），新模型默认支持。 |
| `extractDiscoveredToolNames(messages)` | 扫描消息历史中的 `tool_reference` 块和 compact boundary 的 `preCompactDiscoveredTools`，返回已发现工具名集合。 |
| `getDeferredToolsDelta(tools, messages, scanContext?)` | 对比当前 deferred 工具池与消息历史中已 announced 的 `deferred_tools_delta` attachment，计算新增/移除差异。 |
| `isDeferredToolsDeltaEnabled()` | 控制 deferred 工具池的变更是否通过 system-reminder attachment（delta）形式通知模型，而非旧式的 `<available-deferred-tools>` 前缀。 |
| `isToolReferenceBlock(obj)` / `isToolReferenceWithName(obj)` | 运行时的 `tool_reference` 类型守卫（因该类型尚未进入 SDK 类型定义）。 |
| `getAutoToolSearchCharThreshold(model)` | 字符级阈值计算（token API 不可用时回退使用）。 |

## 具体技术实现

### 1. 模式解析（ENABLE_TOOL_SEARCH）

| 环境变量值 | 模式 | 说明 |
|-----------|------|------|
| `true` | `tst` | 始终启用 |
| `auto` | `tst-auto` | 默认 10% 上下文窗口阈值 |
| `auto:0` | `tst` | 0% 阈值 = 始终启用 |
| `auto:100` | `standard` | 100% 阈值 = 始终禁用 |
| `auto:N` (1-99) | `tst-auto` | N% 阈值 |
| `false` / 未设置 | `tst` / `standard` | 默认行为；若 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` 为 truthy 则强制 `standard` |

- `parseAutoPercentage` 负责解析并 clamp 到 `[0, 100]`。
- `isAutoToolSearchMode` 判断是否为 auto 系列值。

### 2. 乐观禁用的代理网关兼容逻辑

`isToolSearchEnabledOptimistic()` 中有一段针对第三方 API 代理的启发式禁用：

- 当 `ENABLE_TOOL_SEARCH` 未显式设置（空/undefined），且 provider 为 `firstParty`，但 `ANTHROPIC_BASE_URL` 不是 Anthropic 官方域名时，返回 `false`。
- 原因：许多第三方代理网关不支持 `tool_reference` beta 类型，会返回 400。
- 注释明确提到这**可能误伤**支持 `tool_reference` 的代理（如 LiteLLM passthrough、Cloudflare AI Gateway），并建议这些用户显式设置 `ENABLE_TOOL_SEARCH=true`。

### 3. 自动阈值计算（tst-auto）

`checkAutoThreshold` 流程：

1. **精确 token 计数**：调用 `countToolDefinitionTokens`（来自 `src/utils/analyzeContext.ts`），传入当前所有 deferred tools。结果被 `lodash memoize` 缓存，缓存 key 为 deferred tool names 的逗号拼接字符串。
2. **回退字符启发式**：若 token API 返回 null 或抛出异常，则计算每个 deferred tool 的 `name.length + description.length + inputSchema.length`，求和后与 `getAutoToolSearchCharThreshold(model)` 比较。
3. 字符启发式采用 `CHARS_PER_TOKEN = 2.5` 的粗略换算。

### 4. 已发现工具扫描

`extractDiscoveredToolNames(messages)` 遍历消息数组：

- 对 `system` + `subtype === 'compact_boundary'` 消息，读取 `compactMetadata.preCompactDiscoveredTools` 并加入集合（这是 compaction 对 tool_reference 消息的摘要保护机制）。
- 对 `user` 消息，遍历其 content array，找到 `type === 'tool_result'` 且 content 为 array 的块，再遍历其中的 `tool_reference` 块，提取 `tool_name`。

### 5. Deferred Tools Delta

`getDeferredToolsDelta` 扫描所有 `type === 'attachment'` 且 `attachment.type === 'deferred_tools_delta'` 的消息：

- 维护一个 `announced` 集合（按顺序应用 addedNames/removedNames）。
- 当前 `tools` 中仍被 deferred 但不在 `announced` 中的 → `added`。
- 在 `announced` 中但已完全不在 `tools` 池中的 → `removed`。
- 在 `announced` 中但已变为非 deferred（即现在直接加载）的 → **静默忽略**（不报告为 removed，避免模型误以为工具不可用）。
- 每次调用都会发送 `tengu_deferred_tools_pool_change` analytics 事件，携带 `callSite` 和 `querySource` 以便在 BigQuery 中区分不同调用路径（如主线程 attachment、subagent、compact 等）。

## 关键代码路径与文件引用

- **主实现**：`src/utils/toolSearch.ts`（756 行）
- **ToolSearchTool 实现**：`src/tools/ToolSearchTool/ToolSearchTool.ts`
- **Deferred 工具定义与格式化**：`src/tools/ToolSearchTool/prompt.ts`（`isDeferredTool`、`formatDeferredToolLine`、`TOOL_SEARCH_TOOL_NAME`）
- **API 调用层**：`src/services/api/claude.ts`（调用 `isToolSearchEnabled`、处理 `tool_reference` 相关逻辑）
- **Token 计数**：`src/utils/analyzeContext.ts`（`countToolDefinitionTokens`、`TOOL_TOKEN_COUNT_OVERHEAD`）
- **Compact 相关**：`src/services/compact/compact.ts`、`src/services/compact/sessionMemoryCompact.ts`（调用 `extractDiscoveredToolNames`）
- **工具执行/编排**：`src/services/tools/toolExecution.ts`（调用 `isToolSearchEnabled`）
- **Token 估算服务**：`src/services/tokenEstimation.ts`（调用 `isToolReferenceBlock`）
- **消息构造**：`src/utils/messages.ts`（`extractDiscoveredToolNames` 等）
- **Attachment 构造**：`src/utils/attachments.ts`（`getDeferredToolsDelta`、`isDeferredToolsDeltaEnabled`）
- **全局工具注册**：`src/tools.ts`（`isToolSearchEnabledOptimistic`）

## 依赖与外部交互

- **`lodash-es/memoize.js`**：`getDeferredToolTokenCount` 的缓存。
- **`src/services/analytics/growthbook.js`**：`getFeatureValue_CACHED_MAY_BE_STALE`（读取 `tengu_tool_search_unsupported_models`、`tengu_glacier_2xr` 等 flag）。
- **`src/services/analytics/index.js`**：`logEvent`（模式决策、delta 变化事件）。
- **`src/Tool.js`**：`Tool`、`Tools`、`ToolPermissionContext`、`toolMatchesName`。
- **`src/tools/AgentTool/loadAgentsDir.js`**：`AgentDefinition` 类型。
- **`src/tools/ToolSearchTool/prompt.js`**：`formatDeferredToolLine`、`isDeferredTool`、`TOOL_SEARCH_TOOL_NAME`。
- **`src/types/message.js`**：`Message` 类型。
- **`src/utils/analyzeContext.js`**：`countToolDefinitionTokens`、`TOOL_TOKEN_COUNT_OVERHEAD`。
- **`src/utils/array.js`**：`count`。
- **`src/utils/betas.js`**：`getMergedBetas`。
- **`src/utils/context.js`**：`getContextWindowForModel`。
- **`src/utils/debug.js`**：`logForDebugging`。
- **`src/utils/envUtils.js`**：`isEnvDefinedFalsy`、`isEnvTruthy`。
- **`src/utils/model/providers.js`**：`getAPIProvider`、`isFirstPartyAnthropicBaseUrl`。
- **`src/utils/slowOperations.js`**：`jsonStringify`。
- **`src/utils/zodToJsonSchema.js`**：`zodToJsonSchema`。

## 风险、边界与改进建议

### 风险

1. **代理网关误禁用**：`isToolSearchEnabledOptimistic` 对非官方 base URL 的 blanket disable 可能让使用支持 `tool_reference` 代理的用户在默认情况下失去 MCP 工具延迟加载能力，导致所有 MCP 工具被直接加载进上下文，token 占用激增。相关 issue 已在注释中引用（gh-31936 / CC-457）。
2. **Token 计数缓存失效粒度粗**：`getDeferredToolTokenCount` 的 memoize key 仅基于 deferred tool **名称列表**。若某工具的名称不变，但其 `prompt()` 返回的描述文本或 `inputJSONSchema` 发生变化（如 MCP 服务器动态更新工具定义），缓存不会失效，可能导致阈值判断错误。
3. **模型支持列表滞后**：`modelSupportsToolReference` 依赖默认列表 `['haiku']` 和 GrowthBook 覆盖。若未来某新模型系列不支持 `tool_reference` 但未及时加入列表，会导致 API 400 错误。

### 边界

- **负向模型检测**：新模型默认被认为支持 `tool_reference`，这与通常的“保守启用”策略相反。其好处是减少代码变更频率，坏处是一旦模型不支持就会直接报错。
- **字符回退仅覆盖 name+description+schema**：`calculateDeferredToolDescriptionChars` 未计入 API 序列化时的 JSON 括号、换行、转义等开销，因此字符阈值通常会比实际 token 阈值更“保守”（更容易触发 TST）。
- **Delta 扫描的 O(N) 开销**：`getDeferredToolsDelta` 每次调用都线性扫描整个消息历史中的 attachment。对于超长会话，这可能成为微热点。不过该函数通常只在每次 query 前调用一次，影响有限。

### 改进建议

1. **细化代理网关检测**：与其 blanket disable，不如在首次请求收到 400 且错误信息匹配 `tool_reference` 不支持时，自动回退到 `standard` 模式并缓存该会话结论。这样支持 `tool_reference` 的代理可以自动享受延迟加载。
2. **增强 token 计数缓存 key**：将工具描述的哈希（如 `tool.prompt().length` 和 schema 的 JSON 字符串长度）纳入 memoize key，或显式在 MCP 重连/刷新时清空 memoize cache。
3. **模型能力声明中心化**：将 `modelSupportsToolReference` 的负向列表迁移到集中式的模型能力注册表（如 `src/utils/model/capabilities.ts`），与 `getContextWindowForModel`、`getModelMaxOutputTokens` 等并列维护，降低遗漏风险。
4. **Delta 扫描优化**：在 `ContentReplacementState` 或会话状态中维护一个 `lastDeferredToolsDeltaIndex` 指针，避免每次从消息数组开头扫描。
