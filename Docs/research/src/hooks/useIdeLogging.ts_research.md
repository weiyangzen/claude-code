# useIdeLogging.ts 研究文档

## 场景与职责

`useIdeLogging` 是一个用于接收 IDE 扩展发来的**分析事件（analytics events）通知**并将其转发到 Claude Code 本地分析系统的 Hook。当 IDE 扩展中发生了某些用户交互（如点击了某个按钮、使用了某个功能），IDE 可以通过 MCP 通知通道发送 `log_event` 消息。该 Hook 负责监听这些通知，并将事件名称前缀为 `tengu_ide_` 后通过 `logEvent` 上报。

该 Hook 主要被 `REPL.tsx` 使用，用于打通 IDE 侧与 CLI 侧的分析数据链路。

## 功能点目的

1. **注册 MCP 通知处理器**：在已连接的 IDE MCP client 上注册 `log_event` 通知的 handler。

2. **转发分析事件**：将 IDE 发送的事件名加上前缀 `tengu_ide_`，事件数据原样透传，统一纳入 Claude Code 的分析事件体系。

3. **多 client 安全处理**：当 `mcpClients` 数组为空时直接跳过，避免不必要的操作。

## 具体技术实现

### 源码实现

```ts
import { useEffect } from 'react'
import { logEvent } from 'src/services/analytics/index.js'
import { z } from 'zod/v4'
import type { MCPServerConnection } from '../services/mcp/types.js'
import { getConnectedIdeClient } from '../utils/ide.js'
import { lazySchema } from '../utils/lazySchema.js'

const LogEventSchema = lazySchema(() =>
  z.object({
    method: z.literal('log_event'),
    params: z.object({
      eventName: z.string(),
      eventData: z.object({}).passthrough(),
    }),
  }),
)

export function useIdeLogging(mcpClients: MCPServerConnection[]): void {
  useEffect(() => {
    if (!mcpClients.length) {
      return
    }

    const ideClient = getConnectedIdeClient(mcpClients)
    if (ideClient) {
      ideClient.client.setNotificationHandler(
        LogEventSchema(),
        notification => {
          const { eventName, eventData } = notification.params
          logEvent(
            `tengu_ide_${eventName}`,
            eventData as { [key: string]: boolean | number | undefined },
          )
        },
      )
    }
  }, [mcpClients])
}
```

### 设计要点

- **`lazySchema` 延迟初始化**：`LogEventSchema` 使用 `lazySchema` 包装，避免模块加载时立即创建 Zod schema。

- **空数组短路**：`if (!mcpClients.length) return` 快速返回，避免在远程模式（`mcpClients` 为空）时执行无意义的查找。

- **事件名前缀**：所有 IDE 事件统一加上 `tengu_ide_` 前缀，便于在分析后台区分事件来源。

- **类型断言**：`eventData` 被断言为 `{ [key: string]: boolean | number | undefined }`，这是 `logEvent` 第二参数的类型要求。如果 IDE 发送了字符串类型的 eventData 值，TypeScript 编译时不会报错（因为用了 `as`），但运行时可能不符合 `logEvent` 的预期。

- **无 cleanup**：与 `useIdeAtMentioned` 类似，代码中没有在 `useEffect` cleanup 中取消 notification handler，依赖 MCP client 的生命周期管理。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useIdeLogging.ts` | 本 Hook 实现 |
| `src/screens/REPL.tsx` | 主要调用方 |
| `src/utils/ide.ts` | `getConnectedIdeClient` |
| `src/utils/lazySchema.ts` | `lazySchema` |
| `src/services/mcp/types.ts` | `MCPServerConnection`、`ConnectedMCPServer` |
| `src/services/analytics/index.js` | `logEvent` |
| `src/utils/log.ts` | `logError`（未直接使用，但相关） |

## 依赖与外部交互

### 内部依赖
- **React**：`useEffect`
- **Zod**：`zod/v4`，用于通知数据验证
- **Analytics**：`logEvent`

### 外部交互
- **IDE 扩展 MCP 通知**：接收 `method: 'log_event'` 的 JSON-RPC notification
- **Analytics 后端**：`logEvent` 最终可能将事件发送到 Anthropic 的分析服务（如 Segment、GrowthBook 等）

## 风险、边界与改进建议

### 风险与边界

1. **无 cleanup 的 handler 累积**：与 `useIdeAtMentioned` 相同，如果 `mcpClients` 引用频繁变化，`setNotificationHandler` 可能会被多次调用。如果 MCP SDK 的实现是"追加"而非"覆盖"同类型的 handler，会导致同一个事件被重复上报。

2. **`eventData` 类型断言的风险**：`eventData as { [key: string]: boolean | number | undefined }` 假设了 IDE 发送的数据只包含布尔值和数字。如果 IDE 扩展发送了嵌套对象、数组或字符串，`logEvent` 内部可能会丢弃这些字段、序列化失败，甚至抛出异常。

3. **事件名冲突**：IDE 事件名与 CLI 本地事件名之间没有命名空间隔离，仅靠 `tengu_ide_` 前缀。如果 IDE 发送了一个与现有 CLI 事件同名（去掉前缀后）的事件，虽然前缀不同，但在分析后台查询时仍可能造成混淆。

4. **无错误处理**：notification handler 内部没有 try/catch。如果 `logEvent` 抛出异常（如网络不可用、序列化失败），异常会冒泡到 MCP SDK 的事件循环中，可能导致未捕获的异常或影响其他 notification 的处理。

5. **`mcpClients.length` 检查的局限性**：`!mcpClients.length` 只能判断数组是否为空，不能判断数组中的 client 是否有效。如果数组非空但没有 IDE client，`getConnectedIdeClient` 返回 `undefined`，后续逻辑自然跳过，这是正确的。

6. **IDE 扩展发送频率无限制**：如果 IDE 扩展存在 bug 或恶意行为，可能高频发送 `log_event` 通知。该 Hook 没有节流（throttle）或采样机制，所有事件都会直接透传给 `logEvent`。

### 改进建议

1. **增加 try/catch 保护**：在 notification handler 中包裹 try/catch，将错误记录到 `logError` 而不是抛出：
   ```ts
   notification => {
     try {
       const { eventName, eventData } = notification.params
       logEvent(`tengu_ide_${eventName}`, eventData as ...)
     } catch (error) {
       logError(error as Error)
     }
   }
   ```

2. **验证 `eventData` 的内容类型**：在调用 `logEvent` 前，对 `eventData` 进行清理，只保留 `boolean | number | undefined` 类型的顶层字段，过滤掉字符串、对象、数组：
   ```ts
   const sanitizedData = Object.fromEntries(
     Object.entries(eventData).filter(([, v]) =>
       typeof v === 'boolean' || typeof v === 'number' || v === undefined
     )
   )
   ```

3. **增加事件节流/采样**：对于高频事件（如 IDE 中的鼠标移动、滚动），可以在 Hook 中增加一个简单的内存级节流（如每 100ms 同类型事件只上报一次），防止分析系统被淹没。

4. **显式取消旧 handler**：在 `useEffect` cleanup 中，如果之前有注册过 handler，应尝试移除。如果 MCP SDK 不支持，至少应文档化 `setNotificationHandler` 的覆盖行为。

5. **事件名白名单（可选）**：为了防止 IDE 扩展发送意外事件，可以维护一个允许的事件名白名单，只有白名单内的事件才会被转发。这在安全敏感的场景下尤其有用。

6. **增加调试日志**：在开发模式下，可以记录每个转发的 IDE 事件名和数据，便于排查 IDE 与 CLI 之间的分析数据链路问题。
