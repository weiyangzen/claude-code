# toolLimits.ts 深度研究文档

## 场景与职责

`src/constants/toolLimits.ts` 是 Claude Code CLI 的工具结果大小限制常量模块，定义了与工具执行结果存储、截断和持久化相关的系统级限制。该模块的核心目标是防止过大的工具结果消耗过多的上下文窗口（context window），同时确保用户和模型能够获取足够的信息。

**主要使用场景：**
1. **工具结果截断决策**：决定何时将工具结果保存到磁盘而非直接返回给模型
2. **上下文预算管理**：限制单轮对话中工具结果的总体大小
3. **UI 显示优化**：限制工具摘要的显示长度

**设计原则：**
- 在用户体验和上下文效率之间取得平衡
- 提供可配置的阈值，支持通过 GrowthBook 动态调整
- 区分单个工具结果限制和每轮消息聚合限制

## 功能点目的

### 1. 单个工具结果限制

**`DEFAULT_MAX_RESULT_SIZE_CHARS` (50,000 字符)**
- 单个工具结果在返回给模型前的最大字符数
- 超过此限制的结果会被持久化到磁盘，模型只收到文件路径预览
- 工具可以声明更低的 `maxResultSizeChars`，但不能超过此系统上限

**`MAX_TOOL_RESULT_TOKENS` (100,000 tokens)**
- 基于 token 数的工具结果上限
- 约等于 400KB 文本（按每 token 4 字节估算）
- 用于更精确的上下文预算计算

**`BYTES_PER_TOKEN` (4)**
- 从字节数估算 token 数的保守系数
- 实际 token 数可能因内容而异

**`MAX_TOOL_RESULT_BYTES`**
- 从 token 限制派生的字节数限制
- 计算：`MAX_TOOL_RESULT_TOKENS * BYTES_PER_TOKEN = 400,000 bytes`

### 2. 每轮消息聚合限制

**`MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` (200,000 字符)**
- 单轮用户消息中所有 `tool_result` 块的聚合大小限制
- 防止 N 个并行工具各自接近上限，集体产生超大消息（如 10 × 40K = 400K）
- 消息独立评估：一轮的 150K 结果不会影响下一轮

**运行时覆盖**：
- 可通过 GrowthBook flag `tengu_hawthorn_window` 调整
- 参见 `toolResultStorage.ts` 中的 `getPerMessageBudgetLimit()`

### 3. UI 显示限制

**`TOOL_SUMMARY_MAX_LENGTH` (50 字符)**
- 紧凑视图中工具摘要字符串的最大长度
- 用于 `getToolUseSummary()` 实现
- 影响分组 Agent 渲染时的显示效果

## 具体技术实现

### 常量定义

```typescript
/**
 * Default maximum size in characters for tool results before they get persisted
 * to disk. When exceeded, the result is saved to a file and the model receives
 * a preview with the file path instead of the full content.
 */
export const DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000

/**
 * Maximum size for tool results in tokens.
 * Based on analysis of tool result sizes, we set this to a reasonable upper bound
 * to prevent excessively large tool results from consuming too much context.
 *
 * This is approximately 400KB of text (assuming ~4 bytes per token).
 */
export const MAX_TOOL_RESULT_TOKENS = 100_000

/**
 * Bytes per token estimate for calculating token count from byte size.
 * This is a conservative estimate - actual token count may vary.
 */
export const BYTES_PER_TOKEN = 4

/**
 * Maximum size for tool results in bytes (derived from token limit).
 */
export const MAX_TOOL_RESULT_BYTES = MAX_TOOL_RESULT_TOKENS * BYTES_PER_TOKEN

/**
 * Default maximum aggregate size in characters for tool_result blocks within
 * a SINGLE user message (one turn's batch of parallel tool results).
 */
export const MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000

/**
 * Maximum character length for tool summary strings in compact views.
 */
export const TOOL_SUMMARY_MAX_LENGTH = 50
```

### 数值设计 rationale

| 常量 | 值 | 设计理由 |
|------|-----|----------|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | 约 12,500 tokens，足够容纳大多数命令输出，同时保留上下文空间 |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 | 约 400KB，是单个工具结果的硬上限 |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 | 允许 4 个并行工具各输出 ~50K，或 2 个各输出 ~100K |
| `BYTES_PER_TOKEN` | 4 | 保守估计（英文平均约 4 字符/token，代码可能更少） |
| `TOOL_SUMMARY_MAX_LENGTH` | 50 | 终端显示友好，避免过长摘要破坏布局 |

## 关键代码路径与文件引用

### 调用方分析

