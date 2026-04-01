# modeValidation.ts 研究文档

## 场景与职责

modeValidation.ts 实现 PowerShell 的**权限模式验证**，特别是 `acceptEdits` 模式下的自动批准逻辑。

### 权限模式概述

| 模式 | 行为 |
|------|------|
| `bypassPermissions` | 完全绕过权限检查 |
| `dontAsk` | 不询问，按默认规则处理 |
| `acceptEdits` | 自动允许文件系统修改命令 |
| 其他 | 需要显式权限检查 |

### 与 BashTool 的对应关系

本模块功能类似于 `BashTool/modeValidation.ts`，针对 PowerShell cmdlet 和 AST 特性进行了适配。

## 功能点目的

### 1. acceptEdits 模式自动批准

**目的**：在 `acceptEdits` 模式下，自动允许安全的文件系统修改命令，无需用户确认。

**允许的 Cmdlet**（Tier 3 简化）：
```typescript
const ACCEPT_EDITS_ALLOWED_CMDLETS = new Set([
  'set-content',    # 写入文件内容
  'add-content',    # 追加文件内容
  'remove-item',    # 删除文件/目录
  'clear-content',  # 清空文件内容
])
```

**设计决策**：
- 只包含简单的写入 cmdlet（第一个位置参数 = -Path）
- Tier 3 cmdlet（New-Item, Copy-Item, Move-Item 等）需要显式批准
- 别名通过 `resolveToCanonical` 自动解析

### 2. 符号链接创建检测（isSymlinkCreatingCommand）

**目的**：检测 `New-Item` 创建文件系统链接的操作，防止路径验证绕过。

**链接类型风险**：
| 类型 | 风险 |
|------|------|
| `SymbolicLink` | 目录/文件重解析点，相对路径解析到链接目标 |
| `Junction` | 目录重解析点（类似符号链接）|
| `HardLink` | 硬链接，别名同一 inode |

**检测逻辑**：
```typescript
export function isSymlinkCreatingCommand(cmd: { name: string, args: string[] }): boolean {
  // 1. 解析为规范名
  const canonical = resolveToCanonical(cmd.name)
  if (canonical !== 'new-item') return false
  
  // 2. 扫描参数查找 -ItemType 或 -Type
  for (let i = 0; i < cmd.args.length; i++) {
    const raw = cmd.args[i]!
    // 处理 Unicode dash、冒号绑定、反引号转义
    // ...
    if (LINK_ITEM_TYPES.has(val)) return true
  }
  return false
}
```

**参数处理**：
- Unicode dash 前缀（en-dash、em-dash、horizontal-bar）
- 冒号绑定值（`-ItemType:SymbolicLink`）
- 空格分隔值（`-ItemType SymbolicLink`）
- 反引号转义（`-Item`Type`）
- 引号包裹（`-ItemType:'SymbolicLink'`）

### 3. 权限模式检查（checkPermissionMode）

**目的**：主入口函数，根据当前权限模式决定是否自动批准命令。

**处理流程**：

```
1. 跳过 bypass 和 dontAsk 模式
   → 这些模式在其他地方处理

2. 非 acceptEdits 模式
   → 返回 'passthrough'（不处理）

3. 解析失败检查
   → 无法验证未解析的命令

4. 安全检查
   → 子表达式、脚本块、成员调用、展开字符串等
   → 这些需要显式批准

5. 复合命令保护
   a. CWD 变化 + 写入操作
      → Set-Location/Push-Location/Pop-Location + 写入 cmdlet
      → 路径验证使用陈旧的 CWD
   
   b. 链接创建
      → New-Item -ItemType SymbolicLink/Junction/HardLink
      → 路径验证无法跟踪刚创建的链接

