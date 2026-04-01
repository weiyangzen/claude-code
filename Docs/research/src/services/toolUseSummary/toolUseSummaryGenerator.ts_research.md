# src/services/toolUseSummary/toolUseSummaryGenerator.ts 研究文档

> 研究范围：目标文件本身、调用方（`src/query.ts`）、被调用方（`src/services/api/claude.ts` `queryHaiku`）、配置（`src/query/config.ts`）、消息工厂（`src/utils/messages.ts`）、类型/schema（`src/entrypoints/sdk/coreSchemas.ts`）、SDK 适配与透传（`src/QueryEngine.ts`、`src/remote/sdkMessageAdapter.ts`）、并行摘要通道（`src/utils/streamlinedTransform.ts`）、错误常量（`src/constants/errorIds.ts`）及工具函数（`src/utils/errors.ts`、`src/utils/slowOperations.ts`、`src/utils/systemPromptType.ts`、`src/utils/model/model.ts`）。
> 研究时间：2026-04-01
> 代码版本：以仓库当前 HEAD 为准

---

## 1. 场景与职责

### 1.1 核心职责
`toolUseSummaryGenerator.ts`（112 行）是 Claude Code 中**专责为已完成的工具批次生成人类可读摘要**的窄域模块。其职责边界非常清晰：

1. **批量工具执行后摘要生成**：在一组 `tool_use` + `tool_result` 完成后，调用轻量级 LLM（Haiku）生成一句概括性标签。
2. **移动应用/SDK 进度更新**：摘要最终作为 `tool_use_summary` 消息类型被 SDK 消费，用于在移动端或外部客户端展示单行动态进度。
3. **非阻塞异步处理**：摘要生成被包装为 Promise，在 `queryLoop` 的回合末尾"点火"，在下一回合开头 `yield`，利用模型流式响应的 5–30 秒窗口隐藏 Haiku 调用的 ~1 秒延迟。

### 1.2 业务场景

- **触发时机**：`src/query.ts` 的 `queryLoop` 每轮工具执行完毕后（`query_tool_execution_end` 检查点之后）。
- **消费端**：
  - **Headless/SDK 模式**：`src/QueryEngine.ts:959-968` 将 `tool_use_summary` 原样 `yield` 给 SDK 调用方。
  - **REPL/CLI 模式**：`src/remote/sdkMessageAdapter.ts:258-261` 明确忽略该消息类型（`return { type: 'ignored' }`），不在终端渲染。
- **显示约束**：系统提示词明确要求摘要适配"单行移动应用列表项，约 30 字符截断"，因此风格接近 `git commit subject`。

### 1.3 触发条件（严格四重门）

在 `src/query.ts:1415-1419` 中，只有以下四个条件**同时满足**才会触发：

1. `config.gates.emitToolUseSummaries === true`（环境变量 `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` 为 truthy）。
2. 当前回合实际产生了工具调用块：`toolUseBlocks.length > 0`。
3. 会话未被用户中断：`!toolUseContext.abortController.signal.aborted`。
4. **非子 Agent 会话**：`!toolUseContext.agentId`。注释明确说明 "subagents don't surface in mobile UI — skip the Haiku call"。

### 1.4 明确不承担的职责

- **单工具摘要**：各工具自身的 `getToolUseSummary(input)` 方法（定义在 `src/Tool.ts`）负责为单个 `tool_use` 块生成 spinner/权限对话框文案，与本模块无关。
- **Streamlined 模式聚合**：`src/utils/streamlinedTransform.ts` 实现了一套基于规则的工具分类计数逻辑（Searches/Reads/Writes/Commands/Other），生成 `streamlined_tool_use_summary`。这是与 AI 驱动摘要并行的另一条通道，服务于 CLI stream 输出。

---

## 2. 功能点目的

### 2.1 用户体验目标

为长时间、多步骤的 Agentic 执行提供**高层次的进度可见性**。当模型连续执行数十个工具调用时，用户（尤其是移动应用用户）需要一种比原始 `tool_use`/`tool_result` 块更抽象的反馈机制。`tool_use_summary` 正是在两个模型回合之间插入的"进度书签"。

### 2.2 摘要格式约束

系统提示词（`TOOL_USE_SUMMARY_SYSTEM_PROMPT`）对 Haiku 的输出施加了严格约束：

