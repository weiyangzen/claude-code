# toolRendering.tsx 深度研究文档

## 场景与职责

`toolRendering.tsx` 是 **Claude in Chrome** MCP 工具在终端 UI 中的**自定义渲染层**。由于 Claude in Chrome 的工具调用（如 `navigate`、`computer`、`tabs_create_mcp`）具有独特的业务语义，默认的 MCP 工具渲染（仅显示工具名和原始 JSON 参数）用户体验较差。该模块通过覆盖以下四个渲染钩子，为终端提供更具可读性的展示：

1. **`userFacingName`**：将内部工具名（如 `tabs_context_mcp`）转换为更友好的显示名。
2. **`renderToolUseMessage`**：在工具调用时显示精简的上下文摘要（如目标 URL、搜索关键词、鼠标操作坐标）。
3. **`renderToolUseTag`**：在工具调用旁渲染可点击的 "[View Tab]" 超链接（若终端支持 hyperlinks）。
4. **`renderToolResultMessage`**：工具执行成功后显示一句话结果摘要（如 "Navigation completed"）。

该模块采用 React/ink 组件，仅在 TUI（终端用户界面）环境下生效。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `ChromeToolName` | 类型定义，列出所有 18 个 Claude in Chrome 工具名，与 `@ant/claude-for-chrome-mcp` 的 `BROWSER_TOOLS` 保持一致。 |
| `renderChromeToolUseMessage()` | 根据工具名和输入参数生成人类可读的调用摘要。 |
| `renderChromeViewTabLink()` | 若输入包含 `tabId` 且终端支持超链接，渲染指向 `https://clau.de/chrome/tab/<tabId>` 的可点击链接。 |
| `renderChromeToolResultMessage()` | 为每个工具返回一个固定的成功摘要字符串，在 `verbose === false` 时提供简洁反馈。 |
| `getClaudeInChromeMCPToolOverrides()` | 工厂函数，返回完整的工具渲染覆盖对象，供 `src/services/mcp/client.ts` 在构建工具列表时 spread 注入。 |

## 具体技术实现

### 1. ChromeToolName 类型

```typescript
export type ChromeToolName = 
  'javascript_tool' | 'read_page' | 'find' | 'form_input' | 'computer' | 
  'navigate' | 'resize_window' | 'gif_creator' | 'upload_image' | 'get_page_text' | 
  'tabs_context_mcp' | 'tabs_create_mcp' | 'update_plan' | 'read_console_messages' | 
  'read_network_requests' | 'shortcuts_list' | 'shortcuts_execute';
```

> 注释强调该类型必须与 `@ant/claude-for-chrome-mcp` 的 `BROWSER_TOOLS` 数组保持同步。

### 2. 工具调用消息渲染 (`renderChromeToolUseMessage()`)

#### 通用行为

- 若 `input.tabId` 为数字，先调用 `trackClaudeInChromeTabId(tabId)` 进行标签页追踪。
- 根据 `toolName` 进入 `switch` 分支，提取关键参数构建 `secondaryInfo` 数组。
- 返回 `secondaryInfo.join(', ')`；若为空数组则返回 `null`（表示不显示额外参数摘要）。

#### 各工具分支详情

| 工具名 | 渲染逻辑 |
|--------|----------|
| `navigate` | 解析 `input.url` 的 `hostname`；若解析失败则截断显示原始 URL（最多 30 字符）。 |
| `find` | 显示 `pattern: <query>`（截断 30 字符）。 |
| `computer` | 最复杂的分支，支持 `left_click`/`right_click`/`double_click`/`middle_click`（显示 ref 或 coordinate）、`type`（显示输入文本）、`key`（显示按键）、`scroll`（显示方向）、`wait`（显示秒数）、`left_click_drag`（显示 drag）等。 |
| `gif_creator` | 显示 `action` 值。 |
| `resize_window` | 显示 `width x height`。 |
| `read_console_messages` | 显示 `pattern` 和/或 `errors only`。 |
| `read_network_requests` | 显示 `urlPattern`。 |
| `shortcuts_execute` | 显示 `shortcut_id`。 |
| `javascript_tool` | **特殊处理**：`verbose === true` 时直接返回完整代码；`verbose === false` 时返回空字符串 `''`（保留工具头但隐藏参数，避免破坏 View Tab 布局）。 |
| `tabs_create_mcp`, `tabs_context_mcp`, `form_input`, `shortcuts_list`, `read_page`, `upload_image`, `get_page_text`, `update_plan` | 返回 `''`（不显示额外参数）。 |

