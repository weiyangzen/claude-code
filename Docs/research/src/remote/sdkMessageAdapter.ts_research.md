# sdkMessageAdapter.ts 研究文档

## 场景与职责

`sdkMessageAdapter` 是 Claude Code CLI 中负责将远程 CCR（Claude Code Remote）发送的 SDK 格式消息转换为本地 REPL 内部消息类型的适配器模块。它是远程会话功能的核心组件，承担以下职责：

1. **消息格式转换**：将 `SDKMessage` 转换为内部 `Message` 类型，供 REPL 渲染
2. **协议适配**：桥接 CCR 后端协议与本地 REPL 消息系统
3. **消息过滤**：根据配置选项决定哪些消息需要转换、哪些需要忽略
4. **会话结束检测**：识别会话结束消息，触发相应处理

该模块被 `useRemoteSession`、`useDirectConnect` 和 `useAssistantHistory` 等 hooks 广泛使用，是远程会话数据流的关键环节。

## 功能点目的

### 1. 消息类型转换
将 SDK 消息（来自 WebSocket）转换为 REPL 内部消息格式：

| SDK 消息类型 | 内部消息类型 | 用途 |
|-------------|-------------|------|
| `assistant` | `AssistantMessage` | 助手回复 |
| `user` | `UserMessage` (可选) | 用户消息（历史同步） |
| `stream_event` | `StreamEvent` | 流式输出事件 |
| `result` | `SystemMessage` (错误时) | 会话结果 |
| `system` | `SystemMessage` | 系统消息（init/status/compact_boundary） |
| `tool_progress` | `SystemMessage` | 工具执行进度 |

### 2. 转换选项 (`ConvertOptions`)
```typescript
type ConvertOptions = {
  convertToolResults?: boolean    // 转换包含 tool_result 的用户消息
  convertUserTextMessages?: boolean // 转换用户文本消息（历史同步）
}
```

### 3. 会话结束检测
```typescript
function isSessionEndMessage(msg: SDKMessage): boolean
function isSuccessResult(msg: SDKResultMessage): boolean
function getResultText(msg: SDKResultMessage): string | null
```

## 具体技术实现

### 关键流程

#### 1. 主转换流程 (`convertSDKMessage()`)
```
输入: SDKMessage + ConvertOptions
输出: ConvertedMessage (message | stream_event | ignored)

switch (msg.type):
  case 'assistant':
    → convertAssistantMessage() → AssistantMessage
  
  case 'user':
    → 检查 convertToolResults 和 isToolResult
    → 检查 convertUserTextMessages
    → 可能: createUserMessage() → UserMessage
    → 或: { type: 'ignored' }
  
  case 'stream_event':
    → convertStreamEvent() → StreamEvent
  
  case 'result':
    → 仅错误时转换 → SystemMessage
    → 成功: { type: 'ignored' }
  
  case 'system':
    → 根据 subtype 分发:
      - 'init': convertInitMessage()
      - 'status': convertStatusMessage()
      - 'compact_boundary': convertCompactBoundaryMessage()
      - 其他: 记录调试日志，ignored
  
  case 'tool_progress':
    → convertToolProgressMessage() → SystemMessage
  
  case 'auth_status' | 'tool_use_summary' | 'rate_limit_event':
    → 记录调试日志，ignored
  
  default:
    → 记录调试日志（未知类型），ignored
```

#### 2. 助手消息转换 (`convertAssistantMessage`)
```typescript
{
  type: 'assistant',
  message: msg.message,           // 直接使用 SDK 的消息内容
  uuid: msg.uuid,
  requestId: undefined,
  timestamp: new Date().toISOString(),
  error: msg.error,               // 透传错误信息
}
```

#### 3. 流事件转换 (`convertStreamEvent`)
```typescript
{
  type: 'stream_event',
  event: msg.event,               // 透传事件数据
}
```

#### 4. 结果消息转换 (`convertResultMessage`)
```typescript
// 错误时
{
  type: 'system',
  subtype: 'informational',
  content: errors?.join(', ') || 'Unknown error',
  level: 'warning',
  uuid: msg.uuid,
  timestamp: new Date().toISOString(),
}

// 成功时: 返回 ignored（不显示）
```

#### 5. 系统消息转换

**Init 消息**:
```typescript
{
  type: 'system',
  subtype: 'informational',
  content: `Remote session initialized (model: ${msg.model})`,
  level: 'info',
  // ...
}
```

**Status 消息**:
```typescript
// status === 'compacting'
content: 'Compacting conversation…'
// 其他 status
content: `Status: ${msg.status}`
```

**Compact Boundary 消息**:
```typescript
{
  type: 'system',
  subtype: 'compact_boundary',
  content: 'Conversation compacted',
  level: 'info',
  compactMetadata: fromSDKCompactMetadata(msg.compact_metadata),
}
```

