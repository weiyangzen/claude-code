# ListMcpResourcesTool.ts 研究文档

## 场景与职责

`ListMcpResourcesTool` 是 Claude Code CLI 中用于**列举 MCP (Model Context Protocol) 服务器资源**的核心工具。MCP 是 Anthropic 推出的开放协议，允许 AI 助手通过标准化接口与外部工具和服务交互。

### 核心职责

1. **资源发现**：从已连接的 MCP 服务器获取可用资源列表
2. **跨服务器聚合**：支持从所有服务器或指定服务器获取资源
3. **缓存优化**：利用 LRU 缓存机制避免重复请求
4. **错误隔离**：单个服务器失败不影响整体结果

### 使用场景

- 用户需要查看当前可用的 MCP 资源
- 模型需要了解可以访问哪些外部资源
- 配合 `ReadMcpResourceTool` 实现资源的读取流程

---

## 功能点目的

### 1. 输入参数

```typescript
{
  server?: string  // 可选：指定服务器名称过滤
}
```

- 不提供 `server`：返回所有已连接服务器的资源
- 提供 `server`：仅返回指定服务器的资源，若服务器不存在则报错

### 2. 输出结构

```typescript
Array<{
  uri: string        // 资源唯一标识符
  name: string       // 资源名称
  mimeType?: string  // MIME 类型（可选）
  description?: string // 资源描述（可选）
  server: string     // 提供资源的服务器名称
}>
```

### 3. 工具元数据

| 属性 | 值 | 说明 |
|------|-----|------|
| `shouldDefer` | `true` | 延迟加载工具，需通过 ToolSearch 触发 |
| `isConcurrencySafe` | `true` | 支持并发调用 |
| `isReadOnly` | `true` | 只读操作，不修改状态 |
| `maxResultSizeChars` | 100,000 | 结果大小上限 |
| `searchHint` | `'list resources from connected MCP servers'` | 工具搜索关键词 |

---

## 具体技术实现

### 关键流程

#### 1. 工具构建流程

```typescript
export const ListMcpResourcesTool = buildTool({
  // ... 配置项
}) satisfies ToolDef<InputSchema, OutputSchema>
```

使用 `buildTool` 工厂函数（位于 `src/Tool.ts`）创建工具实例，自动填充默认值：
- `isEnabled`: 默认 `true`
- `isConcurrencySafe`: 默认 `false`（本工具覆盖为 `true`）
- `isReadOnly`: 默认 `false`（本工具覆盖为 `true`）
- `checkPermissions`: 默认允许

#### 2. 核心调用流程

```typescript
async call(input, { options: { mcpClients } }) {
  const { server: targetServer } = input

  // 1. 筛选要处理的服务器
  const clientsToProcess = targetServer
    ? mcpClients.filter(client => client.name === targetServer)
    : mcpClients

  // 2. 验证指定服务器存在性
  if (targetServer && clientsToProcess.length === 0) {
    throw new Error(`Server "${targetServer}" not found...`)
  }

  // 3. 并行获取所有服务器资源
  const results = await Promise.all(
    clientsToProcess.map(async client => {
      if (client.type !== 'connected') return []
      try {
        const fresh = await ensureConnectedClient(client)
        return await fetchResourcesForClient(fresh)
      } catch (error) {
        logMCPError(client.name, errorMessage(error))
        return []
      }
    }),
  )

  return { data: results.flat() }
}
```

#### 3. 缓存机制

`fetchResourcesForClient` 函数（位于 `src/services/mcp/client.ts`）使用 LRU 缓存：

```typescript
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    // 1. 检查服务器是否已连接
    if (client.type !== 'connected') return []
    
    // 2. 检查服务器能力（是否支持 resources）
    if (!client.capabilities?.resources) {
      return []
    }

    // 3. 调用 MCP 协议方法
    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema,
    )

    // 4. 添加服务器名称到每个资源
    return result.resources.map(resource => ({
      ...resource,
      server: client.name,
    }))
  },
  (client: MCPServerConnection) => client.name,  // 缓存键：服务器名称
  MCP_FETCH_CACHE_SIZE,  // 默认 20
)
```

**缓存失效策略**：
- 服务器 `onclose` 事件时清除
- `resources/list_changed` 通知时清除
- 重新连接时自动刷新

#### 4. 连接保活机制

```typescript
const fresh = await ensureConnectedClient(client)
```

`ensureConnectedClient` 函数确保：
- 健康连接：直接返回（memoize 命中）
- 断开的连接：重新建立连接后返回
- SDK 类型服务器：跳过连接检查（进程内运行）

#### 5. 结果截断检测

```typescript
isResultTruncated(output: Output): boolean {
  return isOutputLineTruncated(jsonStringify(output))
}
```

使用 `isOutputLineTruncated`（位于 `src/utils/terminal.ts`）检测输出是否超过 3 行，用于控制 UI 折叠显示。

#### 6. 结果映射到 API 格式

