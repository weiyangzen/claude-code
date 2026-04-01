# `src/utils/advisor.ts` 深度研究文档

> 文件路径：`src/utils/advisor.ts`  
> 研究日期：2026-04-01  
> 关联特性：Advisor Tool（服务器端顾问工具，内部代号 `tengu_sage_compass`）

---

## 1. 场景与职责

`advisor.ts` 是 Claude Code 中 **Advisor Tool（顾问工具）** 的核心配置与类型定义模块。该特性是一个**仅限第一方（first-party only）的 Beta 功能**，允许主循环模型在"实质性工作"之前调用一个**更强的评审模型（reviewer model）** 来获取建议。

### 1.1 业务场景

- **前置评审**：在写代码、提交解释、基于假设继续推进之前，主模型可以调用 `advisor` 工具，将整个对话历史自动转发给更强的顾问模型。
- **任务完成校验**：在宣布任务完成前调用 advisor，确保交付物经过复核。
- **陷入困境时**：当错误反复出现、方法不收敛、结果不符合预期时，寻求外部评审意见。
- **切换方案前**：在考虑改变解决思路时获取建议。

### 1.2 模块职责

`advisor.ts` 本身不直接发起 API 调用，而是承担以下职责：

1. **类型定义**：由于 Anthropic SDK 尚未公开发布 advisor 相关的 block 类型，模块内自行定义了 `AdvisorServerToolUseBlock`、`AdvisorToolResultBlock` 等类型（第 9–34 行）。
2. **特性开关控制**：通过 GrowthBook feature flag（`tengu_sage_compass`）、环境变量、以及 first-party-only beta 限制来决定 advisor 是否启用。
3. **模型可用性校验**：硬编码了支持 advisor 的模型族（`opus-4-6`、`sonnet-4-6`），并提供 `modelSupportsAdvisor` / `isValidAdvisorModel` 两个校验函数。
4. **用户配置入口**：提供 `canUserConfigureAdvisor` 判断用户是否可通过 `/advisor <model>` 命令或 `--advisor` CLI 参数自定义顾问模型。
5. **实验模型下发**：支持 GrowthBook 实验配置，在特定 base model 下强制覆盖 advisor 模型。
6. **Token 用量提取**：从 `BetaUsage` 中过滤出 `advisor_message` 类型的迭代用量，用于成本统计。
7. **系统提示注入**：导出 `ADVISOR_TOOL_INSTRUCTIONS`（第 130–145 行），作为 system prompt 的一部分告知主模型何时、如何调用 advisor。

---

## 2. 功能点目的

### 2.1  stronger reviewer model 的架构意图

Advisor Tool 的核心理念是 **"让较弱/较快的主循环模型在关键决策点咨询较强的评审模型"**。这与传统的单模型 agentic 流程不同：

- 主模型（base model）负责快速迭代、执行工具调用；
- 顾问模型（advisor model）不直接执行工具，而是**审阅完整对话历史**后给出高层次的策略建议；
- 该调用通过**服务器端 tool use**（`server_tool_use`，name 为 `advisor`）实现，对客户端而言是透明的 API 行为。

### 2.2 仅限 First-Party 的 Beta 策略

代码中多次强调 advisor 是 **first-party only**（`src/utils/advisor.ts` 第 64–67 行）：

```typescript
// The advisor beta header is first-party only (Bedrock/Vertex 400 on it).
if (!shouldIncludeFirstPartyOnlyBetas()) {
  return false
}
```

这意味着：
- **Bedrock / Vertex 用户无法使用**：这些第三方 API 网关通常会对未知的 beta header 返回 400 错误。
- **API 契约依赖服务器端**：`advisor` 作为 `server_tool_use`，需要后端 API 识别并路由到专门的 advisor 模型，这不是纯客户端能力。
- **渐进式 rollout**：通过 GrowthBook `tengu_sage_compass` 控制灰度，便于在 Anthropic 内部和第一方 API 用户中逐步验证。

