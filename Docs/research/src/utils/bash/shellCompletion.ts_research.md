# Research: src/utils/bash/shellCompletion.ts

## 场景与职责

`shellCompletion.ts` 为 Claude Code 的**输入框提供原生 shell 级别的自动补全建议**。当用户在 bash 模式下输入命令时，该模块：
1. 解析当前光标位置的输入上下文。
2. 判断需要补全的类型（命令名、环境变量、文件路径）。
3. 生成对应 shell（bash 或 zsh）的原生补全命令（`compgen` 或 zsh 参数展开）。
4. 通过 `Shell.exec()` 在子 shell 中执行补全命令，获取结果并格式化为 UI 建议项。

该模块架起了**用户输入与底层操作系统 shell 能力**之间的桥梁，使用户在 AI 助手的输入框中也能获得与终端一致的 tab 补全体验。

## 功能点目的

### 1. `getShellCompletions(input, cursorOffset, abortSignal)`
- **目的**：主入口函数，返回 `SuggestionItem[]`。
- **流程**：
  1. 获取当前 shell 类型（`getShellType()`）。
  2. 仅支持 `bash` 和 `zsh`，其余 shell 直接返回空数组。
  3. 调用 `parseInputContext()` 提取前缀和补全类型。
  4. 若前缀为空，返回空数组（避免无意义补全）。
  5. 调用 `getCompletionsForShell()` 执行补全命令。
  6. 为每个建议附加 `inputSnapshot` 元数据，便于调用方检测输入是否已变化。
- **容错**：全程 `try/catch`，失败时静默返回 `[]`，不阻塞主输入流程。

### 2. `parseInputContext(input, cursorOffset)`
- **目的**：精确判断光标处应补全什么。
- **逻辑分支**：
  - **变量补全**：若光标前文本以 `$` 开头且匹配变量名正则（`\$[a-zA-Z_][a-zA-Z0-9_]*$`），返回 `completionType: 'variable'`。
  - **Shell-quote 解析**：调用 `tryParseShellCommand(beforeCursor)` 解析已输入部分。
  - **解析失败回退**：按空格简单拆分，取最后一个 token；若是第一个 token 则判定为命令，否则根据前缀特征判定。
  - **新参数上下文**：若最后一个 token 后有空格（`beforeCursor.endsWith(' ')`），说明用户在开始新参数，默认补全文件（`completionType: 'file'`）。
  - **命令 vs 文件判定**：
    - 前缀含 `/`、`~`、`.` → 文件。
    - 前缀以 `$` 开头 → 变量。
    - 否则，若最后一个 string token 处于"新命令上下文"（行首或命令操作符后）→ 命令；否则 → 文件。

### 3. `getCompletionTypeFromPrefix(prefix)`
- **目的**：基于前缀字符特征快速判定补全类型。
- **规则**：
  - `$` 开头 → `variable`
  - 含 `/` 或以 `~`、`.` 开头 → `file`
  - 其他 → `command`

### 4. `getBashCompletionCommand(prefix, completionType)` / `getZshCompletionCommand(prefix, completionType)`
- **目的**：生成安全的原生 shell 补全命令。
- **bash 命令**：
  - 变量：`compgen -v ${quote([varName])} 2>/dev/null`
  - 文件：`compgen -f ${quote([prefix])} 2>/dev/null | head -15 | while IFS= read -r f; do [ -d "$f" ] && echo "$f/" || echo "$f "; done`
    - 目录追加 `/`，文件追加空格，防止用户需二次输入分隔符。
    - 使用 `while IFS= read -r` 防御文件名含换行符导致的命令注入。
  - 命令：`compgen -c ${quote([prefix])} 2>/dev/null`
- **zsh 命令**：
  - 变量：`print -rl -- \${(k)parameters[(I)${quote([varName])}*]} 2>/dev/null`
  - 文件：`for f in ${quote([prefix])}*(N[1,15]); do [[ -d "$f" ]] && echo "$f/" || echo "$f "; done`
    - 利用 zsh glob 的 `N`（无匹配时不报错）和 `[1,15]`（限制结果数）。
  - 命令：`print -rl -- \${(k)commands[(I)${quote([prefix])}*]} 2>/dev/null`

### 5. `getCompletionsForShell(shellType, prefix, completionType, abortSignal)`
- **目的**：调用 `Shell.exec()` 执行补全命令，并解析 stdout 为建议列表。
- **限制**：
  - 最大结果数 `MAX_SHELL_COMPLETIONS = 15`。
  - 超时 `SHELL_COMPLETION_TIMEOUT_MS = 1000` 毫秒。
  - 结果按换行分割、过滤空行、截取前 15 条。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 核心常量
```ts
const MAX_SHELL_COMPLETIONS = 15
const SHELL_COMPLETION_TIMEOUT_MS = 1000
const COMMAND_OPERATORS = ['|', '||', '&&', ';'] as const
```

