# textInputTypes.ts 研究文档

## 场景与职责

`src/types/textInputTypes.ts` 是 Claude Code CLI 的文本输入系统核心类型定义文件。它定义了 REPL（Read-Eval-Print Loop）中所有文本输入组件的 Props 和 State 类型，支持：

1. **多模式文本输入**: 普通文本输入、Vim 模式输入、带历史导航的输入
2. **粘贴内容处理**: 支持图片粘贴、大文本粘贴、粘贴状态管理
3. **命令队列系统**: 定义队列优先级（now/next/later）和排队命令结构
4. **提示输入模式**: bash 模式、prompt 模式、孤立权限模式、任务通知模式
5. **内联自动补全**: Ghost text 支持命令中途自动补全

该文件是 REPL 交互系统的类型基础，被 25+ 文件依赖，直接影响用户的输入体验。

## 功能点目的

### 1. 基础文本输入 Props（BaseTextInputProps）
定义所有文本输入组件共享的 30+ 个属性：
- **值管理**: value, onChange, onSubmit
- **历史导航**: onHistoryUp, onHistoryDown, onHistoryReset
- **光标控制**: cursorOffset, onChangeCursorOffset, showCursor
- **粘贴处理**: onImagePaste, onPaste, onIsPastingChange, highlightPastedText
- **显示控制**: placeholder, placeholderElement, multiline, mask, dimColor
- **特殊功能**: onUndo, onExit, onExitMessage, onClearInput
- **自动补全**: inlineGhostText, argumentHint
- **输入过滤**: inputFilter（用于键路由前的输入转换）

### 2. Vim 模式支持（VimTextInputProps）
扩展基础 Props，增加：
- `initialMode`: 初始 Vim 模式（INSERT/NORMAL）
- `onModeChange`: 模式变更回调

### 3. 输入状态管理（BaseInputState）
定义输入 hook 返回的完整状态：
- **输入处理**: onInput 回调
- **渲染状态**: renderedValue（可能包含粘贴占位符）
- **光标位置**: offset, cursorLine, cursorColumn
- **视口管理**: viewportCharOffset, viewportCharEnd（用于大文本窗口化）
- **粘贴状态**: isPasting, pasteState（分块粘贴管理）

### 4. 提示输入模式（PromptInputMode）
定义四种输入模式：
- `bash`: Bash 命令输入
- `prompt`: 普通提示输入（默认）
- `orphaned-permission`: 孤立权限处理（权限提示但对应消息丢失）
- `task-notification`: 任务通知输入

`EditablePromptInputMode` 排除通知模式，用于可编辑输入场景。

### 5. 队列优先级系统（QueuePriority）
定义三种优先级，语义在普通模式和主动模式下相同：
- `now`: 立即中断发送，中止进行中的工具调用（相当于 Esc + 发送）
- `next`: 当前工具完成后发送（turn 间 drain）
- `later`: 当前 turn 完成后发送（end-of-turn drain）

### 6. 排队命令（QueuedCommand）
定义进入队列的命令结构，包含 15+ 字段：
- **内容**: value（字符串或 ContentBlockParam 数组）
- **模式**: mode（PromptInputMode）
- **优先级**: priority（覆盖模式默认优先级）
- **标识**: uuid（用于追踪）
- **权限**: orphanedPermission（孤立权限关联）
- **粘贴内容**: pastedContents（图片等大内容）
- **预处理**: preExpansionValue（粘贴占位符展开前的原始输入）
- **来源控制**: skipSlashCommands（禁止触发斜杠命令）、bridgeOrigin（桥接来源标记）
- **元数据**: isMeta（模型可见但 UI 隐藏）、origin（消息来源）、workload（计费标签）
- **Agent 路由**: agentId（指定接收 Agent）

### 7. 图片粘贴工具函数
- `isValidImagePaste()`: 验证粘贴内容是否为有效图片（非空 base64）
- `getImagePasteIds()`: 从 QueuedCommand 提取有效图片 ID 列表

## 具体技术实现

### 关键数据结构

