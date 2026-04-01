# Research Document: src/utils/settings/validateEditTool.ts

## 场景与职责

`src/utils/settings/validateEditTool.ts` 是 **FileEditTool 的 settings 文件编辑安全闸**，职责极其聚焦：

1. **拦截对 Claude Code 设置文件的不合法修改**：当 AI 通过 `FileEditTool` 编辑 `settings.json` 或 `settings.local.json` 时，在真正落盘前校验编辑后的内容是否符合 `SettingsSchema`。
2. **防止 AI 将设置文件改到无法解析或无法通过 schema 校验的状态**，从而避免用户下次启动 Claude Code 时设置系统失效。
3. **不阻塞对已经损坏文件的编辑**：如果编辑前文件本身就不合法，则放行编辑（给用户/AI 机会修复）。

## 功能点目的

### `validateInputForSettingsFileEdit`
这是文件唯一导出的函数，签名如下：

```ts
export function validateInputForSettingsFileEdit(
  filePath: string,
  originalContent: string,
  getUpdatedContent: () => string,
): Extract<ValidationResult, { result: false }> | null
```

**设计意图**：
- 采用 `getUpdatedContent` 闭包而非直接传入字符串，是因为只有在确认需要校验时才执行编辑模拟（惰性计算）。
- 返回 `null` 表示"无需阻止"；返回 `{ result: false, ... }` 表示"阻止此次编辑"。

**校验逻辑三步走**：

1. **路径过滤**：调用 `isClaudeSettingsPath(filePath)`，只有 `.claude/settings.json` 和 `.claude/settings.local.json`（以及通过 CLI flag 指定的等效路径）才会进入后续校验。
2. **编辑前状态检查**：对 `originalContent` 调用 `validateSettingsFileContent`。
   - 若编辑前已不合法 → **直接返回 null（放行）**。 rationale：不应当阻止用户/AI 修复一个已经坏掉的文件。
   - 若编辑前合法 → 继续第三步。
3. **编辑后状态检查**：执行 `getUpdatedContent()` 获取模拟编辑后的完整内容，再次调用 `validateSettingsFileContent`。
   - 若编辑后不合法 → 返回详细的 `ValidationResult` 错误，包含：
     - 错误摘要（`afterValidation.error`）
     - 完整 JSON Schema（`afterValidation.fullSchema`）
     - 特别提醒：`IMPORTANT: Do not update the env unless explicitly instructed to do so.`
   - 若编辑后合法 → 返回 null，允许 FileEditTool 继续执行。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 依赖的校验函数
```ts
import { validateSettingsFileContent } from './validation.js'
```

`validateSettingsFileContent` 内部实现（见 `validation.ts`）：
1. 使用 `jsonParse(content)` 解析 JSON。
2. 调用 `SettingsSchema().strict().safeParse(jsonData)` 进行严格模式校验（`strict()` 会拒绝 schema 中未定义的键）。
3. 若失败，返回格式化后的错误信息 + 通过 `generateSettingsJSONSchema()` 生成的完整 JSON Schema 字符串。

### 错误返回结构
```ts
{
  result: false,
  message: `Claude Code settings.json validation failed after edit:\n${afterValidation.error}\n\nFull schema:\n${afterValidation.fullSchema}\nIMPORTANT: Do not update the env unless explicitly instructed to do so.`,
  errorCode: 10,
}
```

- `errorCode: 10` 在 `FileEditTool` 的错误码体系中表示 settings 校验失败（与文件不存在、字符串未找到等其他错误码区分）。

### 路径判定
```ts
import { isClaudeSettingsPath } from '../permissions/filesystem.js'
```

`isClaudeSettingsPath` 实现细节：
- 先对路径做 `expandPath` 展开（处理 `~`、相对路径等）。
- 再做 `normalizeCaseForComparison` 小写化，防止大小写绕过（如 `.cLauDe/Settings.locaL.json`）。
- 检查是否以平台分隔符结尾的 `.claude/settings.json` 或 `.claude/settings.local.json`。
- 最后与当前项目所有已知 settings 文件路径（包括 managed settings、flag settings 等）做绝对路径比对。

## 关键代码路径与文件引用

### 上游调用方
| 文件 | 调用位置 | 说明 |
|------|----------|------|
| `src/tools/FileEditTool/FileEditTool.ts` | `validateInput` 阶段（约第 346 行） | 在确认文件存在、old_string 能找到、非 notebook 等所有前置检查之后，最后一步执行 settings 校验 |

### 直接依赖
| 文件 | 导入符号 | 说明 |
|------|----------|------|
| `src/Tool.js` | `ValidationResult` (type) | 工具验证结果联合类型 |
| `src/utils/permissions/filesystem.js` | `isClaudeSettingsPath` | 判断目标文件是否为 Claude 设置文件 |
| `src/utils/settings/validation.js` | `validateSettingsFileContent` | 对字符串内容进行 JSON + Schema 校验 |

