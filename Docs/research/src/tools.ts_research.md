# `src/tools.ts` 深度研究文档

> 研究对象：`/home/sansha/Github/claude-code-instructkr/src/tools.ts`  
> 执行时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 一、场景与职责

`src/tools.ts` 是 Claude Code 整个**工具池（Tool Pool）的中央注册表与装配工厂**。它不负责任何单个工具的业务逻辑，而是决定：

1. **哪些工具在当前运行时被暴露给模型**（built-in + MCP + 条件特性门控）。
2. **工具以什么顺序、什么过滤规则进入 prompt/system 缓存前缀**。
3. **不同运行模式（simple、REPL、coordinator、headless）下可见工具的裁剪规则**。
4. **权限拒绝规则（deny rules）如何前置过滤工具列表**，避免模型看到已被 blanket-deny 的工具。
5. **MCP 工具与内置工具的合并、去重、排序策略**。

简言之，它是“工具清单的源文件（source of truth）”。任何需要把工具列表送进 Anthropic API、渲染到 UI、或供子代理（subagent/coordinator worker）使用的代码，最终都要通过本文件暴露的函数获取工具集。

### 核心使用方（Callers）

| 调用方 | 使用的函数 | 场景 |
|--------|-----------|------|
| `src/main.tsx` | `getTools()` | CLI/REPL 启动时初始化工具池 |
| `src/cli/print.ts` | `assembleToolPool()`, `filterToolsByDenyRules()` | Headless/SKD 模式装配工具 |
| `src/hooks/useMergedTools.ts` | `assembleToolPool()` | React 层在 REPL 中合并 MCP 工具 |
| `src/utils/toolPool.ts` | `assembleToolPool()` (间接) | 纯函数合并与 coordinator 过滤 |
| `src/services/tools/toolExecution.ts` | `getAllBaseTools()` | 执行期通过 alias 做 deprecated 工具 fallback |
| `src/utils/permissions/permissionSetup.ts` | `getToolsForDefaultPreset()` | 解析 `--tools` CLI 参数 |
| `src/tools/AgentTool/AgentTool.tsx` | `assembleToolPool()` | 子代理启动时重建可用工具 |

---

## 二、功能点目的

### 2.1 工具预设（Tool Presets）

- `TOOL_PRESETS = ['default']`：目前仅支持 `default`。
- `parseToolPreset()`：把 CLI 传入的字符串正规化为预设枚举。
- `getToolsForDefaultPreset()`：返回所有 `isEnabled() === true` 的 base tool 名称列表。

### 2.2 Base Tool 注册表（`getAllBaseTools`）

这是**所有可能被暴露的内置工具的穷尽列表**。其设计要点：

- **编译期死代码消除（dead code elimination）**：大量工具通过 `feature('XXX')` 或 `process.env.USER_TYPE === 'ant'` 做条件 `require()`，外部构建会把未命中的分支摇掉，减小包体积。
- **动态 import / lazy require**：对 `TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool` 等使用 getter + `require()`，打破循环依赖（`tools.ts → TeamCreateTool → ... → tools.ts`）。
- **嵌入式搜索工具降级**：当 `hasEmbeddedSearchTools()` 为 true（Ant-native bun 内置了 bfs/ugrep），`GlobTool` 和 `GrepTool` 被省略，因为 Bash 里已 alias 到更快实现。
- **测试工具隔离**：`TestingPermissionTool` 仅在 `NODE_ENV === 'test'` 时注册。

### 2.3 权限上下文过滤（`filterToolsByDenyRules`）

在模型**看到**工具列表之前，先把被 blanket-deny 的工具剔除。这包括：

- 普通内置工具（如 `Bash` 整条规则 deny）。
- MCP 服务器级 deny（如 `mcp__serverName` 规则会一次性剔除该服务器下所有工具）。
- 匹配逻辑复用运行时权限系统的 `getDenyRuleForTool()`（见 `src/utils/permissions/permissions.ts`）。

### 2.4 运行时工具池（`getTools`）

在 `getAllBaseTools` 基础上再做**模式级裁剪**：

- **Simple 模式**（`CLAUDE_CODE_SIMPLE`）：只保留 `BashTool`、`FileReadTool`、`FileEditTool`；若同时开启 coordinator 模式，再追加 `AgentTool`、`TaskStopTool`、`SendMessageTool`。
- **REPL 模式**：若 REPL 启用且 `REPLTool` 在列表中，则把 `REPL_ONLY_TOOLS`（Bash、Read、Edit、Glob、Grep 等）隐藏，强制模型走 REPL 统一入口。
- **最后 `isEnabled()` 过滤**：每个 tool 实例的 `isEnabled()` 再做一次运行时开关校验。