6. 命令遍历
   对每个 pipeline segment 的每个 command：
   
   a. 非 CommandAst 元素
      → 表达式源（如字符串管道到 Remove-Item）
      → 无法静态知道路径值
   
   b. nameType === 'application'
      → 路径解析的命令（如 scripts\Remove-Item）
      → 实际运行脚本而非 cmdlet
   
   c. elementTypes 白名单检查
      → 只允许 StringConstant 和 Parameter
      → 拒绝 Variable、Other（HashtableAst 等）
   
   d. 冒号绑定表达式检查
      → -Path:$(1 > /tmp/x) 包含重定向
      → 无法静态验证
   
   e. 安全输出 cmdlet 跳过
      → Out-Null、Format-* 等不改变语义的命令
   
   f. 允许列表检查
      → 必须在 ACCEPT_EDITS_ALLOWED_CMDLETS 中
   
   g. argLeaksValue 检查
      → 参数是否包含变量、哈希表等不可验证值

7. 嵌套命令检查
   → 控制流语句（if、foreach 等）内部的命令
   → 同样的检查逻辑

8. 全部通过
   → 返回 'allow' 自动批准
```

## 具体技术实现

### 核心数据结构

```typescript
// acceptEdits 允许的 cmdlet
const ACCEPT_EDITS_ALLOWED_CMDLETS = new Set([
  'set-content',
  'add-content',
  'remove-item',
  'clear-content',
])

// 链接类型
const LINK_ITEM_TYPES = new Set(['symboliclink', 'junction', 'hardlink'])