- **长度**：约 30 字符后截断（移动单行显示）。
- **风格**：`git-commit-subject`，不是完整句子。
- **时态**：过去时（verb in past tense）。
- **内容优先级**：保留动词 + 最具辨识度的名词；优先省略冠词、连接词、长路径。

典型示例：
```
Searched in auth/
Fixed NPE in UserService
Created signup endpoint
Read config.json
Ran failing tests
```

### 2.3 上下文增强

`generateToolUseSummary` 接受可选的 `lastAssistantText` 参数（助手最后一条 text block 的内容，截断至 200 字符）。该文本被注入用户提示词作为 "User's intent"，帮助 Haiku 理解用户原始意图，从而生成更相关的摘要。

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 `ToolInfo`（内部类型）
```typescript
type ToolInfo = {
  name: string    // 工具名称，如 "GlobTool"
  input: unknown  // 工具输入参数（原始对象）
  output: unknown // 工具输出结果（原始对象或 null）
}
```

#### 3.1.2 `GenerateToolUseSummaryParams`（导出类型）
```typescript
export type GenerateToolUseSummaryParams = {
  tools: ToolInfo[]
  signal: AbortSignal
  isNonInteractiveSession: boolean
  lastAssistantText?: string
}
```

#### 3.1.3 `ToolUseSummaryMessage`（推断结构）
虽然类型定义在构建产物中，但从 `src/utils/messages.ts:5105-5116` 可精确推断：
```typescript
type ToolUseSummaryMessage = {
  type: 'tool_use_summary'
  summary: string
  precedingToolUseIds: string[]  // 本摘要覆盖的 tool_use_id 列表
  uuid: string
  timestamp: string  // ISO 8601
}
```

### 3.2 Prompt 工程

#### System Prompt
```typescript
const TOOL_USE_SUMMARY_SYSTEM_PROMPT = `Write a short summary label describing what these tool calls accomplished. It appears as a single-line row in a mobile app and truncates around 30 characters, so think git-commit-subject, not sentence.

Keep the verb in past tense and the most distinctive noun. Drop articles, connectors, and long location context first.

Examples:
- Searched in auth/
- Fixed NPE in UserService
- Created signup endpoint
- Read config.json
- Ran failing tests`
```

#### User Prompt 结构
```
{User's intent (from assistant's last message): ...（可选，最多 200 字符）}

Tools completed:

Tool: {name}
Input: {truncated_input}
Output: {truncated_output}

...

Label:
```

### 3.3 数据截断策略

内部函数 `truncateJson(value, maxLength)` 对输入/输出分别截断：

- **截断阈值**：300 字符（`maxLength = 300`）。
- **截断标记**：`...`（保留 3 个字符位置给省略号）。
- **序列化失败**：返回 `[unable to serialize]`。
- **性能监控**：使用 `jsonStringify()`（`src/utils/slowOperations.ts:170`），内部包裹了 `slowLogging`，可对超慢 JSON 序列化进行预警。

### 3.4 主流程 `generateToolUseSummary`

```
输入参数
  │
  ▼
 tools.length === 0 ? ──Yes──► return null
  │ No
  ▼
构建 toolSummaries
  - 遍历 tools
  - 对每个 tool：truncateJson(input, 300) + truncateJson(output, 300)
  - 格式：Tool: {name}\nInput: {input}\nOutput: {output}
  - 用 \n\n 连接多个工具
  │
  ▼
构建 contextPrefix（lastAssistantText ? slice(0,200) : ''）
  │
  ▼
调用 queryHaiku({
  systemPrompt: [TOOL_USE_SUMMARY_SYSTEM_PROMPT],
  userPrompt: "{contextPrefix}Tools completed:\n\n{toolSummaries}\n\nLabel:",
  signal,
  options: {
    querySource: 'tool_use_summary_generation',
    enablePromptCaching: true,
    agents: [],
    isNonInteractiveSession,
    hasAppendSystemPrompt: false,
    mcpTools: [],
  }
})
  │
  ▼
解析 response.message.content
  - 过滤 block.type === 'text'
  - map 提取 text
  - join('').trim()
  │
  ▼
summary 为空 ? ──Yes──► return null
  │ No
  ▼
return summary
```

### 3.5 LLM 调用配置细节

通过 `queryHaiku()`（`src/services/api/claude.ts:3241-3291`）调用：

