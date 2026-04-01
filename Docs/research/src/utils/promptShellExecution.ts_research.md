# promptShellExecution.ts 深度研究文档

## 场景与职责

`promptShellExecution.ts` 是 Claude Code 中负责**提示中嵌入 Shell 命令执行**的核心工具模块。它解析提示文本中的特殊语法（代码块和内联命令），执行对应的 Shell 命令，并将输出替换回提示中。

### 核心职责
1. **语法解析**：解析 `\`\`\`!` 代码块和 \`!`command\`` 内联命令
2. **命令执行**：通过 BashTool 或 PowerShellTool 执行命令
3. **权限检查**：执行前验证工具使用权限
4. **结果替换**：将命令输出替换回原始提示

### 使用场景
- Skill 文件中的动态内容生成
- 提示模板中的变量替换
- 自动化的环境信息收集

---

## 功能点目的

### 1. 支持的语法

**代码块**：
```markdown
```! echo "Hello World"
```
```

**内联命令**：
```markdown
当前目录是: !`pwd`
```

### 2. 正则表达式模式

```typescript
// 代码块: ```! command ```
const BLOCK_PATTERN = /```!\s*\n?([\s\S]*?)\n?```/g

// 内联: !`command`
const INLINE_PATTERN = /(?<=^|\s)!`([^`]+)`/gm
```

**性能优化**：
- 内联模式使用正向后行断言，比代码块模式慢约 100 倍
- 93% 的 skill 文件没有内联命令，因此使用 `includes('!`')` 预检查

### 3. Shell 选择

```typescript
const shellTool: PromptShellTool =
  shell === 'powershell' && isPowerShellToolEnabled()
    ? getPowerShellTool()
    : BashTool
```

**规则**：
- `shell === 'powershell'` 且 PowerShell 工具启用 → PowerShellTool
- 其他情况 → BashTool

### 4. 延迟加载

```typescript
const getPowerShellTool = (() => {
  let cached: PromptShellTool | undefined
  return (): PromptShellTool => {
    if (!cached) {
      cached = require('../tools/PowerShellTool/PowerShellTool.js').PowerShellTool
    }
    return cached
  }
})()
```

**目的**：避免在启动时加载 PowerShellTool 及其依赖（parser.ts、validators 等）。

---

## 具体技术实现

### 数据结构

```typescript
type ShellOut = { 
  stdout: string 
  stderr: string 
  interrupted: boolean 
}

type PromptShellTool = Tool & {
  call(input: { command: string }, context: ToolUseContext): Promise<{ data: ShellOut }>
}
```

### 核心流程

#### 1. 命令解析流程

```
executeShellCommandsInPrompt(text, context, slashCommandName, shell?)
├── 解析 BLOCK_PATTERN → 代码块匹配
├── 预检查 includes('!`') → 是否可能有内联命令
├── 解析 INLINE_PATTERN → 内联命令匹配
└── 合并所有匹配
```

#### 2. 命令执行流程

```
for each match:
├── 提取 command = match[1]?.trim()
├── hasPermissionsToUseTool(shellTool, { command }, context, ...)
│   └── 如果 behavior !== 'allow' → 抛出 MalformedCommandError
├── shellTool.call({ command }, context)
├── processToolResultBlock(shellTool, data, uuid)
├── 提取输出内容
└── result.replace(match[0], () => output)
```

#### 3. 输出格式化

```typescript
function formatBashOutput(stdout: string, stderr: string, inline = false): string {
  const parts: string[] = []
  
  if (stdout.trim()) {
    parts.push(stdout.trim())
  }
  
  if (stderr.trim()) {
    if (inline) {
      parts.push(`[stderr: ${stderr.trim()}]`)
    } else {
      parts.push(`[stderr]\n${stderr.trim()}`)
    }
  }
  
  return parts.join(inline ? ' ' : '\n')
}
```

#### 4. 错误处理

```typescript
function formatBashError(e: unknown, pattern: string, inline = false): never {
  if (e instanceof ShellError) {
    if (e.interrupted) {
      throw new MalformedCommandError(`Shell command interrupted for pattern "${pattern}": [Command interrupted]`)
    }
    const output = formatBashOutput(e.stdout, e.stderr, inline)
    throw new MalformedCommandError(`Shell command failed for pattern "${pattern}": ${output}`)
  }
  
  const message = errorMessage(e)
  const formatted = inline ? `[Error: ${message}]` : `[Error]\n${message}`
  throw new MalformedCommandError(formatted)
}
```

### 替换安全

```typescript
// 使用函数替换避免特殊字符问题
result = result.replace(match[0], () => output)
```

**原因**：`String.replace` 的字符串替换参数会解释 `$$`、`$&`、`$\``、`$'` 等特殊序列。Shell 输出（特别是 PowerShell 的 `$env:PATH`、`$$` 等）可能包含这些字符，直接使用字符串参数会导致内容损坏。

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `crypto` | UUID 生成 |
| `../Tool.js` | Tool 类型 |
| `../tools/BashTool/BashTool.js` | Bash 工具 |
| `./debug.js` | 调试日志 |
| `./errors.js` | 错误类型 |
| `./frontmatterParser.js` | Frontmatter 类型 |
| `./messages.js` | 消息创建 |
| `./permissions/permissions.js` | 权限检查 |
| `./toolResultStorage.js` | 工具结果处理 |
| `./shell/shellToolUtils.js` | PowerShell 工具启用检查 |

### 外部调用方

| 调用方 | 用途 |
|-------|------|
| `src/commands/commit-push-pr.ts` | Commit/PR 命令 |
| `src/skills/loadSkillsDir.ts` | Skill 加载 |
| `src/commands/commit.ts` | Commit 命令 |
| `src/commands/security-review.ts` | 安全审查 |
| `src/utils/plugins/loadPluginCommands.ts` | 插件命令加载 |

---

## 依赖与外部交互

### 运行时依赖

1. **Node.js 内置模块**：
   - `crypto` - UUID 生成

2. **内部模块**：
   - BashTool、PowerShellTool
   - 权限系统
   - 工具结果存储

### 安全考虑

1. **权限检查**：执行前必须通过 `hasPermissionsToUseTool`
2. **命令注入**：用户输入的命令直接执行，依赖权限系统控制
3. **输出处理**：使用函数替换避免替换字符串注入

---

## 风险、边界与改进建议

### 已知风险

1. **正则表达式性能**
   - 内联模式的正向后行断言在大文本上较慢
   - 虽然已优化，但极端情况仍可能影响性能

2. **并发执行**
   - 使用 `Promise.all` 并行执行所有命令
   - 命令之间无顺序保证，可能有竞态条件

3. **递归替换**
   - 命令输出中可能包含触发新替换的模式
   - 当前实现单次替换，但输出中的 `\`\`\`!` 可能意外触发

4. **错误上下文**
   - 错误信息包含原始匹配模式
   - 可能暴露敏感信息

### 边界条件

| 场景 | 处理 |
|------|------|
| 空命令 | 跳过执行 |
| 权限拒绝 | 抛出 MalformedCommandError |
| 命令中断 | 特殊错误消息 |
| 命令失败 | 包含 stdout/stderr 的错误 |
| 无匹配 | 返回原文本 |
| 输出包含 `$` | 使用函数替换正确处理 |

### 改进建议

1. **顺序执行选项**
   ```typescript
   export async function executeShellCommandsInPrompt(
     text: string,
     context: ToolUseContext,
     slashCommandName: string,
     shell?: FrontmatterShell,
     options?: { parallel?: boolean }
   ): Promise<string>
   ```

2. **超时控制**
   ```typescript
   // 添加命令执行超时
   const { data } = await Promise.race([
     shellTool.call({ command }, context),
     sleep(30000).then(() => { throw new TimeoutError() })
   ])
   ```

3. **输出大小限制**
   ```typescript
   // 限制替换输出大小
   const MAX_OUTPUT_SIZE = 10000
   if (output.length > MAX_OUTPUT_SIZE) {
     output = output.slice(0, MAX_OUTPUT_SIZE) + '\n[Output truncated...]'
   }
   ```

4. **更多语法支持**
   ```typescript
   // 支持变量赋值
   const VAR_PATTERN = /\$\{(\w+)\}/g
   
   // 支持条件执行
   const IF_PATTERN = /```!if\s+(.*?)\n([\s\S]*?)```/g
   ```

5. **调试模式**
   ```typescript
   // 显示执行的命令和输出
   if (process.env.CLAUDE_CODE_DEBUG_SHELL) {
     console.log(`[Shell] ${command}`)
     console.log(`[Output] ${output}`)
   }
   ```

### 维护注意事项

1. **正则表达式测试**：确保模式匹配各种边界情况
2. **安全审计**：定期审查命令执行逻辑
3. **性能监控**：监控大 skill 文件的解析性能
4. **Shell 兼容性**：测试 Bash 和 PowerShell 的行为一致性

### 使用示例

```typescript
import { executeShellCommandsInPrompt } from './promptShellExecution.js'

// 简单示例
const text = `
Current directory: \`!\`pwd\`\`
Files:
\`\`\`!
ls -la
\`\`\`
`

const result = await executeShellCommandsInPrompt(
  text,
  toolUseContext,
  'my-skill',
  'bash'
)

console.log(result)
// 输出:
// Current directory: /home/user/project
// Files:
// total 128
// drwxr-xr-x  5 user user  4096 Jan  1 00:00 .
// ...
```
