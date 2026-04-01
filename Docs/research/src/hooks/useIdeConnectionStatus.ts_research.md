# useIdeConnectionStatus.ts 研究文档

## 场景与职责

`useIdeConnectionStatus` 是一个轻量级的 React Hook，用于从 MCP clients 列表中**提取 IDE 连接状态**。它将底层的 `MCPServerConnection` 数组转换为更上层的 UI 友好状态：`status`（connected / disconnected / pending / null）和 `ideName`（IDE 显示名称）。

该 Hook 被多个组件使用，用于在 UI 中展示 IDE 连接指示器、控制 IDE 相关功能的可用性、以及决定显示何种提示信息。

## 功能点目的

1. **简化 IDE 状态查询**：将复杂的 `MCPServerConnection[]` 查询逻辑封装为一个简单的 Hook，调用方无需了解 MCP 连接的内部结构。

2. **提供连接状态枚举**：返回 `'connected'`、`'disconnected'`、`'pending'` 或 `null`（无 IDE client），便于 UI 做条件渲染。

3. **提取 IDE 名称**：从 IDE client 的配置中读取 `ideName`（如 "VS Code"、"Cursor"），用于状态提示和通知文案。

## 具体技术实现

### 源码实现

```ts
import { useMemo } from 'react'
import type { MCPServerConnection } from '../services/mcp/types.js'

export type IdeStatus = 'connected' | 'disconnected' | 'pending' | null

type IdeConnectionResult = {
  status: IdeStatus
  ideName: string | null
}

export function useIdeConnectionStatus(
  mcpClients?: MCPServerConnection[],
): IdeConnectionResult {
  return useMemo(() => {
    const ideClient = mcpClients?.find(client => client.name === 'ide')
    if (!ideClient) {
      return { status: null, ideName: null }
    }
    const config = ideClient.config
    const ideName =
      config.type === 'sse-ide' || config.type === 'ws-ide'
        ? config.ideName
        : null
    if (ideClient.type === 'connected') {
      return { status: 'connected', ideName }
    }
    if (ideClient.type === 'pending') {
      return { status: 'pending', ideName }
    }
    return { status: 'disconnected', ideName }
  }, [mcpClients])
}
```

### 设计要点

- **`useMemo` 优化**：由于 `mcpClients` 数组可能在每次渲染时变化（如引用更新），使用 `useMemo` 避免不必要的对象重建，减少下游组件的重渲染。

- **按 `name === 'ide'` 查找**：Claude Code 的 MCP 客户端命名约定中，IDE 扩展对应的 client 名称固定为 `'ide'`。这是该 Hook 的核心假设。

- **`ideName` 提取逻辑**：
  只有当 `config.type` 是 `'sse-ide'` 或 `'ws-ide'` 时，才读取 `config.ideName`。这是因为其他类型的 MCP server config（如 `'stdio'`、`'sse'`）没有 `ideName` 字段。

- **状态映射**：
  - `connected` → `'connected'`
  - `pending` → `'pending'`
  - 其他（`failed`、`needs-auth`、`disabled`）→ `'disconnected'`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useIdeConnectionStatus.ts` | 本 Hook 实现 |
| `src/hooks/notifs/useIDEStatusIndicator.tsx` | 调用方：IDE 状态通知指示器 |
| `src/components/IdeStatusIndicator.tsx` | 调用方：IDE 状态 UI 组件 |
| `src/components/PromptInput/Notifications.tsx` | 调用方：Prompt 区域通知 |
| `src/services/mcp/types.ts` | `MCPServerConnection`、`McpSSEIDEServerConfig`、`McpWebSocketIDEServerConfig` |

## 依赖与外部交互

### 内部依赖
- **React**：`useMemo`
- **MCP 类型**：`MCPServerConnection`

### 无外部交互
该 Hook 是纯计算逻辑，不涉及网络、文件系统或外部进程。

## 风险、边界与改进建议

### 风险与边界

1. **硬编码的 `'ide'` 名称**：Hook 假设 IDE client 的名称一定是 `'ide'`。如果未来 MCP 连接管理器更改了命名约定（如使用 `'vscode'`、`'jetbrains'` 等具体名称），该 Hook 会返回 `null` 状态，导致所有 IDE 相关功能失效。

2. **`mcpClients` 数组的引用稳定性**：虽然使用了 `useMemo`，但如果父组件每次渲染都传入一个新的 `mcpClients` 数组字面量，`useMemo` 的缓存会失效。在 `REPL.tsx` 中，远程模式使用了 `EMPTY_MCP_CLIENTS` 稳定空数组来避免这个问题，但其他调用方可能没有这么谨慎。

3. **`disconnected` 状态过于宽泛**：`failed`、`needs-auth`、`disabled` 三种完全不同的失败原因都被映射为 `'disconnected'`。调用方无法区分是"连接失败"、"需要认证"还是"被用户禁用"，因此无法给出针对性的提示或恢复建议。

4. **`ideName` 可能为 `null`**：即使找到了 IDE client，如果其 `config.type` 不是预期的 `'sse-ide'` 或 `'ws-ide'`，`ideName` 也会是 `null`。这会导致通知消息中出现 "null disconnected" 或空白名称。

5. **不支持多 IDE**：如果 `mcpClients` 中同时存在多个 IDE client（如 VS Code 和 Cursor 都连接了），该 Hook 只返回第一个找到的状态，忽略其他 IDE。这在多 IDE 场景下会导致信息丢失。

6. **无错误处理或日志**：如果 `mcpClients` 中包含格式异常的对象（如 `config` 为 `undefined`），访问 `config.type` 会抛出异常。虽然 TypeScript 类型系统应该能防止这种情况，但运行时仍可能因数据损坏而崩溃。

### 改进建议

1. **将 `'ide'` 提取为常量**：定义 `const IDE_CLIENT_NAME = 'ide'`，并在相关模块中共享，避免魔法字符串。

2. **细化 disconnected 状态**：将 `IdeStatus` 扩展为更具体的枚举，如：
   ```ts
   export type IdeStatus =
     | 'connected'
     | 'pending'
     | 'failed'
     | 'needs-auth'
     | 'disabled'
     | null
   ```
   这样调用方可以显示 "IDE extension needs authentication" 而不是笼统的 "disconnected"。

3. **支持多 IDE 状态聚合**：可以提供一个高级 Hook（如 `useAllIdeConnectionStatuses`），返回所有 IDE client 的状态数组，供需要展示多个 IDE 状态的组件使用。

4. **增加防御性编程**：在访问 `config.type` 前增加空值检查：
   ```ts
   const ideName =
     config && (config.type === 'sse-ide' || config.type === 'ws-ide')
       ? config.ideName
       : null
   ```

5. **缓存 `ideName` 的显示名称映射**：某些 `ideName` 可能是内部标识符（如 `'vscode'`），可以在 Hook 内部或工具函数中增加一个 `toIDEDisplayName` 的映射，确保 UI 总是显示人类可读的名称（如 "VS Code"）。

6. **单元测试覆盖**：建议增加对以下场景的测试：
   - `mcpClients` 为空数组
   - 存在 `name === 'ide'` 但 `type === 'failed'` 的 client
   - 存在多个 IDE client 时的行为
   - `config` 类型不匹配时的降级行为
