# 研究报告：src/tools/BashTool/bashSecurity.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 执行器：kimi | 模型：k2p5  
> 生成时间：2026-04-01

---

## 1. 场景与职责

### 1.1 所在系统位置
`bashSecurity.ts` 位于 `src/tools/BashTool/` 目录，是 **BashTool** 权限校验链中的**旧版（legacy）安全门**。它在以下两条主路径中被调用：

1. **主权限流** (`bashPermissions.ts` → `bashToolHasPermission` / `checkCommandAndSuggestRules`)：当 tree-sitter AST 解析不可用（`parse-unavailable`）或注入检查被禁用时，作为 `shell-quote` 时代的回退安全门。
2. **操作符/管道分段流** (`bashCommandHelpers.ts` → `checkCommandOperatorPermissions`)：在检测到不安全复合命令（subshell、command group）或管道时，用其生成更具体的拒绝理由。
3. **只读校验流** (`readOnlyValidation.ts`)：同步调用 `bashCommandIsSafe_DEPRECATED`，在只读命令快速路径前做兜底检查。

### 1.2 核心职责
该文件的职责是：**在无法使用 tree-sitter 进行结构化解析时，通过正则与字符级扫描检测可能导致“解析差异攻击（parser differential）”的 shell 构造**，并返回 `PermissionResult`（`allow` / `ask` / `deny` / `passthrough`）。

所谓“解析差异攻击”，是指 `shell-quote`（或本文件的字符扫描器）与真实 `bash` 对同一字符串的 tokenization/语义不一致，导致攻击者能把危险命令（如 `rm -rf /`）藏进看似无害的引号、转义、heredoc、注释或空字符中。

> 注释中反复强调：该路径已被标记为 `@deprecated`，主 gate 是 `src/utils/bash/ast.ts` 的 `parseForSecurity`。但在外部构建（TREE_SITTER_BASH off）或 WASM 加载失败时，本文件仍是唯一的安全防线。

---

## 2. 功能点目的

文件内所有功能可归纳为 **4 大类**：

| 大类 | 目的 | 代表函数/常量 |
|------|------|---------------|
| **A. 早期快速放行/拦截** | 对明显安全或明显危险的命令做短路判断，避免进入 heavy validators | `validateEmpty`, `validateIncompleteCommands`, `validateSafeCommandSubstitution`, `validateGitCommit` |
| **B. 危险模式检测** | 检测命令替换、重定向、变量注入、Zsh 专属绕过、控制字符等 | `validateDangerousPatterns`, `validateRedirections`, `validateDangerousVariables`, `validateIFSInjection`, `validateProcEnvironAccess`, `validateZshDangerousCommands`, `validateJqCommand` |
| **C. 解析差异/混淆检测** | 专门对抗 parser differential 与 flag obfuscation | `validateObfuscatedFlags`, `validateBackslashEscapedWhitespace`, `validateBackslashEscapedOperators`, `validateBraceExpansion`, `validateNewlines`, `validateCarriageReturn`, `validateUnicodeWhitespace`, `validateMidWordHash`, `validateCommentQuoteDesync`, `validateQuotedNewline`, `validateMalformedTokenInjection` |
| **D. 辅助工具函数** | 引号剥离、heredoc 处理、安全重定向剥离、字符逃逸判断等 | `extractQuotedContent`, `stripSafeRedirections`, `hasUnescapedChar`, `isSafeHeredoc`, `stripSafeHeredocSubstitutions`, `hasSafeHeredocSubstitution` |

### 2.1 关键导出接口

```typescript
// 同步旧版入口（readOnlyValidation.ts 等同步调用方使用）
export function bashCommandIsSafe_DEPRECATED(command: string): PermissionResult

// 异步旧版入口（主权限流使用），带可选 tree-sitter 增强
export async function bashCommandIsSafeAsync_DEPRECATED(
  command: string,
  onDivergence?: () => void,
): Promise<PermissionResult>

// 供外部（如 bashPermissions.ts 的 pre-split gate）剥离安全 heredoc 替换
export function stripSafeHeredocSubstitutions(command: string): string | null
export function hasSafeHeredocSubstitution(command: string): boolean
```

---

## 3. 具体技术实现

