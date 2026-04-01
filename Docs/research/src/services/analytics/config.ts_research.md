# config.ts 研究文档

## 场景与职责

`config.ts` 是 Claude Code 分析服务（analytics）的基础配置模块，负责定义整个分析系统的启用/禁用策略。它作为所有分析系统（Datadog、1P 事件日志等）的中央开关，确保在特定环境或用户隐私设置下正确禁用分析功能。

该模块的核心职责包括：
- 提供统一的分析功能启用状态检查
- 区分分析禁用与反馈调查禁用的不同策略
- 整合多种禁用信号（测试环境、第三方云提供商、隐私级别）

## 功能点目的

### 1. `isAnalyticsDisabled()` - 分析功能主开关

**目的**：确定是否应该完全禁用所有分析操作。

**禁用条件**（满足任一即禁用）：
| 条件 | 环境变量 | 说明 |
|------|----------|------|
| 测试环境 | `NODE_ENV === 'test'` | 测试运行时不发送分析数据 |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK` | 第三方云提供商模式 |
| Google Vertex | `CLAUDE_CODE_USE_VERTEX` | 第三方云提供商模式 |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY` | 第三方云提供商模式 |
| 隐私设置 | `isTelemetryDisabled()` | 用户禁用遥测或仅允许必要流量 |

**设计意图**：第三方云提供商（Bedrock/Vertex/Foundry）的数据不应该流向 Anthropic 的分析系统，这是合规性要求。

### 2. `isFeedbackSurveyDisabled()` - 反馈调查开关

**目的**：控制反馈调查（如 compact 后的 NPS 调查）是否显示。

**与 `isAnalyticsDisabled()` 的区别**：
- 反馈调查是本地 UI 提示，不包含会话数据
- 企业客户通过 OTEL 捕获响应，不涉及第三方分析
- 不禁用第三方云提供商的检查（因为数据不离开本地）

**禁用条件**：
- `NODE_ENV === 'test'`
- `isTelemetryDisabled()` 返回 true

## 具体技术实现

### 依赖模块

```typescript
import { isEnvTruthy } from '../../utils/envUtils.js'
import { isTelemetryDisabled } from '../../utils/privacyLevel.js'
```

- `isEnvTruthy`: 安全地检查环境变量是否为真值（处理 '1', 'true', 'yes', 'on'）
- `isTelemetryDisabled`: 检查隐私级别设置（`DISABLE_TELEMETRY` 或 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`）

### 实现逻辑

```typescript
export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test' ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isTelemetryDisabled()
  )
}
```

**性能考虑**：
- 纯同步函数，无 I/O 操作
- 每次调用都实时检查环境变量（无缓存）
- 调用方（如 `datadog.ts`）使用 `memoize` 缓存结果

## 关键代码路径与文件引用

### 被调用方

| 文件 | 调用函数 | 用途 |
|------|----------|------|
| `src/services/analytics/datadog.ts` | `isAnalyticsDisabled()` | Datadog 日志初始化检查 |
| `src/services/analytics/firstPartyEventLogger.ts` | `isAnalyticsDisabled()` | 1P 事件日志启用检查 |
| `src/services/analytics/growthbook.ts` | `is1PEventLoggingEnabled()` → `isAnalyticsDisabled()` | GrowthBook 功能开关 |
| `src/main.tsx` | `isAnalyticsDisabled()` | 启动遥测检查 |
| `src/utils/api.ts` | `isAnalyticsDisabled()` | API 层分析检查 |

### 反馈调查调用方

| 文件 | 调用函数 | 用途 |
|------|----------|------|
| `src/components/FeedbackSurvey/useFeedbackSurvey.tsx` | `isFeedbackSurveyDisabled()` | NPS 调查显示控制 |
| `src/components/FeedbackSurvey/usePostCompactSurvey.tsx` | `isFeedbackSurveyDisabled()` | Compact 后调查控制 |
| `src/components/FeedbackSurvey/useMemorySurvey.tsx` | `isFeedbackSurveyDisabled()` | 内存相关调查控制 |

### 依赖文件

| 文件 | 提供功能 |
|------|----------|
| `src/utils/envUtils.ts` | `isEnvTruthy()` |
| `src/utils/privacyLevel.ts` | `isTelemetryDisabled()` |

## 依赖与外部交互

### 隐私级别系统 (`src/utils/privacyLevel.ts`)

```typescript
type PrivacyLevel = 'default' | 'no-telemetry' | 'essential-traffic'
```

- `default`: 所有功能启用
- `no-telemetry` (`DISABLE_TELEMETRY`): 禁用分析/遥测/反馈调查
- `essential-traffic` (`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`): 禁用所有非必要网络流量

### 环境变量

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `NODE_ENV` | string | 运行环境 |
| `CLAUDE_CODE_USE_BEDROCK` | boolean | AWS Bedrock 模式 |
| `CLAUDE_CODE_USE_VERTEX` | boolean | Google Vertex AI 模式 |
| `CLAUDE_CODE_USE_FOUNDRY` | boolean | Azure Foundry 模式 |
| `DISABLE_TELEMETRY` | boolean | 禁用遥测 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | boolean | 仅允许必要流量 |

## 风险、边界与改进建议

### 风险点

1. **环境变量竞争条件**
   - 问题：如果环境变量在运行时动态修改，不同模块可能看到不同状态
   - 缓解：`datadog.ts` 和 `growthbook.ts` 使用 `memoize` 缓存结果

2. **第三方提供商扩展**
   - 问题：新增云提供商时需要修改此文件
   - 建议：考虑使用前缀匹配（如 `CLAUDE_CODE_USE_*`）自动识别

3. **测试环境误报**
   - 问题：某些集成测试可能希望验证分析功能
   - 现状：强制禁用，无法覆盖

### 边界情况

1. **多条件同时满足**：函数使用短路或逻辑，任一条件满足即返回 true
2. **环境变量未定义**：`isEnvTruthy` 安全处理 undefined 值
3. **大小写敏感**：`isEnvTruthy` 将值转为小写后比较

### 改进建议

1. **集中配置管理**
   ```typescript
   // 建议：将提供商列表提取为配置
   const THIRD_PARTY_PROVIDERS = [
     'CLAUDE_CODE_USE_BEDROCK',
     'CLAUDE_CODE_USE_VERTEX',
     'CLAUDE_CODE_USE_FOUNDRY',
   ] as const
   ```

2. **调试支持**
   - 添加 `getAnalyticsDisabledReason()` 函数返回具体禁用原因
   - 有助于排查用户报告的分析数据缺失问题

3. **文档化扩展点**
   - 在添加新的第三方提供商时，需要同步更新此文件
   - 建议建立检查清单确保不遗漏

4. **测试覆盖**
   - 当前缺乏针对此模块的单元测试
   - 建议添加测试验证各种环境变量组合的行为
