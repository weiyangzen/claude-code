# StreamingToolExecutor.ts 深度研究文档

## 场景与职责

`StreamingToolExecutor` 是 Claude Code 中负责**流式工具执行**的核心类，主要解决以下场景：

1. **并发工具执行控制**：当模型同时请求多个工具时，需要智能判断哪些可以并行执行、哪些必须串行
2. **流式响应处理**：工具调用是逐步流式到达的，需要在接收过程中就开始执行而非等待全部接收完毕
3. **错误级联处理**：当某个工具执行失败时，需要优雅地取消相关并行工具（特别是 Bash 工具的依赖链）
4. **进度实时反馈**：支持工具执行过程中的进度消息实时推送给用户

### 核心职责
- 管理工具执行队列，维护工具状态机（queued → executing → completed → yielded）
- 实现并发安全策略：并发安全工具可并行，非并发安全工具独占执行
- 处理工具错误时的兄弟工具取消（sibling abort）机制
- 支持流式 fallback 场景下的工具丢弃（discard）
- 维护工具执行上下文（ToolUseContext）的更新

---

## 功能点目的

### 1. 并发控制策略

```typescript
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean  // 关键字段：决定并行策略
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]  // 进度消息单独存储，立即 yield
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

**并发规则**：
- 如果当前没有执行中的工具 → 任何工具都可以执行
- 如果当前执行中的工具全是并发安全的 → 新工具如果是并发安全的可以并行加入
- 如果存在非并发安全工具在执行 → 新工具必须等待

### 2. 错误级联机制

**设计原理**：Bash 命令往往有隐式依赖链（如 `mkdir` 失败 → 后续命令无意义），因此 Bash 工具错误会触发兄弟工具取消。

```typescript
// 仅 Bash 错误会取消兄弟工具
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

### 3. AbortController 层级结构

```
toolUseContext.abortController (父级，用户中断/新消息)
    └── siblingAbortController (子级，Bash 错误时触发)
            └── toolAbortController (每个工具的独立控制器)
```

**关键设计**：
- `siblingAbortController` 是 `toolUseContext.abortController` 的子级
- 当 Bash 错误时，abort `siblingAbortController` 会级联取消所有运行中的兄弟工具
- 但**不会**影响父级，因此 query.ts 不会结束整个 turn

### 4. 流式 Fallback 支持

当发生 streaming fallback（如模型切换）时，需要丢弃之前尝试的结果：

```typescript
discard(): void {
  this.discarded = true
  // 后续 getRemainingResults/getCompletedResults 会立即返回
}
```

---

## 具体技术实现

### 关键流程

#### 1. 工具添加流程 (`addTool`)

```typescript
addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
  // 1. 查找工具定义
  const toolDefinition = findToolByName(this.toolDefinitions, block.name)
  
  // 2. 如果工具不存在，直接生成错误结果
  if (!toolDefinition) {
    this.tools.push({
      id: block.id,
      status: 'completed',  // 直接标记完成
      results: [createUserMessage({...})],
      ...
    })
    return
  }
  
  // 3. 解析输入并判断并发安全性
  const parsedInput = toolDefinition.inputSchema.safeParse(block.input)
  const isConcurrencySafe = parsedInput?.success
    ? toolDefinition.isConcurrencySafe(parsedInput.data)
    : false
  
  // 4. 加入队列并触发处理
  this.tools.push({ id: block.id, status: 'queued', isConcurrencySafe, ... })
  void this.processQueue()
}
```

#### 2. 队列处理流程 (`processQueue`)

```typescript
private async processQueue(): Promise<void> {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue
    
    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)  // 启动执行
    } else if (!tool.isConcurrencySafe) {
      break  // 非并发安全工具不能执行，且由于要保持顺序，停止处理
    }
  }
}
```

#### 3. 工具执行流程 (`executeTool`)

