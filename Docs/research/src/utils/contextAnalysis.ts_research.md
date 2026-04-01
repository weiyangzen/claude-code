# src/utils/contextAnalysis.ts 深度研究文档

## 1. 场景与职责

`contextAnalysis.ts` 是 Claude Code CLI 的上下文分析工具模块，提供对对话历史中令牌使用情况的详细统计和分析。它帮助理解 API 调用中的令牌分布，识别潜在的优化机会（如重复文件读取）。

### 主要职责
- **令牌统计**: 按类别统计对话中的令牌使用情况
- **工具调用分析**: 追踪工具请求和结果的令牌消耗
- **重复读取检测**: 识别重复读取同一文件的浪费
- **附件统计**: 按类型统计附件数量
- **遥测指标生成**: 将统计数据转换为 Statsig 指标格式

## 2. 功能点目的

### 2.1 令牌统计分类

```typescript
type TokenStats = {
  toolRequests: Map<string, number>      // 各工具请求的令牌数
  toolResults: Map<string, number>       // 各工具结果的令牌数
  humanMessages: number                  // 用户消息的令牌数
  assistantMessages: number              // 助手消息的令牌数
  localCommandOutputs: number            // 本地命令输出的令牌数
  other: number                          // 其他内容的令牌数
  attachments: Map<string, number>       // 各类型附件的数量
  duplicateFileReads: Map<string, { count: number; tokens: number }>
  total: number                          // 总令牌数
}
```

### 2.2 重复文件读取检测

**检测逻辑**:
1. 追踪 `Read` 工具的 `tool_use_id` 到文件路径的映射
2. 统计每个文件路径的读取次数和总令牌数
3. 计算重复读取的浪费令牌数

```typescript
// 计算重复读取的浪费
fileReadStats.forEach((data, path) => {
  if (data.count > 1) {
    const averageTokensPerRead = Math.floor(data.totalTokens / data.count)
    const duplicateTokens = averageTokensPerRead * (data.count - 1)
    stats.duplicateFileReads.set(path, { count: data.count, tokens: duplicateTokens })
  }
})
```

### 2.3 本地命令输出识别

通过内容特征识别本地命令输出：
```typescript
if (message.type === 'user' && content.includes('local-command-stdout')) {
  stats.localCommandOutputs += tokens
}
```

### 2.4 Statsig 遥测指标

将统计数据转换为遥测格式：
```typescript
export function tokenStatsToStatsigMetrics(stats: TokenStats): Record<string, number>
```

**生成的指标**:
- `total_tokens`: 总令牌数
- `human_message_tokens`: 用户消息令牌
- `assistant_message_tokens`: 助手消息令牌
- `local_command_output_tokens`: 本地命令输出令牌
- `tool_request_{ToolName}_tokens`: 各工具请求令牌
- `tool_result_{ToolName}_tokens`: 各工具结果令牌
- `duplicate_read_tokens`: 重复读取浪费的令牌
- `duplicate_read_file_count`: 重复读取的文件数
- 各类百分比指标

## 3. 具体技术实现

### 3.1 核心算法

```typescript
export function analyzeContext(messages: Message[]): TokenStats {
  // 1. 初始化统计对象
  const stats: TokenStats = { ... }
  
  // 2. 构建工具 ID 到工具名称的映射
  const toolIdsToToolNames = new Map<string, string>()
  const readToolIdToFilePath = new Map<string, string>()
  const fileReadStats = new Map<string, { count: number; totalTokens: number }>()
  
  // 3. 统计附件
  messages.forEach(msg => { ... })
  
  // 4. 规范化消息并处理每个内容块
  const normalizedMessages = normalizeMessagesForAPI(messages)
  normalizedMessages.forEach(msg => {
    content.forEach(block => processBlock(...))
  })
  
  // 5. 计算重复读取
  fileReadStats.forEach((data, path) => { ... })
  
  return stats
}
```

### 3.2 内容块处理

```typescript
function processBlock(
  block: ContentBlockParam | ContentBlock | BetaContentBlock,
  message: UserMessage | AssistantMessage,
  stats: TokenStats,
  toolIds: Map<string, string>,
  readToolPaths: Map<string, string>,
  fileReads: Map<string, { count: number; totalTokens: number }>,
): void {
  const tokens = countTokens(jsonStringify(block))
  stats.total += tokens
  
  switch (block.type) {
    case 'text':
      // 检查是否为本地命令输出
      // 累加到 humanMessages 或 assistantMessages
      break
    case 'tool_use':
      // 记录工具名称和 ID 映射
      // 如果是 Read 工具，记录文件路径
      break
    case 'tool_result':
      // 查找对应的工具名称
      // 如果是 Read 工具结果，更新文件读取统计
      break
    // 其他类型归入 'other'
  }
}
```

### 3.3 令牌计数