#### 6. 工具进度消息转换 (`convertToolProgressMessage`)
```typescript
{
  type: 'system',
  subtype: 'informational',
  content: `Tool ${msg.tool_name} running for ${msg.elapsed_time_seconds}s…`,
  level: 'info',
  toolUseID: msg.tool_use_id,
}
```

### 数据结构

#### 转换结果类型
```typescript
export type ConvertedMessage =
  | { type: 'message'; message: Message }
  | { type: 'stream_event'; event: StreamEvent }
  | { type: 'ignored' }
```

#### 工具结果检测逻辑
```typescript
// 检测用户消息是否为工具结果
const isToolResult =
  Array.isArray(content) && content.some(b => b.type === 'tool_result')
```

**重要说明**: 不使用 `parent_tool_use_id` 判断，因为 `normalizeMessage()` 会将其硬编码为 `null`。

### 协议与消息格式

#### 支持的 SDK 消息类型
```typescript
type SDKMessage =
  | SDKAssistantMessage
  | SDKUserMessage
  | SDKPartialAssistantMessage  // stream_event
  | SDKResultMessage
  | SDKSystemMessage
  | SDKToolProgressMessage
  | SDKCompactBoundaryMessage
  | SDKAuthStatusMessage
  | SDKToolUseSummaryMessage
  | SDKRateLimitEventMessage
```

#### 转换行为矩阵

| SDK 类型 | 默认行为 | viewerOnly 模式 | 说明 |
|---------|---------|----------------|------|
| `assistant` | 转换 | 转换 | 总是显示 |
| `user` (tool_result) | ignored | 转换 | viewerOnly 需要显示工具结果 |
| `user` (text) | ignored | 转换 | 历史同步时使用 |
| `stream_event` | 转换 | 转换 | 流式更新 |
| `result` (error) | 转换 | 转换 | 显示错误 |
| `result` (success) | ignored | ignored | 不显示成功结果 |
| `system/init` | 转换 | 转换 | 显示初始化信息 |
| `system/status` | 转换 | 转换 | 显示状态变化 |
| `system/compact_boundary` | 转换 | 转换 | 显示 compact 信息 |
| `tool_progress` | 转换 | 转换 | 显示进度 |
| `auth_status` | ignored | ignored | 内部使用 |
| `tool_use_summary` | ignored | ignored | 内部使用 |
| `rate_limit_event` | ignored | ignored | 内部使用 |

## 关键代码路径与文件引用

### 核心文件
- **本文件**: `src/remote/sdkMessageAdapter.ts` (302 lines)

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/entrypoints/agentSdkTypes.ts` | SDK 消息类型定义 |
| `src/types/message.ts` | 内部消息类型定义 |
| `src/utils/debug.ts` | `logForDebugging` 调试日志 |
| `src/utils/messages/mappers.ts` | `fromSDKCompactMetadata` 元数据转换 |
| `src/utils/messages.ts` | `createUserMessage` 用户消息创建 |

### 调用方文件
| 文件路径 | 用途 |
|---------|------|
| `src/hooks/useRemoteSession.ts` | 远程会话消息处理 |
| `src/hooks/useDirectConnect.ts` | 直接连接消息处理 |
| `src/hooks/useAssistantHistory.ts` | 历史记录加载 |
| `src/hooks/useSSHSession.ts` | SSH 会话消息处理 |

### 关键函数引用路径
```
useRemoteSession.onMessage(sdkMessage)
  └── convertSDKMessage(sdkMessage, opts)
      ├── case 'assistant':
      │   └── convertAssistantMessage()
      ├── case 'user':
      │   ├── 检测 isToolResult
      │   └── 可能: createUserMessage() [src/utils/messages.ts]
      ├── case 'system':
      │   └── 根据 subtype 分发
      │       └── convertCompactBoundaryMessage()
      │           └── fromSDKCompactMetadata() [src/utils/messages/mappers.ts]
      └── 返回: ConvertedMessage
```

## 依赖与外部交互

### 类型依赖

#### SDK 消息类型 (`agentSdkTypes.ts`)
```typescript
// 核心类型（实际定义在 coreTypes.ts）
type SDKAssistantMessage = {
  type: 'assistant'
  message: APIAssistantMessage
  uuid: string
  error?: SDKAssistantMessageError
}

type SDKUserMessage = {
  type: 'user'
  message: { role: 'user'; content: string | ContentBlockParam[] }
  uuid: string
  timestamp?: string
  tool_use_result?: unknown
}

type SDKResultMessage = {
  type: 'result'
  subtype: 'success' | 'error'
  result?: string
  errors?: string[]
  uuid: string
}
```

#### 内部消息类型 (`types/message.ts`)
```typescript
type Message =
  | AssistantMessage
  | UserMessage
  | SystemMessage
  | ProgressMessage
  | AttachmentMessage
  | StreamEvent

