# processTextPrompt.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`processTextPrompt.ts` 是 Claude Code 用户输入处理流程中的**最终标准化层**，负责将各种格式的用户输入（纯文本或 ContentBlock 数组）转换为统一的消息结构，供后续 LLM 查询使用。

### 1.2 使用场景
- **常规文本输入**：用户通过 CLI 输入的普通文本提示
- **多模态输入**：包含图片粘贴的复合输入（文本 + 图片 ContentBlock）
- **SDK/VS Code 集成**：通过数组形式的 ContentBlockParam 传入的复杂输入
- **附件消息处理**：将附件消息合并到最终消息列表中

### 1.3 在架构中的位置
```
用户输入 → processUserInput.ts → processUserInputBase() → processTextPrompt()
                                              ↓
                                    [图片处理/附件提取/Slash命令]
                                              ↓
                                    processTextPrompt() ← 本文件
                                              ↓
                                    返回标准化消息列表 → LLM 查询
```

---

## 2. 功能点目的

### 2.1 Prompt ID 生成与追踪
- **目的**：为每个用户提示生成唯一标识符，用于全链路追踪
- **实现**：使用 `randomUUID()` 生成 UUID，通过 `setPromptId()` 存入全局状态
- **消费**：OTel 事件、分析日志、会话追踪

### 2.2 交互 Span 启动
- **目的**：标记用户交互的开始，用于性能追踪和遥测
- **实现**：调用 `startInteractionSpan(userPromptText)` 启动 OpenTelemetry Span
- **特殊性**：同时支持字符串输入（CLI）和数组输入（SDK/VS Code）

### 2.3 OTel 事件上报
- **目的**：记录用户提示用于分析（需显式开启）
- **实现**：
  - 字符串输入：直接使用完整文本
  - 数组输入：使用 `findLast` 获取最后一个文本块（实际用户提示）
- **隐私控制**：通过 `redactIfDisabled()` 在未启用时返回 `<REDACTED>`

### 2.4 关键词检测与埋点
- **负面关键词检测**：`matchesNegativeKeyword()`
  - 检测用户沮丧情绪（wtf, shit, damn it 等）
  - 用于产品分析，了解用户痛点
  
- **继续关键词检测**：`matchesKeepGoingKeyword()`
  - 检测 "continue", "keep going", "go on"
  - 用于分析用户是否在使用"继续"功能

### 2.5 图片内容处理
- **场景**：用户粘贴图片时的复合消息构建
- **实现**：将文本内容和图片 ContentBlock 合并为数组形式的消息内容
- **元数据**：记录 `imagePasteIds` 用于后续图片引用

### 2.6 消息创建与返回
- **核心输出**：`UserMessage` 或复合消息数组
- **附件合并**：将附件消息追加到用户消息之后
- **元数据标记**：支持 `isMeta`（系统生成提示）、`permissionMode` 等

---

## 3. 具体技术实现

### 3.1 函数签名
```typescript
export function processTextPrompt(
  input: string | Array<ContentBlockParam>,
  imageContentBlocks: ContentBlockParam[],
  imagePasteIds: number[],
  attachmentMessages: AttachmentMessage[],
  uuid?: string,
  permissionMode?: PermissionMode,
  isMeta?: boolean,
): {
  messages: (UserMessage | AttachmentMessage | SystemMessage)[]
  shouldQuery: boolean
}
```

### 3.2 关键流程

#### 3.2.1 Prompt ID 与 Span 启动
```typescript
const promptId = randomUUID()
setPromptId(promptId)

const userPromptText = typeof input === 'string' 
  ? input 
  : input.find(block => block.type === 'text')?.text || ''
startInteractionSpan(userPromptText)
```

#### 3.2.2 OTel 事件上报（关键修复）
```typescript
// 修复：之前仅对字符串输入上报，导致 VS Code 会话缺失 user_prompt 事件
const otelPromptText = typeof input === 'string'
  ? input
  : input.findLast(block => block.type === 'text')?.text || ''

if (otelPromptText) {
  void logOTelEvent('user_prompt', {
    prompt_length: String(otelPromptText.length),
    prompt: redactIfDisabled(otelPromptText),
    'prompt.id': promptId,
  })
}
```

