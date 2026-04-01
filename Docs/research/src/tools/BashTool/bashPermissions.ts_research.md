# 研究报告：`src/tools/BashTool/bashPermissions.ts`

> 研究范围：目标文件本身、直接调用方（`BashTool.tsx`、`bashCommandHelpers.ts`）、被调用方（`pathValidation.ts`、`bashSecurity.ts`、`modeValidation.ts`、权限基础设施、`ast.ts`）、相关类型定义与配置。不包含 README/Markdown 文档。

---

## 一、场景与职责

`bashPermissions.ts` 是 Claude Code 中 **Bash 工具权限决策的核心引擎**，总长度约 2621 行。它负责在每次模型调用 `BashTool` 前，对输入的命令字符串进行多层次的安全分析与权限判定，最终返回 `PermissionResult`（`allow` / `ask` / `deny` / `passthrough`）。

### 1.1 在系统中的位置

- **调用方**：`BashTool.tsx` 的 `checkPermissions` 方法直接调用 `bashToolHasPermission(input, context)`（`BashTool.tsx:540`）。
- **间接调用方**：`bashCommandHelpers.ts` 中的管道/复合命令处理逻辑会递归调用 `bashToolHasPermission` 来逐段校验权限。
- **被调用方**：文件内部聚合了 `bashSecurity.ts`（正则安全门）、`pathValidation.ts`（路径约束）、`modeValidation.ts`（模式自动放行）、`readOnlyValidation.ts`（只读命令白名单）、`utils/permissions/bashClassifier.ts`（AI 分类器）、`utils/bash/ast.ts`（tree-sitter AST 解析）等多个子系统的判定结果。

### 1.2 核心职责

1. **规则匹配**：将用户命令与用户配置的 `allow` / `ask` / `deny` 规则（精确、前缀、通配符）进行匹配。
2. **命令结构安全分析**：通过 tree-sitter AST 解析或 legacy shell-quote 回退，检测命令注入、结构歧义、不可解析的复合命令。
3. **路径约束检查**：验证命令操作的路径是否超出项目工作目录（如 `cd` 到外部目录、重定向写入敏感路径）。
4. **模式与沙盒集成**：处理 `acceptEdits`、`auto`、`bypassPermissions` 等权限模式，以及沙盒自动放行逻辑。
5. **分类器（Classifier）集成**：在需要用户确认时，启动异步 AI 分类器检查，实现高置信度自动批准。
6. **规则建议生成**：在返回 `ask` 时，向 UI 提供可保存的权限规则建议（如 `Bash(git status:*)`）。

---

## 二、功能点目的

文件导出了 20 余个函数/常量，按职责可划分为 6 组：

### 2.1 主入口与 orchestration

| 函数/常量 | 目的 |
|-----------|------|
| `bashToolHasPermission` | 主入口。协调 AST 解析、沙盒检查、精确匹配、分类器、子命令拆分、管道处理、安全注入检查等完整流程。 |
| `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50` | 防止 `splitCommand_DEPRECATED` 在复杂复合命令上产生指数级子命令数组，导致事件循环饿死（CC-643）。 |
| `MAX_SUGGESTED_RULES_FOR_COMPOUND = 5` | 限制复合命令一次提示中建议的规则数量，避免 UI 噪音（GH#11380）。 |

### 2.2 规则匹配引擎

| 函数 | 目的 |
|------|------|
| `bashToolCheckExactMatchPermission` | 仅做精确规则匹配（`exact`），优先级最高：deny > ask > allow。 |
| `bashToolCheckPermission` | 在精确匹配之后，进行前缀/通配符规则匹配，并串联路径约束、sed 约束、模式检查、只读检查。 |
| `filterRulesByContentsMatchingInput` | 底层规则过滤器。支持 `exact` / `prefix` 两种匹配模式，处理命令剥离（wrapper/env var）、复合命令拦截、通配符匹配。 |
| `matchingRulesForInput` | 对 deny / ask / allow 三类规则分别调用 `filterRulesByContentsMatchingInput`，返回匹配结果三元组。 |

### 2.3 命令前缀提取与建议生成

