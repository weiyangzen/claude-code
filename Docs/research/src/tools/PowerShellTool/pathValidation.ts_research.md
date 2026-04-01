# PowerShellTool/pathValidation.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`src/tools/PowerShellTool/pathValidation.ts` 是 Claude Code 中 PowerShell 工具的核心安全模块，负责**文件系统路径的静态验证**。它与 BashTool 的 `pathValidation.ts` 形成对称架构，但针对 PowerShell 特有的语法和 cmdlet 体系进行了深度适配。

### 1.2 核心职责
- **路径提取与解析**：从 PowerShell 命令的 AST（抽象语法树）中提取文件路径参数
- **权限边界校验**：验证路径是否位于允许的工作目录内
- **危险操作拦截**：阻止对系统关键路径的删除操作（如 `Remove-Item /`）
- **重定向目标验证**：检查文件重定向（`>`, `>>`）的目标路径安全性
- **复合命令处理**：处理包含多个语句或管道操作的复杂命令

### 1.3 安全模型定位
在 PowerShell 权限检查流程中，本模块位于**第 4.44 步**（`powershellPermissions.ts` 中的决策收集阶段）：
```
决策优先级：deny > ask > allow > passthrough
1. 前置 deny/ask 规则匹配
2. 安全标志检查（子表达式、脚本块等）
3. using 语句/requires 指令检查
4. 提供程序路径/UNC 扫描
5. 子命令 deny/ask 规则
6. cd+git 复合守卫
7. 裸 git 仓库守卫
8. git 内部路径写入守卫
9. 【本模块】路径约束检查 ← checkPathConstraints
10. 精确 allow 规则
11. 只读 allowlist
...
```

## 2. 功能点目的

### 2.1 主要功能点

| 功能点 | 目的 | 关键决策 |
|--------|------|----------|
| `checkPathConstraints` | 主入口：验证命令中所有路径参数 | deny/ask/passthrough |
| `extractPathsFromCommand` | 从解析后的命令元素中提取路径 | 返回路径列表+操作类型 |
| `validatePath` | 单一路径的完整验证流程 | allowed + resolvedPath + decisionReason |
| `isPathAllowed` | 权限规则匹配（deny/allow） | 基于 ToolPermissionContext |
| `isDangerousRemovalRawPath` | 危险删除路径检测 | 阻止删除 /, ~, /etc 等 |

### 2.2 CMDLET_PATH_CONFIG 设计

```typescript
const CMDLET_PATH_CONFIG: Record<string, CmdletPathConfig> = {
  'set-content': {
    operationType: 'write',
    pathParams: ['-path', '-literalpath', '-pspath', '-lp'],
    knownSwitches: ['-passthru', '-force', ...],
    knownValueParams: ['-value', '-encoding', ...],
  },
  // ... 更多 cmdlet
}
```

**设计原则**：
- **operationType**: 区分 read/write/create，决定权限检查类型（read/edit）
- **pathParams**: 明确哪些参数接收文件路径，用于路径提取
- **knownSwitches**: 开关参数（无值），避免误吞后续参数
- **knownValueParams**: 已知非路径值参数，避免误识别为路径
- **leafOnlyPathParams**: 仅接受简单文件名（如 New-Item -Name），拒绝路径分隔符
- **positionalSkip**: 跳过前 N 个位置参数（如 Invoke-WebRequest 的 URI）
- **optionalWrite**: 标记仅在有路径参数时才写入的命令（如 iwr 无 -OutFile 时只输出到管道）

## 3. 具体技术实现

### 3.1 关键数据结构

#### PathCheckResult 与 ResolvedPathCheckResult
```typescript
type PathCheckResult = {
  allowed: boolean
  decisionReason?: PermissionDecisionReason
}

type ResolvedPathCheckResult = PathCheckResult & {
  resolvedPath: string
}
```

#### CmdletPathConfig（详细）
```typescript
type CmdletPathConfig = {
  operationType: FileOperationType  // 'read' | 'write' | 'create'
  pathParams: string[]              // 路径参数名（小写带-前缀）
  knownSwitches: string[]           // 已知开关参数
  knownValueParams: string[]        // 已知值参数（非路径）
  leafOnlyPathParams?: string[]     // 仅叶子文件名参数
  positionalSkip?: number           // 跳过的位置参数数量
  optionalWrite?: boolean           // 写入是否可选（取决于路径参数存在性）
}
```

