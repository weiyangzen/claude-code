# Research: src/utils/model/providers.ts

## 场景与职责

本模块是 Claude Code 中判断“当前使用哪家 API provider”的权威来源。所有与 provider 相关的分支逻辑（模型 ID 格式、认证方式、beta 头、定价显示、能力判断等）都依赖此处返回的 `APIProvider` 值。

## 功能点目的

### `getAPIProvider(): APIProvider`

按以下优先级读取环境变量，返回 `'firstParty' | 'bedrock' | 'vertex' | 'foundry'`：
1. `CLAUDE_CODE_USE_BEDROCK` → `'bedrock'`
2. `CLAUDE_CODE_USE_VERTEX` → `'vertex'`
3. `CLAUDE_CODE_USE_FOUNDRY` → `'foundry'`
4. 否则 → `'firstParty'`

所有判断均通过 `isEnvTruthy()` 进行，因此支持 `1`、`true`、`yes` 等真值写法。

### `getAPIProviderForStatsig()`

简单包装 `getAPIProvider()`，将返回值断言为 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型，用于 Statsig/GrowthBook 等分析系统的元数据上报。函数名中的显式声明是为了在代码审查时强调该值仅作为分析标签，不会流入文件路径或代码逻辑。

### `isFirstPartyAnthropicBaseUrl(): boolean`

判断当前 `ANTHROPIC_BASE_URL` 是否指向 Anthropic 官方 API：
- 未设置 → `true`（默认走官方 API）。
- 已设置 → 解析 URL 的 host：
  - 普通用户只允许 `api.anthropic.com`。
  - Ant 用户额外允许 `api-staging.anthropic.com`。
- 解析失败 → `false`（视为自定义代理/网关）。

该函数被 `modelCapabilities.ts` 用于决定是否有资格调用官方 `models.list` 端点。

## 关键代码路径与文件引用

- **入口**: `src/utils/model/providers.ts`
- **被调用方**（极为广泛，仅列核心）:
  - `src/utils/model/model.ts` — 默认模型选择、显示名、legacy remap。
  - `src/utils/model/modelOptions.ts` — 选项定价和 label 构建。
  - `src/utils/model/modelStrings.ts` — 模型 ID 映射。
  - `src/utils/model/deprecation.ts` — 退役日期按 provider 选取。
  - `src/utils/model/validateModel.ts` — 3P fallback 建议。
  - `src/utils/model/modelCapabilities.ts` — 是否允许拉取能力缓存。
  - `src/utils/model/modelSupportOverrides.ts` — 3P 能力覆盖。
  - `src/utils/auth.ts` — 认证策略分支。
  - `src/utils/fastMode.ts` — fast mode 可用性。
  - `src/utils/effort.ts`、`src/utils/thinking.ts`、`src/utils/betas.ts` — 模型能力判断。
  - `src/services/api/client.ts`、`src/services/api/bootstrap.ts`、`src/services/api/claude.ts` — API 客户端初始化。
  - `src/services/policyLimits/index.ts` — 策略限制按 provider 区分。
  - `src/services/settingsSync/index.ts` — 设置同步逻辑。
  - `src/services/remoteManagedSettings/syncCache.ts` — 远程配置同步。
  - `src/services/analytics/datadog.ts` — 分析上报。
  - `src/commands.ts` — 命令可用性。
  - `src/constants/system.ts` — 系统提示前缀。
- **依赖**:
  - `src/utils/envUtils.ts` (`isEnvTruthy`)
  - `src/services/analytics/index.ts` (`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`)

## 依赖与外部交互

- 仅读取环境变量：`CLAUDE_CODE_USE_BEDROCK`、`CLAUDE_CODE_USE_VERTEX`、`CLAUDE_CODE_USE_FOUNDRY`、`ANTHROPIC_BASE_URL`、`USER_TYPE`。
- 无网络请求、无磁盘 IO、无缓存、无全局状态。
- 是纯函数，调用成本极低。

## 风险、边界与改进建议

- **风险**: 由于被大量模块引用，`getAPIProvider()` 的返回值变化会触发全系统行为变更。当前实现基于环境变量，意味着 provider 在进程启动后固定不变；不支持 mid-session 切换 provider（这本身是产品设计，但需确保无代码尝试动态修改相关环境变量）。
- **边界**: `isFirstPartyAnthropicBaseUrl` 对 Ant 用户放宽到 `api-staging.anthropic.com`，若未来增加其他内部环境（如 dev、load-test），需要同步修改 allowedHosts 列表。
- **边界**: 环境变量优先级固定为 Bedrock > Vertex > Foundry > firstParty。若用户同时设置了多个 flag，结果可能不符合直觉；但通常用户只会设置一个。
- **改进建议**: 当前 `APIProvider` 类型是 4 个字符串字面量的联合类型，被硬编码在数十个文件的 switch/case 中。若未来新增 provider（如某云厂商），需要大面积修改代码。建议将 provider-specific 的逻辑进一步抽象为插件/策略表，减少分支扩散。
- **改进建议**: `isFirstPartyAnthropicBaseUrl` 的 host 白名单可考虑从远程配置读取，避免每次新增内部域名都要发版。