| 函数 | 目的 |
|------|------|
| `getSimpleCommandPrefix` | 从命令中提取稳定的 "cmd subcmd" 前缀（如 `git commit`），用于生成前缀规则建议。跳过安全环境变量赋值。 |
| `getFirstWordPrefix` | UI 回退：仅提取第一个单词（如 `python3`），作为可编辑规则建议的初始值。 |
| `suggestionForExactCommand` | 为单个命令生成建议：heredoc 前前缀 > 首行前缀 > `getSimpleCommandPrefix` > 精确匹配。 |
| `suggestionForPrefix` | 包装共享实现，生成 `Bash(<prefix>:*)` 形式的规则建议。 |

### 2.4 命令归一化（绕过防御）

| 函数 | 目的 |
|------|------|
| `stripSafeWrappers` | 剥离安全 wrapper（`timeout`、`time`、`nice`、`stdbuf`、`nohup`）和安全环境变量前缀，使规则匹配与实际执行语义一致。 |
| `stripWrappersFromArgv` | `stripSafeWrappers` 的 argv 级别对应实现，用于 AST 已解析出的 `argv[]` 场景。 |
| `stripAllLeadingEnvVars` | 更激进的环境变量剥离，用于 **deny/ask 规则匹配**，防止 `FOO=bar rm` 绕过 `Bash(rm:*)` 的 deny。 |
| `isNormalizedGitCommand` / `isNormalizedCdCommand` / `commandHasAnyCd` | 在剥离 wrapper 和解析 shell 引用后，检测命令是否为 git/cd/pushd/popd，用于 cd+git 安全门。 |

### 2.5 沙盒与分类器

| 函数 | 目的 |
|------|------|
| `checkSandboxAutoAllow` | 当沙盒和 `autoAllowBashIfSandboxed` 均开启时，对无显式规则命令自动放行，但仍尊重 deny/ask 规则。 |
| `buildPendingClassifierCheck` | 构造待执行的分类器检查元数据（`command`, `cwd`, `descriptions`）。 |
| `startSpeculativeClassifierCheck` / `consumeSpeculativeClassifierCheck` / `peekSpeculativeClassifierCheck` | 投机执行 allow 分类器：在权限弹窗设置期间并行运行，减少用户等待。 |
| `awaitClassifierAutoApproval` | 被 swarm agent 调用，等待分类器结果以决定是否向 leader  escalation。 |
| `executeAsyncClassifierCheck` | 在权限弹窗展示后异步执行分类器，若用户尚未交互且高置信度匹配，则自动批准。 |

### 2.6 辅助安全门（early exit）

| 函数 | 目的 |
|------|------|
| `checkEarlyExitDeny` | 在 AST `too-complex` 路径上快速检查精确匹配和 deny 规则，防止将 deny 降级为 ask。 |
| `checkSemanticsDeny` | 在 AST `simple` 但语义检查失败路径上，逐 `SimpleCommand` 检查 deny 规则。 |
| `filterCdCwdSubcommands` | 过滤掉模型常前置的 `cd ${cwd}` 子命令，避免无意义权限提示。 |
| `logClassifierResultForAnts` | ANT-ONLY 遥测，记录分类器决策结果用于内部分析。 |

---

## 三、具体技术实现

### 3.1 主流程 `bashToolHasPermission` 的关键阶段

函数签名：
```typescript
export async function bashToolHasPermission(
  input: z.infer<typeof BashTool.inputSchema>,
  context: ToolUseContext,
  getCommandSubcommandPrefixFn = getCommandSubcommandPrefix,
): Promise<PermissionResult>
```

流程可分为 10 个阶段：

#### 阶段 0：AST 解析（tree-sitter）

```typescript
let astRoot = injectionCheckDisabled
  ? null
  : feature('TREE_SITTER_BASH_SHADOW') && !shadowEnabled
    ? null
    : await parseCommandRaw(input.command)
let astResult: ParseForSecurityResult = astRoot
  ? parseForSecurityFromAst(input.command, astRoot)
  : { kind: 'parse-unavailable' }
```

- `parseCommandRaw` 调用 tree-sitter WASM 解析 bash 语法树。
- `parseForSecurityFromAst` 对 AST 进行**显式允许列表（allowlist）遍历**：只接受 `program`、`list`、`pipeline`、`redirected_statement` 等结构节点 + 已知的叶子节点。任何未知节点类型直接返回 `too-complex`（fail-closed 设计）。
- 若解析成功且无复杂结构，返回 `kind: 'simple'` 并附带 `SimpleCommand[]`，每个命令包含 `argv`、`envVars`、`redirects`、`text`。

