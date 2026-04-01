# mcp.ts 深度研究文档

## 文件元数据
- **路径**: `src/entrypoints/mcp.ts`
- **大小**: 6,270 bytes
- **类型**: TypeScript MCP 服务器入口

---

## 一、场景与职责

### 1.1 核心定位
`mcp.ts` 是 **Claude Code 的 MCP（Model Context Protocol）服务器实现**，承担以下关键职责：

1. **MCP 协议实现**: 实现 MCP 服务器规范，暴露 Claude Code 工具给 MCP 客户端
2. **工具暴露**: 将内部工具（如 `review` 命令）通过 MCP 协议暴露
3. **无头模式支持**: 支持非交互式工具调用（`isNonInteractiveSession: true`）
4. **文件状态缓存**: 使用 LRU 缓存管理文件状态，防止内存无限增长

### 1.2 使用场景

| 场景 | 说明 |
|------|------|
| **IDE 集成** | VSCode、JetBrains 等 IDE 通过 MCP 协议调用 Claude Code 工具 |
| **自动化脚本** | 外部脚本通过 MCP 调用 Claude Code 功能 |
| **工具链集成** | 与其他支持 MCP 的工具集成 |
| **无头执行** | 无需交互式 TUI 即可执行工具 |

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP 客户端                               │
│         (IDE, 脚本, 其他 MCP 兼容工具)                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ MCP 协议 (stdio)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              src/entrypoints/mcp.ts                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  1. 创建 MCP Server 实例                            │   │
│  │  2. 设置 ListTools 处理程序                         │   │
│  │  3. 设置 CallTool 处理程序                          │   │
│  │  4. 连接到 stdio 传输层                             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Claude Code 工具系统                           │
│         (Tool.ts, tools.ts, commands/)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、功能点目的

### 2.1 MCP 服务器启动

```typescript
export async function startMCPServer(
  cwd: string,
  debug: boolean,
  verbose: boolean,
): Promise<void> {
  // 创建带大小限制的 LRU 缓存
  const READ_FILE_STATE_CACHE_SIZE = 100
  const readFileStateCache = createFileStateCacheWithSizeLimit(
    READ_FILE_STATE_CACHE_SIZE,
  )
  setCwd(cwd)
  
  // 创建 MCP 服务器实例
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } },
  )
  // ... 设置处理程序
}
```

**关键参数**:
- `cwd`: 工作目录
- `debug`: 调试模式
- `verbose`: 详细模式

### 2.2 工具列表处理程序

```typescript
server.setRequestHandler(
  ListToolsRequestSchema,
  async (): Promise<ListToolsResult> => {
    const toolPermissionContext = getEmptyToolPermissionContext()
    const tools = getTools(toolPermissionContext)
    return {
      tools: await Promise.all(
        tools.map(async tool => {
          let outputSchema: ToolOutput | undefined
          if (tool.outputSchema) {
            const convertedSchema = zodToJsonSchema(tool.outputSchema)
            // MCP SDK 要求 outputSchema 根级别有 type: "object"
            // 跳过根级别有 anyOf/oneOf 的 schema
            if (
              typeof convertedSchema === 'object' &&
              convertedSchema !== null &&
              'type' in convertedSchema &&
              convertedSchema.type === 'object'
            ) {
              outputSchema = convertedSchema as ToolOutput
            }
          }
          return {
            ...tool,
            description: await tool.prompt({...}),
            inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
            outputSchema,
          }
        }),
      ),
    }
  },
)
```

**技术要点**:
- 使用 `getEmptyToolPermissionContext()` 获取空权限上下文
- 调用 `getTools()` 获取所有可用工具
- 将 Zod schema 转换为 JSON Schema
- 处理 outputSchema 限制（根级别必须是 `type: "object"`）

### 2.3 工具调用处理程序

```typescript
server.setRequestHandler(
  CallToolRequestSchema,
  async ({ params: { name, arguments: args } }): Promise<CallToolResult> => {
    // 查找工具
    const tool = findToolByName(tools, name)
    if (!tool) throw new Error(`Tool ${name} not found`)
    
    // 构建工具使用上下文
    const toolUseContext: ToolUseContext = {
      abortController: createAbortController(),
      options: {
        commands: MCP_COMMANDS,
        tools,
        mainLoopModel: getMainLoopModel(),
        thinkingConfig: { type: 'disabled' },
        mcpClients: [],
        mcpResources: {},
        isNonInteractiveSession: true,
        debug,
        verbose,
        agentDefinitions: { activeAgents: [], allAgents: [] },
      },
      getAppState: () => getDefaultAppState(),
      setAppState: () => {},
      messages: [],
      readFileState: readFileStateCache,
      setInProgressToolUseIDs: () => {},
      setResponseLength: () => {},
      updateFileHistoryState: () => {},
      updateAttributionState: () => {},
    }
    
    // 验证和调用
    if (!tool.isEnabled()) throw new Error(`Tool ${name} is not enabled`)
    const validationResult = await tool.validateInput?.(...)
    const finalResult = await tool.call(...)
    
    return { content: [{ type: 'text', text: ... }] }
  },
)
```

