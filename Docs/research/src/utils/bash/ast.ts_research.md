# 研究文档：src/utils/bash/ast.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文。不包含 README / docs / Docs / markdown 等文档作为研究目标。

---

## 1. 场景与职责

`src/utils/bash/ast.ts` 是 **BashTool 安全体系的核心解析层**，负责将用户输入的 bash 命令字符串转化为可被下游权限系统信任的静态 `argv[]` 结构。它直接决定了哪些命令可以进入自动权限匹配流程，哪些命令必须回退到用户确认（`too-complex`）。

### 1.1 所处业务链路

完整的命令权限校验链路如下（以 `BashTool.tsx` 为入口）：

1. `src/tools/BashTool/BashTool.tsx` 接收用户/模型输入的 `command`。
2. 调用 `parseForSecurity(command)`（`ast.ts` 导出）。
3. 若返回 `simple`，则进入 `bashPermissions.ts` 的 `bashToolHasPermission`，继续调用 `checkSemantics(astResult.commands)` 做语义级风险扫描。
4. 语义与 AST 均通过后，`pathValidation.ts` 使用 `astResult.commands` 和 `astResult.commands[i].redirects` 做路径级约束校验（如禁止写入 `/proc/*/environ`、校验输出重定向路径等）。
5. 若 `parseForSecurity` 返回 `too-complex` 或 `parse-unavailable`，则回退到 legacy 的 `shell-quote` + 正则路径（`commands.ts`、`bashSecurity.ts`），最终同样走向用户确认或拒绝。

### 1.2 核心职责

- **结构化提取**：基于 AST（由 `bashParser.ts` 提供的纯 TypeScript 解析器生成）提取 `SimpleCommand[]`，包含 `argv`、`envVars`、`redirects` 和原始文本 `text`。
- **Fail-Closed 判定**：对任何无法静态分析的语言特性（如 `$()`、进程替换、子 shell、控制流、brace expansion 等）直接返回 `too-complex`，不尝试“聪明地”理解。
- **变量作用域追踪**：在同一条复合命令内追踪 `VAR=value` 赋值，以支持对后续 `$VAR` 的静态解析（如 `VAR=/tmp && rm $VAR`）。
- **语义后校验**：`checkSemantics` 在解析成功后对 `argv[0]` 及参数内容进行二次扫描，拦截 eval-like builtins、zsh 危险 builtins、`jq system()`、`/proc/*/environ` 访问等。

---

## 2. 功能点目的

### 2.1 `parseForSecurity` / `parseForSecurityFromAst`

| 功能 | 目的 |
|------|------|
| `parseForSecurity(cmd: string)` | 入口函数。先调用 `parseCommandRaw` 获取 AST，再进入 `parseForSecurityFromAst`。若 parser 不可用，返回 `parse-unavailable`。 |
| `parseForSecurityFromAst(cmd, root)` | 核心解析逻辑。对 AST 做前检查（pre-checks）后递归遍历，提取简单命令列表。返回 `simple` / `too-complex` / `parse-unavailable`。 |

**设计哲学**：
- 该模块**不是沙箱**，它不负责阻止危险命令运行；它只回答一个问题："我们能否为这条命令中的每个简单命令生成一个可信的 `argv[]`？"
- 如果答案是 yes，下游的 `bashPermissions.ts` 和 `pathValidation.ts` 才能基于 `argv[0]` 和参数做规则匹配与路径校验。

### 2.2 `checkSemantics`

在 AST 解析成功后运行，属于**后 argv 语义检查**（post-argv semantic checks）。主要目的：
- 剥离安全包装器（`time`、`nohup`、`timeout`、`nice`、`env`、`stdbuf`），让权限规则匹配到被包装的真实命令。
- 拦截 `eval`、`source`、`.`、`exec`、`trap`、`enable`、`hash` 等 eval-like builtins。
- 拦截 zsh 特有的危险 builtins（`zmodload`、`sysopen`、`zpty` 等）。
- 拦截 `jq` 的 `system()` 调用及危险文件读取 flag（`-f` / `--from-file` 等）。
- 拦截 `/proc/*/environ` 读取。
- 检测 `newline + #` 模式，防止 downstream 的 `stripSafeWrappers` 把 `#` 后的参数当成注释而漏检。

### 2.3 `nodeTypeId`

为 `too-complex` 的原因生成数字 ID，供 `logEvent` 上报使用（`analytics` 不接受字符串）。`DANGEROUS_TYPES` 的顺序即 ID 映射表。

---

## 3. 具体技术实现

### 3.1 解析器基础设施

`ast.ts` 本身**不包含词法/语法解析器**，它消费由 `src/utils/bash/parser.ts` 和 `src/utils/bash/bashParser.ts` 提供的 AST：

- `parser.ts` 中的 `parseCommandRaw` 是 `ast.ts` 的直接上游。它会调用 `bashParser.ts` 的 `getParserModule().parse(command)`。
- `bashParser.ts` 是一个**纯 TypeScript 实现的 bash 解析器**，生成与 `tree-sitter-bash` 兼容的 `TsNode` 结构（`type`、`text`、`startIndex`、`endIndex`、`children`）。
- 解析器有预算保护：`PARSE_TIMEOUT_MS = 50ms`，`MAX_NODES = 50_000`。若超时或超节点数，返回 `PARSE_ABORTED`（Symbol）。

### 3.2 关键数据结构