**Shadow 模式**：当 `TREE_SITTER_BASH_SHADOW` feature 开启时，系统会记录 tree-sitter 与 legacy `splitCommand` 的差异遥测（`tengu_tree_sitter_shadow`），但**强制将结果重置为 `parse-unavailable`**，使 legacy 路径保持权威。这是一种灰度观测策略。

#### 阶段 1：too-complex 与 semantic-fail 快速退出

- **`too-complex`**：命令包含 `$(...)`、`<(...)`、brace expansion、控制流等我们不愿静态分析的语法。此时只检查精确匹配和 deny 规则（`checkEarlyExitDeny`），若都不匹配则返回 `ask`，并附加 `pendingClassifierCheck`。
- **`simple` 但语义失败**：`checkSemantics(astResult.commands)` 会检测 `eval`、`zsh` 危险内置命令（`zmodload`、`ztcp` 等）。此时通过 `checkSemanticsDeny` 检查 deny 规则后返回 `ask`。

#### 阶段 2：沙盒自动放行

```typescript
if (
  SandboxManager.isSandboxingEnabled() &&
  SandboxManager.isAutoAllowBashIfSandboxedEnabled() &&
  shouldUseSandbox(input)
) {
  const sandboxAutoAllowResult = checkSandboxAutoAllow(...)
  ...
}
```

`checkSandboxAutoAllow` 会检查完整命令及每个子命令的 deny/ask 规则，有则优先返回；无显式规则时直接 `allow`。

#### 阶段 3：精确匹配

```typescript
const exactMatchResult = bashToolCheckExactMatchPermission(input, appState.toolPermissionContext)
```

精确匹配逻辑：deny > ask > allow > passthrough。若命中 deny/ask 直接返回。

#### 阶段 4：Bash Prompt 分类器（deny/ask）

当 `isClassifierPermissionsEnabled()` 返回 true（ANT 构建通常为 true），且不在 `auto` 模式时，并行调用 `classifyBashCommand` 进行 deny 和 ask 分类：

```typescript
const [denyResult, askResult] = await Promise.all([
  hasDeny ? classifyBashCommand(..., 'deny', ...) : null,
  hasAsk  ? classifyBashCommand(..., 'ask',  ...) : null,
])
```

deny 高置信度匹配直接 `deny`；ask 高置信度匹配返回 `ask` 并带建议。

#### 阶段 5：命令操作符权限（管道）

```typescript
const commandOperatorResult = await checkCommandOperatorPermissions(
  input,
  (i) => bashToolHasPermission(i, context, ...),
  { isNormalizedCdCommand, isNormalizedGitCommand },
  astRoot,
)
```

`checkCommandOperatorPermissions`（位于 `bashCommandHelpers.ts`）使用 `ParsedCommand` 检测管道（`|`）。若有管道：
1. 将命令拆分为 pipe segments；
2. 对每个 segment 递归调用 `bashToolHasPermission`；
3. 若所有 segment 均 allow，则返回 allow；否则 ask。

**关键安全修复**：当 `commandOperatorResult.behavior === 'allow'` 时，`bashToolHasPermission` **不会直接返回**，而是继续对原始命令执行：
- 危险模式检查（backticks 等）
- `checkPathConstraints`（防止重定向被 strip 后绕过路径检查）

#### 阶段 6：Legacy 命令注入检查（misparsing gate）

仅当 AST 解析不可用（`astSubcommands === null`）且未禁用注入检查时执行：

```typescript
const originalCommandSafetyResult = await bashCommandIsSafeAsync(input.command)
```

`bashCommandIsSafeAsync_DEPRECATED`（`bashSecurity.ts`）运行约 20 组正则验证器，检测：
- 命令替换 `$()` / backticks
- 进程替换 `<()` / `>()`
- Zsh equals expansion `=cmd`
- IFS 注入
- 畸形 token 注入
- 反斜杠转义操作符
- 等等

若触发 `isBashSecurityCheckForMisparsing`，会尝试 `stripSafeHeredocSubstitutions` 后重新检查；若仍失败，则返回 `ask`（但允许精确匹配 `allow` 规则覆盖）。

#### 阶段 7：子命令拆分与 cd 过滤

```typescript
const rawSubcommands = astSubcommands ?? shadowLegacySubs ?? splitCommand(input.command)
const { subcommands, astCommandsByIdx } = filterCdCwdSubcommands(...)
```

