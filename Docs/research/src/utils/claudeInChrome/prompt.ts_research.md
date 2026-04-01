# prompt.ts 深度研究文档

## 场景与职责

`prompt.ts` 是 **Claude in Chrome** 功能模块的**提示词与技能提示工厂**。它集中管理所有与浏览器自动化相关的系统提示文本，负责：

1. 向模型注入 **Claude in Chrome 的使用规范**（GIF 录制、控制台调试、对话框规避、避免死循环、标签页上下文管理）。
2. 在 **ToolSearch 启用时** 追加工具加载前置指令。
3. 提供 **技能激活提示**（skill hint），在会话启动时告知模型浏览器自动化工具的可用性。
4. 区分 **纯浏览器扩展模式** 与 **WebBrowser 工具共存模式** 的技能提示变体。

该模块本身无运行时副作用，所有导出均为纯字符串常量或纯函数，是提示工程（prompt engineering）在代码层面的集中体现。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `BASE_CHROME_PROMPT` | 核心系统提示，包含浏览器自动化的完整操作规范。 |
| `CHROME_TOOL_SEARCH_INSTRUCTIONS` | ToolSearch 启用时的追加指令，要求模型在使用任何 `mcp__claude-in-chrome__*` 工具前先用 `ToolSearch` 加载。 |
| `getChromeSystemPrompt()` | 返回基础提示（不含 ToolSearch 指令），由 `claude.ts` 在请求时根据实际 ToolSearch 状态决定是否追加。 |
| `CLAUDE_IN_CHROME_SKILL_HINT` | 会话启动时的轻量提示，引导模型先调用 `Skill(skill: "claude-in-chrome")` 再使用工具。 |
| `CLAUDE_IN_CHROME_SKILL_HINT_WITH_WEBBROWSER` | 当内置 `WebBrowser` 工具也可用时的变体，区分开发任务（用 WebBrowser）与真实 Chrome 会话（用 claude-in-chrome）。 |

## 具体技术实现

### 1. BASE_CHROME_PROMPT 结构解析

提示词按主题划分为 5 个章节：

#### 1.1 GIF recording
- **强制要求**：多步骤浏览器交互必须使用 `mcp__claude-in-chrome__gif_creator` 录制。
- **细节要求**：动作前后多捕获帧、文件命名要有意义。

#### 1.2 Console log debugging
- 推荐使用 `mcp__claude-in-chrome__read_console_messages` 读取控制台输出。
- **效率建议**：使用 `pattern` 参数做正则过滤，避免输出过大。

#### 1.3 Alerts and dialogs（关键安全提示）
- **严禁触发** JavaScript `alert` / `confirm` / `prompt` / 浏览器模态对话框。
- 原因：模态对话框会阻塞所有浏览器事件，导致扩展无法接收后续命令。
- 规避策略：
  1. 避免点击可能触发 alert 的按钮（如带确认的删除按钮）。
  2. 必须操作时先警告用户。
  3. 使用 `javascript_tool` 检查并 dismiss 已有对话框。
- 恢复机制：若已触发并失去响应，明确告知用户需手动 dismiss。

#### 1.4 Avoid rabbit holes and loops
- 定义 6 种必须停止并询问用户的场景：
  - 意外复杂或偏离主题的浏览器探索
  - 工具调用 2-3 次失败后
  - 浏览器扩展无响应
  - 页面元素不响应点击/输入
  - 页面加载超时
  - 多种方法均无法完成任务
- 要求模型解释尝试过程与失败原因，不得盲目重试。

#### 1.5 Tab context and session startup（关键流程规范）
- **会话启动时必须调用** `mcp__claude-in-chrome__tabs_context_mcp` 获取当前标签页上下文。
- **禁止复用**其他会话的 tab ID。
- 标签页使用规则：
  1. 仅当用户明确要求时才复用现有标签页。
  2. 否则使用 `tabs_create_mcp` 创建新标签页。
  3. 遇到 tab 不存在或导航错误时，重新获取上下文。
  4. tab 被用户关闭或导航出错后，重新调用 `tabs_context_mcp`。

### 2. ToolSearch 指令分离设计

```typescript
export const CHROME_TOOL_SEARCH_INSTRUCTIONS = `**IMPORTANT: Before using any chrome browser tools, you MUST first load them using ToolSearch.**`
```

- 该指令**不直接拼接**在 `BASE_CHROME_PROMPT` 中。
- 由 `src/services/api/claude.ts` 在请求组装阶段，根据**实际的 ToolSearch 启用状态**动态决定是否注入。
- 这种分离避免了：
  - ToolSearch 未启用时，模型收到无意义的加载指令。
  - 提示词中硬编码条件判断，增加维护成本。

### 3. 技能提示的两种变体

