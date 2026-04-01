# src/utils/betas.ts 深入研究

## 场景与职责

`betas.ts` 是 Claude Code 的 **Beta Header 决策中心**。它根据当前模型、API 提供商（firstParty / Bedrock / Vertex / Foundry）、用户类型（ant / external）、订阅状态、GrowthBook 实验配置以及环境变量，动态决定向 Anthropic API 发送哪些 beta headers。

这些 beta headers 控制后端能力开关，例如：
- 交错思考（interleaved thinking）
- 结构化输出（structured outputs）
- Web Search、Tool Search
- Context Management、Prompt Caching Scope
- Token-efficient tools、Connector text summarization 等

## 功能点目的

| 功能 | 目的 |
|------|------|
| `getAllModelBetas(model)` | 为指定模型生成完整的 beta header 列表（含所有实验性特性） |
| `getModelBetas(model)` | 在 `getAllModelBetas` 基础上过滤掉 Bedrock 不支持的 headers（Bedrock 需通过 `extraBodyParams` 传递） |
| `getBedrockExtraBodyParamsBetas(model)` | 提取需放入 Bedrock `extraBodyParams` 的 beta headers |
| `getMergedBetas(model, options)` | 合并自动检测的模型 betas 与用户通过 SDK 传入的 `sdkBetas` |
| `getToolSearchBetaHeader()` | 根据提供商返回正确的 tool search beta（1P vs 3P 版本不同） |
| `modelSupportsISP(model)` | 判断模型是否支持 interleaved thinking |
| `modelSupportsStructuredOutputs(model)` | 判断模型是否支持结构化输出 |
| `modelSupportsAutoMode(model)` | 结合 GrowthBook 配置判断模型是否支持 auto mode |
| `modelSupportsContextManagement(model)` | 判断模型是否支持 context management |
| `shouldIncludeFirstPartyOnlyBetas()` | 实验性 beta 是否仅限 firstParty / Foundry |
| `shouldUseGlobalCacheScope()` | 全局 prompt caching scope 是否启用 |
| `filterAllowedSdkBetas(sdkBetas)` | 过滤用户传入的 SDK betas，仅允许白名单内的 header |
| `clearBetasCaches()` | 登出或配置变更时清除 lodash memoize 缓存 |

## 具体技术实现

### 缓存策略
- `getAllModelBetas`、`getModelBetas`、`getBedrockExtraBodyParamsBetas` 均使用 `lodash-es/memoize` 缓存。
- 缓存键为模型名字符串，生命周期为进程级；配置变更后需手动调用 `clearBetasCaches()` 刷新。

### Beta 组装逻辑（`getAllModelBetas`）
1. **基础 beta**：非 Haiku 模型默认带上 `claude-code-20250219`；ant CLI 用户额外带上 `cli-internal`。
2. **订阅相关**：Claude.ai 订阅者带上 `oauth`。
3. **上下文长度**：`has1mContext(model)` 为真时带上 `context-1m`。
4. **交错思考**：`modelSupportsISP(model)` 且未禁用 `DISABLE_INTERLEAVED_THINKING` 时带上。
5. **Redact thinking**：firstParty-only、支持 ISP、非非交互式会话、且 `showThinkingSummaries !== true` 时带上。
6. **Connector text summarization**：ant-only、firstParty-only、GrowthBook `tengu_slate_prism` 或环境变量强制开启。
7. **Context management**：ant 的 `USE_API_CONTEXT_MANAGEMENT` 或模型支持 thinking preservation 时带上。
8. **Structured outputs**：firstParty-only、模型支持、`tengu_tool_pear` gate 开启时带上。
9. **Token-efficient tools**：ant-only、firstParty-only、`tengu_amber_json_tools` 开启且 strict tools 未开启时带上（与 strict 互斥）。
10. **Web search**：Vertex 的 Claude 4+ 或 Foundry 时带上。
11. **Prompt caching scope**：firstParty-only 时带上。
12. **环境变量覆盖**：`ANTHROPIC_BETAS` 逗号分割后追加。

### 模型支持判断
- `modelSupportsISP`：通过 `getCanonicalName(model)` + `getAPIProvider()` 判断；Foundry 全支持，firstParty 排除 `claude-3-`，第三方仅 `claude-opus-4` / `claude-sonnet-4`。
- `modelSupportsStructuredOutputs`：仅 firstParty / Foundry，且模型 ID 匹配 `claude-sonnet-4-6`、`-4-5`、`claude-opus-4-1`、`-4-5`、`-4-6`、`claude-haiku-4-5`。
- `modelSupportsAutoMode`：依赖 `feature('TRANSCRIPT_CLASSIFIER')`；external 非 firstParty 直接拒绝；ant 使用 denylist（`claude-3-`、旧版 `-4`），external 使用 allowlist（`^claude-(opus|sonnet)-4-6`）。GrowthBook `tengu_auto_mode_config.allowModels` 可强制覆盖。

### SDK Beta 过滤
- 白名单 `ALLOWED_SDK_BETAS` 当前仅包含 `CONTEXT_1M_BETA_HEADER`。
- 若用户是 Claude.ai 订阅者，直接忽略所有 SDK betas 并打印警告（订阅者不支持自定义 betas）。

