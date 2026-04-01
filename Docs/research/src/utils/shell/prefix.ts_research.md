# prefix.ts 研究文档

## 场景与职责

`prefix.ts` 提供基于 **Haiku LLM** 的命令前缀提取能力，用于 shell 工具（如 BashTool、PowerShellTool）的权限预检。核心思想是：将复杂命令（如 `git -C /repo status --short`）提取出其“前缀”（如 `git status`），从而与用户的权限规则（如 `Bash(git status:*)`）进行匹配。该模块通过调用轻量级 Haiku 模型完成语义解析，并辅以两层 LRU 缓存避免重复请求。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `createCommandPrefixExtractor(config)` | 工厂函数，创建带 LRU 缓存的单命令前缀提取器。 |
| `createSubcommandPrefixExtractor(getPrefix, splitCommand)` | 工厂函数，创建复合命令（含子命令）的前缀提取器。 |
| `getCommandPrefixImpl(...)` | 实际调用 Haiku、解析响应、校验并记录分析事件的内部实现。 |
| `getCommandSubcommandPrefixImpl(...)` | 对复合命令并行提取主命令及各子命令前缀。 |

## 具体技术实现

### 1. 两层缓存结构

```ts
const memoized = memoizeWithLRU(
  (command, abortSignal, isNonInteractiveSession) => {
    const promise = getCommandPrefixImpl(...)
    promise.catch(() => {
      if (memoized.cache.get(command) === promise) {
        memoized.cache.delete(command)
      }
    })
    return promise
  },
  command => command,
  200,
)
```

- 外层：`memoizeWithLRU` 按 `command` 字符串缓存 Promise。
- 内层：Promise 的 `.catch` 在失败时主动驱逐缓存条目，防止中断/失败的 Haiku 调用污染后续查询。
- 缓存容量 200，防止长会话中无界增长。

### 2. Haiku 查询构造

系统 prompt 分两种模式（由 feature flag `tengu_cork_m4q` 控制）：

- **旧模式**: 
  - system: `Your task is to process ${toolName} commands...`
  - user: `${policySpec}\n\nCommand: ${command}`
- **新模式（system prompt policy spec）**:
  - system: `Your task is to process ${toolName} commands...\n\n${policySpec}`
  - user: `Command: ${command}`
  - 同时开启 prompt caching（`enablePromptCaching: true`）。

### 3. 响应校验与防御

Haiku 返回的文本前缀会经过多层校验：

1. **API 错误前缀**: 若返回以 API 错误前缀开头，视为 `null`。
2. **`command_injection_detected`**: Haiku 检测到可疑注入，返回 `null`。
3. **危险前缀黑名单**: `DANGEROUS_SHELL_PREFIXES` 包含 `sh`、`bash`、`zsh`、`pwsh`、`cmd.exe` 等。若前缀为 `git` 或任何 shell 可执行文件，直接拒绝（防止 `git:*` 或 `bash:*` 绕过权限系统）。
4. **`none`**: 未检测到前缀，返回 `null`。
5. **前缀真实性校验**: 用 `command.startsWith(prefix)` 确保返回的字符串确实是命令的前缀，防止模型幻觉。

### 4. 子命令前缀提取

```ts
const subcommands = await splitCommandFn(command)
const [fullCommandPrefix, ...subcommandPrefixesResults] = await Promise.all([
  getPrefix(command, ...),
  ...subcommands.map(async subcommand => ({ subcommand, prefix: await getPrefix(subcommand, ...) })),
])
```

- 对复合命令（如 `cmd1 && cmd2`），先由调用方提供的 `splitCommand` 拆分为子命令数组。
- 并行对所有子命令调用前缀提取器，最终返回 `Map<subcommand, CommandPrefixResult>`。

### 5. 超时警告

```ts
preflightCheckTimeoutId = setTimeout(..., 10000, toolName, isNonInteractiveSession)
```

