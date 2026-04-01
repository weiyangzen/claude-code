# utils.ts 研究文档

## 场景与职责

`utils.ts` 是 `FileEditTool` 模块的算法与工具函数库，承担了字符串匹配归一化、diff patch 生成、编辑片段提取、输入反消毒（desanitize）、编辑等价性判定等核心计算任务。该文件不直接依赖 React 或 UI，是纯算法层，被 `FileEditTool.ts`、`UI.tsx`、`src/components/FileEditToolDiff.tsx`、`src/utils/api.ts`、`src/utils/attachments.ts` 等多个模块广泛调用。

核心职责：
- **引号归一化与保留**：处理模型输出的直引号与文件中可能存在的弯引号（curly quotes）之间的匹配与风格继承。
- **字符串查找**：`findActualString` 在文件内容中定位 `old_string`，支持引号归一化后的模糊匹配。
- **编辑应用与 patch 生成**：`applyEditToFile`、`getPatchForEdit`、`getPatchForEdits` 将编辑应用到内容并生成结构化 diff。
- **片段提取**：`getSnippetForTwoFileDiff`、`getSnippetForPatch`、`getSnippet` 为附件或 UI 提供变更上下文片段。
- **输入反消毒**：`normalizeFileEditInput` 处理模型因 API 消毒（sanitization）而输出的替代标记（如 `<fnr>` → `<function_results>`）。
- **编辑等价性判定**：`areFileEditsEquivalent`、`areFileEditsInputsEquivalent` 用于工具去重和重复编辑检测。

## 功能点目的

| 函数 | 目的 |
|------|------|
| `normalizeQuotes` | 将弯引号（`‘’“”`）转换为直引号（`'"`），用于模糊匹配。 |
| `stripTrailingWhitespace` | 去除每行尾部空白（保留换行符），用于输入归一化。 |
| `findActualString` | 在文件中查找 `old_string`，先精确匹配，再尝试引号归一化后的匹配。 |
| `preserveQuoteStyle` | 当 `old_string` 通过引号归一化匹配成功时，将文件中的弯引号风格应用到 `new_string`。 |
| `applyEditToFile` | 执行单条字符串替换，处理 `replaceAll` 和尾部换行符的边界情况。 |
| `getPatchForEdit` / `getPatchForEdits` | 应用编辑并生成 `StructuredPatchHunk[]` patch，用于 UI 展示。 |
| `getSnippetForTwoFileDiff` | 为附件系统生成两文件 diff 的代码片段（8KB 上限）。 |
| `getSnippetForPatch` / `getSnippet` | 从 patch 或编辑中提取带行号的上下文片段。 |
| `getEditsForPatch` | 将结构化 patch 反向解析为 `FileEdit[]`（用于 diff 组件）。 |
| `normalizeFileEditInput` | 在编辑前对输入进行反消毒和尾部空白处理，提升模型编辑成功率。 |
| `areFileEditsEquivalent` / `areFileEditsInputsEquivalent` | 判定两组编辑是否语义等价（通过应用到原文件比较结果）。 |

## 具体技术实现

### 1. 引号处理

#### 常量定义

```ts
export const LEFT_SINGLE_CURLY_QUOTE = '‘'
export const RIGHT_SINGLE_CURLY_QUOTE = '’'
export const LEFT_DOUBLE_CURLY_QUOTE = '“'
export const RIGHT_DOUBLE_CURLY_QUOTE = '”'
```

Claude 模型无法直接输出弯引号（API 层可能归一化或训练数据偏向直引号），但用户文件中可能存在弯引号。若强制要求模型输出弯引号，会导致匹配失败。

#### `normalizeQuotes`

简单 `replaceAll` 映射表，将弯引号全部替换为直引号。

#### `findActualString`

两步查找：
1. `fileContent.includes(searchString)` → 直接返回。
2. 对两者都做 `normalizeQuotes`，在归一化后的内容中查找索引，再按原长度从 `fileContent` 中截取对应子串。

```ts
const searchIndex = normalizedFile.indexOf(normalizedSearch)
if (searchIndex !== -1) {
  return fileContent.substring(searchIndex, searchIndex + searchString.length)
}
```