```typescript
mapToolResultToToolResultBlockParam(content, toolUseID) {
  if (!content || content.length === 0) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: 'No resources found. MCP servers may still provide tools...',
    }
  }
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: jsonStringify(content),
  }
}
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `buildTool` 工厂函数、`ToolDef` 类型定义 |
| `src/services/mcp/client.ts` | `ensureConnectedClient`, `fetchResourcesForClient` |
| `src/utils/errors.ts` | `errorMessage` 错误处理 |
| `src/utils/lazySchema.ts` | `lazySchema` 延迟加载 Zod schema |
| `src/utils/log.ts` | `logMCPError` MCP 错误日志 |
| `src/utils/slowOperations.ts` | `jsonStringify` 带性能监控的 JSON 序列化 |
| `src/utils/terminal.ts` | `isOutputLineTruncated` 输出截断检测 |
| `./prompt.ts` | `DESCRIPTION`, `LIST_MCP_RESOURCES_TOOL_NAME`, `PROMPT` |
| `./UI.tsx` | `renderToolResultMessage`, `renderToolUseMessage` |

### MCP 客户端服务关键代码

```typescript
// src/services/mcp/client.ts:2000-2031
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (client.type !== 'connected') return []
    try {
      if (!client.capabilities?.resources) {
        return []
      }
      const result = await client.client.request(
        { method: 'resources/list' },
        ListResourcesResultSchema,
      )
      if (!result.resources) return []
      return result.resources.map(resource => ({
        ...resource,
        server: client.name,
      }))
    } catch (error) {
      logMCPError(client.name, `Failed to fetch resources: ${errorMessage(error)}`)
      return []
    }
  },
  (client: MCPServerConnection) => client.name,
  MCP_FETCH_CACHE_SIZE,
)
```

### 缓存清除触发点

```typescript
// src/services/mcp/client.ts:1386-1396
client.onclose = () => {
  // ...
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)  // <-- 资源缓存清除
  fetchCommandsForClient.cache.delete(name)
  // ...
}
```

---

## 依赖与外部交互

### MCP 协议交互

使用 `@modelcontextprotocol/sdk` 进行通信：

```typescript
import { ListResourcesResultSchema } from '@modelcontextprotocol/sdk/types.js'
```

**协议方法**: `resources/list`
**响应 Schema**: `ListResourcesResultSchema`

### 与 ReadMcpResourceTool 的关系

`ListMcpResourcesTool` 与 `ReadMcpResourceTool` 配合使用：

1. `ListMcpResourcesTool` 获取资源列表（URI、名称、描述）
2. `ReadMcpResourceTool` 根据 URI 读取具体资源内容

两者在 `reconnectMcpServerImpl` 中作为资源工具被条件性添加：

```typescript
// src/services/mcp/client.ts:2182-2191
if (supportsResources) {
  const hasResourceTools = [ListMcpResourcesTool, ReadMcpResourceTool].some(
    tool => tools.some(t => toolMatchesName(t, tool.name)),
  )
  if (!hasResourceTools) {
    resourceTools.push(ListMcpResourcesTool, ReadMcpResourceTool)
  }
}
```

### 工具注册流程

```
getMcpToolsCommandsAndResources()
  └── processServer()
       └── connectToServer()
       └── fetchResourcesForClient()  // 预取资源
       └── onConnectionAttempt()
            └── 添加 ListMcpResourcesTool 到工具列表
```

---

## 风险、边界与改进建议

### 已知风险

1. **缓存一致性问题**
   - 风险：服务器资源在缓存期间发生变化，用户看到过期数据
   - 缓解：缓存通过 `onclose` 和 `resources/list_changed` 通知失效
   - 边界：如果服务器不支持订阅通知，缓存可能长期过期

2. **错误静默处理**
   - 风险：单个服务器错误仅记录日志，返回空数组，用户可能不知道部分服务器失败
   - 代码：`catch (error) { logMCPError(...); return [] }`

3. **内存占用**
   - 风险：大量 MCP 服务器（>20）时，LRU 缓存可能频繁失效
   - 配置：`MCP_FETCH_CACHE_SIZE = 20`

### 边界情况

| 场景 | 行为 |
|------|------|
| 无 MCP 服务器配置 | 返回空数组 |
| 指定服务器不存在 | 抛出错误，列出可用服务器 |
| 服务器未连接 | 跳过该服务器，返回空数组 |
| 服务器不支持 resources 能力 | 返回空数组 |
| 服务器返回 401/需要认证 | 返回空数组（通过 `needs-auth` 状态处理）|
| 结果超过 100KB | 正常返回，由调用方处理截断 |

### 改进建议

1. **增加部分失败提示**
   ```typescript
   // 建议：在返回中包含失败的服务器信息
   return {
     data: results.flat(),
     meta: { failedServers: [...] }  // 新增
   }
   ```

2. **支持强制刷新缓存**
   - 添加 `forceRefresh` 参数，绕过缓存直接请求

3. **优化大数据量处理**
   - 当前 `maxResultSizeChars: 100_000` 可能不足以处理大量资源
   - 考虑分页或流式返回

4. **增强类型安全**
   - `ServerResource` 类型在 `src/services/mcp/types.ts` 中定义：
   ```typescript
   export type ServerResource = Resource & { server: string }
   ```
   - 可考虑添加更多字段校验

5. **监控与可观测性**
   - 添加资源数量指标上报
   - 记录缓存命中率

### 测试建议

1. 测试无服务器配置时的行为
2. 测试部分服务器失败时的错误隔离
3. 测试缓存失效机制（模拟 onclose 事件）
4. 测试大数据量下的性能表现
