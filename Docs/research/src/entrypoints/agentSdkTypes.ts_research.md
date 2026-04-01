# agentSdkTypes.ts 深度研究文档

## 文件元数据
- **路径**: `src/entrypoints/agentSdkTypes.ts`
- **大小**: 13,076 bytes
- **类型**: TypeScript 入口文件 / SDK 类型定义与 API 声明

---

## 一、场景与职责

### 1.1 核心定位
`agentSdkTypes.ts` 是 **Claude Code Agent SDK 的主入口文件**，承担以下关键职责：

1. **公共 API 类型导出中心**: 集中暴露 SDK 消费者需要的所有类型定义
2. **运行时函数声明**: 定义 SDK 核心功能函数（`query`, `unstable_v2_*` 系列）的接口
3. **桥接子路径消费者支持**: 为 `sdk/controlTypes.ts` 提供控制协议类型
4. **SDK 构建者类型**: 为构建 SDK 扩展的开发者提供类型支持

### 1.2 使用场景

| 场景 | 说明 |
|------|------|
| **SDK 消费者** | 外部开发者通过此文件导入类型和函数，编写与 Claude Code 交互的应用 |
| **SDK 构建者** | 需要控制协议类型的开发者从 `sdk/controlTypes.ts` 导入 |
| **内部模块** | 项目内部模块（如 `QueryEngine.ts`, `bridge/replBridge.ts` 等）依赖此文件的类型定义 |
| **MCP 集成** | 与 Model Context Protocol 集成的工具定义在此声明 |

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    SDK Consumer Code                        │
│         (import { query, unstable_v2_createSession }         │
│                 from '@anthropic-ai/claude-code')           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              src/entrypoints/agentSdkTypes.ts               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Re-exports from:                                   │   │
│  │  • sdk/coreTypes.ts (可序列化类型)                  │   │
│  │  • sdk/runtimeTypes.ts (回调、接口)                 │   │
│  │  • sdk/toolTypes.ts (工具类型)                      │   │
│  │  • sdk/settingsTypes.generated.ts (设置类型)        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Internal Implementation                        │
│    (实际实现在其他模块，此文件仅作声明)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、功能点目的

### 2.1 类型再导出系统

文件通过多层再导出组织类型：

```typescript
// 1. 控制协议类型（@alpha 标记）
export type { SDKControlRequest, SDKControlResponse } from './sdk/controlTypes.js'

// 2. 核心可序列化类型
export * from './sdk/coreTypes.js'

// 3. 运行时类型（回调、接口）
export * from './sdk/runtimeTypes.js'

// 4. 设置类型（从 JSON Schema 生成）
export type { Settings } from './sdk/settingsTypes.generated.js'

// 5. 工具类型（标记为 @internal）
export * from './sdk/toolTypes.js'
```

### 2.2 SDK 核心函数声明

所有函数目前均为**存根实现**（stub），抛出 "not implemented" 错误：

| 函数 | 用途 | 状态 |
|------|------|------|
| `tool()` | 创建 MCP 工具定义 | 未实现 |
| `createSdkMcpServer()` | 创建与 SDK 传输层配合的 MCP 服务器实例 | 未实现 |
| `query()` | 发起单次查询（旧版 API） | 未实现 |
| `unstable_v2_createSession()` | V2 API：创建持久会话 | 未实现 (@alpha) |
| `unstable_v2_resumeSession()` | V2 API：恢复现有会话 | 未实现 (@alpha) |
| `unstable_v2_prompt()` | V2 API：单次提示便捷函数 | 未实现 (@alpha) |
| `getSessionMessages()` | 读取会话对话消息 | 未实现 |
| `listSessions()` | 列出带元数据的会话 | 未实现 |
| `getSessionInfo()` | 读取单个会话的元数据 | 未实现 |
| `renameSession()` | 重命名会话 | 未实现 |
| `tagSession()` | 为会话添加标签 | 未实现 |
| `forkSession()` | 分叉会话到新分支 | 未实现 |
| `watchScheduledTasks()` | 监视计划任务 | 未实现 (@internal) |
| `buildMissedTaskNotification()` | 构建错过任务的通知 | 未实现 (@internal) |
| `connectRemoteControl()` | 从守护进程建立远程控制连接 | 未实现 (@internal) |

### 2.3 内部类型定义

文件定义了多个 `@internal` 标记的类型，用于内部子系统：

