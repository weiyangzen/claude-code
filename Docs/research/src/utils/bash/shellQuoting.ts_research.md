# Research: src/utils/bash/shellQuoting.ts

## 场景与职责

`shellQuoting.ts` 是 Claude Code 中**高级 shell 命令引用与规范化工具集**。它位于底层 `shellQuote.ts` 之上，针对实际命令执行场景中的特殊需求提供封装：
1. **Heredoc 感知引用**：`shell-quote` 库在处理 heredoc 和多行字符串时会错误地转义 `!` 字符，`quoteShellCommand()` 对此做了特殊处理。
2. **Stdin 重定向管理**：判断命令是否需要自动追加 `< /dev/null`，以及检测命令是否已包含 stdin 重定向。
3. **Windows CMD 语法兼容**：将模型偶尔幻觉出的 Windows CMD 风格重定向 `>nul` 改写为 POSIX 的 `/dev/null`。

该模块是 `bashProvider.ts` 构建最终执行命令时的**关键预处理层**，直接影响每一条 bash 命令的引用形式和安全性。

## 功能点目的

### 1. `quoteShellCommand(command, addStdinRedirect = true)`
- **目的**：将用户输入的命令字符串安全地引用为可在 `eval` 中执行的格式。
- **Heredoc 特殊处理**：
  - 若命令包含 heredoc（`<<EOF`、`<<'EOF'` 等）或多行引号字符串：
    - 不使用 `shell-quote` 库（因为它会错误转义 `!`）。
    - 改用**单引号整体包裹**，并将命令内部的所有单引号替换为 `'"'"'`（bash 中拼接单引号的标准技巧）。
    - 若含 heredoc，不追加 stdin 重定向（heredoc 自身提供输入）。
    - 若仅含多行字符串（无 heredoc），根据 `addStdinRedirect` 决定是否追加 `< /dev/null`。
- **普通命令处理**：
  - 若 `addStdinRedirect` 为 true，调用 `quote([command, '<', '/dev/null'])`。
  - 否则调用 `quote([command])`。

### 2. `containsHeredoc(command)`
- **目的**：检测命令中是否包含 heredoc 语法。
- **匹配模式**：
  - 先排除位运算表达式（`\d\s*<<\s*\d`、`[[ \d+ << \d+ ]]`、`$((.*<<.*))`）。
  - 再匹配 heredoc 正则：`<<-?\s*(?:(['"]?)(\w+)\1|\\(\w+))`
  - 支持：`<<EOF`、`<<'EOF'`、`<<"EOF"`、`<<-EOF`、`<<-'EOF'`、`<<\EOF`。

### 3. `containsMultilineString(command)`
- **目的**：检测命令中是否包含跨行的引号字符串。
- **匹配模式**：
  - 单引号多行：`'(?:[^'\\]|\\.)*\n(?:[^'\\]|\\.)*'`
  - 双引号多行：`"(?:[^"\\]|\\.)*\n(?:[^"\\]|\\.)*"`
  - 使用非贪婪类匹配并正确处理转义引号 `\'` 和 `\"`。

### 4. `hasStdinRedirect(command)`
- **目的**：检测命令是否已包含 stdin 重定向（`< file`）。
- **正则**：`/(?:^|[\s;&|])<(?![<(])\s*\S+/`
  - 要求 `<` 前面是行首、空白或命令分隔符。
  - 负向前瞻排除 `<<`（heredoc）和 `<(`（进程替换）。
  - 要求 `<` 后跟空白和至少一个非空白字符（文件名）。

### 5. `shouldAddStdinRedirect(command)`
- **目的**：判断是否可以安全地为命令追加 `< /dev/null`。
- **规则**：
  - 若含 heredoc → `false`（会干扰 heredoc 终止符）。
  - 若已有 stdin 重定向 → `false`。
  - 其他情况 → `true`。

