# hooks.ts (permissions) 研究文档

## 场景与职责

`src/components/permissions/hooks.ts` 是 Claude Code CLI 权限系统的**日志记录钩子模块**。它提供了一个 React Hook `usePermissionRequestLogging`，用于在权限请求对话框显示时记录分析事件和一元日志（unary logging）。

该模块解决了权限系统的**可观测性需求**：
- 跟踪权限请求的发生频率和模式
- 为内部调试和分析收集详细的权限决策信息
- 支持特性开发（如 BASH_CLASSIFIER）的数据收集

## 功能点目的

### 1. 权限请求分析日志
记录权限请求事件到分析系统：
- 工具名称和类型（是否 MCP）
- 决策原因类型
- 沙箱启用状态
- 消息 ID 用于关联

### 2. 内部调试日志（ANT-ONLY）
为 Anthropic 内部构建收集详细的调试信息：
- 没有规则建议的 Bash 权限请求
- Bash 命令的详细分解
- 决策原因的详细信息

### 3. 一元事件日志
记录到一元日志系统用于：
- 会话分析
- 性能监控
- 使用模式研究

### 4. 权限提示计数
更新应用状态中的权限提示计数器，用于：
- 归因跟踪
- 使用统计

## 具体技术实现

### 核心数据结构

```typescript
// 一元事件类型
export type UnaryEvent = {
  completion_type: CompletionType;
  language_name: string | Promise<string>;
};

// Hook 参数
function usePermissionRequestLogging(
  toolUseConfirm: ToolUseConfirm,
  unaryEvent: UnaryEvent,
): void
```

### 关键流程

1. **防重入保护**：
   ```typescript
   const loggedToolUseID = useRef<string | null>(null);
   
   useEffect(() => {
     if (loggedToolUseID.current === toolUseConfirm.toolUseID) {
       return;  // 已经记录过，跳过
     }
     loggedToolUseID.current = toolUseConfirm.toolUseID;
     // ... 记录日志
   }, [toolUseConfirm, unaryEvent, setAppState]);
   ```

2. **分析事件记录**：
   ```typescript
   logEvent('tengu_tool_use_show_permission_request', {
     messageID: toolUseConfirm.assistantMessage.message.id,
     toolName: sanitizeToolNameForAnalytics(toolUseConfirm.tool.name),
     isMcp: toolUseConfirm.tool.isMcp ?? false,
     decisionReasonType: toolUseConfirm.permissionResult.decisionReason?.type,
     sandboxEnabled: SandboxManager.isSandboxingEnabled(),
   });
   ```

3. **ANT-ONLY 内部日志**（仅在 `process.env.USER_TYPE === 'ant'` 时执行）：
   
   a. **无规则建议日志**：
   ```typescript
   if (toolUseConfirm.tool.name === BashTool.name &&
       permissionResult.behavior === 'ask' &&
       !hasRules(permissionResult.suggestions)) {
     logEvent('tengu_internal_tool_use_permission_request_no_always_allow', {
       // ... 详细信息
       decisionReasonDetails: decisionReasonToString(permissionResult.decisionReason),
     });
   }
   ```
   
   b. **Bash 命令详细日志**：
   ```typescript
   if (toolUseConfirm.tool.name === BashTool.name &&
       toolUseConfirm.permissionResult.behavior === 'ask') {
     const split = splitCommand_DEPRECATED(parsedInput.data.command);
     logEvent('tengu_internal_bash_tool_use_permission_request', {
       parts: jsonStringify(split),
       input: jsonStringify(toolUseConfirm.input),
       decisionReasonType: ...,
       decisionReason: decisionReasonToString(...),
     });
   }
   ```

4. **一元事件记录**：
   ```typescript
   void logUnaryEvent({
     completion_type: unaryEvent.completion_type,
     event: 'response',
     metadata: {
       language_name: unaryEvent.language_name,
       message_id: toolUseConfirm.assistantMessage.message.id,
       platform: env.platform,
     },
   });
   ```

### 决策原因序列化

```typescript
function decisionReasonToString(
  decisionReason: PermissionDecisionReason | undefined,
): string {
  if (!decisionReason) return 'No decision reason';
  
  // 处理分类器类型
  if ((feature('BASH_CLASSIFIER') || feature('TRANSCRIPT_CLASSIFIER')) &&
      decisionReason.type === 'classifier') {
    return `Classifier: ${decisionReason.classifier}, Reason: ${decisionReason.reason}`;
  }
  
  switch (decisionReason.type) {
    case 'rule':
      return `Rule: ${permissionRuleValueToString(decisionReason.rule.ruleValue)}`;
    case 'mode':
      return `Mode: ${decisionReason.mode}`;
    case 'subcommandResults':
      return `Subcommand Results: ${...}`;
    case 'permissionPromptTool':
      return `Permission Tool: ${...}`;
    case 'hook':
      return `Hook: ${decisionReason.hookName}${...}`;
    case 'workingDir':
      return `Working Directory: ${decisionReason.reason}`;
    case 'safetyCheck':
      return `Safety check: ${decisionReason.reason}`;
    case 'other':
      return `Other: ${decisionReason.reason}`;
    default:
      return jsonStringify(decisionReason, null, 2);
  }
}
```

### 权限结果序列化