### 2.5 工具池合并（`assembleToolPool` / `getMergedTools`）

- `assembleToolPool`：**REPL 与 runAgent 共享的纯函数**，负责：
  1. 取内置工具（`getTools()`）。
  2. 对 MCP 工具应用 deny 规则过滤。
  3. **分区排序**：built-in 按名称排序后在前，MCP 按名称排序后在后——这是为了兼容服务端 `claude_code_system_cache_policy` 的缓存断点策略（built-in 必须保持连续前缀）。
  4. `uniqBy(..., 'name')` 去重，built-in 同名优先。

- `getMergedTools`：与 `assembleToolPool` 类似，但**不做去重**，用于 token 计数、tool search 阈值计算等需要完整列表的场合。

---

## 三、具体技术实现

### 3.1 关键数据结构

#### Tool 类型（定义于 `src/Tool.ts`）

```ts
export type Tool<Input, Output, P> = {
  name: string
  aliases?: string[]
  searchHint?: string
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  inputSchema: Input          // Zod schema
  inputJSONSchema?: ToolInputJSONSchema  // MCP 工具可直接写 JSON Schema
  outputSchema?: z.ZodType
  isConcurrencySafe(input): boolean
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  isSearchOrReadCommand?(input): { isSearch, isRead, isList }
  shouldDefer?: boolean      // 是否延迟加载（需 ToolSearch）
  alwaysLoad?: boolean       // 是否永不延迟
  mcpInfo?: { serverName, toolName }
  maxResultSizeChars: number
  strict?: boolean
  // ... 大量 UI/渲染/权限/分类器相关方法
}
```

#### Tools 类型别名

```ts
export type Tools = readonly Tool[]
```

该别名被刻意引入，以便在类型层面追踪“工具集合”在代码库中的流动路径。

### 3.2 条件加载与死代码消除

文件顶部使用 `/* eslint-disable custom-rules/no-process-env-top-level */` 允许在模块顶层读取 `process.env` 和 `feature()`，这是为了配合打包器的 DCE：

```ts
const REPLTool =
  process.env.USER_TYPE === 'ant'
    ? require('./tools/REPLTool/REPLTool.js').REPLTool
    : null
```

外部构建中 `USER_TYPE !== 'ant'`，打包器可将整个 `REPLTool` 模块及依赖树消除。

### 3.3 循环依赖破解

`TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool` 通过 getter 延迟 `require`：

```ts
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

在 `getAllBaseTools()` 中调用 `getTeamCreateTool()`，此时模块图已稳定，避免静态 import 导致的循环依赖报错。

### 3.4 Prompt-Cache 稳定性排序

`assembleToolPool` 中的排序逻辑极其重要：

```ts
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

- 不使用 `Array.toSorted`（Node 20+），保证 Node 18 兼容。
- built-in 分区在前、MCP 分区在后，确保服务端缓存策略的 prefix breakpoint 有效。若混排，则 MCP 工具插入 built-in 之间会导致下游所有缓存键失效。

### 3.5 REPL 模式下的工具隐藏

```ts
if (isReplModeEnabled()) {
  const replEnabled = allowedTools.some(tool => toolMatchesName(tool, REPL_TOOL_NAME))
  if (replEnabled) {
    allowedTools = allowedTools.filter(tool => !REPL_ONLY_TOOLS.has(tool.name))
  }
}
```

- 仅当 `REPLTool` 确实在允许列表中时才隐藏原始工具；若用户通过权限规则把 `REPLTool` deny 掉，则原始工具重新可见，避免模型无工具可用。

---

## 四、关键代码路径与文件引用

### 4.1 直接依赖（被 import/require）

