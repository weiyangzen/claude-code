# pathValidation.ts 研究文档

## 场景与职责

`pathValidation.ts` 是 Bash 工具中**最关键的安全模块之一**，负责验证所有涉及文件系统路径的命令。它确保 Bash 命令只能访问和操作用户明确允许的目录，防止未经授权的文件访问、修改或删除。

**核心职责：**
1. 验证路径命令（cd, ls, rm, cp 等）的目标路径是否在允许范围内
2. 验证输出重定向的目标路径安全性
3. 提取和解析各种命令的路径参数
4. 检测和阻止危险删除操作（如 `rm -rf /`）
5. 处理复合命令中的 `cd` 与写操作组合的安全风险

## 功能点目的

### 1. 路径命令验证

支持验证 29 种路径相关命令：
- **导航**：`cd`
- **列表**：`ls`
- **搜索**：`find`, `grep`, `rg`
- **文件操作**：`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`
- **读取**：`cat`, `head`, `tail`, `sort`, `uniq`, `wc`, `cut`, `paste`, `column`, `tr`, `file`, `stat`, `diff`, `awk`, `strings`, `hexdump`, `od`, `base64`, `nl`
- **处理**：`sed`, `jq`
- **校验**：`sha256sum`, `sha1sum`, `md5sum`
- **版本控制**：`git`（仅 `--no-index` 模式）

### 2. 操作类型分类

每个命令被分类为读/写/创建操作：
```typescript
export const COMMAND_OPERATION_TYPE: Record<PathCommand, FileOperationType>
// 'read' | 'write' | 'create'
```

### 3. 危险路径保护

防止删除关键系统目录：
- 根目录 `/`
- 家目录 `~`
- 根目录的直接子目录（/usr, /etc 等）
- Windows 驱动器根目录（C:\, D:\）

### 4. 复合命令安全

阻止 `cd` 与写操作组合（防止路径解析绕过）：
```bash
# 被阻止的示例
cd .claude/ && mv test.txt settings.json
# 原因：路径验证基于原始 CWD，但实际写操作在 cd 后的目录
```

## 具体技术实现

### 关键数据结构

#### 1. 路径提取器映射

```typescript
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]>
```

每个命令有专门的路径提取逻辑：
- **简单命令**：使用 `filterOutFlags` 过滤选项
- **模式命令**（grep, rg, jq）：提取模式后的路径参数
- **特殊命令**（find, sed, tr）：自定义解析逻辑

#### 2. filterOutFlags（安全关键）

```typescript
function filterOutFlags(args: string[]): string[]
```

**安全特性**：正确处理 `--` 选项结束标记
```typescript
// SECURITY: Track `--` end-of-options delimiter
// 攻击示例：rm -- -/../.claude/settings.local.json
// 如果不处理 --，`-/../.claude/settings.local.json` 会被当作选项过滤掉
// 导致路径验证被跳过，文件被删除
```

### 核心算法流程

#### 1. checkPathConstraints（主入口）

```typescript
export function checkPathConstraints(
  input: z.infer<typeof BashTool.inputSchema>,
  cwd: string,
  toolPermissionContext: ToolPermissionContext,
  compoundCommandHasCd?: boolean,
  astRedirects?: Redirect[],
  astCommands?: SimpleCommand[],
): PermissionResult
```

**处理流程：**

1. **进程替换检测**
   ```typescript
   if (!astCommands && />>\s*>\s*\(|>\s*>\s*\(|<\s*\(/.test(input.command)) {
     return {
       behavior: 'ask',
       message: 'Process substitution requires manual approval',
     }
   }
   ```

2. **重定向验证**
   - 使用 AST 重定向（如果可用）或提取重定向
   - 验证每个重定向目标路径

3. **命令路径验证**
   - AST 路径：使用 `validateSinglePathCommandArgv`
   - 传统路径：使用 `validateSinglePathCommand`

#### 2. validateSinglePathCommand（传统路径）

