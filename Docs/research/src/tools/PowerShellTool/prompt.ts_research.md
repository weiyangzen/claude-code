# PowerShell Tool Prompt (`prompt.ts`)

## 场景与职责

`prompt.ts` 是 Claude Code 中 PowerShell 工具的提示词生成模块，负责为 AI 模型提供 PowerShell 工具的使用指南、语法说明和安全约束。该模块确保模型了解 PowerShell 的版本差异、最佳实践以及何时应该使用专门的文件操作工具而非 PowerShell。

**核心职责：**
1. 生成 PowerShell 工具的完整使用提示词
2. 提供版本特定的语法指导（PowerShell 5.1 vs 7+）
3. 指导模型正确使用 PowerShell 进行终端操作
4. 明确区分 PowerShell 与专用工具的边界

**在系统中的位置：**
- 被 PowerShell Tool 主模块调用获取提示词
- 与 `FileReadTool`、`FileEditTool`、`FileWriteTool`、`GlobTool`、`GrepTool` 的提示词模块交互
- 依赖 `powershellDetection.ts` 进行版本检测

## 功能点目的

### 1. 主提示词生成 (`getPrompt`)

```typescript
export async function getPrompt(): Promise<string>
```

**目的：** 生成完整的 PowerShell 工具使用指南。

**提示词结构：**
1. **工具用途说明** - 明确 PowerShell 工具的适用范围
2. **版本特定语法** - 根据检测到的 PowerShell 版本提供差异化指导
3. **执行前检查清单** - 目录验证、命令执行最佳实践
4. **语法速查** - 变量、转义、管道、字符串插值等
5. **交互式命令警告** - 会挂起的命令类型
6. **多行字符串传递** - here-string 语法
7. **使用说明** - 超时、后台任务、工具选择建议
8. **Git 命令指导** - 破坏性操作警告

### 2. 版本检测与指导 (`getEditionSection`)

**目的：** 解决 PowerShell 5.1 和 7+ 之间的语法差异问题。

**检测逻辑：**
```typescript
function getEditionSection(edition: PowerShellEdition | null): string
```

**PowerShell 5.1 (Windows PowerShell) 限制：**
- 不支持 `&&` 和 `||` 管道链操作符
- 不支持三元运算符 `?:`、空合并 `??`、空条件 `?.`
- 原生命令的 `2>&1` 重定向会产生 ErrorRecord 包装
- 默认文件编码为 UTF-16 LE (BOM)
- `ConvertFrom-Json` 返回 PSCustomObject，不支持 `-AsHashtable`

**PowerShell 7+ (PowerShell Core) 特性：**
- 支持 `&&` 和 `||` 管道链
- 支持三元、空合并、空条件运算符
- 默认文件编码为 UTF-8 (无 BOM)

**未知版本的保守策略：**
- 假设 Windows PowerShell 5.1 以确保兼容性
- 禁止使用 7+ 特有的语法特性

### 3. 后台任务指导 (`getBackgroundUsageNote`, `getSleepGuidance`)

**目的：** 指导模型正确使用 `run_in_background` 参数，避免不必要的 `Start-Sleep`。

**关键指导原则：**
- 仅在不需要立即结果时使用后台任务
- 不需要立即检查后台任务输出
- 禁止在失败命令上使用 sleep 循环重试
- 禁止对后台任务进行轮询
- 如需轮询外部进程，使用检查命令而非先 sleep

**环境控制：**
- 通过 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` 环境变量禁用后台任务提示

### 4. 工具选择边界

**目的：** 明确 PowerShell 与专用工具的边界，防止模型滥用 PowerShell。

**禁止用 PowerShell 完成的操作：**
| 操作 | 专用工具 | 原因 |
|------|----------|------|
| 文件搜索 | `GLOB_TOOL_NAME` (GlobTool) | 更高效、跨平台一致 |
| 内容搜索 | `GREP_TOOL_NAME` (GrepTool) | 更强大、支持正则 |
| 读取文件 | `FILE_READ_TOOL_NAME` (FileReadTool) | 更好的编码处理 |
| 编辑文件 | `FILE_EDIT_TOOL_NAME` (FileEditTool) | 结构化编辑、diff 支持 |
| 写入文件 | `FILE_WRITE_TOOL_NAME` (FileWriteTool) | 原子写入、备份支持 |
| 输出文本 | 直接输出 | 避免 Write-Output/Write-Host 包装 |

## 具体技术实现

### 超时配置

```typescript
// 文件: src/tools/PowerShellTool/prompt.ts:18-24
export function getDefaultTimeoutMs(): number {
  return getDefaultBashTimeoutMs()  // 复用 Bash 工具默认值
}

