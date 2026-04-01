# toolHooks.ts 深度研究文档

## 场景与职责

`toolHooks.ts` 是 Claude Code 工具执行流程中**钩子系统**的协调层，负责在工具执行前后运行用户自定义的钩子（hooks）。它是连接工具执行引擎与钩子基础设施的关键桥梁。

### 核心场景

1. **PreToolUse Hooks**：工具执行前运行，可用于：
   - 修改工具输入参数
   - 做出权限决策（允许/拒绝/询问）
   - 添加额外上下文信息
   - 阻止工具执行继续

2. **PostToolUse Hooks**：工具成功执行后运行，可用于：
   - 修改工具输出结果
   - 添加额外上下文
   - 执行后续操作

3. **PostToolUseFailure Hooks**：工具执行失败后运行，可用于：
   - 错误处理与恢复
   - 记录失败信息
   - 提供替代建议

### 核心职责
- `runPreToolUseHooks`: 协调 PreToolUse 钩子的执行与结果聚合
- `runPostToolUseHooks`: 协调 PostToolUse 钩子的执行
- `runPostToolUseFailureHooks`: 协调失败场景下的钩子执行
- `resolveHookPermissionDecision`: 将钩子的权限结果解析为最终决策

---

## 功能点目的

### 1. 钩子权限决策解析

核心函数 `resolveHookPermissionDecision` 实现了复杂的权限决策逻辑：

```
Hook 'allow' → 检查规则覆盖 → 如果需要交互 → 调用 canUseTool
                      ↓
              deny/ask 规则存在 → 按规则处理
                      ↓
              无冲突规则 → 允许执行

Hook 'deny' → 直接拒绝

Hook 'ask' / 无决策 → 正常权限流程（可携带 forceDecision）
```

**关键设计原则**：
- Hook 的 `allow` **不**绕过 settings.json 的 deny/ask 规则（安全优先）
- 如果工具需要用户交互且 hook 未提供 `updatedInput`，仍需调用 `canUseTool`
- Hook 可以通过 `updatedInput` 满足交互需求（如 headless wrapper 收集用户回答）

### 2. 钩子结果类型系统

```typescript
export type PostToolUseHooksResult<Output> =
  | MessageUpdateLazy<AttachmentMessage | ProgressMessage<HookProgress>>
  | { updatedMCPToolOutput: Output }

// PreToolUse 钩子产出更丰富的结果类型
type PreToolUseHookResult =
  | { type: 'message'; message: MessageUpdateLazy<...> }
  | { type: 'hookPermissionResult'; hookPermissionResult: PermissionResult }
  | { type: 'hookUpdatedInput'; updatedInput: Record<string, unknown> }
  | { type: 'preventContinuation'; shouldPreventContinuation: boolean }
  | { type: 'stopReason'; stopReason: string }
  | { type: 'additionalContext'; message: MessageUpdateLazy<AttachmentMessage> }
  | { type: 'stop' }  // 终止执行
```

### 3. MCP 工具输出修改

PostToolUse hooks 可以修改 MCP 工具的输出：

```typescript
if ('updatedMCPToolOutput' in hookResult) {
  if (isMcpTool(tool)) {
    toolOutput = hookResult.updatedMCPToolOutput
  }
}
```

这允许 hooks 对 MCP 服务器返回的结果进行后处理。

### 4. 错误处理与遥测

每个钩子执行路径都有完整的错误捕获和遥测：

```typescript
try {
  for await (const result of executePostToolHooks(...)) {
    // 处理结果
  }
} catch (error) {
  logEvent('tengu_post_tool_hook_error', {
    toolName: sanitizeToolNameForAnalytics(tool.name),
    isMcp: tool.isMcp ?? false,
    duration: postToolDurationMs,
    queryChainId: toolUseContext.queryTracking?.chainId,
    queryDepth: toolUseContext.queryTracking?.depth,
  })
  yield {
    message: createAttachmentMessage({
      type: 'hook_error_during_execution',
      content: formatError(error),
      hookName: `PostToolUse:${tool.name}`,
      ...
    })
  }
}
```

---

## 具体技术实现

### 关键流程

#### 1. PreToolUse Hooks 执行流程