## 关键代码路径与文件引用

```
src/services/api/claude.ts
  └── getMergedBetas, getToolSearchBetaHeader, shouldIncludeFirstPartyOnlyBetas, shouldUseGlobalCacheScope

src/utils/api.ts
  └── modelSupportsStructuredOutputs, shouldUseGlobalCacheScope, shouldIncludeFirstPartyOnlyBetas

src/utils/toolSearch.ts
  └── getMergedBetas

src/utils/sideQuery.ts
  └── getModelBetas, modelSupportsStructuredOutputs

src/services/tokenEstimation.ts
  └── getModelBetas

src/services/claudeAiLimits.ts
  └── getModelBetas

src/services/analytics/metadata.ts
  └── getModelBetas

src/utils/advisor.ts
  └── shouldIncludeFirstPartyOnlyBetas

src/utils/permissions/permissionSetup.ts
  └── modelSupportsAutoMode

src/constants/prompts.ts
  └── shouldUseGlobalCacheScope

src/main.tsx
  └── filterAllowedSdkBetas

src/cli/print.ts
  └── modelSupportsAutoMode

src/commands/logout/logout.tsx
  └── clearBetasCaches
```

### 依赖模块
- `src/constants/betas.ts` — beta header 字符串常量定义
- `src/constants/oauth.ts` — `OAUTH_BETA_HEADER`
- `src/services/analytics/growthbook.ts` — `checkStatsigFeatureGate_CACHED_MAY_BE_STALE`, `getFeatureValue_CACHED_MAY_BE_STALE`
- `src/bootstrap/state.ts` — `getIsNonInteractiveSession`, `getSdkBetas`
- `src/utils/auth.ts` — `isClaudeAISubscriber`
- `src/utils/context.ts` — `has1mContext`
- `src/utils/envUtils.ts` — `isEnvDefinedFalsy`, `isEnvTruthy`
- `src/utils/model/model.ts` — `getCanonicalName`
- `src/utils/model/modelSupportOverrides.ts` — `get3PModelCapabilityOverride`
- `src/utils/model/providers.ts` — `getAPIProvider`
- `src/utils/settings/settings.ts` — `getInitialSettings`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| Anthropic API | HTTP headers (`anthropic-beta`) | 最终通过 API 请求携带 beta headers 激活后端能力 |
| GrowthBook / Statsig | `getFeatureValue_CACHED_MAY_BE_STALE` | 动态实验开关（auto mode、structured outputs、connector text summarization 等） |
| 环境变量 | `process.env.*` | `DISABLE_INTERLEAVED_THINKING`、`USE_CONNECTOR_TEXT_SUMMARIZATION`、`ANTHROPIC_BETAS`、`USER_TYPE` 等 |
| 用户设置 | `getInitialSettings()` | `showThinkingSummaries` 等 |

## 风险、边界与改进建议

### 风险
1. **模型 ID 硬编码脆弱性**：`modelSupportsStructuredOutputs`、`modelSupportsISP` 等大量使用字符串 `includes` / 正则匹配模型 ID。新模型发布时必须手动更新列表，极易遗漏。
2. **`@[MODEL LAUNCH]` 注释依赖人工**：代码中虽有 `@[MODEL LAUNCH]` 标记提醒，但无编译期或 CI 检查强制更新。
3. **memoize 缓存键单一**：仅按模型名缓存，未纳入 GrowthBook 配置版本或环境变量变化，理论上配置变更后若未调用 `clearBetasCaches` 会返回 stale headers。
4. **Bedrock / Vertex beta 兼容性复杂**：同一 beta 在不同提供商下的传递方式（header vs extraBodyParams）分散在 `betas.ts`、`api.ts`、`claude.ts` 多处，容易不一致。

### 边界
- `getAllModelBetas` 对 Haiku 模型排除大量 beta（如 `claude-code-20250219`），因为 Haiku 主要用于非 agentic 调用（compaction、classifier）。
- `filterAllowedSdkBetas` 白名单极窄，仅允许 `context-1m`，其余用户自定义 betas 会被丢弃并打印警告。
- ant 与 external 的 auto mode 支持策略完全相反（denylist vs allowlist），external 的 firstParty 限制在 `modelSupportsAutoMode` 中优先于 GrowthBook 覆盖。

### 改进建议
1. **模型能力元数据化**：将模型能力表提取为声明式 JSON / 常量对象（如 `{ 'claude-sonnet-4-6': { structuredOutputs: true, isp: true } }`），减少字符串硬编码。
2. **CI 检查 `@[MODEL LAUNCH]`**：在发布流水线中扫描新增模型 ID 是否已在 `betas.ts` 中注册对应能力。
3. **缓存键增强**：将 `memoize` 替换为支持多依赖的缓存（如结合 `getFeatureValue` 的刷新 token），或降低缓存粒度，在 `getMergedBetas` 层再做合并。
4. **统一提供商适配层**：将 "哪些 beta 走 header、哪些走 extraBodyParams" 的决策收敛到 `betas.ts` 的单一函数，供 `api.ts` 与 `claude.ts` 统一调用。
5. **auto mode 配置文档化**：denylist/allowlist 逻辑较复杂，建议补充内部文档说明 external 与 ant 的差异及安全考量。