### 3.2 核心流程

#### 3.2.1 checkPathConstraints（主入口）
```
输入: { command: string }, ParsedPowerShellCommand, ToolPermissionContext, compoundCommandHasCd

1. 如果解析失败 → passthrough
2. 遍历所有 statements
   └── 对每个 statement 调用 checkPathConstraintsForStatement
3. 收集所有结果，优先级：deny > ask
4. 返回最终决策
```

#### 3.2.2 checkPathConstraintsForStatement（语句级验证）
```
输入: ParsedStatement, ToolPermissionContext, compoundCommandHasCd

1. 初始化 firstAsk（延迟 ask 决策）
2. 如果 compoundCommandHasCd → 设置 firstAsk（相对路径不可信）
3. 遍历 statement.commands
   ├── 检测表达式管道源（非 CommandAst）
   ├── 调用 extractPathsFromCommand 提取路径
   ├── 检查 hasUnvalidatablePathArg（无法静态验证的参数）
   ├── 检查 write 操作但无路径（强制 ask）
   ├── 遍历提取的路径
   │   ├── 危险删除检查（isDangerousRemovalRawPath）
   │   ├── 调用 validatePath 验证路径
   │   └── 处理验证结果（deny/ask/suggestions）
   └── 处理重定向目标（作为 'create' 操作）
4. 遍历 nestedCommands（控制流中的嵌套命令）
   └── 重复上述检查
5. 返回 firstAsk 或 passthrough
```

#### 3.2.3 extractPathsFromCommand（路径提取）
```
输入: ParsedCommandElement
输出: { paths, operationType, hasUnvalidatablePathArg, optionalWrite }

1. 解析命令名到规范形式（resolveToCanonical）
2. 查找 CMDLET_PATH_CONFIG
3. 如果无配置 → 返回空路径（operationType='read'）
4. 构建参数集合：switchParams + COMMON_SWITCHES, valueParams + COMMON_VALUE_PARAMS
5. 遍历 args
   ├── 如果是参数（isPowerShellParameter）
   │   ├── 处理冒号语法：-Path:value
   │   ├── 匹配 pathParams → 提取为路径
   │   ├── 匹配 leafOnlyPathParams → 仅接受简单文件名
   │   ├── 匹配 knownSwitches → 跳过（不消费值）
   │   ├── 匹配 knownValueParams → 消费值但不验证为路径
   │   └── 未知参数 → hasUnvalidatablePathArg = true（防御性设计）
   └── 否则（位置参数）
       └── 如果跳过位置参数已耗尽 → 提取为路径
6. 返回结果
```

#### 3.2.4 validatePath（单路径验证）
```
输入: filePath, cwd, toolPermissionContext, operationType
输出: ResolvedPathCheckResult

1. 清理路径：去除引号，展开波浪号（~）
2. 标准化反斜杠为斜杠（PowerShell Core 行为）
3. 安全检查（顺序重要）：
   ├── 包含反引号（`）→ 尝试剥离后检查 deny 规则 → ask
   ├── 包含 ::（提供程序限定路径）→ 尝试剥离后检查 deny 规则 → ask
   ├── UNC 路径（// 或 DavWWWRoot 或 @SSL@）→ 拒绝
   ├── 包含 $ 或 %（变量扩展）→ ask
   ├── 非文件系统提供程序路径（env:, HKLM: 等）→ ask
   └── glob 模式（*?[]）
       ├── write/create 操作 → 拒绝
       └── read 操作 → 检查基础目录的 deny 规则 → ask
4. 解析路径为绝对路径
5. 安全解析（safeResolvePath）处理符号链接
6. 调用 isPathAllowed 进行权限检查
```

#### 3.2.5 isPathAllowed（权限决策）
```
输入: resolvedPath, context, operationType, precomputedPathsToCheck

1. 确定 permissionType：read → 'read', write/create → 'edit'
2. 检查 deny 规则（matchingRuleForInput）→ 匹配则拒绝
3. 对于 write/create：
   ├── 检查内部可编辑路径（checkEditableInternalPath）→ 允许
   └── 安全检查（checkPathSafetyForAutoEdit）→ 不安全则拒绝