```typescript
function validateSinglePathCommand(
  cmd: string,
  cwd: string,
  toolPermissionContext: ToolPermissionContext,
  compoundCommandHasCd?: boolean,
): PermissionResult
```

**安全处理步骤：**

1. **剥离包装器命令**
   ```typescript
   const strippedCmd = stripSafeWrappers(cmd)
   // 防止：timeout 10 rm -rf / 绕过验证
   // 剥离后：rm -rf /
   ```

2. **解析命令参数**
   ```typescript
   const extractedArgs = parseCommandArguments(strippedCmd)
   // 使用 shell-quote 解析，处理引号和 glob
   ```

3. **检查路径命令类型**
   ```typescript
   if (!SUPPORTED_PATH_COMMANDS.includes(baseCmd as PathCommand)) {
     return { behavior: 'passthrough' }
   }
   ```

4. **sed 特殊处理**
   ```typescript
   const operationTypeOverride =
     baseCmd === 'sed' && sedCommandIsAllowedByAllowlist(strippedCmd)
       ? ('read' as FileOperationType)
       : undefined
   // 只读 sed 命令降级为读操作
   ```

5. **路径验证**
   ```typescript
   const pathChecker = createPathChecker(baseCmd as PathCommand, operationTypeOverride)
   return pathChecker(args, cwd, toolPermissionContext, compoundCommandHasCd)
   ```

#### 3. createPathChecker（路径检查器工厂）

```typescript
export function createPathChecker(
  command: PathCommand,
  operationTypeOverride?: FileOperationType,
) {
  return (args, cwd, context, compoundCommandHasCd?): PermissionResult => {
    // 1. 正常路径验证
    // 2. 危险删除路径检查
    // 3. 建议生成（读/写操作的不同建议）
  }
}
```

#### 4. stripWrappersFromArgv（Argv 级包装器剥离）

**关键安全函数**，与 `stripSafeWrappers` 保持同步：

```typescript
export function stripWrappersFromArgv(argv: string[]): string[]
```

支持的包装器：
- `timeout`（完整 GNU 标志解析）
- `time`
- `nice`（支持 `-n N` 和 `-N` 形式）
- `nohup`
- `stdbuf`
- `env`

**安全注释**：
```typescript
// KEEP IN SYNC with:
//   - SAFE_WRAPPER_PATTERNS in bashPermissions.ts
//   - the wrapper-stripping loop in checkSemantics (src/utils/bash/ast.ts)
// If you add a wrapper in either, add it here too.
```

### 命令特定路径提取

#### find 命令

```typescript
find: args => {
  // 收集路径直到遇到非全局标志
  // 处理接受路径的标志：-newer, -samefile, -path 等
  // SECURITY: -- 后的所有参数都视为路径
}
```

#### sed 命令

```typescript
sed: args => {
  // 处理 -f/--file（脚本文件需要验证）
  // 跳过 -e/--expression（表达式，非文件）
  // 第一个非标志参数是脚本，其余是文件路径
}
```

#### git 命令

```typescript
git: args => {
  // 仅处理 git diff --no-index（比较任意文件）
  // 其他 git 命令在仓库内操作，受 git 自身安全模型约束
}
```

## 关键代码路径与文件引用

### 调用关系

```
bashPermissions.ts:bashToolHasPermission()
  └── checkPathConstraints() [本文件导出]
        ├── validateOutputRedirections()
        │     └── validatePath() [src/utils/permissions/pathValidation.ts]
        ├── validateSinglePathCommand() / validateSinglePathCommandArgv()
        │     ├── stripSafeWrappers() [bashPermissions.ts]
        │     ├── parseCommandArguments()
        │     └── createPathChecker()
        │           ├── validateCommandPaths()
        │           │     ├── PATH_EXTRACTORS[command]()
        │           │     └── validatePath()
        │           └── checkDangerousRemovalPaths()
        └── stripWrappersFromArgv() [本文件导出]
```