- 若 Haiku 调用超过 10 秒，向 stderr 输出警告日志（交互式用 `chalk.yellow`，非交互式用 JSON 日志）。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/prefix.ts` | 本文件 | 前缀提取核心实现。 |
| `src/utils/shell/specPrefix.ts` | 相关 | 基于 Fig spec 的前缀提取（非 LLM，纯规则），被 PowerShell 等场景使用。 |
| `src/tools/BashTool/bashPermissions.ts` | 调用方 | 使用 `createCommandPrefixExtractor` 创建 Bash 前缀提取器。 |
| `src/utils/bash/commands.ts` | 调用方 | 可能使用 `createSubcommandPrefixExtractor` 处理复合命令。 |
| `src/services/api/claude.ts` | 被调用 | `queryHaiku()` 发起 Haiku API 请求。 |
| `src/services/analytics/growthbook.ts` | 被调用 | `getFeatureValue_CACHED_MAY_BE_STALE('tengu_cork_m4q')` 控制 prompt 模式。 |
| `src/services/analytics/index.ts` | 被调用 | `logEvent` 记录前缀提取成功/失败/耗时。 |
| `src/utils/memoize.ts` | 被调用 | `memoizeWithLRU` 提供缓存能力。 |

## 依赖与外部交互

- **外部 API**: Anthropic Haiku（通过 `queryHaiku`）。
- **Feature Flag**: `tengu_cork_m4q`（GrowthBook）。
- **分析系统**: `logEvent` 记录事件 `eventName`（由调用方配置）。
- **Node.js 内置**: `process.env.NODE_ENV`、`console`、`setTimeout`。

## 风险、边界与改进建议

### 风险

1. **LLM 幻觉导致权限绕过**: 虽然有多层校验，但模型仍可能返回一个看似合法但实际过宽的前缀（如把 `git push origin main` 识别为 `git` 已被黑名单拦截，但若识别为 `git push` 则相对安全）。
2. **缓存 key 过于简单**: 仅以原始命令字符串作为 key，未考虑 policy spec 或 feature flag 变化；若配置变更，缓存中的旧结果可能不再适用。不过由于 policy spec 变化极罕见，实际风险低。
3. **AbortSignal 未传递给缓存**: 缓存按命令字符串索引，若同一命令在不同 turn 中以不同的 abort signal 调用，会命中同一 Promise；当旧 signal 已 aborted，新调用不会因此中断。这是设计上的“缓存优先”权衡。
4. **测试环境短路**: `NODE_ENV === 'test'` 时直接返回 `null`，意味着测试不会覆盖 Haiku 调用链路。

### 边界

- 仅支持前缀匹配，不支持通配符或正则权限规则的直接解析。
- `DANGEROUS_SHELL_PREFIXES` 是硬编码集合，新增 shell（如 `nu`、`xonsh`）需要手动加入。
- 10 秒超时仅用于日志警告，不会真正中断 Haiku 调用；中断依赖调用方传入的 `AbortSignal`。

### 改进建议

1. **缓存 key 增强**: 将 `policySpec` 摘要或版本号纳入缓存 key，确保配置更新后缓存失效。
2. **本地规则兜底**: 对常见命令（如 `git`、`docker`、`npm`）在本地维护一个快速规则前缀提取器，仅在本地无法解析时才走 Haiku，降低延迟和 API 成本。
3. **测试 mock 标准化**: 当前测试环境直接短路返回 `null`，建议提供可注入的 mock Haiku 响应机制，使测试能覆盖前缀校验逻辑。
4. **子命令拆分缓存共享**: 复合命令的子命令前缀提取结果可以反向写入单命令缓存，提高后续独立调用的命中率。
5. **Haiku 失败降级策略**: 当 Haiku 连续失败或超时时，可临时降级为更保守的权限策略（如要求显式确认），而不是继续尝试可能不稳定的 API 调用。