export function getMaxTimeoutMs(): number {
  return getMaxBashTimeoutMs()  // 复用 Bash 工具最大值
}
```

### 版本检测集成

```typescript
// 文件: src/tools/PowerShellTool/prompt.ts:73-76
export async function getPrompt(): Promise<string> {
  const backgroundNote = getBackgroundUsageNote()
  const sleepGuidance = getSleepGuidance()
  const edition = await getPowerShellEdition()  // 异步检测 PowerShell 版本
  // ...
}
```

### 提示词模板结构

```typescript
// 文件: src/tools/PowerShellTool/prompt.ts:78-144
return `Executes a given PowerShell command with optional timeout...

IMPORTANT: This tool is for terminal operations via PowerShell...

${getEditionSection(edition)}

Before executing the command, please follow these steps:

1. Directory Verification:
   - If the command will create new directories or files...

2. Command Execution:
   - Always quote file paths that contain spaces...

PowerShell Syntax Notes:
   - Variables use $ prefix...
   
Interactive and blocking commands (will hang...):
   - NEVER use \`Read-Host\`, \`Get-Credential\`...

Passing multiline strings...:
   - Use a single-quoted here-string...

Usage notes:
  - The command argument is required.
  - You can specify an optional timeout...
${backgroundNote ? backgroundNote + '\n' : ''}\
  - Avoid using PowerShell to run commands that have dedicated tools...
${sleepGuidance ? sleepGuidance + '\n' : ''}\
  - For git commands:
    - Prefer to create a new commit...`
```

## 关键代码路径与文件引用

### 导入依赖

```typescript
// 文件: src/tools/PowerShellTool/prompt.ts:1-16
import { isEnvTruthy } from '../../utils/envUtils.js'
import { getMaxOutputLength } from '../../utils/shell/outputLimits.js'
import {
  getPowerShellEdition,
  type PowerShellEdition,
} from '../../utils/shell/powershellDetection.js'
import {
  getDefaultBashTimeoutMs,
  getMaxBashTimeoutMs,
} from '../../utils/timeouts.js'
import { FILE_EDIT_TOOL_NAME } from '../FileEditTool/constants.js'
import { FILE_READ_TOOL_NAME } from '../FileReadTool/prompt.js'
import { FILE_WRITE_TOOL_NAME } from '../FileWriteTool/prompt.js'
import { GLOB_TOOL_NAME } from '../GlobTool/prompt.js'
import { GREP_TOOL_NAME } from '../GrepTool/prompt.js'
import { POWERSHELL_TOOL_NAME } from './toolName.js'
```

### 版本特定指导生成

```typescript
// 文件: src/tools/PowerShellTool/prompt.ts:51-71
function getEditionSection(edition: PowerShellEdition | null): string {
  if (edition === 'desktop') {
    return `PowerShell edition: Windows PowerShell 5.1 (powershell.exe)
   - Pipeline chain operators \`&&\` and \`||\` are NOT available...
   - Ternary (\`?:\`), null-coalescing (\`??\`)...`
  }
  if (edition === 'core') {
    return `PowerShell edition: PowerShell 7+ (pwsh)
   - Pipeline chain operators \`&&\` and \`||\` ARE available...
   - Ternary (\`$cond ? $a : $b\`), null-coalescing...`
  }
  return `PowerShell edition: unknown — assume Windows PowerShell 5.1...`
}
```

### 后台任务提示

```typescript
// 文件: src/tools/PowerShellTool/prompt.ts:26-44
function getBackgroundUsageNote(): string | null {
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS)) {
    return null
  }
  return `  - You can use the \`run_in_background\` parameter...`
}

