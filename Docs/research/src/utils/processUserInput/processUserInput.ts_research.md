# processUserInput.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`processUserInput.ts` 是 Claude Code **用户输入处理的核心编排器**，负责将原始用户输入（文本、图片、粘贴内容等）转换为结构化的消息流，支持：
- 普通文本提示处理
- Slash 命令路由
- Bash 命令处理
- 图片粘贴与处理
- 附件提取（@提及文件、Agent 提及等）
- Ultraplan 关键词路由
- Bridge/远程控制安全过滤

### 1.2 使用场景
| 场景 | 说明 |
|------|------|
| 普通对话 | 用户输入文本，直接发送到 LLM |
| Slash 命令 | `/command` 格式的命令路由到对应处理器 |
| Bash 命令 | `bash` 模式的命令行输入 |
| 图片粘贴 | 用户粘贴截图或图片文件 |
| IDE 集成 | 接收 IDE 选择的代码片段 |
| Bridge 远程控制 | 来自移动/Web 客户端的输入 |
| Ultraplan 触发 | 检测到特定关键词触发 CCR 会话 |

### 1.3 在架构中的位置
```
用户输入（键盘/IDE/Bridge/粘贴）
    ↓
QueuedCommand / 直接调用
    ↓
processUserInput() ← 本文件主入口
    ↓
processUserInputBase() ← 核心处理逻辑
    ↓
    ├─→ Bash 命令处理 (processBashCommand)
    ├─→ Slash 命令处理 (processSlashCommand)
    ├─→ Ultraplan 路由
    └─→ 普通文本处理 (processTextPrompt)
    ↓
UserPromptSubmit Hooks 执行
    ↓
返回消息列表 → query() → LLM API
```

---

## 2. 功能点目的

### 2.1 输入标准化与预处理
- **目的**：统一处理字符串输入和 ContentBlock 数组输入
- **关键处理**：
  - 图片块提取与尺寸调整 (`maybeResizeAndDownsampleImageBlock`)
  - 输入字符串提取（从数组中获取最后一个文本块）
  - 前置内容块跟踪（用于保留上下文）

### 2.2 图片处理流水线
- **存储**：`storeImages()` 将图片持久化到磁盘供后续引用
- **尺寸调整**：并行处理所有粘贴图片，确保符合 API 限制
- **元数据收集**：记录原始尺寸、调整后尺寸、源路径

### 2.3 Bridge 安全过滤
- **目的**：防止远程控制（Bridge）输入执行危险命令
- **实现**：
  - `skipSlashCommands`： blanket 阻止所有 Slash 命令
  - `isBridgeSafeCommand()`：允许列表检查
  - 对不安全命令返回友好错误而非暴露给模型

### 2.4 Ultraplan 关键词路由
- **目的**：检测用户输入中的 CCR（Claude Code Remote）触发词
- **条件**：
  - 功能开关 `ULTRAPLAN` 开启
  - 交互式 prompt 模式
  - 非 Slash 命令
  - 未处于已有 CCR 会话中
  - 使用 `preExpansionInput` 检测（防止粘贴内容误触发）
- **行为**：重写输入并路由到 `/ultraplan` 命令

### 2.5 附件提取
- **目的**：从用户输入中提取 @提及文件、Agent 提及等
- **条件控制**：
  - Slash 命令模式下跳过（在命令内部处理）
  - `skipAttachments` 标志控制
- **输出**：`AttachmentMessage[]` 追加到消息列表

### 2.6 UserPromptSubmit Hooks 执行
- **目的**：在查询前执行用户定义的钩子
- **执行时机**：`processUserInputBase` 之后，返回结果之前
- **处理能力**：
  - 阻塞错误处理
  - 阻止继续执行
  - 添加上下文附件
  - 输出截断（`MAX_HOOK_OUTPUT_LENGTH = 10000`）

---

## 3. 具体技术实现

### 3.1 主入口函数 `processUserInput`

