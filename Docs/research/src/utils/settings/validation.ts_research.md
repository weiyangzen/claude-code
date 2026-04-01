# Research Document: src/utils/settings/validation.ts

## 场景与职责

`src/utils/settings/validation.ts` 是 Claude Code 设置系统的**校验引擎层**，负责将 Zod 解析错误转化为用户/AI 可理解的信息，并在设置加载 pipeline 中提供防御性过滤。核心职责包括：

1. **格式化 Zod v4 校验错误**：将机器友好的 `ZodError` 转换为人可读的 `ValidationError[]`，包含字段路径、错误消息、期望值、无效值、修复建议和文档链接。
2. **校验设置文件内容**：提供 `validateSettingsFileContent` 函数，供 `FileEditTool` 和设置加载流程使用，判断一段 JSON 字符串是否完全合规。
3. **过滤非法权限规则**：在 schema 校验之前，先对 `permissions.allow/deny/ask` 数组做前置过滤，防止一条坏规则导致整个设置文件被丢弃（poisoning）。

## 功能点目的

### 1. `formatZodError` — ZodError → ValidationError[]
这是文件最复杂的函数，约 75 行。它针对 Zod v4 的不同 issue code 做精细化处理：

- **`invalid_type`**：提取 `expected` 类型和实际接收到的类型；特殊处理根路径 `path === ''` 且 `received === 'null'` 的情况，将消息优化为 `"Invalid or malformed JSON"`（帮助用户识别 JSON 语法错误）。
- **`invalid_value`**（枚举值错误）：收集所有合法枚举值，生成 `"Expected one of: ..."` 消息。
- **`unrecognized_keys`**：将未知键列表格式化为 `"Unrecognized field(s): ..."`，并使用 `plural()` 处理单复数。
- **`too_small`**：针对数值下限错误生成 `"Number must be greater than or equal to ..."`。
- **`custom`**：提取 `params.received` 作为无效值展示。

所有格式化后的错误都会调用 `getValidationTip()`（来自 `validationTips.ts`）获取修复建议和文档链接。

### 2. `validateSettingsFileContent` — 字符串级严格校验
```ts
export function validateSettingsFileContent(content: string):
  | { isValid: true }
  | { isValid: false; error: string; fullSchema: string }
```

流程：
1. `jsonParse(content)` 尝试解析 JSON。
2. `SettingsSchema().strict().safeParse(jsonData)` 做严格模式校验。
3. 成功 → `{ isValid: true }`。
4. 失败 → 调用 `formatZodError` 生成错误列表，拼接成字符串，并附上 `generateSettingsJSONSchema()` 返回的完整 JSON Schema。

**关键差异点**：
- `validateEditTool.ts` 调用此函数时使用 `strict()`，因此未知键会被拒绝。
- `settings.ts` 的日常加载使用 `safeParse`（非 strict），未知键通过 `.passthrough()` 保留。

### 3. `filterInvalidPermissionRules` — 权限规则前置过滤
```ts
export function filterInvalidPermissionRules(data: unknown, filePath: string): ValidationError[]
```

设计 rationale：
- 权限规则 `permissions.allow/deny/ask` 是字符串数组，用户很容易写错格式（如缺少括号、工具名小写、非法 MCP 格式）。
- 如果不做前置过滤，一条坏规则会导致整个 `SettingsSchema().safeParse(data)` 失败，进而使整个设置文件被当作无效丢弃。
- 该函数在 schema 校验**之前**运行，遍历三个数组，移除：
  - 非字符串元素；
  - 调用 `validatePermissionRule(rule)` 返回 `valid: false` 的规则。
- 对每一条被移除的规则生成 `ValidationError` 警告（warning），这些警告最终会与 schema 错误一起展示给用户。

## 具体技术实现（关键流程/数据结构/协议/命令）

### Zod v4 issue 类型守卫
文件顶部定义了 4 个类型守卫函数，用于在运行时安全地访问 Zod v4 issue 的扩展字段：

```ts
function isInvalidTypeIssue(issue: ZodIssue): issue is ZodIssue & { code: 'invalid_type'; expected: string; input: unknown }
function isInvalidValueIssue(issue: ZodIssue): issue is ZodIssue & { code: 'invalid_value'; values: unknown[]; input: unknown }
function isUnrecognizedKeysIssue(issue: ZodIssue): issue is ZodIssue & { code: 'unrecognized_keys'; keys: string[] }
function isTooSmallIssue(issue: ZodIssue): issue is ZodIssue & { code: 'too_small'; minimum: number | bigint; origin: string }
```

