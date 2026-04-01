# MCP 类型定义 (types.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`types.ts` 是 Claude Code MCP 系统的**类型定义中心**，提供：
- **配置类型**：所有 MCP 服务器配置的结构化类型定义
- **运行时类型**：服务器连接状态、工具、资源的类型
- **验证 Schema**：基于 Zod 的运行时验证
- **序列化类型**：CLI 状态持久化的类型

### 1.2 设计原则
- **单一职责**：纯类型定义文件，无业务逻辑
- **运行时验证**：使用 Zod schema 实现类型安全
- **向后兼容**：通过 `lazySchema` 延迟初始化避免循环依赖
- **可扩展性**：union 类型支持多种服务器传输方式

---

## 2. 功能点目的

### 2.1 配置类型体系
**目的**：支持 MCP 协议定义的多种传输方式，以及 Claude Code 特有的扩展。

**传输类型覆盖**：
- **stdio**：标准输入输出（本地子进程）
- **sse**：Server-Sent Events（HTTP 流）
- **http**：Streamable HTTP（MCP 2025-03-26 规范）
- **ws**：WebSocket
- **sse-ide/ws-ide**：IDE 扩展专用
- **sdk**：SDK 控制传输（VSCode 等）
- **claudeai-proxy**：Claude.ai 代理连接

### 2.2 运行时状态管理
**目的**：精确描述 MCP 服务器在运行时的各种状态。

**状态机**：
```
pending -> connected
   |         |
   v         v
needs-auth  failed
disabled
```

### 2.3 延迟 Schema 初始化
**目的**：解决 Zod schema 之间的循环依赖问题。

**实现**：使用 `lazySchema` 包装器，在首次访问时才初始化 schema。

---

## 3. 具体技术实现

### 3.1 核心类型定义

#### 3.1.1 配置作用域
```typescript
export const ConfigScopeSchema = lazySchema(() =>
  z.enum([
    'local',      // 项目本地配置（不共享）
    'user',       // 用户全局配置
    'project',    // 项目配置（.mcp.json）
    'dynamic',    // 动态配置（命令行）
    'enterprise', // 企业托管配置
    'claudeai',   // Claude.ai 连接器
    'managed',    // 托管配置
  ])
)
export type ConfigScope = z.infer<ReturnType<typeof ConfigScopeSchema>>
```

#### 3.1.2 传输类型
```typescript
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk'])
)
export type Transport = z.infer<ReturnType<typeof TransportSchema>>
```

#### 3.1.3 服务器配置（Discriminated Union）
```typescript
// stdio 服务器（最常用）
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(),  // 可选，向后兼容
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  })
)

// SSE 服务器（支持 OAuth）
export const McpSSEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sse'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),  // 动态获取 headers 的脚本
    oauth: McpOAuthConfigSchema().optional(),
  })
)

// HTTP 服务器（Streamable HTTP）
export const McpHTTPServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('http'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
    oauth: McpOAuthConfigSchema().optional(),
  })
)

// WebSocket 服务器
export const McpWebSocketServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('ws'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
  })
)

// IDE 专用 SSE
export const McpSSEIDEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sse-ide'),
    url: z.string(),
    ideName: z.string(),
    ideRunningInWindows: z.boolean().optional(),
  })
)

// IDE 专用 WebSocket
export const McpWebSocketIDEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('ws-ide'),
    url: z.string(),
    ideName: z.string(),
    authToken: z.string().optional(),
    ideRunningInWindows: z.boolean().optional(),
  })
)

// SDK 控制服务器
export const McpSdkServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sdk'),
    name: z.string(),
  })
)

// Claude.ai 代理服务器
export const McpClaudeAIProxyServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('claudeai-proxy'),
    url: z.string(),
    id: z.string(),
  })
)
```

#### 3.1.4 OAuth 配置
```typescript
// XAA（Cross-App Access）配置
const McpXaaConfigSchema = lazySchema(() => z.boolean())

// OAuth 配置
const McpOAuthConfigSchema = lazySchema(() =>
  z.object({
    clientId: z.string().optional(),
    callbackPort: z.number().int().positive().optional(),
    authServerMetadataUrl: z
      .string()
      .url()
      .startsWith('https://', {
        message: 'authServerMetadataUrl must use https://',
      })
      .optional(),
    xaa: McpXaaConfigSchema().optional(),  // SEP-990 XAA
  })
)
```

