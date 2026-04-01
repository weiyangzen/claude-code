# useIdeAtMentioned.ts 研究文档

## 场景与职责

`useIdeAtMentioned` 是一个用于接收并处理 IDE **@提及（at-mention）通知**的 Hook。当用户在连接的 IDE 中通过某种机制（如右键菜单、快捷键）将当前文件或选中的代码块 "@提及" 到 Claude Code 时，IDE 扩展会通过 MCP 通知通道发送 `at_mentioned` 消息。该 Hook 负责监听这些通知，并将解析后的文件路径和行号范围传递给调用方。

该 Hook 主要被 `PromptInput.tsx` 使用，用于将 IDE 中 @提及的内容自动插入到用户输入框中，作为上下文引用。

## 功能点目的

1. **注册 MCP 通知处理器**：在已连接的 IDE MCP client 上注册 `at_mentioned` 通知的 handler。

2. **解析通知数据**：使用 Zod schema 验证通知内容的结构，提取 `filePath`、`lineStart`、`lineEnd`。

3. **行号转换**：IDE 扩展通常使用 0-based 行号，而 Claude Code 内部使用 1-based，因此 Hook 将接收到的行号加 1。

4. **IDE client 变化感知**：当 MCP clients 列表变化导致连接的 IDE client 改变时，自动更新 handler 的注册目标。

## 具体技术实现

### 源码实现

```ts
import { useEffect, useRef } from 'react'
import { logError } from 'src/utils/log.js'
import { z } from 'zod/v4'
import type { ConnectedMCPServer, MCPServerConnection } from '../services/mcp/types.js'
import { getConnectedIdeClient } from '../utils/ide.js'
import { lazySchema } from '../utils/lazySchema.js'

export type IDEAtMentioned = {
  filePath: string
  lineStart?: number
  lineEnd?: number
}

const NOTIFICATION_METHOD = 'at_mentioned'

const AtMentionedSchema = lazySchema(() =>
  z.object({
    method: z.literal(NOTIFICATION_METHOD),
    params: z.object({
      filePath: z.string(),
      lineStart: z.number().optional(),
      lineEnd: z.number().optional(),
    }),
  }),
)

export function useIdeAtMentioned(
  mcpClients: MCPServerConnection[],
  onAtMentioned: (atMentioned: IDEAtMentioned) => void,
): void {
  const ideClientRef = useRef<ConnectedMCPServer | undefined>(undefined)

  useEffect(() => {
    const ideClient = getConnectedIdeClient(mcpClients)

    if (ideClientRef.current !== ideClient) {
      ideClientRef.current = ideClient
    }

    if (ideClient) {
      ideClient.client.setNotificationHandler(
        AtMentionedSchema(),
        notification => {
          if (ideClientRef.current !== ideClient) {
            return
          }
          try {
            const data = notification.params
            const lineStart =
              data.lineStart !== undefined ? data.lineStart + 1 : undefined
            const lineEnd =
              data.lineEnd !== undefined ? data.lineEnd + 1 : undefined
            onAtMentioned({
              filePath: data.filePath,
              lineStart: lineStart,
              lineEnd: lineEnd,
            })
          } catch (error) {
            logError(error as Error)
          }
        },
      )
    }
  }, [mcpClients, onAtMified])
}
```

### 设计要点

- **`lazySchema` 延迟初始化**：`AtMentionedSchema` 使用 `lazySchema` 包装，避免在模块加载时就创建 Zod schema，减少启动开销。

- **`ideClientRef` 的防护作用**：
  在 notification handler 内部，通过比较 `ideClientRef.current !== ideClient` 来过滤掉"旧 client 上延迟到达的通知"。这是一种轻量级的竞态保护，防止在 IDE client 切换过程中处理过期消息。

- **行号 +1 转换**：
  ```ts
  const lineStart = data.lineStart !== undefined ? data.lineStart + 1 : undefined
  const lineEnd = data.lineEnd !== undefined ? data.lineEnd + 1 : undefined
  ```
  这是该 Hook 的核心业务逻辑之一，确保与 Claude Code 内部的 1-based 行号约定一致。