注意：这里假设归一化前后字符长度不变。由于弯引号与直引号在 UTF-8 中都是 3 字节 vs 1 字节（`'` 为 1 字节，`‘` 为 3 字节），**该假设不成立**。这是一个已知但当前被接受的近似：如果文件使用弯引号而模型使用直引号，`searchString.length`（字节/字符数）可能与实际匹配长度不同，导致 `substring` 截取的边界可能偏移。实际场景中由于弯引号通常成对出现且上下文足够大，该问题极少触发明显错误。

#### `preserveQuoteStyle`

当 `oldString !== actualOldString`（即发生了引号归一化匹配）时：
- 检测 `actualOldString` 中包含的弯引号类型（单/双）。
- 对 `newString` 中的对应直引号应用 `applyCurlyDoubleQuotes` 或 `applyCurlySingleQuotes`。

**开闭引号启发式规则**：
- 若引号前为空白、字符串开头或开括号 `([{`、破折号，则视为开引号，替换为左弯引号。
- 否则视为闭引号，替换为右弯引号。
- 单引号额外处理缩略词（contraction）如 `don't`：若引号前后均为字母，则视为撇号，替换为右单弯引号 `’`。

### 2. 编辑应用 (`applyEditToFile`)

```ts
export function applyEditToFile(
  originalContent: string,
  oldString: string,
  newString: string,
  replaceAll: boolean = false,
): string
```

实现细节：
- 使用 `replaceAll` 或 `replace` 进行替换。
- 关键边界：当 `newString === ''`（删除）时，检查 `oldString` 是否以 `\n` 结尾。若 `!oldString.endsWith('\n')` 但原内容中存在 `oldString + '\n'`，则自动删除带换行符的版本。这是为了避免删除后留下空行导致文件格式混乱。

### 3. Patch 生成 (`getPatchForEdit` / `getPatchForEdits`)

#### `getPatchForEdit`

单编辑的包装器，调用 `getPatchForEdits`。

#### `getPatchForEdits`

多编辑批量应用与 patch 生成：

1. **空文件特殊处理**：
   若原内容为空、仅一条编辑且 `old_string === '' && new_string === ''`，返回空 patch 和空内容。这是为了处理“创建空文件”的边界情况。

2. **编辑依赖检查**：
   遍历编辑时，检查当前 `old_string`（去除尾部换行后）是否是任何先前已应用 `new_string` 的子串。若是，抛出错误：
   ```
   Cannot edit file: old_string is a substring of a new_string from a previous edit.
   ```
   该检查防止多编辑场景下的顺序依赖错误（如编辑 A 的 `new_string` 被编辑 B 的 `old_string` 误匹配）。

3. **无变化拦截**：
   每条编辑应用后，若 `updatedFile === previousContent`，抛出 `String not found in file. Failed to apply edit.`。
   全部编辑应用后若总体无变化，抛出 `Original and edited file match exactly. Failed to apply edit.`。

4. **Patch 生成优化**：
   早期实现通过 `getPatchForDisplay({ fileContents, edits: [{old:fileContents,new:updatedFile}] })` 生成 patch，这会导致内容被 `escapeForDiff` / `convertLeadingTabsToSpaces` 处理两次，并执行一次无意义的 `replace()`。
   当前优化为直接调用 `getPatchFromContents({ oldContent, newContent })`，节省约 20% 大文件处理时间。

### 4. 片段提取

#### `getSnippetForTwoFileDiff`