#### 3.1.5 统一服务器配置
```typescript
export const McpServerConfigSchema = lazySchema(() =>
  z.union([
    McpStdioServerConfigSchema(),
    McpSSEServerConfigSchema(),
    McpSSEIDEServerConfigSchema(),
    McpWebSocketIDEServerConfigSchema(),
    McpHTTPServerConfigSchema(),
    McpWebSocketServerConfigSchema(),
    McpSdkServerConfigSchema(),
    McpClaudeAIProxyServerConfigSchema(),
  ])
)
export type McpServerConfig = z.infer<ReturnType<typeof McpServerConfigSchema>>
```

### 3.2 带作用域的配置
```typescript
export type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  // 插件来源标识，用于 channel gate
  pluginSource?: string
}
```

### 3.3 JSON 配置结构
```typescript
export const McpJsonConfigSchema = lazySchema(() =>
  z.object({
    mcpServers: z.record(z.string(), McpServerConfigSchema()),
  })
)
export type McpJsonConfig = z.infer<ReturnType<typeof McpJsonConfigSchema>>
```

### 3.4 服务器连接状态
```typescript
// 已连接
export type ConnectedMCPServer = {
  client: Client                    // MCP SDK Client 实例
  name: string
  type: 'connected'
  capabilities: ServerCapabilities  // 服务器能力（工具、资源等）
  serverInfo?: {
    name: string
    version: string
  }
  instructions?: string             // 服务器提供的指令
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>      // 清理函数
}

// 连接失败
export type FailedMCPServer = {
  name: string
  type: 'failed'
  config: ScopedMcpServerConfig
  error?: string
}

// 需要认证
export type NeedsAuthMCPServer = {
  name: string
  type: 'needs-auth'
  config: ScopedMcpServerConfig
}

// 连接中/重连中
export type PendingMCPServer = {
  name: string
  type: 'pending'
  config: ScopedMcpServerConfig
  reconnectAttempt?: number
  maxReconnectAttempts?: number
}

// 已禁用
export type DisabledMCPServer = {
  name: string
  type: 'disabled'
  config: ScopedMcpServerConfig
}

// 联合类型
export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

### 3.5 资源类型
```typescript
// 带服务器标识的资源
export type ServerResource = Resource & { server: string }
```

### 3.6 CLI 状态序列化
```typescript
// 序列化后的工具（用于状态持久化）
export interface SerializedTool {
  name: string
  description: string
  inputJSONSchema?: {
    [x: string]: unknown
    type: 'object'
    properties?: { [x: string]: unknown }
  }
  isMcp?: boolean
  originalToolName?: string  // 原始未规范化的工具名
}

// 序列化后的客户端
export interface SerializedClient {
  name: string
  type: 'connected' | 'failed' | 'needs-auth' | 'pending' | 'disabled'
  capabilities?: ServerCapabilities
}

