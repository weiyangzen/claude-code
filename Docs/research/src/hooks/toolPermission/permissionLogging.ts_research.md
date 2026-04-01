# permissionLogging.ts 深度研究文档

## 场景与职责

`permissionLogging.ts` 是 Claude Code 工具权限系统的**集中式分析/遥测日志模块**。所有权限批准/拒绝事件都通过 `logPermissionDecision()` 函数流转，该函数将事件分发到 Statsig 分析、OTel 遥测和代码编辑指标系统。

### 核心职责

1. **统一权限决策日志入口**：提供 `logPermissionDecision()` 作为所有权限决策记录的唯一入口
2. **多后端事件分发**：将事件同时发送到：
   - Statsig 分析（用户行为分析）
   - OpenTelemetry 遥测（系统可观测性）
   - 代码编辑工具计数器（特定工具的使用统计）
3. **决策来源标准化**：将结构化的决策来源转换为字符串标签
4. **代码编辑工具特殊处理**：为 Edit/Write/NotebookEdit 工具收集语言元数据

### 在系统中的位置

```
权限决策流程:
用户/Hook/分类器/配置 → PermissionDecision 
                              ↓
                    logPermissionDecision()
                              ↓
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
   Statsig Analytics      OTel Telemetry      Code Edit Metrics
   (用户行为分析)          (系统可观测性)       (工具使用统计)
```

---

## 功能点目的

### 1. 权限决策参数类型 (`PermissionDecisionArgs`)

```typescript
type PermissionDecisionArgs =
  | { decision: 'accept'; source: PermissionApprovalSource | 'config' }
  | { decision: 'reject'; source: PermissionRejectionSource | 'config' }
```

**目的**：
- 使用可辨识联合类型确保类型安全
- 区分接受和拒绝决策的不同来源类型
- 支持 `'config'` 作为特殊的配置来源（自动批准/拒绝列表）

### 2. 代码编辑工具识别 (`isCodeEditingTool`)

```typescript
const CODE_EDITING_TOOLS = ['Edit', 'Write', 'NotebookEdit']
```

**目的**：
- 识别需要特殊指标收集的代码编辑工具
- 为这些工具收集编程语言元数据
- 用于分析不同语言文件的编辑模式

### 3. 决策来源字符串化 (`sourceToString`)

将结构化的决策来源转换为分析系统使用的字符串标签：

| 来源类型 | 输出字符串 | 说明 |
|---------|-----------|------|
| `{ type: 'classifier' }` | `'classifier'` | AI 分类器自动审批 |
| `{ type: 'hook' }` | `'hook'` | Hook 自动审批 |
| `{ type: 'user', permanent: true }` | `'user_permanent'` | 用户永久批准 |
| `{ type: 'user', permanent: false }` | `'user_temporary'` | 用户临时批准 |
| `{ type: 'user_abort' }` | `'user_abort'` | 用户中止 |
| `{ type: 'user_reject', ... }` | `'user_reject'` | 用户拒绝 |

### 4. 批准事件分类 (`logApprovalEvent`)

根据批准来源发送不同的事件名称，支持漏斗分析：

| 来源 | 事件名称 | 用途 |
|------|---------|------|
| `'config'` | `tengu_tool_use_granted_in_config` | 配置自动批准 |
| `'classifier'` | `tengu_tool_use_granted_by_classifier` | 分类器批准 |
| `user` permanent=true | `tengu_tool_use_granted_in_prompt_permanent` | 用户永久批准 |
| `user` permanent=false | `tengu_tool_use_granted_in_prompt_temporary` | 用户临时批准 |
| `hook` | `tengu_tool_use_granted_by_permission_hook` | Hook 批准 |

### 5. 代码编辑工具属性构建 (`buildCodeEditToolAttributes`)

```typescript
async function buildCodeEditToolAttributes(
  tool: ToolType,
  input: unknown,
  decision: 'accept' | 'reject',
  source: string,
): Promise<Record<string, string>>
```

**目的**：
- 提取目标文件路径（如果工具提供 `getPath` 方法）
- 使用 `getLanguageName()` 识别编程语言
- 为 OTel 计数器提供维度属性（decision, source, tool_name, language）

---

## 具体技术实现

### 关键流程

#### 1. 主入口流程 (`logPermissionDecision`)

```typescript
function logPermissionDecision(
  ctx: PermissionLogContext,           // 日志上下文
  args: PermissionDecisionArgs,        // 决策参数
  permissionPromptStartTimeMs?: number // 可选的等待时间计算
): void
```

执行流程：

