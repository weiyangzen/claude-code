# teammateModel.ts 深度研究文档

## 场景与职责

`teammateModel.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**Teammate 模型回退模块**，提供当用户未配置 teammate 默认模型时的硬编码回退逻辑。

### 核心场景

当创建新的 teammate 时，如果：
1. 用户从未在 `/config` 中设置 `teammateDefaultModel`
2. 用户选择了 "Default"（跟随领导）

则需要使用硬编码的默认模型。该模块确保 teammate 能够获得适当的模型配置，同时考虑不同 API 提供商（Bedrock、Vertex、Foundry）的模型 ID 差异。

### 职责边界

- 只提供默认模型回退值
- 不处理模型选择逻辑（由调用方处理）
- 需要随新模型发布更新

---

## 功能点目的

### 1. 默认模型回退

**函数**：`getHardcodedTeammateModelFallback()`

**目的**：返回当前推荐的 teammate 默认模型，根据 API 提供商返回正确的模型 ID。

**当前实现**：
- 使用 `CLAUDE_OPUS_4_6_CONFIG` 配置
- 根据 `getAPIProvider()` 返回对应提供商的模型 ID

---

## 具体技术实现

### 数据结构

#### CLAUDE_OPUS_4_6_CONFIG

```typescript
// 来自 model/configs.js
const CLAUDE_OPUS_4_6_CONFIG = {
  firstParty: 'claude-opus-4-6-20251101',
  bedrock: 'anthropic.claude-opus-4-6-20251101-v1:0',
  vertex: 'claude-opus-4-6-20251101',
  foundry: 'claude-opus-4-6-20251101',
};
```

### 关键流程

```
getHardcodedTeammateModelFallback()
├── getAPIProvider() → 获取当前 API 提供商
│   ├── 'firstParty'  -> 使用 Anthropic API
│   ├── 'bedrock'     -> 使用 AWS Bedrock
│   ├── 'vertex'      -> 使用 Google Vertex
│   └── 'foundry'     -> 使用 Azure Foundry
└── CLAUDE_OPUS_4_6_CONFIG[provider] → 返回对应模型 ID
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `getHardcodedTeammateModelFallback()` | Function | 获取默认模型回退值 |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/components/Settings/Config.tsx` | `getHardcodedTeammateModelFallback` | 设置页面显示默认模型 |
| `src/tools/shared/spawnMultiAgent.ts` | `getHardcodedTeammateModelFallback` | 创建 teammate 时获取默认模型 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/utils/model/configs.js` | `CLAUDE_OPUS_4_6_CONFIG` 模型配置 |
| `src/utils/model/providers.js` | `getAPIProvider` 获取 API 提供商 |

---

## 依赖与外部交互

### 模块依赖图

```
teammateModel.ts
├── model/configs.js         # 模型配置
└── model/providers.js       # API 提供商检测
```

### 与 spawnMultiAgent.ts 的交互

```typescript
// spawnMultiAgent.ts
import { getHardcodedTeammateModelFallback } from '../../utils/swarm/teammateModel.js';

function getDefaultTeammateModel(leaderModel: string | null): string {
  const configured = getGlobalConfig().teammateDefaultModel;
  if (configured === null) {
    // 用户选择了 "Default" — 跟随领导
    return leaderModel ?? getHardcodedTeammateModelFallback();
  }
  if (configured !== undefined) {
    return parseUserSpecifiedModel(configured);
  }
  return getHardcodedTeammateModelFallback();
}
```

### 与 Config.tsx 的交互

```typescript
// Config.tsx
import { getHardcodedTeammateModelFallback } from '../../utils/swarm/teammateModel.js';

// 在设置页面显示当前默认模型
const defaultModelDisplay = getHardcodedTeammateModelFallback();
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 模型发布更新遗漏

**风险**：注释明确要求 "@[MODEL LAUNCH]: Update the fallback model below"，但可能被遗漏。

**当前注释**：
```typescript
// @[MODEL LAUNCH]: Update the fallback model below.
// When the user has never set teammateDefaultModel in /config, new teammates
// use Opus 4.6. Must be provider-aware so Bedrock/Vertex/Foundry customers get
// the correct model ID.
```

**改进建议**：
```typescript
// 添加自动化检查
const CURRENT_MODEL_VERSION = '4.6';
const MODEL_RELEASE_DATE = '2025-11-01';