4. 检查是否在允许的工作目录内（pathInAllowedWorkingPath）
5. 对于 read：检查内部可读路径（checkReadableInternalPath）→ 允许
6. 对于 write（且不在工作目录）：检查沙箱写入白名单
7. 检查 allow 规则 → 匹配则允许
8. 默认 → 拒绝（allowed: false）
```

### 3.3 关键安全机制

#### 3.3.1 反引号转义处理
```typescript
// 反引号是 PowerShell 的转义字符，可绕过 Node.js 路径检查
if (normalizedPath.includes('`')) {
  const backtickStripped = normalizedPath.replace(/`/g, '')
  const denyHit = checkDenyRuleForGuessedPath(backtickStripped, ...)
  if (denyHit) {
    return { allowed: false, ... }  // deny 命中
  }
  return {
    allowed: false,
    decisionReason: {
      type: 'other',
      reason: 'Backtick escape characters in paths cannot be statically validated...'
    }
  }
}
```

#### 3.3.2 提供程序路径检测
```typescript
// 平台差异：Windows 需要 2+ 字符排除 C:, D: 等驱动器字母
// POSIX：任何 <letters>: 都是 PSDrive
const providerPathRegex =
  getPlatform() === 'windows' ? /^[a-z0-9]{2,}:/i : /^[a-z0-9]+:/i
```

#### 3.3.3 glob 模式处理
```typescript
// PowerShell 通配符：* ? [ ]
// 注意：{} 是字面字符，不是 bash 的 brace expansion
const GLOB_PATTERN_REGEX = /[*?[\]]/
```

#### 3.3.4 危险删除保护
```typescript
// 检查原始路径（pre-realpath）和解析后路径
// 防止 safeResolvePath 的规范化绕过检查（如 / → C:\）
if (isRemoval && isDangerousRemovalRawPath(filePath)) {
  return dangerousRemovalDeny(filePath)
}
// ... validatePath 后 ...
if (isRemoval && isDangerousRemovalPath(resolvedPath)) {
  return dangerousRemovalDeny(resolvedPath)
}
```

## 4. 关键代码路径与文件引用

### 4.1 调用关系图

```
powershellPermissions.ts (主权限检查)
    └── checkPathConstraints() [本文件导出]
        ├── checkPathConstraintsForStatement()
        │   ├── extractPathsFromCommand()
        │   │   ├── resolveToCanonical() [readOnlyValidation.ts]
        │   │   ├── isPowerShellParameter() [parser.ts]
        │   │   └── CMDLET_PATH_CONFIG [本文件常量]
        │   ├── validatePath()
        │   │   ├── expandTilde() [本文件]
        │   │   ├── checkDenyRuleForGuessedPath() [本文件]
        │   │   ├── safeResolvePath() [fsOperations.ts]
        │   │   └── isPathAllowed()
        │   │       ├── matchingRuleForInput() [filesystem.ts]
        │   │       ├── checkEditableInternalPath() [filesystem.ts]
        │   │       ├── checkPathSafetyForAutoEdit() [filesystem.ts]
        │   │       ├── pathInAllowedWorkingPath() [filesystem.ts]
        │   │       ├── checkReadableInternalPath() [filesystem.ts]
        │   │       └── isPathInSandboxWriteAllowlist() [pathValidation.ts]
        │   └── isDangerousRemovalRawPath() / dangerousRemovalDeny() [本文件]
        └── formatDirectoryList() [本文件]
```

### 4.2 关键文件引用

| 文件路径 | 引用内容 | 用途 |
|----------|----------|------|
| `src/Tool.ts` | `ToolPermissionContext` | 权限上下文类型 |
| `src/utils/permissions/filesystem.ts` | `matchingRuleForInput`, `checkEditableInternalPath`, `pathInAllowedWorkingPath` 等 | 核心权限检查函数 |
| `src/utils/permissions/PermissionResult.ts` | `PermissionResult` | 结果类型 |
| `src/utils/permissions/pathValidation.ts` | `isDangerousRemovalPath`, `isPathInSandboxWriteAllowlist` | 共享路径验证 |
| `src/utils/powershell/parser.ts` | `ParsedCommandElement`, `ParsedPowerShellCommand`, `isPowerShellParameter` | AST 类型和解析工具 |
| `src/utils/fsOperations.ts` | `safeResolvePath`, `getFsImplementation` | 文件系统操作 |
| `src/utils/cwd.ts` | `getCwd` | 获取当前工作目录 |
| `src/utils/platform.ts` | `getPlatform` | 平台检测 |
| `./commonParameters.ts` | `COMMON_SWITCHES`, `COMMON_VALUE_PARAMS` | 通用参数 |
| `./readOnlyValidation.ts` | `resolveToCanonical` | 别名解析 |

### 4.3 代码行号参考

| 功能 | 起始行号 | 说明 |
|------|----------|------|
| CMDLET_PATH_CONFIG | 124 | cmdlet 配置表（~640 行） |
| matchesParam | 772 | 参数前缀匹配（PowerShell 缩写支持） |
| hasComplexColonValue | 793 | 冒号语法值复杂度检测 |
| expandTilde | 820 | 波浪号展开 |
| isDangerousRemovalRawPath | 840 | 原始路径危险删除检测 |
| dangerousRemovalDeny | 848 | 危险删除拒绝结果 |
| isPathAllowed | 863 | 核心权限决策 |
| checkDenyRuleForGuessedPath | 984 | 猜测路径的 deny 检查 |
| validatePath | 1013 | 单路径完整验证 |
| getGlobBaseDirectory | 1266 | glob 基础目录提取 |
| SAFE_PATH_ELEMENT_TYPES | 1294 | 安全路径元素类型 |
| extractPathsFromCommand | 1304 | 命令路径提取 |
| checkPathConstraints | 1528 | 主入口 |
| checkPathConstraintsForStatement | 1569 | 语句级验证 |

## 5. 依赖与外部交互

### 5.1 运行时依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `os.homedir()` | Node.js 内置 | 展开用户主目录 |
| `path.isAbsolute`, `path.resolve` | Node.js 内置 | 路径解析 |

### 5.2 项目内部依赖

```typescript
// 权限系统
import type { ToolPermissionContext } from '../../Tool.js'
import type { PermissionRule } from '../../types/permissions.js'
import {
  allWorkingDirectories,
  checkEditableInternalPath,
  checkPathSafetyForAutoEdit,
  checkReadableInternalPath,
  matchingRuleForInput,
  pathInAllowedWorkingPath,
} from '../../utils/permissions/filesystem.js'
import type { PermissionResult } from '../../utils/permissions/PermissionResult.js'
import { createReadRuleSuggestion } from '../../utils/permissions/PermissionUpdate.js'
import type { PermissionUpdate } from '../../utils/permissions/PermissionUpdateSchema.js'
import {
  isDangerousRemovalPath,
  isPathInSandboxWriteAllowlist,
} from '../../utils/permissions/pathValidation.js'

// PowerShell 解析器
import type {
  ParsedCommandElement,
  ParsedPowerShellCommand,
} from '../../utils/powershell/parser.js'
import {
  isNullRedirectionTarget,
  isPowerShellParameter,
} from '../../utils/powershell/parser.js'

// 工具函数
import { getCwd } from '../../utils/cwd.js'
import { getFsImplementation, safeResolvePath } from '../../utils/fsOperations.js'
import { containsPathTraversal, getDirectoryForPath } from '../../utils/path.js'
import { getPlatform } from '../../utils/platform.js'

// 同目录模块
import { COMMON_SWITCHES, COMMON_VALUE_PARAMS } from './commonParameters.js'
import { resolveToCanonical } from './readOnlyValidation.js'
```

### 5.3 与 BashTool pathValidation.ts 的关系

本文件明确遵循 "Follows the same patterns as BashTool/pathValidation.ts" 的设计原则：

| 方面 | BashTool | PowerShellTool |
|------|----------|----------------|
| 解析器 | 自定义 bash 解析器 | PowerShell AST 解析器（parser.ts） |
| 命令配置 | `PATH_EXTRACTORS` | `CMDLET_PATH_CONFIG` |
| 路径提取 | 基于位置/标志启发式 | 基于 AST elementTypes 和参数配置 |
| 参数处理 | 简单标志识别 | 支持冒号语法、Unicode 短划线、前缀缩写 |
| 危险删除 | `isDangerousRemovalPath` | `isDangerousRemovalRawPath`（增强版） |
| 复合命令 cd | `compoundCommandHasCd` | 相同概念，通过 `isCwdChangingCmdlet` 识别 |

## 6. 风险、边界与改进建议

### 6.1 已知风险与缓解

#### 6.1.1 路径解析 TOCTOU（Time-of-Check-Time-of-Use）
**风险**：验证时的路径解析与执行时的解析不一致（符号链接、竞争条件）
**缓解**：
- `safeResolvePath` 尝试解析符号链接
- `isCanonical` 标志用于检测解析是否完全
- 对 glob 模式采取保守策略（要求手动批准）

#### 6.1.2 复合命令 CWD 变更
**风险**：`Set-Location ./subdir; Get-Content ./file` 在验证时解析为 cwd/file，但执行时从 subdir 读取
**缓解**：
- `compoundCommandHasCd` 标志贯穿验证流程
- 检测到 cd 变更时强制 ask（即使路径在允许范围内）

#### 6.1.3 管道表达式源
**风险**：`'secret.txt' | Get-Content` 中路径来自管道而非参数
**缓解**：
- 检测非 CommandAst 的管道元素
- 尝试对管道源文本进行 deny 规则匹配
- 最终强制 ask

#### 6.1.4 冒号语法复杂性
**风险**：`-Path:$env:HOME/file` 包含变量扩展
**缓解**：
- `hasComplexColonValue` 检测表达式构造
- `children` 树查询验证 colon-bound 参数值类型

### 6.2 边界情况

| 场景 | 行为 | 理由 |
|------|------|------|
| `New-Item -Name ../evil` | hasUnvalidatablePathArg = true | -Name 是 leafOnlyPathParams，但 `../` 包含分隔符 |
| `Invoke-WebRequest https://example.com`（无 -OutFile）| optionalWrite 路径为空不触发 ask | optionalWrite: true，无路径参数时不写入磁盘 |
| `Remove-Item /` | deny | isDangerousRemovalRawPath 拦截 |
| `Get-Content *.txt`（glob）| ask | glob 模式无法静态验证符号链接 |
| `Get-Content -Path:env:HOME/file` | ask | 变量扩展无法静态解析 |

### 6.3 改进建议

#### 6.3.1 架构改进
1. **统一路径验证接口**：BashTool 和 PowerShellTool 的路径验证逻辑有大量重复，可考虑提取到共享模块
2. **增强符号链接追踪**：当前仅解析 glob 基础目录，可考虑在 acceptEdits 模式下进行更激进的预解析
3. **跨参数依赖追踪**：`New-Item -Path /allowed -Name ../secret` 中 -Name 应相对于 -Path 解析，当前实现有限支持

#### 6.3.2 安全加固
1. **PSDrive 动态追踪**：`New-PSDrive -Name Z -Root /etc` 后 `Get-Content Z:/secret` 当前被拦截（非文件系统提供程序），但理想情况下应能追踪到 /etc
2. **变量常量传播**：对简单字符串变量（`$path = '/etc/passwd'; Get-Content $path`）进行常量传播分析
3. **脚本块内容分析**：当前对所有脚本块采取保守策略，可考虑对简单脚本块进行浅层分析

#### 6.3.3 可维护性
1. **CMDLET_PATH_CONFIG 自动生成**：从 PowerShell 的 `Get-Command` 输出自动生成配置，减少手动维护
2. **测试覆盖率**：增加对边缘情况（Unicode 短划线、混合大小写、模块限定名）的单元测试
3. **文档同步**：注释中的行号引用（如 "L930"）容易过时，建议改用符号引用

### 6.4 性能考虑

- **解析缓存**：`parsePowerShellCommand` 有 LRU 缓存，但本模块每次调用都重新提取路径
- **重复解析**：`checkPathConstraints` 和 `checkPermissionMode` 都遍历 AST，可考虑合并遍历
- **文件系统调用**：`safeResolvePath` 涉及实际文件系统访问，在 glob 场景下可能多次调用

---

*研究文档生成时间：2026-04-01*
*基于代码版本：commit 研究时的 HEAD*
