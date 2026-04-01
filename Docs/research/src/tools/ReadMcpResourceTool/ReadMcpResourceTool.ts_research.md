# ReadMcpResourceTool.ts 研究文档

## 场景与职责

`ReadMcpResourceTool.ts` 是 Claude Code 中用于**从已连接的 MCP（Model Context Protocol）服务器读取特定资源**的核心工具实现。它属于 `src/tools/ReadMcpResourceTool/` 目录下的主逻辑文件，与 `ListMcpResourcesTool` 共同构成 MCP 资源发现与读取的完整闭环：

- `ListMcpResourcesTool`：发现/枚举各 MCP 服务器提供的资源列表。
- `ReadMcpResourceTool`：按指定的 `server` + `uri` 从目标服务器读取资源内容。

该工具在交互式 REPL、后台 Agent 以及 SDK 模式下均可被模型调用，属于**只读（read-only）**且**并发安全（concurrency-safe）**的工具。在 `src/tools.ts` 的全局工具注册表中被标记为 `specialTools` 之一，并在 MCP 客户端连接建立后由 `src/services/mcp/client.ts` 动态注入可用工具列表。

## 功能点目的

1. **资源定位**：通过输入参数 `server`（MCP 服务器名称）和 `uri`（资源 URI）唯一定位目标资源。
2. **连接校验**：在调用前校验目标服务器是否存在于当前会话的 `mcpClients` 中，且处于 `connected` 状态，并支持 `resources` 能力。
3. **协议读取**：通过 MCP SDK 向服务器发起 `resources/read` JSON-RPC 请求，获取资源内容。
4. **二进制内容拦截**：对返回的 `blob`（base64 编码二进制数据）进行拦截——解码后写入本地磁盘（tool-results 目录），避免将大段 base64 直接塞入模型上下文。
5. **结果格式化**：将处理后的内容数组包装为 `ToolResult` 返回，并提供截断检测与 `tool_result` 块参数映射。

## 具体技术实现

### 关键数据结构

```ts
// 输入模式（延迟构造）
export const inputSchema = lazySchema(() =>
  z.object({
    server: z.string().describe('The MCP server name'),
    uri: z.string().describe('The resource URI to read'),
  }),
)

// 输出模式
export const outputSchema = lazySchema(() =>
  z.object({
    contents: z.array(
      z.object({
        uri: z.string(),
        mimeType: z.string().optional(),
        text: z.string().optional(),
        blobSavedTo: z.string().optional(), // 二进制内容持久化后的本地路径
      }),
    ),
  }),
)
```

### 核心执行流程（`call` 方法）

1. **解析输入**：从 `input` 提取 `serverName` 和 `uri`。
2. **查找客户端**：
   ```ts
   const client = mcpClients.find(client => client.name === serverName)
   ```
   若未找到，抛出错误并附带可用服务器列表。
3. **状态校验**：
   - `client.type !== 'connected'` → 报错“未连接”。
   - `!client.capabilities?.resources` → 报错“该服务器不支持 resources 能力”。
4. **确保连接 freshness**：
   ```ts
   const connectedClient = await ensureConnectedClient(client)
   ```
   该函数在 `src/services/mcp/client.ts` 中实现。对于非 SDK 服务器，它会通过 `connectToServer` 重新验证/重建连接（内部有 memoize 缓存，健康连接为 no-op）。
5. **发起 MCP 请求**：
   ```ts
   const result = await connectedClient.client.request(
     { method: 'resources/read', params: { uri } },
     ReadResourceResultSchema,
   )
   ```
   使用 `@modelcontextprotocol/sdk` 的 `Client.request`，遵循 JSON-RPC 2.0 规范。
6. **内容后处理（二进制拦截）**：
   - 遍历 `result.contents`。
   - 若条目含 `text`，直接透传。
   - 若条目含 `blob` 且为字符串：
     - 生成唯一持久化 ID：`mcp-resource-${Date.now()}-${i}-${random}`。
     - `Buffer.from(c.blob, 'base64')` 解码。
     - 调用 `persistBinaryContent(bytes, c.mimeType, persistId)` 写入 `tool-results/` 目录，扩展名由 MIME 类型推导（`src/utils/mcpOutputStorage.ts`）。
     - 若写入失败，回退为文本错误说明。
     - 若写入成功，将 `blobSavedTo` 设为本地路径，并在 `text` 中附加 `getBinaryBlobSavedMessage(...)` 生成的说明文本。
7. **返回结果**：
   ```ts
   return { data: { contents } }
   ```

### 截断与序列化