### 2.4 工具使用上下文配置

```typescript
const toolUseContext: ToolUseContext = {
  abortController: createAbortController(),
  options: {
    commands: MCP_COMMANDS,              // 仅包含 review 命令
    tools,                               // 所有可用工具
    mainLoopModel: getMainLoopModel(),   // 主循环模型
    thinkingConfig: { type: 'disabled' },// 禁用思考
    mcpClients: [],                      // 无 MCP 客户端
    mcpResources: {},                    // 无 MCP 资源
    isNonInteractiveSession: true,       // 非交互模式
    debug,
    verbose,
    agentDefinitions: { activeAgents: [], allAgents: [] },
  },
  // ... 状态管理函数（大部分为空实现）
}
```

---

## 三、具体技术实现

### 3.1 文件状态缓存

```typescript
const READ_FILE_STATE_CACHE_SIZE = 100
const readFileStateCache = createFileStateCacheWithSizeLimit(
  READ_FILE_STATE_CACHE_SIZE,
)
```

**设计决策**:
- 限制 100 个文件和 25MB 内存
- 防止 MCP 服务器操作中的无界内存增长
- 使用 LRU（最近最少使用）淘汰策略

### 3.2 Schema 转换处理

**Output Schema 限制处理**:
```typescript
if (tool.outputSchema) {
  const convertedSchema = zodToJsonSchema(tool.outputSchema)
  // MCP SDK 要求 outputSchema 根级别有 type: "object"
  // 跳过根级别有 anyOf/oneOf 的 schema（来自 z.union, z.discriminatedUnion 等）
  if (
    typeof convertedSchema === 'object' &&
    convertedSchema !== null &&
    'type' in convertedSchema &&
    convertedSchema.type === 'object'
  ) {
    outputSchema = convertedSchema as ToolOutput
  }
}
```

**参考**: GitHub Issue #8014

### 3.3 错误处理

```typescript
try {
  // ... 工具调用
} catch (error) {
  logError(error)
  
  const parts = error instanceof Error ? getErrorParts(error) : [String(error)]
  const errorText = parts.filter(Boolean).join('\n').trim() || 'Error'
  
  return {
    isError: true,
    content: [{ type: 'text', text: errorText }],
  }
}
```

**特点**:
- 使用 `logError` 记录错误
- 使用 `getErrorParts` 提取错误信息
- 返回 MCP 格式的错误响应

### 3.4 MCP 命令白名单

```typescript
const MCP_COMMANDS: Command[] = [review]
```

当前仅暴露 `review` 命令给 MCP 客户端。

---

## 四、关键代码路径与文件引用

### 4.1 导入的模块

| 模块路径 | 用途 |
|----------|------|
| `@modelcontextprotocol/sdk/server/index.js` | MCP Server 类 |
| `@modelcontextprotocol/sdk/server/stdio.js` | StdioServerTransport |
| `@modelcontextprotocol/sdk/types.js` | MCP 协议类型 |
| `src/state/AppStateStore.js` | 应用状态管理 |
| `../commands/review.js` | review 命令 |
| `../commands.js` | 命令类型 |
| `../Tool.js` | 工具查找和权限上下文 |
| `../tools.js` | 工具获取 |
| `../utils/abortController.js` | 中止控制器 |
| `../utils/fileStateCache.js` | 文件状态缓存 |
| `../utils/log.js` | 日志记录 |
| `../utils/messages.js` | 消息创建 |
| `../utils/model/model.js` | 模型获取 |
| `../utils/permissions/permissions.js` | 权限检查 |
| `../utils/Shell.js` | 工作目录设置 |
| `../utils/slowOperations.js` | JSON 序列化 |
| `../utils/toolErrors.js` | 错误处理 |
| `../utils/zodToJsonSchema.js` | Zod 到 JSON Schema 转换 |

### 4.2 MCP 协议类型

| 类型 | 来源 | 用途 |
|------|------|------|
| `Server` | `@modelcontextprotocol/sdk/server` | MCP 服务器实例 |
| `StdioServerTransport` | `@modelcontextprotocol/sdk/server/stdio` | stdio 传输层 |
| `CallToolRequestSchema` | `@modelcontextprotocol/sdk/types` | 工具调用请求模式 |
| `CallToolResult` | `@modelcontextprotocol/sdk/types` | 工具调用结果 |
| `ListToolsRequestSchema` | `@modelcontextprotocol/sdk/types` | 工具列表请求模式 |
| `ListToolsResult` | `@modelcontextprotocol/sdk/types` | 工具列表结果 |
| `Tool` | `@modelcontextprotocol/sdk/types` | 工具定义 |

### 4.3 依赖关系图