```typescript
private async executeTool(tool: TrackedTool): Promise<void> {
  tool.status = 'executing'
  this.toolUseContext.setInProgressToolUseIDs(prev => new Set(prev).add(tool.id))
  
  const collectResults = async () => {
    // 1. 检查初始 abort 状态
    const initialAbortReason = this.getAbortReason(tool)
    if (initialAbortReason) {
      // 生成合成错误消息，不实际执行工具
      tool.results = [this.createSyntheticErrorMessage(...)]
      return
    }
    
    // 2. 创建工具级 abort controller
    const toolAbortController = createChildAbortController(this.siblingAbortController)
    
    // 3. 关键：非 sibling_error 的 abort 需要冒泡到父级
    toolAbortController.signal.addEventListener('abort', () => {
      if (toolAbortController.signal.reason !== 'sibling_error' && 
          !this.toolUseContext.abortController.signal.aborted) {
        this.toolUseContext.abortController.abort(toolAbortController.signal.reason)
      }
    }, { once: true })
    
    // 4. 执行工具（通过 runToolUse generator）
    const generator = runToolUse(tool.block, tool.assistantMessage, ...)
    
    for await (const update of generator) {
      // 5. 检查是否被兄弟错误取消
      const abortReason = this.getAbortReason(tool)
      if (abortReason && !thisToolErrored) {
        messages.push(this.createSyntheticErrorMessage(...))
        break
      }
      
      // 6. 处理进度消息和普通结果
      if (update.message.type === 'progress') {
        tool.pendingProgress.push(update.message)
        this.progressAvailableResolve?.()  // 唤醒等待者
      } else {
        messages.push(update.message)
      }
    }
  }
  
  // 7. 启动执行并链式处理队列
  const promise = collectResults()
  tool.promise = promise
  void promise.finally(() => void this.processQueue())
}
```

#### 4. 结果获取流程 (`getCompletedResults` / `getRemainingResults`)

```typescript
*getCompletedResults(): Generator<MessageUpdate, void> {
  for (const tool of this.tools) {
    // 1. 优先 yield 进度消息（无论工具状态）
    while (tool.pendingProgress.length > 0) {
      yield { message: tool.pendingProgress.shift()!, newContext: this.toolUseContext }
    }
    
    // 2. 已 yield 过的跳过
    if (tool.status === 'yielded') continue
    
    // 3. 完成的工具 yield 结果
    if (tool.status === 'completed' && tool.results) {
      tool.status = 'yielded'
      for (const message of tool.results) {
        yield { message, newContext: this.toolUseContext }
      }
      markToolUseAsComplete(this.toolUseContext, tool.id)
    } else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
      break  // 遇到执行中的非并发安全工具，保持顺序等待
    }
  }
}
```

### 数据结构

#### ToolStatus 状态机

```
queued → executing → completed → yielded
  ↑         ↓
  └─────────┘ (abort/discard 时直接生成合成错误)
```

#### 合成错误类型

```typescript
type AbortReason = 'sibling_error' | 'user_interrupted' | 'streaming_fallback'

// sibling_error: 兄弟 Bash 工具错误导致
// user_interrupted: 用户按 ESC 或发送新消息
// streaming_fallback: 流式 fallback 场景丢弃
```

---

## 关键代码路径与文件引用

### 核心文件依赖

| 文件路径 | 用途 |
|---------|------|
| `src/services/tools/toolExecution.ts` | `runToolUse` 函数，实际执行单个工具 |
| `src/services/tools/toolHooks.ts` | 工具钩子执行（PreToolUse/PostToolUse） |
| `src/Tool.ts` | `Tool` 类型定义、`findToolByName`、并发安全判断 |
| `src/hooks/useCanUseTool.tsx` | `CanUseToolFn` 类型，权限检查 |
| `src/utils/abortController.ts` | `createChildAbortController` 实现 |
| `src/utils/messages.ts` | `createUserMessage`、`REJECT_MESSAGE` 等 |
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` 常量 |

### 调用关系

```
query.ts (主查询循环)
    └── StreamingToolExecutor
            ├── addTool() ← 每收到一个 tool_use block 调用
            ├── getCompletedResults() ← 非阻塞获取已完成结果
            └── getRemainingResults() ← 等待所有工具完成
                    └── processQueue()
                            └── executeTool()
                                    └── runToolUse() (toolExecution.ts)
                                            └── checkPermissionsAndCallTool()
                                                    └── runPreToolUseHooks() (toolHooks.ts)
                                                    └── tool.call()
                                                    └── runPostToolUseHooks() (toolHooks.ts)
