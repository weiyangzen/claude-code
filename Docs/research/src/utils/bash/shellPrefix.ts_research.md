# Research: src/utils/bash/shellPrefix.ts

## 场景与职责

`shellPrefix.ts` 是一个**轻量级的 shell 前缀格式化工具**，专门用于处理 `CLAUDE_CODE_SHELL_PREFIX` 环境变量场景。当用户或系统配置要求通过特定 shell 包装器（wrapper）执行 bash 命令时（例如 `CLAUDE_CODE_SHELL_PREFIX="/usr/bin/bash -c"`），该模块负责：
1. 将包装器路径与其参数正确分离。
2. 对路径部分进行安全的 shell 转义。
3. 组装成最终可执行的命令字符串。

该模块是**命令执行链路的最外层包装器**，直接影响 `bashProvider.ts` 生成的最终 `commandString`。

## 功能点目的

### `formatShellPrefixCommand(prefix, command)`
- **目的**：接收一个可能包含可执行路径和参数的 shell 前缀，以及待执行的核心命令，返回正确转义后的完整命令字符串。
- **设计示例**：
  - `"bash"` + `"echo hello"` → `'bash' 'echo hello'`
  - `"/usr/bin/bash -c"` + `"echo hello"` → `'/usr/bin/bash' -c 'echo hello'`
  - `"C:\Program Files\Git\bin\bash.exe -c"` + `"echo hello"` → `'C:\Program Files\Git\bin\bash.exe' -c 'echo hello'`
- **核心逻辑**：
  1. 查找前缀中最后一个 `" -"`（空格后紧跟短横线）的位置。
  2. 若找到且位置大于 0：
     - 前半部分视为可执行路径（如 `/usr/bin/bash`）。
     - 后半部分视为固定参数（如 `-c`）。
  3. 使用 `quote()` 对可执行路径和核心命令分别进行 shell 安全转义。
  4. 拼接为：`${quote([execPath])} ${args} ${quote([command])}`。
  5. 若未找到 `" -"`：
     - 将整个前缀视为可执行路径，对前缀和核心命令都进行转义。
     - 拼接为：`${quote([prefix])} ${quote([command])}`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 实现代码（完整）
```ts
import { quote } from './shellQuote.js'

export function formatShellPrefixCommand(
  prefix: string,
  command: string,
): string {
  // Split on the last space before a dash to separate executable from arguments
  const spaceBeforeDash = prefix.lastIndexOf(' -')
  if (spaceBeforeDash > 0) {
    const execPath = prefix.substring(0, spaceBeforeDash)
    const args = prefix.substring(spaceBeforeDash + 1)
    return `${quote([execPath])} ${args} ${quote([command])}`
  } else {
    return `${quote([prefix])} ${quote([command])}`
  }
}
```

### 关键设计决策
- **按 `" -"` 分割而非简单按空格分割**：
  - 原因：Windows 路径（如 `C:\Program Files\Git\bin\bash.exe`）包含空格，简单按空格分割会把路径拆碎。
  - `" -"` 模式假设包装器参数总是以 `-` 开头（如 `-c`、`-l`、`-i`），这在 Unix shell 调用约定中是成立的。
- **`quote()` 的使用**：
  - 来自 `shellQuote.ts` 的 `quote()` 函数，具备严格的类型校验和回退机制。
  - 对路径中的空格、引号、特殊字符进行安全包裹。
- **固定参数原样透传**：
  - `args`（如 `-c`）不经过 `quote()`，直接拼接。这基于信任假设：`CLAUDE_CODE_SHELL_PREFIX` 来自环境变量，由用户或系统管理员配置，不应包含恶意注入内容。

## 关键代码路径与文件引用

### 依赖
| 文件 | 引用符号 | 作用 |
|------|---------|------|
| `src/utils/bash/shellQuote.ts` | `quote` | 对可执行路径和核心命令进行安全 shell 转义 |

