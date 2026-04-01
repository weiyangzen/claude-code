# remotePermissionBridge.ts 研究文档

## 场景与职责

`remotePermissionBridge` 是 Claude Code CLI 中用于桥接远程 CCR（Claude Code Remote）权限请求与本地权限系统的适配层。它主要解决以下问题：

1. **远程工具使用权限处理**：当远程 CCR 容器需要执行工具时，需要本地用户确认
2. **消息类型适配**：将远程 SDK 的权限请求格式转换为本地 REPL 可显示的 `AssistantMessage`
3. **未知工具处理**：当远程 CCR 使用本地 CLI 不认识的工具（如 MCP 工具）时提供兜底方案

该模块是 `useRemoteSession` 和 `useDirectConnect` hooks 的关键依赖，确保远程会话的权限请求能够正确显示在本地 UI 中。

## 功能点目的

### 1. 合成助手消息创建 (`createSyntheticAssistantMessage`)

**问题背景**: 
- `ToolUseConfirm` 类型需要一个 `AssistantMessage` 来显示工具使用请求
- 但在远程模式下，工具实际在 CCR 容器中运行，本地没有真实的助手消息

**解决方案**:
- 根据远程权限请求数据合成一个虚拟的 `AssistantMessage`
- 包含工具使用块（tool_use block），使 UI 能够正常渲染权限请求

### 2. 工具存根创建 (`createToolStub`)

**问题背景**:
- 远程 CCR 可能使用本地 CLI 没有加载的工具（如 MCP 服务器提供的工具）
- 本地权限系统需要 `Tool` 对象来处理权限请求

**解决方案**:
- 为未知工具创建一个最小化的 `Tool` 存根对象
- 存根路由到 `FallbackPermissionRequest` 进行通用权限处理

## 具体技术实现

### 关键流程

#### 1. 合成消息创建流程
```typescript
export function createSyntheticAssistantMessage(
  request: SDKControlPermissionRequest,
  requestId: string,
): AssistantMessage
```

**实现步骤**:
```
1. 生成新的 UUID (randomUUID())
2. 构造 AssistantMessage 结构:
   - type: 'assistant'
   - uuid: 新生成的 UUID
   - message.id: `remote-${requestId}` (标识远程请求)
   - message.content: [tool_use block]
     - type: 'tool_use'
     - id: request.tool_use_id (来自远程)
     - name: request.tool_name
     - input: request.input
   - message.model: '' (空字符串，因为是合成的)
   - message.stop_reason: null
   - message.usage: 零值对象
   - requestId: undefined
   - timestamp: ISO 格式当前时间
```

#### 2. 工具存根创建流程
```typescript
export function createToolStub(toolName: string): Tool
```

**实现步骤**:
```
1. 返回 Tool 对象，包含:
   - name: 工具名称
   - inputSchema: 空对象（未知 schema）
   - isEnabled: () => true (总是启用)
   - userFacingName: () => toolName
   - renderToolUseMessage: 格式化输入参数显示
   - call: async () => ({ data: '' }) (空实现)
   - description: async () => '' (空实现)
   - prompt: () => '' (空实现)
   - isReadOnly: () => false (假设非只读)
   - isMcp: false
   - needsPermissions: () => true (总是需要权限)
```

### 数据结构

#### 合成助手消息结构
```typescript
{
  type: 'assistant',
  uuid: string,                    // 新生成的 UUID
  message: {
    id: `remote-${requestId}`,     // 关联到远程请求
    type: 'message',
    role: 'assistant',
    content: [
      {
        type: 'tool_use',
        id: request.tool_use_id,   // 远程工具使用 ID
        name: request.tool_name,   // 工具名称
        input: request.input,      // 工具输入参数
      }
    ],
    model: '',
    stop_reason: null,
    stop_sequence: null,
    container: null,
    context_management: null,
    usage: {
      input_tokens: 0,
      output_tokens: 0,
      cache_creation_input_tokens: 0,
      cache_read_input_tokens: 0,
    },
  },
  requestId: undefined,
  timestamp: string,               // ISO 8601 格式
}
```

#### 工具存根结构
```typescript
{
  name: string,
  inputSchema: {},                 // 空 schema
  isEnabled: () => true,
  userFacingName: () => string,
  renderToolUseMessage: (input) => string,
  call: async () => ({ data: '' }),
  description: async () => '',
  prompt: () => '',
  isReadOnly: () => false,
  isMcp: false,
  needsPermissions: () => true,
}
```

### 输入参数格式化

`renderToolUseMessage` 实现细节：
```typescript
renderToolUseMessage: (input: Record<string, unknown>) => {
  const entries = Object.entries(input)
  if (entries.length === 0) return ''
  return entries
    .slice(0, 3)  // 最多显示 3 个参数
    .map(([key, value]) => {
      const valueStr = typeof value === 'string' 
        ? value 
        : jsonStringify(value)  // 非字符串值 JSON 序列化
      return `${key}: ${valueStr}`
    })
    .join(', ')
}
```

## 关键代码路径与文件引用

