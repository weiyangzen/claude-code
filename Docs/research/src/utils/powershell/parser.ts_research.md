# Research: `src/utils/powershell/parser.ts`

## 场景与职责

`parser.ts` 是 Claude Code 中 PowerShell 工具的**核心 AST 解析基础设施**。它通过调用本地安装的 `pwsh`/`powershell.exe`，利用 PowerShell 的原生 `[System.Management.Automation.Language.Parser]::ParseInput()` 将用户输入的命令字符串解析为结构化的 JSON，再在 TypeScript 侧转换为强类型的 `ParsedPowerShellCommand`。

该文件承担以下职责：
1. **跨进程 AST 解析**：内嵌完整的 PowerShell 解析脚本，避免磁盘 I/O 依赖；
2. **安全特征提取**：从 AST 中识别 scriptblock、子表达式 `$(...)`、可扩展字符串 `"...$()..."`、成员调用 `.Method()`、变量赋值、重定向、`--%` stop-parsing token 等；
3. **命令元数据归一化**：处理模块前缀剥离、别名解析、命令名分类（cmdlet / application / unknown）、引号剥离；
4. **解析结果缓存**：使用 LRU memoization 避免对同一命令重复 spawn pwsh；
5. **平台感知的命令长度限制**：针对 Windows CreateProcess 32K argv 限制计算精确的 UTF-8 字节预算，防止解析失败降级为 "ask" 从而绕过安全拒绝规则。

## 功能点目的

### 1. `parsePowerShellCommand`（带 LRU 缓存的异步解析入口）
对给定命令字符串返回 `ParsedPowerShellCommand`。若命令过长、pwsh 不可用、超时或返回非法 JSON，则返回 `valid: false` 的降级结果。

### 2. `getAllCommands` / `getAllCommandNames` / `getAllRedirections`
扁平化遍历所有 statement 及其 `nestedCommands`，为权限引擎提供统一的命令/重定向迭代接口。

### 3. `deriveSecurityFlags`
从解析结果中派生安全布尔标志（`hasScriptBlocks`、`hasSubExpressions`、`hasExpandableStrings`、`hasMemberInvocations`、`hasAssignments`、`hasSplatting`、`hasStopParsing`），供 `powershellSecurity.ts` 进行快速模式匹配。

### 4. `hasCommandNamed`
支持别名双向解析的 case-insensitive 命令存在性检查。例如查询 `Invoke-Expression` 可匹配到 `iex`。

### 5. `commandHasArg` / `commandHasArgAbbreviation` / `isPowerShellParameter`
参数级检查工具，支持 PowerShell 的参数缩写（unambiguous prefix）、冒号绑定值（`-Param:Value`）以及替代 dash 字符（en-dash、em-dash、horizontal bar、`/`）。

### 6. `COMMON_ALIAGES`
维护一个无原型链污染的常见别名表（`iex`→`Invoke-Expression`、`ls`→`Get-ChildItem` 等），是整个 PowerShell 权限系统的别名解析单一事实来源。

### 7. `getFileRedirections` / `isNullRedirectionTarget`
提取文件重定向（`>`、`>>`、`2>` 等），排除 `$null` 丢弃输出，用于判断命令是否涉及文件系统写入。

## 具体技术实现

### 7.1 内嵌 PowerShell 解析脚本 (`PARSE_SCRIPT_BODY`)
该脚本以字符串常量形式硬编码在 TS 文件中（约 11KB），无需外部 `.ps1` 文件。执行流程：
1. TS 侧将用户命令进行 **UTF-8 Base64 编码**，赋值给 `$EncodedCommand`；
2. 将 `$EncodedCommand = '...'` + `PARSE_SCRIPT_BODY` 拼接为完整脚本；
3. 脚本以 **UTF-16LE Base64** 通过 `pwsh -EncodedCommand` 执行，避免命令行转义问题和 here-string 注入攻击。