### 6. `rewriteWindowsNullRedirect(command)`
- **目的**：将 Windows CMD 风格的 `>nul` 重定向改写为 POSIX 的 `> /dev/null`。
- **背景**：模型在 Windows 环境（Git Bash / WSL）中偶尔会生成 `2>nul` 这样的 CMD 语法。在 Git Bash 中，`2>nul` 会创建一个名为 `nul` 的**字面文件**（`nul` 是 Windows 保留设备名，极难删除，会破坏 `git add .`）。
- **正则**：`/(\d?&?>+\s*)[Nn][Uu][Ll](?=\s|$|[|&;)\n])/g`
  - 匹配：`>nul`、`> NUL`、`2>nul`、`&>nul`、`>>nul`（不区分大小写）。
  - 不匹配：`>null`、`>nullable`、`>nul.txt`、`cat nul.txt`。
  - 替换为：`$1/dev/null`。
- **限制**：该正则不解析 shell 引号，因此 `"echo >nul"` 中的 `>nul` 也会被改写。但如注释所述，这种误改在字符串内部将 `>nul` 改为 `> /dev/null` 是**无害的**。

## 具体技术实现（关键流程/数据结构/协议/命令）

### Heredoc 检测正则详解
```ts
const heredocRegex = /<<-?\s*(?:(['"]?)(\w+)\1|\\(\w+))/
```
- `<<-?`：匹配 `<<` 或 `<<-`（heredoc 的 strip-leading-tabs 变体）。
- `\s*`：允许操作符与定界符之间的空白。
- `(?:(['"]?)(\w+)\1|\\(\w+))`：
  - 第一种：可选引号 + 单词 + 相同引号（`EOF`、`"EOF"`、`'EOF'`）。
  - 第二种：反斜杠 + 单词（`\EOF`，表示禁用变量展开的 heredoc）。

### 多行字符串检测正则详解
```ts
const singleQuoteMultiline = /'(?:[^'\\]|\\.)*\n(?:[^'\\]|\\.)*'/
const doubleQuoteMultiline = /"(?:[^"\\]|\\.)*\n(?:[^"\\]|\\.)*"/
```
- `'(?:[^'\\]|\\.)*'`：匹配单引号字符串，内容可以是任何非引号非反斜杠字符，或一个反斜杠加任意字符（转义序列）。
- `\n`：强制包含一个换行符。
- 整体要求字符串跨行且两端有匹配的引号。

### Windows NUL 重定向改写
```ts
const NUL_REDIRECT_REGEX = /(\d?&?>+\s*)[Nn][Uu][Ll](?=\s|$|[|&;)\n])/g

export function rewriteWindowsNullRedirect(command: string): string {
  return command.replace(NUL_REDIRECT_REGEX, '$1/dev/null')
}
```
- `\d?`：可选的文件描述符（如 `2>`）。
- `&?`：可选的 `&`（如 `&>`）。
- `>+`：一个或多个 `>`（`>` 或 `>>`）。
- `\s*`：允许 `>` 与 `nul` 之间的空白。
- `[Nn][Uu][Ll]`：不区分大小写匹配 `nul`。
- `(?=\s|$|[|&;)\n])`：正向前瞻，确保 `nul` 后面是空白、行尾或命令分隔符，避免匹配 `nul.txt` 等。

## 关键代码路径与文件引用

### 依赖
| 文件 | 引用符号 | 作用 |
|------|---------|------|
| `src/utils/bash/shellQuote.ts` | `quote` | 普通命令的安全 shell 引用 |

### 调用方
| 文件 | 引用符号 | 场景 |
|------|---------|------|
| `src/utils/shell/bashProvider.ts` | `quoteShellCommand`, `rewriteWindowsNullRedirect`, `shouldAddStdinRedirect` | 构建执行命令时的预处理与引用 |

## 依赖与外部交互