// 完整的 CLI 状态
export interface MCPCliState {
  clients: SerializedClient[]
  configs: Record<string, ScopedMcpServerConfig>
  tools: SerializedTool[]
  resources: Record<string, ServerResource[]>
  normalizedNames?: Record<string, string>  // 规范化名称到原始名称的映射
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 类型导出

| 类型 | 行号 | 用途 |
|------|------|------|
| `ConfigScope` | 21 | 配置作用域枚举 |
| `Transport` | 26 | 传输类型枚举 |
| `McpServerConfig` | 161 | 统一服务器配置类型 |
| `ScopedMcpServerConfig` | 163-169 | 带作用域的配置 |
| `McpJsonConfig` | 177 | JSON 配置结构 |
| `MCPServerConnection` | 221-227 | 服务器连接状态联合类型 |
| `MCPCliState` | 252-258 | CLI 状态序列化 |

### 4.2 Schema 导出

| Schema | 行号 | 用途 |
|--------|------|------|
| `ConfigScopeSchema` | 10-20 | 作用域验证 |
| `McpServerConfigSchema` | 124-135 | 服务器配置验证 |
| `McpJsonConfigSchema` | 171-175 | JSON 配置验证 |

### 4.3 依赖文件

| 文件 | 用途 |
|------|------|
| `zod/v4` | 运行时类型验证 |
| `../../utils/lazySchema.js` | 延迟 schema 初始化 |
| `@modelcontextprotocol/sdk/client/index.js` | Client 类型 |
| `@modelcontextprotocol/sdk/types.js` | Resource、ServerCapabilities 类型 |

### 4.4 调用方

| 文件 | 用途 |
|------|------|
| `config.ts` | 配置解析和验证 |
| `client.ts` | 服务器连接管理 |
| `utils.ts` | 工具函数 |
| `auth.ts` | OAuth 认证 |
| `headersHelper.ts` | 动态 headers |
| `elicitationHandler.ts` | 引导式配置 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
import type { Client } from '@modelcontextprotocol/sdk/client/index.js'
import type { Resource, ServerCapabilities } from '@modelcontextprotocol/sdk/types.js'
import { z } from 'zod/v4'
import { lazySchema } from '../../utils/lazySchema.js'
```

### 5.2 类型依赖关系

```
McpServerConfig (union)
├── McpStdioServerConfig
├── McpSSEServerConfig ──> McpOAuthConfigSchema?
├── McpHTTPServerConfig ──> McpOAuthConfigSchema?
├── McpWebSocketServerConfig
├── McpSSEIDEServerConfig
├── McpWebSocketIDEServerConfig
├── McpSdkServerConfig
└── McpClaudeAIProxyServerConfig

ScopedMcpServerConfig = McpServerConfig + { scope, pluginSource? }

MCPServerConnection (union)
├── ConnectedMCPServer ──> Client, ServerCapabilities
├── FailedMCPServer
├── NeedsAuthMCPServer
├── PendingMCPServer
└── DisabledMCPServer
```

### 5.3 lazySchema 实现

```typescript
// ../../utils/lazySchema.ts
export function lazySchema<T>(factory: () => z.ZodType<T>): () => z.ZodType<T> {
  let cached: z.ZodType<T> | undefined
  return () => {
    if (!cached) {
      cached = factory()
    }
    return cached
  }
}
```

**作用**：
- 解决 schema 之间的循环依赖
- 延迟初始化，减少启动时间
- 缓存结果，避免重复创建

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 类型安全边界
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| Union 类型穷尽检查 | 新增服务器类型时，switch 语句可能遗漏 | TypeScript 的 `never` 检查 |
| Zod 验证失败 | 运行时数据可能不符合 schema | 所有配置入口都有验证 |
| 类型断言 | 部分地方使用 `as` 类型断言 | 尽量使用类型守卫替代 |

#### 6.1.2 向后兼容性
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 可选 type 字段 | stdio 的 type 是可选的，可能导致歧义 | 默认按 stdio 处理 |
| 新增字段 | 旧版本可能不认识新字段 | Zod 的 `.passthrough()` 保留未知字段 |

### 6.2 潜在问题

#### 6.2.1 Schema 循环依赖
```typescript
// 如果 ASchema 依赖 BSchema，BSchema 又依赖 ASchema
// 会导致初始化时死循环
// 当前通过 lazySchema 解决
```

#### 6.2.2 类型推断性能
```typescript
// 大型 union 类型的推断可能影响编译性能
// 258 行文件，包含 8 个服务器类型的 union
```

### 6.3 改进建议

#### 6.3.1 类型安全增强
1. ** branded types**：使用品牌类型区分不同含义的 string
   ```typescript
   type ServerName = string & { __brand: 'ServerName' }
   ```

2. **更严格的 URL 验证**：
   ```typescript
   url: z.string().url()  // 当前只是 string
   ```

3. **discriminated union 优化**：
   ```typescript
   // 使用 z.discriminatedUnion 替代 z.union
   z.discriminatedUnion('type', [...])
   ```

#### 6.3.2 文档改进
1. **JSDoc 注释**：为复杂类型添加详细注释
2. **示例配置**：每个类型附带示例 JSON
3. **迁移指南**：版本升级时的类型变更说明

#### 6.3.3 代码组织
1. **拆分文件**：按功能拆分为多个类型文件
   ```
   types/
   ├── config.ts      // 配置类型
   ├── connection.ts  // 连接状态类型
   ├── serialization.ts // 序列化类型
   └── index.ts       // 统一导出
   ```

2. **类型测试**：添加类型级别的单元测试
   ```typescript
   // 使用 expect-type 等库
   expectType<McpServerConfig>({ type: 'stdio', command: 'test' })
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| Zod schema 验证边界 | 高 |
| Union 类型穷尽性 | 高 |
| 向后兼容性 | 高 |
| 类型推断性能 | 中 |
