# Research: `src/utils/powershell/dangerousCmdlets.ts`

## 场景与职责

`dangerousCmdlets.ts` 是 PowerShell 安全体系中的**权威危险 cmdlet 清单文件**。它的核心职责是：

1. **集中维护**所有被安全引擎认定为具有高风险的 PowerShell cmdlet 分类集合；
2. **消除重复清单之间的同步漂移**——同一组危险 cmdlet 被权限引擎（`powershellSecurity.ts`）和 UI 建议门控（`staticPrefix.ts`）共同消费；
3. 通过 `NEVER_SUGGEST` 集合，**阻止权限对话框向用户推荐过于宽泛的 wildcard 前缀规则**（例如 `Invoke-Expression:*`），防止一次误授权后永久绕过安全校验。

该文件本身不包含任何运行时逻辑，仅导出若干 `ReadonlySet<string>` 常量，是典型的"配置即代码"（configuration-as-code）安全清单。

## 功能点目的

### 1. `FILEPATH_EXECUTION_CMDLETS`
接受 `-FilePath`（或位置参数路径）并执行文件内容的 cmdlet。例如 `Start-Job -FilePath script.ps1` 会运行脚本，等效于任意代码执行。

### 2. `DANGEROUS_SCRIPT_BLOCK_CMDLETS`
接受 `{ ... }` scriptblock 参数并执行任意代码的 cmdlet。如 `Invoke-Command { rm / }`、`Invoke-Expression '...'`。

### 3. `MODULE_LOADING_CMDLETS`
加载/安装模块的 cmdlet（`Import-Module`、`Install-Module` 等）。`.psm1` 文件在导入时会执行其顶层脚本体，属于代码执行风险。

### 4. `NETWORK_CMDLETS`
网络请求 cmdlet（`Invoke-WebRequest`、`Invoke-RestMethod`）。wildcard 规则放行它们会允许无提示的下载/外泄。

### 5. `ALIAS_HIJACK_CMDLETS`
可篡改运行时命令解析状态的 cmdlet：`Set-Alias`、`New-Alias`、`Set-Variable`、`New-Variable`。攻击者可用其将 `Get-Content` 重绑定到 `Invoke-Expression`。

### 6. `WMI_CIM_CMDLETS`
WMI/CIM 进程生成入口，如 `Invoke-WmiMethod -Class Win32_Process -Name Create` 等价于 `Start-Process` 绕过。

### 7. `ARG_GATED_CMDLETS`
在 `CMDLET_ALLOWLIST` 中带有 `additionalCommandIsDangerousCallback` 的 cmdlet（如 `Select-Object`、`Where-Object`、`ipconfig`、`route` 等）。这些 cmdlet 对安全参数（纯字符串常量）自动放行，但对含有 scriptblock/变量/子表达式的参数会触发权限询问。若 UI 建议 `Cmdlet:*` wildcard，则 callback 永久失效——因此必须加入 `NEVER_SUGGEST`。

### 8. `NEVER_SUGGEST`
由上述所有集合外加小型静态列表（shell 解释器、跨平台代码执行命令如 `node`、`npm` 等）聚合而成，并**自动展开所有已知别名**（通过 `COMMON_ALIASES`）。任何出现在该集合中的命令名，都不会被权限对话框的静态前缀提取器推荐为 wildcard 规则。

## 具体技术实现

### 别名展开函数 `aliasesOf`
```ts
function aliasesOf(targets: ReadonlySet<string>): string[] {
  return Object.entries(COMMON_ALIASES)
    .filter(([, target]) => targets.has(target.toLowerCase()))
    .map(([alias]) => alias)
}
```
遍历 `parser.ts` 中的 `COMMON_ALIASES`，将目标 canonical cmdlet 的所有别名反向映射出来。例如 `iex` → `Invoke-Expression`，因此 `iex` 也会进入 `NEVER_SUGGEST`。

### `NEVER_SUGGEST` 的构建
使用 IIFE 立即执行，步骤：
1. 将各危险集合展开到 `core`；
2. 额外硬编码 `foreach-object`（其 `-MemberName` 位置参数无法在静态 AST 中可靠区分属性/方法调用）；
3. 从 `CROSS_PLATFORM_CODE_EXEC` 过滤掉含空格的条目（如 `npm run` 不会被单名查找命中）；
4. 对 `core` 调用 `aliasesOf(core)` 展开别名；
5. 返回最终的 `ReadonlySet<string>`。

## 关键代码路径与文件引用