**关键设计决策**：
- 使用 `findLast` 而非 `find`：因为 `createUserContent` 将用户消息放在最后（在 ide_selection/attachment 上下文块之后）
- `userPromptText`（第一个文本块）保持不变用于 `startInteractionSpan`，保持与现有 span 属性的兼容性

#### 3.2.3 图片消息构建
```typescript
if (imageContentBlocks.length > 0) {
  const textContent = typeof input === 'string'
    ? input.trim() ? [{ type: 'text' as const, text: input }] : []
    : input
  
  const userMessage = createUserMessage({
    content: [...textContent, ...imageContentBlocks],
    uuid,
    imagePasteIds: imagePasteIds.length > 0 ? imagePasteIds : undefined,
    permissionMode,
    isMeta: isMeta || undefined,
  })
  
  return {
    messages: [userMessage, ...attachmentMessages],
    shouldQuery: true,
  }
}
```

#### 3.2.4 纯文本消息构建
```typescript
const userMessage = createUserMessage({
  content: input,  // 可以是字符串或 ContentBlockParam[]
  uuid,
  permissionMode,
  isMeta: isMeta || undefined,
})

return {
  messages: [userMessage, ...attachmentMessages],
  shouldQuery: true,
}
```

### 3.3 数据结构

#### 3.3.1 输入类型
| 参数 | 类型 | 说明 |
|------|------|------|
| `input` | `string \| ContentBlockParam[]` | 用户输入，字符串（CLI）或数组（SDK） |
| `imageContentBlocks` | `ContentBlockParam[]` | 图片内容块，已预处理好 |
| `imagePasteIds` | `number[]` | 图片粘贴 ID 列表 |
| `attachmentMessages` | `AttachmentMessage[]` | 附件消息（@提及文件等） |
| `uuid` | `string?` | 可选消息 UUID |
| `permissionMode` | `PermissionMode?` | 权限模式 |
| `isMeta` | `boolean?` | 是否为系统生成的元消息 |

#### 3.3.2 输出类型
```typescript
{
  messages: (UserMessage | AttachmentMessage | SystemMessage)[]
  shouldQuery: boolean  // 固定返回 true
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.ts` | `setPromptId()` - 设置当前 prompt ID |
| `src/utils/messages.ts` | `createUserMessage()` - 创建用户消息对象 |
| `src/utils/userPromptKeywords.ts` | `matchesNegativeKeyword()`, `matchesKeepGoingKeyword()` - 关键词检测 |
| `src/utils/telemetry/events.ts` | `logOTelEvent()`, `redactIfDisabled()` - 遥测事件上报 |
| `src/utils/telemetry/sessionTracing.ts` | `startInteractionSpan()` - 启动交互追踪 |
| `src/services/analytics/index.ts` | `logEvent()` - 分析事件上报 |
| `src/types/permissions.ts` | `PermissionMode` 类型定义 |
| `src/types/message.ts` | `UserMessage`, `AttachmentMessage`, `SystemMessage` 类型 |

### 4.2 调用链
```
REPL.tsx / SDK 入口
    ↓
processUserInput() [processUserInput.ts]
    ↓
processUserInputBase() [processUserInput.ts]
    ↓ (对于普通文本提示)
processTextPrompt() [processTextPrompt.ts] ← 本文件
    ↓
返回消息列表 → query() → LLM API
```

### 4.3 代码行号参考
- **Prompt ID 生成**：第 31-32 行
- **OTel 事件上报**：第 40-57 行（含关键修复注释）
- **关键词检测与埋点**：第 59-64 行
- **图片消息构建**：第 67-87 行
- **纯文本消息构建**：第 89-99 行

---

## 5. 依赖与外部交互

### 5.1 外部库依赖
```typescript
import type { ContentBlockParam } from '@anthropic-ai/sdk/resources'
import { randomUUID } from 'crypto'
```

### 5.2 内部模块依赖
```typescript
import { setPromptId } from 'src/bootstrap/state.js'
import type { AttachmentMessage, SystemMessage, UserMessage } from 'src/types/message.js'
import { logEvent } from '../../services/analytics/index.js'
import type { PermissionMode } from '../../types/permissions.js'
import { createUserMessage } from '../messages.js'
import { logOTelEvent, redactIfDisabled } from '../telemetry/events.js'
import { startInteractionSpan } from '../telemetry/sessionTracing.js'
import { matchesKeepGoingKeyword, matchesNegativeKeyword } from '../userPromptKeywords.js'
```

