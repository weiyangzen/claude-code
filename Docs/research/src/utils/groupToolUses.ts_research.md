# groupToolUses.ts 深度研究

## 场景与职责

本模块负责将来自同一 API 响应的多个工具调用进行分组聚合，以提供更紧凑的 UI 展示。当 Claude 在一次响应中并行调用多个同类工具（如多个文件读取）时，这些工具调用会被合并为一个分组消息，减少界面冗余。

**核心场景：**
1. **并行文件读取**：一次读取多个文件时合并展示
2. **批量搜索操作**：多个 Grep/Glob 调用合并
3. **UI 简化**：verbose 模式下保持原始位置，非 verbose 模式下分组显示

## 功能点目的

### 1. 工具调用分组
- **目的**：将同类型、同批次的工具调用聚合展示
- **条件**：
  - 工具支持分组渲染（`renderGroupedToolUse` 存在）
  - 同一 API 响应（相同 `message.id`）
  - 至少 2 个同类型工具

### 2. 结果关联
- **目的**：将工具调用结果与分组关联
- **机制**：通过 `tool_use_id` 匹配 `tool_result` 块

### 3. Verbose 模式支持
- **目的**：详细模式下跳过分组，保持原始消息顺序
- **场景**：用户需要查看完整执行流程时

## 具体技术实现

### 核心数据结构
```typescript
// 分组缓存 - WeakMap 避免内存泄漏
const GROUPING_CACHE = new WeakMap<Tools, Set<string>>()

// 分组结果类型
export type GroupingResult = {
  messages: RenderableMessage[]
}

// 分组后的消息类型
export type GroupedToolUseMessage = {
  type: 'grouped_tool_use'
  toolName: string
  messages: NormalizedAssistantMessage<BetaToolUseBlock>[]
  results: NormalizedUserMessage[]
  displayMessage: NormalizedAssistantMessage<BetaToolUseBlock>
  uuid: string
  timestamp: number
  messageId: string
}
```

### 关键流程

#### applyGrouping(messages, tools, verbose)
```
1. 如 verbose 模式，直接返回原消息
2. 获取支持分组的工具集合（缓存）
3. 第一遍扫描：按 messageId:toolName 分组
4. 识别有效分组（>=2 个工具）
5. 收集分组内所有 tool_use_id
6. 第二遍扫描：查找匹配的 tool_result
7. 第三遍扫描：构建输出
   - 首次遇到分组：创建 GroupedToolUseMessage
   - 后续遇到：跳过
   - 用户消息如全部结果已分组：跳过
```

### 缓存机制
```typescript
function getToolsWithGrouping(tools: Tools): Set<string> {
  let cached = GROUPING_CACHE.get(tools)
  if (!cached) {
    cached = new Set(tools.filter(t => t.renderGroupedToolUse).map(t => t.name))
    GROUPING_CACHE.set(tools, cached)
  }
  return cached
}
```

**设计要点：**
- 使用 `WeakMap`：tools 数组被替换时（如 MCP 连接/断开），缓存自动释放
- 避免每次渲染重建集合

### 工具分组资格
工具需实现 `renderGroupedToolUse` 方法才能参与分组：
```typescript
// Tool.ts 中的定义
renderGroupedToolUse?(
  toolUses: Array<{
    param: ToolUseBlockParam
    isResolved: boolean
    isError: boolean
    isInProgress: boolean
    progressMessages: ProgressMessage<P>[]
    result?: { param: ToolResultBlockParam; output: unknown }
  }>,
  options: { shouldAnimate: boolean; tools: Tools },
): React.ReactNode | null
```

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `applyGrouping` | 函数 | 主分组逻辑 |
| `GroupingResult` | 类型 | 分组结果类型 |
| `MessageWithoutProgress` | 类型 | 排除进度消息的 message 类型 |

### 调用方
1. **Messages.tsx**: 消息列表渲染时调用
2. **REPL.tsx**: 主界面消息处理