- 优先使用 AST 提取的 `text`  spans；
- 过滤掉 `cd ${cwd}` / `cd ${cwdMingw}` 前缀（模型常自动添加）；
- 若 legacy 路径下子命令数超过 50，直接 `ask`（CC-643 防护）。

#### 阶段 8：cd + git 安全门

```typescript
if (compoundCommandHasCd) {
  const hasGitCommand = subcommands.some(cmd => isNormalizedGitCommand(cmd.trim()))
  if (hasGitCommand) { return ask(...) }
}
```

这是为了防止 **bare repository fsmonitor 攻击**：攻击者可在恶意目录放置带有恶意 `core.fsmonitor` 配置的裸 git 仓库，`cd` 进去后执行 `git status` 即可触发任意代码执行。

#### 阶段 9：子命令级权限判定与合并

对每个子命令调用 `bashToolCheckPermission`，然后：
1. 若有 deny，立即返回 deny；
2. 对原始命令调用 `checkPathConstraints` 验证重定向；
3. 若仅有一个子命令 ask，short-circuit 返回；
4. 若所有子命令 allow 且无注入，返回 allow；
5. 否则合并所有子命令的 `suggestions`，去重后返回 ask。

### 3.2 规则匹配引擎详解

#### 3.2.1 规则类型

规则内容由 `utils/permissions/shellRuleMatching.ts` 解析为三种类型：
- `exact`：整条命令字符串完全相等。
- `prefix`： legacy `cmd:*` 语法，或 `getSimpleCommandPrefix` 提取的 `"cmd subcmd"`。
- `wildcard`：包含未转义 `*` 的模式（如 `*test*`）。

#### 3.2.2 `filterRulesByContentsMatchingInput` 的核心逻辑

1. **重定向剥离**：`extractOutputRedirections(command)` 移除输出重定向，使 `Bash(python:*)` 能匹配 `python script.py > out.txt`。
2. **Safe Wrapper 剥离**：`stripSafeWrappers` 移除 `timeout`、`nohup`、安全环境变量等前缀。
3. **Deny/Ask 的激进剥离**：当 `stripAllEnvVars: true` 时，使用 `stripAllLeadingEnvVars` 进行 fixed-point 迭代剥离（wrapper + env var 交替剥离），防止 `nohup FOO=bar timeout 5 claude` 多层绕过。
4. **复合命令拦截**：在 `prefix` 模式下，对 allow 规则使用 `splitCommand(cmd).length > 1` 检测复合命令，阻止 `Bash(cd:*)` 匹配 `cd /path && python3 evil.py`。deny/ask 规则跳过此检查（必须能拦截复合命令中的危险子命令）。
5. **词边界检查**：前缀规则要求 `cmdToMatch === prefix` 或 `cmdToMatch.startsWith(prefix + ' ')`，防止 `ls:*` 匹配 `lsof`。
6. **xargs 特殊处理**：`xargs grep pattern` 被视为匹配 `Bash(grep:*)`。
7. **通配符安全**：wildcard 在 `exact` 模式下拒绝匹配（防止 `foo *` 匹配 `foo arg && curl evil.com`）；在 `prefix` 模式下同样拦截复合命令。

### 3.3 命令归一化与绕过防御

#### 3.3.1 `stripSafeWrappers`

分两阶段执行：
- **Phase 1**：循环剥离注释行 + 安全环境变量（仅 `SAFE_ENV_VARS` 白名单中的变量，如 `NODE_ENV`、`GOOS`、`LANG`）。
- **Phase 2**：循环剥离 wrapper 命令（`timeout`、`time`、`nice`、`stdbuf`、`nohup`）。

**安全设计**：
- 环境变量模式使用 `[ 	]+` 而非 `\s+`，防止匹配跨换行符（换行在 bash 中是命令分隔符）。
- `timeout` 标志值使用严格字符集 `[A-Za-z0-9_.+-]+`，防止 `timeout -k$(id) 10 ls` 的注入绕过（历史漏洞修复）。
- `nice` 模式覆盖 `nice cmd`、`nice -n N cmd`、`nice -N cmd`，与 `checkSemantics` 保持一致。

#### 3.3.2 `stripAllLeadingEnvVars`

用于 deny/ask 规则匹配，正则表达式更为宽泛：
```typescript
const ENV_VAR_PATTERN =
  /^([A-Za-z_][A-Za-z0-9_]*(?:\[[^\]]*\])?)\+?=(?:'[^'\n\r]*'|"(?:\\.|[^"$`\\\n\r])*"|\\.|[^ \t\n\r$`;|&()<>\\'"])*[ \t]+/
