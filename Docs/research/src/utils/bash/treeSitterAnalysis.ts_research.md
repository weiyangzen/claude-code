# Research: src/utils/bash/treeSitterAnalysis.ts

## 场景与职责

`treeSitterAnalysis.ts` 是 Claude Code Bash 安全系统的**树 sitting AST 分析引擎**。它接收来自原生 NAPI tree-sitter bash parser 的纯 JS 对象（AST），提取所有与安全验证相关的结构化信息。该模块是**从 regex/shell-quote  legacy 路径向精确语法分析迁移的核心支柱**，主要职责包括：
1. **引号上下文提取**：识别命令中哪些部分处于单引号、双引号、ANSI-C 引号或 heredoc 中，生成三种不同精度的"去引号"文本视图。
2. **复合命令结构分析**：检测命令是否包含 `&&`、`||`、`;`、管道 `|`、子 shell `$(...)`、命令组 `{...}` 等复合结构。
3. **真实操作符节点检测**：区分 AST 中真正的 `;` 操作符与 `find -exec \;` 这样的转义分号（tree-sitter 将后者解析为 `word` 节点）。
4. **危险模式识别**：检测命令替换 `$()`、进程替换 `<()`、参数扩展 `${}`、heredoc、注释等。

该模块被 `ParsedCommand.ts` 和 `bashSecurity.ts` 调用，是**权限判断和命令拆分的权威数据源**。

## 功能点目的

### 1. `extractQuoteContext(rootNode, command)`
- **目的**：替代 `bashSecurity.ts` 中手写的 `extractQuotedContent()` 字符级状态机，基于 AST 更准确地提取引号上下文。
- **输出三种视图**：
  - `withDoubleQuotes`：移除单引号内容，但保留双引号内容（仅移除 `"` 定界符）。
  - `fullyUnquoted`：移除所有被引用的内容。
  - `unquotedKeepQuoteChars`：移除被引用内容，但保留引号字符本身（如 `'`、`"`、`$'`）。