- `isResultTruncated`：使用 `isOutputLineTruncated(jsonStringify(output))` 快速检测输出是否超过 3 行（基于换行符计数），用于控制 UI 的展开/折叠行为。
- `mapToolResultToToolResultBlockParam`：将输出序列化为 JSON 字符串后封装为 Anthropic API 的 `tool_result` 块。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` | 本文件，工具主逻辑 |
| `src/tools/ReadMcpResourceTool/UI.tsx` | 渲染工具使用与结果消息 |
| `src/tools/ReadMcpResourceTool/prompt.ts` | 描述与提示词文本 |
| `src/Tool.ts` | `buildTool`、`ToolDef`、`ToolUseContext` 定义 |
| `src/services/mcp/client.ts` | `ensureConnectedClient`、连接管理、MCP SDK 封装 |
| `src/services/mcp/types.ts` | `ConnectedMCPServer`、`MCPServerConnection`、`ServerResource` 类型 |
| `src/utils/mcpOutputStorage.ts` | `persistBinaryContent`、`getBinaryBlobSavedMessage` |
| `src/utils/slowOperations.ts` | `jsonStringify`（带慢操作日志） |
| `src/utils/terminal.ts` | `isOutputLineTruncated` |
| `src/utils/lazySchema.ts` | `lazySchema`（延迟构造 Zod schema） |
| `src/tools.ts` | 全局工具注册，将 `ReadMcpResourceTool` 注入 `specialTools` 与普通工具列表 |

## 依赖与外部交互

### 运行时依赖

- **@modelcontextprotocol/sdk**：`ReadResourceResultSchema`、`Client.request` 用于标准 MCP 资源读取协议。
- **zod/v4**：输入/输出 schema 定义与运行时校验。

### 内部服务依赖

- **`ensureConnectedClient`**（`src/services/mcp/client.ts`）：
  - 对 SDK 类型服务器直接返回原 client。
  - 对其他类型通过 `connectToServer`（memoized）重新获取连接，以处理会话过期后的重连。
- **`persistBinaryContent`** / **`getBinaryBlobSavedMessage`**（`src/utils/mcpOutputStorage.ts`）：
  - 将二进制内容写入 `$CLAUDE_CODE_RESULTS_DIR/`（或默认的 `tool-results`）。
  - 支持根据 MIME 类型推导扩展名（pdf、png、jpg、mp3、mp4 等）。
- **`jsonStringify`**（`src/utils/slowOperations.ts`）：
  - 包装了 `JSON.stringify`，在 ANT 构建中会对超过阈值的慢操作进行日志记录。

### 上下文数据

工具通过 `ToolUseContext.options.mcpClients` 获取当前会话中所有 MCP 服务器连接状态数组。该数组由 `src/services/mcp/client.ts` 在启动或重连时维护，并随 `AppState` 流转。

## 风险、边界与改进建议

### 风险与边界

1. **二进制内容大小未限制**：
   - 当前代码对 `c.blob` 的 base64 解码和 `Buffer` 创建没有显式大小限制。若 MCP 服务器返回超大 blob（如数百 MB 的视频），可能导致内存峰值或 OOM。
   - `persistBinaryContent` 本身也不限制写入大小。

2. **会话过期与重连竞态**：
   - `ensureConnectedClient` 内部依赖 memoized `connectToServer`。在并发调用下，若连接在 `ensureConnectedClient` 与 `client.request` 之间断开，可能抛出 `McpSessionExpiredError`。虽然外层有重试逻辑（在 `MCPTool` 或 `client.ts` 中），但 `ReadMcpResourceTool` 本身没有本地重试。

3. **URI 合法性未校验**：
   - 输入 schema 仅要求 `uri` 为字符串，未对其格式（如是否包含非法字符、是否符合 MCP URI 规范）做进一步校验，非法 URI 会直接透传给 MCP 服务器，依赖服务器返回错误。

4. **错误信息泄露可用服务器列表**：
   - 当 `server` 未找到时，错误消息会拼接 `mcpClients.map(c => c.name).join(', ')`，在极少数敏感部署场景下可能泄露内部服务器名称。

5. **结果截断检测过于简单**：
   - `isOutputLineTruncated` 仅按换行符计数，单条极长文本（无换行）可能误判为未截断，导致 UI 展开行为不一致。

### 改进建议

1. **增加 blob 大小上限**：在解码前检查 `c.blob.length`（base64 长度对应原始字节数约为 `length * 3/4`），超过阈值时拒绝加载或流式写入，并返回友好的错误提示。
2. **URI 格式预校验**：增加可选的 URI 格式校验（如禁止空字符串、控制字符），提前失败以减少对 MCP 服务器的无效请求。
3. **统一错误分类**：将连接失败、能力不支持、资源不存在等错误映射为更结构化的错误码，便于上层（如 SDK 模式）做差异化处理。
4. **考虑支持资源订阅/变更通知**：MCP 规范支持 `resources/subscribe` 与 `resources/list_changed`，当前仅实现一次性读取。未来可在读取后根据资源元数据决定是否订阅变更，以提升长会话中的信息新鲜度。
5. **日志与可观测性增强**：在 `call` 中增加 `logMCPDebug` 或 `logEvent`，记录读取的 server/uri/mimeType/内容条目数，便于排查“模型为何读不到资源”的问题。
