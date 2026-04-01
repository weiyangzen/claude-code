# src/utils/context.ts 深度研究文档

## 1. 场景与职责

`context.ts` 是 Claude Code CLI 的上下文窗口管理核心模块，负责计算和管理与 Anthropic API 交互时的上下文窗口大小、模型输出令牌限制等关键参数。它直接影响 API 调用的性能和成本。

### 主要职责
- **上下文窗口大小计算**: 根据模型类型和实验性功能确定上下文窗口大小
- **1M 上下文支持**: 处理 Sonnet-4 系列模型的 1M 令牌上下文窗口
- **输出令牌限制**: 计算模型的默认和最大输出令牌数
- **环境覆盖**: 支持通过环境变量覆盖默认行为
- **HIPAA 合规**: 支持通过环境变量禁用 1M 上下文（C4E 管理员需求）

## 2. 功能点目的

### 2.1 上下文窗口大小计算

```typescript
export function getContextWindowForModel(
  model: string,
  betas?: string[],
): number
```

**决策优先级**（从高到低）：
1. **环境变量覆盖**（仅内部用户）: `CLAUDE_CODE_MAX_CONTEXT_TOKENS`
2. **显式 [1m] 后缀**: 模型名称包含 `[1m]` 标记
3. **模型能力检测**: 从 `getModelCapability()` 获取
4. **Beta 头**: `betas` 数组包含 `CONTEXT_1M_BETA_HEADER`
5. **实验性处理**: `clientDataCache?.['coral_reef_sonnet'] === 'true'`
6. **内部模型解析**: 仅对 `USER_TYPE === 'ant'` 生效
7. **默认值**: `MODEL_CONTEXT_WINDOW_DEFAULT = 200_000`

### 2.2 1M 上下文控制

```typescript
export function is1mContextDisabled(): boolean
export function has1mContext(model: string): boolean
export function modelSupports1M(model: string): boolean
export function getSonnet1mExpTreatmentEnabled(model: string): boolean
```

**HIPAA 合规开关**:
```typescript
// C4E 管理员可通过环境变量禁用 1M 上下文
return isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT)
```

**支持的模型模式**:
```typescript
// @[MODEL LAUNCH]: Update this pattern if the new model supports 1M context
export function modelSupports1M(model: string): boolean {
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4') || canonical.includes('opus-4-6')
}
```

### 2.3 输出令牌限制

```typescript
export function getModelMaxOutputTokens(model: string): {
  default: number
  upperLimit: number
}
```

**模型映射表**:

| 模型模式 | 默认值 | 上限 |
|----------|--------|------|
| `opus-4-6` | 64,000 | 128,000 |
| `sonnet-4-6` | 32,000 | 128,000 |
| `opus-4-5`, `sonnet-4`, `haiku-4` | 32,000 | 64,000 |
| `opus-4-1`, `opus-4` | 32,000 | 32,000 |
| `claude-3-opus` | 4,096 | 4,096 |
| `claude-3-sonnet` | 8,192 | 8,192 |
| `claude-3-haiku` | 4,096 | 4,096 |
| `3-5-sonnet`, `3-5-haiku` | 8,192 | 8,192 |
| `3-7-sonnet` | 32,000 | 64,000 |
| 默认 | 32,000 | 64,000 |

### 2.4 令牌上限优化

```typescript
// Capped default for slot-reservation optimization
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000
export const ESCALATED_MAX_TOKENS = 64_000
```

**背景**: BQ p99 输出约 4,911 令牌，32k/64k 默认值会过度预留 8-16 倍容量。启用上限后，<1% 请求会达到限制，这些请求会干净地重试到 64k。

### 2.5 上下文使用率计算

```typescript
export function calculateContextPercentages(
  currentUsage: {
    input_tokens: number
    cache_creation_input_tokens: number
    cache_read_input_tokens: number
  } | null,
  contextWindowSize: number,
): { used: number | null; remaining: number | null }
```

**计算逻辑**:
```
totalInputTokens = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
usedPercentage = round((totalInputTokens / contextWindowSize) * 100)
clampedUsed = min(100, max(0, usedPercentage))
remaining = 100 - clampedUsed
```

## 3. 具体技术实现

### 3.1 核心常量

```typescript
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000
export const COMPACT_MAX_OUTPUT_TOKENS = 20_000
const MAX_OUTPUT_TOKENS_DEFAULT = 32_000
const MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000
export const ESCALATED_MAX_TOKENS = 64_000
```

### 3.2 模型名称规范化

```typescript
import { getCanonicalName } from './model/model.js'

// 规范化模型名称以进行模式匹配
const canonical = getCanonicalName(model) // e.g., "claude-sonnet-4-6-20251022"
```

### 3.3 内部用户特殊处理

```typescript
if (process.env.USER_TYPE === 'ant') {
  const antModel = resolveAntModel(model.toLowerCase())
  if (antModel?.contextWindow) {
    return antModel.contextWindow
  }
}
```