```
1. 计算等待时间
   waiting_for_user_permission_ms = Date.now() - permissionPromptStartTimeMs
   
2. 记录分析事件
   if (decision === 'accept') → logApprovalEvent()
   else → logRejectionEvent()
   
3. 转换来源为字符串
   sourceString = source === 'config' ? 'config' : sourceToString(source)
   
4. 代码编辑工具指标
   if (isCodeEditingTool(tool.name)):
     buildCodeEditToolAttributes() → getCodeEditToolDecisionCounter()?.add(1, attributes)
   
5. 持久化决策到上下文
   toolUseContext.toolDecisions.set(toolUseID, { source, decision, timestamp })
   
6. 记录 OTel 事件
   logOTelEvent('tool_decision', { decision, source, tool_name })
```

#### 2. 代码编辑工具语言检测流程

```typescript
async function buildCodeEditToolAttributes(tool, input, decision, source):
  if (tool.getPath && input):
    parseResult = tool.inputSchema.safeParse(input)
    if (parseResult.success):
      filePath = tool.getPath(parseResult.data)
      if (filePath):
        language = await getLanguageName(filePath)
  
  return {
    decision,
    source,
    tool_name: tool.name,
    ...(language && { language }),
  }
```

**注意**：语言检测是异步的，但 `logPermissionDecision` 使用 `void` 忽略 Promise，不阻塞主流程。

#### 3. 基础元数据构建 (`baseMetadata`)

```typescript
function baseMetadata(
  messageId: string,
  toolName: string,
  waitMs: number | undefined,
): { [key: string]: boolean | number | undefined }
```

返回的元数据：
- `messageID`: 消息 ID（已验证非代码/文件路径）
- `toolName`: 清理后的工具名称（通过 `sanitizeToolNameForAnalytics`）
- `sandboxEnabled`: 沙箱是否启用
- `waiting_for_user_permission_ms`: 用户等待时间（仅当实际提示用户时）

### 数据结构

#### PermissionLogContext

```typescript
type PermissionLogContext = {
  tool: ToolType           // 工具定义
  input: unknown           // 工具输入（用于代码编辑工具的语言检测）
  toolUseContext: ToolUseContext  // 工具使用上下文（用于存储决策）
  messageId: string        // 消息 ID
  toolUseID: string        // 工具使用 ID
}
```

#### ToolDecision（存储在上下文中）

```typescript
{
  source: string           // 决策来源字符串
  decision: 'accept' | 'reject'
  timestamp: number        // 决策时间戳
}
```

存储位置：`toolUseContext.toolDecisions: Map<string, ToolDecision>`

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `ToolType`, `ToolUseContext` 类型 |
| `src/hooks/toolPermission/PermissionContext.ts` | `PermissionApprovalSource`, `PermissionRejectionSource` 类型 |
| `src/utils/cliHighlight.ts` | `getLanguageName()` 语言检测 |
| `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager.isSandboxingEnabled()` |
| `src/utils/telemetry/events.ts` | `logOTelEvent()` OTel 事件记录 |
| `src/bootstrap/state.ts` | `getCodeEditToolDecisionCounter()` 代码编辑计数器 |
| `src/services/analytics/index.ts` | `logEvent()`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` |
| `src/services/analytics/metadata.ts` | `sanitizeToolNameForAnalytics()` |
| `bun:bundle` | `feature()` 特性开关 |

### 外部调用方

1. **`src/hooks/toolPermission/PermissionContext.ts`** - 主要调用方
   - `logDecision()` 方法内部调用 `logPermissionDecision()`
   - 在 `handleUserAllow()`, `handleHookAllow()`, `runHooks()`, `tryClassifier()` 中调用

2. **`src/hooks/useCanUseTool.ts`** - 权限检查 Hook
   - 直接调用 `logPermissionDecision()` 记录配置来源的决策

3. **其他权限相关模块**
   - 任何需要记录权限决策的模块

---

## 依赖与外部交互

### 与分析系统的交互

```typescript
import { logEvent } from 'src/services/analytics/index.js'

// 同步事件记录
logEvent('tengu_tool_use_granted_in_prompt_permanent', {
  messageID: messageId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  toolName: sanitizeToolNameForAnalytics(tool.name),
  sandboxEnabled: SandboxManager.isSandboxingEnabled(),
  waiting_for_user_permission_ms: waitMs,
})
```

- 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型标记确保不包含敏感数据
- 工具名称通过 `sanitizeToolNameForAnalytics()` 清理

### 与 OTel 遥测系统的交互

```typescript
import { logOTelEvent } from '../../utils/telemetry/events.js'

void logOTelEvent('tool_decision', {
  decision,
  source: sourceString,
  tool_name: sanitizeToolNameForAnalytics(tool.name),
})
```

- 使用 `void` 忽略异步操作，不阻塞主流程
- 事件名称为 `tool_decision`

### 与代码编辑指标系统的交互

```typescript
import { getCodeEditToolDecisionCounter } from '../../bootstrap/state.js'