- **CronTask**: 计划任务结构（来自 `.claude/scheduled_tasks.json`）
- **CronJitterConfig**: Cron 调度器调优参数
- **ScheduledTaskEvent**: 计划任务事件（fire/missed）
- **ScheduledTasksHandle**: 任务监视句柄
- **InboundPrompt**: 来自 claude.ai 的用户消息
- **ConnectRemoteControlOptions**: 远程控制连接选项
- **RemoteControlHandle**: 远程控制句柄

---

## 三、具体技术实现

### 3.1 工具创建函数

```typescript
export function tool<Schema extends AnyZodRawShape>(
  _name: string,
  _description: string,
  _inputSchema: Schema,
  _handler: (args: InferShape<Schema>, extra: unknown) => Promise<CallToolResult>,
  _extras?: {
    annotations?: ToolAnnotations
    searchHint?: string
    alwaysLoad?: boolean
  },
): SdkMcpToolDefinition<Schema> {
  throw new Error('not implemented')
}
```

**技术要点**:
- 使用泛型 `Schema extends AnyZodRawShape` 实现类型安全的输入验证
- `InferShape<Schema>` 从 Zod schema 推断参数类型
- 返回 `SdkMcpToolDefinition<Schema>` 供 `createSdkMcpServer` 使用
- 依赖 `@modelcontextprotocol/sdk/types.js` 的 `CallToolResult` 和 `ToolAnnotations`

### 3.2 MCP 服务器创建

```typescript
type CreateSdkMcpServerOptions = {
  name: string
  version?: string
  tools?: Array<SdkMcpToolDefinition<any>>
}

export function createSdkMcpServer(
  _options: CreateSdkMcpServerOptions,
): McpSdkServerConfigWithInstance {
  throw new Error('not implemented')
}
```

**技术要点**:
- 允许 SDK 用户在同进程中定义自定义工具
- 返回 `McpSdkServerConfigWithInstance` 供 SDK 传输层使用
- 支持超时配置（通过 `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT`）

### 3.3 V2 会话 API（不稳定）

```typescript
export function unstable_v2_createSession(_options: SDKSessionOptions): SDKSession
export function unstable_v2_resumeSession(_sessionId: string, _options: SDKSessionOptions): SDKSession
export async function unstable_v2_prompt(_message: string, _options: SDKSessionOptions): Promise<SDKResultMessage>
```

**技术要点**:
- 标记为 `@alpha`，API 可能变化
- 支持多轮对话的持久会话
- `SDKSessionOptions` 来自 `runtimeTypes.ts`
- `SDKResultMessage` 来自 `coreTypes.ts`

### 3.4 会话管理函数

| 函数 | 关键参数 | 返回值 |
|------|----------|--------|
| `getSessionMessages` | `sessionId`, `options?` (dir, limit, offset, includeSystemMessages) | `Promise<SessionMessage[]>` |
| `listSessions` | `options?` (dir, limit, offset) | `Promise<SDKSessionInfo[]>` |
| `getSessionInfo` | `sessionId`, `options?` (dir) | `Promise<SDKSessionInfo \| undefined>` |
| `renameSession` | `sessionId`, `title`, `options?` | `Promise<void>` |
| `tagSession` | `sessionId`, `tag`, `options?` | `Promise<void>` |
| `forkSession` | `sessionId`, `options?` (dir, upToMessageId, title) | `Promise<ForkSessionResult>` |

### 3.5 远程控制连接（内部）

```typescript
export async function connectRemoteControl(
  _opts: ConnectRemoteControlOptions,
): Promise<RemoteControlHandle | null>
```

**关键特性**:
- 守护进程持有 WebSocket（父进程），代理子进程崩溃时可重新生成
- 与 `query.enableRemoteControl` 对比：后者在子进程中持有 WS，子进程死亡则 WS 死亡
- 跳过 `tengu_ccr_bridge` 门控和策略限制检查（调用者已预授权）
- 需要 OAuth（环境变量或钥匙串）

---

## 四、关键代码路径与文件引用

### 4.1 依赖关系图

```
agentSdkTypes.ts
├── @modelcontextprotocol/sdk/types.js (CallToolResult, ToolAnnotations)
├── ./sdk/controlTypes.js (SDKControlRequest, SDKControlResponse)
├── ./sdk/coreTypes.js (SDKMessage, SDKResultMessage, SDKSessionInfo, SDKUserMessage)
├── ./sdk/runtimeTypes.js (ForkSessionOptions, GetSessionInfoOptions, etc.)
├── ./sdk/settingsTypes.generated.js (Settings)
└── ./sdk/toolTypes.js (SdkMcpToolDefinition, etc.)
```

### 4.2 被依赖关系（部分）