```typescript
export async function processUserInput({
  input,
  preExpansionInput,      // 粘贴扩展前的原始输入
  mode,                   // 'prompt' | 'bash' | 'orphaned-permission' | 'task-notification'
  setToolJSX,            // 工具 JSX 渲染回调
  context,               // ToolUseContext & LocalJSXCommandContext
  pastedContents,        // 粘贴内容映射（图片等）
  ideSelection,          // IDE 选择的代码
  messages,              // 历史消息
  setUserInputOnProcessing, // UI 回调显示处理中输入
  uuid,
  isAlreadyProcessing,
  querySource,           // 查询来源（用于分析）
  canUseTool,            // 工具使用权限检查
  skipSlashCommands,     // 禁用 Slash 命令（Bridge 输入）
  bridgeOrigin,          // 来自 Bridge 的输入
  isMeta,                // 系统生成的提示
  skipAttachments,       // 跳过附件提取
}: {...}): Promise<ProcessUserInputBaseResult>
```

#### 关键流程步骤：
1. **UI 即时反馈**：非 `isMeta` 模式下立即显示用户输入
2. **性能检查点**：`queryCheckpoint('query_process_user_input_base_start')`
3. **调用 Base 处理**：`processUserInputBase()`
4. **Hooks 执行**：遍历 `executeUserPromptSubmitHooks`
5. **结果处理**：根据 hook 结果修改返回消息

### 3.2 核心处理函数 `processUserInputBase`

#### 3.2.1 输入解析与图片处理
```typescript
// 输入类型分支
if (typeof input === 'string') {
  inputString = input
} else if (input.length > 0) {
  // 处理每个图片块：调整尺寸、收集元数据
  for (const block of input) {
    if (block.type === 'image') {
      const resized = await maybeResizeAndDownsampleImageBlock(block)
      if (resized.dimensions) {
        imageMetadataTexts.push(createImageMetadataText(resized.dimensions))
      }
      processedBlocks.push(resized.block)
    }
  }
  // 提取输入字符串（最后一个文本块）
  const lastBlock = processedBlocks[processedBlocks.length - 1]
  if (lastBlock?.type === 'text') {
    inputString = lastBlock.text
    precedingInputBlocks = processedBlocks.slice(0, -1)
  }
}
```

#### 3.2.2 粘贴图片处理（并行）
```typescript
const imageProcessingResults = await Promise.all(
  imageContents.map(async pastedImage => {
    const imageBlock: ImageBlockParam = {
      type: 'image',
      source: {
        type: 'base64',
        media_type: (pastedImage.mediaType || 'image/png') as Base64ImageSource['media_type'],
        data: pastedImage.content,
      },
    }
    logEvent('tengu_pasted_image_resize_attempt', {
      original_size_bytes: pastedImage.content.length,
    })
    const resized = await maybeResizeAndDownsampleImageBlock(imageBlock)
    return { resized, originalDimensions: pastedImage.dimensions, sourcePath }
  }),
)
```

#### 3.2.3 Bridge 安全处理
```typescript
let effectiveSkipSlash = skipSlashCommands
if (bridgeOrigin && inputString !== null && inputString.startsWith('/')) {
  const parsed = parseSlashCommand(inputString)
  const cmd = parsed ? findCommand(parsed.commandName, context.options.commands) : undefined
  if (cmd) {
    if (isBridgeSafeCommand(cmd)) {
      effectiveSkipSlash = false  // 允许执行
    } else {
      // 返回错误，不暴露给模型
      const msg = `/${getCommandName(cmd)} isn't available over Remote Control.`
      return {
        messages: [
          createUserMessage({ content: inputString, uuid }),
          createCommandInputMessage(`<local-command-stdout>${msg}</local-command-stdout>`),
        ],
        shouldQuery: false,
        resultText: msg,
      }
    }
  }
  // 未知命令：作为纯文本处理（不报错）
}
```

#### 3.2.4 Ultraplan 路由
```typescript
if (
  feature('ULTRAPLAN') &&
  mode === 'prompt' &&
  !context.options.isNonInteractiveSession &&
  inputString !== null &&
  !effectiveSkipSlash &&
  !inputString.startsWith('/') &&
  !context.getAppState().ultraplanSessionUrl &&
  !context.getAppState().ultraplanLaunching &&
  hasUltraplanKeyword(preExpansionInput ?? inputString)
) {
  logEvent('tengu_ultraplan_keyword', {})
  const rewritten = replaceUltraplanKeyword(inputString).trim()
  const { processSlashCommand } = await import('./processSlashCommand.js')
  const slashResult = await processSlashCommand(
    `/ultraplan ${rewritten}`,
    precedingInputBlocks,
    imageContentBlocks,
    [],
    context,
    setToolJSX,
    uuid,
    isAlreadyProcessing,
    canUseTool,
  )
  return addImageMetadataMessage(slashResult, imageMetadataTexts)
}
```

#### 3.2.5 附件提取控制
```typescript
const shouldExtractAttachments =
  !skipAttachments &&
  inputString !== null &&
  (mode !== 'prompt' || effectiveSkipSlash || !inputString.startsWith('/'))

