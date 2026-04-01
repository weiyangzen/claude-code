# MCPTool.ts 研究文档

## 场景与职责

MCPTool.ts 是 Claude Code 中 MCP (Model Context Protocol) 工具的**基础定义文件**。它定义了 MCP 工具的骨架结构，但实际运行时的具体行为（如名称、描述、调用逻辑等）会在 `src/services/mcp/client.ts` 中被动态覆盖。

### 核心职责
1. **工具骨架定义**：使用 `buildTool` 工厂函数创建 MCP 工具的基础结构
2. **类型导出**：导出 `MCPProgress` 类型供其他模块使用
3. **Schema 定义**：定义输入/输出的 Zod Schema（宽松透传模式）
4. **UI 渲染绑定**：绑定工具使用消息、进度消息、结果消息的渲染函数

### 使用场景
- 当用户配置并连接 MCP 服务器时，`client.ts` 会基于 MCPTool 创建具体的工具实例
- 作为所有 MCP 工具的模板，被 `fetchToolsForClient` 函数使用展开运算符 `...MCPTool` 复制

---

## 功能点目的

### 1. 输入输出 Schema

```typescript
// 允许任意输入对象，因为 MCP 工具自己定义 schema
export const inputSchema = lazySchema(() => z.object({}).passthrough())

// 输出为字符串
export const outputSchema = lazySchema(() =>
  z.string().describe('MCP tool execution result'),
)
```

**设计意图**：
- MCP 工具由外部服务器提供，每个工具有自己的参数 schema
- 客户端无法预先知道所有 MCP 工具的参数结构
- 使用 `passthrough()` 允许任意字段通过验证

### 2. 工具定义结构

```typescript
export const MCPTool = buildTool({
  isMcp: true,                    // 标记为 MCP 工具
  isOpenWorld() { return false }, // 在 client.ts 中被覆盖
  name: 'mcp',                    // 基础名称，实际会被覆盖为 mcp__server__tool
  maxResultSizeChars: 100_000,    // 最大结果大小限制
  description() { return DESCRIPTION }, // 在 client.ts 中被覆盖
  prompt() { return PROMPT },      // 在 client.ts 中被覆盖
  call() { return { data: '' } },  // 在 client.ts 中被覆盖
  checkPermissions() { ... },      // 返回 passthrough，实际权限检查在 client.ts
  renderToolUseMessage,            // 来自 UI.tsx
  renderToolUseProgressMessage,    // 来自 UI.tsx
  renderToolResultMessage,         // 来自 UI.tsx
  userFacingName: () => 'mcp',
  isResultTruncated(output) { ... },
  mapToolResultToToolResultBlockParam(content, toolUseID) { ... },
})
```

### 3. MCPProgress 类型重导出

```typescript
export type { MCPProgress } from '../../types/tools.js'
```

**目的**：打破循环依赖。实际定义在 `src/Tool.ts` 中集中管理。

---

## 具体技术实现

### 关键流程

#### 1. 工具实例化流程（在 client.ts 中）

```typescript
// fetchToolsForClient 函数中的工具创建
return {
  ...MCPTool,  // 复制基础定义
  name: skipPrefix ? tool.name : fullyQualifiedName,  // 覆盖名称
  mcpInfo: { serverName: client.name, toolName: tool.name },
  async description() { return tool.description ?? '' },  // 覆盖描述
  async prompt() { ... },  // 覆盖 prompt
  async call(args, context, _canUseTool, parentMessage, onProgress) {
    // 实际的工具调用逻辑
  },
  // ... 其他覆盖
}
```

#### 2. 权限检查流程

MCPTool.ts 中的 `checkPermissions` 返回 `passthrough`，表示：
- 基础工具定义不处理权限
- 实际权限检查在 client.ts 中通过 `checkPermissions` 方法实现
- 提示用户可以添加规则到 localSettings

#### 3. 结果截断检测

```typescript
isResultTruncated(output: Output): boolean {
  return isOutputLineTruncated(output)
}
```

使用 `src/utils/terminal.ts` 中的函数检测输出是否需要截断显示。

### 数据结构

#### MCPTool 对象结构

