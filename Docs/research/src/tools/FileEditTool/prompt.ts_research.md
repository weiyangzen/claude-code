# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 负责生成 `FileEditTool`（对外名称为 `'Edit'`）的系统提示词（system prompt / tool description）。该描述在每次模型请求时动态构建，用于指导大语言模型如何正确使用文件编辑工具。提示词内容会根据当前环境（如 `USER_TYPE`、`isCompactLinePrefixEnabled`）进行条件化裁剪，以适配不同用户群体（内部 Ant 用户 vs 外部三方用户）和不同行号前缀格式。

核心职责：
- 提供 `getEditToolDescription()` 作为 `FileEditTool.ts` 中 `prompt()` 方法的实现。
- 强制声明“必须先使用 `FileReadTool`（`Read`）读取文件，才能使用 `Edit` 工具”的前置约束。
- 指导模型正确处理行号前缀、缩进、唯一性上下文、`replace_all` 的使用等编辑规范。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getEditToolDescription()` | 对外暴露的统一描述入口，返回动态生成的 Edit 工具使用说明。 |
| `getPreReadInstruction()` | 强制插入“必须先 Read 再 Edit”的约束，防止模型未读文件直接尝试编辑。 |
| `getDefaultEditDescription()` | 构建完整的默认编辑描述，包含行号格式提示、缩进保留提示、`replace_all` 用法提示等。 |

## 具体技术实现

### 1. 模块结构与导出

```ts
import { isCompactLinePrefixEnabled } from '../../utils/file.js'
import { FILE_READ_TOOL_NAME } from '../FileReadTool/prompt.js'

function getPreReadInstruction(): string { ... }

export function getEditToolDescription(): string {
  return getDefaultEditDescription()
}

function getDefaultEditDescription(): string { ... }
```

- 仅导出 `getEditToolDescription`。
- 内部依赖 `FILE_READ_TOOL_NAME`（来自 `FileReadTool/prompt.ts`）以确保“先 Read 后 Edit”的提示中引用的工具名与注册名一致。
- 依赖 `isCompactLinePrefixEnabled()`（来自 `utils/file.ts`）以动态决定行号前缀格式描述。

### 2. `getPreReadInstruction()`

返回一段 Markdown 格式的强制约束：

```markdown
- You must use your `Read` tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file.
```

这段文本被硬编码插入到 `Usage:` 列表的第一项，是 `FileEditTool.ts` 中 `validateInput` 的 read-before-write 校验在 prompt 层的对应表达。

### 3. `getDefaultEditDescription()`

动态拼接以下元素：

#### A. 行号前缀格式描述

```ts
const prefixFormat = isCompactLinePrefixEnabled()
  ? 'line number + tab'
  : 'spaces + line number + arrow'
```

- **Compact 模式**：行号后直接跟 `\t`，例如 `15\tconst x = 1`。
- **传统模式**：行号右对齐 6 位空格 + `→` 箭头，例如 `    15→const x = 1`。

该提示明确告诉模型："Everything after that is the actual file content to match. Never include any part of the line number prefix in the old_string or new_string."

#### B. 最小唯一性提示（Ant 专属）

```ts
const minimalUniquenessHint =
  process.env.USER_TYPE === 'ant'
    ? `\n- Use the smallest old_string that's clearly unique — usually 2-4 adjacent lines is sufficient. Avoid including 10+ lines of context when less uniquely identifies the target.`
    : ''
```

仅当环境变量 `USER_TYPE=ant` 时附加，用于引导内部模型使用更精简的上下文，减少 token 消耗并降低多行匹配失败概率。

#### C. 完整提示词模板

```markdown
Performs exact string replacements in files.

