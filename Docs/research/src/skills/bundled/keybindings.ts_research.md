# keybindings.ts 研究文档

## 场景与职责

`keybindings.ts` 实现了 `keybindings-help` 内置技能，用于在用户询问键盘快捷键自定义时，生成一份完整的帮助文档。该文档包含键位语法说明、文件格式示例、保留快捷键列表、可用上下文与动作查询表，以及 `/doctor` 验证相关的常见问题与修复方案。

## 功能点目的

1. **动态生成参考表格**：从源码中的 schema 和 default bindings 实时生成 Markdown 表格，确保文档与代码实现始终一致。
2. **教育用户正确编辑**：指导用户如何创建/修改 `~/.claude/keybindings.json`，包括重新绑定、解绑、和弦（chord）等高级用法。
3. **集成验证信息**：说明 `/doctor` 命令如何诊断键位配置错误，并提供常见问题的修复对照表。
4. **最小权限原则**：该技能仅授予 `Read` 工具权限，因为它只需要读取现有配置文件。

## 具体技术实现

### 关键流程

- `registerKeybindingsSkill()` → `registerBundledSkill({ name: 'keybindings-help', ... })`
- `getPromptForCommand(args)` 入口：
  1. `generateContextsTable()`：基于 `KEYBINDING_CONTEXTS` 和 `KEYBINDING_CONTEXT_DESCRIPTIONS` 生成上下文表格
  2. `generateActionsTable()`：基于 `DEFAULT_BINDINGS` 和 `KEYBINDING_ACTIONS` 生成动作表格
  3. `generateReservedShortcuts()`：基于 `NON_REBINDABLE`、`TERMINAL_RESERVED`、`MACOS_RESERVED` 生成保留键位列表
  4. 将多个预定义章节与动态表格拼接为完整 prompt

### 数据结构

- `FILE_FORMAT_EXAMPLE`：`KeybindingsSchemaType` 的 JSON 示例
- `UNBIND_EXAMPLE`、`REBIND_EXAMPLE`、`CHORD_EXAMPLE`：三种常见修改模式的示例对象
- `SECTION_INTRO`、`SECTION_FILE_FORMAT`、`SECTION_KEYSTROKE_SYNTAX` 等：预定义的 Markdown 章节字符串数组

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'keybindings-help'` |
| `allowedTools` | `['Read']` |
| `userInvocable` | `false`（模型自动触发，不显示为显式命令） |
| `isEnabled` | `isKeybindingCustomizationEnabled`（GrowthBook 门控） |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/keybindings.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- 默认绑定定义：`src/keybindings/defaultBindings.ts`
- 保留快捷键定义：`src/keybindings/reservedShortcuts.ts`
- Schema 定义：`src/keybindings/schema.ts`（`KEYBINDING_CONTEXTS`、`KEYBINDING_ACTIONS`、`KeybindingsSchemaType`）
- 用户绑定加载器：`src/keybindings/loadUserBindings.ts`（`isKeybindingCustomizationEnabled`）
- JSON 序列化工具：`src/utils/slowOperations.ts`（`jsonStringify`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `DEFAULT_BINDINGS` | 提供默认键位映射，用于生成动作表格 |
| `KEYBINDING_CONTEXTS` / `KEYBINDING_CONTEXT_DESCRIPTIONS` | 提供上下文名称与说明 |
| `KEYBINDING_ACTIONS` | 提供所有合法动作标识符 |
| `NON_REBINDABLE` / `TERMINAL_RESERVED` / `MACOS_RESERVED` | 提供保留快捷键数据 |
| `isKeybindingCustomizationEnabled` | GrowthBook 门控，决定技能是否对外可见 |
| `jsonStringify` | 将示例对象格式化为 JSON 代码块 |

- **无网络调用**：纯本地 schema 与配置数据读取。
- **用户配置路径**：提示中硬编码 `~/.claude/keybindings.json`，与 `loadUserBindings.ts` 中的 `getKeybindingsPath()` 一致。

## 风险、边界与改进建议

1. **边界：动作表格的上下文推断**：`generateActionsTable` 中，若某个 `KEYBINDING_ACTIONS` 中的动作未在 `DEFAULT_BINDINGS` 出现，则通过 `inferContextFromAction(action)` 按前缀推断上下文。这种推断可能不准确（如自定义动作或新增动作前缀未在映射表中更新）。
2. **风险：硬编码的 `/doctor` 输出示例**：`SECTION_DOCTOR` 中的示例输出是写死的字符串，若 `/doctor` 的实际输出格式发生变化，帮助文档中的示例将过时。
3. **风险：键位自定义功能仅限 ant**：`isKeybindingCustomizationEnabled()` 当前由 GrowthBook 门控控制，外部用户默认不可见。但帮助文档中未明确说明该功能对普通用户不可用，可能导致困惑。
4. **改进建议**：
   - 在 `inferContextFromAction` 的 `prefixToContext` 映射中增加自动化测试，确保所有 `KEYBINDING_ACTIONS` 前缀都有对应条目。
   - 将 `/doctor` 的示例输出改为引用真实验证函数生成的消息模板，减少维护负担。
   - 在 prompt 中增加一段说明，告知用户键位自定义功能当前的可用范围（如 "此功能目前仅对特定用户群体开放"）。