| 属性 | 类型 | 说明 |
|------|------|------|
| `isMcp` | `boolean` | 标记为 MCP 工具 |
| `name` | `string` | 工具名称（基础值 'mcp'） |
| `maxResultSizeChars` | `number` | 100,000 字符限制 |
| `inputSchema` | `z.ZodType` | 透传对象 schema |
| `outputSchema` | `z.ZodType` | 字符串 schema |
| `call` | `Function` | 工具执行函数（占位） |
| `description` | `Function` | 获取描述（占位） |
| `prompt` | `Function` | 获取 prompt（占位） |
| `checkPermissions` | `Function` | 权限检查 |
| `renderToolUseMessage` | `Function` | 渲染工具使用消息 |
| `renderToolUseProgressMessage` | `Function` | 渲染进度消息 |
| `renderToolResultMessage` | `Function` | 渲染结果消息 |
| `isResultTruncated` | `Function` | 检测结果是否截断 |
| `mapToolResultToToolResultBlockParam` | `Function` | 映射到 API 格式 |

---

## 关键代码路径与文件引用

### 直接依赖

```
MCPTool.ts
├── zod/v4                          # Schema 验证
├── ../../Tool.js                   # buildTool 工厂函数
├── ../../utils/lazySchema.js       # 延迟 schema 构造
├── ../../utils/permissions/PermissionResult.js  # 权限结果类型
├── ../../utils/terminal.js         # isOutputLineTruncated
├── ./prompt.js                     # DESCRIPTION, PROMPT 占位符
└── ./UI.js                         # 渲染函数
```

### 被引用位置

```
src/services/mcp/client.ts
├── import { type MCPProgress, MCPTool } from '../../tools/MCPTool/MCPTool.js'
└── 在 fetchToolsForClient 中使用 ...MCPTool 展开

src/Tool.ts
├── 从 './types/tools.js' 导入 MCPProgress（循环重导出）
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `buildTool` | 工具工厂函数，提供默认值填充 |
| `lazySchema` | 延迟构造 Zod schema，避免模块加载时初始化 |
| `isOutputLineTruncated` | 检测输出是否需要截断显示 |
| `UI.tsx` 导出 | 渲染工具使用/进度/结果消息 |

### 被覆盖的方法（在 client.ts 中）

| 方法 | 覆盖内容 |
|------|----------|
| `name` | 实际工具名称（如 `mcp__slack__send_message`） |
| `description` | 从 MCP 服务器获取的工具描述 |
| `prompt` | 截断后的工具描述 |
| `call` | 实际的 MCP 工具调用逻辑 |
| `isOpenWorld` | 基于 `tool.annotations?.openWorldHint` |
| `userFacingName` | 带服务器前缀的显示名称 |
| `checkPermissions` | 添加规则建议的权限检查 |

---

## 风险、边界与改进建议

### 当前风险

1. **占位符方法**：`call()`、`description()`、`prompt()` 都是空实现，依赖 client.ts 覆盖
   - 风险：如果 client.ts 未正确覆盖，工具将无法正常工作
   - 缓解：MCPTool 不直接导出使用，仅作为模板

2. **宽松的输入 Schema**：`z.object({}).passthrough()` 接受任意输入
   - 风险：无效参数可能在后期才被发现
   - 缓解：MCP SDK 会在服务器端验证

3. **循环依赖风险**：通过 `types/tools.js` 重导出 MCPProgress
   - 当前有注释说明是为了打破循环依赖

### 边界情况

1. **空输入处理**：`renderToolUseMessage` 在输入为空对象时返回空字符串
2. **结果截断**：`isResultTruncated` 基于换行符数量判断，可能不适用于单行超长内容

### 改进建议

1. **类型安全**：考虑使用更精确的输入类型，而非完全透传
2. **文档完善**：添加更多 JSDoc 注释说明哪些方法需要被覆盖
3. **错误处理**：在占位符方法中添加警告日志，提示开发者需要覆盖
4. **测试覆盖**：添加单元测试确保 client.ts 正确覆盖所有必要方法

---

## 附录：相关配置

### 环境变量影响

| 变量 | 影响 |
|------|------|
| `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` | 控制是否跳过 `mcp__` 前缀 |
| `MAX_MCP_OUTPUT_TOKENS` | 控制 MCP 输出截断阈值 |

### 相关常数

```typescript
maxResultSizeChars: 100_000  // 最大结果字符数
```
