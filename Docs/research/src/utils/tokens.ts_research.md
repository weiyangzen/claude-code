# tokens.ts 研究文档

## 场景与职责

`tokens.ts` 是 Claude Code CLI 的 Token 计数和上下文管理核心模块，提供对话历史中的 Token 使用统计、估算和管理功能。这是实现上下文窗口管理、自动压缩（auto-compact）和会话记忆（session memory）的关键基础设施。

主要使用场景：
1. **上下文大小计算**：计算当前对话占用的 Token 数量
2. **API 使用率跟踪**：从 API 响应中提取 Token 使用统计
3. **压缩决策**：为自动压缩算法提供准确的上下文大小数据
4. **会话记忆初始化**：确定会话记忆的初始上下文大小

## 功能点目的

### 1. 准确的 Token 计数
- **问题**：累积计数会重复计算（随着上下文增长）
- **解决方案**：基于最后一次 API 响应的 usage 数据，加上新增消息的估算

### 2. 并行工具调用处理
- **复杂性**：单次 API 响应可能产生多个 `tool_use` 块，每个成为独立的 AssistantMessage
- **解决方案**：识别相同 `message.id` 的拆分记录，确保所有穿插的 `tool_result` 都被计入

### 3. 多维度统计
- 输入 Token（input_tokens）
- 输出 Token（output_tokens）
- 缓存创建（cache_creation_input_tokens）
- 缓存读取（cache_read_input_tokens）
- 迭代统计（iterations，用于服务器端工具循环）

## 具体技术实现

### 核心类型

```typescript
// 来自 Anthropic SDK
import type { BetaUsage as Usage } from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'

interface Usage {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens?: number
  cache_read_input_tokens?: number
  iterations?: Array<{
    input_tokens: number
    output_tokens: number
  }> | null
  // ... 其他字段
}
```

### 关键函数

#### `getTokenUsage`

```typescript
export function getTokenUsage(message: Message): Usage | undefined
```

**过滤逻辑**：
```typescript
if (
  message?.type === 'assistant' &&
  'usage' in message.message &&
  !(
    message.message.content[0]?.type === 'text' &&
    SYNTHETIC_MESSAGES.has(message.message.content[0].text)
  ) &&
  message.message.model !== SYNTHETIC_MODEL
) {
  return message.message.usage
}
```

**目的**：
- 只从真实的 AssistantMessage 获取 usage
- 排除合成消息（如中断消息、取消消息）
- 排除合成模型（`<synthetic>`）

#### `getTokenCountFromUsage`

```typescript
export function getTokenCountFromUsage(usage: Usage): number
```

**计算**：
```typescript
return (
  usage.input_tokens +
  (usage.cache_creation_input_tokens ?? 0) +
  (usage.cache_read_input_tokens ?? 0) +
  usage.output_tokens
)
```

**说明**：包含缓存 Token，因为这是 API 计费的完整上下文大小。

#### `finalContextTokensFromLastResponse`

```typescript
export function finalContextTokensFromLastResponse(messages: Message[]): number
```

**特殊用途**：用于 `task_budget.remaining` 计算，跨压缩边界。

**关键逻辑**：
```typescript
const iterations = (usage as { iterations?: Array<{...}> | null }).iterations
if (iterations && iterations.length > 0) {
  const last = iterations.at(-1)!
  return last.input_tokens + last.output_tokens  // 不包含缓存
}
// 无 iterations：顶层 usage 就是最终窗口
return usage.input_tokens + usage.output_tokens  // 不包含缓存
```

**设计说明**：服务器端的预算倒计时是基于上下文的，所以使用非缓存的 input + output。

#### `tokenCountWithEstimation`

```typescript
export function tokenCountWithEstimation(messages: readonly Message[]): number
```

**核心算法**（处理并行工具调用）：

```typescript
let i = messages.length - 1
while (i >= 0) {
  const message = messages[i]
  const usage = message ? getTokenUsage(message) : undefined
  if (message && usage) {
    // 找到 usage 记录，向前查找同一次 API 响应的兄弟记录
    const responseId = getAssistantMessageId(message)
    if (responseId) {
      let j = i - 1
      while (j >= 0) {
        const prior = messages[j]
        const priorId = prior ? getAssistantMessageId(prior) : undefined
        if (priorId === responseId) {
          i = j  // 向前锚定到第一个兄弟
        } else if (priorId !== undefined) {
          break  // 遇到不同的 API 响应
        }
        // priorId === undefined：user/tool_result/attachment，继续向前
        j--
      }
    }
    return (
      getTokenCountFromUsage(usage) +
      roughTokenCountEstimationForMessages(messages.slice(i + 1))
    )
  }
  i--
}
return roughTokenCountEstimationForMessages(messages)
```

**为什么需要向前查找？**

考虑以下消息序列：
```
[assistant(id=A, tool_use:1), user(result:1), 
 assistant(id=A, tool_use:2), user(result:2)]
```

如果停在最后一个 assistant，只会估算 `result:2` 之后的消息（没有），漏掉 `result:1`。
通过向前查找到第一个 `id=A` 的消息，确保两个 `user(result)` 都被计入。

### Token 估算

当没有 API usage 数据时（如新消息还未发送），使用启发式估算：

```typescript
export function roughTokenCountEstimation(content: string, bytesPerToken: number = 4): number {
  return Math.round(content.length / bytesPerToken)
}
```