```typescript
export async function* runPreToolUseHooks(
  toolUseContext: ToolUseContext,
  tool: Tool,
  processedInput: Record<string, unknown>,
  toolUseID: string,
  messageId: string,
  requestId: string | undefined,
  mcpServerType: McpServerType,
  mcpServerBaseUrl: string | undefined,
): AsyncGenerator<PreToolUseHookResult> {
  const hookStartTime = Date.now()
  
  try {
    const appState = toolUseContext.getAppState()
    
    for await (const result of executePreToolHooks(
      tool.name,
      toolUseID,
      processedInput,
      toolUseContext,
      appState.toolPermissionContext.mode,
      toolUseContext.abortController.signal,
      undefined,  // timeoutMs - use default
      toolUseContext.requestPrompt,
      tool.getToolUseSummary?.(processedInput),
    )) {
      try {
        // 处理 message
        if (result.message) {
          yield { type: 'message', message: { message: result.message } }
        }
        
        // 处理 blockingError → 转换为 deny 决策
        if (result.blockingError) {
          const denialMessage = getPreToolHookBlockingMessage(...)
          yield {
            type: 'hookPermissionResult',
            hookPermissionResult: {
              behavior: 'deny',
              message: denialMessage,
              decisionReason: { type: 'hook', hookName: `PreToolUse:${tool.name}`, reason: denialMessage }
            }
          }
        }
        
        // 处理 preventContinuation
        if (result.preventContinuation) {
          yield { type: 'preventContinuation', shouldPreventContinuation: true }
          if (result.stopReason) {
            yield { type: 'stopReason', stopReason: result.stopReason }
          }
        }
        
        // 处理 permissionBehavior
        if (result.permissionBehavior !== undefined) {
          const decisionReason: PermissionDecisionReason = {
            type: 'hook',
            hookName: `PreToolUse:${tool.name}`,
            hookSource: result.hookSource,
            reason: result.hookPermissionDecisionReason,
          }
          
          if (result.permissionBehavior === 'allow') {
            yield {
              type: 'hookPermissionResult',
              hookPermissionResult: { behavior: 'allow', updatedInput: result.updatedInput, decisionReason }
            }
          } else if (result.permissionBehavior === 'ask') {
            yield {
              type: 'hookPermissionResult',
              hookPermissionResult: {
                behavior: 'ask',
                updatedInput: result.updatedInput,
                message: result.hookPermissionDecisionReason || `Hook PreToolUse:${tool.name} asked for this tool`,
                decisionReason
              }
            }
          } else {
            // deny
            yield {
              type: 'hookPermissionResult',
              hookPermissionResult: {
                behavior: 'deny',
                message: result.hookPermissionDecisionReason || `Hook PreToolUse:${tool.name} denied this tool`,
                decisionReason
              }
            }
          }
        }
        
        // 处理 passthrough updatedInput
        if (result.updatedInput && result.permissionBehavior === undefined) {
          yield { type: 'hookUpdatedInput', updatedInput: result.updatedInput }
        }
        
        // 处理 additionalContexts
        if (result.additionalContexts?.length > 0) {
          yield {
            type: 'additionalContext',
            message: { message: createAttachmentMessage({ type: 'hook_additional_context', ... }) }
          }
        }
        
        // 检查 abort 状态
        if (toolUseContext.abortController.signal.aborted) {
          logEvent('tengu_pre_tool_hooks_cancelled', {...})
          yield { type: 'message', message: {...} }
          yield { type: 'stop' }
          return
        }
      } catch (error) {
        // 记录错误并 yield 错误消息
        logEvent('tengu_pre_tool_hook_error', {...})
        yield { type: 'message', message: { message: createAttachmentMessage({ type: 'hook_error_during_execution', ... }) } }
        yield { type: 'stop' }
      }
    }
  } catch (error) {
    logError(error)
    yield { type: 'stop' }
  }
}
```

#### 2. PostToolUse Hooks 执行流程