内部构建支持通过 `resolveAntModel` 解析额外的模型配置。

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 用途 | 调用方 |
|------|------|--------|
| `getContextWindowForModel()` | 获取模型上下文窗口 | `analyzeContext.ts`, `query.ts`, `claude.ts` |
| `getModelMaxOutputTokens()` | 获取输出令牌限制 | `claude.ts`, `query.ts` |
| `calculateContextPercentages()` | 计算使用率百分比 | `stats.tsx`, `StatusLine.tsx` |
| `has1mContext()` | 检查 1M 标记 | 模型选择逻辑 |
| `modelSupports1M()` | 检查模型支持 | 功能标志评估 |
| `is1mContextDisabled()` | 检查禁用状态 | 所有 1M 相关函数 |
| `CAPPED_DEFAULT_MAX_TOKENS` | 令牌上限常量 | `claude.ts` |

### 4.2 调用链

**API 调用准备**:
```
query.ts / QueryEngine.ts
  └── getContextWindowForModel()
        ├── has1mContext() // 检查 [1m] 后缀
        ├── modelSupports1M() // 检查模型类型
        └── getModelCapability() // 获取模型能力
```

**状态栏显示**:
```
StatusLine.tsx / stats.tsx
  └── calculateContextPercentages()
        └── 显示已用/剩余百分比
```

### 4.3 配置文件

```typescript
// src/utils/analyzeContext.ts
import { getContextWindowForModel } from './context.js'

// src/services/api/claude.ts
import { getContextWindowForModel } from '../../utils/context.js'

// src/query.ts
import { getContextWindowForModel } from './utils/context.js'
```

## 5. 依赖与外部交互

### 5.1 直接依赖

```typescript
import { CONTEXT_1M_BETA_HEADER } from '../constants/betas.js'
import { getGlobalConfig } from './config.js'
import { isEnvTruthy } from './envUtils.js'
import { getCanonicalName } from './model/model.js'
import { getModelCapability } from './model/modelCapabilities.js'
```

### 5.2 条件依赖

```typescript
// 仅内部构建
if (process.env.USER_TYPE === 'ant') {
  const antModel = resolveAntModel(model.toLowerCase())
  // resolveAntModel 从 ./model/model.js 导入，但只在 ant 构建中使用
}
```

### 5.3 被依赖情况

主要调用方：
- `src/utils/analyzeContext.ts` - 上下文分析
- `src/services/api/claude.ts` - API 调用
- `src/query.ts` - 查询处理
- `src/QueryEngine.ts` - 查询引擎
- `src/components/StatusLine.tsx` - 状态栏
- `src/context/stats.tsx` - 统计信息

## 6. 风险、边界与改进建议

### 6.1 风险分析

| 风险 | 可能性 | 影响 | 说明 |
|------|--------|------|------|
| 模型名称模式过时 | 中 | 高 | 新模型发布时需要更新 `modelSupports1M` |
| 环境变量滥用 | 低 | 中 | `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 可能被误用 |
| 令牌计算不准确 | 中 | 中 | 令牌计数是估算值，可能与实际 API 计数有偏差 |
| HIPAA 合规绕过 | 低 | 高 | 用户可能通过修改代码绕过 `DISABLE_1M_CONTEXT` |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 未知模型 | 返回默认上下文窗口 200k |
| 无效的环境变量值 | 忽略，使用默认值 |
| 模型名称为空 | 返回默认上下文窗口 |
| 1M 被禁用 | 即使模型支持也返回 200k |
| 同时满足多个条件 | 按优先级顺序，第一个匹配生效 |

### 6.3 改进建议

#### 短期
1. **模型配置外部化**: 将模型参数移到配置文件，避免代码修改：
   ```typescript
   // models.config.json
   {
     "claude-sonnet-4-6": {
       "contextWindow": 1000000,
       "defaultOutputTokens": 32000,
       "maxOutputTokens": 128000
     }
   }
   ```

2. **增强日志**: 记录上下文窗口决策过程，便于调试：
   ```typescript
   logForDebugging(`Context window for ${model}: ${window} (reason: ${reason})`)
   ```

#### 中期
3. **动态模型发现**: 从 API 端点获取模型能力，而非硬编码：
   ```typescript
   const capabilities = await fetchModelCapabilities(model)
   return capabilities.maxInputTokens
   ```

4. **A/B 测试支持**: 更细粒度的上下文窗口实验支持：
   ```typescript
   export function getContextWindowForModel(
     model: string,
     betas?: string[],
     experimentContext?: ExperimentContext, // 新增
   ): number
   ```

#### 长期
5. **自适应上下文**: 根据对话历史动态调整上下文窗口：
   ```typescript
   export function getAdaptiveContextWindow(
     model: string,
     conversationHistory: Message[],
   ): number
   ```

6. **成本感知优化**: 在上下文窗口和成本之间进行智能权衡：
   ```typescript
   export function getCostOptimizedContextWindow(
     model: string,
     budgetConstraints: BudgetConstraints,
   ): number
   ```

### 6.4 维护指南

**添加新模型支持时**:
1. 更新 `modelSupports1M()` 中的模式匹配
2. 在 `getModelMaxOutputTokens()` 中添加输出令牌映射
3. 更新本研究文档的模型映射表
4. 运行集成测试验证上下文窗口计算

**修改 1M 上下文逻辑时**:
1. 确保 HIPAA 合规性检查仍然有效
2. 更新所有相关的实验性处理逻辑
3. 通知 C4E 管理员任何行为变更