const attachmentMessages = shouldExtractAttachments
  ? await toArray(getAttachmentMessages(inputString, context, ideSelection ?? null, [], messages, querySource))
  : []
```

#### 3.2.6 模式路由
```typescript
// Bash 模式
if (inputString !== null && mode === 'bash') {
  const { processBashCommand } = await import('./processBashCommand.js')
  return addImageMetadataMessage(await processBashCommand(...), imageMetadataTexts)
}

// Slash 命令
if (inputString !== null && !effectiveSkipSlash && inputString.startsWith('/')) {
  const { processSlashCommand } = await import('./processSlashCommand.js')
  const slashResult = await processSlashCommand(...)
  return addImageMetadataMessage(slashResult, imageMetadataTexts)
}

// 普通提示
return addImageMetadataMessage(
  processTextPrompt(normalizedInput, imageContentBlocks, imagePasteIds, attachmentMessages, uuid, permissionMode, isMeta),
  imageMetadataTexts,
)
```

### 3.3 Hooks 执行与处理

```typescript
for await (const hookResult of executeUserPromptSubmitHooks(
  inputMessage,
  appState.toolPermissionContext.mode,
  context,
  context.requestPrompt,
)) {
  // 跳过进度消息
  if (hookResult.message?.type === 'progress') continue

  // 阻塞错误处理
  if (hookResult.blockingError) {
    const blockingMessage = getUserPromptSubmitHookBlockingMessage(hookResult.blockingError)
    return {
      messages: [createSystemMessage(`${blockingMessage}\n\nOriginal prompt: ${input}`, 'warning')],
      shouldQuery: false,
      allowedTools: result.allowedTools,
    }
  }

  // 阻止继续执行
  if (hookResult.preventContinuation) {
    result.messages.push(createUserMessage({ content: hookResult.stopReason || 'Operation stopped by hook' }))
    result.shouldQuery = false
    return result
  }

  // 收集额外上下文
  if (hookResult.additionalContexts?.length > 0) {
    result.messages.push(createAttachmentMessage({
      type: 'hook_additional_context',
      content: hookResult.additionalContexts.map(applyTruncation),
      hookName: 'UserPromptSubmit',
      toolUseID: `hook-${randomUUID()}`,
      hookEvent: 'UserPromptSubmit',
    }))
  }
}
```

### 3.4 输出截断
```typescript
const MAX_HOOK_OUTPUT_LENGTH = 10000