- **无 cleanup**：
  代码注释明确说明 "No cleanup needed as MCP clients manage their own lifecycle"。当 MCP client 断开连接时，其上的所有 notification handlers 会随 client 实例一起被清理。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useIdeAtMentioned.ts` | 本 Hook 实现 |
| `src/components/PromptInput/PromptInput.tsx` | 主要调用方，将 @提及内容插入输入框 |
| `src/utils/ide.ts` | `getConnectedIdeClient` |
| `src/utils/lazySchema.ts` | `lazySchema` |
| `src/services/mcp/types.ts` | `ConnectedMCPServer`、`MCPServerConnection` |
| `src/utils/log.ts` | `logError` |

## 依赖与外部交互

### 内部依赖
- **React**：`useEffect`、`useRef`
- **Zod**：`zod/v4`，用于通知数据验证
- **MCP SDK**：`ConnectedMCPServer.client.setNotificationHandler`

### 外部交互
- **IDE 扩展 MCP 通知**：接收 `method: 'at_mentioned'` 的 JSON-RPC notification，参数包含 `filePath`、`lineStart`、`lineEnd`

## 风险、边界与改进建议

### 风险与边界

1. **无 cleanup 的潜在内存泄漏**：虽然注释说 MCP client 管理自己的生命周期，但如果 `setNotificationHandler` 被多次调用（如 `mcpClients` 数组引用频繁变化），底层 MCP client 实现可能会累积多个 handler。当前代码没有显式取消注册旧 handler，这依赖于 MCP SDK 的 `setNotificationHandler` 是否会覆盖同类型的 handler。

2. **`getConnectedIdeClient` 的稳定性**：`getConnectedIdeClient` 遍历 `mcpClients` 找到第一个 `name === 'ide' && type === 'connected'` 的 client。如果 `mcpClients` 数组顺序变化，即使同一个 IDE client 仍在，也可能导致 `ideClientRef.current !== ideClient` 为 true，从而重新注册 handler。虽然通常无害，但增加了不必要的操作。

3. **Zod 验证失败静默丢弃**：如果 IDE 扩展发送了不符合 schema 的通知（如 `lineStart` 是字符串），`setNotificationHandler` 内部的 Zod 验证会失败，但错误不会被捕获和记录（因为 `setNotificationHandler` 通常会在验证失败时自动忽略消息）。这意味着协议变更或 bug 可能很难被发现。

4. **`onAtMentioned` 的闭包 stale 风险**：`useEffect` 的依赖数组包含了 `onAtMentioned`，如果调用方（如 `PromptInput.tsx`）每次渲染都传入一个新的函数引用，会导致 effect 频繁重新执行。虽然 `PromptInput.tsx` 通常会用 `useCallback` 包装，但这不是强制的。

5. **行号转换的假设**：代码假设 IDE 扩展总是发送 0-based 行号。如果未来某个 IDE 扩展改为发送 1-based 行号，这里的 +1 会导致行号整体偏移 1，且没有版本协商机制来检测这种变化。

6. **无文件存在性校验**：`filePath` 直接被传递给 `onAtMentioned`，没有检查文件是否真实存在或是否在项目目录内。如果 IDE 扩展发送了一个错误路径，调用方需要自己处理。

### 改进建议

1. **显式取消旧 handler**：在 `useEffect` 的 cleanup 中，如果之前有注册过 handler，应尝试取消注册。如果 MCP SDK 不支持取消单个 notification handler，至少应确保 `setNotificationHandler` 的行为是"覆盖"而非"追加"。
   ```ts
   return () => {
     if (ideClient) {
       ideClient.client.removeNotificationHandler?.(NOTIFICATION_METHOD)
     }
   }
   ```

2. **使用 `useCallback` 稳定 `onAtMentioned` 的引用**：在调用方 `PromptInput.tsx` 中，应确保 `onAtMentioned` 被 `useCallback` 包裹，减少不必要的 effect 重建。

3. **增加协议版本或格式协商**：在 `at_mentioned` 通知中增加一个 `format` 或 `version` 字段，明确告知行号基准（0-based vs 1-based），使 Hook 能自适应处理。

4. **记录验证失败事件**：如果 MCP SDK 支持验证失败的回调，应记录这些失败以便排查 IDE 扩展与 CLI 之间的协议不匹配问题。

5. **路径规范化**：在调用 `onAtMentioned` 前，可以使用 `expandPath` 或 `safeResolvePath` 将 `filePath` 规范化为绝对路径，减少调用方的处理负担。

6. **支持多 IDE client**：当前只处理第一个找到的 IDE client。如果未来支持同时连接多个 IDE（如 VS Code + JetBrains），需要扩展为注册所有 IDE client 的 handler，并在回调中标识来源。