```typescript
function permissionResultToLog(permissionResult: PermissionResult): string {
  switch (permissionResult.behavior) {
    case 'allow':
      return 'allow';
    case 'ask':
      const rules = extractRules(permissionResult.suggestions);
      const suggestions = rules.length > 0 
        ? rules.map(r => permissionRuleValueToString(r)).join(', ')
        : 'none';
      return `ask: ${permissionResult.message}, suggestions: ${suggestions}, reason: ${...}`;
    case 'deny':
      return `deny: ${permissionResult.message}, reason: ${...}`;
    case 'passthrough':
      // 类似 ask 的处理
  }
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `feature` | `bun:bundle` | 特性开关检查 |
| `useEffect`, `useRef` | `react` | React Hooks |
| `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | `src/services/analytics/index.js` | 分析元数据类型 |
| `logEvent` | `src/services/analytics/index.js` | 分析日志函数 |
| `sanitizeToolNameForAnalytics` | `src/services/analytics/metadata.js` | 工具名称清理 |
| `BashTool` | `src/tools/BashTool/BashTool.js` | Bash 工具引用 |
| `splitCommand_DEPRECATED` | `src/utils/bash/commands.js` | 命令分解 |
| `PermissionDecisionReason`, `PermissionResult` | `src/utils/permissions/PermissionResult.js` | 权限类型 |
| `extractRules`, `hasRules` | `src/utils/permissions/PermissionUpdate.js` | 规则提取 |
| `permissionRuleValueToString` | `src/utils/permissions/permissionRuleParser.js` | 规则格式化 |
| `SandboxManager` | `src/utils/sandbox/sandbox-adapter.js` | 沙箱状态 |
| `ToolUseConfirm` | `./PermissionRequest.js` | 工具使用确认类型 |
| `useSetAppState` | `../../state/AppState.js` | 状态更新 |
| `env` | `../../utils/env.js` | 环境信息 |
| `jsonStringify` | `../../utils/slowOperations.js` | JSON 序列化 |
| `CompletionType`, `logUnaryEvent` | `../../utils/unaryLogging.js` | 一元日志 |

### 被调用方

该 Hook 被各具体权限请求组件使用：

```typescript
// BashPermissionRequest.tsx
function BashPermissionRequest({ toolUseConfirm, onDone, onReject, verbose }: Props) {
  usePermissionRequestLogging(toolUseConfirm, {
     completion_type: 'bash_tool_use',
     language_name: 'bash',
  });
  // ...
}

// FileEditPermissionRequest.tsx
function FileEditPermissionRequest({ toolUseConfirm, ... }: Props) {
  usePermissionRequestLogging(toolUseConfirm, {
     completion_type: 'file_edit_tool_use',
     language_name: 'none',
  });
  // ...
}
```

### 数据流

```
权限请求组件挂载
  ↓
usePermissionRequestLogging 执行
  ↓
检查 loggedToolUseID（防重入）
  ↓
更新 permissionPromptCount
  ↓
logEvent('tengu_tool_use_show_permission_request')
  ↓
[ANT-ONLY] 记录内部调试日志
  ↓
logUnaryEvent('response')
```

## 风险、边界与改进建议

### 当前风险

1. **防重入机制的局限性**：
   - 使用 `useRef` 存储已记录的 toolUseID
   - 组件卸载后重新挂载会重置，可能导致重复记录

2. **ANT-ONLY 代码的维护**：
   - 内部调试代码与生产代码混合
   - 可能意外泄露到外部构建

3. **敏感信息泄露风险**：
   - 内部日志包含命令内容（`parts`, `input`）
   - 注释明确说明"包含代码/文件路径，不应在公共构建中记录"

4. **DEPRECATED 函数依赖**：
   - 使用 `splitCommand_DEPRECATED`
   - 依赖已弃用的功能

### 边界情况

1. **快速连续的权限请求**：
   - 如果同一 toolUseID 的权限请求在短时间内多次触发
   - 防重入机制会阻止后续日志记录

2. **异步 language_name**：
   - `UnaryEvent.language_name` 可以是 Promise
   - `logUnaryEvent` 需要正确处理异步值

3. **分析服务不可用**：
   - 如果分析服务失败，不应阻塞权限流程
   - 当前使用 `void` 忽略 Promise 结果

### 改进建议

1. **更可靠的去重机制**：
   ```typescript
   // 使用 Set 存储已记录的 ID
   const loggedToolUseIDs = useRef<Set<string>>(new Set());
   
   useEffect(() => {
     if (loggedToolUseIDs.current.has(toolUseConfirm.toolUseID)) {
       return;
     }
     loggedToolUseIDs.current.add(toolUseConfirm.toolUseID);
     // ...
   }, []);
   ```

2. **日志采样**：
   - 对于高频事件，考虑添加采样机制
   - 减少分析系统的负载

3. **错误处理**：
   ```typescript
   logEvent(...).catch(err => {
     logForDebugging(`Failed to log permission request: ${err}`);
   });
   ```

4. **替换 DEPRECATED 函数**：
   - 迁移到新的命令分解工具
   - 移除对 `splitCommand_DEPRECATED` 的依赖

5. **配置化日志级别**：
   - 允许通过设置控制日志详细程度
   - 便于调试和性能优化

6. **隐私审查**：
   - 定期审查内部日志内容
   - 确保不会意外收集敏感信息

7. **指标收集**：
   - 除了事件日志，还可以收集指标（metrics）
   - 如权限请求延迟、用户响应时间等