### 依赖文件

| 文件路径 | 依赖内容 | 用途 |
|---------|---------|------|
| `src/utils/bash/commands.ts` | `splitCommand_DEPRECATED`, `extractOutputRedirections` | 命令分割和重定向提取 |
| `src/utils/bash/shellQuote.ts` | `tryParseShellCommand` | Shell 命令解析 |
| `src/utils/bash/ast.ts` | `Redirect`, `SimpleCommand` | AST 类型 |
| `src/utils/permissions/pathValidation.ts` | `validatePath`, `isDangerousRemovalPath`, `expandTilde` | 核心路径验证 |
| `src/utils/permissions/filesystem.ts` | `allWorkingDirectories` | 工作目录查询 |
| `src/utils/permissions/PermissionResult.ts` | `PermissionResult` | 结果类型 |
| `src/utils/permissions/PermissionUpdate.ts` | `createReadRuleSuggestion` | 建议生成 |
| `./BashTool.js` | `BashTool` | 工具类型 |
| `./bashPermissions.ts` | `stripSafeWrappers` | 包装器剥离 |
| `./sedValidation.ts` | `sedCommandIsAllowedByAllowlist` | sed 验证 |

### 被引用位置

| 文件路径 | 引用方式 | 用途 |
|---------|---------|------|
| `src/tools/BashTool/bashPermissions.ts` | `import { checkPathConstraints } from './pathValidation.js'` | 主权限检查 |

## 依赖与外部交互

### 运行时依赖

```typescript
import { homedir } from 'os'
import { isAbsolute, resolve } from 'path'
import type { z } from 'zod/v4'
import type { ToolPermissionContext } from '../../Tool.js'
import type { Redirect, SimpleCommand } from '../../utils/bash/ast.js'
import { extractOutputRedirections, splitCommand_DEPRECATED } from '../../utils/bash/commands.js'
import { tryParseShellCommand } from '../../utils/bash/shellQuote.js'
import { getDirectoryForPath } from '../../utils/path.js'
import { allWorkingDirectories } from '../../utils/permissions/filesystem.js'
import type { PermissionResult } from '../../utils/permissions/PermissionResult.js'
import { createReadRuleSuggestion } from '../../utils/permissions/PermissionUpdate.js'
import type { PermissionUpdate } from '../../utils/permissions/PermissionUpdateSchema.js'
import { expandTilde, type FileOperationType, formatDirectoryList, isDangerousRemovalPath, validatePath } from '../../utils/permissions/pathValidation.js'
import type { BashTool } from './BashTool.js'
import { stripSafeWrappers } from './bashPermissions.js'
import { sedCommandIsAllowedByAllowlist } from './sedValidation.js'
```

### 数据流

```
用户请求执行 Bash 命令
         ↓
checkPathConstraints()
         ↓
┌─────────────────┬─────────────────┐
↓                 ↓                 ↓
重定向验证      命令路径验证      包装器剥离
         ↓                 ↓
validatePath()      PATH_EXTRACTORS
         ↓                 ↓
权限规则检查 ←──── 路径列表
         ↓
PermissionResult
```

## 风险、边界与改进建议

### 已知风险

1. **shell-quote 解析漏洞**
   ```typescript
   // SECURITY: shell-quote has a known single-quote backslash bug
   // that silently merges redirect operators into garbled tokens
   // 缓解：优先使用 AST 解析路径
   ```

2. **TOCTOU（检查时间到使用时间）**
   ```typescript
   // 路径验证后，符号链接目标可能改变
   // 缓解：validatePath 使用 safeResolvePath 解析符号链接
   ```

3. **复杂命令绕过**
   ```bash
   # 可能的绕过示例
   eval "rm -rf /"  # eval 不是路径命令，绕过验证
   sh -c "rm -rf /"  # sh 不是路径命令
   ```

### 边界情况