### 核心文件
- **本文件**: `src/remote/remotePermissionBridge.ts` (78 lines)

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/entrypoints/sdk/controlTypes.ts` | `SDKControlPermissionRequest` 类型 |
| `src/Tool.ts` | `Tool` 类型定义 |
| `src/types/message.ts` | `AssistantMessage` 类型 |
| `src/utils/slowOperations.ts` | `jsonStringify` JSON 序列化 |

### 调用方文件
| 文件路径 | 用途 |
|---------|------|
| `src/hooks/useRemoteSession.ts` | 远程会话 hook，处理权限请求 |
| `src/hooks/useDirectConnect.ts` | 直接连接模式 hook |
| `src/hooks/useSSHSession.ts` | SSH 会话 hook |

### 关键函数引用路径
```
useRemoteSession / useDirectConnect
  ├── RemoteSessionManager (接收权限请求)
  │   └── onPermissionRequest 回调
  │       ├── createSyntheticAssistantMessage()
  │       │   └── 构造虚拟 AssistantMessage
  │       └── findToolByName() 或 createToolStub()
  │           └── 返回 Tool 对象（真实或存根）
  └── 构造 ToolUseConfirm 对象
      └── 加入权限请求队列
```

## 依赖与外部交互

### 类型依赖

#### SDK 控制类型 (`controlTypes.ts`)
```typescript
type SDKControlPermissionRequest = {
  subtype: 'can_use_tool'
  tool_name: string
  input: Record<string, unknown>
  permission_suggestions?: PermissionUpdate[]
  blocked_path?: string
  decision_reason?: string
  title?: string
  display_name?: string
  tool_use_id: string
  agent_id?: string
  description?: string
}
```

#### 内部消息类型 (`types/message.ts`)
```typescript
type AssistantMessage = {
  type: 'assistant'
  uuid: UUID
  message: APIAssistantMessage
  requestId?: string
  timestamp: string
  // ... 其他字段
}
```

#### 工具类型 (`Tool.ts`)
```typescript
type Tool<Input extends AnyObject = AnyObject, Output = unknown> = {
  name: string
  inputSchema: Input
  isEnabled: () => boolean
  userFacingName: (input?: Partial<z.infer<Input>>) => string
  renderToolUseMessage: (input: Record<string, unknown>) => string
  call: (args, context, canUseTool, parentMessage) => Promise<ToolResult<Output>>
  // ... 其他方法
}
```

### 运行时依赖
- **crypto**: `randomUUID()` 用于生成 UUID
- **JSON**: 通过 `jsonStringify` 包装函数序列化参数

## 风险、边界与改进建议

### 已知风险

#### 1. 合成消息的真实性问题
- **风险**: 合成消息的 `model` 为空字符串，`usage` 为零值，可能影响下游处理
- **潜在影响**: 某些分析/统计功能可能误判
- **缓解**: 当前实现中这些字段不被关键路径使用

#### 2. 工具存根的功能缺失
- **风险**: `call()` 返回空结果，如果意外被调用会导致问题
- **潜在影响**: 理论上不应被调用（权限流程会拦截），但防御性不足
- **缓解**: `needsPermissions()` 总是返回 true，强制走权限流程

#### 3. 输入参数截断
- **风险**: `renderToolUseMessage` 只显示前 3 个参数
- **潜在影响**: 复杂工具调用可能信息不完整
- **缓解**: 这是 UI 显示优化，完整数据在 `input` 对象中

### 边界条件

| 边界场景 | 行为 |
|---------|------|
| 空输入对象 | `renderToolUseMessage` 返回空字符串 |
| 嵌套对象输入 | 使用 `jsonStringify` 序列化显示 |
| 超长字符串值 | 完整显示（无截断逻辑） |
| 特殊字符 | 依赖 `jsonStringify` 转义 |

### 改进建议

#### 1. 增强工具存根信息
当前实现：
```typescript
return {
  name: toolName,
  inputSchema: {} as Tool['inputSchema'],
  // ...
  description: async () => '',
}
```

建议改进：
```typescript
return {
  name: toolName,
  inputSchema: {} as Tool['inputSchema'],
  // ...
  description: async () => `Remote tool: ${toolName}`,
  userFacingName: () => `${toolName} (remote)`,
  isMcp: true,  // 标记为远程/MCP 工具
}
```

#### 2. 添加工具来源标记
建议为合成消息添加元数据：
```typescript
return {
  // ...
  remoteOrigin: {
    requestId,
    isSynthetic: true,
  },
}
```

#### 3. 输入参数格式化优化
当前简单截断前 3 个参数，建议：
- 添加总长度限制
- 优先显示关键参数（如 path、command）
- 支持工具特定的格式化

#### 4. 类型安全增强
当前使用 `as` 类型断言：
```typescript
} as AssistantMessage['message']
} as unknown as Tool
```

建议：
- 使用 `satisfies` 关键字验证类型
- 添加运行时类型检查
- 完善类型定义避免 `unknown` 转换

#### 5. 测试覆盖
建议添加：
- 单元测试：验证合成消息结构
- 单元测试：验证工具存根行为
- 集成测试：与权限系统交互
- 边界测试：空输入、特殊字符、大对象

### 代码质量观察

#### 优点
- 职责单一，专注于桥接适配
- 代码简洁清晰，易于理解
- 类型定义完整
- 注释充分说明设计意图

#### 潜在问题
- 使用 `as` 类型断言可能隐藏类型错误
- `createToolStub` 返回的 Tool 对象不完整（缺少许多可选方法）
- 无验证输入参数合法性
- 依赖外部类型定义，可能因上游变更而失效

#### 设计权衡
- **合成 vs 真实消息**: 选择合成消息是为了复用现有权限 UI，但增加了概念复杂性
- **存根 vs 动态加载**: 选择存根是为了避免阻塞权限流程，但牺牲了工具功能完整性
- **简单格式化 vs 智能格式化**: 选择简单格式化是为了代码简洁，但可能影响用户体验
