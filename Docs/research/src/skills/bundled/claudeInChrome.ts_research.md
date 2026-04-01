# claudeInChrome.ts 研究文档

## 场景与职责

`claudeInChrome.ts` 实现了 `claude-in-chrome` 内置技能，用于在用户需要与 Chrome 浏览器交互时（点击元素、填写表单、截图、读取控制台日志、导航网站等）激活浏览器自动化能力。该技能是调用任何 `mcp__claude-in-chrome__*` 工具前的**强制入口**——模型必须先调用此技能，才能获得相关工具的使用权限与操作指南。

## 功能点目的

1. **技能门控（Skill Gate）**：防止模型在未获得上下文的情况下直接调用 Chrome MCP 工具。
2. **注入系统提示**：将 `BASE_CHROME_PROMPT`（来自 `src/utils/claudeInChrome/prompt.ts`）与技能激活消息拼接为系统提示。
3. **动态工具白名单**：根据 `@ant/claude-for-chrome-mcp` 包中导出的 `BROWSER_TOOLS` 列表，自动生成允许使用的工具名称前缀 `mcp__claude-in-chrome__${tool.name}`。
4. **自动启用判断**：通过 `shouldAutoEnableClaudeInChrome()` 决定该技能是否在启动时自动注册。

## 具体技术实现

### 关键流程

- `registerClaudeInChromeSkill()` → `registerBundledSkill({ name: 'claude-in-chrome', ... })`
- `getPromptForCommand(args)` 入口：
  1. `prompt = BASE_CHROME_PROMPT + SKILL_ACTIVATION_MESSAGE`
  2. 若用户提供了额外参数（如具体任务描述），追加 `## Task\n\n${args}`
  3. 返回 `[{ type: 'text', text: prompt }]`

### 数据结构

```ts
const CLAUDE_IN_CHROME_MCP_TOOLS = BROWSER_TOOLS.map(
  tool => `mcp__claude-in-chrome__${tool.name}`,
)

const SKILL_ACTIVATION_MESSAGE = `
Now that this skill is invoked, you have access to Chrome browser automation tools...
IMPORTANT: Start by calling mcp__claude-in-chrome__tabs_context_mcp...
`
```

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'claude-in-chrome'` |
| `allowedTools` | `CLAUDE_IN_CHROME_MCP_TOOLS`（动态从 MCP 包生成） |
| `userInvocable` | `true` |
| `isEnabled` | `() => shouldAutoEnableClaudeInChrome()` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/claudeInChrome.ts`
- 注册入口：`src/skills/bundled/index.ts` 中条件注册（`if (shouldAutoEnableClaudeInChrome()) { registerClaudeInChromeSkill() }`）
- 基础 Prompt：`src/utils/claudeInChrome/prompt.ts`（`BASE_CHROME_PROMPT`、`CHROME_TOOL_SEARCH_INSTRUCTIONS`）
- 启用判断逻辑：`src/utils/claudeInChrome/setup.ts`（`shouldAutoEnableClaudeInChrome`）
- MCP 工具定义：`@ant/claude-for-chrome-mcp` 包中的 `BROWSER_TOOLS`
- 核心注册器：`src/skills/bundledSkills.ts`

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `@ant/claude-for-chrome-mcp` | 提供 `BROWSER_TOOLS` 列表，决定允许哪些 MCP 工具 |
| `BASE_CHROME_PROMPT` | 提供浏览器自动化的通用指南（GIF 录制、控制台调试、弹窗警告、避免循环等） |
| `shouldAutoEnableClaudeInChrome` | 判断扩展是否已安装、是否为交互式会话、Feature Gate 是否开启 |
| `registerBundledSkill` | 注册技能到命令系统 |

- **无直接网络调用**：技能本身仅生成 prompt，实际的浏览器操作由 MCP 服务器通过 stdio 与 Chrome Native Messaging 完成。
- **Native Host 安装**：在 `setup.ts` 中异步安装 native host manifest 与 wrapper 脚本，属于该技能的底层基础设施。

## 风险、边界与改进建议

1. **边界：扩展未安装时技能不可见**：`isEnabled` 返回 false 时技能不会注册，用户无法通过 `/claude-in-chrome` 调用。若扩展被卸载后未刷新缓存，可能出现技能存在但工具连接失败的情况。
2. **风险：工具名称硬编码前缀**：`mcp__claude-in-chrome__` 前缀与 MCP 服务器的命名约定强耦合，若上游包更改工具命名规则，需同步修改此处。
3. **风险：ToolSearch 与直接调用的混淆**：`BASE_CHROME_PROMPT` 的变体 `CHROME_TOOL_SEARCH_INSTRUCTIONS` 在 `prompt.ts` 中定义，但 `claudeInChrome.ts` 使用的是不含 ToolSearch 指令的基础版本。实际运行时，ToolSearch 指令由请求组装层（`claude.ts`）根据全局配置动态注入，存在 prompt 来源分散的风险。
4. **改进建议**：
   - 在技能激活时检测 MCP 客户端连接状态，若未连接成功则提示用户检查 Chrome 扩展或重新运行 `/login`。
   - 将 `mcp__claude-in-chrome__` 前缀提取为共享常量，避免多处硬编码。
   - 考虑在技能描述中增加更明确的 "必须先调用此技能" 的警告，降低模型绕过技能门控的概率。
