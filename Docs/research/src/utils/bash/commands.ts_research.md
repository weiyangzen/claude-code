# 研究文档：src/utils/bash/commands.ts

## 场景与职责

`commands.ts` 是 BashTool 安全体系中的**命令解析与拆分中枢**，承担以下核心职责：

1. **命令拆分**：将复合命令（含 `&&`、`||`、`|`、`;`、重定向等）拆分为独立的子命令，供权限系统逐条校验。
2. **前缀提取**：通过 LLM 策略规范提取命令前缀（如 `git commit`），用于用户 allowlist/denylist 匹配。
3. **重定向提取**：从命令中剥离输出重定向（`>`、`>>`），提取目标路径供 `pathValidation.ts` 校验。
4. **安全分类**：识别" help 命令"快速放行，识别"不安全复合命令"强制进入人工审批。

该文件同时维护**旧版 regex/shell-quote 路径**（标记为 `_DEPRECATED`）和**新版 tree-sitter 路径**的桥接逻辑。当 tree-sitter 不可用时，这些旧函数仍是权限校验的最后一道防线。

## 功能点目的

### 1. `splitCommandWithOperators(command: string): string[]`
- 使用 `shell-quote` 解析命令，并通过占位符机制保留引号、换行符、转义括号。
- 提取并恢复 heredoc（`<<EOF`），因为 `shell-quote` 将 `<<` 错误解析为两个 `<` 操作符。
- 折叠相邻字符串和 glob，生成可供下游进一步处理的字符串数组。
- **安全核心**：处理行继续符（`\\<newline>`）时严格区分奇偶反斜杠数量，防止 `tr\<newline>aceroute` 被错误拆分为两个 token。

### 2. `splitCommand_DEPRECATED(command: string): string[]`
- 在 `splitCommandWithOperators` 基础上进一步剥离重定向操作符（`>`、`>>`、`>&`）及其目标。
- 过滤控制操作符（`&&`、`||`、`|`、`;` 等），最终返回"看似独立的子命令"数组。
- 被 `bashPermissions.ts`、`pathValidation.ts`、`prefix.ts` 等大量下游模块调用。

### 3. `extractOutputRedirections(cmd: string)`
- 从命令中提取输出重定向（`>`、`>>`），返回 `{ commandWithoutRedirections, redirections, hasDangerousRedirection }`。
- 支持 Zsh 强制覆盖语法：`>!`、`>|`、`2>!`、`2>|`、`>>&!`、`>>&|`。
- 对动态目标（含 `$`、`` ` ``、`*`、`~` 等）标记为 `hasDangerousRedirection: true`，强制人工审批。
- **安全关键**：heredoc 必须在行继续合并**之前**提取，否则 `cat <<'ls'\n> /etc/passwd` 中的重定向会被吞入 heredoc 体，绕过路径校验。

### 4. `isHelpCommand(command: string): boolean`
- 快速识别简单的 `--help` 命令（如 `python --help`）。
- 要求命令仅由字母数字 token 和 `--help` 组成，无引号、无其他 flag。
- 命中后跳过前缀提取，直接以完整命令作为前缀，减少 API 调用并提升性能。

### 5. `isUnsafeCompoundCommand_DEPRECATED(command: string): boolean`
- 判断命令是否为"不安全的复合命令"（包含 subshell、命令组等 shell 操作符，但不是简单的命令列表）。
- 若 `shell-quote` 解析失败，按"不安全"处理（fail-closed）。

### 6. `getCommandPrefix` / `getCommandSubcommandPrefix`
- 基于 `BASH_POLICY_SPEC`（内嵌的 LLM prompt）调用 `createCommandPrefixExtractor` / `createSubcommandPrefixExtractor`。
- 在 `isHelpCommand` 命中时作为 pre-check 短路返回。

## 具体技术实现

### splitCommandWithOperators 的占位符安全机制

```ts
const placeholders = generatePlaceholders() // 使用 randomBytes(8).toString('hex') 作为 salt
```

- 在调用 `shell-quote.parse` 前，将 `"` 替换为 `"PLACEHOLDER`、`'` 替换为 `'PLACEHOLDER`、`
` 替换为 `
PLACEHOLDER
`、`\(` 和 `\)` 替换为对应占位符。
- parse 完成后再将占位符恢复为原始字符。
- **安全目的**：防止恶意命令中字面包含占位符字符串，通过随机 salt 确保碰撞概率极低（16 位十六进制 salt，2^64 空间）。

### extractOutputRedirections 的复杂重定向处理

`handleRedirection` 函数是核心状态机，处理以下模式：

| 模式 | 示例 | 处理结果 |
|------|------|----------|
| 标准输出重定向 | `> file.txt` | 提取 `{ target: 'file.txt', operator: '>' }` |
| 追加重定向 | `>> file.txt` | 提取 `{ target: 'file.txt', operator: '>>' }` |
| 文件描述符重定向 | `2>&1` | 保留在命令中，不提取为路径 |
| POSIX 强制覆盖 | `>| file.txt` | 提取目标，跳过 `\|` |
| Zsh 强制覆盖 | `>! file.txt` | 提取目标（去掉 `!`），跳过 `!` |
| Zsh 无空格强制覆盖 | `>!file.txt` | 检查 `afterBang` 是否有危险扩展，安全则提取 `file.txt` |
| 合并 stdout/stderr | `>& file` / `>>& file` | 提取为 `>` / `>>` |

**`isSimpleTarget` 与 `hasDangerousExpansion` 的不变式**：
- 对每一个字符串重定向目标，要么 `isSimpleTarget === true`（被捕获并进入路径校验），要么 `hasDangerousExpansion === true`（被标记为危险并要求人工审批）。不存在两者皆为 false 的漏网之鱼。

### 行继续符（line continuation）的安全处理

```ts
command.replace(/\\+\n/g, match => {
  const backslashCount = match.length - 1
  if (backslashCount % 2 === 1) {
    // 奇数：最后一个反斜杠转义了换行符 → 行继续
    return '\\'.repeat(backslashCount - 1)
  } else {
    // 偶数：反斜杠成对，换行符是真实分隔符
    return match
  }
})
```

- **攻击示例**：`echo \\\<newline>rm -rf /` 若被错误合并，会被当作一条命令处理，而 bash 实际执行两条命令。该逻辑确保偶数反斜杠时保留换行符。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/commands.ts`
  - `splitCommandWithOperators` (L85)
  - `splitCommand_DEPRECATED` (L265)
  - `extractOutputRedirections` (L634)
  - `isHelpCommand` (L388)
  - `isUnsafeCompoundCommand_DEPRECATED` (L609)
  - `handleRedirection` (L860)
  - `reconstructCommand` (L1195)