- **模型**：`getSmallFastModel()` → 默认 `getDefaultHaikuModel()`（当前为 Haiku 4.5，见 `src/utils/model/model.ts:131`）。可通过 `ANTHROPIC_SMALL_FAST_MODEL` 环境变量覆盖。
- **Thinking**：`{ type: 'disabled' }`，不消耗 thinking token。
- **工具**：`tools: []`，不暴露任何工具给 Haiku。
- **缓存**：`enablePromptCaching: true`，降低重复系统提示词的 token 成本。
- **QuerySource**：`'tool_use_summary_generation'`，用于 API 日志、VCR 录制和追踪。
- **VCR**：`queryHaiku` 内部使用 `withVCR`，测试/录制环境可回放。

### 3.6 异步时序：隐藏延迟的关键设计

`queryLoop` 是一个 `while (true)` 异步生成器。`pendingToolUseSummary` 的流转机制如下：

1. **点火（设置）**：在工具执行完成后的回合末尾，`nextPendingToolUseSummary = generateToolUseSummary(...).then(summary => createToolUseSummaryMessage(summary, toolUseIds)).catch(() => null)`，随后存入 `State.pendingToolUseSummary`（`src/query.ts:1722`）。
2. **燃烧（隐藏延迟）**：该 Promise 在后台解析，而主循环立即进入下一次迭代，开始流式调用主模型（Sonnet/Opus）。主模型的流式响应通常耗时 5–30 秒，足以覆盖 Haiku 的 ~1 秒延迟。
3. **产出（消费）**：在下一次循环迭代的开头、真正向客户端 `yield` 模型响应之前，`await pendingToolUseSummary` 并 `yield summary`（`src/query.ts:1055-1060`）。
4. **透传**：`src/QueryEngine.ts:959-968` 在 headless 模式下将其转换为 SDK 格式并 `yield`。

### 3.7 错误处理策略

```typescript
try {
  // ... 生成逻辑
} catch (error) {
  // Log but don't fail - summaries are non-critical
  const err = toError(error)
  err.cause = { errorId: E_TOOL_USE_SUMMARY_GENERATION_FAILED }
  logError(err)
  return null
}
```

- **非关键路径原则**：摘要生成失败绝不阻塞主查询流程。
- **错误标准化**：`toError()`（`src/utils/errors.ts:111`）将任意异常转为 `Error` 实例。
- **错误追踪**：附加 `errorId: 344`（`E_TOOL_USE_SUMMARY_GENERATION_FAILED`），便于生产日志定位。
- **双层防护**：`generateToolUseSummary` 内部 catch 返回 `null`；调用方在 `src/query.ts:1475-1481` 的 `.catch(() => null)` 再次兜底。

---

## 4. 关键代码路径与文件引用

### 4.1 调用链（完整）

```
SDK/Headless 调用方
    │
    ▼
src/QueryEngine.ts:queryLoop()
    │
    ▼
src/query.ts:query() ──► queryLoop()
    │
    ├── 工具执行完成后 ──► src/query.ts:1415-1482
    │   ├── 构建 toolInfoForSummary
    │   └── 调用 generateToolUseSummary()
    │           │
    │           ▼
    │   src/services/toolUseSummary/toolUseSummaryGenerator.ts:45
    │           │
    │           ▼
    │   调用 queryHaiku()
    │           │
    │           ▼
    │   src/services/api/claude.ts:3241
    │           │
    │           ▼
    │   queryModelWithoutStreaming()
    │           │
    │           ▼
    │   queryModel() ──► Anthropic API
    │
    ├── 下一回合开头 ──► src/query.ts:1055-1060
    │   └── await pendingToolUseSummary → yield summary
    │
    └── 消息流出到 SDK/REPL
```

### 4.2 直接相关文件索引

| 文件路径 | 职责 | 关键行号 |
|---------|------|---------|
| `src/services/toolUseSummary/toolUseSummaryGenerator.ts` | 批量摘要生成器 | 1-112 |
| `src/query.ts` | 触发、流转、状态管理 | 57 (import), 211 (State.pendingToolUseSummary), 1055-1060 (消费), 1415-1482 (触发) |
| `src/query/config.ts` | 功能开关快照 | 15-46 (QueryConfig, buildQueryConfig) |
| `src/services/api/claude.ts` | `queryHaiku()` 定义 | 3239-3291 |
| `src/utils/messages.ts` | 消息工厂 | 5101-5116 (createToolUseSummaryMessage) |
| `src/utils/systemPromptType.ts` | Prompt 类型品牌化 | 1-14 (SystemPrompt, asSystemPrompt) |
| `src/utils/slowOperations.ts` | 带性能监控的 JSON 序列化 | 170-190 (jsonStringify) |
| `src/utils/errors.ts` | 错误标准化 | 107-113 (toError) |
| `src/utils/log.ts` | 错误记录 | ~158 (logError) |
| `src/constants/errorIds.ts` | 错误 ID 定义 | 15 (`E_TOOL_USE_SUMMARY_GENERATION_FAILED = 344`) |
| `src/utils/model/model.ts` | 模型选择 | 36-38 (getSmallFastModel), 131 (getDefaultHaikuModel) |