这些守卫确保 `formatZodError` 在访问 `issue.expected`、`issue.values`、`issue.keys`、`issue.minimum` 时不会触发 TypeScript 编译错误，同时也起到文档化作用。

### `getReceivedType` 辅助函数
```ts
function getReceivedType(value: unknown): string {
  if (value === null) return 'null'
  if (value === undefined) return 'undefined'
  if (Array.isArray(value)) return 'array'
  return typeof value
}
```

用于将 JavaScript 运行时类型转化为用户友好的字符串（如 `typeof null === 'object'` 的问题被修正为 `'null'`）。

### `extractReceivedFromMessage`
```ts
function extractReceivedFromMessage(msg: string): string | undefined {
  const match = msg.match(/received (\w+)/)
  return match ? match[1] : undefined
}
```

Zod v4 的 `invalid_type` 错误消息通常包含 `"received <type>"` 片段，该函数尝试从中提取类型字符串，作为 `receivedValue` 的首选来源；若提取失败则回退到 `getReceivedType(issue.input)`。

### `filterInvalidPermissionRules` 的遍历逻辑
```ts
for (const key of ['allow', 'deny', 'ask']) {
  const rules = perms[key]
  if (!Array.isArray(rules)) continue

  perms[key] = rules.filter(rule => {
    if (typeof rule !== 'string') { /* push warning; return false */ }
    const result = validatePermissionRule(rule)
    if (!result.valid) { /* push warning with suggestion/examples; return false */ }
    return true
  })
}
```

注意：该函数直接**修改传入的 `data` 对象**（通过 `obj.permissions = ...` 的数组引用替换），这是有意为之，因为调用方（`settings.ts`、`mdm/settings.ts`）需要过滤后的干净数据继续传给 `SettingsSchema().safeParse()`。

## 关键代码路径与文件引用

### 上游调用方
| 文件 | 调用符号 | 场景 |
|------|----------|------|
| `src/utils/settings/settings.ts` | `formatZodError`, `filterInvalidPermissionRules` | `parseSettingsFile` / `loadSettingsFromDisk` 中解析文件并收集错误 |
| `src/utils/settings/validateEditTool.ts` | `validateSettingsFileContent` | FileEditTool 编辑 settings 文件前的校验 |
| `src/utils/settings/mdm/settings.ts` | `formatZodError`, `filterInvalidPermissionRules` | MDM / HKCU / 注册表设置解析 |
| `src/utils/plugins/validatePlugin.ts` | `validateSettingsFileContent` | 插件配置校验 |

### 直接依赖
| 文件 | 导入符号 | 说明 |
|------|----------|------|
| `zod/v4` | `ZodError`, `ZodIssue` | Zod v4 错误类型 |
| `src/services/mcp/types.js` | `ConfigScope` | MCP 错误元数据中的 scope 类型 |
| `src/utils/slowOperations.js` | `jsonParse` | JSON 解析（带性能/遥测包装） |
| `src/utils/stringUtils.js` | `plural` | 单复数格式化 |
| `src/utils/settings/permissionValidation.js` | `validatePermissionRule` | 单条权限规则语法校验 |
| `src/utils/settings/schemaOutput.js` | `generateSettingsJSONSchema` | 生成完整 JSON Schema 字符串 |
| `src/utils/settings/types.js` | `SettingsJson`, `SettingsSchema` | 设置类型与 Schema |
| `src/utils/settings/validationTips.js` | `getValidationTip` | 获取错误修复建议与文档链接 |

### 下游消费方
- `src/components/InvalidSettingsDialog.tsx`：展示设置校验错误。
- `src/components/ValidationErrorsList.tsx`：通用校验错误列表组件。
- `src/components/mcp/McpParsingWarnings.tsx`：MCP 相关解析警告。
- `src/hooks/notifs/useSettingsErrors.tsx`：将设置错误转为通知。
- `src/dialogLaunchers.tsx`：可能触发无效设置对话框。
- `src/main.tsx`：启动时可能消费设置错误。

## 依赖与外部交互

### Zod v4 的 `.strict()` vs `.passthrough()`
- `SettingsSchema` 本身定义时使用了 `.passthrough()`，因此日常 `safeParse` 允许未知键。
- `validateSettingsFileContent` 显式调用 `.strict()`，覆盖 `.passthrough()` 行为，这是为了对 AI 编辑行为施加更严格的约束。