**优化**：
- JSON 文件使用 `bytesPerToken = 2`（密集 JSON 单字符 Token 多）
- 图像/文档使用固定估算 2000 Token

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/query.ts` | 查询上下文大小计算 |
| `src/services/SessionMemory/sessionMemory.ts` | 会话记忆初始化 |
| `src/services/compact/compact.ts` | 压缩决策 |
| `src/services/compact/sessionMemoryCompact.ts` | 会话记忆压缩 |
| `src/services/compact/autoCompact.ts` | 自动压缩触发 |
| `src/services/api/claude.ts` | API 调用 Token 统计 |
| `src/tools/AgentTool/AgentTool.tsx` | Agent 工具 Token 管理 |
| `src/tools/AgentTool/agentToolUtils.ts` | Agent 工具辅助函数 |
| `src/components/StatusLine.tsx` | 状态栏 Token 显示 |
| `src/components/PromptInput/Notifications.tsx` | 输入框 Token 通知 |
| `src/utils/analyzeContext.ts` | 上下文分析 |
| `src/utils/swarm/inProcessRunner.ts` | Swarm 执行器 |
| `src/utils/attachments.ts` | 附件 Token 估算 |
| `src/utils/permissions/yoloClassifier.ts` | 权限分类器 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 斜杠命令处理 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `@anthropic-ai/sdk` | `BetaUsage` 类型 |
| `../services/tokenEstimation.js` | 消息 Token 估算 |
| `../types/message.js` | `Message`, `AssistantMessage` 类型 |
| `./messages.js` | `SYNTHETIC_MESSAGES`, `SYNTHETIC_MODEL` |
| `./slowOperations.js` | `jsonStringify` |

## 依赖与外部交互

### 与 TokenEstimation 服务的集成

```typescript
import { roughTokenCountEstimationForMessages } from '../services/tokenEstimation.js'
```

**分工**：
- `tokens.ts`：基于 API usage 的精确计数 + 简单估算
- `tokenEstimation.ts`：更复杂的估算逻辑（Haiku fallback、Bedrock CountTokens 等）

### 与消息系统的集成

**合成消息过滤**：
```typescript
const SYNTHETIC_MESSAGES = new Set([
  INTERRUPT_MESSAGE,
  INTERRUPT_MESSAGE_FOR_TOOL_USE,
  CANCEL_MESSAGE,
  REJECT_MESSAGE,
  NO_RESPONSE_REQUESTED,
])
```

这些消息不携带真实的 API usage，需要排除。

### 与 Compact 服务的集成

```typescript
// autoCompact.ts 示例
const contextSize = tokenCountWithEstimation(messages)
if (contextSize > COMPACT_THRESHOLD) {
  triggerCompact()
}
```

## 风险、边界与改进建议

### 潜在风险

1. **并行工具调用复杂性**
   - 向前查找逻辑复杂，容易出错
   - 如果消息顺序异常，可能计数错误

2. **估算误差**
   - `content.length / 4` 是粗略估算
   - 对于代码（符号多）可能低估，对于自然语言可能高估

3. **缓存 Token 处理不一致**
   - `getTokenCountFromUsage` 包含缓存
   - `finalContextTokensFromLastResponse` 不包含缓存
   - 调用方需清楚使用哪个函数

4. **TypeScript 类型断言**
   ```typescript
   const iterations = (usage as { iterations?: ... }).iterations
   ```
   - Stainless 类型不包含 `iterations`，需要类型断言
   - 如果 API 格式变化，编译器无法检测

### 边界条件

| 场景 | 行为 |
|-----|------|
| 空消息数组 | `tokenCountWithEstimation([])` → 0（通过估算）|
| 无 usage 的消息 | 使用 `roughTokenCountEstimationForMessages` |
| 并行工具调用（相同 id） | 正确识别并包含所有穿插消息 |
| 拆分消息中间插入其他消息 | 向前查找会停止（priorId !== undefined）|
| 合成消息 | `getTokenUsage` 返回 undefined |

### 改进建议

1. **更精确的估算**
   ```typescript
   // 基于 Tokenizer 的估算
   export async function estimateTokensWithTokenizer(text: string): Promise<number> {
     // 使用 tiktoken 或类似库
   }
   ```

2. **缓存策略优化**
   ```typescript
   // 缓存 Token 计数结果（消息数组不可变时）
   const tokenCountCache = new WeakMap<readonly Message[], number>()
   ```

3. **类型安全增强**
   ```typescript
   // 定义包含 iterations 的扩展类型
   interface ExtendedUsage extends Usage {
     iterations?: Array<{ input_tokens: number; output_tokens: number }> | null
   }
   ```

4. **调试和可观测性**
   ```typescript
   export function getTokenCountDebugInfo(messages: Message[]): {
     lastUsage: Usage | undefined
     estimatedMessages: number
     estimationBreakdown: Array<{ type: string; tokens: number }>
   }
   ```

5. **支持更多内容类型**
   ```typescript
   // 当前：text, thinking, redacted_thinking, tool_use
   // 建议：image, document, tool_result 等
   function getAssistantMessageContentLength(message: AssistantMessage): number {
     // 扩展支持更多类型
   }
   ```

6. **预算预警**
   ```typescript
   export function getTokenBudgetStatus(
     messages: Message[],
     budget: number,
   ): { status: 'ok' | 'warning' | 'critical'; remaining: number }
   ```

### 测试建议

应覆盖以下场景：
- 基本 usage 提取和计算
- 并行工具调用的拆分/合并
- 无 usage 消息的估算
- 合成消息的过滤
- 缓存 Token 的包含/排除
- iterations 的处理
- 超大消息数组性能