```

支持单引号、双引号、反斜杠转义、未引号值，并排除 shell 元字符。`blocklist` 参数（如 `BINARY_HIJACK_VARS = /^(LD_|DYLD_|PATH$)/`）可阻止剥离特定危险变量。

### 3.4 分类器投机执行与异步自动批准

```typescript
const speculativeChecks = new Map<string, Promise<ClassifierResult>>()
```

- `startSpeculativeClassifierCheck` 在权限检查早期启动 `classifyBashCommand('allow', ...)`，结果缓存到 `speculativeChecks` Map。
- `executeAsyncClassifierCheck` 在 UI 弹窗后消费该投机结果（若存在），否则重新发起请求。
- 若分类器 `matches === true && confidence === 'high'`，且用户尚未与弹窗交互（`callbacks.shouldContinue()`），则通过 `callbacks.onAllow` 自动批准。
- `awaitClassifierAutoApproval` 供 swarm 子 agent 使用：先运行分类器，只有分类器不自动批准时才向主 agent 请示。

### 3.5 关键数据结构

#### `PermissionResult`（来自 `src/types/permissions.ts`）

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?; decisionReason? }
  | { behavior: 'ask'; message; decisionReason?; suggestions?; pendingClassifierCheck? }
  | { behavior: 'deny'; message; decisionReason }
  | { behavior: 'passthrough'; message?; ... }
```

#### `SimpleCommand`（来自 `src/utils/bash/ast.ts`）

```typescript
type SimpleCommand = {
  argv: string[]
  envVars: { name: string; value: string }[]
  redirects: Redirect[]
  text: string
}
```

#### `ParseForSecurityResult`

```typescript
type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

---

## 四、关键代码路径与文件引用

### 4.1 直接调用方

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/tools/BashTool/BashTool.tsx:43` | `import { bashToolHasPermission, ... }` | 工具定义中 `checkPermissions` 的入口。 |
| `src/tools/BashTool/BashTool.tsx:540` | `return bashToolHasPermission(input, context)` | 每次 Bash 工具调用前的权限检查。 |
| `src/tools/BashTool/bashCommandHelpers.ts:91` | `bashToolHasPermissionFn({ command: trimmedSegment })` | 管道 segment 递归校验。 |

### 4.2 被调用方（同目录）

| 文件 | 被调用的导出/函数 | 说明 |
|------|-------------------|------|
| `src/tools/BashTool/bashCommandHelpers.ts` | `checkCommandOperatorPermissions` | 处理管道与复合命令操作符。 |
| `src/tools/BashTool/bashSecurity.ts` | `bashCommandIsSafeAsync_DEPRECATED`, `stripSafeHeredocSubstitutions` | Legacy 安全注入检查。 |
| `src/tools/BashTool/pathValidation.ts` | `checkPathConstraints` | 路径边界、危险删除、重定向检查。 |
| `src/tools/BashTool/modeValidation.ts` | `checkPermissionMode` | `acceptEdits` 等模式自动放行。 |
| `src/tools/BashTool/readOnlyValidation.ts` | `BashTool.isReadOnly(input)`（间接） | 只读命令白名单。 |
| `src/tools/BashTool/shouldUseSandbox.ts` | `shouldUseSandbox` | 判断命令是否应走沙盒。 |

### 4.3 被调用方（跨目录基础设施）

| 文件 | 被调用的导出/函数 | 说明 |
|------|-------------------|------|
| `src/utils/bash/ast.ts` | `parseForSecurityFromAst`, `checkSemantics`, `nodeTypeId`, `SimpleCommand`, `Redirect` | tree-sitter AST 解析与语义检查。 |
| `src/utils/bash/parser.ts` | `parseCommandRaw`, `Node`, `PARSE_ABORTED` | 底层 tree-sitter 解析器。 |
| `src/utils/bash/commands.ts` | `splitCommand_DEPRECATED`, `extractOutputRedirections`, `getCommandSubcommandPrefix` | 命令拆分与重定向提取。 |
| `src/utils/bash/shellQuote.ts` | `tryParseShellCommand` | shell-quote 解析回退。 |
| `src/utils/permissions/shellRuleMatching.ts` | `parsePermissionRule`, `matchWildcardPattern`, `sharedSuggestionForPrefix`, `sharedSuggestionForExactCommand` | 规则解析与通配匹配。 |
| `src/utils/permissions/bashClassifier.ts` | `classifyBashCommand`, `isClassifierPermissionsEnabled`, `getBashPrompt*Descriptions` | AI 分类器（ANT 构建启用）。 |
| `src/utils/permissions/permissions.ts` | `createPermissionRequestMessage`, `getRuleByContentsForTool` | 权限消息构造与规则索引。 |
| `src/utils/permissions/PermissionUpdate.ts` | `extractRules`, `createReadRuleSuggestion` | 规则建议提取。 |
| `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager` | 沙盒状态查询。 |
| `src/services/analytics/index.ts` | `logEvent` | 遥测事件上报。 |