- **AST 节点类型映射**：
  - `raw_string` → 单引号字符串（`'...'`）
  - `string` → 双引号字符串（`"..."`）
  - `ansi_c_string` → ANSI-C 引用（`$'...'`）
  - `heredoc_redirect` → heredoc 重定向（仅当 heredoc_start 以 `'`、`"` 或 `\` 开头时视为"引用 heredoc"）
- **单遍收集优化**：`collectQuoteSpans()` 将原先 5 次独立的树遍历合并为 1 次，减少约 5 倍的 AST 遍历开销。通过 `inDouble` 参数追踪状态，确保只收集最外层的双引号跨度，同时仍能递归进入 `$()`/`${}` 内部收集嵌套的单引号。

### 2. `extractCompoundStructure(rootNode, command)`
- **目的**：替代 `isUnsafeCompoundCommand_DEPRECATED()` 和 `splitCommand_DEPRECATED()` 的 regex 路径，基于 AST 精确拆分复合命令。
- **检测项**：
  - `hasCompoundOperators`：是否存在 `&&`、`||`、`;`
  - `hasPipeline`：是否存在 `|`
  - `hasSubshell`：是否存在 `$(...)` 或 `` `...` ``
  - `hasCommandGroup`：是否存在 `{...}`
  - `operators`：收集到的操作符类型数组
  - `segments`：按操作符拆分后的原始文本片段数组
- **特殊节点处理**：
  - `redirected_statement`：递归进入内部结构，跳过 `file_redirect` 子节点。例如 `cmd1 && cmd2 2>/dev/null && cmd3` 中，tree-sitter 将整个复合结构包装在 `redirected_statement` 中，需要递归才能发现内层的 `&&`。
  - `negated_command`（`! cmd`）：记录完整文本为 segment，同时递归分析内部结构。
  - 控制流语句（`if`/`while`/`for`/`case`/`function_definition`）：将构造本身作为一个 segment，同时递归分析内部。

### 3. `hasActualOperatorNodes(rootNode)`
- **目的**：消除 `find -exec \;` 的误报。
- **关键洞察**：tree-sitter 将 `\;` 解析为 `word` 节点，而不是 `;` 操作符节点。因此，如果 AST 中不存在真正的 `;` / `&&` / `||` / `list` 节点，就可以跳过 `hasBackslashEscapedOperator()` 的昂贵 regex 检查。
- **安全意义**：这是解决 `find` 命令频繁触发复合命令误报的核心函数。

### 4. `extractDangerousPatterns(rootNode)`
- **目的**：基于 AST 节点类型识别危险模式。
- **检测项**：
  - `command_substitution` → `hasCommandSubstitution: true`
  - `process_substitution` → `hasProcessSubstitution: true`
  - `expansion` → `hasParameterExpansion: true`
  - `heredoc_redirect` → `hasHeredoc: true`
  - `comment` → `hasComment: true`

### 5. `analyzeCommand(rootNode, command)`
- **目的**：一站式完整分析入口，返回 `TreeSitterAnalysis` 对象。
- **组成**：
  ```ts
  {
    quoteContext,
    compoundStructure,
    hasActualOperatorNodes,
    dangerousPatterns,
  }
  ```
- **调用时机**：必须在 `tree.delete()` 之前调用（因为原生 parser 返回的是纯 JS 对象，实际上无 cleanup 需求，但注释保留了与旧 WASM 路径一致的提醒）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 内部 AST 节点类型
```ts
type TreeSitterNode = {
  type: string
  text: string
  startIndex: number
  endIndex: number
  children: TreeSitterNode[]
  childCount: number
}
```
注意：这里的 `startIndex`/`endIndex` 是 **UTF-8 字节偏移**（来自 tree-sitter 原生 NAPI 模块），但本模块仅做节点遍历和 `text` 读取，不做字符串切片，因此不受 UTF-8/UTF-16 差异影响。

### 跨度处理工具函数
- `buildPositionSet(spans)`：将 `[start, end)` 跨度内的所有字符位置加入 `Set`，用于逐字符过滤。
- `dropContainedSpans(spans)`：过滤掉完全包含在另一个跨度内的内层跨度，防止重叠跨度导致索引错乱。
- `removeSpans(command, spans)`：按降序排列跨度后，从字符串中逐个删除。用于 `fullyUnquoted`。
- `replaceSpansKeepQuotes(command, spans)`：将跨度内容替换为仅保留定界符（如 `''`、`""`、`$''`）。用于 `unquotedKeepQuoteChars`。

### QuoteSpans 收集状态
```ts
type QuoteSpans = {
  raw: Array<[number, number]>      // raw_string (单引号)
  ansiC: Array<[number, number]>    // ansi_c_string ($'...')
  double: Array<[number, number]>   // string (双引号)
  heredoc: Array<[number, number]>  // 被引用的 heredoc_redirect
}
```

### `collectQuoteSpans` 的状态追踪
```ts
function collectQuoteSpans(node, out, inDouble): void {
  switch (node.type) {
    case 'raw_string':
      out.raw.push([node.startIndex, node.endIndex])
      return // 字面量，无嵌套引号
    case 'ansi_c_string':
      out.ansiC.push([node.startIndex, node.endIndex])
      return
    case 'string':
      if (!inDouble) out.double.push([node.startIndex, node.endIndex])
      for (const child of node.children) collectQuoteSpans(child, out, true)
      return
    case 'heredoc_redirect':
      // 检查 heredoc_start 首字符是否为 ' / " / \
      // 若是，则为引用 heredoc，内容不展开
      // 若否，继续递归（unquoted heredoc 内部可能有 $(cmd 'x')）
  }
}
```

## 关键代码路径与文件引用

### 调用方
| 文件 | 引用符号 | 场景 |
|------|---------|------|
| `src/utils/bash/ParsedCommand.ts` | `analyzeCommand`, `extractQuoteContext`, `extractCompoundStructure`, `hasActualOperatorNodes`, `extractDangerousPatterns` | 构建 `TreeSitterParsedCommand`，提供 `IParsedCommand` 接口 |
| `src/tools/BashTool/bashSecurity.ts` | `TreeSitterAnalysis` (type) | 安全验证器读取 AST 分析结果进行权限判断 |

### 与 `ParsedCommand.ts` 的协作
`ParsedCommand.ts` 中的 `buildParsedCommandFromRoot()` 调用 `analyzeCommand()` 获取完整分析数据，同时自行提取 `pipePositions` 和 `redirectionNodes`。`TreeSitterParsedCommand` 将分析结果缓存，供 `bashSecurity.ts` 的各验证器使用。

### 与 `bashSecurity.ts` 的协作
`bashSecurity.ts` 的 `ValidationContext` 包含可选字段 `treeSitter?: TreeSitterAnalysis | null`。各验证器优先使用 AST 数据（如 `treeSitter.compoundStructure.hasPipeline`），在 tree-sitter 不可用时回退到 regex。

## 依赖与外部交互

- **原生 NAPI tree-sitter bash parser**（`src/utils/bash/bashParser.ts` / `parser.ts`）：提供 AST 的纯 JS 对象表示。该模块本身不直接导入 parser，而是接收调用方传入的 `rootNode`（类型声明为 `unknown`，运行时断言为 `TreeSitterNode`）。
- **`bashSecurity.ts`**：消费 `TreeSitterAnalysis` 的安全判断结果。
- **无第三方 npm 依赖**：该模块是纯算法模块，仅依赖 TypeScript 内置类型和数组/字符串/Set 操作。

## 风险、边界与改进建议

### 风险
1. **AST 节点类型的隐式契约**：模块大量依赖 tree-sitter bash grammar 的节点类型名称（如 `raw_string`、`ansi_c_string`、`heredoc_redirect`、`file_redirect` 等）。如果未来升级 tree-sitter bash grammar 导致节点类型重命名或结构变化，本模块将静默失效或产生错误结果。
2. **`startIndex`/`endIndex` 的编码假设**：虽然本模块主要读取 `node.text` 而不做切片，但 `ParsedCommand.ts` 中的 `TreeSitterParsedCommand` 使用 `Buffer.from(command, 'utf8').subarray(startIndex, endIndex)` 来处理管道分段和重定向移除。如果 tree-sitter 的索引语义从 UTF-8 变为 UTF-16 或代码点，将产生切片错位。
3. **`hasActualOperatorNodes` 的过度乐观**：该函数在发现 `list` 节点时直接返回 `true`。虽然 `list` 节点通常意味着 `&&`/`||`，但某些边缘语法（如空的 list 或解析器错误恢复产生的虚假 `list` 节点）可能导致误报。
4. **复合命令拆分的 `segments` 精度**：`extractCompoundStructure` 将控制流语句（`if`/`for` 等）整体作为一个 segment。对于 `if cmd1; then cmd2; fi && cmd3`，`segments` 会包含整个 `if` 块作为一段，这可能对需要细粒度子命令拆分的权限系统来说过于粗糙。

### 边界
- **仅分析 bash 语法**：tree-sitter bash grammar 对 zsh 特有语法（如 `=()`、glob 限定符 `(e:...)`）的解析可能不准确，导致 AST 结构异常，进而影响分析结果。Claude Code 的 zsh 安全验证仍大量依赖 regex 作为补充。
- **不处理别名展开**：tree-sitter 解析的是字面命令文本，不会展开 shell 别名。如果用户定义了 `alias gs='git status'`，解析器看到的是 `gs`，而不是 `git status`。
- **Heredoc 的引用判定仅检查 `heredoc_start` 首字符**：对于更复杂的 heredoc 定界符形式（如带引号的变量插值、转义序列），当前的 `first === "'" || first === '"' || first === '\\'` 检查可能不够全面。

### 改进建议
1. **节点类型契约的自动化测试**：建立一套基于已知命令字符串的 AST 快照测试，覆盖所有本模块依赖的节点类型。每次升级 tree-sitter bash grammar 后运行，确保节点类型名称和层级结构未变。
2. **与 grammar 版本耦合解耦**：考虑在 `bashParser.ts` 层封装一个稳定的节点类型查询接口（如 `isHeredocRedirect(node)`、`isCommandSubstitution(node)`），而不是在 `treeSitterAnalysis.ts` 中硬编码字符串比较。
3. **增强控制流语句的细分**：对于 `if`/`for`/`while` 等结构，可进一步提取其条件/循环体中的命令作为独立 segments，使权限系统能够对 `if rm -rf /; then echo ok; fi` 中的 `rm` 子命令进行更精确的前缀提取和权限判断。
4. **引入 zsh grammar**：若项目长期支持 zsh 作为主要 shell，应考虑引入 tree-sitter zsh grammar，消除 zsh 语法依赖 regex 回退的问题。
5. **性能基准测试**：`collectQuoteSpans` 虽然已将 5 次遍历合并为 1 次，但对于极深或极宽的 AST（如包含数千个 token 的复杂命令），递归遍历仍可能有栈深度或性能风险。可引入非递归的显式栈遍历作为可选优化。