### 2.3 用户可配置性与实验控制的分离

- 当 `canUserConfigure` 为 `true` 时，用户可以通过 `/advisor opus` 或 CLI `--advisor <model>` 自行选择顾问模型。
- 当 `canUserConfigure` 为 `false` 时，系统可能通过 `getExperimentAdvisorModels()` 从 GrowthBook 获取实验指定的 `baseModel` → `advisorModel` 映射，强制覆盖用户选择，用于 A/B 实验。

---

## 3. 具体技术实现

### 3.1 类型定义：补齐 SDK 未发布的类型（第 9–44 行）

由于 SDK 尚未公开发布 advisor block 类型，代码中手写了一套 TypeScript 类型，并带有 TODO 注释：

```typescript
// The SDK does not yet have types for advisor blocks.
// TODO(hackyon): Migrate to the real anthropic SDK types when this feature ships publicly
```

**关键类型：**

| 类型 | 结构 | 说明 |
|------|------|------|
| `AdvisorServerToolUseBlock` | `type: 'server_tool_use'`, `name: 'advisor'`, `input: object` | 主模型发起 advisor 调用的 block |
| `AdvisorToolResultBlock` | `type: 'advisor_tool_result'`, `tool_use_id`, `content` | 服务器返回的 advisor 结果，content 可能是普通文本、加密内容或错误码 |
| `AdvisorBlock` | 上述两者的联合类型 | 用于消息过滤和渲染时的类型收窄 |
| `isAdvisorBlock(param)` | 类型守卫函数 | 判断一个 content block 是否属于 advisor 相关 block |

`AdvisorToolResultBlock.content` 支持三种变体：
- `advisor_result`：正常文本建议；
- `advisor_redacted_result`：加密内容（可能用于内部审计或数据隔离）；
- `advisor_tool_result_error`：错误码，如 advisor 服务不可用。

### 3.2 配置读取：GrowthBook `tengu_sage_compass`（第 46–58 行）

```typescript
type AdvisorConfig = {
  enabled?: boolean
  canUserConfigure?: boolean
  baseModel?: string
  advisorModel?: string
}

function getAdvisorConfig(): AdvisorConfig {
  return getFeatureValue_CACHED_MAY_BE_STALE<AdvisorConfig>(
    'tengu_sage_compass',
    {},
  )
}
```

- Feature flag 名称为 `tengu_sage_compass`。
- 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 读取，**缓存可能过期**（详见第 6 节风险分析）。
- 返回值包含：是否启用、用户是否可配置、实验用的 base/advisor 模型对。

### 3.3 启用条件链（第 60–69 行）

`isAdvisorEnabled()` 采用三层短路逻辑：

1. **环境变量强制关闭**：`CLAUDE_CODE_DISABLE_ADVISOR_TOOL` 为 truthy 时直接返回 `false`。
2. **First-party 限制**：`shouldIncludeFirstPartyOnlyBetas()` 要求当前 provider 为 `firstParty` 或 `foundry`，且未设置 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`。
3. **GrowthBook 开关**：最终读取 `getAdvisorConfig().enabled`，默认为 `false`。

### 3.4 硬编码的模型支持（第 87–106 行）

```typescript
// @[MODEL LAUNCH]: Add the new model if it supports the advisor tool.
export function modelSupportsAdvisor(model: string): boolean {
  const m = model.toLowerCase()
  return (
    m.includes('opus-4-6') ||
    m.includes('sonnet-4-6') ||
    process.env.USER_TYPE === 'ant'
  )
}

