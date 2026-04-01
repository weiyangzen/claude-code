# PowerShellTool/readOnlyValidation.ts 深度研究文档

## 一、场景与职责

### 1.1 核心定位

`readOnlyValidation.ts` 是 PowerShell 工具的安全核心模块，负责实现 **PowerShell 命令的只读安全验证**。它是 Claude Code 在 Windows/PowerShell 环境下执行安全策略的关键防线，与 BashTool 的 `readOnlyValidation.ts` 形成对应关系。

### 1.2 主要职责

1. **只读命令白名单管理**：维护一个详尽的 PowerShell cmdlet 和外部命令白名单，定义哪些命令可以在无需用户确认的情况下自动执行
2. **命令安全分析**：基于 AST（抽象语法树）解析结果，判断命令是否真正只读、无副作用
3. **参数/标志验证**：验证命令使用的参数是否在安全允许范围内
4. **外部命令桥接**：为 git、gh、docker、dotnet 等外部命令提供共享的安全验证逻辑
5. **安全模式协调**：与 `acceptEdits` 模式配合，在特定场景下允许文件系统修改操作

### 1.3 使用场景

| 场景 | 处理方式 |
|------|----------|
| 用户执行 `Get-Process` | 通过白名单自动允许 |
| 用户执行 `git status` | 通过外部命令验证自动允许 |
| 用户执行 `Remove-Item /etc/passwd` | 被拒绝（危险删除） |
| 用户执行 `Write-Output $env:SECRET` | 被拒绝（参数值泄漏风险） |
| 用户执行 `Set-Content ./file.txt "data"` | 在 acceptEdits 模式下可能允许 |

---

## 二、功能点目的

### 2.1 CMDLET_ALLOWLIST - 核心白名单

**目的**：定义哪些 PowerShell cmdlet 被认为是只读的，可以自动执行。

**设计原则**：
- **最小权限原则**：只允许明确安全的 cmdlet 和参数组合
- **防御性编程**：使用 `Object.create(null)` 防止原型链污染攻击
- **大小写不敏感**：PowerShell cmdlet 不区分大小写，所有键存储为小写

**白名单分类**：

| 类别 | 示例 Cmdlet | 安全考量 |
|------|-------------|----------|
| 文件系统读取 | `Get-ChildItem`, `Get-Content`, `Test-Path` | 只读操作，但需验证路径参数 |
| 导航 | `Set-Location`, `Push-Location`, `Pop-Location` | 改变工作目录，影响后续相对路径解析 |
| 文本搜索 | `Select-String` | 纯读取操作 |
| 数据转换 | `ConvertTo-Json`, `ConvertFrom-Csv` | 纯内存转换，无副作用 |
| 对象操作 | `Get-Member`, `Compare-Object` | 只读检查 |
| 路径工具 | `Join-Path`, `Split-Path` | 字符串操作，不触及文件系统 |
| 系统信息 | `Get-Process`, `Get-Service`, `Get-ComputerInfo` | 只读查询 |
| 格式化输出 | `Format-Table`, `Format-List` | 需验证参数不含变量泄漏 |
| 网络信息 | `Get-NetAdapter`, `Get-NetIPAddress` | 只读查询 |
| Git/Docker/gh | `git`, `docker`, `gh` | 委托给外部命令验证 |

### 2.2 argLeaksValue - 参数值泄漏防护

**目的**：防止通过 cmdlet 参数间接泄漏敏感信息（如环境变量）。

**攻击场景示例**：
```powershell
# 攻击：通过错误消息泄漏 secret
Write-Output $env:AWS_SECRET_ACCESS_KEY
# 输出：无法找到进程 "sk-ant-..."

# 攻击：通过类型转换错误泄漏
Start-Sleep $env:SECRET
# 错误：无法将值 'sk-...' 转换为 System.Double
```

**防护机制**：
1. **ElementType 白名单**：只允许 `StringConstant`（字符串常量）和 `Parameter`（参数名）
2. **冒号绑定参数检查**：`-InputObject:$env:SECRET` 形式的参数需要检查 `.Argument` 子节点
3. **字符串考古回退**：对于不支持 `children` 的旧解析器，通过正则 `[$(@{[]` 检测危险字符

### 2.3 isReadOnlyCommand - 只读命令判断

**目的**：综合判断一个 PowerShell 命令是否可以被归类为只读。

**判断流程**：

