# useIdeSelection.ts 研究文档

## 场景与职责

`useIdeSelection` 是一个用于接收并处理 IDE **文本选择变更通知**的 Hook。当用户在连接的 IDE 中选中某段代码时，IDE 扩展会通过 MCP 通知通道发送 `selection_changed` 消息。该 Hook 负责监听这些通知，解析选中的行数、起始行号、选中文本和文件路径，并将这些信息传递给调用方。

该 Hook 被广泛应用于：
- `REPL.tsx`：维护全局 `ideSelection` 状态
- `PromptInput.tsx`：将 IDE 选中的代码作为上下文引用插入输入框
- `useIDEStatusIndicator.tsx`：根据是否有选中文本显示不同的状态提示
- `handlePromptSubmit.ts`、`attachments.ts`、`processUserInput.ts`：在提交提示时将 IDE 选择作为附件或上下文发送给模型

## 功能点目的

1. **注册 MCP 通知处理器**：在已连接的 IDE MCP client 上注册 `selection_changed` 通知的 handler。

2. **解析并转换选择信息**：将 IDE 发送的 `selection` 对象（包含 `start` 和 `end` 的 `line`/`character`）转换为 `lineCount` 和 `lineStart`。

3. **处理空选择**：当用户没有选中任何文本但光标在某个位置时，IDE 可能发送 `selection: null` 但带有 `text` 和 `filePath`。Hook 也会将这些信息传递给调用方。

4. **IDE client 变化感知**：当 MCP clients 变化导致连接的 IDE client 改变时，自动重置选择状态并重新注册 handler。

## 具体技术实现

### 源码实现

```ts
import { useEffect, useRef } from 'react'
import { logError } from 'src/utils/log.js'
import { z } from 'zod/v4'
import type { ConnectedMCPServer, MCPServerConnection } from '../services/mcp/types.js'
import { getConnectedIdeClient } from '../utils/ide.js'
import { lazySchema } from '../utils/lazySchema.js'

export type SelectionPoint = {
  line: number
  character: number
}

export type SelectionData = {
  selection: {
    start: SelectionPoint
    end: SelectionPoint
  } | null
  text?: string
  filePath?: string
}

export type IDESelection = {
  lineCount: number
  lineStart?: number
  text?: string
  filePath?: string
}

const SelectionChangedSchema = lazySchema(() =>
  z.object({
    method: z.literal('selection_changed'),
    params: z.object({
      selection: z
        .object({
          start: z.object({ line: z.number(), character: z.number() }),
          end: z.object({ line: z.number(), character: z.number() }),
        })
        .nullable()
        .optional(),
      text: z.string().optional(),
      filePath: z.string().optional(),
    }),
  }),
)

export function useIdeSelection(
  mcpClients: MCPServerConnection[],
  onSelect: (selection: IDESelection) => void,
): void {
  const handlersRegistered = useRef(false)
  const currentIDERef = useRef<ConnectedMCPServer | null>(null)

  useEffect(() => {
    const ideClient = getConnectedIdeClient(mcpClients)

    if (currentIDERef.current !== (ideClient ?? null)) {
      handlersRegistered.current = false
      currentIDERef.current = ideClient || null
      onSelect({
        lineCount: 0,
        lineStart: undefined,
        text: undefined,
        filePath: undefined,
      })
    }

    if (handlersRegistered.current || !ideClient) {
      return
    }

    const selectionChangeHandler = (data: SelectionData) => {
      if (data.selection?.start && data.selection?.end) {
        const { start, end } = data.selection
        let lineCount = end.line - start.line + 1
        if (end.character === 0) {
          lineCount--
        }
        const selection = {
          lineCount,
          lineStart: start.line,
          text: data.text,
          filePath: data.filePath,
        }
        onSelect(selection)
      }
    }

    ideClient.client.setNotificationHandler(
      SelectionChangedSchema(),
      notification => {
        if (currentIDERef.current !== ideClient) {
          return
        }
        try {
          const selectionData = notification.params
          if (
            selectionData.selection &&
            selectionData.selection.start &&
            selectionData.selection.end
          ) {
            selectionChangeHandler(selectionData as SelectionData)
          } else if (selectionData.text !== undefined) {
            selectionChangeHandler({
              selection: null,
              text: selectionData.text,
              filePath: selectionData.filePath,
            })
          }
        } catch (error) {
          logError(error as Error)
        }
      },
    )

    handlersRegistered.current = true
  }, [mcpClients, onSelect])
}
```

### 设计要点

- **`handlersRegistered` 和 `currentIDERef` 双 ref 机制**：
  - `currentIDERef` 跟踪当前注册的 IDE client，当 client 变化时重置选择状态并标记 `handlersRegistered = false`
  - `handlersRegistered` 防止对同一个 client 重复注册 handler

- **行数计算逻辑**：
  ```ts
  let lineCount = end.line - start.line + 1
  if (end.character === 0) {
    lineCount--
  }
  ```
  如果选区结束在某一行的第 0 个字符（即行首），说明该行实际上没有被选中，因此不计入行数。这是一个精细的边界处理。

- **空选择处理**：
  当 `selection` 为 null/undefined 但 `text` 存在时，仍然传递 `text` 和 `filePath`，`lineCount` 为 0。这允许调用方知道用户当前焦点在哪个文件，即使没有选中文本。

