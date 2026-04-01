# src/tools/WebSearchTool/WebSearchTool.ts 研究文档

> 文件路径：`/home/sansha/Github/claude-code-instructkr/src/tools/WebSearchTool/WebSearchTool.ts`
> 文件大小：13548 bytes
> 研究日期：2026-04-01

---

## 1. 场景与职责

`WebSearchTool.ts` 是 Claude Code 中 **网络搜索工具** 的核心实现文件。它封装了调用 Anthropic API 内建 `web_search_20250305` 工具的全部逻辑，包括：
- 定义输入/输出 Zod Schema；
- 构造 API 级别的工具 Schema；
- 通过流式请求与模型交互，解析返回的 `BetaContentBlock` 序列；
- 向 UI 层推送进度事件；
- 将搜索结果格式化为 `tool_result` 块供后续对话使用。

该工具属于 **只读（read-only）**、**并发安全（concurrency-safe）** 工具，默认启用（受模型与 provider 限制）。

---

## 2. 功能点目的

### 2.1 Schema 定义
- **inputSchema**：
  - `query`（必填，最小长度 2）：搜索查询字符串。
  - `allowed_domains`（可选）：仅允许来自这些域名的结果。
  - `blocked_domains`（可选）：排除这些域名的结果。
- **searchResultSchema**：定义单条搜索命中 `{ title, url }`。
- **outputSchema**：
  - `query`：实际执行的查询。
  - `results`：`(SearchResult | string)[]`，可混合搜索链接与模型附带的文本 commentary。
  - `durationSeconds`：搜索耗时（秒）。

### 2.2 工具启用策略（`isEnabled`）
- `firstParty` provider：始终启用。
- `vertex` provider：仅当主循环模型为 Claude 4.0+（`claude-opus-4`、`claude-sonnet-4`、`claude-haiku-4`）时启用。
- `foundry` provider：始终启用（Foundry 只分发支持 Web Search 的模型）。
- `bedrock` provider：未显式启用，返回 `false`。

### 2.3 权限检查（`checkPermissions`）
- 行为为 `passthrough`，不直接拦截。
- 建议用户在 `localSettings` 中配置针对 `WebSearch` 的允许规则。

### 2.4 输入校验（`validateInput`）
1. `query` 不能为空。
2. `allowed_domains` 与 `blocked_domains` 不能同时存在（互斥）。

### 2.5 核心调用逻辑（`call`）
- 构造一条用户消息：`"Perform a web search for the query: " + query`。
- 通过 `queryModelWithStreaming` 发起流式请求，携带 `extraToolSchemas: [toolSchema]`。
- **模型降级开关**：当 GrowthBook flag `tengu_plum_vx3` 为 `true` 时，使用 `getSmallFastModel()`（默认 Haiku）替代主循环模型，并强制 `toolChoice: { type: 'tool', name: 'web_search' }`，同时关闭 thinking。
- 流式解析过程中：
  - 追踪 `server_tool_use` 的 `tool_use_id`；
  - 从 `input_json_delta` 中实时提取 `query` 字段，推送 `query_update` 进度；
  - 当收到 `web_search_tool_result` 块时，推送 `search_results_received` 进度。
- 最终调用 `makeOutputFromSearchResponse` 聚合结果。

### 2.6 结果聚合（`makeOutputFromSearchResponse`）
- 遍历 API 返回的 `BetaContentBlock[]`。
- 预期块序列：`text` → `server_tool_use` → `web_search_tool_result` → `text`/`citation`（可重复多轮）。
- 将文本 commentary 以字符串形式、搜索成功结果以 `SearchResult` 对象形式压入 `results`。
- **错误处理**：若 `web_search_tool_result.content` 不是数组（即错误对象），记录错误日志并压入错误描述字符串。

### 2.7 结果映射（`mapToolResultToToolResultBlockParam`）
- 将 `Output` 序列化为面向模型的 `tool_result` 文本：
  - 开头：`Web search results for query: "{query}"\n\n`
  - 中间：交替输出 commentary 字符串或 `Links: <json>`
  - 结尾：强制提醒模型 `You MUST include the sources above in your response to the user using markdown hyperlinks.`

