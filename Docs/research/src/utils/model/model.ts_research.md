# model.ts 深度研究文档

## 场景与职责

本模块是 Claude Code **模型系统的核心控制器**，负责模型选择、解析、显示和默认策略的全生命周期管理。它是用户与底层模型能力之间的抽象层，处理从简单的别名解析到复杂的订阅者分级默认策略的所有逻辑。

**核心职责：**
- 模型别名解析（`opus` → `claude-opus-4-6`）
- 用户指定模型设置的优先级处理
- 基于订阅类型的智能默认模型选择
- 1M 上下文窗口的合并策略（Opus 1M Merge）
- 模型名称的规范化、显示和营销命名
- 内部模型（Ant-only）的解析和显示

## 功能点目的

### 1. 模型选择优先级

用户可以通过多种方式指定模型，模块按以下优先级解析：
1. 会话期间覆盖（`/model` 命令）- 最高优先级
2. 启动时覆盖（`--model` 标志）
3. `ANTHROPIC_MODEL` 环境变量
4. 用户保存的设置
5. 内置默认 - 基于订阅类型

### 2. 订阅者分级默认策略

| 用户类型 | 默认模型 |
|---------|---------|
| Ant（内部） | `defaultModel` flag 配置 或 Opus 1M |
| Max 订阅者 | Opus 4.6 + 1M（如果启用） |
| Team Premium | Opus 4.6 + 1M（如果启用） |
| Pro/Team Standard/Enterprise/PAYG | Sonnet 4.6 |

### 3. 1M 上下文合并策略

`isOpus1mMergeEnabled()` 控制是否将 Opus 1M 作为默认选项展示：
- 1M 全局禁用时关闭
- Pro 订阅者关闭（他们默认使用 Sonnet）
- 3P 提供商关闭
- 订阅类型未知时保守关闭（防止 API 拒绝）

### 4. 模型别名系统

支持的别名：
- `opus` → 当前默认 Opus（4.6）
- `sonnet` → 当前默认 Sonnet（4.6 或 4.5）
- `haiku` → 当前默认 Haiku（4.5）
- `best` → 最佳可用模型（当前为 Opus）
- `opusplan` → Plan 模式使用 Opus，其他使用 Sonnet
- `[1m]` 后缀支持：任意别名可添加 `[1m]` 启用 1M 上下文

### 5. 遗留模型重映射

Opus 4.0 和 4.1 已从 firstParty API 移除，模块自动将其重映射到当前默认 Opus：
- `claude-opus-4-20250514` → `claude-opus-4-6`
- `claude-opus-4-1-20250805` → `claude-opus-4-6`

## 具体技术实现

### 关键数据结构

```typescript
// 行 32-34
export type ModelShortName = string
export type ModelName = string
export type ModelSetting = ModelName | ModelAlias | null
```

### 用户指定模型获取

```typescript
// 行 61-78
export function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  let specifiedModel: ModelSetting | undefined

  const modelOverride = getMainLoopModelOverride()  // 来自 state.ts
  if (modelOverride !== undefined) {
    specifiedModel = modelOverride
  } else {
    const settings = getSettings_DEPRECATED() || {}
    specifiedModel = process.env.ANTHROPIC_MODEL || settings.model || undefined
  }

  // 检查 allowlist
  if (specifiedModel && !isModelAllowed(specifiedModel)) {
    return undefined
  }

  return specifiedModel
}
```

### 默认模型选择

```typescript
// 行 178-200
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ant 用户
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Max 用户
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // Team Premium
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // 其他用户默认 Sonnet
  return getDefaultSonnetModel()
}
```

### 模型别名解析

```typescript
// 行 445-506
export function parseUserSpecifiedModel(
  modelInput: ModelName | ModelAlias,
): ModelName {
  const modelInputTrimmed = modelInput.trim()
  const normalizedModel = modelInputTrimmed.toLowerCase()

  // 提取 [1m] 后缀
  const has1mTag = has1mContext(normalizedModel)
  const modelString = has1mTag
    ? normalizedModel.replace(/\[1m]$/i, '').trim()
    : normalizedModel

  // 解析别名
  if (isModelAlias(modelString)) {
    switch (modelString) {
      case 'opusplan':
        return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
      case 'sonnet':
        return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
      case 'haiku':
        return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
      case 'opus':
        return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
      case 'best':
        return getBestModel()
    }
  }

  // 遗留模型重映射
  if (
    getAPIProvider() === 'firstParty' &&
    isLegacyOpusFirstParty(modelString) &&
    isLegacyModelRemapEnabled()
  ) {
    return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
  }

  // Ant-only 内部模型
  if (process.env.USER_TYPE === 'ant') {
    const antModel = resolveAntModel(baseAntModel)
    if (antModel) {
      return antModel.model + suffix
    }
  }

  // 保留原始大小写（用于自定义模型如 Azure Foundry）
  return has1mTag
    ? modelInputTrimmed.replace(/\[1m\]$/i, '').trim() + '[1m]'
    : modelInputTrimmed
}
```

### 规范化名称映射

```typescript
// 行 217-270
export function firstPartyNameToCanonical(name: ModelName): ModelShortName {
  name = name.toLowerCase()
  // Claude 4+ 模型：按版本特异性排序（4-5 在 4 之前）
  if (name.includes('claude-opus-4-6')) return 'claude-opus-4-6'
  if (name.includes('claude-opus-4-5')) return 'claude-opus-4-5'
  if (name.includes('claude-opus-4-1')) return 'claude-opus-4-1'
  if (name.includes('claude-opus-4')) return 'claude-opus-4'
  if (name.includes('claude-sonnet-4-6')) return 'claude-sonnet-4-6'
  if (name.includes('claude-sonnet-4-5')) return 'claude-sonnet-4-5'
  if (name.includes('claude-sonnet-4')) return 'claude-sonnet-4'
  // Claude 3.x 模型
  if (name.includes('claude-3-7-sonnet')) return 'claude-3-7-sonnet'
  if (name.includes('claude-3-5-sonnet')) return 'claude-3-5-sonnet'
  // ... 更多模式
}
```