使用粗略估计（非精确 API 计数）：
```typescript
import { roughTokenCountEstimation as countTokens } from '../services/tokenEstimation.js'

const tokens = countTokens(jsonStringify(block))
```

**注意**: 这是客户端估计值，可能与实际 API 计数有差异。

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 用途 | 调用方 |
|------|------|--------|
| `analyzeContext(messages)` | 分析上下文统计 | `doctorContextWarnings.ts`, `toolSearch.ts` |
| `tokenStatsToStatsigMetrics(stats)` | 转换为遥测指标 | 遥测系统 |

### 4.2 调用链

**Doctor 命令**:
```
src/utils/doctorContextWarnings.ts
  └── analyzeContext()
        └── 生成上下文警告建议
```

**工具搜索**:
```
src/utils/toolSearch.ts
  └── analyzeContext()
        └── 评估是否需要工具搜索优化
```

**遥测上报**:
```
遥测系统
  └── tokenStatsToStatsigMetrics()
        └── 发送到 Statsig
```

## 5. 依赖与外部交互

### 5.1 直接依赖

```typescript
import type { BetaContentBlock } from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'
import type { ContentBlock, ContentBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'
import { roughTokenCountEstimation as countTokens } from '../services/tokenEstimation.js'
import type { AssistantMessage, Message, UserMessage } from '../types/message.js'
import { normalizeMessagesForAPI } from './messages.js'
import { jsonStringify } from './slowOperations.js'
```

### 5.2 被依赖情况

```
contextAnalysis.ts
    ↑
    ├── src/utils/doctorContextWarnings.ts
    ├── src/utils/toolSearch.ts
    ├── src/components/ContextVisualization.tsx
    ├── src/commands/context/context.tsx
    └── src/commands/context/context-noninteractive.ts
```

### 5.3 与 analyzeContext.ts 的关系

**注意**: 存在命名相似的另一个文件 `src/utils/analyzeContext.ts`（注意大小写和路径差异）：
- `contextAnalysis.ts`: 本文件，提供 `analyzeContext()` 函数，返回 `TokenStats`
- `analyzeContext.ts`: 提供 `analyzeContextUsage()` 函数，返回 `ContextData`

两者功能互补：
- `contextAnalysis.ts`: 轻量级统计，用于遥测和简单分析
- `analyzeContext.ts`: 重量级分析，用于上下文可视化

## 6. 风险、边界与改进建议

### 6.1 风险分析

| 风险 | 可能性 | 影响 | 说明 |
|------|--------|------|------|
| 令牌计数不准确 | 中 | 中 | 使用粗略估计，非 API 精确计数 |
| 大消息处理性能 | 中 | 中 | 需要遍历所有消息和内容块 |
| JSON 序列化失败 | 低 | 低 | 循环引用可能导致 `jsonStringify` 失败 |
| 工具名称映射错误 | 低 | 中 | tool_use_id 找不到对应工具名称 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 空消息数组 | 返回全零统计 |
| 字符串内容（旧格式） | 特殊处理，检查 `local-command-stdout` |
| 缺失 tool_use_id | 归类为 'unknown' 工具 |
| 重复文件读取（1次） | 不记录（count > 1 才记录） |
| 未知内容块类型 | 累加到 `other` 类别 |

### 6.3 改进建议

#### 短期
1. **精确令牌计数**: 提供使用 API 的精确计数选项：
   ```typescript
   export async function analyzeContextPrecise(messages: Message[]): Promise<TokenStats>
   ```

2. **增量分析**: 支持增量更新统计，避免全量重新计算：
   ```typescript
   export function analyzeContextIncremental(
     previousStats: TokenStats,
     newMessages: Message[],
   ): TokenStats
   ```

#### 中期
3. **更多统计维度**: 添加时间序列、对话轮次等维度：
   ```typescript
   type TokenStats = {
     // ... 现有字段
     byTurn: Array<{ turn: number; tokens: number }>
     byTimeWindow: Map<string, number>
   }
   ```

4. **可视化集成**: 直接与上下文可视化组件集成，避免重复计算：
   ```typescript
   // 与 analyzeContext.ts 合并或共享计算结果
   ```

#### 长期
5. **智能建议**: 基于统计数据生成具体的优化建议：
   ```typescript
   export function generateOptimizationSuggestions(stats: TokenStats): Suggestion[]
   ```

6. **历史趋势**: 跟踪令牌使用趋势，预测何时达到上下文限制：
   ```typescript
   export function predictContextExhaustion(
     statsHistory: TokenStats[],
     contextWindow: number,
   ): { estimatedTurns: number; confidence: number }
   ```

### 6.4 测试建议

当前无直接测试，建议添加：
- 空消息数组测试
- 单消息统计测试
- 工具调用/结果配对测试
- 重复文件读取检测测试
- 本地命令输出识别测试
- 大消息数组性能测试
