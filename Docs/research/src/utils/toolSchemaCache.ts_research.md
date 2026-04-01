# 研究文档：src/utils/toolSchemaCache.ts

## 场景与职责

本模块提供**会话级（session-scoped）工具 schema 缓存**。在 Claude Code 的 prompt 构造中，工具 schema 块位于 server position 2（紧接在系统提示之后），任何字节级的变动都会连带 bust 整个约 11K token 的工具块以及下游所有内容的 prompt cache。Mid-session 的 GrowthBook flag 翻转、MCP 服务器重连、或动态 `tool.prompt()` 内容变化都可能触发这种 churn。通过将该模块设计为**叶子节点（leaf module）**，在首次渲染后锁定 schema 字节，使得 mid-session 的变动不再破坏 cache。

模块刻意保持极简且无复杂依赖，以便 `auth.ts` 可以在不引入 `api.ts` 的情况下清空缓存（避免 `plans → settings → file → growthbook → config → bridgeEnabled → auth` 循环）。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `getToolSchemaCache()` | 返回模块级 `Map<string, CachedSchema>` 的引用，供 `api.ts` 在渲染工具列表时读取/写入。 |
| `clearToolSchemaCache()` | 清空该 Map，通常在用户登出、切换账户、或 MCP 服务器状态变化后调用。 |

## 具体技术实现

- 模块级常量 `TOOL_SCHEMA_CACHE = new Map<string, CachedSchema>()`，进程生命周期内唯一实例。
- `CachedSchema` 类型在 `BetaTool` 基础上扩展了两个可选字段：
  - `strict?: boolean` —— 控制工具是否启用严格模式 JSONSchema。
  - `eager_input_streaming?: boolean` —— 与 beta 功能相关的流式输入标记。
- 缓存 key 为工具名字符串；value 为最终渲染后准备发送给 API 的 schema 对象。
- 由于缓存是模块级单例，subagent 或 fork 不会自动隔离；它们共享同一个 schema 缓存池。这在当前架构下是可接受的，因为 schema 内容在同一进程内是一致的。

## 关键代码路径与文件引用

- **主实现**：`src/utils/toolSchemaCache.ts`（26 行）
- **读取缓存（渲染 API 请求）**：`src/utils/api.ts`（`getToolSchemaCache`）
- **清空缓存（认证变化）**：`src/utils/auth.ts`（`clearToolSchemaCache`）
- **清空缓存（登出命令）**：`src/commands/logout/logout.tsx`（`clearToolSchemaCache`）

## 依赖与外部交互

- **`@anthropic-ai/sdk/resources/beta/messages/messages.mjs`**：提供 `BetaTool` 类型定义。
- 无其他内部工具函数或外部 runtime 依赖。

## 风险、边界与改进建议

### 风险

1. **Stale schema 风险**：若 MCP 工具列表发生变化（服务器新增/删除工具）但调用方忘记调用 `clearToolSchemaCache()`，则 `api.ts` 会继续使用旧的缓存 schema，导致 API 请求中的工具定义与实际可用工具不一致。当前清空点覆盖了主要的认证变化路径，但动态 MCP 配置热更新路径需要额外注意。
2. **无失效粒度**：缓存是全部清空（`clear()`），无法针对单个工具或单个 MCP 服务器进行细粒度失效。在 MCP 服务器数量多、变动频繁的场景下，可能导致不必要的重新计数/渲染。

### 边界

- **会话级而非请求级**：缓存不会在每次 API 调用后清空，这意味着一旦某个工具 schema 被缓存，它将固定到进程结束（或显式清空为止）。这是设计意图——保护 prompt cache。
- **无大小上限**：理论上 Map 的大小等于当前会话中所有曾经渲染过的唯一工具名数量。对于正常使用场景（几十到几百个工具），内存占用可忽略。

### 改进建议

1. **增加 MCP 变动监听自动清空**：在 MCP 客户端连接/断开/刷新时，从 `src/services/mcp/client.ts` 或相关事件总线自动触发 `clearToolSchemaCache`，减少人工维护清空点的负担。
2. **引入版本号或 generation 计数**：为每个缓存条目附加一个 `generation` 标记，调用方在已知 schema 源（如 MCP 服务器实例）变化时只需递增 generation，而不必全量清空。实现简单且能保留未变动工具的缓存。
3. **类型安全增强**：当前 `CachedSchema` 是本地扩展类型，若 SDK 的 `BetaTool` 在未来版本增加同名但不同语义的字段，可能产生冲突。可考虑使用 branded type 或明确交叉类型断言来隔离。