```
1. 基础检查（空命令、解析失败）
   ↓
2. 安全标志检查（脚本块、子表达式、成员调用等）
   ↓
3. 复合命令 CWD 变化检查
   ↓
4. 逐语句检查：
   a. 重定向检查（禁止非 $null 的文件重定向）
   b. 首命令白名单检查
   c. 管道后续命令安全检查
   d. 嵌套命令检查
   ↓
5. 全部通过 → 返回 true
```

### 2.4 外部命令验证（git/gh/docker/dotnet）

**目的**：为常用外部 CLI 工具提供共享的只读命令验证。

**实现方式**：
- 复用 `src/utils/shell/readOnlyCommandValidation.ts` 中定义的命令配置
- 针对 PowerShell 环境做适配（如 `$` 变量检测）

**安全特性**：
- **Git**：检测危险全局标志（`-c`, `-C`, `--exec-path` 等），防止配置注入
- **gh**：限制为 ant 用户使用，检测 repo 参数中的 URL/SSH 格式防止数据外泄
- **Docker**：检测 `$` 变量，防止运行时注入
- **dotnet**：只允许 `--version`, `--info`, `--list-runtimes`, `--list-sdks`

---

## 三、具体技术实现

### 3.1 核心数据结构

#### CommandConfig 类型
```typescript
type CommandConfig = {
  safeFlags?: string[]           // 安全参数白名单
  allowAllFlags?: boolean        // 是否允许所有参数
  regex?: RegExp                 // 额外正则约束
  additionalCommandIsDangerousCallback?: (command: string, element?: ParsedCommandElement) => boolean
}
```

#### 关键集合定义
```typescript
// 安全输出 cmdlet（管道尾端无害）
const SAFE_OUTPUT_CMDLETS = new Set(['out-null'])

// 管道尾端 cmdlet（从 SAFE_OUTPUT_CMDLETS 迁移，需要 argLeaksValue 验证）
const PIPELINE_TAIL_CMDLETS = new Set([
  'format-table', 'format-list', 'format-wide', 'format-custom',
  'measure-object', 'select-object', 'sort-object', 'group-object',
  'where-object', 'out-string', 'out-host'
])

// 安全的外部 .exe（绕过 nameType='application' 检查）
const SAFE_EXTERNAL_EXES = new Set(['where.exe'])
```

### 3.2 关键算法流程

#### 3.2.1 resolveToCanonical - 别名解析

```typescript
export function resolveToCanonical(name: string): string {
  let lower = name.toLowerCase()
  // 仅对无路径分隔符的名称剥离 PATHEXT
  if (!lower.includes('\\') && !lower.includes('/')) {
    lower = lower.replace(WINDOWS_PATHEXT, '')  // .exe, .cmd, .bat, .com
  }
  const alias = COMMON_ALIASES[lower]
  if (alias) return alias.toLowerCase()
  return lower
}
```

**安全考量**：
- 路径分隔符检查防止 `scripts\git.exe` 被错误识别为 `git`
- PATHEXT 剥离使 `git.exe` 能匹配到 `git` 的安全配置

#### 3.2.2 isAllowlistedCommand - 白名单验证

```typescript
export function isAllowlistedCommand(
  cmd: ParsedCommandElement,
  originalCommand: string,
): boolean {
  // 1. nameType 安全检查
  if (cmd.nameType === 'application') {
    const rawFirstToken = cmd.text.split(/\s/, 1)[0]?.toLowerCase() ?? ''
    if (!SAFE_EXTERNAL_EXES.has(rawFirstToken)) return false
  }

  // 2. 查找白名单配置
  const config = lookupAllowlist(cmd.name)
  if (!config) return false

  // 3. 正则约束检查
  if (config.regex && !config.regex.test(originalCommand)) return false

  // 4. 额外回调检查
  if (config.additionalCommandIsDangerousCallback?.(originalCommand, cmd)) return false

  // 5. 参数 elementTypes 白名单检查
  //    - 只允许 StringConstant 和 Parameter
  //    - 冒号绑定参数检查子节点

  // 6. 外部命令特殊处理（git/gh/docker/dotnet）

  // 7. 参数标志验证
}
```

#### 3.2.3 hasSyncSecurityConcerns - 同步安全检查

**目的**：在 AST 解析不可用或作为快速预检时，通过正则检测危险模式。

**检测模式**：
| 模式 | 正则 | 风险 |
|------|------|------|
| 子表达式 | `/\$\(/` | 任意代码执行 |
| Splatting | `/(?:^|[^\w.])@\w+/` | 参数注入 |
| 成员调用 | `/\.\w+\s*\(/` | 任意 .NET 方法调用 |
| 赋值 | `/\$\w+\s*[+\-*/]?=/` | 状态修改 |
| 停止解析 | `/--%/` | 原始参数传递 |
| UNC 路径 | `/\\\\/` | 网络请求/凭证泄漏 |
| 静态方法调用 | `/::/` | 任意 .NET 静态方法调用 |

