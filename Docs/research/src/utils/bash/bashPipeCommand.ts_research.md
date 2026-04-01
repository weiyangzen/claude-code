# 研究文档：src/utils/bash/bashPipeCommand.ts

## 场景与职责

`bashPipeCommand.ts` 负责解决 **BashTool 在执行管道命令时的 stdin 重定向绑定问题**。当 BashTool 通过 `eval` 执行用户命令时，为了防止命令阻塞在 stdin 上，系统会在命令后追加 `< /dev/null`。但如果命令包含管道（`|`），直接将 `< /dev/null` 放在 `eval` 的参数末尾会导致重定向绑定到 `eval` 本身或管道的最后一个命令，而不是第一个命令。该模块通过解析并重构管道命令，将 stdin 重定向插入到第一个命令之后，确保 `rg foo | wc -l < /dev/null` 被正确执行为 `rg foo < /dev/null | wc -l`。

## 功能点目的

1. **`rearrangePipeCommand(command: string): string`**
   - 核心功能：识别管道命令，将 `< /dev/null` 从 `eval` 参数层级下移到管道第一个命令之后。
   - 安全策略：对无法安全解析的复杂 shell 结构（反引号、`$()`、变量引用、控制结构、未闭合引号等）直接回退到保守的 `quoteWithEvalStdinRedirect` 方案，避免错误重构导致命令语义改变或注入。

2. **`quoteWithEvalStdinRedirect(command: string): string`**
   - 回退方案：将整个命令用单引号包裹后追加 ` < /dev/null`，生成 `eval 'cmd' < /dev/null`。
   - 这样 `< /dev/null` 绑定到 `eval` 的 stdin，虽然对管道内部分命令的 stdin 控制不如理想方案精确，但避免了错误地将重定向附加到管道末尾。

3. **`singleQuoteForEval(s: string): string`**
   - 自定义单引号转义逻辑，使用 `'"   }'"''` 模式替换嵌入的单引号。
   - 不使用 `shell-quote` 库的 `quote()`，因为后者在遇到单引号时会切换到双引号模式并转义 `!` 为 `\!`，这会破坏 `jq`/`awk` 过滤器（如 `select(.x != .y)` 被错误转为 `select(.x \!= .y)`）。

4. **`buildCommandParts(parsed, start, end): string[]`**
   - 从 `shell-quote` 解析结果中重构命令片段，特殊处理文件描述符重定向（`2>&1`、`2>/dev/null`）和环境变量赋值（`VAR=value`）的引号恢复。

## 具体技术实现

### 关键流程：rearrangePipeCommand

