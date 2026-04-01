# sessionMemoryCompact.ts 深度研究文档

## 场景与职责

`sessionMemoryCompact.ts` 实现了 Claude Code 的 **Session Memory 压缩** 功能，这是一种实验性的对话压缩机制，使用预先生成的 session memory（会话笔记）作为摘要，替代传统的模型调用生成摘要的方式。该功能旨在降低压缩成本、提高压缩速度，并利用结构化的会话记忆提供更准确的上下文保留。

**核心场景：**
1. **正常压缩** - 使用 `lastSummarizedMessageId` 确定已总结的消息边界，仅保留之后的消息
2. **恢复会话压缩** - 会话有 session memory 内容但不知道具体边界时，保留所有消息但使用 session memory 作为摘要
3. **自动压缩集成** - 作为 `autoCompactIfNeeded` 的第一选择，失败时回退到传统压缩

**关键设计决策：**
- 优先尝试 session memory 压缩，失败时优雅回退到传统压缩
- 使用 GrowthBook 远程配置动态调整压缩阈值
- 保留消息数量基于 token 计数和文本块消息数双重约束

## 功能点目的

### 1. Session Memory 压缩配置管理

**配置结构**：
```typescript
export type SessionMemoryCompactConfig = {
  minTokens: number          // 压缩后保留的最小 token 数（默认 10,000）
  minTextBlockMessages: number  // 保留的最小文本块消息数（默认 5）
  maxTokens: number          // 压缩后保留的最大 token 数（硬上限，默认 40,000）
}
```

**配置来源**：
- 默认值：`DEFAULT_SM_COMPACT_CONFIG`
- 远程配置：GrowthBook `tengu_sm_compact_config`
- 运行时更新：`setSessionMemoryCompactConfig()`

**初始化逻辑**：
- 仅首次调用时从 GrowthBook 获取配置
- 远程值为正数时才使用，避免零值覆盖合理默认值

### 2. 文本块检测 (hasTextBlocks)

**目的**：判断消息是否包含用户/助手交互的文本内容

**实现细节**：
- 助手消息：检查 `content` 数组中是否有 `type === 'text'` 的块
- 用户消息：支持字符串内容和数组内容
- 其他类型消息返回 false

### 3. 工具结果 ID 提取 (getToolResultIds)

**目的**：从用户消息中提取所有 `tool_result` 块的 `tool_use_id`

**用途**：用于后续的工具使用配对检查

### 4. 工具使用配对检查 (hasToolUseWithIds)

**目的**：检查助手消息是否包含指定 ID 集合中的任何 `tool_use` 块

**用途**：在调整保留消息索引时，确保保留包含对应 `tool_use` 的助手消息

### 5. 索引调整以保留 API 不变式 (adjustIndexToPreserveAPIInvariants)

**核心问题**：压缩时如果切割位置不当，可能导致：
- `tool_use` 和 `tool_result` 配对断裂（API 错误）
- `thinking` 块丢失（同一 `message.id` 的多条消息被分割）

**两步调整算法**：

**步骤1：处理工具配对**
1. 收集保留范围内所有消息的 `tool_result` ID
2. 收集保留范围内已有的 `tool_use` ID
3. 找出缺失的 `tool_use` ID（需要的外部工具调用）
4. 向前查找包含这些工具调用的助手消息，扩展保留范围

**步骤2：处理 thinking 块**
1. 收集保留范围内所有助手消息的 `message.id`
2. 向前查找具有相同 `message.id` 的助手消息（可能包含 thinking 块）
3. 扩展保留范围以包含这些消息

**边界条件**：
- `startIndex <= 0` 或 `startIndex >= messages.length`：直接返回
- 向前查找直到索引 0 或所有需要的工具都找到

### 6. 保留消息索引计算 (calculateMessagesToKeepIndex)