### 4.4 类型定义

| 文件 | 说明 |
|------|------|
| `src/types/permissions.ts` | `PermissionResult`、`PermissionRule`、`PermissionUpdate`、`PendingClassifierCheck` 等核心类型。 |
| `src/Tool.ts` | `ToolPermissionContext`、`ToolUseContext`。 |

---

## 五、依赖与外部交互

### 5.1 运行时依赖

- **tree-sitter WASM**：`parseCommandRaw` 依赖动态加载的 `tree-sitter-bash` WASM 模块。模块加载失败时无缝回退到 legacy shell-quote 路径。
- **GrowthBook / Feature Flags**：大量使用 `feature('BASH_CLASSIFIER')`、`feature('TREE_SITTER_BASH_SHADOW')`、`feature('TRANSCRIPT_CLASSIFIER')` 控制功能开关。
- **沙盒管理器**：`SandboxManager.isSandboxingEnabled()` 等状态决定自动放行逻辑。
- **Haiku / 分类器 API**：`classifyBashCommand` 在 ANT 构建中通过内部 API（Haiku）对命令进行语义分类。
- **Analytics**：通过 `logEvent` 上报 `tengu_tree_sitter_shadow`、`tengu_bash_ast_too_complex`、`tengu_internal_bash_classifier_result` 等事件。

### 5.2 配置/环境变量

| 变量/配置 | 作用 |
|-----------|------|
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | 禁用 legacy 和 AST 命令注入检查（调试用）。 |
| `USER_TYPE === 'ant'` | 解锁 `ANT_ONLY_SAFE_ENV_VARS` 中的额外环境变量剥离，以及 ANT-ONLY 遥测。 |
| `tengu_birch_trellis` (GrowthBook) | `TREE_SITTER_BASH_SHADOW` 的 killswitch。 |

### 5.3 输入/输出协议

- **输入**：`BashTool.inputSchema`（至少包含 `command: string`，可能还有 `sandbox?: boolean`）。
- **输出**：`PermissionResult`，可能附加 `suggestions: PermissionUpdate[]` 和 `pendingClassifierCheck: PendingClassifierCheck`。

---

## 六、风险、边界与改进建议

### 6.1 已知风险与历史漏洞修复痕迹

1. **Bun DCE Complexity Cliff（多次提及）**
   - 代码注释反复提到 `bashToolHasPermission` 处于 Bun `feature()` 死代码消除的复杂度预算边缘。多个辅助函数（`filterCdCwdSubcommands`、`checkEarlyExitDeny`、`checkSemanticsDeny`、`skipTimeoutFlags`、`stripWrappersFromArgv`）被**刻意提取为独立函数**，仅仅是为了避免内联后导致 `feature('BASH_CLASSIFIER')` 被错误求值为 `false`。
   - **风险**：未来若在该文件中增加内联逻辑，可能意外破坏分类器功能，且问题极其隐蔽（编译期静默降级为 `false`）。

2. **splitCommand_DEPRECATED 的 ReDoS / 指数爆炸**
   - `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50` 是对 `splitCommand_DEPRECATED` 在复杂输入上可能产生指数级子命令数组的缓解（CC-643）。
   - **风险**：legacy 路径仍依赖该函数，攻击者可能构造刚好低于 50 个子命令但仍足够复杂的输入来消耗大量 CPU。