```typescript
export async function* runPostToolUseHooks<Input extends AnyObject, Output>(
  toolUseContext: ToolUseContext,
  tool: Tool<Input, Output>,
  toolUseID: string,
  messageId: string,
  toolInput: Record<string, unknown>,
  toolResponse: Output,
  ...
): AsyncGenerator<PostToolUseHooksResult<Output>> {
  const postToolStartTime = Date.now()
  
  try {
    const appState = toolUseContext.getAppState()
    const permissionMode = appState.toolPermissionContext.mode
    
    let toolOutput = toolResponse
    for await (const result of executePostToolHooks(...)) {
      try {
        // 处理取消
        if (result.message?.type === 'attachment' && 
            result.message.attachment.type === 'hook_cancelled') {
          logEvent('tengu_post_tool_hooks_cancelled', {...})
          yield { message: createAttachmentMessage({ type: 'hook_cancelled', ... }) }
          continue
        }
        
        // 跳过重复的 blocking_error（#31301）
        if (result.message && 
            !(result.message.type === 'attachment' && 
              result.message.attachment.type === 'hook_blocking_error')) {
          yield { message: result.message }
        }
        
        // 处理 blockingError
        if (result.blockingError) {
          yield {
            message: createAttachmentMessage({ type: 'hook_blocking_error', ... })
          }
        }
        
        // 处理 preventContinuation
        if (result.preventContinuation) {
          yield {
            message: createAttachmentMessage({ type: 'hook_stopped_continuation', ... })
          }
          return
        }
        
        // 处理 additionalContexts
        if (result.additionalContexts?.length > 0) {
          yield {
            message: createAttachmentMessage({ type: 'hook_additional_context', ... })
          }
        }
        
        // 处理 MCP 工具输出更新
        if (result.updatedMCPToolOutput && isMcpTool(tool)) {
          toolOutput = result.updatedMCPToolOutput as Output
          yield { updatedMCPToolOutput: toolOutput }
        }
      } catch (error) {
        logEvent('tengu_post_tool_hook_error', {...})
        yield { message: createAttachmentMessage({ type: 'hook_error_during_execution', ... }) }
      }
    }
  } catch (error) {
    logError(error)
  }
}
```

#### 3. 权限决策解析流程

```typescript
export async function resolveHookPermissionDecision(
  hookPermissionResult: PermissionResult | undefined,
  tool: Tool,
  input: Record<string, unknown>,
  toolUseContext: ToolUseContext,
  canUseTool: CanUseToolFn,
  assistantMessage: AssistantMessage,
  toolUseID: string,
): Promise<{ decision: PermissionDecision; input: Record<string, unknown> }> {
  const requiresInteraction = tool.requiresUserInteraction?.()
  const requireCanUseTool = toolUseContext.requireCanUseTool
  
  // Hook 返回 allow
  if (hookPermissionResult?.behavior === 'allow') {
    const hookInput = hookPermissionResult.updatedInput ?? input
    
    // 检查交互需求是否被满足
    const interactionSatisfied =
      requiresInteraction && hookPermissionResult.updatedInput !== undefined
    
    // 如果仍需交互或强制要求 canUseTool
    if ((requiresInteraction && !interactionSatisfied) || requireCanUseTool) {
      return {
        decision: await canUseTool(tool, hookInput, toolUseContext, assistantMessage, toolUseID),
        input: hookInput
      }
    }
    
    // 检查规则覆盖（deny/ask 规则仍然生效）
    const ruleCheck = await checkRuleBasedPermissions(tool, hookInput, toolUseContext)
    if (ruleCheck === null) {
      return { decision: hookPermissionResult, input: hookInput }
    }
    if (ruleCheck.behavior === 'deny') {
      return { decision: ruleCheck, input: hookInput }
    }
    // ask 规则需要显示对话框
    return {
      decision: await canUseTool(tool, hookInput, toolUseContext, assistantMessage, toolUseID),
      input: hookInput
    }
  }
  
  // Hook 返回 deny
  if (hookPermissionResult?.behavior === 'deny') {
    return { decision: hookPermissionResult, input }
  }
  
  // 无 hook 决策或 ask → 正常权限流程
  const forceDecision =
    hookPermissionResult?.behavior === 'ask' ? hookPermissionResult : undefined
  const askInput =
    hookPermissionResult?.behavior === 'ask' && hookPermissionResult.updatedInput
      ? hookPermissionResult.updatedInput
      : input
  
  return {
    decision: await canUseTool(tool, askInput, toolUseContext, assistantMessage, toolUseID, forceDecision),
    input: askInput
  }
}
```