### 3.1 数据结构

#### 3.1.1 `ValidationContext`
所有 validator 共享的上下文对象：

```typescript
type ValidationContext = {
  originalCommand: string        // 原始命令（含 heredoc，未剥离）
  baseCommand: string            // 第一个空格前的词（含 env var 前缀）
  unquotedContent: string        // 单引号内容被移除，双引号内容保留（用于检测 $() 等）
  fullyUnquotedContent: string   // 所有引号内容都被移除，且 stripSafeRedirections
  fullyUnquotedPreStrip: string  // 所有引号内容移除，但保留重定向（给 validateBraceExpansion 用）
  unquotedKeepQuoteChars: string // 移除引号内内容，但保留引号字符本身 '' 或 ""（用于 mid-word hash）
  treeSitter?: TreeSitterAnalysis | null  // 异步路径可选传入
}
```

#### 3.1.2 `BASH_SECURITY_CHECK_IDS`
为避免在 analytics 中记录长字符串，每个检查对应一个数字 ID：

```typescript
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
} as const
```

#### 3.1.3 危险模式常量

- **`COMMAND_SUBSTITUTION_PATTERNS`**：正则数组，检测 `<()`、`>()`、`=$( )`（Zsh）、`$()`、`${}`、`$[]`、`~[`、`(e:`、`(+`、Zsh `always` 块、PowerShell `<#` 等。
- **`ZSH_DANGEROUS_COMMANDS`**：`Set<string>`，包含 `zmodload`, `emulate`, `sysopen`, `sysread`, `syswrite`, `zpty`, `ztcp`, `zsocket`, `zf_rm`, `zf_mv` 等可绕过常规二进制检查的 Zsh 内建命令。

### 3.2 关键流程

#### 3.2.1 `bashCommandIsSafe_DEPRECATED` 执行流程

```
1. CONTROL_CHAR_RE 检测 → 若命中，ask + isBashSecurityCheckForMisparsing=true
2. hasShellQuoteSingleQuoteBug 检测 → 若命中，ask + misparsing flag
3. extractHeredocs(command, { quotedOnly: true }) → 得到 processedCommand
4. extractQuotedContent(processedCommand) → 生成 ValidationContext 的各字段
5. Early validators（短路）:
   - validateEmpty → allow
   - validateIncompleteCommands → ask
   - validateSafeCommandSubstitution → allow（仅限安全 heredoc）
   - validateGitCommit → allow（仅限简单 -m "msg"）
6. Main validators 顺序执行（见下）
7. 若全部通过 → passthrough
```

**Main validators 执行顺序**（注意 `nonMisparsingValidators` 的延迟机制）：

```typescript
const validators = [
  validateJqCommand,                    // 检测 jq system() / -f / --from-file 等
  validateObfuscatedFlags,              // 检测 ANSI-C 引号、空引号接 dash、引号内 flag 等
  validateShellMetacharacters,          // 检测引号内的 ; | &（如 find -name "a;b"）
  validateDangerousVariables,           // 检测 $VAR 与重定向/管道相邻
  validateCommentQuoteDesync,           // 检测 # 注释内的引号导致下游 tracker 失步
  validateQuotedNewline,                // 检测引号内换行后紧跟 # 行（可绕过 stripCommentLines）
  validateCarriageReturn,               // 检测 \r（shell-quote 与 bash 对 \r 的 tokenization 差异）
  validateNewlines,                     // 检测未引号内的换行（非 misparsing，延迟）
  validateIFSInjection,                 // 检测 $IFS / ${...IFS...}
  validateProcEnvironAccess,            // 检测 /proc/*/environ
  validateDangerousPatterns,            // 检测反引号、$()、${}、<() 等
  validateRedirections,                 // 检测 < 或 >（非 misparsing，延迟）
  validateBackslashEscapedWhitespace,   // 检测 \空格 / \tab（parser differential）
  validateBackslashEscapedOperators,    // 检测 \; \| \& 等（splitCommand 会 normalize 成裸操作符）
  validateUnicodeWhitespace,            // 检测 Unicode 空白字符
  validateMidWordHash,                  // 检测 mid-word #（shell-quote 视其为注释起点，bash 视其为字面量）
  validateBraceExpansion,               // 检测 {a,b} / {1..5}（brace expansion 绕过）
  validateZshDangerousCommands,         // 检测 zmodload / emulate / fc -e 等
  validateMalformedTokenInjection,      // 检测 shell-quote 解析出的畸形 token 与操作符组合
]
```

