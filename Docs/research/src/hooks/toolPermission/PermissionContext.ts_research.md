# PermissionContext.ts 深度研究文档

## 场景与职责

`PermissionContext.ts` 是 Claude Code 工具权限系统的核心模块，负责**创建和管理工具使用请求的权限上下文**。它位于权限决策流程的关键路径上，桥接了工具调用、权限检查、用户交互和持久化存储。

### 核心职责

1. **权限上下文创建**：为每个工具使用请求创建完整的权限决策上下文
2. **决策来源管理**：支持多种权限决策来源（用户、Hook、分类器、配置）
3. **权限队列操作**：提供与 React 状态解耦的权限队列操作接口
4. **异步决策解析**：通过 `ResolveOnce` 模式确保权限决策的原子性
5. **分类器集成**：支持 Bash 分类器和 Transcript 分类器的自动审批
6. **Hook 执行**：执行权限请求 Hook 并处理其返回结果

### 在系统中的位置

```
工具调用流程:
Tool.call() → checkPermissions() → PermissionContext → 用户/UI/Hook/分类器
                                           ↓
                              persistPermissions() → Settings
```

---

## 功能点目的

### 1. 权限审批来源类型 (`PermissionApprovalSource` / `PermissionRejectionSource`)

```typescript
type PermissionApprovalSource =
  | { type: 'hook'; permanent?: boolean }      // Hook 自动审批
  | { type: 'user'; permanent: boolean }       // 用户手动审批
  | { type: 'classifier' }                     // AI 分类器自动审批

type PermissionRejectionSource =
  | { type: 'hook' }                           // Hook 拒绝
  | { type: 'user_abort' }                     // 用户中止
  | { type: 'user_reject'; hasFeedback: boolean } // 用户拒绝（可能带反馈）
```

**目的**：区分权限决策的来源，用于：
- 分析事件追踪（Analytics）
- 决策原因记录
- 持久化规则生成

### 2. 权限队列操作抽象 (`PermissionQueueOps`)

```typescript
type PermissionQueueOps = {
  push(item: ToolUseConfirm): void
  remove(toolUseID: string): void
  update(toolUseID: string, patch: Partial<ToolUseConfirm>): void
}
```

**目的**：
- 将权限上下文与 React 状态管理解耦
- 支持 REPL 模式（React 状态）和非交互式模式（无 UI）
- 提供统一的队列操作接口

### 3. 一次性解析器 (`ResolveOnce` / `createResolveOnce`)

**目的**：解决异步权限决策中的竞态条件问题：
- 防止多次解析同一个权限请求
- 提供原子性的 `claim()` 操作
- 确保在并发场景下（分类器 + 用户响应）只有一个决策生效

```typescript
const resolveOnce = createResolveOnce<PermissionDecision>(resolve)
// 在异步回调中使用
if (resolveOnce.claim()) {
  resolveOnce.resolve(decision)
}
```

### 4. 权限上下文工厂 (`createPermissionContext`)

创建包含以下能力的上下文对象：

| 方法 | 用途 |
|------|------|
| `logDecision()` | 记录权限决策到分析系统 |
| `logCancelled()` | 记录用户取消事件 |
| `persistPermissions()` | 将权限更新持久化到设置 |
| `resolveIfAborted()` | 检查中止信号并返回取消决策 |
| `cancelAndAbort()` | 构建取消/拒绝决策并中止控制器 |
| `tryClassifier()` | 尝试使用分类器自动审批（Bash 工具） |
| `runHooks()` | 执行权限请求 Hook |
| `buildAllow()` / `buildDeny()` | 构建允许/拒绝决策对象 |
| `handleUserAllow()` / `handleHookAllow()` | 处理用户/Hook 的允许决策 |
| `pushToQueue()` / `removeFromQueue()` / `updateQueueItem()` | 队列操作 |

---

## 具体技术实现

### 关键流程

#### 1. 权限上下文创建流程

```typescript
function createPermissionContext(
  tool: ToolType,                    // 被请求的工具
  input: Record<string, unknown>,    // 工具输入参数
  toolUseContext: ToolUseContext,    // 工具使用上下文
  assistantMessage: AssistantMessage, // 助手消息
  toolUseID: string,                 // 工具使用唯一ID
  setToolPermissionContext: (context: ToolPermissionContext) => void, // 状态更新函数
  queueOps?: PermissionQueueOps,     // 可选的队列操作
): PermissionContext
```

#### 2. 分类器自动审批流程（Bash 工具）

```
条件检查:
1. feature('BASH_CLASSIFIER') 必须启用
2. tool.name === 'Bash'
3. pendingClassifierCheck 存在

执行流程:
awaitClassifierAutoApproval() → 等待分类器结果
    ↓
分类器返回决策 → 如果允许:
    - 记录分类器审批 (setClassifierApproval)
    - 记录分析事件 (logPermissionDecision)
    - 返回 { behavior: 'allow', decisionReason: { type: 'classifier', ... } }
```