```typescript
// 内联 Ghost Text（命令自动补全）
interface InlineGhostText {
  readonly text: string           // 显示的建议文本（如 "mit"）
  readonly fullCommand: string    // 完整命令名（如 "commit"）
  readonly insertPosition: number // 插入位置
}

// 基础文本输入 Props
interface BaseTextInputProps {
  // 值管理
  readonly value: string
  readonly onChange: (value: string) => void
  readonly onSubmit?: (value: string) => void
  
  // 历史导航
  readonly onHistoryUp?: () => void
  readonly onHistoryDown?: () => void
  readonly onHistoryReset?: () => void
  
  // 光标控制
  readonly cursorOffset: number
  onChangeCursorOffset: (offset: number) => void
  readonly showCursor?: boolean
  
  // 粘贴处理
  readonly onImagePaste?: (
    base64Image: string,
    mediaType?: string,
    filename?: string,
    dimensions?: ImageDimensions,
    sourcePath?: string,
  ) => void
  readonly onPaste?: (text: string) => void
  readonly onIsPastingChange?: (isPasting: boolean) => void
  readonly highlightPastedText?: boolean
  
  // 显示控制
  readonly placeholder?: string
  readonly placeholderElement?: React.ReactNode
  readonly multiline?: boolean
  readonly mask?: string
  readonly dimColor?: boolean
  
  // 特殊功能
  readonly onUndo?: () => void
  readonly onExit?: () => void
  readonly onClearInput?: () => void
  
  // 自动补全
  readonly inlineGhostText?: InlineGhostText
  readonly argumentHint?: string
  
  // 输入过滤
  readonly inputFilter?: (input: string, key: Key) => string
  
  // ... 更多字段
}

// Vim 模式输入 Props
interface VimTextInputProps extends BaseTextInputProps {
  readonly initialMode?: VimMode
  readonly onModeChange?: (mode: VimMode) => void
}

type VimMode = 'INSERT' | 'NORMAL'

// 输入状态
interface BaseInputState {
  onInput: (input: string, key: Key) => void
  renderedValue: string
  offset: number
  setOffset: (offset: number) => void
  cursorLine: number           // 光标所在行（0 索引，考虑换行）
  cursorColumn: number         // 光标所在列（显示宽度）
  viewportCharOffset: number   // 视口起始字符偏移
  viewportCharEnd: number      // 视口结束字符偏移
  isPasting?: boolean
  pasteState?: {
    chunks: string[]
    timeoutId: ReturnType<typeof setTimeout> | null
  }
}
```

### 队列系统

```typescript
// 队列优先级
type QueuePriority = 'now' | 'next' | 'later'

// 排队命令
interface QueuedCommand {
  value: string | Array<ContentBlockParam>
  mode: PromptInputMode
  priority?: QueuePriority       // 覆盖模式默认优先级
  uuid?: UUID
  orphanedPermission?: OrphanedPermission
  pastedContents?: Record<number, PastedContent>
  preExpansionValue?: string     // 粘贴占位符展开前的原始输入
  skipSlashCommands?: boolean    // 禁止触发斜杠命令
  bridgeOrigin?: boolean         // 桥接来源（移动/网页客户端）
  isMeta?: boolean               // 模型可见但 UI 隐藏
  origin?: MessageOrigin         // 消息来源
  workload?: string              // 计费标签
  agentId?: AgentId              // 指定接收 Agent
}

// 孤立权限（权限提示但对应消息丢失）
interface OrphanedPermission {
  permissionResult: PermissionResult
  assistantMessage: AssistantMessage
}
```

### 工具函数

```typescript
// 验证图片粘贴有效性（非空 base64）
export function isValidImagePaste(c: PastedContent): boolean {
  return c.type === 'image' && c.content.length > 0
}

// 提取图片粘贴 ID 列表
export function getImagePasteIds(
  pastedContents: Record<number, PastedContent> | undefined,
): number[] | undefined {
  if (!pastedContents) return undefined
  const ids = Object.values(pastedContents)
    .filter(isValidImagePaste)
    .map(c => c.id)
  return ids.length > 0 ? ids : undefined
}
```

## 关键代码路径与文件引用

### 类型定义
- `src/types/textInputTypes.ts` - 本文件，核心类型定义

### 输入组件
- `src/components/TextInput.tsx` - 普通文本输入组件
- `src/components/VimTextInput.tsx` - Vim 模式输入组件
- `src/components/BaseTextInput.tsx` - 基础输入组件实现
- `src/components/PromptInput/PromptInput.tsx` - 提示输入主组件

### 输入 Hooks
- `src/hooks/useTextInput.ts` - 文本输入 hook
- `src/hooks/useVimInput.ts` - Vim 输入 hook
- `src/hooks/useHistorySearch.ts` - 历史搜索 hook
- `src/hooks/useArrowKeyHistory.tsx` - 箭头键历史导航
- `src/hooks/useInputBuffer.ts` - 输入缓冲

### 队列系统
- `src/hooks/useCommandQueue.ts` - 命令队列 hook
- `src/hooks/useQueueProcessor.ts` - 队列处理器
- `src/utils/queueProcessor.ts` - 队列处理逻辑
- `src/utils/messageQueueManager.ts` - 消息队列管理

### REPL 集成
- `src/screens/REPL.tsx` - REPL 主界面
- `src/QueryEngine.ts` - 查询引擎
- `src/utils/handlePromptSubmit.ts` - 提示提交处理
- `src/utils/processUserInput/processUserInput.ts` - 用户输入处理

### 附件处理
- `src/utils/attachments.ts` - 附件处理

### 状态栏
- `src/components/StatusLine.tsx` - 状态栏

### 提示输入子组件
- `src/components/PromptInput/PromptInputFooter.tsx` - 输入底部栏
- `src/components/PromptInput/PromptInputFooterLeftSide.tsx` - 底部栏左侧
- `src/components/PromptInput/PromptInputQueuedCommands.tsx` - 排队命令显示
- `src/components/PromptInput/PromptInputModeIndicator.tsx` - 模式指示器
- `src/components/PromptInput/inputModes.ts` - 输入模式工具

