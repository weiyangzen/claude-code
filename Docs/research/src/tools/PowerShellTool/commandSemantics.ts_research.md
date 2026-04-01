# commandSemantics.ts 研究文档

## 场景与职责

commandSemantics.ts 负责**解释外部可执行程序的退出码**，解决 PowerShell 中 `$LASTEXITCODE` 的语义歧义问题。

### 背景问题

PowerShell 有两种命令类型，它们的错误信号机制完全不同：

| 类型 | 错误信号 | 退出码行为 |
|------|----------|------------|
| PowerShell Cmdlet | 终止错误（`$?`） | 不设置 `$LASTEXITCODE` |
| 外部可执行程序 | 退出码 | 设置 `$LASTEXITCODE` |

**具体问题**：
- `Select-String`（grep 等效）：无匹配时返回 `$null`，退出码 0
- `Compare-Object`：无论是否不同，退出码都是 0
- `robocopy`：退出码 1 表示"文件复制成功"，8+ 才表示错误
- `grep.exe`：退出码 1 表示"无匹配"，2+ 才表示错误

如果没有本模块，`robocopy` 报告成功时会被误判为错误抛出 `ShellError`。

## 功能点目的

### 1. 命令语义定义（CommandSemantic）

**目的**：为特定命令定义退出码的解释逻辑。

```typescript
export type CommandSemantic = (
  exitCode: number,
  stdout: string,
  stderr: string,
) => {
  isError: boolean
  message?: string
}
```

**设计决策**：
- 接收 `stdout` 和 `stderr` 以便未来扩展（如根据输出内容判断）
- 返回 `message` 用于向用户解释退出码含义

### 2. 默认语义（DEFAULT_SEMANTIC）

**目的**：为未配置命令提供保守的默认行为。

```typescript
const DEFAULT_SEMANTIC: CommandSemantic = (exitCode) => ({
  isError: exitCode !== 0,
  message: exitCode !== 0 ? `Command failed with exit code ${exitCode}` : undefined,
})
```

### 3. GREP 语义（GREP_SEMANTIC）

**目的**：正确处理 grep/ripgrep/findstr 的退出码。

| 退出码 | 含义 | 处理 |
|--------|------|------|
| 0 | 找到匹配 | `isError: false` |
| 1 | 无匹配 | `isError: false`, `message: 'No matches found'` |
| 2+ | 错误 | `isError: true` |

**应用命令**：`grep`, `rg`, `findstr`

### 4. Robocopy 语义

**目的**：处理 Windows 最著名的退出码"陷阱"。

Robocopy 的退出码是位掩码：
```
0  = 无文件复制，无失败（已同步）
1  = 文件复制成功
2  = 检测到额外文件/目录
4  = 检测到不匹配的文件/目录
8  = 某些文件/目录无法复制（错误）
16 = 严重错误
```

**位运算解释**：
- `exitCode >= 8` → 至少有一个失败位被设置 → 错误
- `exitCode & 1` → 检查第 0 位（文件复制成功）

### 5. 命令提取（heuristicallyExtractBaseCommand）

**目的**：从复杂的 PowerShell 命令行中提取主命令名。

**处理步骤**：
1. 按 `;` 和 `|` 分割管道段
2. 取最后一个段（决定退出码的段）
3. 移除调用操作符 `&` 和 `.`
4. 移除路径，只保留基名
5. 移除 `.exe` 后缀
6. 转小写

**示例**：
```powershell
# 输入
& "C:\Program Files\Git\bin\grep.exe" pattern file.txt | Select-String test

# 提取结果
grep
```

## 具体技术实现

### 核心数据结构

```typescript
// 命令语义映射表
const COMMAND_SEMANTICS: Map<string, CommandSemantic> = new Map([
  ['grep', GREP_SEMANTIC],
  ['rg', GREP_SEMANTIC],
  ['findstr', GREP_SEMANTIC],
  ['robocopy', (exitCode) => ({ 
    isError: exitCode >= 8,
    message: /* 位运算解释 */
  })],
])
```

### 命令提取算法