**延迟机制**：`validateNewlines` 和 `validateRedirections` 被标记为 `nonMisparsingValidators`。如果它们先返回 `ask`，结果会被暂存（`deferredNonMisparsingResult`），继续执行后续 validators。若后续有 misparsing validator 返回 `ask`，则优先返回 misparsing 结果（带上 `isBashSecurityCheckForMisparsing: true`）。这是因为 `bashPermissions.ts` 对 misparsing 结果会提前拦截，而对普通 `ask` 可能走 classifier 自动批准路径。

#### 3.2.2 `bashCommandIsSafeAsync_DEPRECATED` 执行流程

与同步版本基本一致，但增加：

1. `await ParsedCommand.parse(command)` 获取 tree-sitter 分析。
2. 若 `tsAnalysis` 存在，使用 `tsAnalysis.quoteContext` 替代 `extractQuotedContent` 生成的 regex quote context。
3. **Divergence 检测**：若 `tsQuote.fullyUnquoted !== regexQuote.fullyUnquoted` 或 `withDoubleQuotes` 不一致，调用 `onDivergence()`（或记录 `tengu_tree_sitter_security_divergence` 事件）。
4. `validateBackslashEscapedOperators` 在 tree-sitter 路径下会检查 `context.treeSitter.hasActualOperatorNodes`——若 AST 中无真实操作符节点，则跳过（消除 `find -exec \;` 的误报）。

### 3.3 重要技术细节

#### 3.3.1 `extractQuotedContent` — 字符级引号剥离

手写状态机遍历命令字符串：
- 单引号内：一切字符原样输出到 `withDoubleQuotes`，但不输出到 `fullyUnquoted` / `unquotedKeepQuoteChars`。
- 双引号内：字符输出到 `withDoubleQuotes` 和 `unquotedKeepQuoteChars`，但不输出到 `fullyUnquoted`。
- 反斜杠：在单引号外视为转义；跳过下一个字符。
- 对 `jq` 特殊处理：双引号保留在 `withDoubleQuotes` 中，因为 jq 的过滤器语法重度依赖双引号字符串。

#### 3.3.2 `stripSafeRedirections` — 安全重定向剥离

```typescript
return content
  .replace(/\s+2\s*>&\s*1(?=\s|$)/g, '')
  .replace(/[012]?\s*>\s*\/dev\/null(?=\s|$)/g, '')
  .replace(/\s*<\s*\/dev\/null(?=\s|$)/g, '')
```

**安全注释强调**：所有模式必须以 `(?=\s|$)` 结尾作为边界。若缺少该边界，`> /dev/nullo` 会匹配 `/dev/null` 前缀，剥离后留下 `o`，导致 `echo hi > /dev/nullo` 被误判为无重定向，从而绕过只读路径检查。

#### 3.3.3 `isSafeHeredoc` — 安全 heredoc 替换的精确匹配

这是**早期放行路径**（early-allow），一旦返回 `true` 会跳过所有后续 validators。因此要求“可证明安全”而非“大概安全”。

允许的模式：
```
[prefix] $(cat <<'DELIM'\n
[body lines]\n
DELIM\n
) [suffix]
```

约束：
- 定界符必须是单引号包裹（`'DELIM'`）或转义（`\DELIM`），确保 heredoc 体无扩展。
- 关闭定界符必须是**第一行**精确匹配（bash 行为）。
- 支持两种形式：`DELIM` 单独一行后下一行是 `)`；或 `DELIM)` 同一行（PST_EOFTOKEN inline 形式）。
- `$(` 必须处于**参数位置**（前面有非空白前缀），不能是命令名位置。
- 剥离 heredoc 后的剩余文本只能包含 `[a-zA-Z0-9 \t"'\.\-/_@=,:+~]*`。
- 剩余文本必须递归通过 `bashCommandIsSafe_DEPRECATED`。
- **拒绝嵌套匹配**：若发现 heredoc 范围嵌套，直接返回 `false`（因为外层 heredoc 的 quoted body 中不可能出现真正的内层 heredoc）。