function applyTruncation(content: string): string {
  if (content.length > MAX_HOOK_OUTPUT_LENGTH) {
    return `${content.substring(0, MAX_HOOK_OUTPUT_LENGTH)}… [output truncated - exceeded ${MAX_HOOK_OUTPUT_LENGTH} characters]`
  }
  return content
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖模块

| 文件 | 用途 |
|------|------|
| `src/utils/messages.ts` | `createUserMessage`, `createCommandInputMessage`, `createSystemMessage`, `createAttachmentMessage`, `getContentText` |
| `src/utils/attachments.ts` | `getAttachmentMessages`, `createAttachmentMessage` |
| `src/utils/imageResizer.ts` | `maybeResizeAndDownsampleImageBlock`, `createImageMetadataText` |
| `src/utils/imageStore.ts` | `storeImages` |
| `src/utils/hooks.ts` | `executeUserPromptSubmitHooks`, `getUserPromptSubmitHookBlockingMessage` |
| `src/utils/slashCommandParsing.ts` | `parseSlashCommand` |
| `src/utils/queryProfiler.ts` | `queryCheckpoint` |
| `src/utils/ultraplan/keyword.ts` | `hasUltraplanKeyword`, `replaceUltraplanKeyword` |
| `src/commands.ts` | `findCommand`, `getCommandName`, `isBridgeSafeCommand` |
| `src/utils/generators.ts` | `toArray` |
| `src/services/analytics/index.ts` | `logEvent` |
| `src/types/message.ts` | 消息类型定义 |
| `src/types/permissions.ts` | `PermissionMode` |
| `src/types/textInputTypes.ts` | `PromptInputMode`, `isValidImagePaste` |

### 4.2 动态导入（懒加载）
```typescript
// Bash 命令处理
const { processBashCommand } = await import('./processBashCommand.js')

// Slash 命令处理
const { processSlashCommand } = await import('./processSlashCommand.js')
```

### 4.3 代码行号参考

| 功能 | 行号范围 |
|------|----------|
| 类型定义 `ProcessUserInputContext` | 62 行 |
| 类型定义 `ProcessUserInputBaseResult` | 64-83 行 |
| 主入口 `processUserInput` | 85-270 行 |
| 输出截断 `applyTruncation` | 272-279 行 |
| 核心处理 `processUserInputBase` | 281-605 行 |
| Bridge 安全处理 | 422-453 行 |
| Ultraplan 路由 | 467-493 行 |
| 附件提取 | 496-514 行 |
| Bash 命令路由 | 517-529 行 |
| Slash 命令路由 | 532-551 行 |
| Agent 提及日志 | 553-574 行 |
| 图片元数据追加 | 592-605 行 |

---

## 5. 依赖与外部交互

### 5.1 外部库依赖
```typescript
import { feature } from 'bun:bundle'
import type { ContentBlockParam, ImageBlockParam, Base64ImageSource } from '@anthropic-ai/sdk/resources/messages.mjs'
import { randomUUID } from 'crypto'
```

### 5.2 核心类型定义

#### 5.2.1 ProcessUserInputContext
```typescript
export type ProcessUserInputContext = ToolUseContext & LocalJSXCommandContext
```

#### 5.2.2 ProcessUserInputBaseResult
```typescript
export type ProcessUserInputBaseResult = {
  messages: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage | ProgressMessage)[]
  shouldQuery: boolean
  allowedTools?: string[]
  model?: string
  effort?: EffortValue
  resultText?: string        // 非交互模式输出
  nextInput?: string         // 链式输入（如 /discover）
  submitNextInput?: boolean  // 自动提交下一个输入
}
```

### 5.3 状态与副作用
- **性能追踪**：多处调用 `queryCheckpoint()` 记录处理阶段
- **分析事件**：`logEvent()` 上报关键指标（图片调整、Agent 提及等）
- **动态导入**：条件加载 Bash/Slash 命令处理器
- **Hook 执行**：可能触发外部命令执行

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 图片处理内存风险
```typescript
const imageProcessingResults = await Promise.all(
  imageContents.map(async pastedImage => { ... })
)
```
- **风险**：大量/大图片并行处理可能导致内存峰值
- **缓解**：`maybeResizeAndDownsampleImageBlock` 内部有尺寸限制，但并行度无上限

#### 6.1.2 Bridge 安全绕过风险
```typescript
// 未知命令作为纯文本处理
// Unknown /foo or unparseable — fall through to plain text
```
- **风险**：精心构造的命令可能绕过安全检查
- **缓解**：仅允许已知命令执行，未知命令作为文本不执行

#### 6.1.3 Hook 执行阻塞
```typescript
for await (const hookResult of executeUserPromptSubmitHooks(...)) {
  // 同步等待所有 hooks 完成
}
```
- **风险**：慢 hook 会阻塞用户查询
- **缓解**：有超时机制（`TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10min`），但仍可能感知延迟

#### 6.1.4 输入验证不足
```typescript
if (inputString === null && mode !== 'prompt') {
  throw new Error(`Mode: ${mode} requires a string input.`)
}
```
- **风险**：仅检查 null，无格式验证
- **缓解**：下游处理负责验证

### 6.2 边界情况

| 场景 | 处理行为 |
|------|----------|
| 空输入数组 | `inputString` 保持 null，可能抛出错误 |
| 图片处理失败 | `maybeResizeAndDownsampleImageBlock` 内部处理，可能抛出 |
| Hook 返回阻塞错误 | 返回系统消息，阻止查询 |
| Hook 阻止继续 | 添加用户消息说明原因，`shouldQuery = false` |
| 附件提取超时 | `getAttachmentMessages` 内部有 1 秒超时 |
| Bridge + 安全命令 | 允许执行，正常处理 |
| Bridge + 不安全命令 | 返回错误消息，不查询 LLM |

### 6.3 改进建议

#### 6.3.1 图片处理限流
```typescript
// 建议：限制并发图片处理数量
import pLimit from 'p-limit'
const limit = pLimit(5)
const imageProcessingResults = await Promise.all(
  imageContents.map(pastedImage => limit(() => processImage(pastedImage)))
)
```

#### 6.3.2 更细粒度的 Bridge 安全
```typescript
// 建议：对未知命令增加确认步骤
if (bridgeOrigin && !cmd) {
  return {
    messages: [createSystemMessage('Unknown command from remote source blocked', 'warning')],
    shouldQuery: false,
  }
}
```

#### 6.3.3 Hook 结果聚合优化
```typescript
// 当前：顺序执行
for await (const hookResult of executeUserPromptSubmitHooks(...))

// 建议：并行执行（如果 hooks 无依赖）
const hookResults = await Promise.all(
  hooks.map(hook => executeHook(hook))
)
```

#### 6.3.4 输入验证增强
```typescript
// 建议：增加对 input 数组的内容验证
function validateContentBlocks(blocks: ContentBlockParam[]): void {
  const validTypes = ['text', 'image', 'tool_use', 'tool_result']
  for (const block of blocks) {
    if (!validTypes.includes(block.type)) {
      throw new Error(`Invalid content block type: ${block.type}`)
    }
  }
}
```

#### 6.3.5 错误处理细化
```typescript
// 当前：统一捕获
} catch (error) {
  logError(error)
}

// 建议：分类处理
} catch (error) {
  if (error instanceof ImageResizeError) {
    // 返回用户友好的图片错误
  } else if (error instanceof PermissionError) {
    // 返回权限错误
  } else {
    // 通用错误处理
  }
}
```

### 6.4 测试建议

应覆盖以下场景：

#### 6.4.1 输入类型
- 纯字符串输入（各种长度、特殊字符）
- ContentBlock 数组（纯文本、图文混合、纯图片）
- 空输入、null 输入

#### 6.4.2 图片处理
- 单图片、多图片
- 大图片（需要调整尺寸）
- 无效图片数据
- 图片 + 文本组合

#### 6.4.3 模式路由
- Bash 模式
- Slash 命令（内置、技能、插件）
- 普通 prompt
- 各种 mode 组合

#### 6.4.4 Bridge 安全
- Bridge 源 + 安全命令
- Bridge 源 + 不安全命令
- Bridge 源 + 未知命令
- 非 Bridge 源 + 任意命令

#### 6.4.5 Hooks
- Hook 成功
- Hook 返回阻塞错误
- Hook 阻止继续
- Hook 添加上下文
- 多 hooks 组合

#### 6.4.6 Ultraplan
- 关键词检测（各种变体）
- 粘贴内容包含关键词（不应触发）
- 已有 CCR 会话时的行为

---

## 7. 架构设计亮点

### 7.1 分层处理架构
```
processUserInput (编排层)
    ↓
processUserInputBase (核心处理层)
    ↓
process{Bash,Slash,Text}Prompt (专用处理器)
```

### 7.2 安全设计
- Bridge 输入的显式安全过滤
- 未知命令的保守处理（作为文本）
- Hook 执行的信任检查前置

### 7.3 性能优化
- 图片并行处理
- 动态导入减少启动时间
- 检查点埋点支持性能分析

### 7.4 可扩展性
- Hook 机制支持自定义逻辑注入
- 附件系统的模块化设计
- Slash 命令的动态发现

---

*文档生成时间：2026-04-01*
*研究范围：src/utils/processUserInput/processUserInput.ts 及其直接依赖*