### 3.3 安全边界与特殊处理

#### 3.3.1 已移除的危险 Cmdlet

| Cmdlet | 移除原因 |
|--------|----------|
| `Select-Xml` | XXE 攻击：可通过 DOCTYPE 触发网络请求 |
| `Test-Json` | 外部 $ref：JSON Schema 可引用外部 URL |
| `Get-Command` | 模块自动加载：触发 .psm1 初始化代码执行 |
| `Get-Help` | 模块自动加载：同 Get-Command |
| `Get-WmiObject` | 网络请求：Win32_PingStatus 发送 ICMP |
| `Get-CimInstance` | 网络请求：可查询远程计算机 |

#### 3.3.2 特殊处理的 Cmdlet

```typescript
// ipconfig：macOS 上 `ipconfig set` 可修改网络配置
ipconfig: {
  safeFlags: ['/all', '/displaydns', '/allcompartments'],
  additionalCommandIsDangerousCallback: (_cmd, element) => {
    return (element?.args ?? []).some(a => !a.startsWith('/') && !a.startsWith('-'))
  }
}

// hostname：Linux/macOS 上 `hostname NAME` 可设置主机名
hostname: {
  safeFlags: ['-a', '-d', '-f', '-i', '-I', '-s', '-y', '-A'],
  additionalCommandIsDangerousCallback: (_cmd, element) => {
    return (element?.args ?? []).some(a => !a.startsWith('-'))
  }
}

// route：只允许 `route print`（显示路由表）
route: {
  safeFlags: ['print', 'PRINT', '-4', '-6'],
  additionalCommandIsDangerousCallback: (_cmd, element) => {
    const verb = element.args.find(a => !a.startsWith('-'))
    return verb?.toLowerCase() !== 'print'
  }
}
```

---

## 四、关键代码路径与文件引用

### 4.1 模块依赖图

```
readOnlyValidation.ts
├── 导入依赖
│   ├── ../../utils/powershell/parser.js
│   │   ├── ParsedCommandElement, ParsedPowerShellCommand (类型)
│   │   ├── COMMON_ALIASES, deriveSecurityFlags, getPipelineSegments
│   │   ├── isNullRedirectionTarget, isPowerShellParameter
│   │   └── getPlatform (来自 ../../utils/platform.js)
│   ├── ../../utils/shell/readOnlyCommandValidation.js
│   │   ├── DOCKER_READ_ONLY_COMMANDS, EXTERNAL_READONLY_COMMANDS
│   │   ├── GH_READ_ONLY_COMMANDS, GIT_READ_ONLY_COMMANDS
│   │   └── validateFlags, ExternalCommandConfig (类型)
│   └── ./commonParameters.js
│       └── COMMON_PARAMETERS (通用参数)
├── 导出接口
│   ├── argLeaksValue                    # 参数泄漏检测
│   ├── isReadOnlyCommand                # 主只读判断函数
│   ├── isAllowlistedCommand             # 白名单检查
│   ├── isCwdChangingCmdlet              # CWD 变化检测
│   ├── isSafeOutputCommand              # 安全输出 cmdlet 检测
│   ├── isAllowlistedPipelineTail        # 管道尾端检查
│   ├── isProvablySafeStatement          # 语句安全证明
│   ├── hasSyncSecurityConcerns          # 同步安全检查
│   └── resolveToCanonical               # 别名解析
└── 内部实现
    ├── lookupAllowlist                  # 白名单查找
    ├── isExternalCommandSafe            # 外部命令安全判断
    ├── isGitSafe/isGhSafe/isDockerSafe/isDotnetSafe  # 具体外部命令验证
```

### 4.2 关键调用链

#### 4.2.1 主权限检查流程

```
powershellToolHasPermission (powershellPermissions.ts:639)
├── powershellToolCheckExactMatchPermission (前缀/精确匹配)
├── matchingRulesForInput (规则匹配)
├── powershellCommandIsSafe (powershellSecurity.ts - 基础安全检查)
├── checkPermissionMode (modeValidation.ts - acceptEdits 模式)
│   └── isAcceptEditsAllowedCmdlet
│   └── isSymlinkCreatingCommand
├── isReadOnlyCommand (readOnlyValidation.ts:1168) [本文件入口]
│   ├── hasSyncSecurityConcerns (快速预检)
│   ├── deriveSecurityFlags (从 parser 获取安全标志)
│   ├── isCwdChangingCmdlet (CWD 变化检测)
│   └── isAllowlistedCommand (白名单验证)
│       ├── lookupAllowlist
│       ├── argLeaksValue (参数泄漏检查)
│       └── isExternalCommandSafe (外部命令验证)
└── checkPathConstraints (pathValidation.ts - 路径约束)
```