```
输入 command
├── 包含 ` 或 $( 或 $VAR 或控制结构(for/while/if等)
│   └── 返回 quoteWithEvalStdinRedirect(command)
├── 包含未处理的换行符（非 continuation line）
│   └── 返回 quoteWithEvalStdinRedirect(command)
├── 存在 shell-quote 单引号 bug（\' 在单引号内被错误解析）
│   └── 返回 quoteWithEvalStdinRedirect(command)
├── tryParseShellCommand 解析失败
│   └── 返回 quoteWithEvalStdinRedirect(command)
├── 解析出的 token 存在 malformed（未闭合的括号/引号/花括号）
│   └── 返回 quoteWithEvalStdinRedirect(command)
├── 未找到管道操作符或在第一个位置
│   └── 返回 quoteWithEvalStdinRedirect(command)
└── 正常情况
    └── 返回：first_command < /dev/null | rest_of_pipeline
```

### 数据结构

- **ParseEntry[]**：来自 `shell-quote` 的解析结果，元素为字符串或 `{ op: string }` 对象。
- **buildCommandParts 的状态跟踪**：使用 `seenNonEnvVar` 布尔值确保环境变量赋值只在命令开头被识别，遇到实际命令或操作符后重置。

### 安全相关的特殊处理

| 检查项 | 目的 |
|--------|------|
| 反引号/ `$()` | `shell-quote` 对这两种结构解析不正确，可能错误识别管道边界 |
| `$VAR` / `${VAR}` | `shell-quote` 在无 env 时会将变量扩展为空字符串，导致 `$` 被转义，破坏运行时扩展 |
| 控制结构 | `for/while/until/if/case/select` 内部可能包含 `\|` 字符，但不是真正的管道 |
| `hasShellQuoteSingleQuoteBug` | `shell-quote` 将单引号内的 `\'` 视为转义，而 bash 视其为字面量 `\` + 闭引号，存在解析差分注入风险 |
| `hasMalformedTokens` | 检测未闭合的引号/括号/花括号，防止 `echo {"hi":"hi;evil"}` 这类将语法错误转化为有效注入命令的攻击 |
| `joinContinuationLines` | 正确处理 `\\<newline>` 的奇偶性：奇数反斜杠表示行继续，偶数表示反斜杠成对，换行是真实分隔符 |

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/bashPipeCommand.ts`
  - `rearrangePipeCommand` (L14)
  - `quoteWithEvalStdinRedirect` (L262)
  - `singleQuoteForEval` (L273)
  - `buildCommandParts` (L119)
  - `joinContinuationLines` (L283)

- **依赖文件**：`src/utils/bash/shellQuote.ts`
  - `tryParseShellCommand`：安全包装 `shell-quote.parse`，返回 `{ success, tokens }` 或 `{ success, error }`
  - `quote`：包装 `shell-quote.quote`
  - `hasMalformedTokens`：检测畸形 token
  - `hasShellQuoteSingleQuoteBug`：检测单引号解析差分

- **调用方**：`src/utils/shell/bashProvider.ts`
  - `createBashShellProvider.buildExecCommand` (L152-154)：当命令包含 `\|` 且需要 `addStdinRedirect` 时调用 `rearrangePipeCommand`

## 依赖与外部交互

- **外部库**：`shell-quote`（通过 `shellQuote.ts` 间接使用）
- **内部模块**：`src/utils/bash/shellQuote.ts`
- **调用方**：`src/utils/shell/bashProvider.ts`
- **无网络/IO 交互**：纯字符串处理模块

## 风险、边界与改进建议

### 已知风险

1. **解析器差分攻击（Parser Differential）**
   - `shell-quote` 与 bash 的解析行为存在多处不一致。本模块通过大量黑名单检查来回避，但新增 bash 语法特性或 `shell-quote` 的未知 bug 可能引入新的差分。
   - 例如 `#9732`、`#32515`、`HackerOne #3482049` 等历史漏洞均与解析差分相关。

2. **回退方案的精度损失**
   - 当触发回退时，`< /dev/null` 绑定到 `eval` 而非管道中的具体命令。对于某些依赖第一个命令读取 stdin 的管道场景（如 `cat | grep`），这可能导致行为变化，但当前场景下主要是防止阻塞，因此可接受。

3. **环境变量赋值的引号恢复**
   - `buildCommandParts` 中对 `VAR=value` 使用 `quote([value])` 重新引号。如果 `value` 本身包含复杂的 shell 特殊字符，引号恢复可能与原始命令不完全等价。

### 边界情况

- **空命令/无管道**：直接回退到 `quoteWithEvalStdinRedirect`
- **多个管道**：只处理第一个管道操作符之前和之后的部分，保留后续所有管道结构
- **glob 模式**：`entry.op === 'glob'` 时不进行 `quote()`，保留原始模式供 shell 展开
- **文件描述符重定向**：`2>&1`、`2>/dev/null`、`2> &1` 等三种变体均被识别并作为单一 token 保留

### 改进建议

1. **迁移到 tree-sitter AST 重构**
   - 项目已有 `bashParser.ts`（纯 TypeScript 实现的 tree-sitter-bash 兼容解析器）。未来可直接用 AST 定位 pipeline 的第一个 command，精确插入重定向，彻底摆脱对 `shell-quote` 的依赖和相应的解析差分风险。

2. **统一引号处理**
   - `singleQuoteForEval` 与 `shellQuote.ts` 中的 `quote()` 存在两条并行的引号路径。建议将 `singleQuoteForEval` 提升为通用工具函数，并在 `shellQuote.ts` 中统一维护。

3. **增加 fuzz 测试**
   - 针对 `rearrangePipeCommand` 与 bash 实际执行结果的对比，建立自动化 fuzz 测试框架，持续发现新的解析差分。
