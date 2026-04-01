# toolExecution.ts 深度研究文档

## 场景与职责

`toolExecution.ts` 是 Claude Code 工具执行层的**核心实现文件**，负责单个工具的完整生命周期管理。它是 `StreamingToolExecutor` 的底层支撑，直接处理：

1. **工具查找与输入验证**：解析工具名称、验证输入参数
2. **权限检查与决策**：协调 hooks、规则、分类器的权限判定
3. **工具实际调用**：执行工具的 `call` 方法并处理结果
4. **错误处理与遥测**：分类错误、记录日志、发送分析事件
5. **MCP 工具支持**：处理外部 MCP 服务器的工具调用

### 核心职责
- `runToolUse`: 工具执行的入口函数，返回 AsyncGenerator 支持流式结果
- `checkPermissionsAndCallTool`: 权限检查与实际调用的协调
- 错误分类与遥测安全处理
- MCP 服务器连接管理与工具路由

---

## 功能点目的

### 1. 工具查找与别名支持

支持通过别名查找已弃用工具（向后兼容）：

```typescript
// 先在可用工具中查找
let tool = findToolByName(toolUseContext.options.tools, toolName)

// 如果未找到，检查是否是别名（如 "KillShell" → "TaskStop"）
if (!tool) {
  const fallbackTool = findToolByName(getAllBaseTools(), toolName)
  if (fallbackTool && fallbackTool.aliases?.includes(toolName)) {
    tool = fallbackTool
  }
}
```

### 2. 输入验证双阶段

```typescript
// 阶段 1: Zod schema 验证
const parsedInput = tool.inputSchema.safeParse(input)

// 阶段 2: 工具自定义验证（如路径存在性检查）
const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext)
```

### 3. 权限决策链

```
PreToolUse Hooks → Hook Permission Result → Rule Check → canUseTool → Decision
                      ↓
              如果 hook 返回 allow，仍需检查 deny/ask 规则
              如果 hook 返回 ask/deny，按对应逻辑处理
```

### 4. 延迟工具（Deferred Tools）支持

当工具通过 ToolSearch 动态加载时，可能出现 schema 未发送的情况：

```typescript
export function buildSchemaNotSentHint(...): string | null {
  if (!isDeferredTool(tool)) return null
  const discovered = extractDiscoveredToolNames(messages)
  if (discovered.has(tool.name)) return null
  return `
    This tool's schema was not sent to the API — it was not in the discovered-tool set.
    Load the tool first: call ${TOOL_SEARCH_TOOL_NAME} with query "select:${tool.name}"
  `
}
```

### 5. Bash 分类器预检

在权限检查阶段并行启动 Bash 分类器检查：

```typescript
if (tool.name === BASH_TOOL_NAME && 'command' in parsedInput.data) {
  startSpeculativeClassifierCheck(
    command,
    appState.toolPermissionContext,
    signal,
    isNonInteractiveSession,
  )
}
```

---

## 具体技术实现

### 关键流程

