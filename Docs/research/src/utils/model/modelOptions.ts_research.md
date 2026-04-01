# modelOptions.ts 深度研究文档

## 场景与职责

本模块负责**构建和管理模型选择器（model picker）的选项列表**。根据用户类型、订阅级别、API 提供商和可用权限，动态生成不同的模型选项列表，为用户提供清晰、相关的模型选择体验。

**核心职责：**
- 为不同用户类型生成定制化的模型选项
- 处理 1M 上下文选项的显示逻辑
- 支持自定义模型选项（通过环境变量）
- 实现模型 allowlist 过滤
- 检测新版本可用性并提供升级提示

## 功能点目的

### 1. 分层用户模型选项

不同用户看到不同的模型选项：

| 用户类型 | 默认模型 | 可选模型 |
|---------|---------|---------|
| Ant（内部） | 配置或 Opus 1M | 内部模型 + 公开模型 |
| Max/Team Premium | Opus 4.6 (1M) | Opus 1M, Sonnet, Sonnet 1M, Haiku |
| Pro/Team Standard | Sonnet 4.6 | Sonnet 1M, Opus, Opus 1M, Haiku |
| PAYG 1P | Sonnet 4.6 | Sonnet 1M, Opus, Opus 1M, Haiku |
| PAYG 3P | Sonnet 4.5 | 取决于 3P 提供商支持的模型 |

### 2. 1M 上下文选项控制

- `isOpus1mMergeEnabled`: 控制 Opus 1M 是否合并到默认选项
- `checkOpus1mAccess()` / `checkSonnet1mAccess()`: 控制是否显示 1M 选项

### 3. 自定义模型支持

通过环境变量支持自定义模型选项：
- `ANTHROPIC_CUSTOM_MODEL_OPTION`: 自定义模型 ID
- `ANTHROPIC_DEFAULT_*_MODEL`: 覆盖默认模型

### 4. 新版本检测

检测用户是否在使用旧版本模型，提示升级到当前默认版本。

## 具体技术实现

### 选项数据结构

```typescript
// 行 38-43
export type ModelOption = {
  value: ModelSetting           // 模型值（别名或完整 ID）
  label: string                 // 显示标签
  description: string           // 描述文本
  descriptionForModel?: string  // 用于模型选择的详细描述
}
```

### 用户分层选项生成

```typescript
// 行 271-376
function getModelOptionsBase(fastMode = false): ModelOption[] {
  // Ant 用户
  if (process.env.USER_TYPE === 'ant') {
    const antModelOptions = getAntModels().map(m => ({
      value: m.alias,
      label: m.label,
      description: m.description ?? `[ANT-ONLY] ${m.label} (${m.model})`,
    }))
    return [
      getDefaultOptionForUser(),
      ...antModelOptions,
      getMergedOpus1MOption(fastMode),
      getSonnet46Option(),
      getSonnet46_1MOption(),
      getHaiku45Option(),
    ]
  }

  // Max/Team Premium 订阅者
  if (isMaxSubscriber() || isTeamPremiumSubscriber()) {
    const premiumOptions = [getDefaultOptionForUser(fastMode)]
    if (!isOpus1mMergeEnabled() && checkOpus1mAccess()) {
      premiumOptions.push(getMaxOpus46_1MOption(fastMode))
    }
    premiumOptions.push(MaxSonnet46Option)
    if (checkSonnet1mAccess()) {
      premiumOptions.push(getMaxSonnet46_1MOption())
    }
    premiumOptions.push(MaxHaiku45Option)
    return premiumOptions
  }

  // Pro/Team Standard/Enterprise 订阅者
  if (isClaudeAISubscriber()) {
    const standardOptions = [getDefaultOptionForUser(fastMode)]
    if (checkSonnet1mAccess()) {
      standardOptions.push(getMaxSonnet46_1MOption())
    }
    if (isOpus1mMergeEnabled()) {
      standardOptions.push(getMergedOpus1MOption(fastMode))
    } else {
      standardOptions.push(getMaxOpusOption(fastMode))
      if (checkOpus1mAccess()) {
        standardOptions.push(getMaxOpus46_1MOption(fastMode))
      }
    }
    standardOptions.push(MaxHaiku45Option)
    return standardOptions
  }

  // PAYG 1P 和 3P 逻辑...
}
```

### 默认选项生成

```typescript
// 行 45-74
export function getDefaultOptionForUser(fastMode = false): ModelOption {
  // Ant 用户
  if (process.env.USER_TYPE === 'ant') {
    const currentModel = renderDefaultModelSetting(getDefaultMainLoopModelSetting())
    return {
      value: null,
      label: 'Default (recommended)',
      description: `Use the default model for Ants (currently ${currentModel})`,
    }
  }

  // 订阅者
  if (isClaudeAISubscriber()) {
    return {
      value: null,
      label: 'Default (recommended)',
      description: getClaudeAiUserDefaultModelDescription(fastMode),
    }
  }

  // PAYG
  const is3P = getAPIProvider() !== 'firstParty'
  return {
    value: null,
    label: 'Default (recommended)',
    description: `Use the default model (currently ${renderDefaultModelSetting(getDefaultMainLoopModelSetting())})`,
  }
}
```

### 新版本检测