#### 3. Hook 执行流程

```typescript
async runHooks(
  permissionMode: string | undefined,
  suggestions: PermissionUpdate[] | undefined,
  updatedInput?: Record<string, unknown>,
  permissionPromptStartTimeMs?: number,
): Promise<PermissionDecision | null>
```

流程：
1. 调用 `executePermissionRequestHooks()` 获取 Hook 结果迭代器
2. 遍历每个 Hook 结果：
   - 如果 `behavior === 'allow'` → 调用 `handleHookAllow()`
   - 如果 `behavior === 'deny'` → 记录拒绝并返回拒绝决策
   - 如果 `interrupt === true` → 中止控制器
3. 返回 null 表示没有 Hook 做出决策

#### 4. 用户允许决策处理流程

```typescript
async handleUserAllow(
  updatedInput: Record<string, unknown>,
  permissionUpdates: PermissionUpdate[],
  feedback?: string,
  permissionPromptStartTimeMs?: number,
  contentBlocks?: ContentBlockParam[],
  decisionReason?: PermissionDecisionReason,
): Promise<PermissionAllowDecision>
```

流程：
1. 调用 `persistPermissions()` 持久化权限更新
2. 记录分析事件（包含等待时间）
3. 检查用户是否修改了输入（通过 `tool.inputsEquivalent`）
4. 构建并返回 `PermissionAllowDecision`

### 数据结构

#### PermissionContext 对象结构

```typescript
interface PermissionContext {
  // 原始数据
  tool: ToolType
  input: Record<string, unknown>
  toolUseContext: ToolUseContext
  assistantMessage: AssistantMessage
  messageId: string
  toolUseID: string
  
  // 日志方法
  logDecision(args: PermissionDecisionArgs, opts?: {...}): void
  logCancelled(): void
  
  // 持久化方法
  persistPermissions(updates: PermissionUpdate[]): Promise<boolean>
  
  // 中止处理
  resolveIfAborted(resolve: (decision: PermissionDecision) => void): boolean
  cancelAndAbort(feedback?: string, isAbort?: boolean, contentBlocks?: ContentBlockParam[]): PermissionDecision
  
  // 分类器（条件性存在，取决于 feature('BASH_CLASSIFIER')）
  tryClassifier?(pendingClassifierCheck: PendingClassifierCheck | undefined, updatedInput?: Record<string, unknown>): Promise<PermissionDecision | null>
  
  // Hook 执行
  runHooks(permissionMode?: string, suggestions?: PermissionUpdate[], ...): Promise<PermissionDecision | null>
  
  // 决策构建
  buildAllow(updatedInput: Record<string, unknown>, opts?: {...}): PermissionAllowDecision
  buildDeny(message: string, decisionReason: PermissionDecisionReason): PermissionDenyDecision
  
  // 决策处理
  handleUserAllow(...): Promise<PermissionAllowDecision>
  handleHookAllow(...): Promise<PermissionAllowDecision>
  
  // 队列操作
  pushToQueue(item: ToolUseConfirm): void
  removeFromQueue(): void
  updateQueueItem(patch: Partial<ToolUseConfirm>): void
}
```

#### ResolveOnce 状态机

```
初始状态: claimed = false, delivered = false

claim():
  - 如果 claimed = true → 返回 false
  - 否则 claimed = true → 返回 true

resolve(value):
  - 如果 delivered = true → 无操作
  - 否则 delivered = true, claimed = true → 调用原始 resolve

isResolved():
  - 返回 claimed
```

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `ToolType`, `ToolUseContext`, `ToolPermissionContext` 类型 |
| `src/types/permissions.ts` | `PendingClassifierCheck`, `PermissionAllowDecision`, `PermissionDenyDecision`, `PermissionDecisionReason` |
| `src/types/message.ts` | `AssistantMessage` |
| `src/utils/permissions/PermissionResult.ts` | `PermissionDecision` 类型 |
| `src/utils/permissions/PermissionUpdate.ts` | `applyPermissionUpdates`, `persistPermissionUpdates`, `supportsPersistence` |
| `src/utils/permissions/PermissionUpdateSchema.ts` | `PermissionUpdate` 类型 |
| `src/utils/hooks.ts` | `executePermissionRequestHooks` |
| `src/utils/classifierApprovals.ts` | `setClassifierApproval` |
| `src/utils/messages.ts` | `REJECT_MESSAGE`, `REJECT_MESSAGE_WITH_REASON_PREFIX`, `SUBAGENT_REJECT_MESSAGE`, `SUBAGENT_REJECT_MESSAGE_WITH_REASON_PREFIX`, `withMemoryCorrectionHint` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/tools/BashTool/bashPermissions.ts` | `awaitClassifierAutoApproval` |
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` |
| `src/services/analytics/index.ts` | `logEvent`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` |
| `src/services/analytics/metadata.ts` | `sanitizeToolNameForAnalytics` |
| `@anthropic-ai/sdk/resources/messages.mjs` | `ContentBlockParam` |
| `bun:bundle` | `feature` 函数（特性开关） |

### 外部调用方

1. **`src/hooks/useCanUseTool.ts`** - 主要的权限检查 Hook，创建并使用 PermissionContext
2. **权限相关组件** - 使用 `createPermissionQueueOps` 创建 React 绑定的队列操作

---

## 依赖与外部交互

### 与分类器系统的交互

```typescript
// 条件性包含的分类器支持
...(feature('BASH_CLASSIFIER')
  ? {
      async tryClassifier(pendingClassifierCheck, updatedInput) {
        if (tool.name !== BASH_TOOL_NAME || !pendingClassifierCheck) {
          return null
        }
        const classifierDecision = await awaitClassifierAutoApproval(...)
        // 处理分类器决策...
      },
    }
  : {}),