### 下游被影响方
- `FileEditTool` 的 `validateInput` 若返回 `{ result: false, errorCode: 10 }`，则工具调用会被拒绝，AI 会收到错误提示并有机会修正后重试。

## 依赖与外部交互

### FileEditTool 生命周期
`validateInputForSettingsFileEdit` 嵌入在 `FileEditTool.validateInput` 的末尾：
1. 检查路径权限（deny 规则）
2. 检查文件大小（< 1 GiB）
3. 读取文件内容
4. 检查文件存在性、notebook 类型、读取时间戳、old_string 匹配、replace_all 逻辑
5. **→ 调用 `validateInputForSettingsFileEdit` ←**
6. 若通过，返回 `{ result: true, meta: { actualOldString } }`

### validation.ts 的 strict 校验
由于 `validateSettingsFileContent` 使用了 `SettingsSchema().strict()`，这意味着：
- 不仅 schema 内定义的字段类型要正确，**任何未知顶层键都会导致校验失败**。
- 这与 `settings.ts` 中 `parseSettingsFile` 使用的非 strict `safeParse` 不同（日常加载允许未知键通过 `.passthrough()`）。
- 因此 `validateEditTool.ts` 对 AI 编辑的要求**比用户手动编辑更严格**：AI 不能引入拼写错误的字段名。

## 风险、边界与改进建议

### 风险

1. **"已损坏则放行" 策略的双刃剑**
   - 优点：不阻止修复行为。
   - 风险：如果文件只是"轻微不合法"（如某个字段类型错误），AI 可能进一步将其改得更糟，甚至删除大量有效内容。由于编辑前已不合法，本函数不会拦截。

2. **strict 校验与日常加载行为不一致**
   - 用户手动编辑 settings.json 添加一个拼写错误的字段（如 `cleanuPeriodDays`），日常启动时 `.passthrough()` 会保留该字段并忽略它；但 AI 通过 `FileEditTool` 添加同样的字段会被拒绝。
   - 这种不一致在用户体验上可能造成困惑：用户自己写进去没问题，AI 改却报错。

3. **错误提示信息过长**
   - `fullSchema` 是 `generateSettingsJSONSchema()` 输出的完整 JSON Schema，可能非常大（数十 KB）。直接塞进 `message` 返回给 AI，可能：
     - 超出上下文长度限制；
     - 淹没真正有用的错误信息；
     - 增加 token 消耗。

4. **只覆盖 settings.json，不覆盖其他配置文件**
   - `.mcp.json`、`.claude/commands/`、`.claude/agents/` 等文件的编辑不受此函数保护。若 AI 错误编辑 `.mcp.json`，同样可能导致配置损坏。

### 边界

- **不执行实际编辑**：仅通过闭包模拟编辑结果，不会修改磁盘。
- **不处理 JSON 语法错误修复**：如果 `originalContent` 本身不是合法 JSON，`validateSettingsFileContent` 会返回 `isValid: false`，此时直接放行，不做进一步分析。
- **不处理权限决策**：文件是否允许被编辑由 `FileEditTool.checkPermissions` 和 `checkWritePermissionForTool` 决定，本函数只负责 schema 合规性。

### 改进建议

1. **区分 "已损坏放行" 的修复方向**
   - 可考虑在 `beforeValidation.isValid === false` 时，仍然对 `afterValidation` 做"是否变得更坏"的评估。例如：若编辑后 JSON 都无法解析（而编辑前至少能解析），仍可拦截。

2. **缩短返回的 schema 信息**
   - 不必返回完整 `fullSchema`，可改为返回：
     - 错误字段的文档链接（利用 `validationTips.ts`）；
     - 或仅返回 schema 中相关子树的片段。
   - 可新增一个 `generateSettingsJSONSchemaSnippet(path)` 辅助函数，按需裁剪。

3. **统一 strict 策略或增加说明**
   - 考虑在错误消息中明确告知 AI："Unknown fields are not allowed. Please only use documented settings fields." 以减少因 strict 模式导致的困惑。

4. **扩展保护范围**
   - 可将类似的校验机制推广到 `.mcp.json` 或其他 Claude 配置文件，建立统一的 "Claude config file edit validator" 框架。

5. **增加单元测试覆盖**
   - 本次调研未在仓库中找到相关测试。建议补充以下场景：
     - 编辑合法 settings → 放行
     - 编辑后引入未知键 → 拦截
     - 编辑前已损坏 → 放行
     - 非 settings 文件路径 → 返回 null
     - 编辑后类型错误（如 `cleanupPeriodDays: "30"`）→ 拦截