---

## 3. 具体技术实现

### 3.1 API Schema 构造

```typescript
function makeToolSchema(input: Input): BetaWebSearchTool20250305 {
  return {
    type: 'web_search_20250305',
    name: 'web_search',
    allowed_domains: input.allowed_domains,
    blocked_domains: input.blocked_domains,
    max_uses: 8, // 硬编码上限：最多 8 次搜索
  }
}
```

### 3.2 流式事件处理状态机

在 `call` 方法内部维护以下状态：

| 变量 | 类型 | 用途 |
|------|------|------|
| `allContentBlocks` | `BetaContentBlock[]` | 收集完整的 assistant content 块 |
| `currentToolUseId` | `string \| null` | 当前正在解析的 `server_tool_use` ID |
| `currentToolUseJson` | `string` | 累积 `input_json_delta` 的 JSON 片段 |
| `toolUseQueries` | `Map<string, string>` | 记录每个 tool_use_id 对应的实际 query |
| `progressCounter` | `number` | 自增进度 ID 计数器 |

流式事件处理分支：

1. **`event.type === 'assistant'`**：直接将其 `message.content` 追加到 `allContentBlocks`。
2. **`content_block_start` + `server_tool_use`**：初始化 `currentToolUseId` 与 `currentToolUseJson`。
3. **`content_block_delta` + `input_json_delta`**：
   - 追加 JSON 片段；
   - 用正则 `/"query"\s*:\s*"((?:[^"\\]|\\.)*)"/` 尝试提取完整 query；
   - 提取成功后通过 `jsonParse` 去转义，并推送 `query_update` 进度（去重）。
4. **`content_block_start` + `web_search_tool_result`**：
   - 根据 `tool_use_id` 查找对应 query；
   - 推送 `search_results_received` 进度，附带结果数。

### 3.3 进度类型定义