function getSleepGuidance(): string | null {
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS)) {
    return null
  }
  return `  - Avoid unnecessary \`Start-Sleep\` commands:
    - Do not sleep between commands that can run immediately...
    - If your command is long running...`
}
```

## 依赖与外部交互

### 上游依赖

| 文件 | 依赖内容 | 用途 |
|------|----------|------|
| `envUtils.ts` | `isEnvTruthy()` | 环境变量布尔值解析 |
| `outputLimits.ts` | `getMaxOutputLength()` | 输出长度限制提示 |
| `powershellDetection.ts` | `getPowerShellEdition()`, `PowerShellEdition` | 版本检测 |
| `timeouts.ts` | `getDefaultBashTimeoutMs()`, `getMaxBashTimeoutMs()` | 超时配置 |
| `toolName.ts` | `POWERSHELL_TOOL_NAME` | 工具名称常量 |
| `FileEditTool/constants.ts` | `FILE_EDIT_TOOL_NAME` | 文件编辑工具名称 |
| `FileReadTool/prompt.ts` | `FILE_READ_TOOL_NAME` | 文件读取工具名称 |
| `FileWriteTool/prompt.ts` | `FILE_WRITE_TOOL_NAME` | 文件写入工具名称 |
| `GlobTool/prompt.ts` | `GLOB_TOOL_NAME` | Glob 工具名称 |
| `GrepTool/prompt.ts` | `GREP_TOOL_NAME` | Grep 工具名称 |

### 下游消费

| 消费方 | 消费方式 | 用途 |
|--------|----------|------|
| PowerShell Tool 主模块 | `getPrompt()` | 获取工具提示词 |

### 数据流

```
环境变量 + 系统检测
    ↓
getPowerShellEdition() → 'desktop' | 'core' | null
    ↓
getEditionSection() → 版本特定指导文本
    ↓
getPrompt() → 完整提示词
    ↓
PowerShell Tool → AI 模型
```

## 风险、边界与改进建议

### 已知风险

1. **版本检测延迟**
   - `getPowerShellEdition()` 是异步操作，可能在首次提示生成时引入延迟
   - 检测失败时降级到保守的 5.1 指导（fail-safe）

2. **提示词长度**
   - 完整提示词约 9.8KB，可能占用模型上下文窗口
   - 但相对于安全收益，此开销可接受

3. **环境变量依赖**
   - `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` 控制后台任务提示
   - 如果环境变量设置不一致，可能导致行为不一致

### 边界限制

1. **静态提示词**
   - 提示词在工具初始化时生成，无法根据具体命令动态调整
   - 复杂场景需要模型自行判断

2. **版本检测粒度**
   - 仅区分 5.1 和 7+，不区分 7.0、7.1、7.2 等次要版本
   - 某些特性（如 `&&` 链）在 7.0 就已引入，但提示词不区分

3. **平台假设**
   - 对 Windows PowerShell 5.1 的描述假设 Windows 环境
   - 在 Linux/macOS 上运行 PowerShell 时，某些指导可能不适用

### 改进建议

1. **动态提示词增强**
   - 根据具体命令类型追加针对性提示
   - 示例：如果命令包含 `git rebase`，追加交互式编辑器警告

2. **版本检测细化**
   - 检测具体版本号（7.0、7.1、7.2、7.3、7.4）
   - 针对不同版本提供精确的特性支持列表

3. **平台特定指导**
   - 检测运行平台（Windows/Linux/macOS）
   - 提供平台特定的路径格式和命令指导

4. **命令历史感知**
   - 如果模型最近犯过某类错误，在提示中强化相关指导
   - 示例：如果模型最近滥用了 `Start-Sleep`，强化后台任务指导

5. **交互式命令列表扩展**
   - 当前列表：`Read-Host`, `Get-Credential`, `Out-GridView`, `$Host.UI.PromptForChoice`, `pause`
   - 可补充：`vim`, `nano`, `less` 等需要终端控制的命令

6. **性能优化**
   - 缓存 `getPowerShellEdition()` 结果避免重复检测
   - 提示词模板可预编译

7. **国际化考虑**
   - 当前提示词为英文
   - 考虑根据用户语言设置提供本地化版本

### 安全相关提示词审查

提示词中包含以下安全相关指导：

1. **破坏性操作警告**
   ```
   Before running destructive operations (e.g., git reset --hard, 
   git push --force, git checkout --), consider whether there is 
   a safer alternative...
   ```

2. **钩子执行警告**
   ```
   Never skip hooks (--no-verify) or bypass signing (--no-gpg-sign, 
   -c commit.gpgsign=false) unless the user has explicitly asked for it.
   ```

3. **工具边界明确**
   ```
   DO NOT use it for file operations (reading, writing, editing, 
   searching, finding files) - use the specialized tools for this instead.
   ```

这些指导与代码层面的安全检查形成互补，但提示词指导依赖模型遵循，而代码检查是强制性的。