Usage:
- You must use your `Read` tool at least once in the conversation before editing...
- When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix. The line number prefix format is: {prefixFormat}. Everything after that is the actual file content to match. Never include any part of the line number prefix in the old_string or new_string.
- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
- Only use emojis if the user explicitly requests it. Avoid adding emojis to files unless asked.
- The edit will FAIL if `old_string` is not unique in the file. Either provide a larger string with more surrounding context to make it unique or use `replace_all` to change every instance of `old_string`.{minimalUniquenessHint}
- Use `replace_all` for replacing and renaming strings across the file. This parameter is useful if you want to rename a variable for instance.
```

## 关键代码路径与文件引用

- **定义文件**：`src/tools/FileEditTool/prompt.ts`
- **直接调用方**：
  - `src/tools/FileEditTool/FileEditTool.ts` — `prompt()` 方法返回 `getEditToolDescription()`。
- **依赖文件**：
  - `src/utils/file.ts` — `isCompactLinePrefixEnabled()`
  - `src/tools/FileReadTool/prompt.ts` — `FILE_READ_TOOL_NAME`
- **间接引用方**：
  - 系统提示词构建链路：`src/constants/prompts.ts` 在组装完整 system prompt 时，会遍历各工具的 `prompt()` 输出。
  - `src/services/tools/toolExecution.ts` 等模块在构建模型请求时也会间接消费该描述。

## 依赖与外部交互

| 依赖模块 | 交互方式 | 说明 |
|----------|----------|------|
| `utils/file.ts` | 函数调用 | `isCompactLinePrefixEnabled()` 决定行号前缀格式描述。 |
| `FileReadTool/prompt.ts` | 常量导入 | `FILE_READ_TOOL_NAME` 确保 Read 工具名称引用正确。 |
| `FileEditTool.ts` | 被调用 | `prompt()` 方法直接返回 `getEditToolDescription()`。 |

## 风险、边界与改进建议

### 风险与边界

1. **`USER_TYPE` 环境变量的脆弱性**：`minimalUniquenessHint` 完全依赖 `process.env.USER_TYPE === 'ant'`。该环境变量若被外部用户设置，将导致提示词泄露内部优化策略；若未设置，内部用户则无法获得优化提示。
2. **行号前缀格式与 `file.ts` 的耦合**：`isCompactLinePrefixEnabled()` 的实现位于 `utils/file.ts`，与 `addLineNumbers()` 和 `stripLineNumberPrefix()` 强耦合。若 `addLineNumbers` 的格式变更而 `prompt.ts` 未同步，模型将收到错误的编辑指导，导致大量 `old_string` 匹配失败。
3. **提示词缺乏动态文件类型适配**：当前提示词对所有文件类型使用同一套描述，未针对二进制文件、JSON（无 trailing comma 敏感）、Markdown（hard line break 由两个空格构成）等特殊格式提供额外指导。
4. **无版本化或 A/B 测试支持**：提示词为纯字符串拼接，未接入 GrowthBook 等实验框架，难以对提示词变更进行受控实验。

### 改进建议

1. **将 `USER_TYPE` 检查替换为更正式的 feature flag**：例如通过 `checkStatsigFeatureGate` 或 `getFeatureValue` 判断是否为内部实验组，避免依赖裸环境变量。
2. **建立行号前缀格式的单一事实来源**：在 `utils/file.ts` 中增加 `getLinePrefixFormatDescription(): string` 函数，由 `prompt.ts` 直接引用，消除硬编码格式描述与实现之间的漂移风险。
3. **按文件类型追加编辑提示**：在 `getEditToolDescription` 中接收可选的 `fileExtension` 参数，针对 `.md`、`.json`、`.yaml` 等常见格式追加简短但关键的编辑注意事项（如 Markdown 的双空格换行、JSON 的引号要求）。
4. **提示词版本化与遥测**：为生成的描述附加一个隐式版本标识（如哈希或递增版本号），在 `validateInput` 失败时记录模型收到的提示词版本，便于分析是提示词本身还是模型理解问题导致的编辑失败。
5. **考虑将 `getPreReadInstruction` 提取为可复用约束模板**：仓库中多个写工具（`FileWriteTool`、`NotebookEditTool` 等）可能都需要“先 Read 再 Write”的约束。可将其提升为 `src/constants/toolConstraints.ts` 级别的共享模板，确保各工具文案一致。