export function getHardcodedTeammateModelFallback(): string {
  // 在开发/测试环境中检查是否需要更新
  if (process.env.NODE_ENV === 'development') {
    checkModelVersionCurrent();
  }
  
  return CLAUDE_OPUS_4_6_CONFIG[getAPIProvider()];
}

function checkModelVersionCurrent(): void {
  // 检查是否有更新的模型配置可用
  const latestConfig = getLatestModelConfig();
  if (latestConfig.version !== CURRENT_MODEL_VERSION) {
    console.warn(
      `[teammateModel] Model fallback may be outdated. ` +
      `Current: ${CURRENT_MODEL_VERSION}, Latest: ${latestConfig.version}`
    );
  }
}
```

#### 2. 提供商配置缺失

**风险**：如果新增 API 提供商但 `CLAUDE_OPUS_4_6_CONFIG` 未更新，可能导致 `undefined`。

**当前行为**：
```typescript
return CLAUDE_OPUS_4_6_CONFIG[getAPIProvider()];
// 如果提供商不存在，返回 undefined
```

**改进建议**：
```typescript
export function getHardcodedTeammateModelFallback(): string {
  const provider = getAPIProvider();
  const modelId = CLAUDE_OPUS_4_6_CONFIG[provider];
  
  if (!modelId) {
    // 回退到 firstParty
    console.warn(`[teammateModel] Unknown provider: ${provider}, falling back to firstParty`);
    return CLAUDE_OPUS_4_6_CONFIG.firstParty;
  }
  
  return modelId;
}
```

#### 3. 硬编码模型过时

**风险**：随着新模型发布，Opus 4.6 可能不再是最佳选择。

**改进建议**：
```typescript
// 支持基于日期的动态选择
const MODEL_RECOMMENDATIONS: Array<{
  effectiveDate: string;
  config: typeof CLAUDE_OPUS_4_6_CONFIG;
}> = [
  { effectiveDate: '2025-11-01', config: CLAUDE_OPUS_4_6_CONFIG },
  { effectiveDate: '2025-06-01', config: CLAUDE_OPUS_4_5_CONFIG },
  // ...
];

export function getHardcodedTeammateModelFallback(): string {
  const now = new Date();
  const recommendation = MODEL_RECOMMENDATIONS.find(r => 
    new Date(r.effectiveDate) <= now
  );
  
  if (!recommendation) {
    throw new Error('No model recommendation available');
  }
  
  return recommendation.config[getAPIProvider()];
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| 未知 API 提供商 | 返回 `undefined`（当前）或回退到 firstParty（建议） |
| 配置中模型 ID 为空字符串 | 返回空字符串（调用方需要处理） |

### 改进建议

#### 1. 配置化默认模型

```typescript
// 允许通过环境变量覆盖
export function getHardcodedTeammateModelFallback(): string {
  const override = process.env.CLAUDE_CODE_TEAMMATE_MODEL_FALLBACK;
  if (override) {
    return override;
  }
  
  return CLAUDE_OPUS_4_6_CONFIG[getAPIProvider()];
}
```

#### 2. 模型能力声明

```typescript
// 添加模型能力信息
export const TEAMMATE_MODEL_CAPABILITIES = {
  'claude-opus-4-6-20251101': {
    contextWindow: 200000,
    recommendedFor: ['complex-reasoning', 'long-context', 'code-generation'],
    costTier: 'high',
  },
  // ...
};

export function getTeammateModelCapabilities(modelId: string): ModelCapabilities | undefined {
  return TEAMMATE_MODEL_CAPABILITIES[modelId];
}
```

#### 3. 模型选择建议

```typescript
// 基于任务类型推荐模型
export function recommendTeammateModel(taskType: TaskType): string {
  const provider = getAPIProvider();
  
  switch (taskType) {
    case 'research':
      return CLAUDE_OPUS_4_6_CONFIG[provider];  // 需要长上下文
    case 'simple-editing':
      return CLAUDE_SONNET_4_6_CONFIG[provider];  // 更快更便宜
    default:
      return getHardcodedTeammateModelFallback();
  }
}
```

### 测试建议

1. **单元测试**：
   - 验证每个提供商返回正确的模型 ID
   - 验证未知提供商的处理

2. **集成测试**：
   - 验证返回的模型 ID 在 API 调用中有效

3. **发布检查清单**：
   - 新模型发布时更新此文件
   - 验证所有提供商的模型 ID
