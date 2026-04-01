# 研究文档：src/utils/bash/bashParser.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文。不涉及 README/docs/Docs/markdown 等文档作为研究目标。
> 研究日期：2026-04-01
> 执行器：kimi（model=k2p5）

---

## 1. 场景与职责

`src/utils/bash/bashParser.ts` 是一个**纯 TypeScript 实现的 Bash 语法解析器**，其核心职责是：

1. **替代/兜底原生 tree-sitter WASM/NAPI 解析器**：在无法加载原生 tree-sitter-bash 模块时，提供零依赖的纯 JS/TS 解析能力，输出与 tree-sitter-bash 完全兼容的 AST（`TsNode`）。
2. **为安全分析提供结构化 AST**：下游的 `ast.ts`（安全 walker）、`ParsedCommand.ts`（命令分段/重定向提取）、`prefix.ts`（命令前缀提取）都依赖该 AST 进行权限判定、路径校验和命令分类。
3. **防御 adversarial 输入**：通过硬超时（50ms）和节点预算（50,000 节点）防止路径ological 输入导致解析挂起或 OOM。

### 1.1 在系统中的位置

```
BashTool 输入字符串
    │
    ▼
src/utils/bash/parser.ts  ┌─────────────────────────────────────┐
    │                     │  feature('TREE_SITTER_BASH')        │
    │                     │  为真时尝试加载原生 WASM/NAPI 解析器 │
    │                     │  为假时直接 fallback 到本文件       │
    ▼                     └─────────────────────────────────────┘
src/utils/bash/bashParser.ts  ← 本文件（纯 TS 解析器）
    │
    ├──► ast.ts              安全 walker（parseForSecurity）
    ├──► ParsedCommand.ts    管道分段、重定向提取
    ├──► prefix.ts           命令前缀静态提取
    └──► treeSitterAnalysis.ts  复合结构/危险模式/引号上下文分析
```

### 1.2 关键设计约束

- **AST 兼容性**：所有节点类型、字段名、`startIndex`/`endIndex`（UTF-8 字节偏移）必须与 tree-sitter-bash 保持一致，否则下游 walker 会崩溃或产生错误的安全判定。
- **Fail-closed**：解析超时或节点超限返回 `null`，`parser.ts` 将其映射为 `PARSE_ABORTED`，强制走 `too-complex` 路径（即提示用户手动确认）。
- **零异步初始化**：`ensureParserInitialized()` 直接返回 `Promise.resolve()`，与原生解析器的异步 WASM 初始化保持 API 兼容。

---

## 2. 功能点目的

### 2.1 主要导出符号

| 导出符号 | 类型 | 目的 |
|---------|------|------|
| `TsNode` | `type` | AST 节点接口：`{ type, text, startIndex, endIndex, children }` |
| `ensureParserInitialized` | `function` | 异步初始化兼容层（本实现为 no-op） |
| `getParserModule` | `function` | 返回 `{ parse }` 模块对象，供 `parser.ts` 调用 |
| `SHELL_KEYWORDS` | `Set<string>` | 供下游 `ast.ts` 等引用，判断保留字 |

### 2.2 解析器能力覆盖

本文件实现了一个接近完整的 Bash 词法/语法解析器，支持：

