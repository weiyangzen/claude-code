# Research Document: src/utils/settings/validationTips.ts

## 场景与职责

`src/utils/settings/validationTips.ts` 是 Claude Code 设置校验错误提示系统的**知识库层**，职责单一而明确：

1. **将机器生成的 Zod 校验错误映射为用户可操作的修复建议**：当用户的 `settings.json` 或 managed settings 文件包含非法值时，不仅告诉他们"错了"，还要告诉他们"怎么改"。
2. **提供相关文档链接**：将错误与官方文档的对应章节关联，降低用户排查成本。
3. **作为 `validation.ts` 中 `formatZodError` 的可插拔补充**：通过 `getValidationTip` 函数解耦错误格式化与提示内容，使提示规则可以独立演进。

## 功能点目的

### 1. `ValidationTip` 与 `TipContext` 类型
```ts
export type ValidationTip = {
  suggestion?: string   // 人可读的修复建议
  docLink?: string      // 官方文档链接
}

export type TipContext = {
  path: string          // 点分字段路径，如 "permissions.defaultMode"
  code: ZodIssueCodeType | string  // Zod issue code，如 "invalid_value"
  expected?: string     // 期望值
  received?: unknown    // 实际接收值
  enumValues?: string[] // 枚举合法值列表
  message?: string      // Zod 原始错误消息
  value?: unknown       // 原始值别名
}
```

这两个类型定义了本模块的接口契约：`validation.ts` 在格式化每个 Zod issue 时，构造一个 `TipContext` 并调用 `getValidationTip`，获取可选的 `suggestion` 和 `docLink`。

### 2. `TIP_MATCHERS` — 规则驱动的提示匹配表
核心数据结构是一个 `TipMatcher[]` 数组，每个元素包含：
- `matches(context: TipContext): boolean`：判断当前错误是否命中该规则；
- `tip: ValidationTip`：命中的提示内容。

当前数组包含 11 条匹配规则（按优先级顺序排列），覆盖最常见的 settings 配置错误：

| # | 匹配条件 | 建议内容 | 文档链接 |
|---|----------|----------|----------|
| 1 | `permissions.defaultMode` + `invalid_value` | 解释四种模式含义（acceptEdits / plan / bypassPermissions / default） | `.../iam#permission-modes` |
| 2 | `apiKeyHelper` + `invalid_type` | 提供脚本路径示例 | 无 |
| 3 | `cleanupPeriodDays` + `too_small` + expected `0` | 解释取值范围及 0 的特殊含义（禁用持久化） | 无 |
| 4 | `env.*` + `invalid_type` | 强调 env 值必须是字符串，给出加引号示例 | `.../settings#environment-variables` |
| 5 | `permissions.allow/deny` + `invalid_type` + expected `array` | 说明权限规则必须是数组，给出 `Bash(...)`、`Edit(...)` 示例 | 无 |
| 6 | `path.includes('hooks')` + `invalid_type` | 纠正 hooks 的 matcher 是字符串而非对象，给出正确 JSON 示例 | 无 |
| 7 | `invalid_type` + expected `boolean` | 提醒 boolean 不要加引号 | 无 |
| 8 | `unrecognized_keys` | 提示检查拼写或查阅文档 | `.../settings` |
| 9 | `invalid_value` + `enumValues !== undefined` | suggestion 留空，由 `getValidationTip` 后续动态生成枚举值列表 | 无 |
| 10 | `invalid_type` + expected `object` + received `null` + path `''` | 提示 JSON 语法错误（缺逗号、括号不匹配等） | 无 |
| 11 | `permissions.additionalDirectories` + `invalid_type` | 说明必须是目录路径数组，给出示例 | `.../iam#working-directories` |

### 3. `getValidationTip` — 动态提示生成
```ts
export function getValidationTip(context: TipContext): ValidationTip | null
```