### 依赖模块
```typescript
import type { BetaToolUseBlock } from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'
import type { ToolResultBlockParam } from '@anthropic-ai/sdk/resources/messages/messages.mjs'
import type { Tools } from '../Tool.js'
import type {
  GroupedToolUseMessage,
  NormalizedAssistantMessage,
  NormalizedMessage,
  NormalizedUserMessage,
  ProgressMessage,
  RenderableMessage,
} from '../types/message.js'
```

### 消息类型定义（types/message.ts）
```typescript
export type NormalizedAssistantMessage<Content = MessageContent> = {
  type: 'assistant'
  message: AssistantMessage<Content>
  uuid: string
  timestamp: number
}

export type NormalizedUserMessage = {
  type: 'user'
  message: UserMessage
  uuid: string
  timestamp:  
  origin?: UserMessageOrigin
}

export type GroupedToolUseMessage = {
  type: 'grouped_tool_use'
  toolName: string
  messages: NormalizedAssistantMessage<BetaToolUseBlock>[]
  results: NormalizedUserMessage[]
  displayMessage: NormalizedAssistantMessage<BetaToolUseBlock>
  uuid: string
  timestamp: number
  messageId: string
}

export type RenderableMessage =
  | NormalizedUserMessage
  | NormalizedAssistantMessage
  | GroupedToolUseMessage
  | ProgressMessage
```

## 依赖与外部交互

### 上游依赖

1. **Tool.ts**: 工具定义
   - `renderGroupedToolUse`: 分组渲染方法
   - 工具名称和类型定义

2. **types/message.ts**: 消息类型
   - 各种消息类型定义
   - `RenderableMessage` 联合类型

### 下游影响

1. **Messages.tsx**: 消息渲染
   - 接收 `GroupingResult`
   - 根据 `type: 'grouped_tool_use'` 渲染分组组件

2. **GroupedToolUseContent.tsx**: 分组内容渲染
   - 专门处理分组消息的展示

## 风险、边界与改进建议

### 已知风险

1. **分组过度聚合**
   - 风险：不同语义的同类型工具被错误分组
   - 缓解：仅分组同一 API 响应（相同 messageId）

2. **结果匹配失败**
   - 风险：tool_result 未找到对应 tool_use
   - 缓解：未匹配的结果保留在消息列表中

3. **缓存失效**
   - 风险：WeakMap 缓存可能在工具更新前失效
   - 缓解：tools 数组引用变化时重建缓存

4. **时序问题**
   - 风险：工具调用和结果到达顺序不一致
   - 缓解：分组前收集所有消息，不依赖到达顺序

### 边界情况

1. **单工具调用**：不满足分组条件，保持原样
2. **混合工具类型**：不同类型不互相分组
3. **跨响应工具**：不同 messageId 的工具不分组
4. **无 renderGroupedToolUse**：该类型工具不参与分组
5. **部分结果到达**：已分组工具的结果后续到达时正确处理

### 改进建议

1. **智能分组策略**
   - 建议：考虑工具调用的语义关联性
   - 场景：读取同一目录的文件可分组，不相关的文件分开

2. **用户控制**
   - 建议：快捷键临时展开/折叠分组
   - 收益：用户可查看分组内详情

3. **分组动画**
   - 建议：分组形成时添加过渡动画
   - 收益：更好的视觉反馈

4. **分组统计**
   - 建议：显示分组内成功/失败数量
   - 场景：10 个文件读取中 8 成功 2 失败

5. **渐进式分组**
   - 建议：工具完成时动态加入分组
   - 挑战：需要处理 React 渲染时序

### 测试要点

1. 同一响应内多工具分组
2. 不同响应工具不分组
3. 不同类型工具不分组
4. 不支持分组的工具处理
5. verbose 模式跳过分组
6. 结果与工具正确关联
7. 缓存命中和重建