```typescript
type WebSearchProgress =
  | { type: 'query_update'; query: string }
  | { type: 'search_results_received'; resultCount: number; query: string };
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件 | 职责 |
|------|------|
| `src/tools/WebSearchTool/WebSearchTool.ts` | 工具主逻辑（本文件） |
| `src/tools/WebSearchTool/UI.tsx` | 渲染函数（调用前/中/后 UI） |
| `src/tools/WebSearchTool/prompt.ts` | 系统提示文本与工具名常量 |

### 4.2 直接依赖

| 依赖文件 | 说明 |
|----------|------|
| `@anthropic-ai/sdk/resources/beta/messages/messages.mjs` | `BetaContentBlock`、`BetaWebSearchTool20250305` 类型 |
| `src/utils/model/providers.js` | `getAPIProvider` |
| `src/utils/permissions/PermissionResult.js` | `PermissionResult` 类型 |
| `zod/v4` | Schema 校验 |
| `src/services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` |
| `src/services/api/claude.js` | `queryModelWithStreaming` |
| `src/Tool.js` | `buildTool`、`ToolDef` |
| `src/utils/lazySchema.js` | `lazySchema`（延迟构造 Zod Schema） |
| `src/utils/log.js` | `logError` |
| `src/utils/messages.js` | `createUserMessage` |
| `src/utils/model/model.js` | `getMainLoopModel`、`getSmallFastModel` |
| `src/utils/slowOperations.js` | `jsonParse`、`jsonStringify` |
| `src/utils/systemPromptType.js` | `asSystemPrompt` |
| `./prompt.js` | `getWebSearchPrompt`、`WEB_SEARCH_TOOL_NAME` |
| `./UI.js` | 渲染函数 |
| `../../types/tools.js` | `WebSearchProgress` 类型 |

### 4.3 调用方与注册位置

- **`src/tools.ts`**：将 `WebSearchTool` 实例加入 `getAllBaseTools()` 返回的全局工具列表。
- **`src/constants/tools.ts`**：引用 `WEB_SEARCH_TOOL_NAME`，用于定义异步代理允许使用的工具集合（`ASYNC_AGENT_ALLOWED_TOOLS`）。
- **`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts`**：在系统提示中提及 WebSearchTool 的能力（当文档未覆盖主题时建议使用）。
- **`src/services/compact/microCompact.ts`** / **`apiMicrocompact.ts`**：在 compact 逻辑中识别并处理 WebSearch 相关的工具结果（`WEB_SEARCH_TOOL_NAME` 属于 `COMPACTABLE_TOOLS` 和 `TOOLS_CLEARABLE_RESULTS`）。

---

## 5. 依赖与外部交互

### 5.1 Anthropic SDK 内建工具

`WebSearchTool` 并不直接调用公共搜索引擎 API，而是将 `web_search_20250305` 作为 Anthropic Messages API 的 **原生工具（native tool）** 注入请求。模型在生成过程中自主决定何时触发搜索，搜索执行由 Anthropic 服务端完成，客户端仅消费返回的 `web_search_tool_result` 块。

### 5.2 模型降级实验（`tengu_plum_vx3`）

通过 GrowthBook 开关控制：
- 开启时，使用更快的轻量模型（默认 Haiku）执行搜索子请求；
- 关闭时，沿用当前会话的主循环模型与 thinking 配置。

### 5.3 与 UI 层的耦合

`call` 方法通过 `onProgress` 回调（类型为 `ToolCallProgress<WebSearchProgress>`）向 REPL 渲染层推送事件。UI.tsx 消费这些事件并绘制进度行。

---

## 6. 风险、边界与改进建议

### 6.1 当前风险与边界

1. **`max_uses: 8` 硬编码**：搜索次数上限写死在 `makeToolSchema` 中。若服务端策略或模型能力变化，需要发版修改源码。
2. **部分 JSON 解析的健壮性**：`input_json_delta` 的 query 提取依赖正则匹配 `"query": "..."`。若未来 API 改变字段顺序或引入嵌套引号，正则可能失效（虽然 `jsonParse` 处理了转义，但正则本身无法处理所有合法 JSON 边界情况）。
3. **错误信息仅记录日志**：当 `web_search_tool_result` 返回错误对象时，仅调用 `logError` 并将错误码字符串推入 `results`，不会向用户展示更友好的错误提示，也不会让工具调用失败（返回的仍是 `result: true` 的 `ToolResult`）。
4. **`bedrock` provider 未启用**：`isEnabled` 对 `bedrock` 返回 `false`。若未来 Bedrock 支持 Web Search，需要显式添加白名单逻辑。
5. **无测试覆盖**：当前目录下无 `.test.ts` 或 `.spec.ts` 文件。`makeOutputFromSearchResponse` 和 `validateInput` 的边界行为（如错误块、混合 commentary、空 query）缺乏自动化验证。

### 6.2 改进建议

- **将 `max_uses` 配置化**：可通过 GrowthBook 开关或环境变量动态调整搜索次数上限，避免硬编码。
- **增强 JSON 提取的健壮性**：考虑使用增量 JSON 解析器（如 `partial-json`）替代正则，以更安全地从 `input_json_delta` 中提取字段。
- **改进错误用户体验**：在 `makeOutputFromSearchResponse` 遇到错误时，除了记录日志，可考虑向模型/用户返回更具操作性的提示（如重试建议、网络状态说明）。
- **增加测试覆盖**：建议补充单元测试，覆盖以下场景：
  - 正常多轮搜索的块序列解析；
  - `web_search_tool_result` 错误对象的聚合；
  - `allowed_domains` 与 `blocked_domains` 互斥校验；
  - 空 query 的输入校验。
- **监控搜索耗时与失败率**：可在 `call` 方法中增加 telemetry 事件，记录每次搜索的耗时、结果数、以及 `web_search_tool_result` 错误码的分布，便于后续优化。
- **明确 `src/types/tools.js` 的源码位置**：`WebSearchProgress` 类型通过 `../../types/tools.js` 导入，但该路径在源码树中不可见（可能是构建产物或 Bun 路径别名）。建议在项目中补充类型定义的源码映射，降低维护成本。