| 依赖文件 | 用途 |
|---------|------|
| `src/Tool.js` | `Tool`/`Tools` 类型、`toolMatchesName`、`buildTool` |
| `src/constants/tools.js` | 各类模式允许/禁止工具常量 |
| `src/utils/toolSearch.js` | `isToolSearchEnabledOptimistic` |
| `src/utils/tasks.js` | `isTodoV2Enabled` |
| `src/utils/permissions/permissions.js` | `getDenyRuleForTool` |
| `src/utils/embeddedTools.js` | `hasEmbeddedSearchTools` |
| `src/utils/envUtils.js` | `isEnvTruthy` |
| `src/utils/shell/shellToolUtils.js` | `isPowerShellToolEnabled` |
| `src/utils/agentSwarmsEnabled.js` | `isAgentSwarmsEnabled` |
| `src/utils/worktreeModeEnabled.js` | `isWorktreeModeEnabled` |
| `src/tools/REPLTool/constants.js` | `REPL_TOOL_NAME`, `REPL_ONLY_TOOLS`, `isReplModeEnabled` |
| `src/tools/SyntheticOutputTool/SyntheticOutputTool.js` | `SYNTHETIC_OUTPUT_TOOL_NAME` |
| `bun:bundle` | `feature()` 编译期特性门控 |
| `lodash-es/uniqBy.js` | 按 name 去重 |

### 4.2 下游调用方文件网络

```
src/tools.ts
├── src/main.tsx                    (REPL 启动取工具)
├── src/cli/print.ts                (headless 取工具)
├── src/hooks/useMergedTools.ts     (React 层工具合并)
├── src/utils/toolPool.ts           (mergeAndFilterTools)
├── src/services/tools/toolExecution.ts  (fallback 查找)
├── src/utils/permissions/permissionSetup.ts (preset 解析)
└── src/tools/AgentTool/AgentTool.tsx     (子代理装配)
```

### 4.3 权限过滤链路

```
getTools(permissionContext)
  → filterToolsByDenyRules(baseTools, permissionContext)
      → getDenyRuleForTool(permissionContext, tool)
          → src/utils/permissions/permissions.ts
```

`getDenyRuleForTool` 支持 MCP 服务器级规则（`mcp__serverName` 或 `mcp__serverName__*`），可在模型层就把整个 MCP 服务器的工具全部屏蔽。

---

## 五、依赖与外部交互

### 5.1 编译期/运行时混合门控

`tools.ts` 是编译期 DCE 与运行时 flag 的交汇点：

- **编译期**：`feature('XXX')` 和 `process.env.USER_TYPE` 在打包时由 bun 的 `--define` 注入，未命中的 `require()` 分支会被摇树优化掉。
- **运行时**：`isEnabled()`、`isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)`、`isReplModeEnabled()` 等是进程启动后才确定的。

### 5.2 MCP 协议集成

- MCP 工具不是在本文件里静态 import 的，而是通过 `appState.mcp.tools` 动态传入。
- `assembleToolPool` 是 MCP 工具与内置工具合并的**唯一官方入口**，确保 REPL 和 headless 两条路径行为一致。
- MCP 工具带有 `mcpInfo: { serverName, toolName }`，用于 deny 规则匹配和 telemetry。

### 5.3 Tool Search / Deferred Loading

- `isToolSearchEnabledOptimistic()` 决定是否在 base tool 列表中**预先包含** `ToolSearchTool`。
- 真正的 defer 决策（是否给 API 发 `defer_loading: true`）在 `src/query.ts`（或 claude.ts）中根据模型、阈值、token 数做最终判定。
- `getAllBaseTools()` 中的注释明确提醒：该列表必须与 Statsig Dynamic Config (`claude_code_global_system_caching`) 保持同步，否则系统提示缓存会失效。

### 5.4 Coordinator 模式

- `feature('COORDINATOR_MODE')` 下，`coordinatorModeModule.isCoordinatorMode()` 判断当前进程是 coordinator 还是 worker。
- Simple 模式下，coordinator 额外获得 `AgentTool`、`TaskStopTool`、`SendMessageTool`，而 worker 只保留 `Bash/Read/Edit`（由 `filterToolsForAgent` 在别处裁剪）。

---

## 六、风险、边界与改进建议

### 6.1 当前风险

#### R1: `getAllBaseTools()` 与远程缓存配置的人工同步负担

文件中有一段醒目注释：

> "NOTE: This MUST stay in sync with https://console.statsig.com/.../claude_code_global_system_caching"

`getAllBaseTools` 的顺序和成员直接影响服务端 prompt cache 的 breakpoint 匹配。若 Statsig 配置更新而代码未同步，会导致：
- 缓存命中率骤降。
- 不同用户/版本间缓存键不一致。

**建议**：在 CI 中增加 lint 规则或自动化测试，校验 `getAllBaseTools()` 输出与 Statsig config snapshot 的一致性。

