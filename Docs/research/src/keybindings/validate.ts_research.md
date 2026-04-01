# validate.ts 研究文档

## 场景与职责

`src/keybindings/validate.ts` 是 Claude Code 快捷键系统的**配置校验中心**。它负责验证用户自定义的 `keybindings.json` 文件，检测并报告各类配置问题，包括语法错误、重复绑定、保留快捷键冲突、无效上下文/动作等。该校验是用户自定义快捷键的安全网，确保错误配置不会导致应用行为异常。

## 功能点目的

1. **结构校验**：验证 JSON 结构是否符合预期（数组包裹、对象字段等）。
2. **语法校验**：检测键名解析错误（如空键部分、无法识别的键）。
3. **重复检测**：发现同一上下文内的重复键定义（JSON 解析会静默使用后值，可能导致用户困惑）。
4. **保留键冲突**：检查用户绑定是否与系统/终端保留快捷键冲突。
5. **动作校验**：验证动作格式（字符串或 null）、命令绑定格式（`command:` 前缀）。
6. **上下文校验**：确保使用的上下文名称在有效列表中。
7. **特殊动作校验**：如 `voice:pushToTalk` 的绑定合理性（避免使用裸字母键）。

## 具体技术实现

### 关键数据结构

```ts
export type KeybindingWarningType =
  | 'parse_error'
  | 'duplicate'
  | 'reserved'
  | 'invalid_context'
  | 'invalid_action'

export type KeybindingWarning = {
  type: KeybindingWarningType
  severity: 'error' | 'warning'
  message: string
  key?: string
  context?: string
  action?: string
  suggestion?: string
}
```

### 关键函数

#### `validateUserConfig(userBlocks: unknown): KeybindingWarning[]`
- 顶层校验入口，验证用户配置是否为数组。
- 遍历每个 block 调用 `validateBlock`。

#### `validateBlock(block, index): KeybindingWarning[]`
- **上下文校验**：检查 `context` 字段存在且值在 `VALID_CONTEXTS` 列表中。
- **绑定对象校验**：确保 `bindings` 是对象。
- **键值对遍历**：
  - 调用 `validateKeystroke` 检查键名语法。
  - 检查动作类型（字符串或 null）。
  - 命令绑定格式校验：`/^command:[a-zA-Z0-9:\-_]+$/`。
  - 命令绑定上下文限制：必须在 `Chat` 上下文。
  - `voice:pushToTalk` 特殊校验：若使用裸字母键（无修饰符），发出警告（避免输入时意外触发）。

#### `checkDuplicateKeysInJson(jsonString): KeybindingWarning[]`
- **关键洞察**：`JSON.parse` 会静默丢弃重复键（保留最后一个），用户可能 unaware。
- 使用正则 `/"bindings"\s*:\s*\{...\}/g` 提取 bindings 块内容。
- 在每个块内使用 `/"([^"]+)"\s*:/g` 匹配所有键，统计出现次数，第二次出现时报告警告。

#### `checkDuplicates(blocks): KeybindingWarning[]`
- 在解析后的数据结构层面检测重复（处理不同写法但等价的键，如 `"ctrl+k"` 与 `"Ctrl+K"`）。
- 使用 `normalizeKeyForComparison`（来自 `reservedShortcuts.ts`）统一键名格式。
- 按上下文分组，使用 `Map<string, Map<string, string>>` 跟踪已见键。

#### `checkReservedShortcuts(bindings): KeybindingWarning[]`
- 调用 `getReservedShortcuts()` 获取平台相关的保留键列表。
- 使用 `chordToString` 和 `normalizeKeyForComparison` 比较用户绑定与保留键。
- 保留键的 `severity` 决定警告级别。

#### `validateBindings(userBlocks, _parsedBindings): KeybindingWarning[]`
- 组合校验入口，调用上述所有校验函数。
- 去重：使用 `` `${w.type}:${w.key}:${w.context}` `` 作为键，确保同一问题不重复报告。

### 格式化输出

#### `formatWarning(warning): string`
- 根据 `severity` 选择图标：`✗`（error）或 `⚠`（warning）。
- 包含 `suggestion` 时追加提示。

#### `formatWarnings(warnings): string`
- 分类统计 errors 和 warnings。
- 生成人类可读的汇总文本，供 `/doctor` 命令使用。

## 关键代码路径与文件引用