**算法流程**：
1. 从 `lastSummarizedIndex + 1` 开始作为初始索引
2. 计算当前保留范围的 token 数和文本块消息数
3. 检查是否已满足最小要求或超过最大限制
4. 如不满足，向后扩展（降低索引）直到满足条件或到达边界
5. 边界限制：不能低于最近的 compact boundary 消息
6. 最后调用 `adjustIndexToPreserveAPIInvariants` 调整索引

**边界处理**：
- 使用 `findLastIndex` 找到最近的 compact boundary 作为下限
- 防止跨越磁盘不连续点导致的消息链断裂

### 7. Session Memory 压缩启用检查 (shouldUseSessionMemoryCompaction)

**启用条件**：
1. 环境变量 `ENABLE_CLAUDE_CODE_SM_COMPACT` 为 truthy（覆盖启用）
2. 或同时满足：
   - 环境变量 `DISABLE_CLAUDE_CODE_SM_COMPACT` 不为 truthy
   - GrowthBook `tengu_session_memory` 为 true
   - GrowthBook `tengu_sm_compact` 为 true

**调试日志**：
- 仅 `USER_TYPE === 'ant'` 时记录标志状态
- 事件：`tengu_sm_compact_flag_check`

### 8. 从 Session Memory 创建压缩结果 (createCompactionResultFromSessionMemory)

**构建组件**：
1. **边界标记** (`boundaryMarker`)：
   - 类型：`'auto'`
   - 记录压缩前 token 数
   - 记录最后消息的 UUID
   - 记录压缩前发现的工具

2. **摘要内容处理**：
   - 调用 `truncateSessionMemoryForCompact` 截断过长的 session memory
   - 如果发生截断，添加指向完整 session memory 文件的链接

3. **摘要消息** (`summaryMessages`)：
   - 使用 `getCompactUserSummaryMessage` 格式化
   - 标记为 `isCompactSummary: true`
   - 标记为 `isVisibleInTranscriptOnly: true`

4. **附件** (`attachments`)：
   - 调用 `createPlanAttachmentIfNeeded` 添加计划附件

5. **Token 计数**：
   - `postCompactTokenCount` 和 `truePostCompactTokenCount` 相同（无压缩 API 调用）
   - 使用 `estimateMessageTokens` 估算

### 9. 主入口：尝试 Session Memory 压缩 (trySessionMemoryCompaction)

**执行流程**：

```
1. 检查是否应该使用 session memory 压缩
   └─ 否 → 返回 null

2. 初始化配置（从 GrowthBook，仅一次）

3. 等待进行中的 session memory 提取完成（15秒超时）

4. 获取 lastSummarizedMessageId 和 session memory 内容

5. 检查 session memory 是否存在且非空
   └─ 不存在或为模板 → 记录事件，返回 null

6. 确定 lastSummarizedIndex
   ├─ 有 lastSummarizedMessageId → 查找对应消息索引
   │   └─ 未找到 → 记录事件，返回 null（回退到传统压缩）
   └─ 无 ID（恢复会话）→ 设为 messages.length - 1

7. 计算保留消息的起始索引
   └─ 调用 calculateMessagesToKeepIndex

8. 过滤消息（移除旧边界消息）

9. 执行 session start hooks

10. 获取转录路径

11. 创建压缩结果
    └─ 调用 createCompactionResultFromSessionMemory

12. 构建压缩后消息并估算 token 数

13. 检查是否超过自动压缩阈值
    └─ 超过 → 记录事件，返回 null

14. 返回压缩结果
```

**错误处理**：
- 所有错误被捕获并记录为 `tengu_sm_compact_error` 事件
- 返回 null 允许调用方回退到传统压缩

## 具体技术实现

### 关键数据结构

```typescript
export type SessionMemoryCompactConfig = {
  minTokens: number
  minTextBlockMessages: number
  maxTokens: number
}

export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,
  minTextBlockMessages: 5,
  maxTokens: 40_000,
}
```

### 状态管理