```typescript
// 行 385-424
function getModelFamilyInfo(model: string): { alias: string; currentVersionName: string } | null {
  const canonical = getCanonicalName(model)

  // Sonnet 家族
  if (canonical.includes('claude-sonnet-4-6') || 
      canonical.includes('claude-sonnet-4-5') ||
      canonical.includes('claude-3-7-sonnet') ||
      canonical.includes('claude-3-5-sonnet')) {
    const currentName = getMarketingNameForModel(getDefaultSonnetModel())
    if (currentName) {
      return { alias: 'Sonnet', currentVersionName: currentName }
    }
  }

  // Opus 家族
  if (canonical.includes('claude-opus-4')) {
    const currentName = getMarketingNameForModel(getDefaultOpusModel())
    if (currentName) {
      return { alias: 'Opus', currentVersionName: currentName }
    }
  }

  // Haiku 家族...
}
```

### Allowlist 过滤

```typescript
// 行 531-540
function filterModelOptionsByAllowlist(options: ModelOption[]): ModelOption[] {
  const settings = getSettings_DEPRECATED() || {}
  if (!settings.availableModels) {
    return options  // 无限制
  }
  return options.filter(
    opt => opt.value === null || (opt.value !== null && isModelAllowed(opt.value)),
  )
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 依赖内容 |
|---------|---------|
| `src/utils/auth.ts` | `isMaxSubscriber()`, `isTeamPremiumSubscriber()`, `isClaudeAISubscriber()` |
| `src/utils/model/model.ts` | `getDefault*Model()`, `getUserSpecifiedModelSetting()`, `isOpus1mMergeEnabled()` |
| `src/utils/model/check1mAccess.ts` | `checkOpus1mAccess()`, `checkSonnet1mAccess()` |
| `src/utils/model/modelStrings.ts` | `getModelStrings()` |
| `src/utils/model/providers.ts` | `getAPIProvider()` |
| `src/utils/model/modelAllowlist.ts` | `isModelAllowed()` |
| `src/utils/modelCost.ts` | `formatModelPricing()`, `COST_TIER_*` |
| `src/utils/context.ts` | `has1mContext()` |
| `src/utils/config.ts` | `getGlobalConfig()` |
| `src/bootstrap/state.ts` | `getInitialMainLoopModel()` |

### 被调用方

| 文件路径 | 使用场景 |
|---------|---------|
| UI 组件（模型选择器） | 显示可用模型列表 |
| `/model` 命令处理 | 处理模型切换 |
| 设置页面 | 显示当前模型选项 |

## 依赖与外部交互

### 环境变量

| 变量 | 用途 |
|-----|------|
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | 添加自定义模型选项 |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | 自定义模型显示名称 |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | 自定义模型描述 |
| `ANTHROPIC_DEFAULT_*_MODEL` | 覆盖默认模型 |
| `ANTHROPIC_DEFAULT_*_MODEL_NAME` | 自定义默认模型名称 |
| `ANTHROPIC_DEFAULT_*_MODEL_DESCRIPTION` | 自定义默认模型描述 |

### 选项列表构建流程

```
getModelOptions(fastMode)
    ↓
getModelOptionsBase(fastMode)
    ├── 确定用户类型
    ├── 构建基础选项列表
    ├── 根据权限添加 1M 选项
    └── 添加自定义选项
    ↓
添加 ANTHROPIC_CUSTOM_MODEL_OPTION
    ↓
添加 additionalModelOptionsCache
    ↓
添加当前使用的自定义模型
    ↓
filterModelOptionsByAllowlist()
    ↓
返回最终选项列表
```

## 风险、边界与改进建议

### 风险点

1. **复杂度过高**: 用户分层逻辑复杂，涉及多个条件判断，容易出错

2. **硬编码模型版本**: 选项描述中硬编码模型版本号（如 "Sonnet 4.6"），新模型发布时需要多处修改

3. **3P 提供商差异**: 3P 提供商支持的模型版本差异大，当前逻辑可能无法覆盖所有情况

4. **自定义模型安全风险**: 环境变量注入的自定义模型未经验证，可能存在安全风险

### 边界情况

| 场景 | 行为 |
|-----|------|
| 用户无 1M 权限 | 不显示 1M 选项 |
| allowlist 为空 | 仅显示 "Default" 选项 |
| 当前模型不在选项中 | 作为自定义选项添加到列表 |
| 3P 提供商无 Opus 4.6 | 显示 Opus 4.1 作为替代 |
| 用户在使用 `opusplan` | 添加 Opus Plan 选项到列表 |

### 改进建议

1. **配置驱动**: 将选项列表配置从代码移到 JSON/YAML，支持动态调整

2. **模型版本自动检测**: 从 `configs.ts` 自动获取当前模型版本，避免硬编码

3. **A/B 测试支持**: 添加选项排序和展示的 A/B 测试能力

4. **3P 提供商自动探测**: 自动检测 3P 提供商支持的模型版本，动态调整选项

5. **选项分组**: 将选项按类别分组（如 "推荐"、"高性能"、"经济型"）

6. **使用统计驱动**: 根据用户使用历史，智能排序选项

```typescript
// 建议：配置驱动选项
type ModelOptionConfig = {
  id: string
  value: string
  label: string
  description: string
  requirements: {
    minSubscription?: 'pro' | 'max' | 'team_premium'
    providers?: APIProvider[]
    features?: ('1m_context' | 'fast_mode')[]
  }
}

const MODEL_OPTIONS_CONFIG: ModelOptionConfig[] = [
  {
    id: 'sonnet-46',
    value: 'sonnet',
    label: 'Sonnet',
    description: 'Sonnet 4.6 · Best for everyday tasks',
    requirements: {},
  },
  // ...
]
```