| 导出符号 | 消费者文件 | 用途 |
|---------|-----------|------|
| `DANGEROUS_SCRIPT_BLOCK_CMDLETS` | `src/tools/PowerShellTool/powershellSecurity.ts` | `checkScriptBlockInjection` 判断 scriptblock 是否出现在危险 cmdlet 上 |
| `FILEPATH_EXECUTION_CMDLETS` | `src/tools/PowerShellTool/powershellSecurity.ts` | `checkDangerousFilePathExecution` 检测 `-FilePath` 向量 |
| `MODULE_LOADING_CMDLETS` | `src/tools/PowerShellTool/powershellSecurity.ts` | `checkModuleLoading` 拦截模块加载 |
| `NEVER_SUGGEST` | `src/utils/powershell/staticPrefix.ts` | `extractPrefixFromElement` 中直接拒绝建议这些命令的 wildcard 前缀 |
| `NEVER_SUGGEST` | 间接通过 `staticPrefix.ts` → `PowerShellPermissionRequest.tsx` | 控制权限对话框的 "Yes, and don't ask again for" 可编辑输入的默认值 |
| `ARG_GATED_CMDLETS` | 注释中声明需与 `readOnlyValidation.ts` 同步 | `test/utils/powershell/dangerousCmdlets.test.ts` 断言覆盖（注释引用） |

## 依赖与外部交互

### 入依赖
- `src/utils/permissions/dangerousPatterns.js` → `CROSS_PLATFORM_CODE_EXEC`：跨平台解释器列表（`node`、`python`、`ruby` 等）。
- `src/utils/powershell/parser.js` → `COMMON_ALIASES`：PowerShell 常见别名映射表，用于别名反向展开。

### 出依赖
- `src/tools/PowerShellTool/powershellSecurity.ts`：安全校验器消费多个危险集合。
- `src/utils/powershell/staticPrefix.ts`：UI 前缀提取器消费 `NEVER_SUGGEST`。

## 风险、边界与改进建议

### 风险
1. **清单遗漏**：新 cmdlet 或社区模块中的代码执行入口未加入对应集合，会导致安全引擎放行。例如任何接受 `-ScriptBlock` 的新 cmdlet 都需要同步更新 `DANGEROUS_SCRIPT_BLOCK_CMDLETS`。
2. **别名漂移**：`COMMON_ALIASES` 未覆盖的别名（如用户自定义别名、模块导出的别名）不会被 `aliasesOf` 展开，可能绕过 `NEVER_SUGGEST`。文件中已显式列出部分不在 `COMMON_ALIASES` 中的别名（如 `sal`、`nal`、`sv`、`nv`、`iwmi`）作为补偿。
3. **跨平台代码执行过滤**：`CROSS_PLATFORM_CODE_EXEC.filter(p => !p.includes(' '))` 会跳过 `npm run` 等多词条目，这意味着 `npm` 单独被加入 `NEVER_SUGGEST`，而 `npm run` 不会被单名查找命中——这是有意为之（注释说明 `NEVER_SUGGEST` 是单名查找），但 `npm:*` 仍可能因 `npm` 被加入而过于宽泛。

### 边界
- 该文件**不做任何运行时判断**，仅提供静态集合；实际的参数级危险判断（如 `Start-Job` 是否真带了 `-FilePath`）由 `powershellSecurity.ts` 负责。
- `ARG_GATED_CMDLETS` 的同步是**人工约定**：注释要求与 `readOnlyValidation.ts` 中所有带 `additionalCommandIsDangerousCallback` 的条目保持一致，但没有编译期或自动化测试强制约束（注释提到测试文件断言，但仓库中未找到该测试文件）。

### 改进建议
1. **自动化同步检查**：在 CI 中增加脚本，扫描 `readOnlyValidation.ts` 的 `CMDLET_ALLOWLIST` 中所有包含 `additionalCommandIsDangerousCallback` 的键，并断言它们都存在于 `ARG_GATED_CMDLETS` 中。
2. **模块别名扩展**：考虑在运行时通过 `Get-Alias` 动态获取当前会话的所有别名，补充静态 `COMMON_ALIASES` 的不足。但这需要额外的 pwsh spawn，性能与复杂度需权衡。
3. **注释中引用的测试文件不存在**：`test/utils/powershell/dangerousCmdlets.test.ts` 在仓库中未找到，建议补充单元测试以确保 `NEVER_SUGGEST` 覆盖所有预期别名和核心危险 cmdlet。