```ts
// ast.ts ~25-46
export type Redirect = {
  op: '>' | '>>' | '<' | '<<' | '>&' | '>|' | '<&' | '&>' | '&>>' | '<<<'
  target: string
  fd?: number
}

export type SimpleCommand = {
  argv: string[]
  envVars: { name: string; value: string }[]
  redirects: Redirect[]
  text: string
}

export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

### 3.3 前检查（Pre-checks）： parser differential 防御

在信任 AST 之前，`parseForSecurityFromAst` 先对原始字符串执行一系列正则检查（~404-437），这些检查针对的是 **tree-sitter 与 bash 实际行为不一致** 的已知差异点：

| 检查项 | 正则 / 逻辑 | 防御目标 |
|--------|-------------|----------|
| 控制字符 | `CONTROL_CHAR_RE = /[\x00-\x08\x0B-\x1F\x7F]/` | tree-sitter 可能将 CR 等视为分隔符，而 bash 的默认 IFS 不包含 CR，导致分词差异。 |
| Unicode 空白 | `UNICODE_WHITESPACE_RE` (NBSP, ZWSP, BOM 等) | 终端中不可见，但 bash 将其视为普通字符；tree-sitter 可能将其视为空白。 |
| 反斜杠转义空白 | `BACKSLASH_WHITESPACE_RE = /\\[ \t]|[^ \t\n\\]\\\n/` | `\ ` 与 `\<newline>` 在 tree-sitter 和 bash 中的处理不同，可能隐藏命令名。 |
| Zsh 动态目录 | `ZSH_TILDE_BRACKET_RE = /~\[/` | `~[name]` 在 zsh 中可执行任意代码。 |
| Zsh equals 扩展 | `ZSH_EQUALS_EXPANSION_RE = /(?:^|[\s;&\|])=[a-zA-Z_]/` | `=cmd` 在 zsh 中等价于 `$(which cmd)`，会绕过基于命令名的拒绝规则。 |
| Brace + Quote 混淆 | `BRACE_WITH_QUOTE_RE` + `maskBracesInQuotedContexts` | 防止 `{a'}',b}` 这类利用引号混淆 brace expansion 的攻击。 |

### 3.4 AST 遍历架构：Fail-Closed 的显式 Allowlist

遍历入口是 `walkProgram(root)` -> `collectCommands(node, commands, varScope)`。

`collectCommands` 是一个巨大的 switch/if-else 链，对每种 `node.type` 显式处理。任何未匹配的类型最终落入 `return tooComplex(node)`。

#### 3.4.1 结构节点（Structural Types）

```ts
const STRUCTURAL_TYPES = new Set([
  'program', 'list', 'pipeline', 'redirected_statement'
])
```

这些节点仅表示命令的组合关系，遍历器会递归进入子节点寻找真正的 `command` 叶子。

#### 3.4.2 作用域隔离（Scope Isolation）

`varScope: Map<string, string>` 用于追踪同一条命令字符串内的变量赋值。

**关键安全设计**（~504-564）：
- `&&` 和 `;` **保留**作用域：`VAR=x && cmd $VAR` 是顺序执行，`VAR` 对后续命令可见。
- `||`、`|`、`|&`、`&` **重置**作用域：
  - `||` 右侧可能不执行。
  - `|` 和 `&` 的各段在 subshell 中执行，变量赋值不会泄露到外部。
- 对于 `pipeline`，所有 stage 都在 subshell 中运行，因此遍历前直接复制一份 `varScope`。

这防止了著名的 **flag-omission 攻击**：`true || FLAG=--dry-run && cmd $FLAG`。如果线性传递作用域，解析器会认为 `FLAG` 被赋值为 `--dry-run`，从而生成 `['cmd', '--dry-run']` 的 argv，但 bash 实际执行时 `||` 右侧被跳过，`FLAG` 为空。

#### 3.4.3 支持的节点类型

`collectCommands` 显式支持的节点类型包括：

- `command`：普通命令，进入 `walkCommand`。
- `redirected_statement`：带重定向的命令，进入 `walkRedirectedStatement`。
- `declaration_command`：`export` / `declare` / `typeset` / `readonly` / `local`。会被解析为 `argv[0]='declare'` 等，并处理其内部的 `variable_assignment`。
- `variable_assignment`：裸赋值（如 `VAR=x`），不推入 `commands`，但更新 `varScope`。
- `for_statement`：提取循环体和迭代词，循环变量始终标记为 `VAR_PLACEHOLDER`（未知值），防止路径校验被绕过。
- `if_statement` / `while_statement`：条件命令和分支体分别提取。分支体使用 `varScope` 的**副本**，防止条件分支内的赋值泄露到分支外。
- `subshell`：`( ... )` 使用 `varScope` 副本，内部赋值不泄露。
- `test_command`：`[[ ... ]]` 或 `[ ... ]`，解析为 synthetic command `argv[0]='[['`。
- `unset_command`：`unset FOO`，同时从 `varScope` 中删除对应变量。
- `negated_command`：`! cmd`，仅反转退出码，不影响 argv。
- `comment`：忽略。

**危险节点类型**（`DANGEROUS_TYPES`，~186-205）：
包括 `command_substitution`（注意：bare arg 位置拒绝，但 string 内允许并递归提取）、`process_substitution`、`brace_expression`、`subshell`（注意 `subshell` 实际被允许，但 `DANGEROUS_TYPES` 包含它用于文档和 analytics）、`for_statement`（实际被允许）、各种控制流等。

> 注意：`DANGEROUS_TYPES` 并不 exhaustive，真正的安全属性在于 `collectCommands` / `walkArgument` 的 default branch 返回 `tooComplex`。

### 3.5 命令解析：`walkCommand`

`walkCommand` 遍历一个 `command` 节点的子节点，按顺序提取：

1. `variable_assignment` -> `envVars`（**不加入全局 `varScope`**，因为 env-prefix 赋值仅对当前命令有效）。
2. `command_name` -> `argv[0]`。
3. `word` / `number` / `raw_string` / `string` / `concatenation` / `arithmetic_expansion` -> `argv`。
4. `simple_expansion`（bare `$VAR`）-> 调用 `resolveSimpleExpansion`，根据 `varScope` 决定是替换为实际值、替换为 `VAR_PLACEHOLDER`，还是拒绝（`tooComplex`）。
5. `file_redirect` -> 调用 `walkFileRedirect`，提取到 `redirects`。
6. `herestring_redirect` -> 调用 `walkHerestringRedirect`，验证内容无扩展后丢弃（内容属于 stdin，不影响 argv）。

**`.text` 重建安全机制**（~1349-1358）：
如果 `node.text` 中包含 `$[A-Za-z_]`（说明有 simple_expansion 被解析并替换）或包含换行符，则 `walkCommand` 不会直接返回 `node.text`，而是用 `argv` 重建一个单引号转义后的命令文本。这防止了下游基于 `.text` 的 deny 规则匹配失败（如 `SUB=push && git $SUB --force`，原始 text 是 `git $SUB --force`， deny 规则 `git push:*` 不会匹配；重建后变成 `git 'push' --force`，规则即可命中）。

### 3.6 参数解析：`walkArgument`

`walkArgument` 是参数位置的 allowlist：

| 节点类型 | 处理方式 |
|----------|----------|
| `word` | 去除反斜杠转义（`\X -> X`），检查 brace expansion。 |
| `number` | **安全检查**：若 `node.children.length > 0`，拒绝。因为 `10#$(cmd)` 这种算术进制语法中，`number` 节点会包含 `command_substitution` 子节点，bash 会执行其中的命令。 |
| `raw_string` | 剥去外层单引号（`stripRawString`）。 |
| `string` | 进入 `walkString`。 |
| `concatenation` | 如 `"foo"bar`，递归解析各部分并拼接。检查 brace expansion。 |
| `arithmetic_expansion` | 进入 `walkArithmetic`。 |
| `simple_expansion` | 进入 `resolveSimpleExpansion`（视为 bare arg，非 string 内部）。 |

**注意**：bare arg 位置的 `command_substitution`（`$()`）**不被处理**，直接落入 `default -> tooComplex`。这是故意的：如果 `rm $(foo)` 被替换为 `__CMDSUB_OUTPUT__`，下游路径校验就看不到真实路径，会被绕过。

### 3.7 字符串解析：`walkString`

处理双引号字符串 `"..."` 的内部：

- `string_content`：去除双引号内的反斜杠转义（只转义 `$`、`` ` ``、`"`、`\`）。
- `command_substitution`：
  - 特殊 carve-out：`$(cat <<'DELIM'...DELIM)` 被识别为安全 heredoc 替换。若通过 `extractSafeCatHeredoc` 验证，则将其 heredoc body 内容作为字符串值拼接（单行的 body 直接拼接，多行的 body 丢弃以避免 `NEWLINE_HASH_RE` 误报）。
  - 一般情况：递归调用 `collectCommandSubstitution` 提取内部命令供权限校验，同时向结果字符串追加 `CMDSUB_PLACEHOLDER`。
- `simple_expansion`：`$VAR` 在双引号内允许被解析（`insideString=true`）。
- `arithmetic_expansion`：调用 `walkArithmetic` 验证后拼接原文字符串。

**Solo-Placeholder 安全门**（~1636-1638）：
如果字符串中只有 placeholder（如 `"$(cmd)"` 或 `"$UNKNOWN_VAR"`）而没有 literal content，则返回 `tooComplex`。原因是：placeholder 作为单个 argv 元素会被下游路径校验视为相对路径（如 `validatePath('__CMDSUB_OUTPUT__')` 解析为 cwd 下的文件），从而绕过敏感路径检查。

**Tree-sitter 换行丢失修复**（~1508-1652）：
Tree-sitter 在双引号内的换行符不会出现在 `string_content` 的 text 中（如 `"a\nb"` 变成两个 `string_content` 节点 `"a"` 和 `"b"`）。`walkString` 通过比较相邻子节点的 `startIndex` 间隙，自动插入 `\n` 来补偿，确保 argv 与 bash 实际行为一致。

### 3.8 算术扩展校验：`walkArithmetic`

只允许纯数字字面量和运算符：

```ts
const ARITH_LEAF_RE =
  /^(?:[0-9]+|0[xX][0-9a-fA-F]+|[0-9]+#[0-9a-zA-Z]+|[-+*/%^&|~!<>=?:(),]+|<<|>>|\*\*|&&|\|\||[<>=!]=|\$\(\(|\)\))$/
```

**为什么拒绝变量？**
因为 bash 的算术求值会递归解析变量值。如果 `x='a[$(cmd)]'`，则 `$((x))` 会在运行时执行 `cmd`。tree-sitter 将 `x` 视为 `variable_name` 叶子节点，无法发现其中的 payload。

### 3.9 变量赋值解析：`walkVariableAssignment`

处理 `VAR=value` 和 `VAR+=value`：

- 支持 `command_substitution` 作为 value：`VAR=$(date)` -> value 设为 `CMDSUB_PLACEHOLDER`，同时内部 `date` 命令被提取到 `commands` 供权限校验。
- 支持 `simple_expansion` 作为 value：`VAR=$OTHER` -> 按 string 内规则解析。
- **安全检查**：
  - 变量名必须符合 `[A-Za-z_][A-Za-z0-9_]*`，否则 bash 会将其视为命令执行，而非赋值。
  - `IFS` 赋值直接拒绝（改变 word-splitting 行为，静态模型无法追踪）。
  - `PS4` 赋值有严格的 allowlist 字符集检查（防止 `set -x` 时的 trace-time RCE）。
  - value 中不能包含 `~`（bash 的 tilde expansion 在赋值时发生，静态模型无法追踪）。
  - `declare` / `typeset` / `local` 的 `-n`（nameref）、`-i`（integer）、`-a`/`-A`（array）flag 被拒绝；带数组下标的赋值（如 `declare 'x[$(id)]=val'`）也被拒绝。

### 3.10 变量解析：`resolveSimpleExpansion`

处理 `$VAR` 节点：

- **Tracked vars**：若 `varScope` 中有该变量：
  - 若值包含任何 placeholder（`CMDSUB_PLACEHOLDER` 或 `VAR_PLACEHOLDER`），则：
    - bare arg -> `tooComplex`
    - inside string -> `VAR_PLACEHOLDER`
  - 若值为纯字面量：
    - bare arg -> 检查是否为空字符串或包含 IFS/glob 字符（`BARE_VAR_UNSAFE_RE = /[ \t\n*?[]/`），若包含则 `tooComplex`。
    - inside string -> 直接返回实际值。
- **Safe env vars**：`HOME`、`PWD`、`PATH`、`USER` 等（`SAFE_ENV_VARS`，~125-149）仅在 `insideString=true` 时返回 `VAR_PLACEHOLDER`；bare arg 位置拒绝。
- **Special vars**：`$?`、`$$`、`$!`、`$#`、`$0`-`$9`、`$-` 等，同样仅允许在 string 内。
- **`$@` / `$*`**：显式**不在**允许列表中。因为 BashTool 总是在全新 shell 中执行，它们为空，但返回 placeholder 会导致 argv 重建错误（`git "push$*"` 变成 `git push__TRACKED_VAR__`，与 bash 实际行为不符）。

### 3.11 命令替换递归：`collectCommandSubstitution`

对 `$()` 或 `` `...` `` 内部的命令递归调用 `collectCommands`。

- 传入 `varScope` 的**副本**：内部赋值不泄露到外部。
- 提取出的内部命令被追加到 `innerCommands`（最终合并到 `commands`），这意味着权限系统会分别对外层命令和内层命令做规则匹配。例如 `echo $(git rev-parse HEAD)` 会提取出 `echo $(git rev-parse HEAD)` 和 `git rev-parse HEAD` 两个命令，两者都必须匹配权限规则。

### 3.12 重定向解析

#### `walkFileRedirect`
处理 `file_redirect` 节点，提取 `op`、`target`、`fd`。
- target 可以是 `word`、`number`（需检查无子节点）、`raw_string`、`string`、`concatenation`。
- 对 `word` 和 `number` 检查 `BRACE_EXPANSION_RE`。
- 对 `word` 做反斜杠去除（`\(.) -> $1`），防止 `\environ` 绕过 `PROC_ENVIRON_RE`。

#### `walkHeredocRedirect`
处理 heredoc（`<<` / `<<-`）。
- **仅允许 quoted delimiter**（`<<'EOF'` 或 `<<\EOF`）。unquoted delimiter 直接 `tooComplex`，因为 bash 会对其 body 做参数/命令/算术扩展，且 tree-sitter 对 heredoc 内的 backtick 替换存在 grammar gap（不会解析为 `command_substitution` 子节点）。
- body 内只允许 `heredoc_content` 子节点。
- 若 heredoc 后紧跟 pipeline 等结构（如 `ls <<'EOF' | rm x`），tree-sitter 会将这些结构作为 `heredoc_redirect` 的子节点；`walkHeredocRedirect` 的 default branch 会 `tooComplex`，防止隐藏管道后的命令。

#### `walkHerestringRedirect`
处理 `<<< content`。
- 用 `walkArgument` 验证 content 是静态字面量（无扩展）。
- 验证结果字符串被丢弃（因为 herestring 是 stdin），但仍需检查 `NEWLINE_HASH_RE`。

### 3.13 语义检查：`checkSemantics`

`checkSemantics(commands: SimpleCommand[])` 在 `parseForSecurity` 返回 `simple` 后执行。

#### 3.13.1 安全包装器剥离

对每条 `SimpleCommand` 的 `argv` 循环剥离以下包装器，直到露出真实命令：

- `time`、`nohup`：直接 `slice(1)`。
- `timeout`：解析 GNU 风格的 flag（`--foreground`、`-k`、`-s`、`--kill-after` 等），跳过 duration（`/^\d+(?:\.\d+)?[smhd]?$/`），然后 `slice(i+1)`。对任何未知 flag 或无法识别的 duration（如 `.5`、`inf`）**fail-closed**。
- `nice`：支持 `nice -n N cmd` 和 `nice -N cmd`（legacy）。
- `env`：跳过 `VAR=val`、`-i`、`-0`、`-v`、`-u NAME`。对 `-S`（argv splitter）、`-C`（altwd）、`-P`（altpath）或任何未知 flag **fail-closed**。
- `stdbuf`：跳过 `-i`/`-o`/`-e` 的 short/long/fused 形式。未知 flag **fail-closed**。

**重要**：`pathValidation.ts` 中的 `stripWrappersFromArgv` 必须与 `checkSemantics` 保持同步，否则会出现 `checkSemantics` 已暴露真实命令名但路径校验仍看到包装器名、从而 passthrough 绕过路径检查的漏洞（PR #21503 修复过此类问题）。

#### 3.13.2 内置命令拦截

- **Zsh dangerous builtins**：`zmodload`、`emulate`、`sysopen`、`zpty` 等（`ZSH_DANGEROUS_BUILTINS`）。
- **Eval-like builtins**：`eval`、`source`、`.`、`exec`、`command`、`builtin`、`fc`、`coproc`、`noglob`、`nocorrect`、`trap`、`enable`、`mapfile`、`readarray`、`hash`、`bind`、`complete`、`compgen`、`alias`、`let`。
  - 部分 carve-outs：
    - `command -v` / `command -V` 允许（仅打印路径）。
    - `fc` 无 `-e`/`-s` flag 时允许（如 `fc -l` 仅列出历史）。
    - `compgen` 无 `-C`/`-F`/`-W` 时允许。

#### 3.13.3 数组下标求值攻击

bash 在多个 builtin 的 `NAME` 参数位置会重新解析并算术求值数组下标，即使该参数来自单引号的 `raw_string`：

- `test -v 'a[$(id)]'` -> 执行 `id`。
- `printf -v 'a[$(id)]' ...` -> 执行 `id`。
- `wait -p 'a[$(id)]' %1` -> 执行 `id`（bash 5.1+）。
- `[[ 'a[$(id)]' -eq 0 ]]` -> 执行 `id`（因为 `-eq` 触发算术求值）。

`checkSemantics` 通过以下机制防御：
- `SUBSCRIPT_EVAL_FLAGS`：记录哪些 builtin 的哪些 flag 的下一个参数是 NAME，并检查该参数是否包含 `[`。
- `TEST_ARITH_CMP_OPS`：对 `[[ ... ]]` 中的 `-eq`、`-ne`、`-lt` 等操作符，检查两侧操作数是否包含 `[`。
- `BARE_SUBSCRIPT_NAME_BUILTINS`：`read` 和 `unset` 的每个 bare positional 参数都被视为 NAME，检查是否包含 `[`。

#### 3.13.4 其他语义检查

- **Shell keywords as argv[0]**：若 `argv[0]` 是 `if`/`then`/`for` 等关键字，说明 tree-sitter 发生了 mis-parse（如 `! for i in a; do :; done` 会被错误解析为多个 `command` 节点），直接拒绝。
- **Empty / Placeholder command name**：`argv[0]` 为空字符串或包含 placeholder 时拒绝。
- **Fragment detection**：`argv[0]` 以 `-`、`|`、`&` 开头时拒绝。
- **`jq` 检查**：检测 `system(` 及危险 flag（`-f`、`--from-file`、`-L` 等）。
- **`/proc/*/environ` 检查**：在 `argv` 和 `redirects` 的目标中检查。
- **`NEWLINE_HASH_RE`**：在 `argv`、`envVars.value`、`redirects.target` 中检查 `\n[ \t]*#`，防止 downstream 的 `stripSafeWrappers` 按行处理时将 `#` 后内容误认为注释而跳过校验。

---

## 4. 关键代码路径与文件引用

### 4.1 入口与主调用链

```
BashTool.tsx:451
  └── parseForSecurity(command)                       [ast.ts:381]
        └── parseCommandRaw(cmd)                        [parser.ts:104]
              └── getParserModule().parse(command)      [bashParser.ts:44]
        └── parseForSecurityFromAst(cmd, root)          [ast.ts:400]
              └── walkProgram(root)                     [ast.ts:462]
                    └── collectCommands(...)            [ast.ts:482]
                          ├── walkCommand(...)          [ast.ts:1237]
                          │     ├── walkArgument(...)   [ast.ts:1399]
                          │     ├── walkFileRedirect    [ast.ts:1067]
                          │     └── walkHerestringRedirect [ast.ts:1211]
                          ├── walkRedirectedStatement   [ast.ts:1017]
                          ├── walkVariableAssignment    [ast.ts:1777]
                          └── collectCommandSubstitution [ast.ts:1374]

bashPermissions.ts:1686
  └── parseForSecurityFromAst(input.command, astRoot)   [ast.ts:400]
  └── checkSemantics(astResult.commands)                [ast.ts:2213]

pathValidation.ts:1077
  └── validateSinglePathCommandArgv(cmd, ...)           [pathValidation.ts]
        └── stripWrappersFromArgv(cmd.argv)             [pathValidation.ts:1263]
```

### 4.2 核心安全常量位置

| 常量 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `STRUCTURAL_TYPES` | `ast.ts` | ~54 | 允许递归的结构节点类型。 |
| `SEPARATOR_TYPES` | `ast.ts` | ~65 | 命令分隔符（`&&`、`\|\|`、`\|`、`;`、`&`、`|&`、`\n`）。 |
| `CMDSUB_PLACEHOLDER` | `ast.ts` | ~74 | `$()` 输出的占位符。 |
| `VAR_PLACEHOLDER` | `ast.ts` | ~82 | 未知值变量的占位符。 |
| `SAFE_ENV_VARS` | `ast.ts` | ~125 | 已知安全的环境变量白名单。 |
| `SPECIAL_VAR_NAMES` | `ast.ts` | ~167 | `$?`、`$$` 等特殊变量白名单。 |
| `DANGEROUS_TYPES` | `ast.ts` | ~186 | 用于 analytics 的危险类型列表。 |
| `REDIRECT_OPS` | `ast.ts` | ~224 | 重定向操作符映射表。 |
| `ZSH_DANGEROUS_BUILTINS` | `ast.ts` | ~2060 | zsh 危险内置命令。 |
| `EVAL_LIKE_BUILTINS` | `ast.ts` | ~2086 | eval-like 内置命令。 |
| `SUBSCRIPT_EVAL_FLAGS` | `ast.ts` | ~2143 | 会触发数组下标求值的 builtin+flag 组合。 |
| `TEST_ARITH_CMP_OPS` | `ast.ts` | ~2169 | `[[ ]]` 中的算术比较操作符。 |
| `BARE_SUBSCRIPT_NAME_BUILTINS` | `ast.ts` | ~2182 | 所有 positional 参数都是 NAME 的 builtin。 |
| `PROC_ENVIRON_RE` | `ast.ts` | ~2197 | `/proc/*/environ` 访问检测。 |
| `NEWLINE_HASH_RE` | `ast.ts` | ~2204 | 换行后接注释的攻击模式检测。 |

### 4.3 相关文件及其角色

| 文件 | 角色 |
|------|------|
| `src/utils/bash/parser.ts` | `ast.ts` 的直接上游。提供 `parseCommandRaw`、`PARSE_ABORTED`、`Node` 类型。负责调用 `bashParser.ts` 并处理超时/节点预算。 |
| `src/utils/bash/bashParser.ts` | 纯 TypeScript 的 bash 解析器，生成 `TsNode` AST。提供 `SHELL_KEYWORDS`、`ensureParserInitialized`、`getParserModule`。 |
| `src/tools/BashTool/BashTool.tsx` | `parseForSecurity` 的调用入口。在命令执行前做权限校验。 |
| `src/tools/BashTool/bashPermissions.ts` | 权限 orchestrator。调用 `parseForSecurityFromAst` 和 `checkSemantics`，并根据结果决定 allow / ask / deny。 |
| `src/tools/BashTool/pathValidation.ts` | 路径约束校验。消费 `SimpleCommand[]` 和 `Redirect[]`。包含 `stripWrappersFromArgv`，必须与 `checkSemantics` 的包装器剥离逻辑保持同步。 |
| `src/tools/BashTool/bashSecurity.ts` | Legacy 安全校验（基于正则和 `shell-quote`）。在 `parse-unavailable` 或某些早期 allow 路径时作为 fallback。 |
| `src/utils/bash/commands.ts` | Legacy 命令拆分工具（`splitCommand_DEPRECATED`、`extractOutputRedirections`）。在 tree-sitter 不可用时被调用。 |
| `src/utils/bash/ParsedCommand.ts` | 基于 AST 构建 `IParsedCommand` 对象，供其他模块（如 pipe segment 提取、重定向提取）使用。 |

---

## 5. 依赖与外部交互

### 5.1 导入依赖

```ts
import { SHELL_KEYWORDS } from './bashParser.js'
import type { Node } from './parser.js'
import { PARSE_ABORTED, parseCommandRaw } from './parser.js'
```

- `bashParser.js`：提供 `SHELL_KEYWORDS`（用于 `checkSemantics` 中检测 shell keyword 作为 `argv[0]` 的 mis-parse）。
- `parser.js`：提供 AST 节点类型别名 `Node`（实际为 `TsNode`），以及解析入口 `parseCommandRaw` 和 `PARSE_ABORTED` sentinel。

### 5.2 无外部运行时依赖

`ast.ts` 本身不依赖任何 npm 包或系统命令。它不直接操作文件系统、网络或数据库。所有逻辑均为纯函数（除 `logEvent` 在 `bashPermissions.ts` 中调用外，`ast.ts` 内无 side effect）。

### 5.3 与权限系统的数据契约

`ast.ts` 与 `bashPermissions.ts` 之间存在强数据契约：

1. `parseForSecurityFromAst` 返回的 `SimpleCommand.text` 必须能被 downstream 的 `stripSafeWrappers`（基于正则，按行处理）正确解析。因此 `walkCommand` 在检测到 `$VAR` 替换或换行符时会**重建 `.text`**，用单引号包裹每个参数。
2. `checkSemantics` 剥离包装器后的 `argv` 结构，必须与 `pathValidation.ts` 中的 `stripWrappersFromArgv` 一致。历史上有多次因不一致导致的路径校验绕过（如 `stdbuf -o0 -eL rm` 在 `checkSemantics` 中暴露为 `rm`，但 `stripWrappersFromArgv` 未正确剥离，导致 `pathValidation` 看到 `stdbuf` 而 passthrough）。
3. `nodeTypeId` 的返回值被 `bashPermissions.ts` 用于 `logEvent` 上报 `too-complex` 的原因分布。

### 5.4 与解析器的版本耦合

`ast.ts` 深度依赖 `bashParser.ts` 生成的 AST 节点类型名称（如 `command_name`、`variable_assignment`、`file_redirect`、`heredoc_redirect`、`simple_expansion`、`command_substitution`、`arithmetic_expansion` 等）。如果 `bashParser.ts` 的语法树结构发生变化（例如新增中间节点、改变 `redirected_statement` 的子节点顺序），`ast.ts` 的遍历逻辑可能产生 mis-match，导致本应 `simple` 的命令被误判为 `too-complex`，或更危险地——漏过某些节点。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 Parser Differential 的持续军备竞赛

`ast.ts` 的前检查（pre-checks）和 `walkArgument` 中的各种 workaround 本质上是在填补 **tree-sitter-bash（或本项目的纯 TS 复刻版）与真实 bash/zsh 之间的语义鸿沟**。这是一个典型的“打地鼠”模型：

- 已修复的 differential 包括：CR 字符分词差异、Unicode 空白、反斜杠转义空白、zsh `~[` 和 `=cmd`、brace expansion 与 quote 的混淆、`number` 节点内的 `NN#$(cmd)` 注入、heredoc 内的 backtick 执行等。
- **风险**：新的 differential 可能随时被发现。例如 bash 的 locale-specific word splitting、ANSI-C quoting（`$'...'`）的转义差异、或更隐蔽的 arithmetic expansion 注入向量。

#### 6.1.2 `checkSemantics` 与 `pathValidation.ts` 的同步风险

`checkSemantics` 和 `pathValidation.ts` 中存在**两份几乎相同但又不完全相同的包装器剥离逻辑**：

- `checkSemantics` 中的 `timeout`/`nice`/`env`/`stdbuf` 解析（`ast.ts` ~2220-2380）。
- `pathValidation.ts` 中的 `skipTimeoutFlags` / `skipStdbufFlags` / `skipEnvFlags` / `stripWrappersFromArgv`（`pathValidation.ts` ~1183-1303）。

历史漏洞（PR #21503）已经证明，当 `checkSemantics` 剥离了包装器暴露出真实命令名，但 `pathValidation` 没有同步剥离时，会导致**路径校验完全跳过**（因为 `baseCmd` 是包装器名，不在 `SUPPORTED_PATH_COMMANDS` 列表中，直接 passthrough）。

**当前状态**：代码中有大量注释强调“KEEP IN SYNC”，但人工维护两份逻辑仍然是高风险。

#### 6.1.3 Bun DCE（Dead Code Elimination）复杂性悬崖

`bashPermissions.ts` 被注释明确标记为处于 **Bun `feature()` DCE 的复杂性阈值边缘**（`bashPermissions.ts:81-87`）。

- 在该文件中，连 `import { X as Y }` 的别名都会推高复杂度，导致 Bun 无法将 `feature('BASH_CLASSIFIER')` 证明为常量，从而将其静默求值为 `false`，丢弃所有 `pendingClassifierCheck` 逻辑。
- 这导致 `pathValidation.ts` 中不得不保留一份**死代码**（`stripWrappersFromArgv` 的旧版拷贝无法删除），因为删除 `bashPermissions.ts` 中的对应代码会触发 DCE 悬崖，破坏 classifier 功能。
- **风险**：任何对 `bashPermissions.ts` 的大规模重构（例如将 `checkSemantics` 的部分逻辑移回 `bashPermissions.ts` 以消除重复）都可能意外触发 DCE 回归。

#### 6.1.4 缺乏单元测试

在仓库中**未找到**针对 `ast.ts` 的 `.test.ts` 或 `.spec.ts` 文件。`parseForSecurity` 和 `checkSemantics` 的众多边界条件（如 `for` 循环作用域隔离、`if` 分支作用域副本、`PS4` allowlist、数组下标求值拦截、各种 wrapper stripping 的 corner cases）目前似乎仅通过集成测试或生产流量验证。

**风险**：
- 新 PR 引入回归时难以在 CI 阶段捕获。
- 安全相关的边界条件（如 PR #21503 的 `timeout .5 eval` 绕过）缺乏自动化回归防护。

#### 6.1.5 `walkCommand` 的 `.text` 重建盲区

`walkCommand` 在以下两种情况下会重建 `.text`：
1. `node.text` 包含 `$[A-Za-z_]`（说明有 simple_expansion 被解析）。
2. `node.text` 包含换行符。

**盲区**：如果 `node.text` 中不包含 `$[A-Za-z_]`（例如变量名以数字开头，或被其他字符包裹导致正则未命中），但 `argv` 实际上已被替换，则 `.text` 与 `argv` 不一致，下游基于 `.text` 的 deny 规则可能漏匹配。

不过，变量名以数字开头的情况会在 `walkVariableAssignment` 中因 `Invalid variable name` 被拒绝，因此该盲区的实际可利用性较低。

### 6.2 边界与限制

#### 6.2.1 不支持动态内容作为 bare 参数

任何会导致参数值在运行时才能确定的内容，在 bare arg 位置都会被拒绝：
- `$VAR`（除非 `VAR` 是同命令内赋值的纯静态字面量）。
- `$(cmd)`（bare arg 位置）。
- `${...}` 参数扩展（任何位置）。
- `~` tilde expansion（赋值中直接拒绝；bare arg 中由 `BARE_VAR_UNSAFE_RE` 或路径校验处理）。
- brace expansion（`{a,b}`）。

这意味着大量合法的 bash 用法会被标记为 `too-complex` 并走向用户确认。这是设计上的 trade-off：牺牲自动化覆盖率以换取静态分析的可信度。

#### 6.2.2 `for` / `while` / `if` 的循环变量永远视为未知

`for` 循环的循环变量（如 `$i`）在 body 中始终被设为 `VAR_PLACEHOLDER`。即使迭代列表是静态的（如 `for i in /etc/passwd; do cat $i; done`），也不会尝试展开。这导致 `for` 循环内的路径敏感命令（如 `cat`、`rm`）如果使用了循环变量，几乎总是 `too-complex`。

#### 6.2.3 `command_substitution` 在 string 内的 placeholder 限制

`"$(cmd)"` 这种 solo-placeholder 字符串会被拒绝。但 `"prefix$(cmd)"` 允许。这导致像 `cd "$(pwd)"` 这样的常见用法也会被拒绝（因为整个字符串只有 placeholder）。用户必须改写为不使用 `$()` 的形式，或接受手动确认。

### 6.3 改进建议

#### 6.3.1 提取共享的包装器剥离逻辑

将 `timeout`/`nice`/`env`/`stdbuf` 的 flag 解析逻辑提取到一个独立的纯函数模块（如 `src/utils/bash/wrapperStripping.ts`），同时被 `checkSemantics` 和 `pathValidation.ts` 消费。

**挑战**：需要验证该提取不会将 `bashPermissions.ts` 的复杂度推到 Bun DCE 悬崖以下。可以考虑将该模块放在 `pathValidation.ts` 旁边，由 `pathValidation.ts` 导出相关函数，然后 `bashPermissions.ts` 通过间接方式使用（如果直接 import 会触发 DCE 问题）。

#### 6.3.2 增加 `ast.ts` 的单元测试套件

建议新增 `src/utils/bash/ast.test.ts`（或放在 `src/tools/testing` 下），覆盖以下场景：

- **作用域隔离**：`VAR=x && cmd $VAR` vs `VAR=x || cmd $VAR` vs `VAR=x | cmd $VAR`。
- **包装器剥离**：`timeout -k 5 10s eval ...`、`stdbuf -o0 -eL rm ...`、`env -i VAR=val cmd ...`。
- **危险模式拦截**：`test -v 'a[$(id)]'`、`printf -v 'a[$(id)]'`、`[[ 'a[$(id)]' -eq 0 ]]`、`read 'a[$(id)]'`。
- **重定向安全**：`> /proc/self/environ`、`> {a,b}`、`> $(mktemp)`、`<<EOF`（unquoted heredoc）。
- **变量追踪**：`VAR=/tmp && rm $VAR`（应解析为 `['rm', '/tmp']`）vs `VAR=$(date) && rm $VAR`（应 too-complex）。
- **`.text` 重建**：`SUB=push && git $SUB --force` 的 `.text` 应为 `git 'push' --force`。
- **Solo placeholder**：`"$(cmd)"` 应 too-complex，`"prefix$(cmd)"` 应 simple。

#### 6.3.3 引入 AST-based 的 `ParsedCommand` 统一消费

目前 `pathValidation.ts` 同时消费 `SimpleCommand[]`（AST 路径）和 `splitCommand_DEPRECATED`（legacy 路径）。可以进一步将路径校验完全迁移到 `SimpleCommand` + `Redirect` 的结构化数据上，逐步淘汰 `splitCommand_DEPRECATED` 在路径校验中的使用，减少 legacy 代码的维护面。

#### 6.3.4 对 `bashParser.ts` 的变更增加契约测试

由于 `ast.ts` 与 `bashParser.ts` 的 AST 结构强耦合，建议为 `bashParser.ts` 增加**快照测试（snapshot tests）**或**契约测试**：对一组代表性的 bash 命令，断言其生成的 AST 节点类型和父子关系不变。这样可以在 `bashParser.ts` 升级或重构时，第一时间发现对 `ast.ts` 的破坏性变更。

#### 6.3.5 考虑对 `for` 循环的有限展开

对于迭代列表完全静态且无 brace expansion / glob 的 `for` 循环（如 `for i in foo bar; do echo $i; done`），可以考虑在解析阶段将 body 展开为多个 `SimpleCommand` 实例（`echo foo`、`echo bar`），从而避免将循环变量视为完全未知。但这会显著增加复杂度，需要谨慎评估安全性。

---

## 7. 总结

`src/utils/bash/ast.ts` 是 BashTool 安全体系中最关键、也最复杂的模块之一。它以 **Fail-Closed** 和 **显式 Allowlist** 为核心设计原则，通过纯 TypeScript 解析器生成的 AST，将 bash 命令转化为可信的 `argv[]` 结构，供下游权限和路径校验系统使用。

该模块的成功依赖于对 bash/zsh 语义的深入理解，以及对 parser differential 的持续修补。其最大的维护风险在于：
1. 与 `pathValidation.ts` 的包装器剥离逻辑同步。
2. 与 `bashParser.ts` 的 AST 结构耦合。
3. 缺乏独立的单元测试回归防护。

未来的改进应优先解决**逻辑重复**和**测试覆盖**问题，同时警惕 Bun DCE 复杂性悬崖对重构的约束。
