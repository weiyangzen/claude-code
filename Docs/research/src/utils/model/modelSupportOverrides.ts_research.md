# Research: src/utils/model/modelSupportOverrides.ts

## 场景与职责

为第三方（3P）provider（Bedrock、Vertex、Foundry）用户提供一种临时能力：当官方尚未在代码中内置支持某模型的新能力（如 `thinking`、`adaptive_thinking`、`interleaved_thinking`、`effort`、`max_effort`）时，允许通过环境变量为自定义 pinned 模型显式声明其支持的能力集合。

## 功能点目的

- `get3PModelCapabilityOverride(model, capability): boolean | undefined`
  - 若当前 provider 为 `firstParty`，直接返回 `undefined`（不生效）。
  - 否则检查该 `model` 是否匹配三个环境变量 `ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL` 中的某一个。
  - 若匹配，则读取对应的能力环境变量（如 `ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES`），将其按逗号分割后判断是否包含请求的 `capability`。
  - 若未匹配任何 pinned 模型，或对应能力环境变量未定义，返回 `undefined`。

## 具体技术实现

### 能力类型

```ts
export type ModelCapabilityOverride =
  | 'effort'
  | 'max_effort'
  | 'thinking'
  | 'adaptive_thinking'
  | 'interleaved_thinking'
```

### 匹配层级

内部定义了 `TIERS` 数组，将三个默认模型环境变量与其对应的能力环境变量配对：

| modelEnvVar | capabilitiesEnvVar |
|-------------|-------------------|
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `ANTHROPIC_DEFAULT_HAIKU_MODEL_SUPPORTED_CAPABILITIES` |

### Memoization

使用 `lodash-es/memoize` 缓存结果，缓存 key 为 `${model.toLowerCase()}:${capability}`，避免重复解析环境变量字符串。

## 关键代码路径与文件引用

- **入口**: `src/utils/model/modelSupportOverrides.ts`
- **被调用方**:
  - `src/utils/effort.ts` — 判断模型是否支持 effort 相关功能。
  - `src/utils/thinking.ts` — 判断模型是否支持 thinking 相关功能。
  - `src/utils/betas.ts` — 判断模型 beta 能力。
  - `src/utils/advisor.ts` —  Advisor 功能中的模型能力判断。
- **依赖**:
  - `src/utils/model/providers.ts` (`getAPIProvider`)
  - `lodash-es/memoize.js`

## 依赖与外部交互

- 仅读取本地环境变量，无网络请求、无磁盘 IO、无全局状态依赖。
- 完全独立于 settings 系统，专为 CI/高级用户场景设计。

## 风险、边界与改进建议

- **风险**: 该机制仅对通过环境变量 `ANTHROPIC_DEFAULT_*_MODEL` 显式 pinned 的模型生效。若 3P 用户通过 `--model` 或 settings 指定了一个不在上述三个环境变量中的模型，则无法使用此覆盖机制，导致新能力不可用。
- **边界**: 返回 `undefined` 与返回 `false` 语义不同：
  - `undefined` 表示“此覆盖机制未命中，请走默认逻辑”。
  - `false` 表示“机制命中了，但模型不支持该能力”。
  - 调用方需正确处理 `undefined` 的 fallback 行为。
- **边界**: 环境变量中的能力列表大小写不敏感（内部做了 `toLowerCase()`），但逗号前后空格需由 `trim()` 处理；若用户使用了中文逗号或其他分隔符会解析失败。
- **改进建议**: 当前仅支持 3 个 tier（Opus/Sonnet/Haiku），若未来增加新的模型家族（如某种专门的 code 模型），需要同步扩展 `TIERS` 数组。建议将 tier 配置外置或动态化。
- **改进建议**: 可考虑将 `modelSupportOverrides` 与 `modelOverrides`（`settings.json` 中的模型字符串覆盖）打通，让用户在 settings 中一并声明能力和模型 ID，而非依赖分散的环境变量。
