# bashCommandHelpers.ts 研究文档

## 场景与职责

`bashCommandHelpers.ts` 是 Bash 工具**权限检查链路中的操作符/复合命令分析层**，位于 `bashPermissions.ts` 的下游、具体权限规则匹配的上游。它的核心使命是：当用户输入的 Bash 命令包含**管道（`|`）、输出重定向（`>` / `>>`）、子 shell（`()`）、命令组（`{}`）**等操作符时，将命令拆解为可独立权限检查的段（segment），逐段评估，再聚合结果。

该模块解决的关键问题：
1. **管道命令的权限检查**：`cat file | grep foo` 不能简单视为一个整体命令，需要分别检查 `cat file` 与 `grep foo` 的权限。
2. **重定向剥离**：`echo hello > file.txt` 在权限检查时，应将 `> file.txt` 剥离，避免把文件名误判为命令。
3. **复合命令的安全降级**：子 shell、命令组等复杂结构超出当前权限系统的细粒度检查能力，直接降级为 `"ask"`（请求用户确认）。
4. **跨段安全模式检测**：防止 `cd` 与 `git` 分别位于不同管道段时绕过 bare repository 的安全检查。

## 功能点目的

### 1. segmentedCommandPermissionResult — 分段权限结果聚合
- **目的**：对已经拆分好的命令段数组（如管道段），逐个调用完整的权限检查函数，然后按统一策略聚合各段的 `PermissionResult`。
- **实现**：
  - **多 cd 检测**：若多个段中包含 `cd` 命令，直接返回 `behavior: 'ask'`，理由是 "Multiple directory changes in one command require approval for clarity"。
  - **cd+git 跨段检测**：遍历每个段，再用 `splitCommand_DEPRECATED` 拆分为子命令，检查是否存在同时包含 `cd` 与 `git` 的情况。若存在，返回 `behavior: 'ask'`，理由是 "Compound commands with cd and git require approval to prevent bare repository attacks"。这是针对 **bare repo fsmonitor bypass** 的安全补丁。
  - **逐段权限检查**：对每个非空段调用 `bashToolHasPermissionFn({ ...input, command: trimmedSegment })`，结果存入 `Map<string, PermissionResult>`。
  - **聚合策略**：
    - 任一段为 `deny` → 整体 `deny`（携带具体段的拒绝信息）。
    - 全部段为 `allow` → 整体 `allow`。
    - 其他情况（混合 `ask` / `passthrough`）→ 整体 `ask`，并合并所有段的 `suggestions`（自动建议规则）。

### 2. buildSegmentWithoutRedirections — 重定向剥离
- **目的**：在分段权限检查前，将每个管道段中的输出重定向（`>` / `>>`）剥离，防止文件名被当作命令进行权限匹配。
- **实现**：
  - 快速路径：若段中不含 `>`，直接原样返回。
  - 否则调用 `ParsedCommand.parse(segmentCommand)` 获取解析对象，再调用 `parsed.withoutOutputRedirections()` 得到剥离后的命令文本。
  - 若解析失败，回退到原命令。

### 3. checkCommandOperatorPermissions — 公开入口
- **目的**：为 `bashPermissions.ts` 提供一个统一的、带 AST 缓存的入口，用于检查包含操作符的复合命令。
- **实现**：
  - 若调用方已提供预解析的 AST root（且未解析失败 `PARSE_ABORTED`），则直接通过 `buildParsedCommandFromRoot` 构造 `IParsedCommand`，避免重复解析。
  - 否则调用 `ParsedCommand.parse(input.command)` 进行解析。
  - 解析失败返回 `behavior: 'passthrough', message: 'Failed to parse command'`。
  - 解析成功则委托给 `bashToolCheckCommandOperatorPermissions`。