### 输入上下文类型
```ts
type InputContext = {
  prefix: string
  completionType: ShellCompletionType // 'command' | 'variable' | 'file'
}
```

### 安全设计
- **参数注入防御**：所有补全命令中的用户输入均通过 `quote()`（来自 `shellQuote.ts`）进行转义，防止恶意前缀如 `; rm -rf /` 被直接注入 shell。
- **bash 文件补全的换行防御**：使用 `while IFS= read -r` 循环处理 `compgen -f` 输出，避免文件名中的换行符被误解析为多个建议项或命令分隔符。
- **zsh 文件补全的 glob 安全**：zsh 的 glob 扩展在语法层面是安全的（不会像 bash `for-in` 那样解析文件名中的特殊字符）。
- **AbortSignal 支持**：通过 `AbortController` 可在用户继续输入时取消 pending 的补全请求，避免 stale result 覆盖新输入。

### 命令操作符检测
```ts
function isCommandOperator(token: ParseEntry): boolean {
  return (
    typeof token === 'object' &&
    token !== null &&
    'op' in token &&
    (COMMAND_OPERATORS as readonly string[]).includes(token.op as string)
  )
}
```
该函数用于判断当前 token 是否处于新命令的起始位置（如 `git status && npm ` 中 `npm` 是命令补全）。

## 关键代码路径与文件引用

### 依赖
| 文件 | 引用符号 | 作用 |
|------|---------|------|
| `src/utils/bash/shellQuote.ts` | `quote`, `tryParseShellCommand`, `ParseEntry` | 安全转义与 shell 解析 |
| `src/utils/localInstaller.ts` | `getShellType` | 获取当前系统 shell 类型 |
| `src/utils/Shell.ts` | `Shell.exec` | 执行补全命令 |
| `src/utils/debug.ts` | `logForDebugging` | 调试日志 |

### 调用方
| 文件 | 引用符号 | 场景 |
|------|---------|------|
| `src/hooks/useTypeahead.tsx` | `getShellCompletions`, `ShellCompletionType` | 输入框 typeahead 系统的 bash 建议生成 |

## 依赖与外部交互

- **底层操作系统 shell**：实际补全能力完全依赖宿主系统的 `bash` 或 `zsh`。若系统未安装对应 shell，或 `compgen` 不可用，补全将静默失败。
- **`shell-quote` 库**（间接）：`shellQuote.ts` 封装了 `shell-quote` 的解析与转义能力。
- **`useTypeahead.tsx`**：调用方负责管理 `AbortController` 的生命周期、去抖（debounce）和 stale result 丢弃。

## 风险、边界与改进建议

### 风险
1. **Shell 注入的残余风险**：虽然 `quote()` 提供了保护，但 `compgen` 本身是一个复杂的 bash 内置命令，某些特殊构造的前缀（如含反引号、控制字符）在 `compgen` 内部可能有不可预期的行为。当前 `quote()` 的防御已被证明对常见攻击有效，但尚未经过形式化验证。
2. **超时与性能问题**：在大型文件系统目录下（如根目录 `/`），`compgen -f /` 可能产生数万结果，即使 `head -15` 截断，bash 仍需遍历所有文件。1 秒超时可能不够，导致补全频繁失败。
3. **zsh 与 bash 的行为差异**：`getShellType()` 返回的 shell 类型可能与 `Shell.exec()` 实际使用的 shell 不一致（例如用户配置 `CLAUDE_CODE_SHELL_PREFIX` 时）。当前代码在 `Shell.exec()` 中硬编码 `'bash'` 作为 shellType 参数，这可能与 zsh 补全命令不兼容。

### 边界
- **仅支持 bash 和 zsh**：fish、powershell、cmd 等 shell 无补全支持。
- **无子命令/选项级补全**：该模块只补全命令名、变量名和文件路径，**不理解具体命令的选项和子命令**（这部分由 Fig autocomplete 或其他 suggestion 系统负责）。
- **光标位置限制**：`parseInputContext` 假设光标位于行尾或某个 token 中间，对复杂的多行 heredoc、命令替换内部的补全不支持。

### 改进建议
1. **优化文件补全性能**：
   - 对 bash 文件补全，考虑使用 `compgen -f ${prefix} | head -15` 前先限制搜索范围（如 `compgen -G '${prefix}*' -f`）。
   - 对根目录等超大目录，可回退到 `ls` 或 `find -maxdepth 1` 以减少遍历开销。
2. **统一 shell 类型传递**：`Shell.exec(command, abortSignal, 'bash', ...)` 中的硬编码 `'bash'` 应与 `getShellType()` 的结果一致，避免 zsh 用户执行 bash 语法。
3. **增加目录优先排序**：当前结果按 `compgen` 原始顺序返回，可考虑将目录项排在文件项前面，提升用户体验。
4. **缓存常用补全结果**：对命令补全（`compgen -c`）这类不频繁变化的结果，可引入短期内存缓存，减少重复子进程开销。