```

---

## 依赖与外部交互

### 1. 与 ToolExecution 的交互

`StreamingToolExecutor` 通过 `runToolUse` 函数委托实际工具执行：

```typescript
const generator = runToolUse(
  tool.block,
  tool.assistantMessage,
  this.canUseTool,
  { ...this.toolUseContext, abortController: toolAbortController },
)
```

`runToolUse` 返回 AsyncGenerator，支持流式产出结果更新。

### 2. 与 AbortController 的交互

使用 WeakRef 实现的父子级联取消机制：

```typescript
// 父 abort → 子 abort（通过 WeakRef 避免内存泄漏）
const weakChild = new WeakRef(child)
const weakParent = new WeakRef(parent)
const handler = propagateAbort.bind(weakParent, weakChild)
parent.signal.addEventListener('abort', handler, { once: true })
```

### 3. 与 ToolUseContext 的交互

- `setInProgressToolUseIDs`: 跟踪正在执行的工具 ID（用于 UI 显示）
- `setHasInterruptibleToolInProgress`: 标记是否有可中断工具在执行
- `toolUseContext` 可能被 context modifiers 更新

### 4. 与权限系统的交互

通过 `canUseTool` 回调（来自 `useCanUseTool` hook）检查工具使用权限。

---

## 风险、边界与改进建议

### 已知风险

#### 1. Context Modifier 并发限制

```typescript
// NOTE: we currently don't support context modifiers for concurrent
//       tools. None are actively being used, but if we want to use
//       them in concurrent tools, we need to support that here.
if (!tool.isConcurrencySafe && contextModifiers.length > 0) {
  for (const modifier of contextModifiers) {
    this.toolUseContext = modifier(this.toolUseContext)
  }
}
```

**风险**：并发工具的 context modifiers 被忽略，可能导致状态不一致。

#### 2. 非并发工具的顺序阻塞

```typescript
if (tool.status === 'executing' && !tool.isConcurrencySafe) {
  break  // 停止处理后续工具
}
```

**风险**：如果一个非并发工具长时间执行，会阻塞队列中后续所有工具。

#### 3. 兄弟错误只针对 Bash

```typescript
if (tool.block.name === BASH_TOOL_NAME) {
  this.siblingAbortController.abort('sibling_error')
}
```

**风险**：其他类型工具的隐式依赖链错误不会触发级联取消。

### 边界情况

| 场景 | 处理逻辑 |
|-----|---------|
| 工具不存在 | 直接标记 completed，生成错误结果 |
| 输入解析失败 | `isConcurrencySafe` 设为 false |
| 用户中断（ESC） | 根据 `interruptBehavior` 决定 cancel 或 block |
| 权限对话框拒绝 | abort 冒泡到父级，结束 turn |
| 流式 fallback | `discard()` 后所有操作变为 no-op |

### 改进建议

1. **支持并发工具的 Context Modifiers**
   - 需要设计线程安全的 context 更新机制
   - 可能需要引入锁或原子更新

2. **优化非并发工具的队列阻塞**
   - 考虑引入超时机制
   - 或允许某些非并发工具在特定条件下并行

3. **扩展兄弟错误机制**
   - 允许工具声明依赖关系
   - 基于依赖图而非仅工具类型进行级联取消

4. **增强可观测性**
   - 添加更多调试日志记录队列状态变化
   - 暴露工具执行时间指标

5. **内存优化**
   - `pendingProgress` 数组在工具完成后未清理
   - 考虑在 `yielded` 后释放不再需要的引用