type AssistantMessage = {
  type: 'assistant'
  message: APIAssistantMessage
  uuid: UUID
  requestId?: string
  timestamp: string
  error?: SDKAssistantMessageError
}
```

### 元数据转换依赖

`fromSDKCompactMetadata` (`src/utils/messages/mappers.ts`):
```typescript
// SDK 格式 → 内部格式
{
  trigger: string
  pre_tokens: number
  preserved_segment?: {
    head_uuid: string
    anchor_uuid: string
    tail_uuid: string
  }
}
↓
{
  trigger: string
  preTokens: number
  preservedSegment?: {
    headUuid: string
    anchorUuid: string
    tailUuid: string
  }
}
```

## 风险、边界与改进建议

### 已知风险

#### 1. 消息类型扩展性
- **风险**: 新增 SDK 消息类型需要修改 switch case，容易遗漏
- **当前缓解**: `default` 分支记录调试日志， graceful degradation
- **潜在问题**: 新功能可能静默失效，难以发现

#### 2. 工具结果检测可靠性
- **风险**: 依赖 `content.some(b => b.type === 'tool_result')` 检测
- **潜在问题**: 如果消息结构变化，检测可能失效
- **注释说明**: 明确提到不使用 `parent_tool_use_id` 的原因

#### 3. 时间戳一致性
- **风险**: 所有转换后的消息使用 `new Date().toISOString()` 生成时间戳
- **潜在问题**: 与原始消息时间戳可能不一致，影响消息排序

#### 4. 错误消息聚合
- **风险**: `convertResultMessage` 简单用 `, ` 连接多个错误
- **潜在问题**: 大量错误时消息过长，可读性差

### 边界条件

| 边界场景 | 行为 |
|---------|------|
| 空 content 数组 | `isToolResult` 返回 false |
| 未知 system subtype | 记录调试日志，返回 ignored |
| 未知 message type | 记录调试日志，返回 ignored |
| status 为 null/undefined | 返回 null，调用方处理为 ignored |
| 成功 result 消息 | 返回 ignored（不显示） |

### 改进建议

#### 1. 增强类型安全
当前 `default` 分支使用 `as` 断言：
```typescript
default: {
  logForDebugging(
    `[sdkMessageAdapter] Unknown message type: ${(msg as { type: string }).type}`,
  )
}
```

建议：
```typescript
// 使用穷尽检查
const _exhaustive: never = msg
default:
  logForDebugging(`[sdkMessageAdapter] Unknown message type: ${_exhaustive.type}`)
```

#### 2. 添加消息验证
建议添加 Zod schema 验证输入消息：
```typescript
import { SDKMessageSchema } from '../entrypoints/sdk/coreSchemas.js'

export function convertSDKMessage(msg: unknown, opts?: ConvertOptions): ConvertedMessage {
  const parsed = SDKMessageSchema.safeParse(msg)
  if (!parsed.success) {
    logForDebugging(`[sdkMessageAdapter] Invalid message: ${parsed.error}`)
    return { type: 'ignored' }
  }
  // ... 继续处理
}
```

#### 3. 优化错误消息处理
当前简单连接错误：
```typescript
const content = isError
  ? msg.errors?.join(', ') || 'Unknown error'
  : 'Session completed successfully'
```

建议：
```typescript
const content = isError
  ? formatErrors(msg.errors)  // 智能格式化，限制长度
  : null  // 成功时返回 null，让调用方决定是否显示
```

#### 4. 保留原始时间戳
对于历史同步场景：
```typescript
function convertAssistantMessage(msg: SDKAssistantMessage): AssistantMessage {
  return {
    // ...
    timestamp: msg.timestamp || new Date().toISOString(),
  }
}
```

#### 5. 添加转换指标
建议收集转换统计：
```typescript
// 在 debug 模式下
let stats = { converted: 0, ignored: 0, errors: 0 }
// 定期输出或按需查询
```

#### 6. 支持更多 system subtypes
当前忽略的 subtypes：
- `hook_response`
- 其他未明确处理的 subtypes

建议：
- 明确列出所有已知 subtypes
- 为常用 subtypes 添加转换支持
- 为未知 subtypes 提供通用处理方式

### 代码质量观察

#### 优点
- switch case 结构清晰，易于理解
- 转换逻辑与业务逻辑分离良好
- 丰富的调试日志便于问题排查
- 选项设计灵活，支持不同使用场景

#### 潜在问题
- 函数较长（278 行），职责较多
- 部分转换逻辑重复（timestamp 生成）
- 无单元测试（从代码中未看到）
- 依赖全局 `Date`，不利于测试

#### 性能考虑
- 每次转换创建新的 Date 对象
- `isToolResult` 遍历整个 content 数组
- 字符串拼接使用模板字面量（高效）
- 无明显的内存泄漏风险