```

- 使用 Bun 的 `feature()` 进行编译时特性开关
- 仅对 Bash 工具启用分类器检查
- 与 `src/utils/classifierApprovals.ts` 协作记录分类器审批

### 与 Hook 系统的交互

- 调用 `executePermissionRequestHooks()` 执行权限请求 Hook
- 处理 Hook 返回的 `PermissionRequestResult`
- 支持 Hook 的中断行为（`interrupt: true`）

### 与持久化系统的交互

```typescript
async persistPermissions(updates: PermissionUpdate[]): Promise<boolean> {
  if (updates.length === 0) return false
  persistPermissionUpdates(updates)  // 写入设置文件
  const appState = toolUseContext.getAppState()
  setToolPermissionContext(
    applyPermissionUpdates(appState.toolPermissionContext, updates),
  )
  return updates.some(update => supportsPersistence(update.destination))
}
```

- 调用 `persistPermissionUpdates()` 将规则写入设置文件
- 调用 `applyPermissionUpdates()` 更新内存中的权限上下文
- 返回是否有更新被持久化（排除仅 session 的更新）

### 与分析系统的交互

- 通过 `logDecision()` 调用 `logPermissionDecision()` 记录所有权限决策
- 通过 `logCancelled()` 记录工具使用取消事件
- 使用 `sanitizeToolNameForAnalytics()` 清理工具名称

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件风险**
   - 分类器和用户响应可能同时到达
   - 通过 `ResolveOnce` 模式缓解，但需要确保所有异步路径都使用 `claim()`

2. **特性开关复杂性**
   - `feature('BASH_CLASSIFIER')` 是编译时开关，代码在特性关闭时会被 DCE
   - 但类型定义仍然存在，可能导致类型不匹配

3. **子代理消息混淆**
   - `cancelAndAbort()` 需要区分主线程和子代理的消息
   - 使用 `toolUseContext.agentId` 判断是否为子代理

4. **内存泄漏风险**
   - `PermissionContext` 对象被冻结（`Object.freeze`）
   - 但队列操作可能持有对 React 状态的引用

### 边界情况

| 场景 | 行为 |
|------|------|
| 空权限更新数组 | `persistPermissions` 立即返回 false |
| 中止信号已触发 | `resolveIfAborted` 返回 true 并自动解决为取消 |
| 非 Bash 工具的分类器检查 | `tryClassifier` 返回 null |
| 无队列操作提供 | 队列方法变为无操作（optional chaining） |
| 特性开关关闭 | 分类器相关代码被完全移除 |

### 改进建议

1. **类型安全改进**
   ```typescript
   // 当前：使用类型断言
   return { behavior: 'allow' as const, ... }
   
   // 建议：使用 satisfies 确保类型安全
   return { behavior: 'allow', ... } satisfies PermissionAllowDecision
   ```

2. **错误处理增强**
   - `persistPermissions` 中的设置更新可能失败，但当前没有错误传播
   - 建议添加 try-catch 并返回更详细的状态

3. **测试覆盖率**
   - `createResolveOnce` 的竞态条件需要专门的并发测试
   - 分类器集成路径需要 mock 测试

4. **文档改进**
   - `PermissionQueueOps` 的契约（如是否允许重复 push）需要更明确的文档
   - `ResolveOnce` 的使用模式需要示例代码

5. **性能优化**
   - `Object.freeze(ctx)` 在每次创建上下文时执行，可能有一定开销
   - 考虑使用原型模式减少属性复制

### 安全注意事项

1. **Hook 执行安全**
   - Hook 可以返回 `interrupt: true` 中止控制器
   - 需要确保 Hook 来源可信（由 Hook 系统保证）

2. **持久化安全**
   - 权限规则写入用户设置文件
   - 需要验证规则格式（由 `PermissionUpdateSchema` 保证）

3. **分类器信任**
   - 分类器自动审批的决策被记录但不再二次验证
   - 依赖分类器本身的准确性