PS 脚本内部结构：
- **`Get-RawCommandElements`**：遍历 `CommandAst.CommandElements`，输出每个元素的 `.GetType().Name`、`.Extent.Text`、`.Value`（字符串常量时可用）、`.Expression.GetType().Name`（`CommandExpressionAst` 时）、以及 `.Argument`（冒号绑定参数的子节点）。
- **`Get-RawRedirections`**：提取 `FileRedirectionAst` 与 `MergingRedirectionAst` 的 append/fromStream/locationText。
- **`Get-SecurityPatterns`**：对每条 statement 使用 `FindAll()` 深度搜索 `MemberExpressionAst`、`SubExpressionAst`、`ArrayExpressionAst`、`ParenExpressionAst`、`ExpandableStringExpressionAst`、`ScriptBlockExpressionAst`，返回布尔标志。
- **变量提取**：`FindAll` 搜索 `VariableExpressionAst`，记录 `VariablePath` 与 `Splatted`。
- **类型字面量提取**：`FindAll` 搜索 `TypeExpressionAst` + `TypeConstraintAst`，输出 `TypeName.FullName`（如 `"int"`），供 CLM 允许列表检查。
- **`--%` token 检测**：遍历 token 流，匹配 `TokenKind::MinusMinus` 或 Generic token 中的 `--%`（兼容 PS5.1 与 PS7）。
- **`Process-BlockStatements`**：递归处理 `BeginBlock`、`ProcessBlock`、`EndBlock`、`CleanBlock`、`DynamicParamBlock` 中的 statements。
- **ParamBlock 安全补丁**：历史上 ParamBlock 是 named block 的**兄弟节点**而非子节点，导致其中命令（如 `param($x = (Remove-Item /))`）对下游检查不可见。脚本单独对 `$ast.ParamBlock` 执行 `FindAll` 并输出为 `ParamBlockAst` statement。
- **`using` / `#Requires` 检测**：`$ast.UsingStatements` 与 `$ast.ScriptRequirements` 直接读取，这些节点同样不在 block statements 中。

### 7.2 类型映射层（Raw → Parsed）
`transformRawOutput` → `transformStatement` → `transformCommandAst`/`transformExpressionElement` 的级联转换：
- `mapStatementType`：将 `.NET` AST 类型名映射到 TS `StatementType` union；
- `mapElementType`：将原始 AST 节点类型映射到 `CommandElementType`（`ScriptBlock`、`SubExpression`、`ExpandableString`、`MemberInvocation`、`Variable`、`StringConstant`、`Parameter`、`Other`）。**关键安全映射**：
  - `ArrayExpressionAst`（`@()`）被映射为 `SubExpression`，因为 `@(Remove-Item ./data)` 内部会执行副作用；
  - `ConstantExpressionAst`（数字字面量）映射为 `StringConstant`，避免无害数字参数被误判为危险；
  - `ParenExpressionAst` 映射为 `SubExpression`。