#### 1. 主执行流程 (`runToolUse`)

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void> {
  // 1. 工具查找（含别名回退）
  let tool = findToolByName(toolUseContext.options.tools, toolName)
  if (!tool) { /* 别名回退逻辑 */ }
  
  // 2. 检查 abort 状态
  if (toolUseContext.abortController.signal.aborted) {
    yield { message: createUserMessage({ content: [createToolResultStopMessage(...)] }) }
    return
  }
  
  // 3. 委托给流式权限检查与调用
  for await (const update of streamedCheckPermissionsAndCallTool(...)) {
    yield update
  }
}
```

#### 2. 流式封装 (`streamedCheckPermissionsAndCallTool`)

使用 `Stream` 类将异步回调转换为 AsyncIterable：

```typescript
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  
  checkPermissionsAndCallTool(..., progress => {
    // 进度事件入队
    stream.enqueue({ message: createProgressMessage({...}) })
  })
  .then(results => {
    // 最终结果入队
    for (const result of results) stream.enqueue(result)
  })
  .catch(error => stream.error(error))
  .finally(() => stream.done())
  
  return stream
}
```

#### 3. 权限检查与调用核心 (`checkPermissionsAndCallTool`)

```typescript
async function checkPermissionsAndCallTool(...): Promise<MessageUpdateLazy[]> {
  // 1. Zod 输入验证
  const parsedInput = tool.inputSchema.safeParse(input)
  if (!parsedInput.success) {
    return [{
      message: createUserMessage({
        content: [{
          type: 'tool_result',
          content: `<tool_use_error>InputValidationError: ${errorContent}</tool_use_error>`,
          is_error: true,
          tool_use_id: toolUseID,
        }]
      })
    }]
  }
  
  // 2. 工具自定义验证
  const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext)
  if (isValidCall?.result === false) { /* 返回验证错误 */ }
  
  // 3. 启动 Bash 分类器预检（并行）
  if (tool.name === BASH_TOOL_NAME) {
    startSpeculativeClassifierCheck(command, ...)
  }
  
  // 4. 防御性过滤内部字段
  if (tool.name === BASH_TOOL_NAME && '_simulatedSedEdit' in processedInput) {
    const { _simulatedSedEdit: _, ...rest } = processedInput
    processedInput = rest
  }
  
  // 5. 回填可观察输入（用于 hooks/权限检查）
  if (tool.backfillObservableInput) {
    const backfilledClone = { ...processedInput }
    tool.backfillObservableInput(backfilledClone)
    processedInput = backfilledClone
  }
  
  // 6. 执行 PreToolUse Hooks
  for await (const result of runPreToolUseHooks(...)) {
    // 处理 message/hookPermissionResult/hookUpdatedInput/preventContinuation/stop/additionalContext
  }
  
  // 7. 解析权限决策
  const resolved = await resolveHookPermissionDecision(...)
  const permissionDecision = resolved.decision
  processedInput = resolved.input
  
  // 8. 权限被拒绝处理
  if (permissionDecision.behavior !== 'allow') {
    // 记录决策、运行 PermissionDenied hooks、返回错误消息
  }
  
  // 9. 工具实际调用
  const result = await tool.call(callInput, context, canUseTool, assistantMessage, onProgress)
  
  // 10. 执行 PostToolUse Hooks
  for await (const hookResult of runPostToolUseHooks(...)) {
    // 处理 hook 结果
  }
  
  // 11. 构建并返回结果消息
  await addToolResult(toolOutput, preMappedBlock)
}
```

#### 4. 错误分类 (`classifyToolError`)

```typescript
export function classifyToolError(error: unknown): string {
  // 1. TelemetrySafeError: 使用预审核的遥测消息
  if (error instanceof TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS) {
    return error.telemetryMessage.slice(0, 200)
  }
  
  // 2. Node.js fs 错误: 使用错误代码 (ENOENT, EACCES 等)
  if (error instanceof Error) {
    const errnoCode = getErrnoCode(error)
    if (typeof errnoCode === 'string') return `Error:${errnoCode}`
    
    // 3. 已知错误类型: 使用稳定的 name 属性
    if (error.name && error.name !== 'Error' && error.name.length > 3) {
      return error.name.slice(0, 60)
    }
  }
  
  return 'Error'  // 避免返回压缩后的 3 字符标识符
}
```

### 数据结构

#### MessageUpdateLazy

```typescript
export type MessageUpdateLazy<M extends Message = Message> = {
  message: M
  contextModifier?: {
    toolUseID: string
    modifyContext: (context: ToolUseContext) => ToolUseContext
  }
}
```

#### MCP Server 类型

```typescript
export type McpServerType =
  | 'stdio'
  | 'sse'
  | 'http'
  | 'ws'
  | 'sdk'
  | 'sse-ide'
  | 'ws-ide'
  | 'claudeai-proxy'
  | undefined
