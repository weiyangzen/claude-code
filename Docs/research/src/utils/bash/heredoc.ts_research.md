# 研究文档：src/utils/bash/heredoc.ts

## 场景与职责

`heredoc.ts` 负责解决 **shell-quote 库无法正确解析 heredoc 语法（`<<`）的问题**。`shell-quote` 将 `<<` 视为两个独立的 `<` 重定向操作符，这会破坏：
1. 命令拆分（`splitCommandWithOperators` 会错误地产生两个 `<` token）
2. 重定向提取（`extractOutputRedirections` 可能将 heredoc 体内容误判为独立命令）
3. 安全校验（heredoc 体中的命令替换如 `$(evil)` 可能被隐藏，绕过权限检查）

该模块在调用 `shell-quote.parse` **之前**将 heredoc 从命令中提取出来，替换为随机占位符；在解析完成**之后**再将占位符恢复为原始 heredoc 文本。

## 功能点目的

### 1. `extractHeredocs(command: string, options?): HeredocExtractionResult`
- 扫描命令字符串，识别所有合法的 heredoc 语法变体。
- 将每个 heredoc 替换为形如 `__HEREDOC_0_a1b2c3d4__` 的占位符（含随机 salt）。
- 返回 `{ processedCommand, heredocs: Map<placeholder, HeredocInfo> }`。
- 支持选项 `quotedOnly`：仅提取带引号或转义的分隔符（`<<'EOF'`、`<<\EOF`），跳过无引号的 `<<EOF`（因其体会被 bash 扩展，可能包含可执行代码）。

### 2. `restoreHeredocs(parts: string[], heredocs: Map): string[]`
- 在 `splitCommandWithOperators` 或其他后处理完成后，将字符串数组中的占位符替换回原始 heredoc 内容。

### 3. `containsHeredoc(command: string): boolean`
- 快速检查命令是否包含 heredoc 语法模式，用于早期短路判断。

## 具体技术实现

### Heredoc 语法支持

| 语法 | 示例 | 说明 |
|------|------|------|
| 基本 heredoc | `<<EOF` | 无引号分隔符，bash 会对体内容进行参数/命令/算术扩展 |
| 单引号分隔符 | `<<'EOF'` | 内容完全字面化，无扩展 |
| 双引号分隔符 | `<<"EOF"` | 分隔符带引号，内容字面化（与单引号行为相同） |
| 带横线 | `<<-EOF` | 剥除体内容每行前导的 **制表符**（非空格） |
| 转义分隔符 | `<<\EOF` | 反斜杠转义的分隔符，内容字面化 |

### 核心正则：`HEREDOC_START_PATTERN`

```ts
/(?&lt;!)&lt;&lt;(?&lt;!)(-)?[ \t]*(?:(['"])(\\?\w+)\2|\\?(\w+))/
```

- 使用负向回顾发 `(?&lt;!)` 确保 `<<` 不是 `<<<`（here-string）或 `<<-` 的一部分。
- **两个分支**：
  1. **引号分支**：`(['"])(\\?\w+)\2` — 捕获引号及内部的分隔符词（允许前导反斜杠）。
  2. **无引号分支**：`\\?(\w+)` — 可选的前导反斜杠被消耗为转义符，不进入捕获组。
- **安全修复历史**：旧版正则将 `\\?` 无条件放在捕获组外，导致 `<<'\EOF'` 提取的分隔符为 `EOF` 而 bash 实际使用 `\EOF`，造成命令走私。

### 增量式引号/注释扫描器（advanceScan）

为了避免对长 heredoc 体进行 O(n²) 的重复扫描（例如 C++ 代码 heredoc 中包含大量 `<<`），该模块实现了**增量状态机**：

```ts
let scanPos = 0
let scanInSingleQuote = false
let scanInDoubleQuote = false
let scanInComment = false
let scanDqEscapeNext = false
let scanPendingBackslashes = 0
```

- `advanceScan(target)` 从当前 `scanPos` 推进到目标位置，更新引号/注释/转义状态。
- **关键安全语义**：
  - 引号跟踪是**注释盲的**（comment-blind）：即使 `#` 开启了注释模式，引号字符仍继续更新状态。这防止了 `echo x#"\n<<...` 这类攻击（bash 将 `#` 视为单词的一部分，不是注释）。
  - 注释状态在每次遇到物理换行符时被清除（即使换行符在引号内），这与旧版 `isInsideComment` 的行为完全一致。

### 安全过滤与 Bailout 条件

在提取前，以下情况会导致**完全放弃提取**（返回原始命令）：

| 条件 | 原因 |
|------|------|
| 命令包含 `$'` 或 `$"` | ANSI-C / locale 引号，增量扫描器无法正确跟踪 |
| 第一个 `<<` 之前存在反引号 | 反引号嵌套规则复杂，且反引号可作为 `shell_eof_token` 提前关闭 heredoc |
| 第一个 `<<` 之前存在未匹配的 `((` | `(( x = 1 << 2 ))` 中的 `<<` 是位运算符，不是 heredoc；误提取会导致后续命令被吞入 heredoc 体 |

对每个匹配到的 `<<` 操作符，还会进行以下安全检查：

