# Research: src/utils/bash/prefix.ts

## 场景与职责

`prefix.ts` 是 Bash 工具权限系统的**同步前缀提取器**。它的核心任务是将用户输入的 bash 命令解析为"命令前缀"（command prefix），用于：
1. **权限 allowlist 的匹配键**：例如 `git commit -m "hello"` 提取为 `git commit`，用户可保存规则 `Bash(git commit:*)` 以允许未来同类命令。
2. **复合命令的批量前缀计算**：对于 `cd src && npm test`，为每个子命令分别提取前缀，再按根命令分组折叠。
3. **BashPermissionRequest 对话框的默认前缀建议**：`BashPermissionRequest.tsx` 调用 `getCompoundCommandPrefixesStatic` 来初始化可编辑前缀输入框，减少用户手动输入。

该模块处于**安全与用户体验的交界点**：前缀过宽会扩大权限范围（`git` 允许所有 git 子命令），前缀过窄会产生大量死规则（`git show 81210f8:*` 永不再现）。

## 功能点目的

### 1. `getCommandPrefixStatic(command, recursionDepth, wrapperCount)`
- **目的**：为单个命令字符串提取最有意义的前缀。
- **关键行为**：
  - 调用 `parseCommand()` 进行 tree-sitter 解析（或回退）。
  - 提取环境变量前缀（如 `GOEXPERIMENT=synctest go test` → 保留 `GOEXPERIMENT=synctest go test`）。
  - 识别"包装命令"（wrapper commands，如 `nice`、`timeout`、`sudo`），递归提取被包装命令的前缀。
  - 对普通命令，调用 `buildPrefix()` 基于 Fig spec 计算深度。
- **防护限制**：`wrapperCount > 2` 或 `recursionDepth > 10` 时返回 `null`，防止递归爆炸或恶意嵌套绕过。

### 2. `handleWrapper(command, args, recursionDepth, wrapperCount)`
- **目的**：处理包装命令的参数扫描。
- **逻辑**：
  - 优先使用 spec 中标记 `isCommand` 的参数位置，直接定位被包装命令。
  - 若无 spec，回退到启发式扫描：找第一个不以 `-` 开头、非纯数字、非 `KEY=val` 的参数作为被包装命令。

### 3. `getCompoundCommandPrefixesStatic(command, excludeSubcommand?)`
- **目的**：处理复合命令（含 `&&`、`||`、`;`）。
- **流程**：
  1. 用 `splitCommand_DEPRECATED()` 拆分为子命令。
  2. 对每个子命令调用 `getCommandPrefixStatic()`。
  3. 按根命令（第一个词）分组，组内通过**词边界最长公共前缀（LCP）**折叠。
     - 例：`["git fetch", "git worktree"]` → `"git"`；`["npm run test", "npm run lint"]` → `"npm run"`。

### 4. `longestCommonPrefix(strings)`
- **目的**：词边界对齐的 LCP 计算，确保折叠结果始终停在空格边界，不会产生半截词。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构
```ts
// 硬编码的包装命令白名单（无法通过 spec 表达其复杂行为）
const WRAPPER_COMMANDS = new Set(['nice'])

// 环境变量与数值正则
const NUMERIC = /^\d+$/
const ENV_VAR = /^[A-Za-z_][A-Za-z0-9_]*=/
```

### 关键流程
1. **解析阶段**：`parseCommand(command)` → 返回 `{ rootNode, envVars, commandNode, originalCommand }`。
2. **参数提取**：`extractCommandArguments(commandNode)` → 得到 `[cmd, arg1, arg2, ...]`。
3. **Spec 查询**：`getCommandSpec(cmd)` → 从本地 specs + `@withfig/autocomplete` 获取命令元数据。
4. **包装判定**：
   - `WRAPPER_COMMANDS.has(cmd)` **或** spec.args 中存在 `isCommand` 标记。
   - **例外**：若第一个参数匹配 spec 中的已知子命令，则取消包装判定（如 `git` spec 有 `isCommand args` 用于 alias，但 `git status` 应视为普通命令）。