### 4. bashToolCheckCommandOperatorPermissions — 核心操作符检查
- **目的**：判断命令是否包含需要特殊处理的结构（unsafe compound、管道），并决定是继续走正常权限流还是进行分段检查。
- **实现**：
  - **步骤 1：Unsafe Compound 检查**
    - 通过 `parsed.getTreeSitterAnalysis()` 检查是否存在 `subshell` 或 `commandGroup`。
    - 若 tree-sitter 不可用，回退到 `isUnsafeCompoundCommand_DEPRECATED(input.command)`。
    - 若判定为 unsafe compound，进一步调用 `bashCommandIsSafeAsync_DEPRECATED(input.command)` 获取更具体的拒绝消息（若可用）。
    - 最终返回 `behavior: 'ask'`，**不生成 suggestions**（因为这类命令无法通过规则自动允许）。
  - **步骤 2：管道检查**
    - 调用 `parsed.getPipeSegments()` 获取管道段。
    - 若段数 `<= 1`，说明没有管道，返回 `behavior: 'passthrough'`，让上层正常流程处理。
    - 若段数 `> 1`，对每个段异步调用 `buildSegmentWithoutRedirections` 剥离重定向，然后调用 `segmentedCommandPermissionResult` 进行分段权限检查。

## 具体技术实现

### 数据结构与类型

**CommandIdentityCheckers**
```ts
export type CommandIdentityCheckers = {
  isNormalizedCdCommand: (command: string) => boolean
  isNormalizedGitCommand: (command: string) => boolean
}
```

这两个 checker 由 `bashPermissions.ts` 注入，用于识别经过规范化后的 `cd` 与 `git` 命令。之所以不直接在这里写死正则，是为了与 `bashPermissions.ts` 中的命令规范化逻辑保持一致。

**PermissionResult 结构（来自 `src/types/permissions.ts`）**
```ts
type PermissionResult =
  | { behavior: 'allow'; updatedInput?; decisionReason? }
  | { behavior: 'deny'; message; decisionReason? }
  | { behavior: 'ask'; message; decisionReason?; suggestions? }
  | { behavior: 'passthrough'; message? }
```

### 关键流程图

```
输入: input (BashToolInput), astRoot (可选), checkers, bashToolHasPermissionFn
  │
  ▼
checkCommandOperatorPermissions
  │
  ├─ 有有效 astRoot? ──► buildParsedCommandFromRoot
  └─ 无 ───────────────► ParsedCommand.parse
  │
  ▼
bashToolCheckCommandOperatorPermissions
  │
  ├─ 含 subshell / commandGroup? ──► behavior: 'ask' (unsafe compound)
  │
  ├─ getPipeSegments() ──► 段数 <= 1? ──► behavior: 'passthrough'
  │
  └─ 段数 > 1?
       │
       ▼
  对每个段: buildSegmentWithoutRedirections
       │
       ▼
  segmentedCommandPermissionResult
       │
       ├─ 多 cd? ──► ask
       ├─ cd+git 跨段? ──► ask
       └─ 逐段调用 bashToolHasPermissionFn
            │
            ▼
       聚合: deny > ask > allow
```

### ParsedCommand 的使用细节

`ParsedCommand.parse` 内部采用 **tree-sitter 优先、regex 回退** 的双轨策略：

1. **Tree-sitter 路径**：调用原生 NAPI 解析器（`src/utils/bash/parser.js`），返回 AST root node。`buildParsedCommandFromRoot` 从中提取 pipe 位置、重定向节点、tree-sitter 分析数据，构造 `TreeSitterParsedCommand`。
2. **Regex 回退路径**：当 tree-sitter 不可用时，使用 `RegexParsedCommand_DEPRECATED`，基于 `splitCommandWithOperators` 和 `extractOutputRedirections` 做简单拆分。

`ParsedCommand.parse` 还实现了 **size-1 缓存**（`lastCmd` / `lastResult`），避免在同一轮权限检查中对相同命令重复解析。

### cd+git 跨段安全检查的详细逻辑

```ts
for (const segment of segments) {
  const subcommands = splitCommand_DEPRECATED(segment)
  for (const sub of subcommands) {
    const trimmed = sub.trim()
    if (checkers.isNormalizedCdCommand(trimmed)) hasCd = true
    if (checkers.isNormalizedGitCommand(trimmed)) hasGit = true
  }
}
if (hasCd && hasGit) {
  // 返回 ask，防止 bare repository fsmonitor bypass
}
```