- **依赖文件**：
  - `src/utils/bash/heredoc.ts`：`extractHeredocs`、`restoreHeredocs`
  - `src/utils/bash/shellQuote.ts`：`quote`、`tryParseShellCommand`
  - `src/utils/shell/prefix.ts`：`createCommandPrefixExtractor`、`createSubcommandPrefixExtractor`

- **调用方**：
  - `src/tools/BashTool/bashPermissions.ts`：`splitCommand_DEPRECATED`、`isUnsafeCompoundCommand_DEPRECATED`
  - `src/tools/BashTool/pathValidation.ts`：`splitCommand_DEPRECATED`、`extractOutputRedirections`
  - `src/utils/bash/ParsedCommand.ts`：`splitCommandWithOperators`、`extractOutputRedirections`
  - `src/utils/bash/prefix.ts`：`splitCommand_DEPRECATED`、`parseCommand`
  - `src/tools/BashTool/bashCommandHelpers.ts`：`splitCommand_DEPRECATED`、`isUnsafeCompoundCommand_DEPRECATED`

## 依赖与外部交互

- **外部库**：`shell-quote`、`crypto`（randomBytes）
- **内部模块**：`heredoc.ts`、`shellQuote.ts`、`../shell/prefix.js`
- **LLM 交互**：`getCommandPrefix` 通过 `createCommandPrefixExtractor` 在需要时调用 LLM 提取前缀
- **无直接网络/文件 IO**

## 风险、边界与改进建议

### 已知风险

1. **shell-quote 与 bash 的解析差分**
   - 这是本文件及整个旧路径的最大风险源。`shell-quote` 不是真正的 bash 解析器，对注释、heredoc、反引号嵌套、算术扩展等支持不完整。虽然通过大量黑名单和回退机制缓解了已知攻击，但未知差分仍可能存在。
   - 项目已通过 `bashParser.ts` + `ast.ts` 建立 tree-sitter 主路径，旧路径应逐步退役。

2. **相邻字符串折叠导致的重定向目标污染**
   - `splitCommandWithOperators` 会将相邻字符串折叠为一个带空格的字符串。对于 `cat > out /etc/passwd`，bash 将 `out` 作为目标、`/etc/passwd` 作为源文件，但折叠后得到 `out /etc/passwd`。`isStaticRedirectTarget` 通过拒绝含空格的目标进行防御，但这是一种事后补偿。

3. **`extractOutputRedirections` 的 subshell 跟踪**
   - 使用 `cmdSubDepth` 跟踪 `$(` 和 `)` 来避免在命令替换内部提取重定向。如果解析器对括号的识别与 bash 不一致（例如复杂的嵌套或转义），可能产生差分。

### 边界情况

- **空命令/空白命令**：`splitCommandWithOperators` 返回 `[]`；`extractOutputRedirections` 返回空重定向数组但 `hasDangerousRedirection: false`
- **解析失败**：所有函数均 fail-closed。`splitCommandWithOperators` 返回原始命令的单元素数组；`extractOutputRedirections` 返回 `hasDangerousRedirection: true`
- **heredoc 与行继续的交织**：已在多个安全注释中强调，提取顺序必须是 heredoc → line-continuation → parse

### 改进建议

1. **全面迁移到 tree-sitter 路径**
   - `ast.ts` 的 `parseForSecurity` 和 `bashParser.ts` 已提供更准确的 AST。建议将 `splitCommand_DEPRECATED`、`extractOutputRedirections` 等旧函数的使用点逐步替换为 AST 遍历。
   - `ParsedCommand.ts` 已提供 `TreeSitterParsedCommand` 和 `RegexParsedCommand_DEPRECATED` 的双轨实现，可作为迁移模板。

2. **统一重定向处理逻辑**
   - 当前 `extractOutputRedirections`（基于 shell-quote token）与 `ast.ts` 的 `walkFileRedirect`（基于 tree-sitter AST）存在两套并行的重定向解析逻辑。建议将 AST 路径作为唯一真实来源，旧路径仅作极端回退。

3. **移除 `_DEPRECATED` 函数的缓存污染**
   - `getCommandPrefix` 和 `getCommandSubcommandPrefix` 带有缓存。`clearCommandPrefixCaches` 在 `/clear` 时被调用，但旧路径的缓存键不包含命令解析版本信息。若 tree-sitter 和旧路径结果不一致，缓存可能导致错误命中。

4. **增加回归测试覆盖**
   - 当前未在 `src/` 下找到针对 `commands.ts` 的单元测试文件。建议为 `splitCommandWithOperators`、`extractOutputRedirections` 和 `isStaticRedirectTarget` 建立独立测试，覆盖历史漏洞的 PoC（如 `>!~root/.bashrc`、`2>!filename`、heredoc 绕过等）。