### 4.3 SDK/Schema 相关文件

| 文件路径 | 职责 | 关键行号 |
|---------|------|---------|
| `src/QueryEngine.ts` | Headless 模式透传 | 959-968 |
| `src/remote/sdkMessageAdapter.ts` | REPL 端忽略该消息 | 258-261 |
| `src/entrypoints/sdk/coreSchemas.ts` | SDK Zod Schema | 1769-1777 (SDKToolUseSummaryMessageSchema) |
| `src/cli/print.ts` | CLI 消息过滤 | ~907 (排除 streamlined_tool_use_summary) |

### 4.4 并行摘要通道（规则驱动）

| 文件路径 | 职责 | 说明 |
|---------|------|------|
| `src/utils/streamlinedTransform.ts` | Streamlined 模式工具计数摘要 | 与 AI 摘要并行存在，互不影响 |

### 4.5 类型兼容引用

| 文件路径 | 职责 | 关键行号 |
|---------|------|---------|
| `src/query/stopHooks.ts` | 生成器联合类型包含 ToolUseSummaryMessage | 17 (import), 65-81 (handleStopHooks 签名) |
| `src/tools/AgentTool/runAgent.ts` | QueryMessage 联合类型包含 ToolUseSummaryMessage | 40 (import), 220-225 (QueryMessage 定义) |

---

## 5. 依赖与外部交互

### 5.1 直接代码依赖

```typescript
// 错误处理与日志
import { E_TOOL_USE_SUMMARY_GENERATION_FAILED } from '../../constants/errorIds.js'
import { toError } from '../../utils/errors.js'
import { logError } from '../../utils/log.js'

// 工具函数
import { jsonStringify } from '../../utils/slowOperations.js'
import { asSystemPrompt } from '../../utils/systemPromptType.js'

// LLM 服务
import { queryHaiku } from '../api/claude.js'
```

### 5.2 外部系统交互

- **Anthropic API**：通过 `queryHaiku → queryModelWithoutStreaming → queryModel` 调用 Haiku 模型。请求包含系统提示词、用户提示词、`querySource: 'tool_use_summary_generation'`。
- **VCR 录制系统**：`queryHaiku` 内部包裹 `withVCR`，在测试或录制模式下可缓存/回放响应。
- **Prompt Caching**：`enablePromptCaching: true` 让系统提示词享受 API 层面的缓存折扣。
- **AbortSignal**：传入的 `signal` 来自 `toolUseContext.abortController.signal`，用户中断时会取消正在进行的 Haiku 请求。

### 5.3 环境变量依赖

| 环境变量 | 用途 | 定义/消费位置 |
|---------|------|--------------|
| `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` | 功能总开关 | `src/query/config.ts:36-38` |
| `ANTHROPIC_SMALL_FAST_MODEL` | 覆盖默认 Haiku 模型 | `src/utils/model/model.ts:37` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 覆盖默认 Haiku 基座 | `src/utils/model/model.ts:132` |
| `USER_TYPE` | `isAnt` 判断（影响 dumpPromptsFetch 等周边） | `src/query/config.ts:39` |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### R1: 静默失败导致可观测性不足
`generateToolUseSummary` 在 catch 块中仅调用 `logError(err)` 并返回 `null`。没有结构化日志事件（`logEvent`），也没有 metrics，导致生产环境中难以统计摘要生成失败率、分析失败原因或监控延迟分布。

#### R2: Prompt 截断可能丢失关键上下文
`truncateJson` 对输入和输出各截断到 300 字符。对于输出巨大的工具（如 `GlobTool` 返回大量路径、`GrepTool` 返回大量匹配、`ReadFile` 读取大文件），Haiku 可能只看到 `"..."`，生成的摘要可能与实际结果偏差较大，甚至出现"幻觉"式概括。

#### R3: 子 Agent 完全跳过摘要
`!toolUseContext.agentId` 的硬性排除意味着所有子 Agent 执行的工具批次都不会产生摘要。如果未来移动端 UI 需要展示嵌套 Agent 的进度，该设计将成为瓶颈。

