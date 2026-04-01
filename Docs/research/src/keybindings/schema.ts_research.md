# schema.ts 研究文档

## 场景与职责

`src/keybindings/schema.ts` 是 Claude Code 快捷键配置的**权威契约定义**。它使用 Zod 库定义了 `keybindings.json` 文件的完整 JSON Schema，包括有效的上下文名称、动作标识符、键绑定格式等。该模块既是运行时校验器，也是类型生成器（通过 Zod 的 `z.infer`），确保用户配置与代码实现的一致性。

## 功能点目的

1. **定义有效上下文**：枚举所有可应用快捷键的 UI 上下文（如 `Global`、`Chat`、`Autocomplete` 等）。
2. **定义有效动作**：枚举所有可绑定的动作标识符（如 `app:interrupt`、`chat:submit` 等）。
3. **Zod Schema 定义**：提供 `KeybindingBlockSchema` 和 `KeybindingsSchema`，用于运行时校验和 JSON Schema 生成。
4. **人类可读描述**：为每个上下文提供描述文本，用于帮助文档和编辑器提示。

## 具体技术实现

### 关键数据结构

#### 上下文定义
```ts
export const KEYBINDING_CONTEXTS = [
  'Global', 'Chat', 'Autocomplete', 'Confirmation', 'Help',
  'Transcript', 'HistorySearch', 'Task', 'ThemePicker', 'Settings',
  'Tabs', 'Attachments', 'Footer', 'MessageSelector', 'DiffDialog',
  'ModelPicker', 'Select', 'Plugin',
] as const

export const KEYBINDING_CONTEXT_DESCRIPTIONS: Record<...> = {
  Global: 'Active everywhere, regardless of focus',
  Chat: 'When the chat input is focused',
  // ... 每个上下文的描述
}
```

#### 动作定义
```ts
export const KEYBINDING_ACTIONS = [
  // App-level
  'app:interrupt', 'app:exit', 'app:toggleTodos', // ...
  // Chat input
  'chat:cancel', 'chat:submit', 'chat:newline', // ...
  // Autocomplete
  'autocomplete:accept', 'autocomplete:dismiss', // ...
  // ... 共 70+ 个动作
  'voice:pushToTalk',
] as const
```

#### Zod Schema
```ts
export const KeybindingBlockSchema = lazySchema(() =>
  z.object({
    context: z.enum(KEYBINDING_CONTEXTS)
      .describe('UI context where these bindings apply...'),
    bindings: z.record(
      z.string().describe('Keystroke pattern...'),
      z.union([
        z.enum(KEYBINDING_ACTIONS),
        z.string().regex(/^command:[a-zA-Z0-9:\-_]+$/)
          .describe('Command binding...'),
        z.null().describe('Set to null to unbind...'),
      ])
    ).describe('Map of keystroke patterns to actions'),
  }).describe('A block of keybindings for a specific context')
)

export const KeybindingsSchema = lazySchema(() =>
  z.object({
    $schema: z.string().optional(),
    $docs: z.string().optional(),
    bindings: z.array(KeybindingBlockSchema()),
  }).describe('Claude Code keybindings configuration...')
)

// TypeScript 类型导出
export type KeybindingsSchemaType = z.infer<typeof KeybindingsSchema>
```

### 关键设计决策

#### 1. `lazySchema` 包装
- 使用 `../utils/lazySchema.js` 延迟 Zod Schema 的构造，避免模块加载时的立即执行开销。
- 对于大型枚举（70+ 动作、18+ 上下文），这能有效减少启动时间。

#### 2. 命令绑定支持
- 动作值可以是 `command:` 前缀的字符串（如 `"command:help"`），表示执行对应的 slash 命令。
- 正则校验：`/^command:[a-zA-Z0-9:\-_]+$/`，限制允许的字符集。

#### 3. `null` 表示解绑
- 显式设置 `"ctrl+s": null` 可取消默认绑定，这是用户自定义的核心机制。

#### 4. 元数据字段
- `$schema`：指向 SchemaStore 的 JSON Schema URL，供编辑器验证。
- `$docs`：指向官方文档 URL。

## 关键代码路径与文件引用

| 导出项 | 被调用方 | 用途 |
|--------|----------|------|
| `KEYBINDING_CONTEXTS` | `validate.ts`（校验上下文有效性）、`src/skills/bundled/keybindings.ts`（生成文档表） | 上下文枚举 |
| `KEYBINDING_CONTEXT_DESCRIPTIONS` | `src/skills/bundled/keybindings.ts` | 帮助文档 |
| `KEYBINDING_ACTIONS` | `validate.ts`（间接，通过 schema 校验）、`src/skills/bundled/keybindings.ts` | 动作枚举 |
| `KeybindingBlockSchema` / `KeybindingsSchema` | 外部工具（JSON Schema 生成）、潜在的未来校验入口 | 运行时校验 |
| `KeybindingsSchemaType` | `src/skills/bundled/keybindings.ts`（类型标注示例对象） | TypeScript 类型 |

## 依赖与外部交互

- `zod/v4`：Zod 库 v4 版本，用于 Schema 定义。
- `../utils/lazySchema.js`：延迟初始化工具。
- 无 React 依赖，可在任意上下文使用。

## 风险、边界与改进建议

### 风险与边界

1. **Schema 与代码的同步风险**
   - `KEYBINDING_ACTIONS` 和 `KEYBINDING_CONTEXTS` 是硬编码数组，若组件新增动作或上下文但忘记更新此处，会导致用户无法绑定新动作。
   - 当前缓解：代码审查 + `validate.ts` 的额外校验（重复检测）。

2. **命令绑定正则的局限性**
   - 正则 `/^command:[a-zA-Z0-9:\-_]+$/` 不允许空格、Unicode 或斜杠，这意味着 `"command:my command"` 或 `"command:help/verbose"` 会被拒绝。这是有意设计（限制复杂度），但可能限制未来扩展。

3. **Zod v4 的兼容性**
   - 明确导入 `zod/v4`，表明项目依赖 Zod v4 的特定特性。若未来升级 Zod 版本，需要验证兼容性。

4. **缺少运行时校验入口**
   - 当前代码中未发现直接调用 `KeybindingsSchema.parse()` 的位置，Schema 主要用于类型推断和外部 JSON Schema 生成。实际校验逻辑分散在 `validate.ts` 中，存在重复实现的风险。

### 改进建议

1. **统一校验入口**
   - 将 `validate.ts` 的校验逻辑迁移到 Zod Schema 的 `.refine()` 和 `.transform()` 中，实现单一数据源。例如：
     ```ts
     KeybindingBlockSchema.refine(block => {
       // 检查保留快捷键、重复键等
     })
     ```

2. **自动生成动作列表**
   - 考虑使用代码生成工具扫描 `useKeybinding` 调用，自动提取动作名称，避免人工维护 `KEYBINDING_ACTIONS` 的遗漏风险。

3. **命令绑定增强**
   - 若未来需要支持带参数的 slash 命令（如 `"command:git status"`），需要放宽正则或引入转义机制，但这会增加解析复杂度，需审慎评估。

4. **文档与 Schema 的自动化同步**
   - 当前 `src/skills/bundled/keybindings.ts` 手动维护帮助文档表格。可考虑从 Schema 自动生成 Markdown 表格，减少文档与代码的不一致。

5. **补充 `types.ts` 引用**
   - Schema 中定义的上下文和动作类型应与 `types.ts` 中的 `KeybindingContextName` 保持同步（若该类型存在）。建议建立类型级别的关联，确保重构时的一致性。