// 参数缩写检测
function isItemTypeParamAbbrev(p: string): boolean {
  return (
    (p.length >= 3 && '-itemtype'.startsWith(p)) ||
    (p.length >= 3 && '-type'.startsWith(p))
  )
}
```

### 符号链接检测详解

```typescript
export function isSymlinkCreatingCommand(cmd: { name: string, args: string[] }): boolean {
  const canonical = resolveToCanonical(cmd.name)
  if (canonical !== 'new-item') return false
  
  for (let i = 0; i < cmd.args.length; i++) {
    const raw = cmd.args[i] ?? ''
    if (raw.length === 0) continue
    
    // 规范化 Unicode dash 和 forward-slash
    const normalized = PS_TOKENIZER_DASH_CHARS.has(raw[0]!) || raw[0] === '/'
      ? '-' + raw.slice(1)
      : raw
    const lower = normalized.toLowerCase()
    
    // 分割冒号绑定值
    const colonIdx = lower.indexOf(':', 1)
    const paramRaw = colonIdx > 0 ? lower.slice(0, colonIdx) : lower
    
    // 移除反引号转义
    const param = paramRaw.replace(/`/g, '')
    
    if (!isItemTypeParamAbbrev(param)) continue
    
    // 获取参数值
    const rawVal = colonIdx > 0
      ? lower.slice(colonIdx + 1)
      : (cmd.args[i + 1]?.toLowerCase() ?? '')
    
    // 清理值（反引号、引号）
    const val = rawVal.replace(/`/g, '').replace(/^['"]|['"]$/g, '')
    
    if (LINK_ITEM_TYPES.has(val)) return true
  }
  return false
}
```

### 复合命令保护详解

```typescript
// CWD 变化 + 写入操作检测
if (totalCommands > 1) {
  let hasCdCommand = false
  let hasSymlinkCreate = false
  let hasWriteCommand = false
  
  for (const seg of segments) {
    for (const cmd of seg.commands) {
      if (cmd.elementType !== 'CommandAst') continue
      if (isCwdChangingCmdlet(cmd.name)) hasCdCommand = true
      if (isSymlinkCreatingCommand(cmd)) hasSymlinkCreate = true
      if (isAcceptEditsAllowedCmdlet(cmd.name)) hasWriteCommand = true
    }
  }
  
  if (hasCdCommand && hasWriteCommand) {
    return {
      behavior: 'passthrough',
      message: 'Compound command contains a directory-changing command...'
    }
  }
  
  if (hasSymlinkCreate) {
    return {
      behavior: 'passthrough',
      message: 'Compound command creates a filesystem link...'
    }
  }
}
```

### elementTypes 白名单检查

```typescript
// 安全标志检查
if (cmd.elementTypes) {
  for (let i = 1; i < cmd.elementTypes.length; i++) {
    const t = cmd.elementTypes[i]
    
    // 只允许 StringConstant 和 Parameter
    if (t !== 'StringConstant' && t !== 'Parameter') {
      return {
        behavior: 'passthrough',
        message: `Command argument has unvalidatable type (${t})...`
      }
    }
    
    // 冒号绑定参数检查
    if (t === 'Parameter') {
      const arg = cmd.args[i - 1] ?? ''
      const colonIdx = arg.indexOf(':')
      if (colonIdx > 0 && /[$(@{[]/.test(arg.slice(colonIdx + 1))) {
        return {
          behavior: 'passthrough',
          message: 'Colon-bound parameter contains an expression...'
        }
      }
    }
  }
}
```

## 关键代码路径与文件引用

### 调用链

```
1. 权限检查入口
   src/tools/PowerShellTool/powershellPermissions.ts: powershellToolHasPermission()
   → 调用 checkPermissionMode()

2. acceptEdits 处理
   → 如果返回 'allow'，自动批准命令
   → 如果返回 'passthrough'，继续其他检查

3. 符号链接检测复用
   → 在复合命令保护中使用
   → 也在其他安全检查中可能使用
```

### 相关文件

- `src/tools/PowerShellTool/powershellPermissions.ts` - 调用方
- `src/tools/PowerShellTool/readOnlyValidation.ts` - 提供 isCwdChangingCmdlet, argLeaksValue 等
- `src/utils/powershell/parser.ts` - 提供 AST 解析和 deriveSecurityFlags

## 依赖与外部交互

### 外部依赖

```typescript
import type { ToolPermissionContext } from '../../Tool.js'
import type { PermissionResult } from '../../utils/permissions/PermissionResult.js'
import type { ParsedPowerShellCommand } from '../../utils/powershell/parser.js'
import {
  deriveSecurityFlags,
  getPipelineSegments,
  PS_TOKENIZER_DASH_CHARS,
} from '../../utils/powershell/parser.js'
import {
  argLeaksValue,
  isAllowlistedPipelineTail,
  isCwdChangingCmdlet,
  isSafeOutputCommand,
  resolveToCanonical,
} from './readOnlyValidation.js'
```

### 被依赖方

```typescript
// powershellPermissions.ts
import { checkPermissionMode, isSymlinkCreatingCommand } from './modeValidation.js'
```

## 风险、边界与改进建议

### 已知风险

1. **TOCTOU（检查时间到使用时间）**：
   - 路径验证和实际执行之间有时间窗口
   - 符号链接可能在此期间被创建
   - 缓解：复合命令链接创建检测

2. **AST 解析限制**：
   - 动态生成的路径无法静态验证
   - 变量展开在运行时发生

3. **参数绑定复杂性**：
   - PowerShell 的参数绑定规则复杂
   - 可能存在未覆盖的边界情况

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| 空命令 | 返回 'passthrough' |
| 解析失败 | 返回 'passthrough'（无法验证）|
| 非 acceptEdits 模式 | 快速返回 'passthrough' |
| 包含子表达式 | 返回 'passthrough'（需要批准）|
| 脚本块 | 同上 |
| 成员调用 | 同上 |
| 展开字符串 | 同上 |
| 变量赋值 | 同上 |

### 改进建议

1. **扩展允许列表**：
   - 考虑添加更多安全的写入 cmdlet
   - 需要仔细评估每个 cmdlet 的风险

2. **更智能的路径跟踪**：
   - 跟踪 Set-Location 的参数
   - 相对路径解析到实际目标

3. **符号链接目标验证**：
   - 检查 `-Value` 参数（链接目标）
   - 确保目标在允许范围内

4. **性能优化**：
   - 缓存 resolveToCanonical 结果
   - 提前退出（最明显的拒绝先检查）

5. **更多复合命令模式**：
   ```typescript
   // 考虑检测
   - New-PSDrive + 写入（新驱动器命名空间）
   - Import-Module + 写入（模块可能改变行为）
   - Set-Variable + 写入（变量可能影响路径）
   ```

### 测试要点

- 各种权限模式的行为
- acceptEdits 模式的批准和拒绝场景
- 符号链接创建的各种语法变体
- 复合命令保护（cd + 写入，链接创建）
- elementTypes 白名单边界
- 冒号绑定参数表达式
- 嵌套命令（控制流内部）
- 安全输出 cmdlet 的跳过
- argLeaksValue 的各种情况
