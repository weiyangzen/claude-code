# grouping.ts 深度研究文档

## 场景与职责

`grouping.ts` 实现了消息按 API 轮次分组的算法，是上下文压缩和响应式压缩功能的基础工具模块。它将消息列表分割为逻辑上的"API 轮次"组，每组代表一次完整的用户-助手交互循环。

该模块的提取是为了解决 `compact.ts` 和 `compactMessages.ts` 之间的循环依赖问题（CC-1180），同时提供更细粒度的消息分组能力，支持单轮次 agentic 会话的压缩。

## 功能点目的

### 1. API 轮次分组
- 将消息列表按助手消息 ID 变化边界分割
- 每个分组代表一次完整的 API 请求-响应周期

### 2. 循环依赖解耦
- 从 `compact.ts` 中提取，打破与 `compactMessages.ts` 的循环依赖
- 解决模块初始化顺序导致的 ws CJS/ESM 解析竞争问题

### 3. 细粒度压缩支持
- 支持单轮次 agentic 会话的压缩（SDK/CCR/eval 场景）
- 替代之前仅基于真实用户输入的人类轮次分组

## 具体技术实现

### 核心算法

```typescript
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    // 边界检测：新的助手消息 ID 且当前组非空
    if (
      msg.type === 'assistant' &&
      msg.message.id !== lastAssistantId &&
      current.length > 0
    ) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    // 更新最后看到的助手消息 ID
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }

  if (current.length > 0) {
    groups.push(current)
  }
  return groups
}
```

### 算法详解

**边界条件**:
- 助手消息类型 (`msg.type === 'assistant'`)
- 消息 ID 变化 (`msg.message.id !== lastAssistantId`)
- 当前组非空 (`current.length > 0`)

**关键设计决策**:

1. **仅使用 `lastAssistantId` 作为边界**:
   - API 契约保证每次助手响应前所有 tool_use 都已解析
   - 无需跟踪未解析的 tool_use ID
   - 简化逻辑，避免畸形输入导致的无限合并

2. **消息 ID vs UUID**:
   - 使用 `message.id`（API 返回的消息 ID）而非 `uuid`（内部生成的消息 UUID）
   - 同一 API 响应的流式块共享相同的 `message.id`
   - 正确保持 `[tu_A(id=X), result_A, tu_B(id=X)]` 在同一组

### 分组示例

```
输入消息序列:
  [0] user: "Hello"                                    ─┐
  [1] assistant(id=1): "Hi"                            ─┤ Group 0 (Preamble)
  [2] user: "Read file A"                              ─┤
  [3] assistant(id=2): [tool_use: read]                ─┤
  [4] user: [tool_result: content of A]                ─┘
  [5] assistant(id=3): "Here's the content..."         ─┐
  [6] user: "Edit line 5"                              ─┤ Group 1
  [7] assistant(id=4): [tool_use: edit]                ─┤
  [8] user: [tool_result: success]                     ─┘
  [9] assistant(id=5): "Done"                          ──> Group 2

输出分组:
  Group 0: [0, 1, 2, 3, 4]
  Group 1: [5, 6, 7, 8]
  Group 2: [9]
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `../../types/message.js` | `Message` 类型定义 |

### 外部调用方

| 调用方 | 路径 | 用途 |
|-------|------|------|
| `compact.ts` | `src/services/compact/compact.ts:115,267` | PTL 重试时的消息截断 |
| `reactiveCompact.ts` | `src/services/compact/reactiveCompact.ts` | 响应式压缩的消息分组 |

### 调用代码片段

```typescript
// src/services/compact/compact.ts:truncateHeadForPTLRetry
const groups = groupMessagesByApiRound(input)
if (groups.length < 2) return null

// 计算需要丢弃的组数...
const sliced = groups.slice(dropCount).flat()
```

## 依赖与外部交互

### 类型依赖

```typescript
// 来自 ../../types/message.js
interface Message {
  type: 'user' | 'assistant' | 'system' | ...
  message: {
    id?: string  // API 消息 ID
    content: ...
  }
  uuid: string   // 内部 UUID（本模块不使用）
  // ...
}
```

## 风险、边界与改进建议

### 已知风险

1. **畸形输入处理**:
   - 如果输入包含悬挂的 tool_use（恢复/截断后的畸形状态）
   - 边界仍会触发，但可能导致 tool_use/tool_result 配对问题
   - 依赖调用方的 `ensureToolResultPairing` 修复

2. **单消息场景**:
   - 如果消息少于 2 组，`truncateHeadForPTLRetry` 返回 null
   - 可能导致 PTL 无法恢复

### 边界条件

| 场景 | 行为 |
|------|------|
| 空消息数组 | 返回空数组 `[]` |
| 单条消息 | 返回 `[[msg]]` |
| 无助手消息 | 所有消息归入一组 |
| 连续相同 ID 的助手消息 | 归入同一组（流式块） |
| 第一条消息是助手 | 作为新组开始（但通常 API 以用户消息开头） |

### 改进建议

1. **添加单元测试覆盖边界**:
   ```typescript
   // 建议测试用例
   - 空输入
   - 单消息
   - 无助手消息
   - 所有助手消息相同 ID
   - 助手消息 ID 交替变化
   - 真实世界消息序列
   ```

2. **性能优化**（大数据量场景）:
   ```typescript
   // 当前实现是 O(n)，已足够高效
   // 如需进一步优化，可考虑惰性分组
   export function* iterateApiRounds(messages: Message[]): Generator<Message[]> {
     // 惰性生成器实现
   }
   ```

3. **文档增强**:
   - 添加更多可视化示例
   - 说明与 `normalizeMessagesForAPI` 的协作关系

4. **当前设计保持**: 该模块职责单一，算法简洁，无需重大修改