### 调用方
| 文件 | 引用符号 | 场景 |
|------|---------|------|
| `src/utils/shell/bashProvider.ts` | `formatShellPrefixCommand` | 在 `buildExecCommand` 中，若 `process.env.CLAUDE_CODE_SHELL_PREFIX` 存在，则包装最终命令字符串 |
| `src/utils/hooks.ts` | `formatShellPrefixCommand` | （需进一步确认具体 hook，但 grep 显示存在引用） |

## 依赖与外部交互

- **`CLAUDE_CODE_SHELL_PREFIX` 环境变量**：这是该模块唯一的外部输入源。该变量允许用户指定一个自定义 shell 包装器来执行所有 bash 命令。典型使用场景包括：
  - 在容器或 CI 中强制使用特定路径的 bash。
  - 通过 `ssh host bash -c` 在远程主机上执行命令。
- **`shellQuote.ts`**：提供底层的安全转义能力。若 `quote()` 在未来被修改（如增加更严格的校验），`shellPrefix.ts` 的行为会同步变化。

## 风险、边界与改进建议

### 风险
1. **分割逻辑的局限性**：`lastIndexOf(' -')` 假设参数总是以 `-` 开头。若用户配置 `CLAUDE_CODE_SHELL_PREFIX="/path/to/my shell arg"`（参数不以 `-` 开头），整个字符串会被视为路径，导致生成的命令为 `'/path/to/my shell arg' 'command'`，这在语义上可能是正确的（将整个字符串作为可执行文件路径），但如果用户意图是 `my` 为可执行文件、`shell arg` 为参数，则会产生错误命令。
2. **参数注入的残余风险**：`args` 部分未经过 `quote()` 或任何校验，直接拼接。虽然 `CLAUDE_CODE_SHELL_PREFIX` 来自受信任的环境变量，但在某些多用户或 CI 场景中，环境变量可能被污染。例如 `CLAUDE_CODE_SHELL_PREFIX="bash -c ; rm -rf / #"` 会导致灾难性后果。
3. **Windows 路径与 Unix 参数混合**：在 Windows Git Bash 环境中，前缀可能是 `C:\Program Files\Git\bin\bash.exe -c`。`quote()` 能正确处理反斜杠和空格，但 `args` 中的 `-c` 是 Unix 参数，而底层实际执行环境是 Windows —— 这种跨平台混合依赖 `bashProvider.ts` 和 `Shell.ts` 的正确处理。

### 边界
- 仅处理**单个前缀字符串**，不支持多个包装器嵌套（如 `ssh host docker exec container bash -c`）。
- 不支持前缀中包含**多个独立参数**的精确拆分（如 `bash -l -c` 会被拆为 `bash` 和 `-l -c`，这在 `-c` 作为最后一个参数时通常正确，但不通用）。
- 对前缀中的**引号**不做解析：如果用户已经手动引号了前缀（如 `"bash -c"`），`quote()` 会再次对其整体加引号，可能导致双重引号。

### 改进建议
1. **更 robust 的参数拆分**：
   - 可考虑使用 `shell-quote` 的 `parse()` 功能来解析前缀字符串，从而正确区分路径和参数，而不是依赖简单的 `" -"` 启发式。
   - 例：`parse('/usr/bin/bash -c')` → `['/usr/bin/bash', '-c']`，然后分别对每项 `quote()`。
2. **对 `args` 也进行安全校验**：
   - 即使信任环境变量，也应对 `args` 部分做基本的危险字符检查（如分号、管道符、反引号），防止环境变量污染导致的命令注入。
3. **支持多参数精确拆分**：
   - 当前 `" -"` 只能把前缀分成两部分。若需要支持 `bash -l -c command`，应考虑解析出所有参数并分别转义。
4. **增加单元测试覆盖**：
   - 该模块目前无测试文件。建议补充对 Windows 路径、含空格路径、无参数前缀、多参数前缀等场景的测试。