```
mcp.ts
├── MCP SDK
│   ├── Server (MCP 服务器)
│   ├── StdioServerTransport (stdio 传输)
│   └── 协议类型 (CallToolResult, ListToolsResult, etc.)
├── Claude Code 核心
│   ├── commands/review.js (review 命令)
│   ├── Tool.js (工具查找、权限上下文)
│   ├── tools.js (工具获取)
│   └── state/AppStateStore.js (应用状态)
├── 工具支持
│   ├── abortController.js (取消支持)
│   ├── fileStateCache.js (文件缓存)
│   ├── messages.js (消息创建)
│   ├── model/model.js (模型配置)
│   ├── permissions/permissions.js (权限)
│   ├── Shell.js (工作目录)
│   ├── slowOperations.js (序列化)
│   ├── toolErrors.js (错误处理)
│   └── zodToJsonSchema.js (schema 转换)
└── 被调用方
    └── main.tsx (mcp serve 命令)
```

---

## 五、依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk` | MCP 协议实现 |

### 5.2 内部服务交互

```
mcp.ts
├── 工具系统
│   ├── getTools() - 获取所有工具
│   ├── findToolByName() - 按名称查找工具
│   └── tool.call() - 执行工具
├── 权限系统
│   ├── getEmptyToolPermissionContext() - 空权限上下文
│   └── hasPermissionsToUseTool() - 权限检查
├── 状态管理
│   ├── getDefaultAppState() - 默认应用状态
│   └── setCwd() - 设置工作目录
└── 模型系统
    └── getMainLoopModel() - 获取主循环模型
```

### 5.3 调用入口

```typescript
// main.tsx 中的 mcp serve 命令处理
program
  .command('mcp')
  .addCommand(
    new Command('serve')
      .description('Start the MCP server')
      .action(async () => {
        await startMCPServer(getCwd(), isDebugMode(), isVerboseMode())
      })
  )
```

---

## 六、风险、边界与改进建议

### 6.1 当前风险

| 风险点 | 严重程度 | 说明 |
|--------|----------|------|
| **Schema 转换限制** | 中 | 某些 Zod schema（union、discriminatedUnion）无法转换为 MCP outputSchema |
| **空状态管理** | 中 | 大部分状态管理函数为空实现，可能影响工具行为 |
| **有限命令暴露** | 低 | 仅暴露 review 命令，功能受限 |
| **无 MCP 工具暴露** | 低 | TODO 注释表明计划暴露 MCP 工具但未实现 |
| **输入验证** | 低 | TODO 注释表明需要添加 Zod 输入验证 |

### 6.2 边界条件

1. **Output Schema 限制**
   - MCP SDK 要求根级别 `type: "object"`
   - 跳过 `anyOf`/`oneOf` 根级别的 schema
   - 可能导致某些工具输出 schema 不完整

2. **非交互模式**
   - `isNonInteractiveSession: true`
   - 某些依赖交互的工具可能无法正常工作

3. **文件状态缓存**
   - 100 文件 / 25MB 限制
   - 超出限制时 LRU 淘汰

4. **工具启用检查**
   - 调用前检查 `tool.isEnabled()`
   - 禁用的工具返回错误

5. **输入验证**
   - 可选的 `tool.validateInput` 调用
   - 验证失败返回错误消息

### 6.3 改进建议

1. **Schema 转换增强**
   ```typescript
   // 建议：处理更多 Zod schema 类型
   // 考虑使用更灵活的 schema 转换策略
   // 或包装 union 类型到 object 中
   ```

2. **状态管理完善**
   - 实现关键状态管理函数
   - 或明确文档说明哪些工具在 MCP 模式下受限

3. **命令暴露扩展**
   - 评估暴露更多命令的安全性和实用性
   - 添加配置选项控制暴露的命令

4. **输入验证**
   ```typescript
   // 实现 TODO 中提到的 Zod 输入验证
   const parsed = tool.inputSchema.safeParse(args)
   if (!parsed.success) {
     throw new Error(`Invalid input: ${parsed.error.message}`)
   }
   ```

5. **MCP 工具暴露**
   ```typescript
   // 实现 TODO 中提到的 MCP 工具暴露
   // TODO: Also re-expose any MCP tools
   ```

6. **错误处理增强**
   - 添加更详细的错误分类
   - 提供错误恢复建议

7. **性能监控**
   - 添加工具调用耗时记录
   - 监控缓存命中率

### 6.4 相关配置

| 配置项 | 位置 | 说明 |
|--------|------|------|
| `READ_FILE_STATE_CACHE_SIZE` | 硬编码 | 文件状态缓存大小（100） |
| `MCP_COMMANDS` | 硬编码 | 暴露的命令列表 |
| `isNonInteractiveSession` | 硬编码 | 始终为 true |
| `thinkingConfig` | 硬编码 | 禁用思考 |

---

## 七、总结

`mcp.ts` 是 Claude Code 的 **MCP 协议适配器**，设计哲学是：

1. **协议兼容**: 实现标准 MCP 协议，兼容任何 MCP 客户端
2. **轻量级**: 仅暴露核心工具功能，避免复杂状态管理
3. **安全**: 使用空权限上下文，依赖工具自身的权限检查
4. **性能**: 使用 LRU 缓存防止内存泄漏

当前实现是**功能有限的 MVP**，主要限制包括：
- 仅暴露 review 命令
- 某些 schema 类型转换受限
- 状态管理函数为空实现

未来发展方向：
- 暴露更多工具和功能
- 完善 schema 转换
- 实现完整的状态管理
- 支持双向 MCP 工具暴露