- `classifyCommandName`：正则 `^[A-Za-z]+-[A-Za-z][A-Za-z0-9_]*$` 判定 cmdlet；含 `.`/`\`/`/` 判定为 application；其余为 unknown。
- `stripModulePrefix`：剥离模块限定前缀（如 `Microsoft.PowerShell.Utility\Invoke-Expression` → `Invoke-Expression`），但**保留文件路径**（通过 `C:`、`\\`、`.\`、`..\` 前缀检测）。

### 7.3 `transformCommandAst` 中的深度安全处理
- **nameType 必须在 stripModulePrefix 之前计算**：`scripts\Get-Process` 原始含 `\`，`classifyCommandName` 返回 `application`；若先剥离则变成 `Get-Process`（`cmdlet`），导致 allowlist 误放行。实际 `name` 使用剥离后的值用于规则匹配（deny 规则 over-match 是 fail-safe）。
- **引号剥离**：`.value` 存在时优先使用（PowerShell 解析器已剥离引号并解析反引号转义）；否则对 `.text` 做 `replace(/^['"]|['"]$/g, '')`。
- **非 ASCII 字符防御**：若命令名含 `\u0080-\uFFFF`，强制将 `nameType` 设为 `application`。这是针对 .NET `OrdinalIgnoreCase` 可能将 `ſtart-proceſſ` 折叠为 `Start-Process` 的防御性措施（finding #31）。
- **children 映射**：仅对 `CommandParameterAst` 的 `.Argument` 填充一层子节点，使 TS 侧能检查 `-InputObject:$env:SECRET` 中冒号绑定值的实际类型，而无需再次解析文本。

### 7.4 Windows argv 长度限制（Finding #36）
Windows `CreateProcess` 有 32,767 字符限制。整个编码链为：
```
command (N UTF-8 bytes)
  → Base64 (~4N/3 chars)
  → $EncodedCommand = '...'\n
  → full script (wrapper + PARSE_SCRIPT_BODY)
  → UTF-16LE (2× bytes)
  → Base64 (4/3× chars)
  → -EncodedCommand argv
```
最终命令行长度 ≈ `argv_overhead + (wrapper + 4N/3 + body) × 8/3`。

代码中通过**基于 `PARSE_SCRIPT_BODY.length` 的数学推导**计算 `WINDOWS_MAX_COMMAND_LENGTH`（单位：UTF-8 字节），而非硬编码：
- `SCRIPT_CHARS_BUDGET = ((32767 - 200) * 3) / 8`
- `CMD_B64_BUDGET = SCRIPT_CHARS_BUDGET - PARSE_SCRIPT_BODY.length - ENCODED_CMD_WRAPPER`
- `WINDOWS_MAX_COMMAND_LENGTH = floor((CMD_B64_BUDGET * 3) / 4) - 100`

**安全关键**：比较时使用 `Buffer.byteLength(command, 'utf8')` 而非 `command.length`。因为 CJK 字符是 1 个 UTF-16 code unit 但 3 个 UTF-8 字节，若用 `.length` 会导致 Windows 上 `CreateProcess` 失败 → `valid:false` → 所有基于 AST 的 deny 规则降级为 ask（finding #36）。

Unix 平台保持原有 `4500` 字节限制不变，因为 Unix argv 限制为 ~128KB/2MB，收紧反而会导致原本可解析的复合命令中的 deny 规则降级。

### 7.5 缓存与错误处理
`parsePowerShellCommandCached` 使用 `memoizeWithLRU`（容量 256）。对**瞬态错误**（`PwshSpawnError`、`PwshError`、`PwshTimeout`、`EmptyOutput`、`InvalidJson`）在 Promise resolve 后主动从 cache 中 `delete`，使后续调用可以重试。确定性错误（`CommandTooLong`、语法错误）则保留在缓存中。

### 7.6 重定向提取的安全补丁
`transformStatement` 对 `PipelineAst` 做了双重重定向收集：
1. 遍历每个 pipeline element 的直接 `.redirections`；
2. 对整个 statement 做 `FindAll` 搜索 `FileRedirectionAst`，捕获隐藏在括号参数或 hashtable 中的重定向（如 `-Name:('payload' > file)`）。
通过 `(operator, target)` 元组去重，避免测试与消费者看到重复计数。

对于非 `PipelineAst` 的 statement（如 `IfStatementAst`），同样通过 `raw.redirections` 提取控制流内部的重定向（如 `if ($x) { 1 > /tmp/evil }`）。

## 关键代码路径与文件引用

| 导出符号 | 主要消费者 | 用途 |
|---------|-----------|------|
| `parsePowerShellCommand` | `src/tools/PowerShellTool/powershellPermissions.ts` | 主权限检查流程的 AST 来源 |
| `parsePowerShellCommand` | `src/tools/PowerShellTool/powershellSecurity.ts` | 间接通过 `powershellPermissions.ts` 传入 |
| `parsePowerShellCommand` | `src/utils/powershell/staticPrefix.ts` | 静态前缀提取需要先解析命令 |
| `parsePowerShellCommand` | `src/tools/SkillTool/SkillTool.ts` | 技能工具中的 PowerShell 命令解析 |
| `getAllCommands` | `src/tools/PowerShellTool/powershellPermissions.ts` | 步骤 4/5 的子命令拆分与独立权限检查 |
| `getAllCommands` | `src/tools/PowerShellTool/powershellSecurity.ts` | 各类安全检查（`checkInvokeExpression`、`checkDynamicCommandName` 等） |
| `deriveSecurityFlags` | `src/tools/PowerShellTool/powershellSecurity.ts` | 快速判断 scriptblock/子表达式/成员调用等 |
| `deriveSecurityFlags` | `src/tools/PowerShellTool/readOnlyValidation.ts` | `isReadOnlyCommand` 的前置过滤 |
| `COMMON_ALIASES` | `src/utils/powershell/dangerousCmdlets.ts` | `NEVER_SUGGEST` 别名展开 |
| `COMMON_ALIASES` | `src/tools/PowerShellTool/readOnlyValidation.ts` | `resolveToCanonical`、allowlist 查找 |
| `COMMON_ALIASES` | `src/tools/PowerShellTool/powershellSecurity.ts` | 别名解析（如 `iex` → `Invoke-Expression`） |
| `getFileRedirections` | `src/tools/PowerShellTool/powershellPermissions.ts` | 判断是否存在文件写入重定向 |
| `PS_TOKENIZER_DASH_CHARS` | `src/tools/PowerShellTool/powershellPermissions.ts` | 参数前缀检测（en-dash 等） |
| `classifyCommandName` | `src/tools/PowerShellTool/powershellPermissions.ts` | parse-failed fallback 中的 application 门控 |
| `stripModulePrefix` | `src/tools/PowerShellTool/powershellPermissions.ts` | 规则匹配时的名称归一化 |

## 依赖与外部交互

### 入依赖
- `execa`：用于 spawn `pwsh` 进程。
- `src/utils/debug.js` → `logForDebugging`：解析失败/超时时的诊断日志。
- `src/utils/memoize.js` → `memoizeWithLRU`：解析结果缓存。
- `src/utils/shell/powershellDetection.js` → `getCachedPowerShellPath`：获取 pwsh 可执行路径。
- `src/utils/slowOperations.js` → `jsonParse`：JSON 解析（可能带容错）。

### 出依赖
- `src/utils/powershell/dangerousCmdlets.ts`：被其导入 `COMMON_ALIASES`。
- `src/tools/PowerShellTool/powershellPermissions.ts`：主权限引擎。
- `src/tools/PowerShellTool/powershellSecurity.ts`：安全子检查集合。
- `src/tools/PowerShellTool/readOnlyValidation.ts`：只读白名单与 `resolveToCanonical`。
- `src/utils/powershell/staticPrefix.ts`：前缀提取器。
- `src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx`：UI 层间接调用前缀提取。

## 风险、边界与改进建议

### 风险
1. **pwsh 不可用的降级路径**：当 `pwsh` 未安装、超时或返回非法 JSON 时，`parsePowerShellCommand` 返回 `valid: false`。`powershellPermissions.ts` 对此有复杂的 fallback 扫描（基于正则的分割与规则匹配），但 fallback 无法提供 AST 级别的精确分析，可能导致：
   - deny 规则无法匹配到被注释/字符串包裹的命令；
   - 某些安全模式（如 `checkPermissionMode`）完全跳过。
2. **Windows 长度限制依然偏紧**：`WINDOWS_MAX_COMMAND_LENGTH` 经计算约为 ~1,092 UTF-8 字节（随脚本体大小浮动）。对于包含长路径或多语句的 PowerShell 脚本，很容易触发长度限制并降级为 ask。虽然这是 fail-safe，但可能影响合法使用。
3. **LRU 缓存容量 256**：在高频、长会话场景中，缓存可能频繁淘汰；且缓存键是原始命令字符串，语义等价但文本不同的命令（如空格数量不同）会占用多个缓存槽。
4. **PS1 脚本大小与维护**：`PARSE_SCRIPT_BODY` 是一个巨大的字符串模板，没有语法高亮、类型检查或单元测试覆盖。修改 PS 脚本时极易引入语法错误，且错误只能在运行时通过 `pwsh` 的 stderr 捕获。

### 边界
- **仅支持本地 pwsh**：无法解析 PowerShell 7+ 与 Windows PowerShell 5.1 之间的语法差异；某些仅在 5.1 中存在的 AST 类型可能映射为 `UnknownStatementAst`。
- **单层级 children**：`Get-RawCommandElements` 只对 `CommandParameterAst.Argument` 展开一层子节点，更深层的嵌套表达式（如 `-Param:(Get-Date).Year`）在 children 中只能看到 `MemberExpressionAst` 的类型，无法看到其内部结构。
- **无类型解析**：`typeLiterals` 输出的是用户书写的文本（如 `"int"`），而非解析后的 .NET 全限定类型名（`System.Int32`）。CLM 允许列表检查（`isClmAllowedType`）必须基于文本别名做匹配。

### 改进建议
1. **大命令输入的替代传输方式**：当命令超过 `WINDOWS_MAX_COMMAND_LENGTH` 时，当前直接拒绝解析。可考虑将命令写入临时文件，通过 `pwsh -File` 传递脚本路径，绕过 argv 限制。需要评估临时文件的安全清理与跨平台一致性。
2. **PS1 脚本的独立化与测试**：将 `PARSE_SCRIPT_BODY` 提取到独立的 `.ps1` 文件，在构建时通过 `fs.readFileSync` 或打包工具内联。这样可获得：
   - PowerShell 编辑器的语法高亮与 lint；
   - 独立的 Pester/PS 单元测试；
   - 更容易的代码审查。
3. **缓存键的规范化**：在缓存前对命令字符串做标准化（如统一空格、去除首尾空白），减少缓存碎片。
4. **增加解析健康度指标**：在 `logForDebugging` 之外，增加结构化的解析失败率/超时率指标，便于在遥测中发现 Windows CI 或特定用户环境中的稳定性问题。
5. **ParamBlock 与 UsingStatements 的进一步细化**：当前 `ParamBlockAst` 被当作一个普通 statement 处理，但权限引擎的 `isProvablySafeStatement` 要求 `PipelineAst` 且所有元素为 `CommandAst`，因此 ParamBlock 默认会被 fail-closed gate 捕获。这通常是安全的，但可能导致无害的 `param($x)` 声明也触发询问。可考虑在 `isReadOnlyCommand` 或 `powershellPermissions.ts` 中显式忽略空的 ParamBlock。