- **词法层**：WORD、NUMBER、OP（重定向/控制/逻辑运算符）、NEWLINE、COMMENT、DQUOTE、SQUOTE、ANSI_C、DOLLAR 系列（`$((`/`$(`/`${`/`$'`/`$`）、BACKTICK、LT_PAREN/GT_PAREN（进程替换）。
- **语句层**：`program` → `statement` 序列，支持 `;` / `&` / `&&` / `||` / `|` / `|&` / newline 分隔。
- **命令层**：simple command、pipeline、list、redirected_statement、negated_command、subshell、compound_statement（`{...}` / `((...))`）。
- **控制流**：`if`/`while`/`until`/`for`（含 C-style for）/`case`/`function definition`。
- **展开式**：`$VAR`、`${...}`（含参数替换运算符 `# ## % %% / // ^ ^^ , ,, :- := :? :+ 等）、算术展开 `$((...))` / `$[...]`、命令替换 `$(...)` / `` `...` ``、进程替换 `<(...)`/`>(...)`。
- **测试命令**：`[ ... ]` / `[[ ... ]]`，含 unary/binary/parenthesized/negated/regex/extglob 表达式。
- **Heredoc**：`<<` / `<<-`，支持引号定界符（决定 body 是否做变量展开），body 在下一个 newline 处延迟扫描。
- **重定向**：`>` / `>>` / `>&` / `>|` / `&>` / `&>>` / `<` / `<&` / `<<` / `<<-` / `<<<` / `<&-` / `>&-` / 文件描述符前缀（如 `2>`）。

---

## 3. 具体技术实现

### 3.1 核心数据结构

#### 3.1.1 TsNode（AST 节点）

```ts
export type TsNode = {
  type: string
  text: string
  startIndex: number   // UTF-8 byte offset（非 JS char index）
  endIndex: number
  children: TsNode[]
}
```

**关键约束**：`startIndex`/`endIndex` 是 **UTF-8 字节偏移**，因为 tree-sitter 原生输出使用字节偏移。对于包含多字节字符（如中文、Emoji）的命令字符串，JS `String.slice()` 不能直接按这些偏移切片。文件内实现了 `byteAt()` / `sliceBytes()` / `restoreLexToByte()` 来完成字节偏移 ↔ 字符索引的转换。

#### 3.1.2 Lexer（词法分析器状态）

```ts
type Lexer = {
  src: string
  len: number
  i: number          // JS string char index（用于 charAt）
  b: number          // UTF-8 byte offset（用于节点位置）
  heredocs: HeredocPending[]
  byteTable: Uint32Array | null  // 懒加载：char index → byte offset 映射表
}
```

- **双索引设计**：`i` 用于快速访问 JS 字符串字符，`b` 用于生成兼容 tree-sitter 的字节偏移。
- **byteTable 懒加载**：如果输入全是 ASCII（`srcBytes === source.length`），则走 fast path，不建表；遇到第一个非 ASCII 字符时，才一次性构建 `Uint32Array`。

#### 3.1.3 ParseState（解析器状态）

```ts
type ParseState = {
  L: Lexer
  src: string
  srcBytes: number
  isAscii: boolean
  nodeCount: number
  deadline: number        // performance.now() + timeoutMs
  aborted: boolean
  inBacktick: number      // 反引号嵌套深度
  stopToken: string | null // 用于 `[` 回退时的提前终止标记
}
```

### 3.2 关键流程

#### 3.2.1 入口：`parseSource`

```ts
function parseSource(source: string, timeoutMs?: number): TsNode | null
```

流程：
1. 创建 Lexer，计算 `srcBytes`（UTF-8 字节长度）。
2. 初始化 `ParseState`，设置 deadline（默认 50ms）。
3. 调用 `parseProgram(P)` 生成 AST。
4. 若 `P.aborted` 为 true 或抛出异常，返回 `null`。

#### 3.2.2 词法分析：`nextToken`

`nextToken(L, ctx: 'cmd' | 'arg' = 'arg')` 是上下文敏感的分词器：

- **ctx='cmd'**：`[` 和 `[[` 被识别为 OP（测试命令开始），`{` 在特定条件下也是 OP。
- **ctx='arg'**：`[`、`{`、`}` 被视为 WORD 的一部分（用于 glob、brace expansion）。

特殊处理：
- **行继续符**：`\<newline>` 和 `\<space>`/`\<tab>` 被 `skipBlanks` 吸收，与 tree-sitter 的 `_whitespace` extras 对齐。
- **文件描述符前缀**：数字串后紧跟 `>` 或 `<` 时，数字串作为 WORD 返回（如 `2>` 中的 `2`）。
- **单引号字符串**：`nextToken` 直接扫描完整内容并返回 `SQUOTE` token（含首尾引号）。

#### 3.2.3 语法分析：递归下降 + 回溯

解析器采用**递归下降**架构，大量使用**手动回溯**（`saveLex` / `restoreLex`）来模拟 tree-sitter 的 GLR/歧义消解行为。

关键示例：

**`[` 测试命令的双重解析**

```ts
if (t.type === 'OP' && (t.value === '[' || t.value === '[[')) {
  // 先尝试 parseTestExpr（标准测试表达式）
  const exprSave = saveLex(P.L)
  let expr = parseTestExpr(P, closer)
  skipBlanks(P.L)
  if (t.value === '[' && peek(P.L) !== ']') {
    // 没遇到 `]`，回退并尝试 parseSimpleCommand（如 `[ ! cmd -v go &>/dev/null ]`）
    restoreLex(P.L, exprSave)
    const prevStop = P.stopToken
    P.stopToken = ']'
    const rstmt = parseCommand(P)
    P.stopToken = prevStop
    if (rstmt && rstmt.type === 'redirected_statement') {
      expr = rstmt
    } else {
      restoreLex(P.L, exprSave)
      expr = parseTestExpr(P, closer)
    }
  }
  // ...
}
```

**`parseWord`：词片段组合**

`parseWord` 不是简单的 token 消费，而是一个**词片段组合器**：
- 扫描 bare word、双引号字符串、单引号 raw_string、`$` 展开、反引号、进程替换、`{...}` brace expression。
- 若扫描到多个相邻片段，则包装为 `concatenation` 节点；若只有一个，直接返回该片段。
- 这精确模拟了 tree-sitter 对 `"pre$VARpost"` 的处理方式。

#### 3.2.4 Heredoc 处理：延迟扫描 + 占位

Heredoc 是 Bash 解析中较复杂的部分，因为 body 在**下一个逻辑 newline 之后**才开始，且可能跨多行。

流程：
1. `tryParseRedirect` 遇到 `<<` / `<<-` 时，扫描定界符（支持引号/反斜杠转义），将 heredoc 信息以 `HeredocPending` 形式压入 `L.heredocs`。
2. `parseStatements` 在每次遇到 NEWLINE 时调用 `scanHeredocBodies(P)`。
3. `scanHeredocBodies` 从当前位置开始逐行扫描，匹配定界符行（`<<-` 会 strip 前导 tab）。
4. 匹配成功后，在 `heredoc_redirect` 节点下追加 `heredoc_body` 和 `heredoc_end` 子节点。
5. **SECURITY 注释**：`tryParseRedirect` 中 heredoc 定界符后的同一行内容（如 `cat <<EOF | rm -rf /tmp/evil`）会被继续解析。若遇到 `|`/`&&`/`||`/`;` 等结构运算符，会生成 `ERROR` 节点，确保下游 `ast.ts` 的 fail-closed 路径能拒绝这类复杂构造。

#### 3.2.5 算术表达式解析： precedence climbing

`parseArithExpr` 使用 **precedence climbing（运算符优先级爬升）** 算法处理 `$((...))`、`((...))`、C-style for 中的算术表达式：

- 支持三元运算符 `?:`。
- 支持前缀/后缀 `++`/`--`。
- 支持赋值运算符 `=`、`+=`、`-=`、`*=`、`/=`、`%=`、`<<=`、`>>=`、`&=`、`^=`、`|=`。
- 支持幂运算 `**`（右结合）。
- 三种模式：`var`（标识符 → `variable_name`）、`word`（标识符 → `word`）、`assign`（含 `=` 时生成 `variable_assignment`）。

#### 3.2.6 UTF-8 字节偏移处理

由于 `TsNode` 的索引是字节偏移，而 JS 字符串是 UTF-16 code unit，文件实现了以下辅助函数：

- `byteLengthUtf8(source)`：计算字符串的 UTF-8 字节长度。
- `advance(L)`：每前进一个字符，同时更新 `i`（char index）和 `b`（byte offset）。对 surrogate pair（4 字节 UTF-8）做了特殊处理。
- `byteAt(L, charIdx)`：懒构建 `byteTable`，返回给定字符索引对应的字节偏移。
- `sliceBytes(P, startByte, endByte)`：通过二分查找 `byteTable`，将字节偏移转换为字符索引，再用 `String.slice` 提取文本。

### 3.3 安全相关实现细节

#### 3.3.1 资源限制

```ts
const PARSE_TIMEOUT_MS = 50
const MAX_NODES = 50_000
```

- `checkBudget(P)` 在每次 `mk()` 创建节点时被调用。
- 每创建 128 个节点（`P.nodeCount & 0x7f === 0`）检查一次 `performance.now() > P.deadline`，减少高频计时调用开销。
- 超时或超节点数时设置 `P.aborted = true` 并抛出异常，外层 `parseSource` catch 后返回 `null`。

#### 3.3.2 反引号嵌套深度

`P.inBacktick` 用于跟踪当前是否处于反引号命令替换内部。在 `parseWord` 中，若 `P.inBacktick > 0`，遇到 `` ` `` 会终止当前词（因为反引号是闭合符），防止无限递归。

#### 3.3.3 `[` 回退的 `stopToken`

`parseSimpleCommand` 在解析参数时，正常情况下会尽可能消费 token。但在 `[ ... ]` 回退场景中，必须让 `parseSimpleCommand` 在 `]` 前停止，否则外层 `parseCommand` 无法正确闭合测试命令。`P.stopToken = ']'` 就是为此设计的提前终止机制。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件内部结构

| 行号范围 | 内容 |
|---------|------|
| 1-47 | 导出接口与常量（`TsNode`、`PARSE_TIMEOUT_MS`、`MAX_NODES`） |
| 48-591 | 词法分析器（`Lexer`、`nextToken`、`skipBlanks`、各类 `isXxxChar`） |
| 593-752 | 顶层语法：`parseSource`、`parseProgram`、`parseStatements` |
| 753-992 | 表达式级语法：`parseAndOr`、`parsePipeline`、`parseCommand` |
| 993-1404 | `parseSimpleCommand`、`maybeRedirect`、`tryParseAssignment` |
| 1405-1987 | 重定向、Heredoc、进程替换（`tryParseRedirect`、`scanHeredocBodies` 等） |
| 1988-2161 | `parseWord`、`parseBareWord`、`tryParseBraceExpr`、`tryParseBraceLikeCat` |
| 2162-3150 | 引号与展开式：`parseDoubleQuoted`、`parseDollarLike`、`parseExpansionBody`、`parseBacktick` |
| 3151-3697 | 控制流：`parseIf`、`parseWhile`、`parseFor`、`parseCase`、`parseFunction`、`parseDeclaration`、`parseUnset` |
| 3698-4061 | 测试表达式：`parseTestExpr`、`parseTestOr`、`parseTestAnd`、`parseTestBinary`、`parseTestRegexRhs`、`parseTestExtglobRhs` |
| 4062-4436 | 算术表达式：`parseArithExpr`、`parseArithTernary`、`parseArithBinary`、`parseArithUnary`、`parseArithPostfix`、`parseArithPrimary` |

### 4.2 下游调用方

#### 4.2.1 `src/utils/bash/parser.ts`

- **导入**：`ensureParserInitialized`、`getParserModule`、`type TsNode`
- **职责**：解析器门面（Facade）。根据 feature flag 决定使用原生 tree-sitter 还是本纯 TS 解析器。
- **关键函数**：
  - `parseCommand(command)` → 返回 `ParsedCommandData`（含 `rootNode`、`envVars`、`commandNode`）。
  - `parseCommandRaw(command)` → 返回 `Node | null | typeof PARSE_ABORTED`。
- **交互**：调用 `getParserModule().parse(command)` 获取本文件输出的 AST。

#### 4.2.2 `src/utils/bash/ast.ts`

- **导入**：`SHELL_KEYWORDS`、`type Node`（来自 `parser.ts`，即 `TsNode` 别名）、`PARSE_ABORTED`、`parseCommandRaw`
- **职责**：安全 walker，基于显式 allowlist 遍历 AST，提取 `SimpleCommand[]` 或返回 `too-complex`。
- **关键交互**：
  - `walkProgram(root)` 递归遍历 `program`、`list`、`pipeline`、`redirected_statement` 等结构类型。
  - 对 `command_substitution`、`process_substitution`、`subshell`、`expansion` 等危险类型返回 `too-complex`。
  - `parseForSecurityFromAst(cmd, root)` 被 `bashPermissions.ts` 调用。

#### 4.2.3 `src/utils/bash/ParsedCommand.ts`

- **导入**：`type Node`（来自 `parser.ts`）
- **职责**：实现 `IParsedCommand` 接口，基于 AST 提取管道位置、输出重定向节点、tree-sitter 分析数据。
- **关键交互**：`buildParsedCommandFromRoot(command, root)` 被 `doParse` 调用，将 AST 转换为 `TreeSitterParsedCommand` 实例。

#### 4.2.4 `src/utils/bash/prefix.ts`

- **导入**：`parseCommand`、`extractCommandArguments`
- **职责**：静态提取命令前缀（如 `git commit`），用于权限规则匹配和 UI 展示。
- **关键交互**：`getCommandPrefixStatic` 先调用 `parseCommand`，再对 `commandNode` 做 `extractCommandArguments`。

#### 4.2.5 `src/utils/bash/treeSitterAnalysis.ts`

- **导入**：无直接导入 `bashParser.ts`，但操作的是 `TreeSitterNode`（与 `TsNode` 同构）。
- **职责**：从 AST 中提取 quote context、compound structure、dangerous patterns、actual operator nodes。
- **关键交互**：`analyzeCommand(root, command)` 被 `ParsedCommand.ts` 调用，为安全校验提供辅助数据。

#### 4.2.6 `src/tools/BashTool/bashPermissions.ts`

- **导入**：`parseCommandRaw`（来自 `parser.ts`）→ 间接使用本解析器。
- **职责**：Bash 权限判定的核心，调用 `parseForSecurityFromAst` 进行命令安全分析。

#### 4.2.7 `src/tools/BashTool/pathValidation.ts`

- **导入**：`type Redirect`、`type SimpleCommand`（来自 `ast.ts`）。
- **职责**：路径约束校验，对 AST 提取出的 `argv` 和 `redirects` 进行允许目录校验。

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

本文件**无任何外部 npm 依赖**，完全基于 TypeScript 标准库和 Web API：

- `performance.now()`：用于超时检测。
- `Uint32Array`：用于字节偏移映射表。

### 5.2 与原生 tree-sitter 的关系

| 维度 | 原生 tree-sitter-bash | 本文件（bashParser.ts） |
|------|----------------------|------------------------|
| 实现语言 | Rust + WASM / NAPI | 纯 TypeScript |
| 初始化 | 异步（WASM 加载） | 同步（零初始化） |
| 输出格式 | `TsNode` 兼容 | 完全相同的 `TsNode` 结构 |
| 性能 | 更快（原生代码） | 足够快（50ms 内完成常见命令） |
| 覆盖度 | 完整 Bash 语法 | 覆盖常用 Bash 子集 + 安全关键路径 |

### 5.3 与 shell-quote 的关系

`shell-quote` 是 legacy 路径使用的库（`commands.ts`、`shellQuote.ts`）。当 tree-sitter（原生或本文件）不可用时，系统会降级到 `shell-quote` + 正则的解析路径。本文件的存在大幅减少了需要走 legacy 路径的场景。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 解析器差异风险（Parser Differential）

虽然文件注释中多次提到 "与 tree-sitter 保持一致"，但**任何 hand-written 递归下降解析器与 GLR 解析器之间都存在固有差异风险**。具体表现：

- **边缘语法**：某些极端嵌套（如多层混合的 `${...}`、`$(...)`、算术展开、引号）可能与 tree-sitter 产生不同的 AST 形状。
- **安全影响**：若本解析器比 tree-sitter "更宽容"，可能将危险命令解析为 `simple` 而原生解析器会生成 `ERROR`；反之若更严格，则可能导致本可正常执行的命令被误判为 `too-complex`。
- **现有缓解**：文件开头注释提到 "Validated against a 3449-input golden corpus generated from the WASM parser"，说明有语料库验证，但语料库覆盖度未知。

#### 6.1.2 资源限制误伤

- `PARSE_TIMEOUT_MS = 50` 在 CI 或高负载环境下可能因 jitter 导致正常但较长的命令被 abort。注释中已说明测试场景可传 `Infinity` 禁用。
- `MAX_NODES = 50_000` 对极长管道命令（如 `a | b | c | ...` 上千段）可能触发 abort。这类输入在正常业务中罕见，但 adversarial 输入可能故意构造。

#### 6.1.3 字节偏移计算性能

- 对包含大量非 ASCII 字符的输入，每次 `sliceBytes` 都要在 `byteTable` 上做二分查找。虽然单次是 `O(log n)`，但在高频创建节点时可能累积为不可忽视的开销。
- **现有缓解**：`isAscii` fast path 跳过了大部分常见 ASCII 命令的额外开销。

#### 6.1.4 Heredoc 安全边界

`tryParseRedirect` 中关于 heredoc 后尾随运算符的处理（`ls <<'EOF' | rm -rf /tmp/evil`）通过生成 `ERROR` 节点来 fail-closed。但这里的解析逻辑非常手工化，若未来 tree-sitter 语法升级导致 heredoc 内嵌 pipeline 的 AST 形状变化，可能需要同步修改 `ast.ts` 的 `walkHeredocRedirect`。

#### 6.1.5 无单元测试覆盖

经全局搜索，**未找到专门针对 `bashParser.ts` 的 `.test.ts` 或 `.spec.ts` 文件**。该文件的测试可能依赖于：
- 集成测试（通过 `parser.ts` 或 `ast.ts` 的测试间接覆盖）。
- 3449-input golden corpus（外部语料库，未在仓库内发现相关测试脚本）。

**缺乏独立单元测试意味着**：
- 局部重构（如修改 `nextToken` 或 `parseWord`）的风险较高。
- 新加入的 edge-case 修复难以被 regression test 保护。

### 6.2 边界情况

| 边界 | 行为 |
|------|------|
| 空字符串 | `parseProgram` 返回 `program` 节点，无 children |
| 仅注释/空白 | `program` 节点范围可能为 `progStart` 到 `progStart` |
| 未闭合引号 | `parseDoubleQuoted` 会在 EOF 处生成缺失的闭合 `"` 节点 |
| 未闭合 `${` | `parseExpansionBody` 会在 EOF 处生成缺失的 `}` 节点 |
| 未闭合 heredoc | `scanHeredocBodies` 将剩余全部内容视为 body，`endStart=endEnd=bodyEnd` |
| 尾随反斜杠 | `parseBareWord` 在 EOF 处停止，不将 `\` 纳入 word，由外层生成 `ERROR` |
| 反引号嵌套 | `inBacktick` 深度控制，防止无限递归 |

### 6.3 改进建议

#### 6.3.1 增加独立单元测试套件

建议为 `bashParser.ts` 增加独立的测试文件（如 `src/utils/bash/bashParser.test.ts`），覆盖：
- 词法层：各类 token 的边界（`nextToken` 的 `cmd` vs `arg` 模式）。
- 语法层：simple command、pipeline、list、redirected_statement 的 AST 形状断言。
- 展开式：`$VAR`、`${...}`、`$((...))`、`$(...)`、`` `...` ``、`<(...)` 的嵌套组合。
- Heredoc：引号定界符、`<-`、未闭合、多 heredoc 共享一行。
- 安全相关：资源限制（构造 50k+ 节点输入验证 abort）、 adversarial 输入（如 `(( a[0][0]... ))`）。
- UTF-8：多字节字符的位置和 `sliceBytes` 正确性。

#### 6.3.2 引入差异模糊测试（Differential Fuzzing）

既然有原生 tree-sitter 可用，建议编写一个 differential fuzzer：
1. 随机生成合法/非法 bash 片段。
2. 同时用原生 tree-sitter 和本解析器解析。
3. 对比 AST 结构（节点类型序列、父子关系、字节范围）。
4. 自动报告不一致 case。

这比静态的 3449-input corpus 更能发现未知 edge case。

#### 6.3.3 优化非 ASCII 路径性能

对于已知的高频非 ASCII 场景（如包含大量 Emoji 或 CJK 的命令字符串），可考虑：
- 预计算 `byteTable` 时同时缓存常用 `sliceBytes` 结果（如按行或按 token 缓存）。
- 或改用 `TextEncoder` 将源字符串编码为 `Uint8Array`，直接在字节数组上操作，彻底消除字节↔字符索引转换开销。但这会重构大量现有代码，成本较高。

#### 6.3.4 统一 heredoc 处理逻辑

当前 heredoc 的扫描逻辑分散在：
- `bashParser.ts` 的 `tryParseRedirect` + `scanHeredocBodies`
- `commands.ts` 的 `extractHeredocs`（用于 shell-quote legacy 路径）
- `heredoc.ts` 的 `extractHeredocs` / `restoreHeredocs`

这种分散增加了维护成本和安全一致性风险。建议将 heredoc 的定界符扫描、body 范围计算、引号状态判断抽象为共享工具模块，供两种解析路径复用。

#### 6.3.5 增加解析覆盖率监控

在 `ast.ts` 的 `tooComplex` 路径增加更细粒度的 `reason` 分类和 `logEvent`，可以：
- 统计哪些 `nodeType` 最常导致 `too-complex`。
- 识别用户实际输入中哪些语法结构尚未被安全 walker 支持。
- 为 `bashParser.ts` 的语法覆盖优先级提供数据驱动依据。

#### 6.3.6 文档化 AST 形状契约

`bashParser.ts` 与 `ast.ts`、`ParsedCommand.ts` 之间存在隐式的 AST 形状契约（例如 `redirected_statement` 的子节点顺序、`list` 中运算符的位置、`command` 中 `command_name` 的位置）。建议：
- 在 `bashParser.ts` 的 JSDoc 中增加 "AST 契约" 章节，明确各节点类型的子节点排列规则。
- 或引入运行时断言（在 `mk()` 中根据 `type` 校验 children 结构），在开发和测试阶段快速捕获契约破坏。

---

## 7. 附录：核心类型速查

```ts
// AST 节点（与 tree-sitter 兼容）
export type TsNode = {
  type: string
  text: string
  startIndex: number   // UTF-8 byte offset
  endIndex: number
  children: TsNode[]
}

// 解析器模块接口
export type ParserModule = {
  parse: (source: string, timeoutMs?: number) => TsNode | null
}

// 资源限制
const PARSE_TIMEOUT_MS = 50
const MAX_NODES = 50_000
```

---

*文档结束*