### Skill 模型覆盖解析

```typescript
// 行 523-536
export function resolveSkillModelOverride(
  skillModel: string,
  currentModel: string,
): string {
  // 如果 skill 已指定 [1m] 或当前模型没有 [1m]，直接返回
  if (has1mContext(skillModel) || !has1mContext(currentModel)) {
    return skillModel
  }
  // 如果目标模型支持 1M，继承 [1m] 后缀
  if (modelSupports1M(parseUserSpecifiedModel(skillModel))) {
    return skillModel + '[1m]'
  }
  return skillModel
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 依赖内容 |
|---------|---------|
| `src/bootstrap/state.ts` | `getMainLoopModelOverride()` - 会话模型覆盖 |
| `src/utils/auth.ts` | `isMaxSubscriber()`, `isTeamPremiumSubscriber()`, `isClaudeAISubscriber()` - 订阅类型检测 |
| `src/utils/context.ts` | `has1mContext()`, `is1mContextDisabled()`, `modelSupports1M()` - 1M 上下文处理 |
| `src/utils/model/modelStrings.ts` | `getModelStrings()` - 获取模型字符串映射 |
| `src/utils/model/modelAllowlist.ts` | `isModelAllowed()` - 模型 allowlist 检查 |
| `src/utils/model/aliases.ts` | `isModelAlias()` - 别名类型检查 |
| `src/utils/model/providers.ts` | `getAPIProvider()` - 提供商检测 |
| `src/utils/settings/settings.ts` | `getSettings_DEPRECATED()` - 用户设置 |
| `src/utils/modelCost.ts` | `formatModelPricing()`, `getOpus46CostTier()` - 成本计算 |

### 被调用方

| 文件路径 | 使用场景 |
|---------|---------|
| `src/utils/model/modelOptions.ts` | 构建模型选择器选项 |
| `src/utils/model/validateModel.ts` | 验证用户输入的模型 |
| `src/utils/model/contextWindowUpgradeCheck.ts` | 获取当前模型设置 |
| `src/utils/context.ts` | 获取规范化名称 |
| `src/services/api/client.ts` | 解析最终 API 调用模型 |
| UI 组件 | 显示模型名称、描述 |

## 依赖与外部交互

### 环境变量

| 变量 | 用途 |
|-----|------|
| `ANTHROPIC_MODEL` | 模型选择环境变量 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | 覆盖默认 Opus 模型 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | 覆盖默认 Sonnet 模型 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 覆盖默认 Haiku 模型 |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | 禁用遗留模型重映射 |
| `USER_TYPE=ant` | 启用内部模型支持 |

### 订阅类型检测

依赖 `auth.ts` 中的函数检测用户订阅类型：
- `isMaxSubscriber()`: Max 计划订阅者
- `isTeamPremiumSubscriber()`: Team Premium 订阅者
- `isProSubscriber()`: Pro 计划订阅者
- `isClaudeAISubscriber()`: 任何 Claude.ai 订阅者

## 风险、边界与改进建议

### 风险点

1. **优先级复杂性**: 模型选择涉及 5 层优先级 + allowlist + 订阅类型，容易出错

2. **3P 提供商滞后**: 3P 提供商（Bedrock/Vertex/Foundry）模型可用性通常滞后 firstParty，代码中需要特殊处理

3. **Ant-only 代码路径**: 内部模型解析代码在生产环境不会执行，但增加了复杂性和维护负担

4. **大小写敏感问题**: 自定义模型 ID（如 Azure Foundry）需要保留大小写，但别名解析需要小写，容易混淆

5. **遗留模型重映射副作用**: 自动重映射可能让用户困惑（"我明明指定了 4.1，为什么显示 4.6？"）

### 边界情况

| 场景 | 行为 |
|-----|------|
| 用户指定 `opus[1m]` 但无权限 | 解析成功，但 API 调用会失败 |
| 3P 用户指定最新模型 | 可能不可用，需回退到旧版本 |
| 订阅类型检测失败 | `isOpus1mMergeEnabled()` 保守返回 false |
| 空模型设置 | 返回 `undefined`，触发默认模型选择 |
| 模型不在 allowlist | 返回 `undefined`，忽略用户设置 |
| Skill 指定 `opus` 而用户在使用 `opus[1m]` | 通过 `resolveSkillModelOverride` 继承 [1m] |

### 改进建议

1. **配置驱动**: 将默认模型策略从代码移到配置，支持更灵活的 A/B 测试

2. **模型可用性预检**: 在解析模型后、API 调用前，检查该模型在当前提供商是否可用

3. **用户意图追踪**: 记录用户原始输入（如 `opus` 别名）而非仅解析后的模型 ID，便于调试

4. **3P 提供商自动检测**: 自动检测 3P 提供商支持的模型版本，动态调整默认模型

5. **模型推荐系统**: 基于用户历史使用模式，推荐最适合的模型

6. **代码简化**: 考虑将 Ant-only 代码分离到独立模块，减少主路径复杂性

```typescript
// 建议：模型可用性预检
export async function validateModelAvailability(
  model: ModelName,
): Promise<{ available: boolean; suggestedAlternative?: ModelName }> {
  const provider = getAPIProvider()
  // 检查模型是否在该提供商可用
  // 返回建议的替代模型（如 4.6 不可用则建议 4.5）
}
```
