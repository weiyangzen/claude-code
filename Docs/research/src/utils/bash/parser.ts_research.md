# 研究文档：src/utils/bash/parser.ts

## 场景与职责

`parser.ts` 是 BashTool 的 **tree-sitter bash 解析器入口层**，负责：
1. **异步初始化管理**：在首次解析前确保底层解析模块（WASM 或纯 TS）已加载。
2. **命令解析**：将原始 bash 命令字符串解析为 AST（`TsNode` 树）。
3. **结构化数据提取**：从 AST 中提取环境变量列表、主命令节点、命令参数等，供权限系统和前缀提取使用。
4. **解析失败分类**：区分"模块未加载/不可用"（`null`）与"解析被主动中止"（`PARSE_ABORTED`），后者必须被调用方按 fail-closed 处理。

该模块位于底层解析器（`bashParser.ts`）与上层业务逻辑（`ast.ts`、`prefix.ts`、`ParsedCommand.ts`）之间，起到**适配器与网关**的作用。

## 功能点目的

### 1. `ensureInitialized(): Promise<void>`
- 在 `TREE_SITTER_BASH` 或 `TREE_SITTER_BASH_SHADOW` feature flag 开启时，触发底层解析器初始化。
- 幂等设计：多次调用安全。

### 2. `parseCommand(command: string): Promise<ParsedCommandData | null>`
- 解析命令并返回结构化数据：`{ rootNode, envVars, commandNode, originalCommand }`。
- 长度限制：`MAX_COMMAND_LENGTH = 10000`，超限直接返回 `null`。
- 仅在 `feature('TREE_SITTER_BASH')` 为 true 时尝试加载和解析。
- 解析成功后：
  - `rootNode`：完整的 AST 根节点（`program`）
  - `commandNode`：通过 `findCommandNode` 定位到的第一个实际命令节点（`command` 或 `declaration_command`）
  - `envVars`：通过 `extractEnvVars` 提取的前置环境变量赋值列表

### 3. `parseCommandRaw(command: string): Promise<Node | null | typeof PARSE_ABORTED>`
- 原始解析接口，跳过 `findCommandNode` 和 `extractEnvVars`，直接返回 AST 根节点。
- **核心安全设计**：当底层解析器已加载但解析返回 `null` 时（超时或节点预算耗尽），返回 `PARSE_ABORTED` 符号而非 `null`。
- 这防止了调用方将"解析失败"误判为"解析器不可用"而回退到安全性较弱的旧版 regex/shell-quote 路径。

### 4. `extractCommandArguments(commandNode: Node): string[]`
- 从 `command` 或 `declaration_command` 节点中提取参数列表。
- 对 `declaration_command`（`export`、`declare`、`local` 等）特殊处理：返回 `[builtinName]`。
- 对普通 `command`：依次提取 `command_name` 和后续 `word`/`string`/`raw_string`/`number` 参数，遇到 `command_substitution` 或 `process_substitution` 时停止。
- 对字符串参数自动剥除外层引号（`"..."` 或 `'...'`）。

## 具体技术实现

### `findCommandNode` 的递归定位逻辑

```ts
function findCommandNode(node: Node, parent: Node | null): Node | null
```

- **匹配类型**：`command` 或 `declaration_command`
- **variable_assignment 上下文**：若当前节点是 `variable_assignment`，则在父节点的子节点中查找位于其后的第一个 `command`/`declaration_command`。这处理了 `VAR=value cmd` 的场景。
- **pipeline 上下文**：递归到 pipeline 的第一个子节点（可能是 `redirected_statement`）。
- **redirected_statement 上下文**：直接查找子节点中的 `command`/`declaration_command`。

### `extractEnvVars` 的提取规则

```ts
function extractEnvVars(commandNode: Node | null): string[]
```

- 仅当 `commandNode.type === 'command'` 时生效。
- 遍历 `commandNode.children`，收集连续的 `variable_assignment` 节点文本。
- 遇到 `command_name` 或 `word` 时立即停止。这确保只提取**命令前缀**的环境变量，而非后续参数中碰巧包含 `=` 的字符串。

### `PARSE_ABORTED` 的安全语义

在 `parseCommandRaw` 中：

```ts
if (result === null) {
  logEvent('tengu_tree_sitter_parse_abort', { cmdLength: command.length, panic: false })
  return PARSE_ABORTED
}
```

- **攻击背景**： adversarial 输入（如 `(( a[0][0]... ))` 配合约 2800 个子脚本）可在 10K 长度限制内触发 `bashParser.ts` 的 `PARSE_TIMEOUT_MS`（50ms）或 `MAX_NODES`（50,000）上限。
- **历史问题**：旧代码将超时/预算耗尽与"模块未加载"统一返回为 `null`，导致 `ast.ts` 回退到 `parse-unavailable`，进而让调用方走旧版路径。旧版路径缺少对 `trap`、`enable`、`hash` 等 EVAL_LIKE_BUILTINS 的拦截，造成安全绕过。
- **修复**：引入 `PARSE_ABORTED` 符号，调用方（如 `ast.ts` 的 `parseForSecurityFromAst`）将其明确分类为 `too-complex`，强制人工审批。

### feature flag 控制

```ts
import { feature } from 'bun:bundle'
```

- `TREE_SITTER_BASH`：主功能开关。仅在开启时才会真正尝试加载和解析。
- `TREE_SITTER_BASH_SHADOW`：影子模式开关。开启时 `parseCommandRaw` 会尝试加载，但 `parseCommand` 仍受 `TREE_SITTER_BASH` 主开关控制。用于灰度验证或遥测收集。