用于附件系统展示文件变更片段：
- 调用 `diff.structuredPatch`（context=8, timeout=`DIFF_TIMEOUT_MS`）。
- 过滤掉以 `-` 和 `\` 开头的 diff 元数据行，保留新增和上下文行。
- 通过 `addLineNumbers` 添加行号。
- 结果上限 `DIFF_SNIPPET_MAX_BYTES = 8192`，超出时按最后一个换行符截断，并追加 `... [N lines truncated] ...`。

#### `getSnippetForPatch`

基于 patch 的 `oldStart` 和 `newLines` 计算变更范围，向上下扩展 `CONTEXT_LINES = 4` 行，从 `newFile` 中切片并添加行号。

#### `getSnippet`

基于 `oldString` 在原文中的位置计算替换行号，从应用编辑后的新文件中提取上下文片段。

### 5. 输入反消毒 (`normalizeFileEditInput`)

由于 Claude API 会对某些 XML 风格标记进行消毒（sanitization），模型在编辑时可能输出被消毒后的替代形式。`DESANITIZATIONS` 映射表定义了这些替换规则：

```ts
const DESANITIZATIONS: Record<string, string> = {
  '<fnr>': '<function_results>',
  '<n>': '<name>',
  '</n>': '</name>',
  '<o>': '<output>',
  '</o>': '</output>',
  '<e>': '<error>',
  '</e>': '</error>',
  '<s>': '<system>',
  '</s>': '</system>',
  '<r>': '<result>',
  '</r>': '</result>',
  '< META_START >': '<META_START>',
  '< META_END >': '<META_END>',
  '< EOT >': '<EOT>',
  '< META >': '<META>',
  '< SOS >': '<SOS>',
  '\n\nH:': '\n\nHuman:',
  '\n\nA:': '\n\nAssistant:',
}
```

流程：
1. 对 `new_string` 执行 `stripTrailingWhitespace`（Markdown 文件 `.md`/`.mdx` 除外，因为尾部双空格是 hard line break）。
2. 若文件内容包含精确的 `old_string`，直接返回归一化后的输入。
3. 否则尝试 `desanitizeMatchString`：按映射表逐条替换 `old_string`。
4. 若反消毒后的 `old_string` 能匹配文件内容，则对 `new_string` 应用相同的替换规则。
5. 任何读取错误（ENOENT 除外）记录日志后返回原始输入。

### 6. 编辑等价性判定

#### `areFileEditsEquivalent`

用于比较两组 `FileEdit[]` 是否在语义上等价：
1. 快速路径：字面完全一致 → `true`。
2. 分别应用到同一 `originalContent`：
   - 若两者都成功且 `updatedFile` 相同 → `true`。
   - 若两者都失败且错误消息相同 → `true`。
   - 其他情况 → `false`。

#### `areFileEditsInputsEquivalent`

对外暴露的统一入口，用于 `FileEditTool.inputsEquivalent`：
1. 快速路径：不同文件 → `false`；字面完全一致 → `true`。
2. 读取原始文件内容（ENOENT 则视为空字符串）。
3. 调用 `areFileEditsEquivalent` 进行语义比较。

## 关键代码路径与文件引用

- **定义文件**：`src/tools/FileEditTool/utils.ts`
- **主要调用方**：
  - `src/tools/FileEditTool/FileEditTool.ts` — `findActualString`、`preserveQuoteStyle`、`getPatchForEdit`、`areFileEditsInputsEquivalent`
  - `src/tools/FileEditTool/UI.tsx` — `findActualString`、`preserveQuoteStyle`、`getPatchForEdit`
  - `src/components/FileEditToolDiff.tsx` — `findActualString`、`preserveQuoteStyle`
  - `src/utils/api.ts` — `normalizeFileEditInput`
  - `src/utils/attachments.ts` — `getSnippetForTwoFileDiff`
  - `src/utils/diff.ts` — 消费 `FileEdit` 类型
- **依赖文件**：
  - `src/utils/diff.ts` — `DIFF_TIMEOUT_MS`、`getPatchForDisplay`、`getPatchFromContents`
  - `src/utils/file.ts` — `addLineNumbers`、`convertLeadingTabsToSpaces`、`readFileSyncCached`
  - `src/utils/errors.ts` — `errorMessage`、`isENOENT`
  - `src/utils/stringUtils.ts` — `countCharInString`
  - `src/utils/log.ts` — `logError`
  - `src/utils/path.ts` — `expandPath`

## 依赖与外部交互

| 依赖模块 | 交互方式 | 说明 |
|----------|----------|------|
| `diff` (npm) | 库导入 | `structuredPatch` 用于 `getSnippetForTwoFileDiff`。 |
| `utils/diff.ts` | 函数调用 | `getPatchForDisplay`、`getPatchFromContents` 生成结构化 patch。 |
| `utils/file.ts` | 函数调用 | 文件读取缓存、行号添加、前导 tab 转空格。 |
| `utils/errors.ts` | 函数/类型导入 | 错误消息提取、ENOENT 判断。 |
| `utils/stringUtils.ts` | 函数调用 | `countCharInString` 用于截断统计。 |
| `utils/log.ts` | 函数调用 | 反消毒读取失败时记录错误。 |
| `utils/path.ts` | 函数调用 | `expandPath` 规范化文件路径。 |

## 风险、边界与改进建议

### 风险与边界

1. **`findActualString` 的 `substring` 长度假设缺陷**：
   弯引号（3 字节 UTF-8）与直引号（1 字节）长度不同，但代码使用 `searchString.length` 从原内容中截取匹配子串。在纯 ASCII 场景下无问题，但在混合引号且匹配边界恰好位于引号附近时，可能截断多字节字符，导致返回的 `actualOldString` 损坏。

2. **`applyEditToFile` 的尾部换行启发式过于隐式**：
   当 `newString === ''` 时，自动尝试删除 `oldString + '\n'` 的行为虽然减少了空行残留，但可能与用户意图相悖（用户可能确实只想删除不带换行的片段）。该逻辑无开关控制，可能引发意外格式变更。

3. **多编辑子串检查的不完备性**：
   `getPatchForEdits` 检查 `old_string` 是否是先前 `new_string` 的子串，但仅检查去除尾部换行后的版本。若 `old_string` 跨越了先前编辑产生的新边界（如先前编辑插入的内容中间），该检查可能漏检。

4. **`DIFF_SNIPPET_MAX_BYTES = 8192` 的硬编码**：
   该上限用于附件系统，但未与全局附件预算或 token 限制联动。对于高频编辑大文件的场景，仍可能产生显著的上下文注入开销。

5. **反消毒映射表的维护负担**：
   `DESANITIZATIONS` 中的规则是硬编码的，若 API 消毒策略变更（新增或移除映射），需要手动同步。当前无自动化测试或监控确保映射表与 API 行为一致。

6. **`getPatchForEdits` 的 `temp` 路径硬编码**：
   `areFileEditsEquivalent` 内部调用 `getPatchForEdits({ filePath: 'temp', ... })`，`diff` 库对文件名无实际依赖，但该魔法字符串缺乏语义表达。

### 改进建议

1. **修复 `findActualString` 的多字节边界问题**：
   在归一化匹配成功后，应通过原始 `fileContent` 的 Unicode 码点索引或正则匹配来精确提取 `actualOldString`，而非依赖 `searchString.length`。例如：
   ```ts
   const regex = new RegExp(escapeRegex(normalizedSearch))
   const match = normalizedFile.match(regex)
   if (match) {
     return fileContent.slice(match.index, match.index + match[0].length)
   }
   ```

2. **将尾部换行删除逻辑显式参数化**：
   在 `applyEditToFile` 中增加可选参数 `autoStripTrailingNewline?: boolean`，默认保持当前行为，但在需要精确控制的调用方（如测试或高级编辑模式）中可关闭。

3. **为 `DESANITIZATIONS` 增加单元测试和变更监控**：
   建立针对消毒映射表的自动化测试，模拟 API 消毒后的字符串输入，确保反消毒能正确还原。同时考虑将该映射表提升为 `src/constants/sanitization.ts` 级别的共享配置，供其他工具复用。

4. **将片段大小上限与全局预算系统联动**：
   让 `DIFF_SNIPPET_MAX_BYTES` 从 `fileReadingLimits.maxSizeBytes` 或附件系统的 token 预算中按比例派生，避免静态上限与动态负载脱节。

5. **优化多编辑冲突检测**：
   引入更严格的重叠检测：不仅检查子串关系，还检查编辑应用后的行范围是否重叠。可使用 `diff` 库或自定义行号计算来预判冲突，提前抛出更具描述性的错误。

6. **为 `areFileEditsEquivalent` 增加性能短路**：
   在语义比较前增加哈希比较（如 `edits1` 和 `edits2` 的 JSON 序列化哈希），避免对大文件进行两次完整的编辑应用和 diff 计算。