```

#### OTel Source 映射

```typescript
// 将权限决策原因映射到 OTel 的 source 标签
function decisionReasonToOTelSource(reason: PermissionDecisionReason, behavior: 'allow' | 'deny'): string {
  switch (reason.type) {
    case 'permissionPromptTool':
      // 处理 decisionClassification (user_temporary/user_permanent/user_reject)
    case 'rule':
      return ruleSourceToOTelSource(reason.rule.source, behavior)
    case 'hook':
      return 'hook'
    default:
      return 'config'
  }
}
```

---

## 关键代码路径与文件引用

### 核心依赖

| 文件路径 | 用途 |
|---------|------|
| `src/services/tools/toolHooks.ts` | `runPreToolUseHooks`, `runPostToolUseHooks`, `resolveHookPermissionDecision` |
| `src/Tool.ts` | `Tool`, `ToolUseContext`, `findToolByName`, `ToolResult` |
| `src/hooks/useCanUseTool.tsx` | `CanUseToolFn` 类型定义 |
| `src/utils/permissions/permissions.ts` | `checkRuleBasedPermissions` |
| `src/utils/permissions/PermissionResult.ts` | `PermissionResult`, `PermissionDecision` 类型 |
| `src/utils/hooks.ts` | `executePermissionDeniedHooks` |
| `src/utils/telemetry/sessionTracing.ts` | `startToolSpan`, `endToolSpan`, `addToolContentEvent` |
| `src/services/mcp/utils.ts` | `isMcpTool`, `getMcpServerScopeFromToolName` |
| `src/services/mcp/client.ts` | `McpAuthError`, `McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` |

### 工具相关常量

| 常量 | 来源文件 |
|-----|---------|
| `BASH_TOOL_NAME` | `src/tools/BashTool/toolName.ts` |
| `FILE_EDIT_TOOL_NAME` | `src/tools/FileEditTool/constants.ts` |
| `FILE_READ_TOOL_NAME` | `src/tools/FileReadTool/prompt.ts` |
| `FILE_WRITE_TOOL_NAME` | `src/tools/FileWriteTool/prompt.ts` |
| `NOTEBOOK_EDIT_TOOL_NAME` | `src/tools/NotebookEditTool/constants.ts` |
| `POWERSHELL_TOOL_NAME` | `src/tools/PowerShellTool/toolName.ts` |
| `TOOL_SEARCH_TOOL_NAME` | `src/tools/ToolSearchTool/prompt.ts` |

### 调用关系

```
runToolUse
    ├── findToolByName (Tool.ts)
    ├── logEvent (analytics)
    └── streamedCheckPermissionsAndCallTool
            └── checkPermissionsAndCallTool
                    ├── tool.inputSchema.safeParse
                    ├── tool.validateInput?
                    ├── startSpeculativeClassifierCheck (bashPermissions.ts)
                    ├── runPreToolUseHooks (toolHooks.ts)
                    ├── resolveHookPermissionDecision (toolHooks.ts)
                    ├── canUseTool (useCanUseTool.tsx)
                    ├── tool.call
                    ├── runPostToolUseHooks (toolHooks.ts)
                    └── addToolResult
                            ├── processPreMappedToolResultBlock / processToolResultBlock
                            └── createUserMessage (messages.ts)
```

---

## 依赖与外部交互

### 1. 与 MCP 系统的交互

```typescript
// 查找 MCP 服务器连接
function findMcpServerConnection(toolName: string, mcpClients: MCPServerConnection[]): MCPServerConnection | undefined {
  if (!toolName.startsWith('mcp__')) return undefined
  const mcpInfo = mcpInfoFromString(toolName)
  return mcpClients.find(client => normalizeNameForMCP(client.name) === mcpInfo.serverName)
}

// 获取 MCP 服务器类型用于遥测
function getMcpServerType(toolName: string, mcpClients: MCPServerConnection[]): McpServerType {
  const serverConnection = findMcpServerConnection(toolName, mcpClients)
  if (serverConnection?.type === 'connected') {
    return serverConnection.config.type ?? 'stdio'
  }
  return undefined
}
```

### 2. 与遥测系统的交互

```typescript
// 工具决策事件
void logOTelEvent('tool_decision', {
  decision: 'accept' | 'reject',
  source: 'config' | 'hook' | 'user_permanent' | 'user_temporary' | 'user_reject',
  tool_name: sanitizeToolNameForAnalytics(tool.name),
})