## 关键代码路径与文件引用

- **本文件**：`src/utils/bash/parser.ts`
  - `ensureInitialized` (L50)
  - `parseCommand` (L56)
  - `parseCommandRaw` (L104)
  - `findCommandNode` (L138)
  - `extractEnvVars` (L175)
  - `extractCommandArguments` (L189)
  - `MAX_COMMAND_LENGTH` (L19)
  - `PARSE_ABORTED` (L93)

- **依赖文件**：
  - `src/utils/bash/bashParser.ts`：`ensureParserInitialized`、`getParserModule`、`TsNode`
  - `src/services/analytics/index.js`：`logEvent`
  - `src/utils/debug.js`：`logForDebugging`

- **调用方**：
  - `src/utils/bash/ast.ts`：`parseCommandRaw`（`parseForSecurity`）
  - `src/utils/bash/prefix.ts`：`parseCommand`（`getCommandPrefixStatic`）
  - `src/utils/bash/ParsedCommand.ts`：`parseCommand`（`doParse`）
  - `src/tools/BashTool/bashPermissions.ts`：`parseCommandRaw`、`PARSE_ABORTED`
  - `src/tools/BashTool/pathValidation.ts`：间接通过 `ParsedCommand`
  - `src/utils/settings/mdm/settings.ts`：`ensureInitialized`

## 依赖与外部交互

- **外部库**：`bun:bundle`（`feature` 函数）
- **内部模块**：
  - `bashParser.ts`：底层解析器实现
  - `services/analytics/index.js`：解析加载/中止事件遥测
  - `utils/debug.js`：调试日志
- **无直接网络/文件 IO**

## 风险、边界与改进建议

### 已知风险

1. **`MAX_COMMAND_LENGTH` 的静态上限**
   - 10000 字符的硬上限可防御超长 adversarial 输入，但也可能误伤合法的长命令（如大型 heredoc 或复杂 pipeline）。当前没有配置化入口。

2. **`findCommandNode` 的简化语义**
   - 该函数只返回**第一个**匹配的命令节点。对于包含多个独立命令的复合命令（如 `cmd1; cmd2`），`prefix.ts` 的 `getCommandPrefixStatic` 只会看到 `cmd1`。
   - 这在前缀提取场景下是可接受的（前缀通常关注主命令），但在某些安全分析场景中可能需要遍历所有命令节点。

3. **`extractCommandArguments` 的引号剥离**
   - `stripQuotes` 简单检查首尾字符是否为 `"` 或 `'`，然后 `slice(1, -1)`。对于嵌套引号或不平衡引号（tree-sitter 的 ERROR 恢复节点），可能产生不准确的结果。不过调用方（如 `prefix.ts`）通常只关心命令名和前几个参数，对精确性要求不高。

4. **初始化与 feature flag 的竞态**
   - `ensureInitialized` 和 `parseCommandRaw` 都检查 `feature('TREE_SITTER_BASH')`。如果在会话中途动态切换 feature flag，可能出现首次解析时模块未加载、后续解析时模块已加载的状态不一致。当前代码通过 `ensureParserInitialized` 的幂次性和 `getParserModule` 的同步返回缓解了这一问题。

### 边界情况

- **空字符串/空白命令**：`parseCommand('')` 和 `parseCommandRaw('')` 均返回 `null`
- **解析器模块不可用**：`getParserModule()` 返回 `null`，`parseCommand` 返回 `null`
- **底层抛出异常**：`parseCommandRaw` 的 try-catch 将其捕获并返回 `PARSE_ABORTED`（异常路径会记录 `panic: true`）
- **非 ASCII 字符**：`bashParser.ts` 内部使用 UTF-8 byte offset，`parser.ts` 本身不处理字符编码，仅透传 `Node` 对象

### 改进建议

1. **统一解析入口**
   - 当前存在 `parseCommand`（带结构化提取）和 `parseCommandRaw`（纯 AST）两个入口。建议增加一个 `parseCommandSegments` 接口，直接返回命令列表，减少 `prefix.ts` 和 `ast.ts` 各自重复调用 `findCommandNode` 的冗余。

2. **配置化长度上限**
   - 将 `MAX_COMMAND_LENGTH` 从硬编码常量改为可通过环境变量或 feature flag 覆盖的配置项，便于在特定场景（如 CI、大型脚本生成）中调整。

3. **增强 `extractCommandArguments` 的健壮性**
   - 对于 `concatenation` 节点（如 `"pre"$VAR"post"`），当前 `extractCommandArguments` 仅返回 `stripQuotes(child.text)`，丢失了内部结构信息。建议增加对 `concatenation` 的递归处理，或在前缀提取场景下明确标记"含动态拼接"以便调用方决策。

4. **遥测细化**
   - 当前 `tengu_tree_sitter_parse_abort` 仅记录命令长度和 panic 标志。建议增加中止原因分类（timeout vs node-budget vs Rust panic）和输入特征哈希（如 heredoc 数量、嵌套深度），帮助识别 adversarial 输入模式。

5. **逐步退役旧路径后的清理**
   - 当 `TREE_SITTER_BASH` 成为默认且不可关闭的功能后，可考虑移除 `parseCommand` 中的 feature flag 分支，简化代码并减少 Bun DCE（Dead Code Elimination）的依赖。