#### `truncateToWidth` 的使用

对可能较长的字符串（URL、查询词、代码、文本输入）使用 `truncateToWidth(text, maxWidth)` 进行截断，防止终端行溢出。

### 3. View Tab 超链接 (`renderChromeViewTabLink()`)

```typescript
const CHROME_EXTENSION_FOCUS_TAB_URL_BASE = 'https://clau.de/chrome/tab/'
```

渲染条件：
1. `supportsHyperlinks()` 返回 `true`（终端支持 OSC 8 超链接协议）。
2. `input` 是对象且包含 `tabId`。
3. `tabId` 能成功解析为数字（支持 `number` 和 `string` 类型）。

渲染结果示例：
```
Claude in Chrome[navigate] example.com [View Tab]
                              ^^^^^^^^^^^^^^
                              可点击，指向 https://clau.de/chrome/tab/123
```

### 4. 工具结果摘要 (`renderChromeToolResultMessage()`)

行为：
- 若 `verbose === true`，回退到默认的 MCP 工具结果渲染（显示完整结构化输出）。
- 若 `verbose === false`，按工具名返回固定的一句话摘要：

| 工具名 | 摘要 |
|--------|------|
| `navigate` | Navigation completed |
| `tabs_create_mcp` | Tab created |
| `tabs_context_mcp` | Tabs read |
| `computer` | Action completed |
| `find` | Search completed |
| `javascript_tool` | Script executed |
| `read_page` | Page read |
| `read_console_messages` | Console messages retrieved |
| `read_network_requests` | Network requests retrieved |
| `shortcuts_list` | Shortcuts retrieved |
| `shortcuts_execute` | Shortcut executed |
| `resize_window` | Window resized |
| `gif_creator` | GIF action completed |
| `form_input` | Input completed |
| `upload_image` | Image uploaded |
| `get_page_text` | Page text retrieved |
| `update_plan` | Plan updated |

返回组件：
```tsx
<MessageResponse height={1}>
  <Text dimColor>{summary}</Text>
</MessageResponse>
```

### 5. 覆盖对象工厂 (`getClaudeInChromeMCPToolOverrides()`)

返回对象结构：

```typescript
{
  userFacingName: (input?) => string,
  renderToolUseMessage: (input, { verbose }) => React.ReactNode,
  renderToolUseTag: (input) => React.ReactNode,
  renderToolResultMessage: (output, progressMessages, { verbose }) => React.ReactNode,
}
```

- `userFacingName`：去掉工具名末尾的 `_mcp` 后缀，格式为 `Claude in Chrome[<name>]`。
- `renderToolUseTag`：调用 `renderChromeViewTabLink()`。
- `renderToolResultMessage`：先通过 `isMCPToolResult()` 类型守卫确认输出是对象而非字符串，再调用 `renderChromeToolResultMessage()`。

## 关键代码路径与文件引用

```
src/services/mcp/client.ts:1977-1982
  └── isClaudeInChromeMCPServer(client.name)
      └── claudeInChromeToolRendering()
          └── require('../../utils/claudeInChrome/toolRendering.js')
              └── getClaudeInChromeMCPToolOverrides(tool.name)
                  ├── userFacingName()        → "Claude in Chrome[navigate]"
                  ├── renderToolUseMessage()  → renderChromeToolUseMessage()
                  │   └── trackClaudeInChromeTabId()  ← common.ts
                  ├── renderToolUseTag()      → renderChromeViewTabLink()
                  │   └── supportsHyperlinks()        ← src/ink/supports-hyperlinks.js
                  └── renderToolResultMessage() → renderChromeToolResultMessage()
                      └── renderDefaultMCPToolResultMessage() ← src/tools/MCPTool/UI.js
```

### 调用方与依赖