#### 3.3.4 `validateObfuscatedFlags` — 多层混淆检测

这是文件中最复杂的 validator 之一，防御各种“引号拼接出 flag”的攻击：

1. **ANSI-C 引号**：`$'...'` 可编码任意字符，直接 ask。
2. **Locale 引号**：`$"..."` 同理。
3. **空引号接 dash**：`$''-exec`、`''-exec`、`""-exec` 等。
4. **同质空引号对紧邻引号 dash**：`"""-f"`（空 `""` + 引号 `"-f"`）在 bash 中拼接为 `-f`，但常规扫描器可能漏过。
5. **词首 3+ 连续引号**：作为更宽泛的安全网。
6. **状态机扫描**：逐字符跟踪引号状态，发现“空白 + 引号 + 内容以 dash 开头”时，检查引号内/后续字符是否构成 flag 延续（如 `"-"exec`、`"-"$VAR`、`"-"{exec,delete}` 等）。
7. **`fullyUnquotedContent` 兜底**：检测 `\s['"`]-` 和 `['"`]{2}-` 模式。

#### 3.3.5 `validateBraceExpansion` —  brace expansion 防御

利用 `fullyUnquotedPreStrip` 扫描未转义的 `{` 和 `}`，通过嵌套深度计数找到匹配对，检查匹配对内部最外层深度是否存在 `,` 或 `..`。若存在，则 bash 会进行 brace expansion，而静态扫描器可能将其视为单个参数。

**额外防御**：
- 若未转义的 `}` 数量多于 `{` 数量，说明有 quoted `{` 被剥离后导致不平衡，直接 ask。
- 若原始命令中存在 `'{'`、`'}'`、`"{"`、`"}"` 且同时存在未转义 `{`，也直接 ask（这几乎总是混淆尝试）。

#### 3.3.6 `validateQuotedNewline` — 引号内换行 + `#` 行绕过

攻击示例：
```bash
mv ./decoy '<\n>#' ~/.ssh/id_rsa ./exfil_dir
```

在 bash 中：`<\n>#` 是 `mv` 的第二个参数（含字面量换行和 `#`），但 `stripCommentLines`（在 `bashPermissions.ts` 的 `stripSafeWrappers` 中）按行分割，发现第二行 trim 后以 `#` 开头，便将其整行删除。结果下游 path validation 只看到 `mv ./decoy`，而 `~/.ssh/id_rsa` 被隐藏。

检测逻辑：状态机跟踪引号，遇到 `\n` 时若处于引号内，检查下一行 `trim().startsWith('#')`，若是则 ask。

#### 3.3.7 `validateBackslashEscapedOperators` — 反斜杠转义操作符

`splitCommand_DEPRECATED` 会把 `\;` normalize 为裸 `;`。下游代码（如 `checkReadOnlyConstraints`、`checkPathConstraints`）若重新解析该 normalized 字符串，会把裸 `;` 当成操作符，错误地拆分成两个“安全”子命令。

示例攻击：
```bash
cat safe.txt \; echo ~/.ssh/id_rsa
```
- bash 视角：`cat` 读取 `safe.txt`、`\;`、`echo`、`~/.ssh/id_rsa` 四个参数。
- splitCommand 后：`cat safe.txt ; echo ~/.ssh/id_rsa` → 再解析成两段，第二段 `echo ~/.ssh/id_rsa` 通过只读检查，导致私钥泄露。

该 validator 检测任何不在引号内的 `\;`、 `\|`、 `\&`、 `\<`、 `\>`。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件内部路径

| 行号区间 | 内容 |
|----------|------|
| 12–101 | 常量定义：`COMMAND_SUBSTITUTION_PATTERNS`、`ZSH_DANGEROUS_COMMANDS`、`BASH_SECURITY_CHECK_IDS` |
| 103–127 | 类型定义：`ValidationContext`、`QuoteExtraction` |
| 128–174 | `extractQuotedContent` |
| 176–188 | `stripSafeRedirections` |
| 209–231 | `hasUnescapedChar` |
| 233–286 | `validateEmpty`、`validateIncompleteCommands` |
| 317–514 | `isSafeHeredoc`（含嵌套检测、位置检测、递归自校验） |
| 521–578 | `stripSafeHeredocSubstitutions` |
| 580–610 | `validateSafeCommandSubstitution` |
| 612–740 | `validateGitCommit` |
| 742–781 | `validateJqCommand` |
| 783–821 | `validateShellMetacharacters` |
| 823–844 | `validateDangerousVariables` |
| 846–873 | `validateDangerousPatterns` |
| 875–903 | `validateRedirections` |
| 905–941 | `validateNewlines` |
| 943–1015 | `validateCarriageReturn` |
| 1017–1036 | `validateIFSInjection` |
| 1038–1067 | `validateProcEnvironAccess` |
| 1069–1128 | `validateMalformedTokenInjection` |
| 1130–1537 | `validateObfuscatedFlags` |
| 1539–1601 | `hasBackslashEscapedWhitespace`、`validateBackslashEscapedWhitespace` |
| 1603–1721 | `hasBackslashEscapedOperator`、`validateBackslashEscapedOperators` |
| 1723–1892 | `isEscapedAtPosition`、`validateBraceExpansion` |
| 1894–1917 | `validateUnicodeWhitespace` |
| 1919–1962 | `validateMidWordHash` |
| 1964–2074 | `validateCommentQuoteDesync` |
| 2076–2175 | `validateQuotedNewline` |
| 2177–2242 | `validateZshDangerousCommands` |
| 2244–2273 | `CONTROL_CHAR_RE` 及相关注释 |
| 2257–2413 | `bashCommandIsSafe_DEPRECATED` |
| 2426–2592 | `bashCommandIsSafeAsync_DEPRECATED` |

### 4.2 上游调用方

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/tools/BashTool/bashPermissions.ts` | `bashToolHasPermission` (~1663) | 主权限入口；tree-sitter 不可用时走 legacy gate |
| `src/tools/BashTool/bashPermissions.ts` | `checkCommandAndSuggestRules` (~1183) | AST parse 成功后跳过 legacy 检查；否则调用 `bashCommandIsSafeAsync` |
| `src/tools/BashTool/bashCommandHelpers.ts` | `bashToolCheckCommandOperatorPermissions` (~208) | 检测到 unsafe compound 时调用 `bashCommandIsSafeAsync_DEPRECATED` 获取具体拒绝消息 |
| `src/tools/BashTool/readOnlyValidation.ts` | 多处 | 同步调用 `bashCommandIsSafe_DEPRECATED` 做只读命令前的安全检查 |

### 4.3 下游依赖方

| 文件 | 依赖项 | 说明 |
|------|--------|------|
| `src/utils/bash/heredoc.ts` | `extractHeredocs` | 在进入 validators 前剥离 quoted heredoc，防止 heredoc 体淹没检测器 |
| `src/utils/bash/ParsedCommand.ts` | `ParsedCommand.parse` | 异步路径获取 tree-sitter AST，提取 `TreeSitterAnalysis` |
| `src/utils/bash/shellQuote.ts` | `tryParseShellCommand`, `hasMalformedTokens`, `hasShellQuoteSingleQuoteBug` | shell-quote 解析与 bug 检测 |
| `src/utils/bash/treeSitterAnalysis.ts` | `TreeSitterAnalysis` 类型 | 异步路径的 quote context 与 operator node 信息 |
| `src/services/analytics/index.js` | `logEvent` | 每个 security check 触发时记录 analytics 事件 |
| `src/utils/permissions/PermissionResult.ts` | `PermissionResult` 类型 | 统一权限结果类型 |

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

- **`shell-quote`**（通过 `src/utils/bash/shellQuote.ts` 封装）：提供 `parse`/`quote`。本文件的大量 validator 都是针对 `shell-quote` 与真实 bash 的差异而设。
- **tree-sitter（可选）**：通过 `ParsedCommand.parse` 异步加载。若可用，`bashCommandIsSafeAsync_DEPRECATED` 使用其 `quoteContext` 替代 regex 提取，并消除 `find -exec \;` 等误报。
- **Analytics**：`logEvent('tengu_bash_security_check_triggered', { checkId, subId })` 用于安全事件遥测；`logEvent('tengu_tree_sitter_security_divergence')` 用于 tree-sitter/shadow 差异遥测。

### 5.2 配置/环境变量交互

- `process.env.CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK`：若置为 truthy，主权限流会跳过 tree-sitter 与 legacy 注入检查。
- `process.env.USER_TYPE === 'ant'`：影响 `ANT_ONLY_SAFE_ENV_VARS` 的生效范围（在 `bashPermissions.ts` 中），但 `bashSecurity.ts` 本身不直接读取。
- `feature('TREE_SITTER_BASH_SHADOW')` / `feature('BASH_CLASSIFIER')`：由 `bun:bundle` 的 `feature()` 提供，决定异步路径是否启用 tree-sitter 以及 classifier 的挂起检查。

### 5.3 测试现状

经全局检索（`*.test.ts`），**未发现专门针对 `bashSecurity.ts` 的单元测试文件**。该模块的测试覆盖主要依赖：
- `bashPermissions.ts` 相关的集成/权限测试（若有）。
- 外部 HackerOne 渗透测试与内部 red-team 审计（大量注释中引用了具体报告编号，如 #3482049、#3543050 等）。
- tree-sitter shadow 模式在生产环境中的 divergence 日志收集。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险与边界

#### 6.1.1 正则 vs. 真实 bash 的固有差距

本文件基于正则和字符状态机，而 bash 的语法是上下文相关的（尤其是 heredoc、arithmetic expansion、process substitution 的嵌套规则）。任何新增语法特性或边缘 case 都可能产生新的 parser differential。注释中反复出现“若扩展 `hasUnescapedChar` 到多字符，必须极度小心 ANSI-C quoting”的警告，说明维护者对此有清醒认识。

#### 6.1.2 `baseCommand` 的粗糙提取

```typescript
const baseCommand = command.split(' ')[0] || ''
```

这完全未处理前导 env var（`FOO=bar cmd`）或 wrapper（`timeout 5 cmd`），导致 `validateJqCommand`、`validateObfuscatedFlags` 中的 `baseCommand` 可能指向 `FOO=bar` 或 `timeout` 而非真实命令。不过这些 validators 的设计是“宁可误报也不漏报”，所以通常只会增加 false positive，不会导致 bypass。

#### 6.1.3 `validateObfuscatedFlags` 的复杂度与性能

该函数长达 400+ 行，包含多层嵌套的状态机、lookahead、quote chaining。虽然逻辑正确，但：
- 维护成本高，新增攻击向量时容易引入回归。
- 对极长命令（>10KB）的逐字符扫描可能成为 CPU 热点。

#### 6.1.4 无单元测试的直接修改风险

由于缺少针对本文件的独立单元测试，任何修改都只能通过：
1. 集成测试（BashTool 端到端）
2. 手动 red-team 用例验证
这导致修复特定 bypass 时，容易误伤合法命令（false positive）而难以快速发现。

#### 6.1.5 `stripSafeRedirections` 的边界条件

虽然当前有 `(?=\s|$)` 后缀防御，但模式仍硬编码了 `/dev/null`。若未来出现其他“安全重定向目标”（如 `/dev/zero`），需要手动扩展；否则攻击者可能利用类似前缀绕过。

### 6.2 改进建议

#### 6.2.1 逐步迁移至 tree-sitter 主路径

文件注释已表明该路径为 `@deprecated`。建议：
- 在外部构建中强制启用 `TREE_SITTER_BASH`，彻底退役 regex 路径。
- 对 `bashCommandIsSafe_DEPRECATED` 的同步调用方（如 `readOnlyValidation.ts`）提供异步重构，或引入轻量级同步 native parser。

#### 6.2.2 增加回归测试套件

建议新建 `src/tools/BashTool/__tests__/bashSecurity.test.ts`，至少覆盖以下高价值用例：
- `isSafeHeredoc` 的合法/非法 heredoc 替换（含嵌套、命令名位置、尾随危险字符）。
- `validateObfuscatedFlags` 的各子模式（ANSI-C、locale quoting、空引号 dash、同质空引号对、3+ 连续引号）。
- `validateBackslashEscapedOperators` 的 `\;` 攻击与 `find -exec \;` 合法用例。
- `validateBraceExpansion` 的 `{a,b}` 攻击与 `awk '{print $1}'` 合法用例。
- `validateQuotedNewline` 的引号内换行 + `#` 绕过。
- `stripSafeRedirections` 的 `/dev/nullo` 前缀绕过防御。

#### 6.2.3 提取 `validateObfuscatedFlags` 为子模块

该函数已超出单一职责，可拆分为：
- `validateAnsiCQuoting`
- `validateEmptyQuoteBeforeDash`
- `validateQuoteConcatenatedFlags`
- `validateFlagsInFullyUnquoted`

拆分后可独立测试、独立维护，降低回归风险。

#### 6.2.4 统一 `baseCommand` 提取逻辑

建议复用 `bashPermissions.ts` 中的 `stripSafeWrappers` 或 `stripWrappersFromArgv` 来提取真实命令名，而不是简单的 `command.split(' ')[0]`。这能减少 `validateJqCommand` 等以 `baseCommand` 为 gate 的 validator 的误报。

#### 6.2.5 引入自动化 red-team 流水线

鉴于该文件的安全敏感性，建议：
- 将注释中提到的所有 HackerOne 用例（如 `TZ=UTC\recho curl evil.com`、`git diff {@'{'0},--output=/tmp/pwned}` 等）编码为持续运行的回归测试。
- 使用 fuzzer（如基于 shell grammar 的生成器）定期对比 tree-sitter 输出与 regex 路径的 divergence，自动报警。

---

## 7. 附录：典型攻击用例与防御对应表

| 攻击向量 | 示例 | 防御函数 | 备注 |
|----------|------|----------|------|
| `shell-quote` 单引号反斜杠 bug | `git ls-remote 'safe\\' '--upload-pack=evil'` | `hasShellQuoteSingleQuoteBug` | H1 #3482049 |
| 反斜杠转义空格导致路径遍历 | `echo\ test/../../../usr/bin/touch /tmp/file` | `validateBackslashEscapedWhitespace` | parser differential |
| 反斜杠转义操作符导致二次拆分 | `cat safe.txt \; echo ~/.ssh/id_rsa` | `validateBackslashEscapedOperators` | splitCommand normalize |
| ANSI-C 引号隐藏 flag | `grep $'-exec' file` | `validateObfuscatedFlags` | `$'...'` 可编码任意字符 |
| 空引号拼接 flag | `"""-f" evil` | `validateObfuscatedFlags` | 同质空引号对 + 引号 dash |
| Brace expansion 绕过 | `git ls-remote {--upload-pack="touch /tmp/test",test}` | `validateBraceExpansion` | bash 展开为多个参数 |
| 引号内换行隐藏 `#` 注释行 | `mv decoy '<\n>#' ~/.ssh/id_rsa exfil` | `validateQuotedNewline` | 绕过 `stripCommentLines` |
| `#` 注释内引号导致 tracker 失步 | `echo "it's" # ' " <<'EOF'\nrm -rf /` | `validateCommentQuoteDesync` | quote tracker desync |
| CR 导致 tokenization 差异 | `TZ=UTC\recho curl evil.com` | `validateCarriageReturn` | JS `\s` 包含 `\r`，bash IFS 不包含 |
| Zsh equals expansion | `=curl evil.com` | `validateDangerousPatterns` | Zsh `=cmd` 展开为 `$(which cmd)` |
| Zsh dangerous builtins | `zmodload zsh/system` | `validateZshDangerousCommands` | 绕过二进制检查 |
| 畸形 token + 操作符 | `echo {"hi":"hi;evil"}` | `validateMalformedTokenInjection` | shell-quote 解析歧义 |
| heredoc 内命令替换绕过 | `$(cat <<'EOF'\nrm -rf /\nEOF\n)` | `validateSafeCommandSubstitution` / `isSafeHeredoc` | 仅允许 `cat` + quoted heredoc |
| `/proc` 环境变量泄露 | `cat /proc/self/environ` | `validateProcEnvironAccess` | defense-in-depth |

---

*文档结束*