// 工具结果事件
void logOTelEvent('tool_result', {
  tool_name: sanitizeToolNameForAnalytics(tool.name),
  success: 'true' | 'false',
  duration_ms: String(durationMs),
  tool_parameters: jsonStringify(toolParameters),
  decision_source: decisionInfo?.source,
  decision_type: decisionInfo?.decision,
  mcp_server_scope: mcpServerScope,
})
```

### 3. 与权限系统的交互

通过 `canUseTool` 回调进行交互式权限检查：

```typescript
const resolved = await resolveHookPermissionDecision(
  hookPermissionResult,
  tool,
  processedInput,
  toolUseContext,
  canUseTool,  // ← 来自 useCanUseTool hook
  assistantMessage,
  toolUseID,
)
```

### 4. 与 Analytics 的交互

```typescript
logEvent('tengu_tool_use_success', {
  messageID: messageId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  toolName: sanitizeToolNameForAnalytics(tool.name),
  isMcp: tool.isMcp ?? false,
  durationMs,
  preToolHookDurationMs,
  toolResultSizeBytes,
  fileExtension,
  queryChainId: toolUseContext.queryTracking?.chainId,
  queryDepth: toolUseContext.queryTracking?.depth,
  mcpServerType,
  mcpServerBaseUrl,
  requestId,
})
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 输入回填的复杂性

```typescript
// 回填逻辑涉及多个变量：processedInput, backfilledClone, callInput
let callInput = processedInput
const backfilledClone = tool.backfillObservableInput
  ? ({ ...processedInput } as typeof processedInput)
  : null
if (backfilledClone) {
  tool.backfillObservableInput!(backfilledClone as Record<string, unknown>)
  processedInput = backfilledClone
}

// 后续还有复杂的收敛逻辑...
if (backfilledClone && processedInput !== callInput && ...) {
  callInput = { ...processedInput, file_path: (callInput as Record<string, unknown>).file_path }
}
```

**风险**：路径处理逻辑复杂，容易引入回归（如注释中提到的 #21056）。

#### 2. MCP 工具的特殊处理

```typescript
// TODO(hackyon): refactor so we don't have different experiences for MCP tools
if (!isMcpTool(tool)) {
  await addToolResult(toolOutput, mappedToolResultBlock)
}
// ... PostToolUse hooks ...
if (isMcpTool(tool)) {
  await addToolResult(toolOutput)
}
```

**风险**：MCP 工具与非 MCP 工具的处理路径不一致，可能导致行为差异。

#### 3. 遥测数据敏感性

```typescript
// 工具参数仅在 OTEL_LOG_TOOL_DETAILS 启用时记录
if (isToolDetailsLoggingEnabled()) {
  if (tool.name === BASH_TOOL_NAME && 'command' in processedInput) {
    toolParameters = {
      bash_command: commandParts[0],
      full_command: bashInput.command,
      // ...
    }
  }
}
```

**风险**：敏感信息（如 bash 命令）的日志记录需要严格控制。

### 边界情况

| 场景 | 处理逻辑 |
|-----|---------|
| 工具不存在 | 记录错误事件，返回 "No such tool available" |
| 输入验证失败 | 返回 InputValidationError，附带 schema hint（延迟工具场景） |
| Zod 解析成功但工具验证失败 | 记录具体错误信息，返回工具自定义错误 |
| 权限被拒绝 | 运行 PermissionDenied hooks，支持 retry 逻辑 |
| 用户中断 | 生成取消消息，运行 PostToolUseFailure hooks |
| MCP 认证错误 | 更新客户端状态为 'needs-auth' |
| 工具执行抛出异常 | 分类错误，记录遥测，运行失败 hooks |

### 改进建议

1. **简化输入处理逻辑**
   - 将 `processedInput` / `backfilledClone` / `callInput` 的复杂交互封装为专用函数
   - 添加单元测试覆盖各种输入变换场景

2. **统一 MCP 与非 MCP 工具路径**
   - 消除 `isMcpTool` 的条件分支
   - 通过抽象层统一处理工具结果映射

3. **增强错误分类**
   - 扩展 `classifyToolError` 支持更多已知错误类型
   - 为 MCP 工具添加特定的错误分类

4. **优化遥测性能**
   - `jsonStringify` 是同步慢操作，考虑异步化
   - 减少重复的属性访问（如多次访问 `toolUseContext.queryTracking`）

5. **代码分割**
   - 文件超过 1700 行，考虑按功能拆分为：
     - `toolExecutionCore.ts` - 核心执行逻辑
     - `toolExecutionMcp.ts` - MCP 相关逻辑
     - `toolExecutionTelemetry.ts` - 遥测相关逻辑

6. **类型安全改进**
   - `decisionReasonToOTelSource` 中的 `toolResult` 使用 `unknown` 类型，应该使用更精确的类型
   - `processedInput` 的多次类型断言应该通过泛型约束消除