| 函数 | 被调用方 | 用途 |
|------|----------|------|
| `validateBindings` | `loadUserBindings.ts` (加载时校验) | 用户配置加载时的完整校验 |
| `checkDuplicateKeysInJson` | `loadUserBindings.ts` | JSON 原始文本层面的重复检测 |
| `formatWarnings` | `src/commands/Doctor.tsx` (推测) | 格式化输出给用户 |
| `KeybindingWarning` 类型 | `loadUserBindings.ts` (KeybindingsLoadResult) | 类型契约 |

## 依赖与外部交互

- `../utils/stringUtils.js`：`plural`（单复数格式化）。
- `./parser.js`：`chordToString`, `parseChord`, `parseKeystroke` —— 键名解析和字符串化。
- `./reservedShortcuts.js`：`getReservedShortcuts`, `normalizeKeyForComparison` —— 保留键检查和键名规范化。
- `./types.js`：`KeybindingBlock`, `KeybindingContextName`, `ParsedBinding` —— 类型依赖。

## 风险、边界与改进建议

### 风险与边界

1. **正则解析的脆弱性**
   - `checkDuplicateKeysInJson` 使用正则解析 JSON，虽能处理常见格式，但面对极端情况（如字符串内包含 `"bindings"` 字样、嵌套对象深度过大）可能误报或漏报。
   - 缓解：JSON 格式通常由编辑器/用户手动维护，极端情况罕见；且误报为 warning 级别，不会阻断使用。

2. **VALID_CONTEXTS 的同步风险**
   - 文件中硬编码了 `VALID_CONTEXTS` 数组，与 `schema.ts` 的 `KEYBINDING_CONTEXTS` 需保持一致。若新增上下文但忘记更新此处，校验会错误地拒绝有效配置。
   - 缓解：两处数组内容相同，建议未来抽取到共享常量。

3. **命令绑定上下文限制**
   - 当前强制要求 `command:` 绑定必须在 `Chat` 上下文，这是产品决策（命令通常在聊天输入执行），但限制了灵活性（如想在 Global 上下文绑定 `/help` 命令）。

4. **voice:pushToTalk 的启发式校验**
   - 检查裸字母键（`/^[a-z]$/`）时未考虑大写（`A-Z`），虽然 `parseKeystroke` 会将键转为小写，但若用户配置 `"Shift+a"` 仍可能漏检。

5. **性能考虑**
   - 对大型配置文件（数百条绑定），多次遍历（`validateBlock` → `checkDuplicates` → `checkReservedShortcuts`）可能成为瓶颈。但正常用户配置规模下（<50 条），性能影响可忽略。

### 改进建议

1. **统一校验入口**
   - 将校验逻辑迁移到 Zod Schema（`schema.ts`）的 `.refine()` 中，实现单一数据源。例如：
     ```ts
     KeybindingBlockSchema.refine(block => {
       // 保留键检查
       // 重复键检查
     })
     ```

2. **增强 JSON 解析鲁棒性**
   - 考虑使用流式 JSON 解析器或 AST 遍历（如 `json-to-ast`）替代正则，更可靠地检测重复键。

3. **上下文限制可配置**
   - 若产品决策允许，可考虑放宽 `command:` 绑定的上下文限制，或提供白名单机制。

4. **校验缓存**
   - 对未变更的文件，可缓存校验结果（基于 mtime 或内容哈希），避免重复计算。

5. **补充单元测试**
   - 当前仓库快照中未发现测试文件。validate.ts 是纯函数模块，非常适合单元测试，建议补充覆盖：
     - 各种键名解析错误
     - 重复键检测（JSON 层面和数据层面）
     - 保留键匹配（不同平台）
     - 命令绑定格式校验

6. **与 `types.ts` 的关联**
   - 当前依赖 `./types.js` 的多个类型，若该类型定义变更（如 `KeybindingWarning` 新增字段），需要同步更新。建议建立类型级别的关联。

## 文件大小与复杂度

- 代码行数：~498 行（本批次最大文件之一）
- 功能密度高，包含 10+ 个校验函数
- 是 keybindings 系统的"守门员"，确保配置质量

## 与相关模块的协作关系

```
validate.ts
    ├── 使用 parser.ts (键名解析)
    ├── 使用 reservedShortcuts.ts (保留键检查、键名规范化)
    ├── 被 loadUserBindings.ts 调用 (加载时校验)
    ├── 输出到 Doctor 命令 (用户可见的警告)
    └── 依赖 types.ts (类型定义)
```

该模块在整个 keybindings 系统中处于"质量控制层"，是保障用户自定义配置安全性的关键防线。
