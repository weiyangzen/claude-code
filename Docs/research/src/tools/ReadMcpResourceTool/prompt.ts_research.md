# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 `ReadMcpResourceTool` 的**提示词与描述文本源文件**，职责单一且明确：为工具提供面向大语言模型（LLM）的说明性文本。它导出了两个字符串常量：

- `DESCRIPTION`：工具的简短功能描述，用于系统提示（system prompt）中的工具列表展示。
- `PROMPT`：更详细的参数说明与使用指南，在模型需要理解工具调用方式时提供上下文。

该文件作为“纯文本配置”存在，与工具的业务逻辑（`ReadMcpResourceTool.ts`）和 UI 渲染（`UI.tsx`）完全解耦，符合 Claude Code 中多数工具的代码组织惯例。

## 功能点目的

1. **DESCRIPTION —— 工具摘要**：
   - 在系统提示的工具枚举段落中，向模型说明 `ReadMcpResourceTool` 的核心能力：从指定的 MCP 服务器读取特定资源。
   - 包含参数速览（`server`、`uri`）和一个最小化的调用示例，降低模型的使用门槛。

2. **PROMPT —— 参数详解**：
   - 在需要更详细说明的场景（如 `tool.prompt()` 被调用时）向模型强调参数含义。
   - 明确标记 `server` 和 `uri` 为 required，减少模型因漏填参数导致的调用失败。

## 具体技术实现

### 源码结构

```ts
export const DESCRIPTION = `
Reads a specific resource from an MCP server.
- server: The name of the MCP server to read from
- uri: The URI of the resource to read

Usage examples:
- Read a resource from a server: \`readMcpResource({ server: "myserver", uri: "my-resource-uri" })\`
`

export const PROMPT = `
Reads a specific resource from an MCP server, identified by server name and resource URI.

Parameters:
- server (required): The name of the MCP server from which to read the resource
- uri (required): The URI of the resource to read
`
```

### 使用方式

在 `ReadMcpResourceTool.ts` 中，这两个文本通过异步 getter 方法注入到 `buildTool` 定义：

```ts
import { DESCRIPTION, PROMPT } from './prompt.js'

export const ReadMcpResourceTool = buildTool({
  // ...
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return PROMPT
  },
  // ...
})
```

`buildTool`（`src/Tool.ts`）会将它们挂载到 `Tool` 接口的 `description()` 和 `prompt()` 方法上。上层系统（如 `src/utils/systemPrompt.ts` 或 `src/services/tools/toolAssembly.ts`）在构建发给 Anthropic API 的系统提示时，会调用这些方法获取文本。

### 文本设计特点

- **简洁性**：两段文本都控制在 10 行以内，避免占用过多 prompt tokens。
- **示例驱动**：`DESCRIPTION` 中提供了一个可直接复制的 JavaScript/TypeScript 调用示例，符合 Claude Code 中工具提示词的常见风格。
- **参数明确**：`PROMPT` 使用 `(required)` 标签强调必填项，与 `inputSchema`（Zod schema）的约束保持一致。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ReadMcpResourceTool/prompt.ts` | 本文件，描述与提示词文本 |
| `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` | 导入 `DESCRIPTION` 和 `PROMPT` 并注册到工具定义 |
| `src/Tool.ts` | `Tool` 接口定义，`description()` 与 `prompt()` 的契约说明 |
| `src/utils/systemPromptType.ts` / `src/utils/systemPrompt.ts`（推测） | 系统提示组装逻辑，实际调用 `tool.description()` 与 `tool.prompt()` |

## 依赖与外部交互

- **无运行时外部依赖**：该文件不导入任何第三方库或内部模块，仅导出两个模板字符串常量。
- **与 Zod schema 的隐性契约**：`prompt.ts` 中描述的参数名称（`server`、`uri`）和必填属性必须与 `ReadMcpResourceTool.ts` 中的 `inputSchema` 严格一致。若一方发生变更而另一方未同步，会导致模型收到错误的参数说明，增加调用失败率。
- **与 `userFacingName` 的隐性契约**：`DESCRIPTION` 中的示例使用了 `readMcpResource(...)`，这与 `UI.tsx` 中 `userFacingName()` 返回的 `readMcpResource` 保持一致。

## 风险、边界与改进建议

### 风险与边界

1. **文本与实现不同步**：
   - 当前 `prompt.ts` 是独立维护的纯文本文件，没有自动化机制保证 `DESCRIPTION`/`PROMPT` 与 `inputSchema` 的字段、类型、必填性保持一致。若后续在 `inputSchema` 中新增参数（如 `offset`、`limit`）或修改描述，容易遗漏更新 `prompt.ts`。

2. **国际化缺失**：
   - 文本为硬编码英文，Claude Code 若未来支持多语言系统提示，需要重构为基于 locale 的文本加载机制。

3. **提示词工程效果未量化**：
   - `DESCRIPTION` 和 `PROMPT` 的内容没有经过 A/B 测试或显式的 prompt 工程迭代记录。示例的格式（`readMcpResource({...})`）是否比 `ReadMcpResourceTool({...})` 更利于模型理解，缺乏数据支撑。

4. **无动态上下文注入**：
   - 与某些工具（如 `BashTool`）的动态 prompt 不同，`ReadMcpResourceTool` 的 prompt 是静态字符串，无法根据当前会话中实际可用的 MCP 服务器列表或资源 URI 模式进行个性化补充。模型在调用时可能因对 `server` 名称记忆模糊而选择错误的服务器。

### 改进建议

1. **从 Schema 自动生成 Prompt 片段**：
   - 借鉴 OpenAPI / MCP 的 schema-to-prompt 技术，通过脚本或构建时工具从 `inputSchema` 和 `outputSchema` 自动生成参数说明表格，减少人工维护成本并保证一致性。
   - 例如，在 `buildTool` 中增加可选的 `autoPromptFromSchema: true`，自动拼接 `zod` 的 `.describe()` 文本。

2. **动态增强 Prompt**：
   - 将 `prompt()` 方法改造为异步函数，接收当前 `mcpClients` 列表，在提示词末尾追加一句：
     > "Currently connected MCP servers: serverA, serverB, serverC."
   - 这能显著降低模型因记错服务器名称而导致的调用失败。

3. **统一示例风格**：
   - 检查所有 MCP 相关工具的 `DESCRIPTION`，确保示例代码风格一致（引号类型、对象格式、函数调用形式），提升模型对工具族的整体理解。

4. **增加边界行为说明**：
   - 在 `PROMPT` 中补充一句关于二进制资源的说明，例如：
     > "If the resource contains binary data (e.g., images, PDFs), the content will be saved to disk and the result will include the file path."
   - 这能让模型在读取二进制资源前就有正确预期，减少后续对 `text` 字段的误解。

5. **版本化 Prompt 管理**：
   - 若后续对 `DESCRIPTION` 或 `PROMPT` 进行实验性调整，建议通过 GrowthBook feature flag 或类似机制做灰度，以便观察对工具调用准确率的影响。