#### R2: 条件 `require()` 的 TypeScript 类型脆弱性

大量 ant-only 工具使用 `const X = condition ? require(...).X : null` 模式。虽然外部构建会 DCE，但类型推断上 `X` 是 `Tool | null`。在 `getAllBaseTools` 的数组构造中通过 spread 条件过滤（`...(X ? [X] : [])`），若有人误写成直接 push `null`，运行时可能把 `null` 注入工具列表。

**建议**：给条件 require 的结果包一层类型断言或 helper：`maybeIncludeTool(condition, () => require(...).X)`。

#### R3: `getTools()` 与 `assembleToolPool()` 的重复过滤

`getTools()` 内部已经做了 `filterToolsByDenyRules` 和 `isEnabled()` 过滤；`assembleToolPool()` 又调用一次 `getTools()`，然后对 MCP 工具再做一次 `filterToolsByDenyRules`。虽然逻辑正确，但 deny 规则的扫描是 O(rules × tools) 的重复计算。

**建议**：在性能敏感路径（如每轮 API 请求前）考虑把 `filterToolsByDenyRules` 的结果 memoize，或把 MCP deny 过滤下沉到 `getTools()` 的一个统一入口中。

#### R4: REPL 工具隐藏的边界情况

REPL 隐藏逻辑只检查 `REPLTool` 是否在列表中，但如果用户通过权限规则把 `REPLTool` 设为 `ask`（而非 deny），则模型仍可能直接调用原始工具，绕过 REPL 的批处理与沙箱语义。

**建议**：明确文档化 REPL 模式与权限规则的交互行为，或把 REPL 隐藏逻辑与权限决策结果（而非仅 deny 规则）联动。

#### R5: 循环依赖的延迟 require 是运行时开销

`getTeamCreateTool` 等 getter 在 `getAllBaseTools()` 每次调用时都会执行 `require()`。虽然 Node/Bun 的 require 有模块缓存，但函数调用和属性访问仍有微小开销。该函数在 REPL 每轮渲染中可能被调用多次（通过 `useMergedTools` 的 `useMemo`，但 deps 变化时仍会重算）。

**建议**：把条件工具实例缓存到模块级变量，首次 require 后复用：

```ts
let cachedTeamCreateTool: typeof TeamCreateTool | null | undefined
function getTeamCreateTool() {
  if (cachedTeamCreateTool === undefined) {
    cachedTeamCreateTool = require(...).TeamCreateTool ?? null
  }
  return cachedTeamCreateTool
}
```

### 6.2 边界行为

| 边界场景 | 行为 |
|---------|------|
| `CLAUDE_CODE_SIMPLE=1` + REPL 启用 | 返回 `[REPLTool]`（或加 coordinator 工具），原始工具全部隐藏 |
| `REPLTool` 被 deny | 原始工具重新可见，模型可直接调用 Bash/Read/Edit |
| MCP 工具与内置工具同名 | `assembleToolPool` 中 built-in 优先（`uniqBy` 保留先出现的） |
| `feature('COORDINATOR_MODE')` 关闭 | 相关代码在打包期被 DCE，`coordinatorModeModule` 为 `null` |
| `hasEmbeddedSearchTools() === true` | `GlobTool` 和 `GrepTool` 从 base 列表中移除 |
| `NODE_ENV === 'test'` | `TestingPermissionTool` 被注册，用于测试权限流 |

### 6.3 改进建议汇总

1. **缓存条件工具实例**：避免每次 `getAllBaseTools()` 都重新 `require()` feature-gated 模块。
2. **统一 deny 过滤入口**：把 MCP deny 过滤合并进 `getTools()` 的一个变体，减少重复扫描。
3. **增加与 Statsig 缓存配置的自动化一致性校验**：防止人工同步遗漏。
4. **文档化模式交互矩阵**：Simple × REPL × Coordinator × Headless 四种模式下工具可见性的组合行为目前分散在代码注释中，建议整理成决策表。
5. **考虑把 `getAllBaseTools` 拆分为静态注册表 + 动态过滤器**：当前函数既负责“罗列所有可能工具”又负责“根据运行时 flag 过滤”，职责略重。可拆成 `ALL_TOOLS_REGISTRY`（静态只读数组）和 `getEnabledBaseTools()`（动态过滤），提升可测试性。

---

*文档结束*