### 4.3 代码行号参考

| 功能 | 行号范围 | 说明 |
|------|----------|------|
| 类型定义 | 39-56 | CommandConfig 类型 |
| argLeaksValue | 76-115 | 参数值泄漏检测函数 |
| CMDLET_ALLOWLIST | 129-882 | 主白名单定义 |
| SAFE_OUTPUT_CMDLETS | 888-917 | 安全输出 cmdlet |
| PIPELINE_TAIL_CMDLETS | 931-943 | 管道尾端 cmdlet |
| SAFE_EXTERNAL_EXES | 964 | 安全外部 exe |
| resolveToCanonical | 984-996 | 别名解析 |
| isCwdChangingCmdlet | 1017-1033 | CWD 变化检测 |
| isSafeOutputCommand | 1038-1041 | 安全输出检测 |
| isAllowlistedPipelineTail | 1052-1061 | 管道尾端检查 |
| isProvablySafeStatement | 1072-1082 | 语句安全证明 |
| hasSyncSecurityConcerns | 1112-1159 | 同步安全检查 |
| isReadOnlyCommand | 1168-1305 | 主只读判断函数 |
| isAllowlistedCommand | 1310-1516 | 白名单验证函数 |
| 外部命令验证 | 1518-1823 | git/gh/docker/dotnet 验证 |

---

## 五、依赖与外部交互

### 5.1 上游依赖（被调用方）

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `../../utils/powershell/parser.js` | `ParsedCommandElement`, `ParsedPowerShellCommand`, `COMMON_ALIASES`, `deriveSecurityFlags`, `getPipelineSegments`, `isNullRedirectionTarget`, `isPowerShellParameter` | AST 解析结果类型和安全标志提取 |
| `../../utils/platform.js` | `getPlatform` | 平台检测（Windows/POSIX） |
| `../../utils/shell/readOnlyCommandValidation.js` | `DOCKER_READ_ONLY_COMMANDS`, `EXTERNAL_READONLY_COMMANDS`, `GH_READ_ONLY_COMMANDS`, `GIT_READ_ONLY_COMMANDS`, `validateFlags`, `ExternalCommandConfig` | 外部命令共享配置 |
| `./commonParameters.js` | `COMMON_PARAMETERS` | PowerShell 通用参数（-Verbose, -ErrorAction 等） |

### 5.2 下游调用方

| 模块 | 调用内容 | 用途 |
|------|----------|------|
| `powershellPermissions.ts` | `isReadOnlyCommand`, `isAllowlistedCommand`, `isCwdChangingCmdlet`, `isSafeOutputCommand`, `isAllowlistedPipelineTail`, `isProvablySafeStatement`, `resolveToCanonical`, `argLeaksValue` | 主权限检查流程 |
| `modeValidation.ts` | `isCwdChangingCmdlet`, `isSafeOutputCommand`, `isAllowlistedPipelineTail`, `resolveToCanonical`, `argLeaksValue` | acceptEdits 模式验证 |

### 5.3 与 BashTool 的关系

| 方面 | PowerShellTool | BashTool |
|------|----------------|----------|
| 白名单机制 | CMDLET_ALLOWLIST + 别名解析 | READONLY_COMMANDS |
| 参数验证 | elementTypes AST 检查 | 字符串正则检查 |
| 外部命令 | 复用 shared readOnlyCommandValidation | 直接定义 |
| CWD 变化检测 | isCwdChangingCmdlet | compoundCommandHasCd |
| 变量泄漏防护 | argLeaksValue | 基于 `$` 的正则检查 |

---

## 六、风险、边界与改进建议

### 6.1 已知风险与缓解措施

#### 风险 1：模块自动加载攻击
**描述**：攻击者预先将恶意模块放入 PSModulePath，通过 `Get-Command EvilCmdlet` 触发自动加载。

**缓解**：
- 已从白名单移除 `Get-Command` 和 `Get-Help`
- 这些 cmdlet 的 `-Name` 参数支持管道输入，可绕过参数级回调