| 调用文件 | 调用内容 | 用途 |
|----------|----------|------|
| `src/utils/toolResultStorage.ts` | 所有限制常量 | 工具结果持久化决策和预算管理 |
| `src/tools/BashTool/BashTool.tsx` | `TOOL_SUMMARY_MAX_LENGTH` | Bash 命令摘要生成 |
| `src/tools/WebSearchTool/UI.tsx` | `TOOL_SUMMARY_MAX_LENGTH` | 网页搜索摘要显示 |
| `src/tools/GlobTool/UI.tsx` | `TOOL_SUMMARY_MAX_LENGTH` | 文件匹配摘要显示 |
| `src/tools/GrepTool/UI.tsx` | `TOOL_SUMMARY_MAX_LENGTH` | 文本搜索摘要显示 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | `TOOL_SUMMARY_MAX_LENGTH` | PowerShell 命令摘要 |
| `src/tools/WebFetchTool/UI.tsx` | `TOOL_SUMMARY_MAX_LENGTH` | 网页获取摘要显示 |

### 核心消费代码示例

```typescript
// src/utils/toolResultStorage.ts
import {
  BYTES_PER_TOKEN,
  DEFAULT_MAX_RESULT_SIZE_CHARS,
  MAX_TOOL_RESULT_BYTES,
  MAX_TOOL_RESULTS_PER_MESSAGE_CHARS,
} from '../constants/toolLimits.js'

/**
 * Resolve the effective persistence threshold for a tool.
 */
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Infinity = hard opt-out. Read self-bounds via maxTokens; persisting its
  // output to a file the model reads back with Read is circular.
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<
    string,
    number
  > | null>(PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (
    typeof override === 'number' &&
    Number.isFinite(override) &&
    override > 0
  ) {
    return override
  }
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

## 依赖与外部交互

### 外部依赖

该模块是纯常量模块，无运行时依赖。

### 动态配置

虽然常量是静态定义的，但实际使用中有动态覆盖机制：

1. **GrowthBook 覆盖** (`tengu_satin_quoll`)
   - 工具级别的持久化阈值覆盖
   - 格式：`{ toolName: thresholdInChars }`

2. **GrowthBook 每消息预算** (`tengu_hawthorn_window`)
   - 覆盖 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS`
   - 用于紧急调整上下文预算

3. **工具声明的 `maxResultSizeChars`**
   - 单个工具可以声明自己的上限
   - 实际阈值 = `min(工具声明, DEFAULT_MAX_RESULT_SIZE_CHARS)`

### 工具结果持久化流程

```
工具执行 → 检查大小
    ↓
超过阈值? → 是 → 保存到磁盘 → 返回文件路径预览
    ↓ 否
直接返回内容给模型
```

## 风险、边界与改进建议

### 潜在风险

1. **阈值设置不当**
   - 过低：频繁触发持久化，影响用户体验（需要额外读取文件）
   - 过高：消耗过多上下文，导致后续对话受限

2. **Token 估算不准确**
   - `BYTES_PER_TOKEN = 4` 是保守估计
   - 代码内容 token 密度可能不同（符号较多时更高）
   - 非英文内容 token 数可能显著不同

3. **并行工具累积**
   - 即使单个工具在限制内，并行执行可能触发聚合限制
   - 用户可能不理解为什么某些结果被持久化而其他没有

4. **GrowthBook 配置错误**
   - 覆盖值类型错误（如 string 而非 number）可能导致意外行为
   - 当前代码有防御性检查，但仍需小心

### 边界情况

1. **Infinity 处理**：某些工具可能声明 `maxResultSizeChars = Infinity` 表示 opt-out
2. **零或负数**：未明确处理，可能导致意外行为
3. **空结果**：零长度内容应正常通过
4. **二进制内容**：字节到字符的转换可能不准确

### 改进建议

1. **自适应阈值**
   ```typescript
   // 根据上下文剩余空间动态调整
   export function getAdaptiveThreshold(
     remainingContextTokens: number,
     baseThreshold: number = DEFAULT_MAX_RESULT_SIZE_CHARS,
   ): number {
     const safetyMargin = 10_000
     return Math.min(baseThreshold, (remainingContextTokens - safetyMargin) * BYTES_PER_TOKEN)
   }
   ```

2. **更精确的 Token 计数**
   - 集成轻量级 tokenizer 进行更准确的估算
   - 或使用 API 的 token counting 端点

3. **用户可见的预算指示器**
   - 显示当前轮次的工具结果预算使用情况
   - 帮助用户理解为什么结果被持久化

4. **工具级别的预算配置**
   ```typescript
   // 允许工具声明预算权重
   interface ToolBudgetConfig {
     maxResultSizeChars: number
     budgetWeight: number // 在聚合预算中的权重
   }
   ```

5. **持久化策略优化**
   - 智能摘要：对于超大结果，生成 AI 摘要而非完整保存
   - 分层存储：热数据内存缓存，冷数据磁盘存储

6. **监控和告警**
   - 跟踪阈值命中的频率
   - 监控 GrowthBook 覆盖的使用情况
   - 识别频繁触发持久化的工具

7. **文档改进**
   - 添加阈值选择的决策树
   - 提供调优指南
   - 解释 token 估算的局限性