#### R4: 环境变量开关无动态切换能力
`emitToolUseSummaries` 在 `query()` 入口通过 `buildQueryConfig()` 快照一次（`src/query.ts:295`）。整个 `query()` 会话期间无法动态开启或关闭，必须重启进程或重新发起 `query()` 调用。

#### R5: 成本累积
每轮有工具调用的回合都会触发一次 Haiku API 调用。在长会话、高频工具调用场景下，即使 Haiku 单价低，token 成本仍会线性累积。当前仅依赖 `enablePromptCaching: true` 缓解，但用户提示词（含工具输入输出）部分无法被缓存。

#### R6: 无单元测试覆盖
在仓库中未找到针对 `generateToolUseSummary` 或 `truncateJson` 的单元测试。该模块的回归风险完全依赖集成测试或手工验证。

### 6.2 边界条件

| 边界条件 | 处理行为 |
|---------|---------|
| `tools` 为空数组 | 立即返回 `null`（`toolUseSummaryGenerator.ts:51-53`） |
| JSON 序列化失败 | `truncateJson` 返回 `[unable to serialize]` |
| LLM 返回空/空白内容 | 返回 `null`（`summary || null`） |
| LLM 调用抛出异常 | 记录错误（errorId 344），返回 `null` |
| 信号已中止 | 由调用方在触发前过滤，不点火摘要生成 |
| 子代理会话 | `src/query.ts:1419` 硬性跳过 |
| `lastAssistantText` 超长 | 在 Prompt 构建时截断至 200 字符 |

### 6.3 改进建议

#### 可观测性
1. **添加结构化埋点**：在 `catch` 块中增加 `logEvent('tengu_tool_use_summary_failed', { toolCount: tools.length, errorId: 344 })`；在成功路径增加 `logEvent('tengu_tool_use_summary_succeeded', { toolCount: tools.length, summaryLength: summary.length })`。
2. **延迟指标**：测量并上报 `generateToolUseSummary` 的端到端延迟（从调用到 Promise resolve）。

#### 性能与成本
3. **摘要 LRU 缓存**：对相同工具名组合 + 相似输入哈希的摘要进行本地缓存，避免重复调用 Haiku。
4. **智能截断/摘要**：对于超大输出工具，在调用 Haiku 前先做一层规则化摘要（如 "GlobTool returned 47 paths"），而非直接截断为 `"..."`。
5. **批量合并**：考虑将多轮短工具批次合并为一次摘要生成，降低 API 调用频次。

#### 功能扩展
6. **子 Agent 摘要支持**：移除或放宽 `!toolUseContext.agentId` 限制，改为通过配置或层级标识控制是否向移动端透传子 Agent 摘要。
7. **多语言/本地化**：当前系统提示词固定为英文，可根据 `toolUseContext` 中的用户语言偏好动态切换提示词语言。
8. **结构化输出**：使用 `outputFormat`（JSON mode）让 Haiku 返回 `{ summary: string }`，提高格式可控性，减少空内容或异常格式的概率。

#### 代码质量
9. **补充单元测试**：为 `generateToolUseSummary` 和 `truncateJson` 补充测试，覆盖空数组、序列化失败、LLM 返回空内容、正常摘要生成等场景。
10. **类型定义集中化**：`ToolUseSummaryMessage` 类型目前分散在构建产物和推断中，建议在源码侧建立明确的类型声明文件。

---

## 7. 附录：常量与配置速查

| 常量/配置 | 值 | 位置 |
|----------|-----|------|
| `E_TOOL_USE_SUMMARY_GENERATION_FAILED` | `344` | `src/constants/errorIds.ts:15` |
| `truncateJson` 截断长度 | `300` | `toolUseSummaryGenerator.ts:59-60` |
| `lastAssistantText` 截断长度 | `200` | `toolUseSummaryGenerator.ts:66` |
| 功能开关环境变量 | `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` | `src/query/config.ts:36-38` |
| 模型覆盖环境变量 | `ANTHROPIC_SMALL_FAST_MODEL` | `src/utils/model/model.ts:37` |
| 默认模型 | `getDefaultHaikuModel()`（当前 Haiku 4.5） | `src/utils/model/model.ts:131` |
| QuerySource 标识 | `'tool_use_summary_generation'` | `toolUseSummaryGenerator.ts:74` |