if (isCodeEditingTool(tool.name)) {
  void buildCodeEditToolAttributes(tool, input, decision, sourceString).then(
    attributes => getCodeEditToolDecisionCounter()?.add(1, attributes),
  )
}
```

- 通过 `getCodeEditToolDecisionCounter()` 获取 OTel 计数器
- 计数器可能为 undefined（如果遥测未初始化）
- 使用 `?.` 可选链安全调用

### 与沙箱系统的交互

```typescript
import { SandboxManager } from '../../utils/sandbox/sandbox-adapter.js'

// 在基础元数据中包含沙箱状态
sandboxEnabled: SandboxManager.isSandboxingEnabled()
```

---

## 风险、边界与改进建议

### 已知风险

1. **异步操作未等待**
   ```typescript
   void buildCodeEditToolAttributes(...).then(...)  // 未等待
   void logOTelEvent(...)  // 未等待
   ```
   - 代码编辑工具属性构建和 OTel 事件记录都是异步的
   - 使用 `void` 忽略可能导致事件丢失（如果进程在 Promise 完成前退出）
   - 但在正常流程中，这些操作的延迟很短，风险较低

2. **语言检测失败静默处理**
   ```typescript
   if (parseResult.success) {
     const filePath = tool.getPath(parseResult.data)
     if (filePath) {
       language = await getLanguageName(filePath)
     }
   }
   ```
   - 如果 `inputSchema.safeParse` 失败，语言检测被跳过
   - 如果 `getPath` 返回 undefined，语言检测被跳过
   - 这些失败没有日志记录，可能导致指标缺失

3. **计数器可能未初始化**
   ```typescript
   getCodeEditToolDecisionCounter()?.add(1, attributes)
   ```
   - 如果遥测系统未初始化，计数器为 undefined
   - 使用可选链避免错误，但指标数据丢失

4. **等待时间计算精度**
   ```typescript
   const waiting_for_user_permission_ms =
     permissionPromptStartTimeMs !== undefined
       ? Date.now() - permissionPromptStartTimeMs
       : undefined
   ```
   - 使用 `Date.now()` 可能受系统时间调整影响
   - 对于高精度计时，应考虑使用 `performance.now()`

### 边界情况

| 场景 | 行为 |
|------|------|
| `permissionPromptStartTimeMs` 未定义 | 不包含 `waiting_for_user_permission_ms` 字段 |
| `source` 为 `'config'` | 直接使用 `'config'` 字符串，不调用 `sourceToString()` |
| 代码编辑工具语言检测失败 | 返回的属性中不包含 `language` 字段 |
| `tool.getPath` 未定义 | 语言检测被跳过 |
| `toolUseContext.toolDecisions` 未初始化 | 创建新的 Map 并赋值 |

### 改进建议

1. **错误处理增强**
   ```typescript
   // 当前：静默失败
   if (isCodeEditingTool(tool.name)) {
     void buildCodeEditToolAttributes(...).then(...)
   }
   
   // 建议：添加错误日志
   if (isCodeEditingTool(tool.name)) {
     void buildCodeEditToolAttributes(...)
       .then(attributes => ...)
       .catch(err => logForDebugging(`Failed to build code edit attributes: ${err}`))
   }
   ```

2. **使用性能计时器**
   ```typescript
   // 当前
   const waitMs = Date.now() - permissionPromptStartTimeMs
   
   // 建议
   const waitMs = performance.now() - permissionPromptStartTimeMs
   ```

3. **批量事件发送**
   - 当前每个权限决策触发多个独立的事件发送
   - 考虑实现批处理机制减少开销

4. **元数据验证**
   ```typescript
   // 添加运行时验证确保元数据不包含敏感信息
   function validateMetadata(metadata: Record<string, unknown>): boolean {
     // 检查是否包含代码片段、文件路径等
   }
   ```

5. **类型安全改进**
   ```typescript
   // 当前：使用类型断言
   messageId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
   
   // 建议：使用 branded types 提供编译时保证
   type SafeMetadata = string & { __brand: 'SafeMetadata' }
   ```

### 测试建议

1. **单元测试覆盖**
   - `sourceToString` 的所有分支
   - `isCodeEditingTool` 的工具名称匹配
   - `baseMetadata` 的字段生成

2. **集成测试**
   - 验证事件正确发送到所有后端
   - 验证代码编辑工具的语言检测
   - 验证决策持久化到上下文

3. **性能测试**
   - 高并发权限决策的场景
   - 验证异步操作不会堆积

### 安全注意事项

1. **数据隐私**
   - 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 标记确保不包含代码/文件路径
   - 工具名称通过 `sanitizeToolNameForAnalytics()` 清理
   - 但 `buildCodeEditToolAttributes` 中的 `filePath` 可能包含敏感路径信息，需要确保不泄露到分析系统

2. **注入风险**
   - 工具名称和来源字符串被用于事件名称和元数据键
   - 需要确保这些值来自受控的枚举，而非用户输入