- **`shellQuote.ts`**：提供基础的 `quote()` 函数。`shellQuoting.ts` 在 heredoc/多行场景下绕过 `shell-quote` 库，直接手动构造单引号包裹字符串。
- **`bashProvider.ts`**：唯一的业务调用方。在 `buildExecCommand` 中：
  1. 先调用 `rewriteWindowsNullRedirect(command)` 规范化重定向。
  2. 调用 `shouldAddStdinRedirect(normalizedCommand)` 判断是否需要 stdin 重定向。
  3. 调用 `quoteShellCommand(normalizedCommand, addStdinRedirect)` 生成最终引用命令。
  4. 若命令包含管道 `|`，还会调用 `rearrangePipeCommand()` 调整 stdin 重定向位置。

## 风险、边界与改进建议

### 风险
1. **Heredoc 检测的误报与漏报**：
   - `containsHeredoc` 使用正则匹配，而非真正的 shell 解析。某些合法但不常见的 heredoc 变体（如带变量名的 heredoc、含非单词字符的定界符）可能无法匹配。
   - 位运算排除逻辑（`\d\s*<<\s*\d` 等）可能过于严格，导致某些合法的 heredoc 被误判为位运算（虽然概率极低）。
2. **`rewriteWindowsNullRedirect` 的引号内误改**：
   - 如注释所述，该正则不解析引号状态，因此 `"echo >nul"` 会被改为 `"echo > /dev/null"`。在字符串内部这是无害的，但如果命令是 `'echo >nul'`（单引号内），bash 会按字面执行 `echo > /dev/null`，行为改变但仍是安全的。真正的风险在于：如果某处将该改写后的命令再次解析，可能引入语义差异。
3. **`hasStdinRedirect` 的正则边界**：
   - 正则 `/(?:^|[\s;&|])<(?![<(])\s*\S+/` 要求 `<` 前是空白或分隔符。对于 `cmd< file`（无空格）不会匹配，导致 `shouldAddStdinRedirect` 返回 `true`，最终命令变成 `cmd< file < /dev/null`。bash 会执行两个 stdin 重定向，最后一个生效（`/dev/null`），这可能改变命令预期行为。

### 边界
- **仅处理 bash/zsh 语法**：Windows PowerShell、CMD 的语法不在设计范围内（`rewriteWindowsNullRedirect` 只是将 CMD 幻觉语法转换为 POSIX，并非完整支持）。
- **Heredoc 的引用策略是全局单引号包裹**：对于非常复杂的 heredoc（如 heredoc 内部本身含有 `eval` 或嵌套命令替换），全局单引号包裹可能导致 bash 执行时将其全部视为字面量，这可能与用户的预期行为不一致。但安全角度这是可接受的——heredoc 内容本就不应被 shell 展开。
- **多行字符串检测仅检测引号跨行**：对于使用反斜杠续行（`echo hello \\\n world`）或 ANSI-C 引用（`$'...\n...'`）的多行形式不检测。

### 改进建议
1. **使用 tree-sitter 替代正则检测 heredoc**：`treeSitterAnalysis.ts` 已经能准确识别 `heredoc_redirect` 节点。当 tree-sitter 可用时，应优先使用 AST 信息判断 heredoc 存在性，消除正则的误报/漏报风险。
2. **增强 `hasStdinRedirect` 的鲁棒性**：
   - 支持 `<` 前无空格的紧贴形式（`cmd<file`）。
   - 考虑使用 tree-sitter 的 `file_redirect` / `heredoc_redirect` 节点进行精确检测。
3. **将 `rewriteWindowsNullRedirect` 集成到更早期的预处理阶段**：当前仅在 `bashProvider.ts` 中调用。如果其他模块（如 `commands.ts` 的 `splitCommandWithOperators`）也处理原始命令字符串，可能也需要同样的规范化，以保持解析一致性。
4. **为 `quoteShellCommand` 增加更细粒度的 heredoc 类型处理**：
   - 当前所有 heredoc 统一使用单引号整体包裹。对于已经包含单引号的 heredoc，替换策略 `'"'"'` 是正确的，但可提取为独立工具函数并增加单元测试。
5. **补充测试覆盖**：该模块无测试文件。建议补充 heredoc 检测、多行字符串检测、stdin 重定向检测、Windows NUL 改写等核心功能的单元测试。