### 数据结构

#### 钩子结果聚合类型

```typescript
export type AggregatedHookResult = {
  message?: HookResultMessage
  blockingError?: HookBlockingError
  preventContinuation?: boolean
  stopReason?: string
  hookPermissionDecisionReason?: string
  hookSource?: string
  permissionBehavior?: PermissionResult['behavior']
  additionalContexts?: string[]
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  watchPaths?: string[]
  elicitationResponse?: ElicitationResponse
  elicitationResultResponse?: ElicitationResponse
  retry?: boolean
}
```

#### 钩子阻塞错误

```typescript
export interface HookBlockingError {
  blockingError: string
  command: string
}
```

---

## 关键代码路径与文件引用

### 核心依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/hooks.ts` | `executePreToolHooks`, `executePostToolHooks`, `executePostToolUseFailureHooks`, `getPreToolHookBlockingMessage` |
| `src/Tool.ts` | `Tool`, `ToolUseContext`, `AnyObject` |
| `src/hooks/useCanUseTool.tsx` | `CanUseToolFn` |
| `src/utils/permissions/permissions.ts` | `checkRuleBasedPermissions` |
| `src/utils/permissions/PermissionResult.ts` | `PermissionResult`, `PermissionDecisionReason`, `getRuleBehaviorDescription` |
| `src/utils/attachments.ts` | `createAttachmentMessage` |
| `src/types/message.ts` | `AssistantMessage`, `AttachmentMessage`, `ProgressMessage` |
| `src/types/hooks.ts` | `HookProgress` |
| `src/services/mcp/utils.ts` | `isMcpTool` |
| `src/services/analytics/index.ts` | `logEvent` |

### 类型依赖

| 类型 | 来源 |
|-----|------|
| `McpServerType` | `toolExecution.ts` |
| `MessageUpdateLazy` | `toolExecution.ts` |
| `PermissionDecision` | `src/types/permissions.ts` |

### 调用关系

```
toolExecution.ts
    ├── runPreToolUseHooks (toolHooks.ts)
    │       └── executePreToolHooks (utils/hooks.ts)
    ├── resolveHookPermissionDecision (toolHooks.ts)
    │       └── checkRuleBasedPermissions (utils/permissions/permissions.ts)
    │       └── canUseTool (hooks/useCanUseTool.tsx)
    ├── runPostToolUseHooks (toolHooks.ts)
    │       └── executePostToolHooks (utils/hooks.ts)
    └── runPostToolUseFailureHooks (toolHooks.ts)
            └── executePostToolUseFailureHooks (utils/hooks.ts)
```

---

## 依赖与外部交互

### 1. 与底层钩子执行的交互

`toolHooks.ts` 是协调层，实际钩子执行委托给 `utils/hooks.ts`：

```typescript
// PreToolUse
for await (const result of executePreToolHooks(
  tool.name,
  toolUseID,
  processedInput,
  toolUseContext,
  permissionMode,
  signal,
  timeoutMs,
  requestPrompt,
  toolUseSummary,
)) { ... }

// PostToolUse
for await (const result of executePostToolHooks(
  tool.name,
  toolUseID,
  toolInput,
  toolOutput,
  toolUseContext,
  permissionMode,
  signal,
)) { ... }
```

### 2. 与权限系统的交互

```typescript
// 规则检查（不经过交互式对话框）
const ruleCheck = await checkRuleBasedPermissions(tool, hookInput, toolUseContext)

// 完整权限检查（可能显示对话框）
const decision = await canUseTool(tool, hookInput, toolUseContext, assistantMessage, toolUseID)
```

### 3. 与遥测系统的交互

```typescript
logEvent('tengu_pre_tool_hook_error', {
  messageID: messageId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  toolName: sanitizeToolNameForAnalytics(tool.name),
  isMcp: tool.isMcp ?? false,
  duration: durationMs,
  queryChainId: toolUseContext.queryTracking?.chainId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  queryDepth: toolUseContext.queryTracking?.depth,
  mcpServerType: mcpServerType as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  requestId: requestId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
})
```

