# Research: src/utils/bash/shellQuote.ts

## 场景与职责

`shellQuote.ts` 是 Claude Code 中**所有 shell 引用与解析操作的统一安全网关**。它围绕 npm 包 `shell-quote` 构建了一层防御性包装，核心职责包括：
1. **安全地解析用户输入的 bash 命令**（`tryParseShellCommand`），将命令字符串转换为结构化的 token 数组。
2. **安全地将参数数组引用为 shell 命令字符串**（`quote` / `tryQuoteShellArgs`），防止参数注入。
3. **检测解析结果中的畸形 token**（`hasMalformedTokens`），防御利用 `shell-quote` 解析歧义进行的命令注入攻击（HackerOne #3482049）。
4. **检测 `shell-quote` 库自身的单引号反斜杠 bug**（`hasShellQuoteSingleQuoteBug`），防御 parser differential 攻击。

该模块被**大量上游模块依赖**，是安全验证链和命令执行链的**关键基础设施**。

## 功能点目的

### 1. `tryParseShellCommand(cmd, env?)`
- **目的**：安全地解析 shell 命令字符串，失败时返回错误信息而非抛出异常。
- **实现**：
  - 调用 `shell-quote` 的 `parse()` 函数。
  - 支持可选的环境变量解析函数/对象。
  - `catch` 所有异常，记录错误日志，返回 `{ success: false, error: string }`。
- **使用场景**：`commands.ts` 的命令拆分、`bashPermissions.ts` 的安全验证、`sedEditParser.ts` 的 sed 命令解析等。

### 2. `tryQuoteShellArgs(args)`
- **目的**：严格类型校验后的 shell 参数引用。
- **实现**：
  - 遍历参数数组，逐项校验类型。
  - 仅允许 `string`、`number`、`boolean`、`null`、`undefined`。
  - 对 `object`、`symbol`、`function` 等类型抛出明确错误。
  - 校验通过后调用 `shellQuoteQuote()`。
- **使用场景**：间接通过 `quote()` 被调用，是严格路径的入口。

### 3. `hasMalformedTokens(command, parsed)`
- **目的**：检测 `shell-quote` 解析结果是否包含畸形 token，这些畸形 token 通常意味着输入利用了解析器的歧义性。
- **背景**：HackerOne #3482049 报告了利用 JSON 风格字符串中的分号进行命令注入。例如 `echo {"hi":"hi;evil"}` 会被 `shell-quote` 将 `;` 解析为操作符，产生 `echo {hi:"hi` 等不完整的 token。
- **检测项**：
  1. **未终止引号**：遍历原始命令，按 bash 语义统计单双引号数量，奇数为畸形。
  2. **不平衡花括号**：token 中 `{` 和 `}` 数量不等。
  3. **不平衡圆括号**：`(` 和 `)` 数量不等。
  4. **不平衡方括号**：`[` 和 `]` 数量不等。
  5. **不平衡双引号**（token 内未转义）：`(?<!\\)"` 匹配到的数量为偶数。
  6. **不平衡单引号**（token 内未转义）：`(?<!\\)'` 匹配到的数量为偶数。
- **安全意义**：若检测到畸形，调用方（如 `bashSecurity.ts`）会拒绝该命令或将其标记为需要人工审批。

### 4. `hasShellQuoteSingleQuoteBug(command)`
- **目的**：检测利用 `shell-quote` 库对单引号内反斜杠处理错误的攻击。
- **背景**：在 bash 中，单引号内反斜杠是字面量（无转义意义），所以 `'\'` 表示字符串 `\` 且引号已关闭。但 `shell-quote` 错误地将 `\'` 视为转义序列，认为引号未关闭，导致后续 token 被合并。
  - 例：`git ls-remote 'safe\\' '--upload-pack=evil' 'repo'`
    - bash 解析：`["git","ls-remote","safe\\\\","--upload-pack=evil","repo"]`
    - `shell-quote` 解析：`["git","ls-remote","safe\\\\ --upload-pack=evil repo"]`
    - 这种 parser differential 可绕过基于 token 的安全检查。