```typescript
// 当前配置（可运行时更新）
let smCompactConfig: SessionMemoryCompactConfig = { ...DEFAULT_SM_COMPACT_CONFIG }

// 配置是否已从远程初始化
let configInitialized = false

// 导出函数用于测试
export function setSessionMemoryCompactConfig(config: Partial<SessionMemoryCompactConfig>): void
export function getSessionMemoryCompactConfig(): SessionMemoryCompactConfig
export function resetSessionMemoryCompactConfig(): void
```

### 远程配置获取

```typescript
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // 仅使用显式设置的正数值
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens > 0 ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    minTextBlockMessages: remoteConfig.minTextBlockMessages > 0 ? remoteConfig.minTextBlockMessages : DEFAULT_SM_COMPACT_CONFIG.minTextBlockMessages,
    maxTokens: remoteConfig.maxTokens > 0 ? remoteConfig.maxTokens : DEFAULT_SM_COMPACT_CONFIG.maxTokens,
  }
  setSessionMemoryCompactConfig(config)
}
```

### 工具配对调整算法

```typescript
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number,
): number {
  if (startIndex <= 0 || startIndex >= messages.length) return startIndex

  let adjustedIndex = startIndex

  // Step 1: Handle tool_use/tool_result pairs
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }

  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    for (let i = adjustedIndex; i < messages.length; i++) {
      // 收集保留范围内的 tool_use ID
    }

    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)),
    )

    // 向前查找包含所需 tool_use 的消息
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // 从 neededToolUseIds 中移除已找到的 ID
      }
    }
  }

  // Step 2: Handle thinking blocks with same message.id
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    const msg = messages[i]!
    if (msg.type === 'assistant' && msg.message.id) {
      messageIdsInKeptRange.add(msg.message.id)
    }
  }

  // 向前查找具有相同 message.id 的消息
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    const message = messages[i]!
    if (message.type === 'assistant' &&
        message.message.id &&
        messageIdsInKeptRange.has(message.message.id)) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

## 关键代码路径与文件引用

### 调用方 (Callers)

| 文件路径 | 调用函数 | 场景 |
|---------|---------|------|
| `src/services/compact/autoCompact.ts:288` | `trySessionMemoryCompaction` | 自动压缩时优先尝试 |
| `src/commands/compact/compact.ts:58` | `trySessionMemoryCompaction` | 手动 `/compact` 命令 |

### 被调用方 (Callees)

| 函数 | 来源文件 | 用途 |
|-----|---------|------|
| `getDynamicConfig_BLOCKS_ON_INIT` | `growthbook.ts` | 获取远程配置 |
| `getFeatureValue_CACHED_MAY_BE_STALE` | `growthbook.ts` | 获取特性标志 |
| `logEvent` | `analytics/index.ts` | 记录分析事件 |
| `isEnvTruthy` | `envUtils.ts` | 检查环境变量 |
| `getLastSummarizedMessageId` | `sessionMemoryUtils.ts` | 获取上次总结的消息 ID |
| `setLastSummarizedMessageId` | `sessionMemoryUtils.ts` | 重置总结消息 ID |
| `getSessionMemoryContent` | `sessionMemoryUtils.ts` | 获取 session memory 内容 |
| `waitForSessionMemoryExtraction` | `sessionMemoryUtils.ts` | 等待提取完成 |
| `isSessionMemoryEmpty` | `SessionMemory/prompts.ts` | 检查是否为空模板 |
| `truncateSessionMemoryForCompact` | `SessionMemory/prompts.ts` | 截断过长的内容 |
| `getTranscriptPath` | `sessionStorage.ts` | 获取转录路径 |
| `getSessionMemoryPath` | `permissions/filesystem.ts` | 获取 session memory 路径 |
| `processSessionStartHooks` | `sessionStart.ts` | 执行 session start hooks |
| `getMainLoopModel` | `model/model.ts` | 获取主循环模型 |
| `tokenCountFromLastAPIResponse` | `tokens.ts` | 获取 token 计数 |
| `estimateMessageTokens` | `microCompact.ts` | 估算消息 token |
| `createCompactBoundaryMessage` | `messages.ts` | 创建边界标记 |
| `createUserMessage` | `messages.ts` | 创建用户消息 |
| `isCompactBoundaryMessage` | `messages.ts` | 检查是否为边界消息 |
| `getCompactUserSummaryMessage` | `prompt.ts` | 格式化用户摘要消息 |
| `createPlanAttachmentIfNeeded` | `compact.ts` | 创建计划附件 |
| `annotateBoundaryWithPreservedSegment` | `compact.ts` | 标注保留段 |
| `buildPostCompactMessages` | `compact.ts` | 构建压缩后消息 |
| `extractDiscoveredToolNames` | `toolSearch.ts` | 提取发现的工具名 |

### 导出函数

| 函数 | 可见性 | 用途 |
|-----|-------|------|
| `SessionMemoryCompactConfig` | type | 配置类型定义 |
| `DEFAULT_SM_COMPACT_CONFIG` | const | 默认配置 |
| `setSessionMemoryCompactConfig` | export | 设置配置（测试用） |
| `getSessionMemoryCompactConfig` | export | 获取配置（测试用） |
| `resetSessionMemoryCompactConfig` | export | 重置配置（测试用） |
| `hasTextBlocks` | export | 检测文本块（测试用） |
| `adjustIndexToPreserveAPIInvariants` | export | 调整索引（测试用） |
| `calculateMessagesToKeepIndex` | export | 计算保留索引（测试用） |
| `shouldUseSessionMemoryCompaction` | export | 检查是否启用 |
| `trySessionMemoryCompaction` | export | 主入口函数 |

## 依赖与外部交互

### 导入依赖

```typescript
import type { AgentId } from '../../types/ids.js'
import type { HookResultMessage, Message } from '../../types/message.js'
import { logForDebugging } from '../../utils/debug.js'
import { isEnvTruthy } from '../../utils/envUtils.js'
import { errorMessage } from '../../utils/errors.js'
import { createCompactBoundaryMessage, createUserMessage, isCompactBoundaryMessage } from '../../utils/messages.js'
import { getMainLoopModel } from '../../utils/model/model.js'
import { getSessionMemoryPath } from '../../utils/permissions/filesystem.js'
import { processSessionStartHooks } from '../../utils/sessionStart.js'
import { getTranscriptPath } from '../../utils/sessionStorage.js'
import { tokenCountFromLastAPIResponse } from '../../utils/tokens.js'
import { extractDiscoveredToolNames } from '../../utils/toolSearch.js'
import { getDynamicConfig_BLOCKS_ON_INIT, getFeatureValue_CACHED_MAY_BE_STALE } from '../analytics/growthbook.js'
import { logEvent } from '../analytics/index.js'
import { isSessionMemoryEmpty, truncateSessionMemoryForCompact } from '../SessionMemory/prompts.js'
import { getLastSummarizedMessageId, getSessionMemoryContent, waitForSessionMemoryExtraction } from '../SessionMemory/sessionMemoryUtils.js'
import { annotateBoundaryWithPreservedSegment, buildPostCompactMessages, type CompactionResult, createPlanAttachmentIfNeeded } from './compact.js'
import { estimateMessageTokens } from './microCompact.js'
import { getCompactUserSummaryMessage } from './prompt.js'
```

### 特性标志依赖

| 特性标志 | 用途 |
|---------|------|
| `tengu_session_memory` | 控制 session memory 功能启用 |
| `tengu_sm_compact` | 控制 session memory 压缩启用 |

### GrowthBook 配置键

| 配置键 | 类型 | 用途 |
|-------|------|------|
| `tengu_sm_compact_config` | `SessionMemoryCompactConfig` | 压缩阈值配置 |

### 环境变量

| 变量 | 用途 |
|-----|------|
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | 强制启用（覆盖） |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | 强制禁用（覆盖） |
| `USER_TYPE` | 控制调试日志（仅 'ant' 记录详细日志） |

## 风险、边界与改进建议

### 已知风险

1. **Session Memory 提取竞争条件**
   - 压缩可能在 session memory 提取进行时触发
   - 缓解：`waitForSessionMemoryExtraction` 等待最多 15 秒，60 秒后视为过期

2. **Message ID 不匹配**
   - 如果消息被修改，`lastSummarizedMessageId` 可能找不到对应消息
   - 缓解：回退到传统压缩

3. **Token 估算不准确**
   - 使用 `estimateMessageTokens` 进行估算，可能与实际 API 计数有偏差
   - 可能导致保留消息过多或过少

4. **阈值检查竞争**
   - 压缩后检查 `postCompactTokenCount >= autoCompactThreshold` 时，估算可能不准确
   - 可能导致立即再次触发压缩

### 边界情况

1. **空消息数组**
   ```typescript
   if (messages.length === 0) return 0
   ```
   `calculateMessagesToKeepIndex` 正确处理空数组

2. **所有消息都已总结**
   - `lastSummarizedIndex = messages.length - 1`
   - `startIndex = messages.length`（初始不保留任何消息）
   - 扩展逻辑会向后查找满足最小要求

3. **Session Memory 为空模板**
   - `isSessionMemoryEmpty` 检查内容与模板匹配
   - 回退到传统压缩

4. **配置值为零或负数**
   - 远程配置获取时检查 `> 0`
   - 使用默认值作为后备

### 改进建议

1. **添加缓存机制**
   - 缓存 `hasTextBlocks` 结果避免重复计算
   - 缓存工具配对检查结果

2. **优化索引调整算法**
   - 当前是 O(n²) 复杂度（嵌套循环）
   - 可优化为 O(n) 使用单次遍历

3. **增强错误恢复**
   - 区分可恢复错误和致命错误
   - 添加重试机制

4. **改进 token 估算**
   - 使用模型特定的 token 计数器
   - 定期校准估算偏差

5. **添加更多遥测**
   - 记录索引调整的次数和范围
   - 记录 session memory 命中率

6. **并发安全**
   - `configInitialized` 是模块级状态
   - 考虑使用原子操作或锁

### 代码示例：优化的索引调整

```typescript
export function adjustIndexToPreserveAPIInvariantsOptimized(
  messages: Message[],
  startIndex: number,
): number {
  if (startIndex <= 0 || startIndex >= messages.length) return startIndex

  // 单次遍历收集所需信息
  const toolResultIds = new Set<string>()
  const messageIds = new Set<string>()
  const toolUseLocations = new Map<string, number>()  // tool_use_id -> index

  for (let i = startIndex; i < messages.length; i++) {
    const msg = messages[i]!
    
    // 收集 tool_result ID
    if (msg.type === 'user' && Array.isArray(msg.message.content)) {
      for (const block of msg.message.content) {
        if (block.type === 'tool_result') {
          toolResultIds.add(block.tool_use_id)
        }
      }
    }
    
    // 收集 message.id
    if (msg.type === 'assistant' && msg.message.id) {
      messageIds.add(msg.message.id)
    }
  }

  // 向前扫描，记录所需 tool_use 的位置
  let adjustedIndex = startIndex
  for (let i = startIndex - 1; i >= 0; i--) {
    const msg = messages[i]!
    
    if (msg.type === 'assistant') {
      // 检查 tool_use
      if (Array.isArray(msg.message.content)) {
        for (const block of msg.message.content) {
          if (block.type === 'tool_use' && toolResultIds.has(block.id)) {
            adjustedIndex = i
            toolResultIds.delete(block.id)
          }
        }
      }
      
      // 检查 message.id 匹配
      if (msg.message.id && messageIds.has(msg.message.id)) {
        adjustedIndex = Math.min(adjustedIndex, i)
      }
    }
    
    if (toolResultIds.size === 0) break
  }

  return adjustedIndex
}
```