// @[MODEL LAUNCH]: Add the new model if it can serve as an advisor model.
export function isValidAdvisorModel(model: string): boolean {
  // ... 同上
}
```

- `modelSupportsAdvisor`：判断**主循环模型**是否有能力调用 advisor 工具。
- `isValidAdvisorModel`：判断某个模型**是否可以被指定为 advisor 模型**。
- 两者当前逻辑完全一致，均硬编码了 `opus-4-6` 和 `sonnet-4-6` 两个模型族，并对内部员工（`USER_TYPE === 'ant'`）放行全部模型。
- `@[MODEL LAUNCH]` 注释明确提示：**每次新模型发布都需要手动更新这两个函数**，这是显著的维护负担（详见第 6 节）。

### 3.5 实验模型覆盖逻辑（第 75–85 行）

```typescript
export function getExperimentAdvisorModels():
  | { baseModel: string; advisorModel: string }
  | undefined {
  const config = getAdvisorConfig()
  return isAdvisorEnabled() &&
    !canUserConfigureAdvisor() &&
    config.baseModel &&
    config.advisorModel
    ? { baseModel: config.baseModel, advisorModel: config.advisorModel }
    : undefined
}
```

- 仅在 advisor 启用**且用户不可配置**时生效。
- 用于 GrowthBook A/B 实验：系统强制指定某 base model 使用某 advisor model，用户无感知。
- 实际覆盖逻辑在 `src/services/api/claude.ts` 第 1084–1094 行：当 `options.model` 与 `baseModel` 匹配时，将 `advisorOption` 覆盖为实验指定的 `advisorModel`。

### 3.6 Token 用量提取（第 115–128 行）

```typescript
export function getAdvisorUsage(
  usage: BetaUsage,
): Array<BetaUsage & { model: string }> {
  const iterations = usage.iterations as Array<{ type: string }> | null | undefined
  if (!iterations) {
    return []
  }
  return iterations.filter(
    it => it.type === 'advisor_message',
  ) as unknown as Array<BetaUsage & { model: string }>
```

- `BetaUsage.iterations` 是 SDK 类型中尚未完全公开的字段（注释提到 "Stainless types don't include iterations yet"）。
- 通过 `type === 'advisor_message'` 过滤出 advisor 模型产生的用量子项。
- 返回结果用于 `src/utils/tokens.ts` 的成本统计和 `src/cost-tracker` 相关的计费展示。

### 3.7 系统提示：`ADVISOR_TOOL_INSTRUCTIONS`（第 130–145 行）

这是一段直接注入到 system prompt 中的 Markdown 指令，核心要求包括：

- **调用时机**：在实质性工作**之前**调用；如果只是做方向性探索（读文件、看代码），则不需要调用。
- **完成前调用**：在认为任务完成前调用，且**必须先让交付物持久化**（写文件、stage 变更），因为 advisor 调用耗时较长，可能因会话中断而丢失未保存结果。
- **冲突处理**：如果已有证据指向某结论，而 advisor 给出相反建议，不要默默切换，应再次发起 "reconcile call" 明确请求裁决。
- **权重**：对 advisor 的建议应给予高度重视，除非有原始证据（代码、文件内容）直接反驳。

这段指令在 `src/services/api/claude.ts` 第 1366 行被条件性地追加到 system prompt 数组中。

### 3.8 API 层集成（以 `src/services/api/claude.ts` 为例）

在 API 调用构造阶段，advisor 的集成分为以下几个步骤：

1. **Beta Header 注入**（第 1076–1078 行）：只要 `isAdvisorEnabled()` 为 true，无论是否实际携带 `advisorModel`，都向 betas 数组追加 `ADVISOR_BETA_HEADER`（值为 `'advisor-tool-2026-03-01'`）。这是为了确保非 agentic 查询（如 compact、side_question）也能正确解析对话历史中已有的 advisor block。

2. **Advisor Model 解析**（第 1080–1116 行）：
   - 优先读取 `options.advisorModel`；
   - 若存在实验配置且 base model 匹配，则覆盖；
   - 校验 `modelSupportsAdvisor(options.model)` 和 `isValidAdvisorModel(normalizedAdvisorModel)`；
   - 通过校验后赋值给局部变量 `advisorModel`。

3. **System Prompt 注入**（第 1366 行）：`...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : [])`。

4. **Tool Schema 注册**（第 1386–1395 行）：将 advisor 作为 server tool 追加到 `extraToolSchemas`：
   ```typescript
   extraToolSchemas.push({
     type: 'advisor_20260301',
     name: 'advisor',
     model: advisorModel,
   } as unknown as BetaToolUnion)
   ```
   注意 `type` 使用的是 API 内部版本标识 `advisor_20260301`，与 beta header 的日期保持一致。

5. **Block 过滤保护**（第 1303–1306 行）：如果最终 betas 中不包含 `ADVISOR_BETA_HEADER`（例如某些代码路径意外移除），则在发送请求前调用 `stripAdvisorBlocks(messagesForAPI)` 清理掉所有 advisor block，防止 API 返回 400 错误。

6. **流式状态追踪**（第 2008–2048 行）：在流式响应处理中，通过 `isAdvisorInProgress` 布尔值追踪 advisor 调用的开始和结束，并记录调试日志和 analytics 事件（`tengu_advisor_tool_call`、`tengu_advisor_tool_interrupted`）。

### 3.9 消息层的 Block 过滤（`src/utils/messages.ts`）

`stripAdvisorBlocks`（第 5466–5494 行）用于在 advisor beta header 缺失时清理消息：

```typescript
export function stripAdvisorBlocks(
  messages: (UserMessage | AssistantMessage)[],
): (UserMessage | AssistantMessage)[] {
  // ...
  const filtered = content.filter(b => !isAdvisorBlock(b))
  // 如果过滤后只剩下 thinking/redacted_thinking/空文本，则插入占位文本 [Advisor response]
  // 防止消息内容为空导致 API 拒绝
}
```

这个兜底机制对于**恢复远程会话（teleport/resume）**尤为重要，因为旧会话的消息历史中可能包含 advisor block，而新会话的 provider 或配置可能已改变。

---

## 4. 关键代码路径与文件引用

### 4.1 核心定义文件

| 文件 | 行号 | 作用 |
|------|------|------|
| `src/utils/advisor.ts` | 1–145 | 类型定义、配置读取、模型校验、用量提取、系统提示 |
| `src/constants/betas.ts` | 31 | `ADVISOR_BETA_HEADER = 'advisor-tool-2026-03-01'` |

### 4.2 API 调用与流式处理

| 文件 | 行号 | 作用 |
|------|------|------|
| `src/services/api/claude.ts` | 192 | 导入 `ADVISOR_BETA_HEADER` |
| `src/services/api/claude.ts` | 150–155 | 从 `advisor.ts` 导入各类函数 |
| `src/services/api/claude.ts` | 700 | `advisorModel?: string` 出现在 `QueryOptions` 类型中 |
| `src/services/api/claude.ts` | 1073–1078 | 条件注入 advisor beta header |
| `src/services/api/claude.ts` | 1080–1116 | 解析并校验 advisor model |
| `src/services/api/claude.ts` | 1303–1306 | 无 beta header 时调用 `stripAdvisorBlocks` |
| `src/services/api/claude.ts` | 1366 | 条件追加 `ADVISOR_TOOL_INSTRUCTIONS` |
| `src/services/api/claude.ts` | 1386–1395 | 将 advisor 注册为 server tool schema |
| `src/services/api/claude.ts` | 2008–2017 | 流式响应中追踪 advisor tool 调用开始 |
| `src/services/api/claude.ts` | 2044–2049 | 流式响应中追踪 advisor tool 结果返回 |
| `src/services/api/claude.ts` | 2207, 2588, 2683 | 在生成的 assistant message 中写入 `advisorModel` 字段 |
| `src/services/api/claude.ts` | 2443–2450 | 用户中断流时记录 `tengu_advisor_tool_interrupted` |

### 4.3 命令与配置

| 文件 | 行号 | 作用 |
|------|------|------|
| `src/commands/advisor.ts` | 1–109 | `/advisor` 命令的实现：查看、设置、关闭 advisor model |
| `src/main.tsx` | 47 | 导入 advisor 相关函数 |
| `src/main.tsx` | 2117–2138 | CLI 初始化时解析 `--advisor` 参数 |
| `src/main.tsx` | 2637–2639, 3027–3029 | 将 `advisorModel` 写入 app state |
| `src/main.tsx` | 3813–3815 | 条件注册 `--advisor` CLI option |
| `src/utils/settings/types.ts` | 712–715 | `advisorModel` 的 Zod schema 定义（用户设置持久化） |

### 4.4 消息处理与渲染

| 文件 | 行号 | 作用 |
|------|------|------|
| `src/utils/messages.ts` | 74 | 导入 `isAdvisorBlock` |
| `src/utils/messages.ts` | 771 | `advisorModel` 出现在 `NormalizedAssistantMessage` 中 |
| `src/utils/messages.ts` | 1252, 1262–1268 | 追踪 errored advisor tool results |
| `src/utils/messages.ts` | 5223 | 注释说明 orphaned advisor block 会导致 API 400 错误 |
| `src/utils/messages.ts` | 5466–5494 | `stripAdvisorBlocks` 实现 |
| `src/components/Messages.tsx` | 22, 583–587 | 渲染 advisor tool result，判断是否为 advisor 结果消息 |
| `src/components/Message.tsx` | 12, 99–113, 560–572 | 将 advisor block 渲染为 `AdvisorMessage` 组件 |
| `src/query.ts` | 695 | 将 `advisorModel` 传入 `queryTracking` / API 调用选项 |

### 4.5 Token 与成本

| 文件 | 行号 | 作用 |
|------|------|------|
| `src/utils/tokens.ts` | 87–90 | 注释提到 "cast like advisor.ts"，处理 `iterations` 字段以统计 advisor token |

---

## 5. 依赖与外部交互

### 5.1 直接依赖模块

```typescript
import type { BetaUsage } from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'
import { shouldIncludeFirstPartyOnlyBetas } from './betas.js'
import { isEnvTruthy } from './envUtils.js'
import { getInitialSettings } from './settings/settings.js'
```

| 依赖 | 来源 | 作用 |
|------|------|------|
| `@anthropic-ai/sdk` | 外部 npm 包 | 提供 `BetaUsage` 类型（但 `iterations` 字段尚未完全类型化） |
| `growthbook.js` | 内部服务 | `getFeatureValue_CACHED_MAY_BE_STALE<AdvisorConfig>('tengu_sage_compass', {})` |
| `betas.js` | 内部工具 | `shouldIncludeFirstPartyOnlyBetas()` 限制 provider |
| `envUtils.js` | 内部工具 | `isEnvTruthy()` 解析环境变量布尔值 |
| `settings/settings.js` | 内部工具 | `getInitialSettings().advisorModel` 读取用户持久化配置 |

### 5.2 GrowthBook 集成细节

- **Flag 名称**：`tengu_sage_compass`
- **返回结构**：`{ enabled?: boolean, canUserConfigure?: boolean, baseModel?: string, advisorModel?: string }`
- **缓存语义**：`CACHED_MAY_BE_STALE` 后缀表明该读取可能命中本地缓存，不会每次都向 GrowthBook 服务端请求最新值。这意味着：
  - 优点：低延迟、不阻塞启动；
  - 缺点：feature flag 的变更（如紧急关闭）可能存在分钟级延迟；
  - 对于 advisor 这种可能涉及新模型上线的功能，缓存延迟可能导致用户在 flag 关闭后仍短暂看到 advisor 选项。

### 5.3 API 后端契约

- **Beta Header**：`advisor-tool-2026-03-01`
- **Server Tool Schema Type**：`advisor_20260301`
- **Block Types**：`server_tool_use` (name=`advisor`)、`advisor_tool_result`
- **行为**：客户端只需在 tools 数组中声明 advisor schema 并携带 beta header，实际的路由、模型调用、结果封装全部由 API 后端处理。

### 5.4 环境变量影响矩阵

| 环境变量 | 影响 |
|----------|------|
| `CLAUDE_CODE_DISABLE_ADVISOR_TOOL` | 强制关闭 advisor 功能（优先级最高） |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | 通过 `shouldIncludeFirstPartyOnlyBetas()` 间接关闭 advisor |
| `USER_TYPE=ant` | 内部员工可绕过模型族硬编码限制 |
| `ANTHROPIC_BETAS` | 用户自定义 betas，但不会自动注入 advisor header |

---

## 6. 风险、边界与改进建议

### 6.1 `CACHED_MAY_BE_STALE` 的缓存风险

**风险**：`getAdvisorConfig()` 使用 `getFeatureValue_CACHED_MAY_BE_STALE`，如果 GrowthBook 缓存更新延迟，可能出现以下场景：
- advisor 因后端问题被紧急关闭，但本地缓存仍显示 `enabled: true`，导致 API 调用携带 `advisor-tool-2026-03-01` header 后失败或产生异常行为。
- 实验配置（`baseModel` / `advisorModel`）变更后，用户仍沿用旧配置。

**建议**：
- 对于 advisor 这种强依赖后端能力的特性，在 API 调用前增加一次**轻量服务端能力探测**（如通过已有的模型列表接口或一个低成本的 ping），而不是完全依赖本地缓存的 flag。
- 或在 API 返回 400 且错误信息包含 unknown beta header 时，自动降级并清除 advisor block（当前已有 `stripAdvisorBlocks` 机制，但仅在 header 缺失时触发，未覆盖 header 存在但后端拒绝的情况）。

### 6.2 `@[MODEL LAUNCH]` 硬编码模型的维护负担

**风险**：`modelSupportsAdvisor` 和 `isValidAdvisorModel` 中硬编码了 `opus-4-6` 和 `sonnet-4-6`：

```typescript
m.includes('opus-4-6') || m.includes('sonnet-4-6')
```

每次新模型族发布（如未来的 `haiku-4-6`、`opus-4-7` 等），工程师必须手动找到这两个函数并更新。历史表明类似注释在 `betas.ts` 中已多次出现（`modelSupportsStructuredOutputs`、`modelSupportsAutoMode` 等），说明这是项目内的系统性技术债。

**建议**：
- 将 advisor 支持信息迁移到**模型能力表（capability matrix）**中，例如 `src/utils/model/modelSupportOverrides.ts` 或一个统一的 JSON/YAML 配置。
- 或者，在 `get3PModelCapabilityOverride` 模式基础上增加 `advisor` 能力键，让模型支持逻辑与模型元数据解耦，避免字符串匹配。
- 至少应将 `['opus-4-6', 'sonnet-4-6']` 提取为模块顶层的常量数组，减少重复和遗漏概率。

### 6.3 类型安全：SDK 类型缺失

**风险**：`AdvisorBlock` 类型是手工定义的，与真实 SDK/后端契约可能存在漂移。例如：
- `input` 被定义为 `{ [key: string]: unknown }`，但实际上 advisor 工具 "takes NO parameters"（见 `ADVISOR_TOOL_INSTRUCTIONS`），这个类型过于宽松。
- `BetaUsage.iterations` 和 `BetaToolUnion` 的强制类型断言（`as unknown as ...`）在 SDK 升级后可能失效。

**建议**：
- 在 SDK 正式发布 advisor 类型后，按 TODO 注释移除自定义类型，全面迁移到官方类型。
- 在此之前，可考虑在 CI 中增加一个**契约校验测试**：将后端 OpenAPI schema 或内部类型定义与 `advisor.ts` 中的类型进行比对，防止静默漂移。

### 6.4 用户配置与实验覆盖的边界模糊

**风险**：`getExperimentAdvisorModels()` 只在 `!canUserConfigureAdvisor()` 时返回实验模型，但 CLI 参数 `--advisor` 的处理在 `main.tsx` 中早于 app state 的完全初始化。如果 GrowthBook 缓存导致 `canUserConfigureAdvisor()` 在 CLI 解析时和实际 API 调用时返回值不一致，可能出现：
- CLI 允许传入 `--advisor` 参数；
- 但 API 调用时因 `canUserConfigureAdvisor()` 变为 `false` 而被实验覆盖，用户选择被静默忽略。

**建议**：
- 在 `main.tsx` 和 `claude.ts` 中统一 advisor model 的解析顺序，并增加调试日志（当前已有部分日志，但可更明确地标出 "user choice overridden by experiment"）。
- 在 `/advisor` 命令的回复中，如果当前 advisor model 被实验锁定，应明确告知用户 "Advisor model is currently locked by an experiment"。

### 6.5 `stripAdvisorBlocks` 的占位文本问题

当前实现中，如果过滤 advisor block 后消息只剩下 thinking/redacted_thinking/空文本，会硬编码插入 `"[Advisor response]"` 作为占位：

```typescript
filtered.push({
  type: 'text' as const,
  text: '[Advisor response]',
  citations: [],
})
```

**风险**：
- 这段文本对用户可见（尤其是在 transcript 导出或日志中），可能引起困惑。
- 如果消息被重新发送给 API，这段占位文本会成为对话历史的一部分，可能影响模型行为。

**建议**：
- 考虑使用更中性的占位符，或在 UI 层做特殊处理（不显示占位文本），仅在 API 序列化时保留一个最小有效内容。
- 或者，在 `stripAdvisorBlocks` 后检查消息是否变得完全无意义（如只有占位文本），如果是，则直接移除整条 assistant message，而不是保留一条空洞消息。

### 6.6 First-Party Only 与 Foundry 的歧义

`shouldIncludeFirstPartyOnlyBetas()` 的实现在 `betas.ts` 第 215–220 行为：

```typescript
return (
  (getAPIProvider() === 'firstParty' || getAPIProvider() === 'foundry') &&
  !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS)
)
```

但 `ADVISOR_BETA_HEADER` 的注释明确说 "first-party only (Bedrock/Vertex 400 on it)"，而 Foundry 是否真正支持 advisor 后端路由并不明确。如果 Foundry 只是透传了 beta header 但没有实际部署 advisor 服务，会导致客户端发送了 header 和 tool schema 但后端无法处理。

**建议**：
- 明确 advisor 在 Foundry 上的支持状态。如果不支持，应在 `isAdvisorEnabled()` 中显式排除 `foundry`，而不是依赖 `shouldIncludeFirstPartyOnlyBetas()` 的泛化逻辑。

---

## 7. 总结

`src/utils/advisor.ts` 是 Claude Code Advisor Tool 的**配置中枢和类型基座**。它通过 GrowthBook 控制功能开关，通过硬编码模型列表控制可用性，通过自定义类型弥补 SDK 的暂时缺失，并通过 `ADVISOR_TOOL_INSTRUCTIONS` 向主模型传达调用策略。

该模块本身不直接处理网络 I/O，但与 `src/services/api/claude.ts` 紧密协作：后者负责将 advisor 注册为 server tool、注入 beta header、追加系统提示，并在流式响应中追踪 advisor 的生命周期。

当前实现的主要风险集中在：
1. **缓存延迟**（`CACHED_MAY_BE_STALE`）；
2. **硬编码模型维护负担**（`@[MODEL LAUNCH]`）；
3. **类型安全债务**（SDK 未发布官方类型）；
4. **First-Party/Foundry 边界模糊**。

这些风险在功能从 Beta 向 GA 演进的过程中需要优先解决。