5. **前缀构建**：
   - 包装命令 → `handleWrapper()` 递归。
   - 普通命令 → `buildPrefix(cmd, args, spec)`（位于 `src/utils/shell/specPrefix.ts`）。
6. **环境变量拼接**：最终前缀前追加 `envVars.join(' ')`。

### 复合命令折叠算法
```ts
// 按根命令分组
const groups = new Map<string, string[]>()
// 每组计算词边界 LCP
collapsed.push(longestCommonPrefix(group))
```

## 关键代码路径与文件引用

| 被引用文件 | 引用方式 | 作用 |
|-----------|---------|------|
| `src/utils/shell/specPrefix.ts` | `import { buildPrefix } from '../shell/specPrefix.js'` | 基于 Fig spec 构建前缀深度 |
| `src/utils/bash/commands.ts` | `import { splitCommand_DEPRECATED } from './commands.js'` | 复合命令拆分（legacy regex/shell-quote 路径） |
| `src/utils/bash/parser.ts` | `import { extractCommandArguments, parseCommand } from './parser.js'` | Tree-sitter 解析命令与参数提取 |
| `src/utils/bash/registry.ts` | `import { getCommandSpec } from './registry.js'` | 获取命令 spec |

### 调用方
| 调用方文件 | 调用符号 | 场景 |
|-----------|---------|------|
| `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` | `getCompoundCommandPrefixesStatic` | 权限对话框默认前缀建议 |

## 依赖与外部交互

- **Tree-sitter parser**（`parser.ts`）：提供 AST 级命令解析；当 tree-sitter 不可用时，`parseCommand()` 返回 `null`，但 `prefix.ts` 目前对此未做显式回退（会传播 `null`）。
- **Fig autocomplete npm 包**（`@withfig/autocomplete`）：`registry.ts` 动态导入其 build 产物，为数千个 CLI 工具提供参数语义。
- **本地 specs**（`src/utils/bash/specs/*.ts`）：对 Fig 未覆盖或行为特殊的命令做补充（如 `timeout`、`nohup`、`srun` 等）。

## 风险、边界与改进建议

### 风险
1. **Tree-sitter 回退时的空返回值**：若 `parseCommand()` 返回 `null`（如命令超长 >10000 字符、tree-sitter 加载失败），`getCommandPrefixStatic` 直接返回 `null`，调用方 `BashPermissionRequest` 会回退到更粗糙的 `getSimpleCommandPrefix`（仅取前两个词），可能生成过宽规则。
2. **包装命令的启发式扫描绕过**：`handleWrapper` 的回退扫描依赖参数不以 `-` 开头。构造如 `nice -- -c rm -rf /` 的恶意参数可能误导扫描（但 `wrapperCount` 和 `recursionDepth` 限制了攻击深度）。
3. **复合命令拆分依赖 legacy 路径**：`splitCommand_DEPRECATED` 基于 regex/shell-quote，对复杂引号、heredoc、转义操作符的处理存在已知的历史 bypass（参见 `commands.ts` 中的大量安全注释）。

### 边界
- 仅支持单个命令前缀提取和简单复合命令（`&&`、`||`、`;`）。管道 `|` 不被视为复合命令拆分点（由权限系统的其他模块处理）。
- `longestCommonPrefix` 的最小保留长度为 1 个词（`Math.max(1, commonWords)`），因此 `git fetch` 和 `git worktree` 至少保留 `git`。

### 改进建议
1. **统一复合命令拆分器**：当 tree-sitter 可用时，应使用 `treeSitterAnalysis.ts` 的 `extractCompoundStructure` 替代 `splitCommand_DEPRECATED`，消除 regex/shell-quote 路径的 parser differential。
2. **包装命令 spec 扩展**：目前 `WRAPPER_COMMANDS` 硬编码只有 `nice`，但 `env`、`time`、`stdbuf` 等也是常见包装命令，应考虑通过本地 spec 统一表达，减少硬编码。
3. **前缀提取失败时的降级策略**：当前返回 `null` 导致调用方完全放弃 tree-sitter 结果。可考虑在解析失败时回退到 `shell-quote` 解析（类似 `ParsedCommand.ts` 的做法），至少保留命令名的前缀。