执行流程：
1. 按顺序遍历 `TIP_MATCHERS`，找到第一个 `matches(context)` 为 `true` 的匹配项。
2. 复制匹配项的 `tip` 对象。
3. **动态枚举值补充**：如果 issue code 是 `invalid_value` 且 `enumValues` 存在，但匹配项本身没有预定义 `suggestion`，则自动生成 `"Valid values: \"<val1>\", \"<val2>\"..."`。
4. **路径前缀文档链接回退**：如果 `tip.docLink` 仍为空，则根据 `context.path` 的第一级前缀（如 `permissions`、`env`、`hooks`）查找 `PATH_DOC_LINKS` 映射表，自动补充对应文档链接。
5. 返回最终 `ValidationTip`；若无匹配则返回 `null`。

### 4. `PATH_DOC_LINKS` — 路径前缀文档映射
```ts
const PATH_DOC_LINKS: Record<string, string> = {
  permissions: `${DOCUMENTATION_BASE}/iam#configuring-permissions`,
  env: `${DOCUMENTATION_BASE}/settings#environment-variables`,
  hooks: `${DOCUMENTATION_BASE}/hooks`,
}
```

作为 `TIP_MATCHERS` 的补充，为未显式指定文档链接的匹配项提供基于路径前缀的默认链接。

## 具体技术实现（关键流程/数据结构/协议/命令）

### `DOCUMENTATION_BASE`
```ts
const DOCUMENTATION_BASE = 'https://code.claude.com/docs/en'
```

所有文档链接均基于该域名构建。当前为英文文档路径（`/docs/en`）。

### 匹配器优先级与短路逻辑
`TIP_MATCHERS` 是一个普通数组，`getValidationTip` 使用 `Array.prototype.find` 查找第一个匹配项。这意味着：
- **数组顺序至关重要**。更具体的规则必须放在更通用的规则之前。
- 例如规则 4（`env.*` + `invalid_type`）必须放在规则 7（通用 `invalid_type` + expected `boolean`）之前，否则环境变量的类型错误会被错误地提示为 "Use true or false without quotes"。

### 动态枚举值生成的实现细节
```ts
if (
  context.code === 'invalid_value' &&
  context.enumValues &&
  !tip.suggestion
) {
  tip.suggestion = `Valid values: ${context.enumValues.map(v => `"${v}"`).join(', ')}`
}
```

这条逻辑覆盖了所有枚举字段（如 `defaultShell`、`forceLoginMethod`、`effortLevel`、`autoUpdatesChannel` 等），无需为每个枚举字段单独写匹配规则。

### Hooks 提示的历史修正
规则 6（`path.includes('hooks')`）的 `suggestion` 注释中特别提到：
> "gh-31187 / CC-282: prior example showed `{\"matcher\": {\"tools\": [\"BashTool\"]}}` — an object format that never existed in the schema... Users copied the tip's example and got the same validation error again."

这说明该提示内容经历过一次**教训深刻的迭代**：早期的示例错误地展示了一个不存在的对象格式，导致用户复制粘贴后仍然报错。当前提示已修正为字符串 matcher 格式，并明确说明 `"matcher"` 是字符串（可以是工具名、管道分隔列表或空字符串匹配全部）。

## 关键代码路径与文件引用

### 上游调用方
| 文件 | 调用符号 | 场景 |
|------|----------|------|
| `src/utils/settings/validation.ts` | `getValidationTip` | `formatZodError` 中为每个 Zod issue 获取修复建议和文档链接 |

### 直接依赖
| 文件 | 导入符号 | 说明 |
|------|----------|------|
| `zod/v4` | `ZodIssueCode` | 用于推导 `ZodIssueCodeType` |

### 下游消费方
- `src/components/InvalidSettingsDialog.tsx`：将 `ValidationError.suggestion` 和 `ValidationError.docLink` 展示在无效设置对话框中。
- `src/components/ValidationErrorsList.tsx`：通用错误列表组件渲染 suggestion 和链接。
- `src/hooks/notifs/useSettingsErrors.tsx`：通知系统可能展示 suggestion 文本。

## 依赖与外部交互

### 与 `validation.ts` 的协作流程
```
ZodError (from SettingsSchema)
    ↓
validation.ts::formatZodError(issue, filePath)
    构造 TipContext { path, code, expected, received, enumValues, message, value }
    ↓
validationTips.ts::getValidationTip(context)
    匹配 TIP_MATCHERS → 动态补充枚举值/文档链接
    ↓
返回 { suggestion?, docLink? }
    ↓
validation.ts 将其写入 ValidationError
    ↓
React 组件层展示给用户
```

### 与文档站点的耦合
所有文档链接均硬编码指向 `https://code.claude.com/docs/en/...`。若文档站点结构调整（如章节 ID 变更、国际化路径变更），这些链接可能 404。当前没有链接有效性检查机制。

## 风险、边界与改进建议

### 风险

1. **提示规则与 schema 演进不同步**
   - `TIP_MATCHERS` 是手动维护的数组。当 `types.ts` 中新增字段、修改枚举值或重命名字段时，很容易忘记同步更新对应的提示规则。
   - 例如：若 `permissions.defaultMode` 未来新增一个枚举值，规则 1 的 suggestion 需要手动更新；若遗漏，用户看到的提示将不完整。

2. **匹配器顺序脆弱**
   - 当前依赖数组顺序实现优先级。新增通用规则时，若不小心放在具体规则之前，会导致具体规则的提示被覆盖。
   - 没有自动化测试验证匹配器顺序和覆盖范围。

3. **文档链接硬编码且无版本控制**
   - 链接指向线上最新文档。若用户使用的是旧版本 Claude Code，而线上文档已更新描述新版本行为，链接内容可能与用户实际软件行为不一致。
   - 没有按版本号（如 `/docs/en/v1.2/...`）区分文档。

4. **国际化（i18n）缺失**
   - 所有提示文本均为英文硬编码。Claude Code 支持多语言 UI，但设置校验提示目前无法随用户语言偏好切换。

5. **`path.includes('hooks')` 的过度匹配**
   - 规则 6 使用 `path.includes('hooks')`，这意味着任何包含 "hooks" 子串的路径都会命中（虽然实际 schema 中不太可能有其他字段名包含 "hooks"，但理论上存在误匹配风险）。

### 边界

- **不执行任何校验逻辑**：只负责"解释错误"，不负责"发现错误"。
- **不处理非 Zod 错误**：JSON 语法错误（`SyntaxError`）由 `validation.ts` 的 `jsonParse` 异常处理，不经过 `getValidationTip`。
- **不处理 AI 专用的策略提示**：如 `validateEditTool.ts` 中的 `"IMPORTANT: Do not update the env unless explicitly instructed to do so."` 不在本文件中管理。

### 改进建议

1. **引入部分自动化生成**
   - 对于纯枚举字段的提示，可考虑在 `types.ts` 的 `.describe()` 中增加标准化标记（如 `"@enum-tip"`），然后由脚本自动生成 `TIP_MATCHERS` 的子集，减少手动维护负担。

2. **增加匹配器顺序的单元测试**
   - 编写测试确保：
     - `env.*` 规则优先于通用 `invalid_type` 规则；
     - `permissions.defaultMode` 规则优先于通用 `invalid_value` 规则；
     - 新加入的通用规则不会意外覆盖已有具体规则。

3. **文档链接版本化或增加回退页**
   - 将链接改为包含版本号（如 `.../docs/en/v{VERSION}/settings`），或在文档站点设置旧版重定向规则。
   - 至少增加一个通用回退页（`.../docs/en/settings`）作为兜底。

4. **i18n 基础设施预留**
   - 将 `TIP_MATCHERS` 中的硬编码字符串抽离为键值映射表（如 `tips.en.json`），为未来多语言切换预留接口。

5. **细化 `hooks` 匹配条件**
   - 将 `path.includes('hooks')` 改为更精确的前缀匹配，如 `path.startsWith('hooks.') || path.startsWith('strictPluginOnlyCustomization') && ...`，避免未来 schema 中出现含 "hooks" 子串的无关字段时误触发。

6. **增加链接健康检查脚本**
   - 在 CI 中增加一个轻量脚本，定期抓取 `PATH_DOC_LINKS` 和 `TIP_MATCHERS` 中的所有 `docLink`，检查 HTTP 200，防止文档重构导致链接失效。

7. **扩展提示覆盖范围**
   - 当前 11 条规则主要覆盖权限、环境变量、hooks、API key helper、清理周期等高频字段。对于较新的字段（如 `autoMode`、`sandbox`、`sshConfigs`、`pluginConfigs`）尚无专门提示，可根据用户反馈数据逐步补充。