3. **环境变量剥离的绕过与过度剥离的平衡**
   - allow 规则仅剥离 `SAFE_ENV_VARS` 白名单变量，防止 `DOCKER_HOST=evil docker ps` 自动匹配 `Bash(docker ps:*)`。
   - deny 规则使用 `stripAllLeadingEnvVars` 激进剥离，防止 `FOO=bar rm` 绕过。
   - **风险**：白名单变量若被新增时审核不严，可能引入绕过（如将 `PATH` 误加入白名单将导致灾难性后果）。注释中明确列出了禁止加入的变量清单。

4. **AST 与 Legacy 路径的行为差异**
   - Shadow 模式遥测显示团队正在观测 tree-sitter 与 legacy `splitCommand` 的差异（`subsDiffer`）。
   - **风险**：当 tree-sitter 最终切为主路径时，任何未观测到的差异都可能改变权限判定结果。例如 AST 路径跳过 `bashCommandIsSafeAsync` 的 legacy 正则检查，某些边缘 case 的行为可能不一致。

5. **cd+git 复合命令的绕过**
   - `cd` 与 `git` 必须处于**同一复合命令**才会触发安全门。若通过管道分隔（`cd evil || true | git status`），`bashCommandHelpers.ts` 中的跨 segment 检查会捕获，但复杂度更高，存在遗漏可能。

6. **xargs 的特权提升**
   - `xargs` 被允许作为 bare prefix 匹配（`xargs grep pattern` 匹配 `Bash(grep:*)`）。
   - **风险**：`xargs` 的某些 flag（如 `-I`）是安全的，但历史上 `-i` / `-e` 因 GNU getopt 可选参数语义与验证器不一致而被移除。若未来 xargs 版本引入新的危险 flag，白名单可能滞后。

### 6.2 边界条件

- **命令长度 > 10000**：Shadow 遥测会记录 `cmdOverLength: true`，但代码本身没有针对超长命令的特殊截断或拒绝逻辑。
- **非交互式会话**：分类器调用传入 `isNonInteractiveSession`，可能影响分类器的行为或超时策略。
- **Windows 路径**：`windowsPathToPosixPath(cwd)` 用于过滤 `cd ${cwd}` 前缀，确保 Windows 路径也能被正确识别。
- **Heredoc 命令**：`extractPrefixBeforeHeredoc` 确保 heredoc 命令不会生成无用的精确匹配规则（因为 heredoc 内容每次 invocation 都不同）。

### 6.3 改进建议

1. **消除 Bun DCE 隐忧**
   - 建议将 `feature('BASH_CLASSIFIER')` 的求值提取到模块顶层常量，避免在 `bashToolHasPermission` 内部多次内联调用。或者与构建团队沟通，提升/消除该函数的 DCE 复杂度阈值，使代码组织不再受编译器限制。

2. **统一 argv 处理，减少重复 tokenization**
   - 当前 AST 路径提取出 `SimpleCommand.argv` 后，下游 `pathValidation.ts` 和 `readOnlyValidation.ts` 仍大量依赖字符串级别的 `splitCommand_DEPRECATED` 或 `tryParseShellCommand` 重新分词。
   - 建议将 `argv` 直接传递到 `checkPathConstraints` 和 `BashTool.isReadOnly`，消除二次解析带来的不一致性和性能开销。

3. **增强 `too-complex` 路径的透明度**
   - `too-complex` 目前仅返回一个通用 reason。建议在 UI 中向用户展示具体触发 `too-complex` 的节点类型（如 "command substitution"），帮助用户理解为何需要确认，同时减少误报感。

4. **对 `stripSafeWrappers` 和 `stripWrappersFromArgv` 进行统一测试矩阵**
   - 两者必须保持同步（代码注释多次强调）。建议引入自动化测试或代码生成，确保新增 wrapper 时两个函数同时更新，避免安全绕过。

5. **限制 `speculativeChecks` Map 的内存增长**
   - `speculativeChecks` 是模块级全局 Map，命令字符串作为 key。若在高并发场景下大量不同命令被投机执行且从未被消费，Map 可能无限增长。建议增加 TTL 或 LRU 淘汰机制。

6. **减少 `getAppState()` 的重复调用**
   - `bashToolHasPermission` 中 `appState = context.getAppState()` 被调用了 5 次以上（注释说明是为了响应用户 `shift+tab` 等交互状态变化）。
   - 建议明确区分 "需要响应交互变化" 的边界点，在中间阶段使用缓存状态，减少不必要的状态同步开销。

---

*报告生成时间：2026-04-01*  
*基于仓库 commit 状态：当前工作目录最新代码*