- **Zod schema 验证**：`selection` 字段被标记为 `nullable().optional()`，兼容 IDE 扩展发送的不同格式。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useIdeSelection.ts` | 本 Hook 实现 |
| `src/screens/REPL.tsx` | 调用方：维护全局 `ideSelection` 状态 |
| `src/components/PromptInput/PromptInput.tsx` | 调用方：将 IDE 选择插入输入框 |
| `src/hooks/notifs/useIDEStatusIndicator.tsx` | 调用方：根据选择状态显示通知 |
| `src/utils/handlePromptSubmit.ts` | 调用方：提交时将选择作为上下文 |
| `src/utils/attachments.ts` | 调用方：构建附件消息时引用选择 |
| `src/utils/processUserInput/processUserInput.ts` | 调用方：处理用户输入时包含选择上下文 |
| `src/components/IdeStatusIndicator.tsx` | 调用方：状态指示器 UI |
| `src/components/PromptInput/PromptInputFooter.tsx` | 调用方：页脚显示选择信息 |
| `src/components/PromptInput/Notifications.tsx` | 调用方：通知系统 |
| `src/utils/ide.ts` | `getConnectedIdeClient` |
| `src/utils/lazySchema.ts` | `lazySchema` |
| `src/services/mcp/types.ts` | `ConnectedMCPServer`、`MCPServerConnection` |

## 依赖与外部交互

### 内部依赖
- **React**：`useEffect`、`useRef`
- **Zod**：`zod/v4`，用于通知数据验证
- **MCP SDK**：`ConnectedMCPServer.client.setNotificationHandler`

### 外部交互
- **IDE 扩展 MCP 通知**：接收 `method: 'selection_changed'` 的 JSON-RPC notification，参数包含 `selection`（`start`/`end` 的 `line`/`character`）、`text`、`filePath`

## 风险、边界与改进建议

### 风险与边界

1. **无 cleanup 的 handler 累积**：与 `useIdeAtMentioned` 和 `useIdeLogging` 相同，该 Hook 没有在 `useEffect` cleanup 中取消 notification handler。如果 MCP SDK 的 `setNotificationHandler` 是追加行为，会导致重复回调。

2. **`lineCount` 计算未处理负值**：虽然正常情况下 `end.line >= start.line`，但如果 IDE 扩展发送了异常数据（如 `end.line < start.line`），`lineCount` 会变成 0 或负数。当前代码没有对这种异常进行防护。

3. **`lineStart` 是 0-based**：Hook 将 `start.line` 直接作为 `lineStart` 返回。根据 `useIdeAtMentioned` 中的逻辑，IDE 发送的行号通常是 0-based，但这里并没有像 `useIdeAtMentioned` 那样做 +1 转换。这可能导致调用方（如 `PromptInput`）在显示行号时出现 0-based 和 1-based 混用的问题。

4. **`onSelect` 的闭包 stale 风险**：`useEffect` 依赖 `onSelect`，如果调用方每次渲染都传入新函数，会导致频繁重新注册 handler。

5. **空选择时 `lineCount: 0` 的语义模糊**：当 `selection` 为 null 时，Hook 传递 `{ lineCount: 0, text, filePath }`。调用方需要自行判断 `lineCount === 0` 是"没有选中文本"还是"选择信息尚未初始化"。

6. **无文件存在性校验**：`filePath` 直接透传，没有验证文件是否真实存在。如果 IDE 扩展发送了错误路径，调用方可能构建出无效的附件或上下文。

### 改进建议

1. **统一行号基准**：明确 IDE 选择通知中的行号是 0-based 还是 1-based，并在 Hook 中统一转换。如果与 `useIdeAtMentioned` 保持一致（0-based 转 1-based），则 `lineStart` 应该做 +1：
   ```ts
   lineStart: start.line + 1,
   ```

2. **增加 `lineCount` 的边界检查**：
   ```ts
   let lineCount = Math.max(0, end.line - start.line + 1)
   if (end.character === 0 && lineCount > 0) {
     lineCount--
   }
   ```

3. **显式取消旧 handler**：在 `useEffect` cleanup 中增加 handler 移除逻辑，或至少文档化 `setNotificationHandler` 的覆盖语义。

4. **使用 `useCallback` 稳定 `onSelect`**：在调用方（如 `REPL.tsx`）中，应确保 `onSelect` 被 `useCallback` 包裹，减少不必要的 effect 重建。

5. **增加选择防抖**：IDE 中的选择变更可能非常频繁（如鼠标拖拽选区时每秒发送数十次通知）。可以在 Hook 内部增加一个防抖（debounce，如 100ms），减少下游组件的渲染压力。
   ```ts
   const debouncedOnSelect = useRef(debounce(onSelect, 100)).current
   useEffect(() => { debouncedOnSelect.current = debounce(onSelect, 100) }, [onSelect])
   ```

6. **路径规范化**：在传递 `filePath` 前，使用 `expandPath` 将其转为绝对路径，减少调用方的处理负担并提高一致性。

7. **区分"无选择"和"空选择"**：可以考虑在 `IDESelection` 类型中增加一个 `hasSelection: boolean` 字段，让调用方更清晰地判断当前状态：
   ```ts
   export type IDESelection = {
     hasSelection: boolean
     lineCount: number
     lineStart?: number
     text?: string
     filePath?: string
   }
   ```