| 场景 | 处理方式 |
|------|---------|
| 空命令 | 返回 passthrough |
| 未知命令 | 返回 passthrough |
| 相对路径 | 基于 cwd 解析为绝对路径 |
| 符号链接 | 解析为目标路径后验证 |
| glob 模式 | 验证基础目录 |
| 环境变量 | 拒绝（`$VAR` 需要手动批准） |
| 波浪号扩展 | 仅支持 `~` 和 `~/`，其他变体拒绝 |

### 改进建议

1. **增强命令覆盖**
   ```typescript
   // 建议添加：
   'rsync': // 文件同步
   'rsync': // 文件同步
   'install': // 文件安装（带权限设置）
   'dd': // 磁盘操作（极高风险）
   ```

2. **参数级验证**
   ```typescript
   // 当前：cp 命令整体标记为 'write'
   // 建议：区分源（读）和目标（写）
   interface CommandPathSpec {
     command: string
     pathArgs: Array<{ position: number; type: FileOperationType }>
   }
   ```

3. **动态路径分析**
   ```typescript
   // 建议：使用 AST 进行更精确的路径提取
   // 当前：正则和字符串操作
   // 改进：tree-sitter AST 遍历
   ```

4. **速率限制**
   ```typescript
   // 建议：对批量文件操作添加速率限制
   interface RateLimit {
     maxOperationsPerSecond: number
     maxTotalOperations: number
   }
   ```

5. **审计和回滚**
   ```typescript
   // 建议：记录所有写操作，支持撤销
   interface FileOperation {
     command: string
     paths: string[]
     timestamp: number
     backupPath?: string
   }
   ```

### 安全关键代码审查

#### 高危代码段 1：-- 处理

```typescript
// filterOutFlags 中的 -- 处理是安全关键
if (arg === '--') {
  afterDoubleDash = true
  continue
}
// 确保 -- 后的所有参数都被视为路径，即使以 - 开头
```

#### 高危代码段 2：包装器剥离

```typescript
// stripWrappersFromArgv 必须与 checkSemantics 保持同步
// 不同步会导致：checkSemantics 看到剥离后的命令通过检查
// 但 pathValidation 看到未剥离的命令绕过验证
```

#### 高危代码段 3：cd + 写操作阻止

```typescript
// 阻止复合命令中的 cd + 写操作
if (compoundCommandHasCd && operationType !== 'read') {
  return {
    behavior: 'ask',
    message: 'Commands that change directories and perform write operations require explicit approval',
  }
}
```

### 测试建议

```typescript
describe('pathValidation', () => {
  it('blocks dangerous removal paths', () => {
    const result = checkDangerousRemovalPaths('rm', ['-rf', '/'], '/home/user')
    expect(result.behavior).toBe('ask')
  })
  
  it('handles -- correctly', () => {
    const args = filterOutFlags(['--', '-file-starting-with-dash'])
    expect(args).toContain('-file-starting-with-dash')
  })
  
  it('strips wrappers correctly', () => {
    const argv = ['timeout', '10', 'rm', 'file']
    expect(stripWrappersFromArgv(argv)).toEqual(['rm', 'file'])
  })
  
  it('blocks cd with write operation', () => {
    const result = validateCommandPaths(
      'mv', ['a', 'b'], '/home/user', context, true
    )
    expect(result.behavior).toBe('ask')
  })
})
```

### 架构评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 安全性 | ⭐⭐⭐⭐⭐ | 多层防御，边界情况处理完善 |
| 复杂性 | ⭐⭐⭐ | 1300+ 行，逻辑复杂 |
| 可维护性 | ⭐⭐⭐ | 需要与多个模块保持同步 |
| 可测试性 | ⭐⭐⭐⭐ | 函数式接口，但依赖较多 |
| 性能 | ⭐⭐⭐⭐ | 缓存和优化已实施 |

**总体评价**：这是系统中最关键的安全模块之一，实现了多层防御机制。代码复杂度高，需要严格的代码审查和全面的测试覆盖。