#### 风险 2：CWD 变化导致的 TOCTOU
**描述**：`Set-Location ~; Get-Content ./.ssh/id_rsa` 中，验证器在旧 CWD 解析路径，PowerShell 在新 CWD 执行。

**缓解**：
- `isCwdChangingCmdlet` 检测 CWD 变化命令
- 复合命令包含 CWD 变化时，`isReadOnlyCommand` 返回 false
- `checkPathConstraints` 接收 `hasCdSubCommand` 参数进行额外检查

#### 风险 3：冒号绑定参数注入
**描述**：`-Path:(Remove-Item /etc)` 形式的参数在 AST 中显示为 `Parameter`，但包含可执行代码。

**缓解**：
- `argLeaksValue` 检查冒号后的值是否包含 `$`, `(`, `@`, `{`, `[`
- 使用 `children` 数组（如果可用）检查子节点类型

#### 风险 4：脚本路径欺骗
**描述**：`scripts\Get-Process` 剥离模块前缀后变成 `Get-Process`，但实际执行的是本地脚本。

**缓解**：
- `nameType` 从原始名称（剥离前）计算
- 包含路径分隔符的名称归类为 `'application'`
- `isAllowlistedCommand` 在 `nameType === 'application'` 时拒绝（除非在 `SAFE_EXTERNAL_EXES` 中）

### 6.2 边界情况

| 场景 | 行为 | 原因 |
|------|------|------|
| 解析失败（pwsh 不可用） | 降级为询问 | 无法验证 AST，保守处理 |
| 命令超过 MAX_COMMAND_LENGTH | 解析失败 | Windows CreateProcess 限制 |
| 模块限定名（`Module\Cmdlet`） | 可能过度提示 | 被归类为 'application' |
| Unicode  dash（en-dash/em-dash） | 正确处理 | `isPowerShellParameter` 检查 PS_TOKENIZER_DASH_CHARS |
| 空参数列表 | 通过 | 无参数可泄漏 |

### 6.3 改进建议

#### 建议 1：增强参数交叉验证
**问题**：`New-Item -Path /allowed -Name ../secret` 中，`-Name` 是相对于 `-Path` 解析的，但验证器可能分别处理。

**改进**：实现跨参数跟踪，理解参数间的依赖关系。

#### 建议 2：动态白名单更新
**问题**：白名单是静态代码，新发现的攻击向量需要代码修改。

**改进**：考虑支持配置化的白名单更新机制（需严格签名验证）。

#### 建议 3：更精确的脚本块分析
**问题**：`Where-Object { $true }` 中的脚本块内容目前被完全拒绝。

**改进**：对简单脚本块进行静态分析，识别真正危险的模式。

#### 建议 4：性能优化
**问题**：每个命令都进行完整的 AST 解析和多层验证。

**改进**：
- 对已知安全模式实现快速路径
- 缓存解析结果（已实现 LRU 缓存）
- 考虑预编译常用正则

#### 建议 5：更好的错误信息
**问题**：被拒绝时用户可能不清楚具体原因。

**改进**：在 `decisionReason` 中包含更详细的拒绝原因分类。

### 6.4 测试建议

| 测试类型 | 覆盖点 |
|----------|--------|
| 别名解析 | 验证所有 COMMON_ALIASES 正确映射 |
| 参数泄漏 | `Write-Output $env:X`, `Format-Table @{N='x';E={}}` |
| 冒号绑定 | `-Path:(1 > /tmp/x)`, `-InputObject:$env:X` |
| 路径欺骗 | `scripts\Get-Process`, `.\git.exe` |
| 外部命令 | `git -c core.pager=sh log`, `gh pr view --repo evil.com/x/y` |
| CWD 变化 | `Set-Location ~; Get-Content ./.ssh/id_rsa` |
| 复合命令 | `Get-Process | Stop-Process` |
| 重定向 | `Get-Process > /tmp/x` |

---

## 七、总结

`readOnlyValidation.ts` 是 PowerShell 工具安全架构的核心组件，通过多层防御机制确保只有真正只读的命令才能自动执行：

1. **白名单机制**：详尽的 cmdlet 和参数白名单
2. **AST 分析**：基于 PowerShell 原生解析器的深度分析
3. **参数验证**：elementTypes 白名单和冒号绑定检查
4. **外部命令桥接**：复用共享的外部命令安全配置
5. **同步预检**：正则快速检测危险模式

该模块与 `powershellPermissions.ts`、`modeValidation.ts`、`pathValidation.ts` 紧密协作，形成完整的 PowerShell 权限控制体系。