### `ValidationError` 数据结构
```ts
export type ValidationError = {
  file?: string      // 相对/绝对文件路径
  path: FieldPath    // 点分路径，如 "permissions.defaultMode"
  message: string    // 人可读错误
  expected?: string  // 期望值
  invalidValue?: unknown // 实际传入的非法值
  suggestion?: string // 修复建议
  docLink?: string   // 文档链接
  mcpErrorMetadata?: { scope: ConfigScope; serverName?: string; severity?: 'fatal' | 'warning' }
}
```

`mcpErrorMetadata` 字段是 MCP 配置错误的专用扩展，虽然在本文件中尚未被填充，但类型已预留，供 MCP 配置解析层使用。

### `SettingsWithErrors`
```ts
export type SettingsWithErrors = {
  settings: SettingsJson
  errors: ValidationError[]
}
```

这是 `settings.ts` 中 `getSettingsWithErrors()`、`parseSettingsFile()`、`loadSettingsFromDisk()` 等函数的通用返回结构。

## 风险、边界与改进建议

### 风险

1. **`formatZodError` 对 Zod v4 的强耦合**
   - 代码中硬编码了 `invalid_type`、`invalid_value`、`unrecognized_keys`、`too_small`、`custom` 等 issue code。若未来升级 Zod v5 或 issue 结构变化，该函数需要大面积重写。
   - 当前缺少对 `too_big`、`invalid_literal`、`invalid_union` 等常见 issue code 的专门处理，这些错误会回退到 Zod 原始 `issue.message`，用户体验可能不佳。

2. **`validateSettingsFileContent` 的 `fullSchema` 体积**
   - `generateSettingsJSONSchema()` 输出完整的 settings JSON Schema，可能非常大。在 `validateEditTool.ts` 场景中，这个字符串被直接嵌入 AI 的错误提示中，存在上下文膨胀风险。

3. **`filterInvalidPermissionRules` 的副作用**
   - 函数直接修改输入对象。虽然当前调用方都依赖这一行为，但如果未来有调用方需要保留原始数据做对比分析，这种隐式副作用会造成 bug。

4. **错误去重依赖调用方**
   - `formatZodError` 本身不做去重。`settings.ts` 的 `loadSettingsFromDisk` 使用 `seenErrors` Set（`${file}:${path}:${message}`）做去重，但如果不同文件有相同路径和消息，会被误去重。

### 边界

- **不做文件 I/O**：只处理内存中的字符串/对象。
- **不做权限决策**：`validatePermissionRule` 只检查语法格式，不判断规则是否应该被允许。
- **不处理 UI 渲染**：错误展示完全交给 React 组件层。
- **不处理设置合并**：多源设置合并逻辑在 `settings.ts`。

### 改进建议

1. **补齐未覆盖的 Zod issue code 处理**
   - 增加对 `too_big`（数值上限）、`invalid_date`、`invalid_string`（如 email/URL/url 校验失败）、`invalid_arguments` 等 code 的分支处理，使错误消息更统一、更友好。

2. **`filterInvalidPermissionRules` 改为纯函数**
   - 返回 `{ filteredData, warnings }` 而不是直接修改 `data`，降低副作用风险，同时让调用方更清晰地感知数据变换。

3. **为 `validateSettingsFileContent` 增加轻量模式**
   - 可新增一个 `validateSettingsFileContentLite(content)`，不返回 `fullSchema`，仅返回错误摘要，供 AI 场景使用，减少 token 消耗。

4. **增强 `mcpErrorMetadata` 的填充逻辑**
   - 当前 `ValidationError` 中 `mcpErrorMetadata` 始终为空。可在 `formatZodError` 或 MCP 配置解析层中，根据路径前缀（如 `pluginConfigs.*.mcpServers.*`）自动注入 `scope` 和 `serverName`，提升 MCP 配置错误的可定位性。

5. **引入 issue code 映射表**
   - 将 `formatZodError` 中的分支逻辑抽离为策略表/插件化结构，便于未来扩展新 issue code 或支持国际化（i18n）。

6. **测试覆盖**
   - 本次调研未在仓库中找到专门针对 `validation.ts` 的单元测试。建议补充：
     - 各类 Zod issue code 的格式化输出测试；
     - `filterInvalidPermissionRules` 对混合合法/非法规则数组的处理；
     - `validateSettingsFileContent` 对合法 JSON、非法 JSON、strict 未知键的判定。