```
被以下文件导入:
├── src/server/directConnectManager.ts
├── src/QueryEngine.ts
├── src/remote/SessionsWebSocket.ts
├── src/remote/RemoteSessionManager.ts
├── src/bridge/replBridge.ts
├── src/bridge/createSession.ts
├── src/bridge/remoteBridgeCore.ts
├── src/Tool.ts
├── src/cost-tracker.ts
├── src/cli/structuredIO.ts
├── src/utils/hooks.ts
├── src/utils/queryHelpers.ts
└── ... (共 50+ 个文件)
```

### 4.3 核心类型来源

| 类型 | 来源文件 | 用途 |
|------|----------|------|
| `SDKMessage`, `SDKUserMessage`, `SDKResultMessage` | `sdk/coreTypes.ts` | 消息传递基础类型 |
| `SDKSession`, `Query`, `InternalQuery` | `sdk/runtimeTypes.ts` | 运行时对象类型 |
| `SDKSessionOptions` | `sdk/runtimeTypes.ts` | 会话配置选项 |
| `Settings` | `sdk/settingsTypes.generated.ts` | 用户设置类型 |
| `CallToolResult`, `ToolAnnotations` | `@modelcontextprotocol/sdk` | MCP 协议类型 |

---

## 五、依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk/types.js` | MCP 协议类型（CallToolResult, ToolAnnotations） |

### 5.2 内部依赖

| 依赖路径 | 说明 |
|----------|------|
| `./sdk/controlTypes.js` | 控制协议类型（SDKControlRequest/Response） |
| `./sdk/coreTypes.js` | 核心可序列化类型 |
| `./sdk/runtimeTypes.js` | 运行时类型（回调、接口） |
| `./sdk/settingsTypes.generated.js` | 从 JSON Schema 生成的设置类型 |
| `./sdk/toolTypes.js` | 工具相关类型 |

### 5.3 运行时交互

此文件**仅包含类型定义和函数声明**，实际实现分布在：

- **QueryEngine.ts**: `query()` 的实现
- **远程会话管理**: `unstable_v2_*` 函数的实现
- **会话历史**: `getSessionMessages`, `listSessions` 等的实现
- **守护进程桥接**: `watchScheduledTasks`, `connectRemoteControl` 的实现

---

## 六、风险、边界与改进建议

### 6.1 当前风险

| 风险点 | 严重程度 | 说明 |
|--------|----------|------|
| **存根实现** | 高 | 所有函数均抛出 "not implemented"，SDK 实际上不可用 |
| **API 不稳定** | 中 | V2 API 标记为 `@alpha`，可能重大变化 |
| **类型与实现分离** | 中 | 类型定义与实际实现分离，可能导致不一致 |
| **内部类型暴露** | 低 | `@internal` 标记的类型仍被导出，可能被误用 |

### 6.2 边界条件

1. **超时处理**: `createSdkMcpServer` 文档提到超过 60s 的调用需要覆盖 `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT`
2. **会话恢复限制**: `forkSession` 不复制文件历史快照（undo history）
3. **远程控制限制**: `connectRemoteControl` 在无 OAuth 或注册失败时返回 null
4. **策略绕过**: `connectRemoteControl` 跳过策略限制检查，依赖调用者预授权

### 6.3 改进建议

1. **实现 SDK 功能**
   - 优先级：高
   - 当前所有函数均为存根，需要实际实现

2. **统一错误处理**
   - 当前统一抛出 `Error('not implemented')`
   - 建议定义专门的 SDK 错误类（如 `SDKNotImplementedError`）

3. **文档完善**
   - 为 `@internal` 类型添加更多使用说明
   - 明确 V2 API 的稳定化路线图

4. **类型安全增强**
   - 考虑为 `tool()` 函数的 `extra` 参数添加更具体的类型
   - 为 `_extras` 参数添加更详细的文档

5. **测试覆盖**
   - 添加类型级别的测试确保类型定义正确
   - 实现后添加集成测试

### 6.4 相关配置

| 环境变量 | 用途 |
|----------|------|
| `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT` | 覆盖 MCP 调用超时 |

---

## 七、总结

`agentSdkTypes.ts` 是 Claude Code Agent SDK 的**类型门面（facade）**，负责：

1. **统一类型导出**: 从多个内部模块收集并重新组织类型
2. **API 契约定义**: 声明 SDK 公共 API 的函数签名
3. **版本分层**: 区分稳定 API 和实验性 V2 API
4. **内部功能暴露**: 为高级用例提供内部类型（如远程控制、计划任务）

当前状态为**设计完成但未实现**，所有函数均为存根。实际功能需要查看调用这些类型的实现文件（如 `QueryEngine.ts`, `bridge/` 目录等）。