1. **在引号内**：跳过（不是真正的 heredoc 操作符）
2. **在注释内**：跳过（`# <<EOF` 是注释，后续行是独立命令）
3. **被反斜杠转义**：跳过（`\<<EOF` 是字面 `<` + 输入重定向）
4. **在 skipped heredoc 体内**：跳过（防止 `quotedOnly` 模式下，无引号 heredoc 体内的带引号 `<<'SAFE'` 被错误提取）
5. **分隔符后紧跟非元字符**：跳过（如 `<<'EOF'a`，bash 的分隔符是 `EOFa`）
6. **同一行末尾存在奇数反斜杠**：跳过（行继续会改变 heredoc 起始行的逻辑边界）
7. **关闭分隔符行存在 PST_EOFTOKEN 特征**：如 `EOF)`、`EOF}`、`` EOF` `` 等，bash 可能在 subshell/替换内提前关闭 heredoc，导致解析差分

### 多 heredoc 的嵌套与重叠处理

- **嵌套过滤**：通过 `topLevelHeredocs` 过滤掉 operator 位于另一个 heredoc 内容范围内的嵌套 heredoc。
- **共享起始行检测**：若多个 heredoc 的 `contentStartIndex` 相同（如 `cat <<EOF <<'SAFE'`），则放弃提取。因为索引计算基于原始字符串，但替换是逐次修改字符串的，会导致索引错位。
- **quotedOnly 模式下的 skipped 范围跟踪**：无引号 heredoc 即使被跳过，其内容范围也会被记录到 `skippedHeredocRanges`，确保后续迭代不会提取位于其体内的带引号 heredoc。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/heredoc.ts`
  - `extractHeredocs` (L113)
  - `restoreHeredocs` (L711)
  - `containsHeredoc` (L731)
  - `HEREDOC_START_PATTERN` (L69)
  - `advanceScan` (L231)

- **依赖文件**：无（仅使用 Node.js 内置 `crypto`）

- **调用方**：
  - `src/utils/bash/commands.ts`：`extractHeredocs` / `restoreHeredocs`（`splitCommandWithOperators`、`extractOutputRedirections`）
  - `src/tools/BashTool/bashSecurity.ts`：`containsHeredoc`
  - `src/utils/bash/shellQuoting.ts`：`containsHeredoc`
  - `src/utils/bash/treeSitterAnalysis.ts`：`containsHeredoc`

## 依赖与外部交互

- **外部库**：Node.js `crypto`（`randomBytes`）
- **内部模块**：无直接依赖
- **无网络/文件 IO**

## 风险、边界与改进建议

### 已知风险

1. **正则表达式与 bash 的解析差分**
   - `HEREDOC_START_PATTERN` 使用 `\w+` 匹配分隔符，但 bash 实际接受更多字符（如 `!`、`-`、`.`）。虽然 `\w+` 更严格（只匹配 `[A-Za-z0-9_]`），但这可能导致合法 heredoc 未被提取，而非错误提取——属于 fail-closed 方向，相对安全。
   - 然而，引号内分隔符含非单词字符时（如 `<<"EO F"`），正则无法匹配完整分隔符，通过 `command[operatorEndIndex - 1] !== quoteChar` 检测并跳过，这是正确的 fail-closed。

2. **增量扫描器的简化语义**
   - `advanceScan` 不处理 `$'...'`、ANSI-C 引号、算术扩展 `$((...))` 内部的引号状态。这些通过前置 bailout 条件规避，但攻击者可能构造出尚未被发现的 bypass 模式。

3. **PST_EOFTOKEN 的保守扩展**
   - 对关闭行中分隔符后的 `|&;(<>`` 等字符触发 bailout，这可能导致一些合法 heredoc（如体内容以 `EOF(` 作为巧合文本）被跳过。但这是安全优先的权衡。

### 边界情况

- **无关闭分隔符**：视为 malformed，跳过提取
- **空 heredoc 体**（`<<EOF\nEOF`）：正常提取
- **`<<-` 与空格**：`<<-` 只剥除前导 **制表符**，不剥除空格。实现使用 `line.replace(/^\t*/, '')`，与 POSIX/bash 一致。
- **CRLF 行尾**：当前实现基于 `\n` 分割，若输入使用 `\r\n`，`\r` 会保留在分隔符行末尾，导致关闭分隔符匹配失败。虽然大多数 BashTool 输入来自模型生成（通常为 `\n`），但这是一个潜在的跨平台问题。

### 改进建议

1. **统一使用 tree-sitter AST 处理 heredoc**
   - `bashParser.ts` 已内置完整的 heredoc 解析（包括 `<<-`、引号分隔符、体内容扫描）。未来可直接在 AST 层面处理 heredoc，彻底消除正则与 bash 的差分风险。
   - `ast.ts` 的 `walkHeredocRedirect` 已提供基于 AST 的安全策略（仅允许引号分隔符 heredoc），可作为统一入口。

2. **处理 CRLF 输入**
   - 在 `contentLines.split('\n')` 前统一将 `\r\n` 规范化为 `\n`，避免 Windows 风格行尾导致匹配失败。

3. **提取性能优化**
   - 当前 `advanceScan` 已将扫描复杂度从 O(n²) 降到 O(n)。对于超长 heredoc（如数千行脚本体），进一步优化的空间不大。若需极致性能，可直接用 tree-sitter 的 byte-range 切片，避免字符串分割。

4. **增加单元测试**
   - 建议为以下历史漏洞场景建立回归测试：
     - `<<'\EOF'` 分隔符差分
     - `cat <<EOF <<'SAFE'\n$(evil)\nEOF\nsafe body\nSAFE` 的嵌套/重叠处理
     - `cat <<'EOF' && \\\nrm -rf /` 的行继续绕过
     - `(( a = 1 << 2 ))` 的位运算符误识别