| 依赖 | 用途 |
|------|------|
| `react` | React 组件与类型 |
| `src/components/MessageResponse.js` | 结果摘要的容器组件 |
| `src/ink/supports-hyperlinks.js` | 检测终端是否支持 OSC 8 超链接 |
| `src/ink.js` | `Link`, `Text` 组件 |
| `src/tools/MCPTool/UI.js` | 默认 MCP 工具结果渲染（verbose 模式回退） |
| `src/utils/mcpValidation.js` | `MCPToolResult` 类型 |
| `src/utils/format.js` | `truncateToWidth()` |
| `./common.js` | `trackClaudeInChromeTabId()` |
| `@modelcontextprotocol/sdk/types.js` | `Tool` 类型（re-export） |

### 懒加载设计

在 `src/services/mcp/client.ts` 中：

```typescript
const claudeInChromeToolRendering =
  (): typeof import('../../utils/claudeInChrome/toolRendering.js') =>
    require('../../utils/claudeInChrome/toolRendering.js')
```

- 由于 `toolRendering.tsx` 会拉入 `react` 和 `ink` 整个组件树，该模块被设计为**懒加载**。
- 仅在 Claude in Chrome MCP server 实际连接且为 `stdio` 类型时才被 `require`，避免对不使用 Chrome 功能的会话造成启动开销。

## 风险、边界与改进建议

### 风险

1. **工具名与 `@ant/claude-for-chrome-mcp` 不同步**：`ChromeToolName` 是硬编码的 18 个字符串联合类型。若上游包新增工具（如 `pdf_export`）或重命名现有工具，TypeScript 编译不会自动报错，运行时 `renderChromeToolUseMessage` 会落入 `default` 分支返回空摘要，导致用户体验降级。
2. **`computer` 工具分支的维护负担**：`computer` 动作类型最多，随着浏览器自动化能力扩展（如新增 `hover`、`focus`），`switch` 语句会持续增长，容易遗漏新动作的友好渲染。
3. **View Tab 链接的可用性依赖外部短链服务**：`https://clau.de/chrome/tab/<tabId>` 是一个短链/重定向服务，若该域名不可用或网络不通，超链接将失效。
4. **`javascript_tool` 在 verbose 模式下的完整代码暴露**：当 `verbose === true` 时直接显示完整 JS 代码，若代码中包含敏感信息（如内联 token、密码），可能在终端历史中留下痕迹。

### 边界

- 该模块**仅在 TUI 环境**下生效；VS Code 扩展、Web UI 等其他前端不会执行这些 React/ink 组件。
- `renderToolResultMessage` 的简洁摘要只在 `verbose === false` 时显示；verbose 模式下回退到默认渲染，可能输出大量 JSON。
- `supportsHyperlinks()` 在大多数 Windows 终端（如旧版 cmd、部分 PowerShell 配置）中返回 false，因此 `[View Tab]` 链接对 Windows 用户不可见。
- `trackClaudeInChromeTabId` 的副作用发生在渲染阶段（`renderToolUseMessage` 被调用时），而非实际工具执行阶段。这在绝大多数情况下无差异，但在某些 UI 重渲染场景可能多次触发追踪。

### 改进建议

1. **编译时工具名同步检查**：在 CI 或构建脚本中增加断言，验证 `ChromeToolName` 联合类型与 `@ant/claude-for-chrome-mcp` 导出的 `BROWSER_TOOLS.map(t => t.name)` 完全一致。
2. **`computer` 动作渲染表驱动化**：将 `computer` 工具的各动作渲染逻辑从 `switch-case` 改为配置表（`Record<action, Renderer>`），新增动作时只需追加表项，降低维护成本。
3. **敏感代码脱敏**：在 `javascript_tool` 的 verbose 渲染中，对常见敏感模式（如 `password`、`token`、`api_key` 赋值）进行自动检测并高亮/截断，减少信息泄露风险。
4. **Windows 终端超链接降级方案**：当 `supportsHyperlinks()` 为 false 时，可考虑在工具调用消息后追加纯文本提示（如 `Tab 123: https://clau.de/chrome/tab/123`），让 Windows 用户也能复制访问。
5. **将 `trackClaudeInChromeTabId` 移出渲染层**：将 tab ID 追踪逻辑下沉到实际工具调用执行层（如 `MCPTool.call()` 或 MCP client 的 `callTool` 中），确保副作用与业务执行严格一一对应，避免重渲染带来的重复追踪。
6. **增加快照测试**：对 `renderChromeToolUseMessage` 和 `renderChromeToolResultMessage` 的各分支输出建立 React 组件快照测试，防止未来重构意外改变终端展示效果。
