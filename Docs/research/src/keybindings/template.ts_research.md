# template.ts 研究文档

## 场景与职责

`src/keybindings/template.ts` 负责生成**用户配置文件模板**（`~/.claude/keybindings.json`）。当用户首次运行 `/keybindings` 命令时，系统会创建包含所有默认绑定的模板文件，用户可在此基础上进行自定义。该模块确保模板中**不包含不可重新绑定的快捷键**（如 `ctrl+c`），避免用户配置后立即触发校验错误。

## 功能点目的

1. **生成标准格式的 keybindings.json**：包含 `$schema` 和 `$docs` 元数据字段，支持编辑器验证。
2. **过滤保留快捷键**：从模板中移除 `NON_REBINDABLE` 列表中的键，避免用户误配置。
3. **提供可读的默认配置**：用户可直接在模板基础上修改，无需从零编写。

## 具体技术实现

### 关键函数

```ts
export function generateKeybindingsTemplate(): string
```

#### 执行流程
1. 调用 `filterReservedShortcuts(DEFAULT_BINDINGS)` 过滤保留键。
2. 构造配置对象：
   ```ts
   {
     $schema: 'https://www.schemastore.org/claude-code-keybindings.json',
     $docs: 'https://code.claude.com/docs/en/keybindings',
     bindings: filteredBindings,
   }
   ```
3. 使用 `jsonStringify`（来自 `../utils/slowOperations.js`）格式化输出，缩进 2 空格，末尾加换行。

### 过滤逻辑

```ts
function filterReservedShortcuts(blocks: KeybindingBlock[]): KeybindingBlock[]
```

- 从 `NON_REBINDABLE` 提取保留键集合，使用 `normalizeKeyForComparison` 统一键名格式。
- 遍历每个 block 的 `bindings` 对象，排除保留键。
- 过滤后若 block 为空（无剩余绑定），则丢弃该 block。
- 返回保留非空 block 的新数组。

### 关键设计决策

1. **仅过滤 `NON_REBINDABLE`**
   - 不过滤 `TERMINAL_RESERVED` 或 `MACOS_RESERVED`，因为这些是警告级别而非错误级别，用户可能确实希望在特定终端配置下使用它们。

2. **保持原始键名格式**
   - 比较时使用规范化形式，但输出保留原始键名（如 `"ctrl+shift+o"`）。

## 关键代码路径与文件引用

| 函数 | 被调用方 | 用途 |
|------|----------|------|
| `generateKeybindingsTemplate` | `src/commands/keybindings/keybindings.ts` | `/keybindings` 命令创建/打开配置文件 |
| `filterReservedShortcuts` | 内部使用 | 模板生成时过滤保留键 |

## 依赖与外部交互

- `../utils/slowOperations.js`：`jsonStringify` 用于格式化 JSON 输出。
- `./defaultBindings.js`：`DEFAULT_BINDINGS` 作为模板数据源。
- `./reservedShortcuts.js`：`NON_REBINDABLE` 和 `normalizeKeyForComparison` 用于过滤。
- `./types.js`：`KeybindingBlock` 类型。

## 风险、边界与改进建议

### 风险与边界

1. **Schema URL 的可用性**
   - `$schema` 指向 `schemastore.org`，依赖外部服务。若该 URL 失效或用户处于离线状态，编辑器无法提供自动补全和验证。

2. **模板文件大小**
   - 默认绑定包含 70+ 动作，生成的 JSON 约 5-8KB。对于只想修改一两个绑定的用户，这可能显得冗长。

3. **过滤逻辑的重复**
   - `filterReservedShortcuts` 与 `validate.ts` 的 `checkReservedShortcuts` 都操作保留键列表，但目的不同。若 `NON_REBINDABLE` 更新，两处行为需要保持一致。

4. **无动态重新生成**
   - 模板只在文件创建时生成一次。若后续版本新增默认绑定或修改键位，现有用户的配置文件不会自动更新。

### 改进建议

1. **注释增强**
   - 在生成的 JSON 中插入注释（如 `"//": "Use null to unbind a default shortcut"`），帮助用户理解配置语法。

2. **版本标记**
   - 在模板中增加 `"// generated_for_version": "1.x.x"` 注释，便于未来识别旧版配置并提供迁移提示。

3. **Schema 内嵌**
   - 考虑将 schema 内容内嵌到 CLI 中，通过本地路径引用，消除对外部网络的依赖。

4. **补充 `types.ts` 引用**
   - 当前直接依赖 `./types.js` 的 `KeybindingBlock`，若该类型定义变更，模板生成逻辑需要同步更新。