```typescript
function extractBaseCommand(segment: string): string {
  // 1. 移除调用操作符
  const stripped = segment.trim().replace(/^[&.]\s+/, '')
  
  // 2. 获取第一个 token
  const firstToken = stripped.split(/\s+/)[0] || ''
  
  // 3. 移除引号
  const unquoted = firstToken.replace(/^["']|["']$/g, '')
  
  // 4. 提取基名（处理路径）
  const basename = unquoted.split(/[\\/]/).pop() || unquoted
  
  // 5. 移除 .exe 后缀并转小写
  return basename.toLowerCase().replace(/\.exe$/, '')
}

function heuristicallyExtractBaseCommand(command: string): string {
  const segments = command.split(/[;|]/).filter(s => s.trim())
  const last = segments[segments.length - 1] || command
  return extractBaseCommand(last)
}
```

### 主入口函数

```typescript
export function interpretCommandResult(
  command: string,
  exitCode: number,
  stdout: string,
  stderr: string,
): {
  isError: boolean
  message?: string
} {
  const baseCommand = heuristicallyExtractBaseCommand(command)
  const semantic = COMMAND_SEMANTICS.get(baseCommand) ?? DEFAULT_SEMANTIC
  return semantic(exitCode, stdout, stderr)
}
```

## 关键代码路径与文件引用

### 调用链

```
1. PowerShell 命令执行完成
   src/tools/PowerShellTool/PowerShellTool.ts: execute()
   → 获取 $LASTEXITCODE

2. 退出码解释
   src/tools/PowerShellTool/commandSemantics.ts: interpretCommandResult()
   → 提取命令名
   → 查找语义规则
   → 解释退出码

3. 结果处理
   → 如果 isError: true，可能抛出 ShellError
   → 如果 message 存在，显示给用户
```

### 相关文件

- `src/tools/PowerShellTool/PowerShellTool.ts` - 调用 interpretCommandResult
- `src/utils/shell/shellExecution.ts` - 可能复用相同模式

## 依赖与外部交互

### 无外部依赖

本模块是纯逻辑模块，不依赖项目其他部分。

### 被依赖方

```typescript
// PowerShellTool.ts
import { interpretCommandResult } from './commandSemantics.js'
```

## 风险、边界与改进建议

### 已知风险

1. **启发式提取的局限性**：
   - 引号内的 `|` 或 `;` 会被错误分割
   - 复杂表达式可能提取错误的命令名
   - 注释中的特殊字符可能干扰

   **安全缓解**：提取失败时回退到 DEFAULT_SEMANTIC（保守处理）

2. **遗漏的外部命令**：
   - 许多 Windows/Linux 工具也有非零成功退出码
   - 当前只覆盖最常见的几个

3. **PowerShell 版本差异**：
   - 不同版本的 PowerShell 可能对某些命令行为不同

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| 空命令 | 返回 DEFAULT_SEMANTIC |
| 只有空白 | 同上 |
| 无法识别的命令 | DEFAULT_SEMANTIC（非零即错误）|
| 负数退出码 | 按非零处理（isError: true）|
| 非常大的退出码 | 正常比较运算 |

### 故意省略的命令

代码注释明确说明以下命令**故意不添加**：

```typescript
// - 'diff': 歧义。PS 5.1 中 `diff` → Compare-Object（退出码 0）
//           PS Core / Git for Windows 中 `diff` → diff.exe（退出码 1 表示不同）
// - 'fc': 歧义。`fc` → Format-Custom（cmdlet）vs `fc.exe`（文件比较）
// - 'find': 歧义。Windows find.exe vs Unix find.exe 语义不同
// - 'test', '[': 非 PowerShell 构造
// - 'select-string', 'compare-object', 'test-path': 原生 cmdlet，退出码始终 0
```

### 改进建议

1. **扩展命令覆盖**：
   - `chkdsk`：退出码位掩码类似 robocopy
   - `xcopy`：特定退出码含义
   - `curl`/`wget`：HTTP 状态码映射
   - `npm`/`yarn`/`pnpm`：特定退出码

2. **更智能的命令提取**：
   - 使用 PowerShell AST 提取命令名（更准确）
   - 处理脚本块和子表达式

3. **用户可配置**：
   - 允许用户为自定义工具定义退出码语义
   - 配置文件或 UI 设置

4. **平台检测**：
   - Windows 和 Unix 上同名工具可能有不同语义
   - 根据平台选择正确的语义

### 测试要点

- 各种命令行格式的提取准确性
- 管道和语句分隔符处理
- 引号和转义字符
- 路径处理（Windows/Unix 风格）
- 各命令的边界退出码值
- 未配置命令的默认行为