## 依赖与外部交互

### 导入依赖
```typescript
import type { ContentBlockParam } from '@anthropic-ai/sdk/resources/messages.mjs'  // Anthropic SDK
import type { UUID } from 'crypto'                                                   // Node.js crypto
import type React from 'react'                                                       // React
import type { PermissionResult } from '../entrypoints/agentSdkTypes.js'              // 权限结果
import type { Key } from '../ink.js'                                                 // Ink 键盘
import type { PastedContent } from '../utils/config.js'                              // 粘贴内容
import type { ImageDimensions } from '../utils/imageResizer.js'                      // 图片尺寸
import type { TextHighlight } from '../utils/textHighlighting.js'                    // 文本高亮
import type { AgentId } from './ids.js'                                              // Agent ID
import type { AssistantMessage, MessageOrigin } from './message.js'                  // 消息类型
```

### 被依赖方（25+ 文件）
主要分布：
- 输入组件（`src/components/*`）
- 输入 hooks（`src/hooks/*`）
- REPL 和查询引擎（`src/screens/REPL.tsx`, `src/QueryEngine.ts`）
- 队列系统（`src/utils/queueProcessor.ts`, `src/utils/messageQueueManager.ts`）

## 风险、边界与改进建议

### 潜在风险

1. **BaseTextInputProps 字段膨胀**
   - 30+ 个字段的接口难以维护
   - 部分字段互斥（如 placeholder 和 placeholderElement）
   - 建议按职责分组（显示、交互、粘贴等）

2. **QueuePriority 的语义复杂性**
   - `now` 会中止进行中的工具调用
   - 消费者需要监听队列变化并正确处理中止
   - 容易在并发场景下产生竞态条件

3. **preExpansionValue 的用途不明确**
   - 用于 ultraplan 关键字检测
   - 但注释说明桥接/UDS/MCP 来源无粘贴展开
   - 需要仔细处理 fallback 逻辑

4. ** pastedContents 的内存管理**
   - 图片内容以 base64 存储，可能占用大量内存
   - 需要确保在命令处理后及时释放

### 边界情况

1. **视口窗口化（Viewport Windowing）**
   - `viewportCharOffset` 和 `viewportCharEnd` 用于大文本
   - 当文本超过 `maxVisibleLines` 时只渲染视口内内容
   - 需要正确处理光标滚动和文本换行

2. **粘贴分块处理**
   - `pasteState.chunks` 存储分块粘贴的内容
   - `timeoutId` 用于合并快速连续的粘贴事件
   - 需要处理粘贴超时和清理

3. **skipSlashCommands 与 bridgeOrigin 的交互**
   - `skipSlashCommands`: 完全禁止斜杠命令
   - `bridgeOrigin`: 允许斜杠命令但过滤不安全的
   - 两者都用于远程来源，但语义不同

4. **isMeta 消息的处理**
   - `isMeta: true` 的消息对模型可见但 UI 隐藏
   - 用于系统生成的提示（proactive ticks, teammate 消息）
   - 需要确保不会意外丢失

5. **agentId 路由**
   - 子代理共享模块级命令队列
   - `agentId` 用于过滤，防止通知泄漏到协调器上下文
   - 需要确保队列处理器正确过滤

### 改进建议

1. **Props 分层**
   ```typescript
   // 建议：按职责分组
   interface TextInputDisplayProps { /* 显示相关 */ }
   interface TextInputInteractionProps { /* 交互相关 */ }
   interface TextInputPasteProps { /* 粘贴相关 */ }
   interface BaseTextInputProps extends 
     TextInputDisplayProps, 
     TextInputInteractionProps, 
     TextInputPasteProps {}
   ```

2. **队列优先级状态机**
   ```typescript
   // 建议：明确定义状态转换
   type QueueState = 'idle' | 'processing' | 'draining-now' | 'draining-next'
   ```

3. **粘贴内容优化**
   ```typescript
   // 建议：使用引用而非内联存储
   interface QueuedCommand {
     pastedContentRefs?: string[]  // 引用外部存储
   }
   // 大内容存储在专用存储中，命令完成后清理
   ```

4. **输入模式枚举优化**
   ```typescript
   // 建议：使用 const enum 减少运行时开销
   export const enum PromptInputMode {
     Bash = 'bash',
     Prompt = 'prompt',
     OrphanedPermission = 'orphaned-permission',
     TaskNotification = 'task-notification',
   }
   ```

5. **工具函数增强**
   ```typescript
   // 建议：增加粘贴内容统计
   export function getPasteStats(
     pastedContents: Record<number, PastedContent>
   ): { imageCount: number; textCount: number; totalSize: number }
   ```

6. **文档完善**
   - 增加各输入模式的适用场景说明
   - 提供队列优先级选择的决策树
   - 说明视口窗口化的触发条件和性能影响