**安全背景**：攻击者可能构造类似 `cd sub && echo | git status` 的命令。由于管道将命令分成两段，每段独立检查时，`cd sub && echo` 不会触发 `git` 检查，`git status` 也不会触发 `cd` 检查。该跨段检测专门堵住这个绕过路径。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/BashTool/bashCommandHelpers.ts` | **本文件**，操作符/复合命令权限分析 |
| `src/tools/BashTool/bashPermissions.ts` | **主调用方**，在 `bashToolHasPermission` 中调用 `checkCommandOperatorPermissions` |
| `src/utils/bash/ParsedCommand.ts` | 提供 `IParsedCommand`、`ParsedCommand.parse`、`buildParsedCommandFromRoot` |
| `src/utils/bash/parser.ts` | 提供 `Node` 类型与 `PARSE_ABORTED` 常量 |
| `src/utils/bash/commands.ts` | 提供 `splitCommand_DEPRECATED`、`isUnsafeCompoundCommand_DEPRECATED` |
| `src/utils/bash/treeSitterAnalysis.ts` | 提供 `TreeSitterAnalysis` 类型，用于检测 subshell / commandGroup |
| `src/utils/permissions/PermissionResult.ts` | 提供 `PermissionResult` 类型 |
| `src/utils/permissions/PermissionUpdateSchema.ts` | 提供 `PermissionUpdate` 类型 |
| `src/utils/permissions/permissions.ts` | 提供 `createPermissionRequestMessage`，用于生成权限请求消息 |
| `src/tools/BashTool/bashSecurity.ts` | 提供 `bashCommandIsSafeAsync_DEPRECATED`，用于 unsafe compound 的降级消息 |
| `src/tools/BashTool/BashTool.tsx` | 提供 `BashTool.inputSchema` 与 `BashTool.name` |

## 依赖与外部交互

### 直接依赖模块

```ts
import type { z } from 'zod/v4'
import {
  isUnsafeCompoundCommand_DEPRECATED,
  splitCommand_DEPRECATED,
} from '../../utils/bash/commands.js'
import {
  buildParsedCommandFromRoot,
  type IParsedCommand,
  ParsedCommand,
} from '../../utils/bash/ParsedCommand.js'
import { type Node, PARSE_ABORTED } from '../../utils/bash/parser.js'
import type { PermissionResult } from '../../utils/permissions/PermissionResult.js'
import type { PermissionUpdate } from '../../utils/permissions/PermissionUpdateSchema.js'
import { createPermissionRequestMessage } from '../../utils/permissions/permissions.js'
import { BashTool } from './BashTool.js'
import { bashCommandIsSafeAsync_DEPRECATED } from './bashSecurity.js'
```

### 运行时交互
- **输入**：由 `bashPermissions.ts` 的 `bashToolHasPermission` 函数调用，传入：
  - `input`：经过初步处理的 Bash 命令输入（`z.infer<typeof BashTool.inputSchema>`）
  - `bashToolHasPermissionFn`：闭包，指向 `bashToolHasPermission` 自身，用于对子命令递归检查
  - `checkers`：`isNormalizedCdCommand` / `isNormalizedGitCommand`
  - `astRoot`：可选的预解析 AST，避免重复解析
- **输出**：返回 `Promise<PermissionResult>`，决定该命令是 `allow`、`deny`、`ask` 还是 `passthrough`。
- **无直接网络/文件 IO**：纯计算模块，但 `ParsedCommand.parse` 可能触发一次原生 NAPI 调用（tree-sitter 解析）。

## 风险、边界与改进建议

### 风险与边界

1. **`splitCommand_DEPRECATED` 的递归爆炸风险**
   - `bashPermissions.ts` 中已有注释（CC-643）指出：`splitCommand_DEPRECATED` 在复杂复合命令上可能产生指数级增长的子命令数组，导致事件循环饥饿、REPL 冻结。
   - 虽然 `bashCommandHelpers.ts` 中的 `segmentedCommandPermissionResult` 仅在管道段内部再次调用 `splitCommand_DEPRECATED`（用于 cd+git 检测），但攻击者仍可构造极深的嵌套命令触发性能问题。
   - **建议**：为 `segmentedCommandPermissionResult` 中的子命令拆分增加数量上限（如每段最多 50 个子命令），超限直接 `ask`。

2. **`ParsedCommand.parse` 的 size-1 缓存局限性**
   - 缓存仅保存最近一次的解析结果。在并发或交替解析多个不同命令时，缓存命中率低。
   - **建议**：评估是否将缓存升级为基于 `LRU` 的有限容量缓存（如 8-16 条），但需注意内存泄漏风险（避免长期持有 `TreeSitterParsedCommand` 实例）。

3. **重定向剥离的语义边界**
   - `buildSegmentWithoutRedirections` 仅剥离**输出重定向**（`>` / `>>`），不处理输入重定向（`<`）、here-document（`<<`）、文件描述符重定向（`2>&1`）。
   - 这意味着 `cat < input.txt` 在权限检查时仍能看到 `< input.txt`，而 `cat > output.txt` 则看不到 `> output.txt`。两边处理不一致可能导致用户困惑。
   - **建议**：统一重定向剥离策略文档，明确为何只剥离输出重定向（通常是因为输出文件不影响命令本身的权限分类），或扩展剥离范围。

4. **unsafe compound 的 "无 suggestion" 策略**
   - 当命令包含子 shell 或命令组时，返回的 `ask` 结果**不带 `suggestions`**。这意味着用户无法通过 "Yes, and don't ask again" 快速建立规则，每次都需要手动确认。
   - 这是有意为之（注释明确说明 "we dont want to suggest rules since we wont be able to allow it"），但会降低高级用户的使用效率。
   - **建议**：若未来权限系统支持对子 shell 做更细粒度分析，可重新评估是否为 safe-looking 的 compound 命令生成前缀规则建议。

5. **`bashCommandIsSafeAsync_DEPRECATED` 的双重调用**
   - 在 `bashToolCheckCommandOperatorPermissions` 中，unsafe compound 路径会调用 `bashCommandIsSafeAsync_DEPRECATED` 以获取更具体的错误消息。该函数本身内部也会调用 `ParsedCommand.parse`，与上层的解析存在重复计算。
   - **建议**：将已解析的 `parsed` 对象直接传递给 `bashCommandIsSafeAsync_DEPRECATED` 的重载版本，避免重复解析。

6. **`checkers` 注入的隐式契约**
   - `isNormalizedCdCommand` 与 `isNormalizedGitCommand` 的具体实现不在本模块，而是由 `bashPermissions.ts` 注入。若注入的 checker 与 `bashPermissions.ts` 内部的逻辑不一致，可能导致安全检测漏报或误报。
   - **建议**：将 checker 实现与 `bashPermissions.ts` 中的命令规范化逻辑抽取到同一模块（如 `src/utils/bash/commandIdentity.ts`），消除隐式契约。

7. **聚合逻辑中的 `passthrough` 处理**
   - `segmentedCommandPermissionResult` 在收集需要 approval 的段时，将 `behavior === 'ask' || behavior === 'passthrough'` 都视为 "需要批准"。这意味着若某段返回 `passthrough`（如解析失败），整体命令会被提升为 `ask`。
   - 这是安全的设计（fail-safe），但可能导致一些本可正常执行的命令被不必要地拦截。
   - **建议**：增加日志或 metrics，监控 `passthrough` 被提升为 `ask` 的频率，识别解析器的薄弱环节。

### 改进建议

- **增加单元测试覆盖**：当前 `src/tools/BashTool/` 目录下未发现测试文件。建议为 `segmentedCommandPermissionResult` 和 `bashToolCheckCommandOperatorPermissions` 编写单元测试，覆盖以下场景：
  - 简单单段命令（应 passthrough）
  - 两段管道命令（应分段检查并聚合）
  - cd + git 跨段组合（应 ask）
  - 多 cd 命令（应 ask）
  - 含子 shell 的命令（应 ask，无 suggestions）
  - 含输出重定向的管道命令（重定向应被正确剥离）
- **性能优化**：为 `segmentedCommandPermissionResult` 中的子命令拆分增加上限保护，防止恶意/畸形命令导致 CPU 占满。
- **文档化重定向剥离策略**：在代码注释或设计文档中明确说明为何只剥离 `>` / `>>`，以及输入重定向、here-doc、fd 重定向的处理原则。