| 变体 | 使用场景 | 核心差异 |
|------|----------|----------|
| `CLAUDE_IN_CHROME_SKILL_HINT` | 仅 Claude in Chrome 可用 | 直接声明浏览器自动化可用，强调先调用 skill。 |
| `CLAUDE_IN_CHROME_SKILL_HINT_WITH_WEBBROWSER` | WebBrowser 与 Claude in Chrome 同时可用 | **任务分流**：开发相关（dev servers、JS eval、console、screenshots）用 WebBrowser；真实 Chrome（登录态、OAuth、computer-use）用 claude-in-chrome。 |

## 关键代码路径与文件引用

```
src/utils/claudeInChrome/prompt.ts
  ├── BASE_CHROME_PROMPT
  ├── CHROME_TOOL_SEARCH_INSTRUCTIONS
  ├── getChromeSystemPrompt() → BASE_CHROME_PROMPT
  ├── CLAUDE_IN_CHROME_SKILL_HINT
  └── CLAUDE_IN_CHROME_SKILL_HINT_WITH_WEBBROWSER
```

### 调用方分布

| 调用方 | 使用的导出项 | 用途 |
|--------|-------------|------|
| `src/utils/claudeInChrome/setup.ts` | `getChromeSystemPrompt()` | `setupClaudeInChrome()` 将返回值作为 `systemPrompt` 注入 MCP 配置。 |
| `src/main.tsx` | `CLAUDE_IN_CHROME_SKILL_HINT`, `CLAUDE_IN_CHROME_SKILL_HINT_WITH_WEBBROWSER` | 在 `autoEnableClaudeInChrome` 分支中，将 hint 追加到系统提示末尾。 |
| `src/skills/bundled/claudeInChrome.ts` | `BASE_CHROME_PROMPT` | 构建 skill 激活时的完整提示词（`BASE_CHROME_PROMPT + SKILL_ACTIVATION_MESSAGE + 可选任务`）。 |
| `src/services/api/claude.ts`（推测） | `CHROME_TOOL_SEARCH_INSTRUCTIONS` | 在组装 API 请求时，根据 ToolSearch 开关动态注入。 |
| `src/utils/attachments.ts`（推测） | 可能引用 skill hint 相关常量 | 处理附件上下文中的 skill 提示。 |

## 依赖与外部交互

该模块**零运行时依赖**，无 `import` 任何本地工具函数或第三方库。所有导出均为：
- 模板字符串字面量（`BASE_CHROME_PROMPT`）
- 简单包装函数（`getChromeSystemPrompt`）

这种设计确保了：
- **构建时确定性**：提示词内容不会被 tree-shaking 意外移除。
- **快速迭代**：修改提示词无需重新分析复杂的依赖图。
- **可测试性**：可直接对字符串内容进行断言和快照测试。

## 风险、边界与改进建议

### 风险

1. **提示词与工具实现不同步**：`BASE_CHROME_PROMPT` 中提到的工具名（如 `mcp__claude-in-chrome__tabs_context_mcp`）必须与 `@ant/claude-for-chrome-mcp` 包实际导出的 `BROWSER_TOOLS` 严格一致。若包新增/重命名工具而提示词未更新，模型可能尝试调用不存在的工具。
2. **Tab ID 复用规则的执行依赖模型遵循**：提示词是软性约束，模型仍可能因上下文窗口限制或指令理解偏差而违反 tab ID 管理规则。
3. **对话框规避的局限性**：某些现代网页使用自定义 UI 模态（非原生 `alert`），提示词未明确区分，模型可能误判为安全操作。
4. **多语言支持缺失**：提示词为纯英文，若用户主要使用其他语言交互，英文提示词可能降低指令遵循度。

### 边界

- `getChromeSystemPrompt()` 返回的提示词**不包含** ToolSearch 指令，调用方必须自行决定是否追加。
- `BASE_CHROME_PROMPT` 长度约 2.3KB（英文），在 token 预算紧张的长会话中占用了固定的上下文开销。
- 技能提示（`SKILL_HINT`）仅在 `autoEnableClaudeInChrome` 为真时注入，手动 `--chrome` 模式下由 `setupClaudeInChrome()` 返回的完整 `systemPrompt` 覆盖。

### 改进建议

1. **工具名自动化校验**：在 CI 中增加测试，断言 `BASE_CHROME_PROMPT` 中出现的所有 `mcp__claude-in-chrome__*` 工具名都存在于 `@ant/claude-for-chrome-mcp` 的 `BROWSER_TOOLS` 列表中。
2. **提示词版本化**：为 `BASE_CHROME_PROMPT` 增加隐式版本标识（如注释中的 `v1.2`），便于在分析系统中追踪不同提示词版本的性能差异。
3. **自定义 UI 模态补充说明**：在 "Alerts and dialogs" 章节增加对自定义遮罩层（overlay/modal）的识别建议，减少误操作。
4. **国际化（i18n）探索**：评估将关键约束（如 tab ID 规则、对话框规避）以结构化 JSON 形式存储，根据用户语言动态渲染的可行性。
5. **A/B 实验支持**：将 `BASE_CHROME_PROMPT` 拆分为可组合的子模块（GIF、Console、Dialogs、Loops、Tabs），便于通过 feature flag 做局部提示词实验。