### 5.3 状态与副作用
- **全局状态写入**：`setPromptId()` 修改全局 prompt ID
- **遥测追踪**：`startInteractionSpan()` 启动 OTel Span
- **异步事件上报**：`logOTelEvent()` 和 `logEvent()` 为异步调用（使用 `void` 忽略 Promise）

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 OTel 事件上报失败静默
```typescript
void logOTelEvent('user_prompt', { ... })  // 错误被静默忽略
```
- **风险**：上报失败无感知，可能导致分析数据丢失
- **缓解**：该设计有意为之，避免阻塞用户交互流程

#### 6.1.2 空图片内容处理
```typescript
if (imageContentBlocks.length > 0) {
  // 假设所有图片内容都有效，无空内容检查
}
```
- **风险**：传入空内容图片块可能导致 API 错误
- **缓解**：前置处理（processUserInputBase 中的 `isValidImagePaste` 检查）

#### 6.1.3 输入类型不一致
```typescript
const userPromptText = typeof input === 'string'
  ? input
  : input.find(block => block.type === 'text')?.text || ''
```
- **风险**：数组输入无文本块时返回空字符串，可能导致 span 属性为空
- **缓解**：空字符串是有效值，不影响功能

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 空字符串输入 | 正常处理，创建空内容消息 |
| 数组输入无文本块 | `userPromptText` 为空字符串，OTel 不上报 |
| 图片数组为空 | 走纯文本路径 |
| `isMeta` 为 true | 标记消息为系统生成，UI 隐藏显示 |
| `uuid` 未提供 | `createUserMessage` 内部生成新 UUID |

### 6.3 改进建议

#### 6.3.1 增加输入校验
```typescript
// 建议：增加对 imageContentBlocks 内容格式的校验
if (imageContentBlocks.some(block => !block.source?.data)) {
  logForDebugging('Invalid image content block detected')
}
```

#### 6.3.2 统一文本提取逻辑
当前有两处文本提取逻辑（`userPromptText` 和 `otelPromptText`），虽然设计意图不同，但容易混淆：
```typescript
// 建议：添加注释说明差异，或提取为带明确意图的函数
function getTextForSpan(input: ...)  // 用于 span 属性
function getTextForOtel(input: ...)  // 用于 OTel 事件（最后一个文本块）
```

#### 6.3.3 错误上报增强
```typescript
// 当前：静默忽略
void logOTelEvent('user_prompt', { ... })

// 建议：至少记录到调试日志
try {
  await logOTelEvent('user_prompt', { ... })
} catch (e) {
  logForDebugging('Failed to log user_prompt event', e)
}
```

#### 6.3.4 类型安全增强
`ContentBlockParam` 是联合类型，当前代码假设数组输入时存在 `type` 字段：
```typescript
// 建议：增加类型守卫
function isTextBlock(block: ContentBlockParam): block is TextBlockParam {
  return block.type === 'text'
}
```

### 6.4 测试建议

应覆盖以下场景：
1. **字符串输入**：常规文本、空字符串、超长文本
2. **数组输入**：纯文本块、图文混合、无文本块、多文本块
3. **图片处理**：单图、多图、图文混合
4. **附件合并**：空附件列表、多附件
5. **元数据传递**：`isMeta`、`permissionMode`、`uuid` 正确传递
6. **副作用验证**：`setPromptId`、`startInteractionSpan`、`logOTelEvent` 被正确调用

---

## 7. 历史变更记录

### 7.1 关键修复（代码注释中提及）
- **Issue #33301**：修复 VS Code 会话未上报 `user_prompt` 事件的问题
  - 原代码：`typeof input === 'string'` 门控导致数组输入跳过上报
  - 修复：对数组输入使用 `findLast` 获取最后一个文本块

---

*文档生成时间：2026-04-01*
*研究范围：src/utils/processUserInput/processTextPrompt.ts 及其直接依赖*