- **检测逻辑**：
  - 按 bash 语义遍历命令字符，追踪单双引号状态。
  - 当遇到关闭单引号的位置时，检查紧邻的尾部反斜杠数量：
    - **奇数尾部反斜杠**：一定存在 bug（如 `'\'`、`'abc\'`）。
    - **偶数尾部反斜杠**：仅当命令中后续还存在另一个 `'` 时才存在 bug（因为 `shell-quote` 的 chunker 正则可能回溯错误并吞噬后面的引号）。

### 5. `quote(args)`
- **目的**：对外暴露的**主引用函数**，被数十个模块直接调用。
- **实现**：
  1. 先调用 `tryQuoteShellArgs([...args])` 走严格路径。
  2. 若严格路径成功，返回结果。
  3. 若失败（如参数中包含对象），进入**宽容回退路径**：
     - 将 `null`/`undefined` 转为字符串。
     - 将 `string`/`number`/`boolean` 转为字符串。
     - 对不支持的类型使用 `jsonStringify(arg)` 作为安全回退。
     - 再次调用 `shellQuoteQuote()`。
  4. 若宽容路径也失败，抛出 `Error('Failed to quote shell arguments safely')`。
- **安全注释**：代码中明确警告不能使用 `JSON.stringify` 作为 shell 引用的回退，因为 `JSON.stringify(['echo', '$(whoami)'])` 产生 `"echo" "$(whoami)"`，双引号不能阻止 shell 执行命令替换。当前实现通过 `jsonStringify` 仅用于将复杂对象转为可引用字符串，最终仍由 `shellQuoteQuote()` 处理。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 导出类型
```ts
export type { ParseEntry } from 'shell-quote'

export type ShellParseResult =
  | { success: true; tokens: ParseEntry[] }
  | { success: false; error: string }

export type ShellQuoteResult =
  | { success: true; quoted: string }
  | { success: false; error: string }
```

### `ParseEntry` 说明
`shell-quote` 的 `ParseEntry` 是 `string | { op: string } | { comment: string } | { pattern: string, op: 'glob' }` 等的联合类型。Claude Code 的多个安全模块通过检查 `typeof token === 'object' && 'op' in token` 来识别操作符、glob、注释等特殊 token。

### 关键正则与算法
- **未终止引号检测**：字符级状态机，处理反斜杠转义。
  ```ts
  let inSingle = false, inDouble = false
  for (let i = 0; i < command.length; i++) {
    const c = command[i]
    if (c === '\\' && !inSingle) { i++; continue }
    if (c === '"' && !inSingle) { doubleCount++; inDouble = !inDouble }
    else if (c === "'" && !inDouble) { singleCount++; inSingle = !inSingle }
  }
  ```
- **单引号 bug 检测**：
  ```ts
  let backslashCount = 0
  let j = i - 1
  while (j >= 0 && command[j] === '\\') {
    backslashCount++
    j--
  }
  if (backslashCount > 0 && backslashCount % 2 === 1) return true
  if (backslashCount > 0 && backslashCount % 2 === 0 && command.indexOf("'", i + 1) !== -1) return true
  ```

## 关键代码路径与文件引用

### 依赖
| 文件 | 引用符号 | 作用 |
|------|---------|------|
| `shell-quote` (npm) | `parse`, `quote` | 底层解析与引用 |
| `src/utils/log.ts` | `logError` | 错误日志记录 |
| `src/utils/slowOperations.ts` | `jsonStringify` | 宽容回退时的对象序列化 |

### 调用方（部分关键）
| 文件 | 引用符号 | 场景 |
|------|---------|------|
| `src/utils/bash/commands.ts` | `quote`, `tryParseShellCommand` | 命令拆分、重定向提取 |
| `src/utils/bash/shellCompletion.ts` | `quote`, `tryParseShellCommand` | 补全命令生成与输入解析 |
| `src/utils/bash/shellPrefix.ts` | `quote` | 前缀命令格式化 |
| `src/utils/bash/shellQuoting.ts` | `quote` | 高级引用逻辑 |
| `src/utils/bash/bashPipeCommand.ts` | `quote`, `tryParseShellCommand` | 管道命令处理 |
| `src/tools/BashTool/bashPermissions.ts` | `tryParseShellCommand` | 权限安全验证 |
| `src/tools/BashTool/pathValidation.ts` | `tryParseShellCommand` | 路径约束验证 |
| `src/tools/BashTool/sedEditParser.ts` | `tryParseShellCommand` | sed 命令解析 |
| `src/tools/BashTool/sedValidation.ts` | `tryParseShellCommand` | sed 验证 |
| `src/tools/BashTool/readOnlyValidation.ts` | `tryParseShellCommand` | 只读约束验证 |
| `src/utils/argumentSubstitution.ts` | `tryParseShellCommand` | 参数替换解析 |
| `src/utils/messages.ts` | `quote` | 消息中的命令引用 |
| `src/utils/bash/ShellSnapshot.ts` | `quote` | 快照命令构建 |
| `src/utils/shell/bashProvider.ts` | `quote` | bash 提供者命令构建 |
| `src/utils/crossProjectResume.ts` | `quote` | 跨项目恢复 |
| `src/utils/swarm/spawnUtils.ts` | `quote` | 多代理 spawn |
| `src/utils/swarm/backends/PaneBackendExecutor.ts` | `quote` | Pane 后端执行 |
| `src/tools/shared/spawnMultiAgent.ts` | `quote` | 多代理共享 spawn |