### 4. 与消息系统的交互

通过 `createAttachmentMessage` 创建各种钩子相关的附件消息：

```typescript
createAttachmentMessage({
  type: 'hook_success' | 'hook_cancelled' | 'hook_blocking_error' | 
        'hook_error_during_execution' | 'hook_additional_context' | 
        'hook_stopped_continuation' | 'hook_permission_decision',
  hookName: `PreToolUse:${tool.name}`,
  toolUseID,
  hookEvent: 'PreToolUse' | 'PostToolUse' | 'PostToolUseFailure',
  ...
})
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 重复的 blocking_error 消息

```typescript
// 问题：executeHooks 会 yield {blockingError} 和 {message: hook_blocking_error attachment}
// 两者都会创建相同的附件，导致重复显示

// 当前修复（#31301）：
if (
  result.message &&
  !(
    result.message.type === 'attachment' &&
    result.message.attachment.type === 'hook_blocking_error'
  )
) {
  yield { message: result.message }
}
```

**风险**：如果底层钩子执行逻辑改变，重复问题可能复发。

#### 2. MCP 工具输出修改的类型安全

```typescript
if ('updatedMCPToolOutput' in hookResult) {
  if (isMcpTool(tool)) {
    toolOutput = hookResult.updatedMCPToolOutput as Output
    yield { updatedMCPToolOutput: toolOutput }
  }
}
```

**风险**：使用 `in` 操作符和类型断言，缺乏编译时类型安全。

#### 3. 错误处理中的重复代码

PreToolUse、PostToolUse、PostToolUseFailure 三个函数有几乎相同的错误处理逻辑：

```typescript
// 重复模式：
logEvent('tengu_xxx_hook_error', {...})
yield {
  message: createAttachmentMessage({
    type: 'hook_error_during_execution',
    content: formatError(error),
    hookName: `...`,
    toolUseID,
    hookEvent: '...',
  })
}
```

### 边界情况

| 场景 | 处理逻辑 |
|-----|---------|
| Hook 返回 allow 但工具需要交互 | 检查 `updatedInput` 是否满足交互需求，否则调用 `canUseTool` |
| Hook 返回 allow 但存在 deny 规则 | 规则优先，转为拒绝 |
| Hook 返回 ask | 作为 `forceDecision` 传递给 `canUseTool`，对话框显示 hook 的消息 |
| 多个 hooks 返回不同权限决策 | 按顺序处理，第一个决定性结果生效 |
| Hook 执行中 abort | 记录取消事件，yield `hook_cancelled` 附件，终止执行 |
| Hook 抛出异常 | 记录错误事件，yield `hook_error_during_execution` 附件，终止执行 |
| MCP 工具输出被 hook 修改 | 验证 `isMcpTool` 后更新输出，yield `updatedMCPToolOutput` |

### 改进建议

1. **统一错误处理**
   - 提取公共的错误处理函数
   - 减少三个钩子执行函数中的重复代码

2. **增强类型安全**
   - 为 `hookResult` 引入更精确的联合类型
   - 消除 `in` 操作符检查和类型断言

3. **优化权限决策逻辑**
   - `resolveHookPermissionDecision` 函数较长，可拆分为子函数
   - 添加决策路径的详细日志记录（用于调试复杂的权限场景）

4. **改进遥测一致性**
   - 统一事件命名（`tengu_pre_tool_hook_error` vs `tengu_post_tool_hook_error`）
   - 确保所有路径都记录适当的遥测数据

5. **支持更多 Hook 事件**
   - 当前仅支持 PreToolUse、PostToolUse、PostToolUseFailure
   - 可考虑添加 ToolUseProgress（工具执行过程中的周期性钩子）

6. **性能优化**
   - `checkRuleBasedPermissions` 在 `resolveHookPermissionDecision` 中同步调用
   - 如果规则检查较慢，会阻塞钩子结果处理

7. **文档与测试**
   - 添加更多关于钩子权限决策优先级的文档
   - 增加单元测试覆盖复杂的决策组合场景