## 依赖与外部交互

- **`shell-quote` npm 包**：这是 Node.js 生态中广泛使用的 shell 解析库。Claude Code 对其进行了多层安全加固，但本质上仍受该库解析行为的约束。
- **HackerOne 安全报告**：模块中的 `hasMalformedTokens` 和 `hasShellQuoteSingleQuoteBug` 直接响应了已知的安全漏洞报告，体现了 defense-in-depth 的安全设计。

## 风险、边界与改进建议

### 风险
1. **`shell-quote` 库的未知 bug**：`hasMalformedTokens` 和 `hasShellQuoteSingleQuoteBug` 针对已知漏洞做了补丁，但 `shell-quote` 作为一个通用库，仍可能存在其他 parser differential。任何新发现的 bug 都可能绕过当前的安全检查。
2. **宽容回退路径的潜在风险**：`quote()` 的宽容路径使用 `jsonStringify` 处理对象。虽然最终仍由 `shellQuoteQuote()` 包裹，但如果 `jsonStringify` 产生包含未转义特殊字符的字符串（如 `$()`），而 `shellQuoteQuote()` 又因某种原因未正确处理，则存在注入风险。
3. **`hasMalformedTokens` 的性能**：对每个 token 使用多次正则匹配（`match(/{/g)` 等），对于超长命令或大量 token 可能有性能开销。虽然命令长度被 `parser.ts` 的 `MAX_COMMAND_LENGTH = 10000` 限制，但在高频调用场景下仍需注意。

### 边界
- `tryParseShellCommand` 的解析结果是一个**扁平 token 数组**，不包含 AST 级别的嵌套结构。对于复杂命令（如嵌套的命令替换 `$(echo $(date))`），token 数组无法表达层级关系。Claude Code 的 tree-sitter 路径（`parser.ts`）才是处理这类复杂结构的正确方式。
- `hasMalformedTokens` 的括号平衡检查是**基于单个 token 的局部检查**。跨 token 的括号不平衡（如 `echo {` 和 `}` 分属两个 token）不会被检测到，但这种情况在 bash 中本身可能是合法的（如 `echo {a,b}`）。
- `hasShellQuoteSingleQuoteBug` 仅检测**单引号相关的反斜杠 bug**，不涵盖双引号或其他引号类型的潜在 differential。

### 改进建议
1. **逐步迁移到 tree-sitter 路径**：对于命令解析，Claude Code 已经在 `parser.ts` 中集成了 tree-sitter。应考虑在更多场景下优先使用 tree-sitter，将 `shell-quote` 降级为 tree-sitter 不可用时的纯回退方案。
2. **引入 fuzz 测试**：针对 `hasMalformedTokens` 和 `hasShellQuoteSingleQuoteBug`，引入自动化 fuzz 测试，随机生成命令字符串并对比 bash、`shell-quote`、tree-sitter 三者的解析结果，提前发现新的 parser differential。
3. **移除或限制宽容回退路径**：当前 `quote()` 的宽容路径允许对象被 `jsonStringify` 后引用。建议进一步收紧：若参数数组中包含非原始类型，直接抛出错误，由调用方负责在传入前进行类型转换。这样可以消除 `jsonStringify` 回退的潜在风险。
4. **性能优化**：`hasMalformedTokens` 中的多次全局正则匹配可合并为单次遍历，减少重复扫描每个 token 的开销。
