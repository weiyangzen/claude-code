# Research: `src/utils/shell/specPrefix.ts`

## 场景与职责

`specPrefix.ts` 是 Claude Code **权限前缀系统的共享核心引擎**。它基于 Fig autocomplete 的 `CommandSpec` 定义，为 Bash 和 PowerShell 两条 Shell 路径提供**统一的命令前缀提取能力**。

其核心职责：

1. **从命令参数数组中提取有意义的命令前缀**：例如 `git -C /repo status --short` → `git status`。它利用 spec 中 `options` 的定义知道 `-C` 接受一个值，从而跳过 `/repo`，将 `status` 识别为子命令。
2. **为权限系统的 "Yes, and don't ask again for: ___" 功能生成合理的 wildcard 规则**：前缀过宽会扩大权限范围（`git:*` 允许所有 git 子命令），前缀过窄会产生死规则（`git show 81210f8:*` 永不再现）。`specPrefix.ts` 负责在这两者之间找到平衡。
3. **跨 Shell 复用**：git、npm、kubectl 等外部 CLI 的行为与 shell 无关，因此 Bash 的前缀提取器（`src/utils/bash/prefix.ts`）和 PowerShell 的前缀提取器（`src/utils/powershell/staticPrefix.ts`）共享此模块，避免重复实现。

## 功能点目的

### 1. `buildPrefix(command, args, spec)`

- **目的**：根据命令名、参数数组和 Fig spec，构建最有意义的前缀字符串。
- **关键行为**：
  - 调用 `calculateDepth()` 确定前缀最多应包含多少个词（word）。
  - 遍历 `args`，跳过全局 flag 及其值（如 `git -C /repo` 跳过 `-C` 和 `/repo`）。
  - 对 `python -c` 做特殊截断：遇到 `-c` 立即停止，避免将脚本内容纳入前缀。
  - 对 `isCommand` / `isModule` flag（如 `node --eval`、`-m`）做保留：这些 flag 本身应纳入前缀，因为它们改变了后续参数的含义。
  - 当遇到文件路径、URL、未知子命令时，通过 `shouldStopAtArg()` 停止消费。
  - 最终返回空格拼接的前缀字符串。

### 2. `calculateDepth(command, args, spec)`

- **目的**：决定前缀应包含的最大词数（深度）。这是防止前缀过宽或过窄的核心算法。
- **优先级（从高到低）**：
  1. **`DEPTH_RULES` 硬编码表**：针对 `gcloud`、`aws`、`kubectl`、`docker`、`dotnet`、`git push` 等命令，以及动态导入在 native/node 构建中不生效时的兜底规则。
  2. **Fig spec 中的 `isCommand` / `isVariadic` 标记**：若子命令参数被标记为 `isCommand`，深度通常为 3；若为 `isVariadic`，深度为 2。
  3. **子命令嵌套深度**：若子命令本身还有子命令（如 `gcloud scheduler jobs`），深度为 4。
  4. **叶子子命令无参数**：如 `git show`、`git log`、`git tag` 未声明 `args` 时，深度为 2（避免过度具体的 SHA/ref/tag 规则）。
  5. **无 spec 时默认深度为 2**。

### 3. `DEPTH_RULES: Record<string, number>`

- **目的**：为那些在运行时无法动态加载 Fig spec 的命令（native/node 构建中动态 `import` 不工作）提供硬编码的深度规则。
- **覆盖范围**：
  - 通用深度：`rg: 2`、`pre-commit: 2`、`gcloud: 4`、`aws: 4`、`az: 4`、`kubectl: 3`、`docker: 3`、`dotnet: 3`
  - 复合键深度：`gcloud compute: 6`、`gcloud beta: 6`、`git push: 2`

### 4. `isKnownSubcommand(arg, spec)` / `flagTakesArg(flag, nextArg, spec)` / `findFirstSubcommand(args, spec)`

- **目的**：辅助 `buildPrefix` 和 `calculateDepth` 识别子命令、判断 flag 是否带值、在参数列表中定位第一个子命令。
- **大小写不敏感**：Fig spec 中的 `subcommand.name` 为小写，但调用方可能传入原始大小写（尤其是 PowerShell），因此比较时统一转小写。

### 5. `shouldStopAtArg(arg, priorArgs, spec)`

- **目的**：判断当前参数是否应作为前缀的终止点。
- **停止条件**：
  - 参数以 `-` 开头（flag）。
  - 参数包含 `/` 或文件扩展名（视为文件路径）。
  - 参数以 `http://`、`https://`、`ftp://` 开头（视为 URL）。
- **例外**：若前一个参数是 `-m` 且 spec 声明其参数为 `isModule`，则不停止（允许 `python -m http.server` 中的 `http.server` 被纳入前缀）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```ts
// 针对无 spec 或动态导入失败命令的硬编码深度规则
export const DEPTH_RULES: Record<string, number> = {
  rg: 2,
  'pre-commit': 2,
  gcloud: 4,
  'gcloud compute': 6,
  'gcloud beta': 6,
  aws: 4,
  az: 4,
  kubectl: 3,
  docker: 3,
  dotnet: 3,
  'git push': 2,
}

// URL 协议前缀，用于 shouldStopAtArg
const URL_PROTOCOLS = ['http://', 'https://', 'ftp://']
```

### `buildPrefix` 核心流程

```
输入: command, args, spec
1. maxDepth = calculateDepth(command, args, spec)
2. parts = [command]
3. foundSubcommand = false
4. 遍历 args:
   a. 若 parts.length >= maxDepth → break
   b. 若 arg 以 '-' 开头:
      - python -c 特殊处理 → break
      - isCommand/isModule flag → push(arg), continue
      - 有子命令且未找到子命令 → flagTakesArg? 跳过值, continue
      - 否则 → break
   c. 若 shouldStopAtArg(arg) → break
   d. 若 hasSubcommands && !foundSubcommand → foundSubcommand = isKnownSubcommand(arg, spec)
   e. parts.push(arg)
5. 返回 parts.join(' ')
```

### `calculateDepth` 决策流程

```
1. firstSubcommand = findFirstSubcommand(args, spec)
2. key = firstSubcommand ? `${commandLower} ${firstSubcommandLower}` : commandLower
3. 若 DEPTH_RULES[key] 存在 → 返回该值
4. 若 DEPTH_RULES[commandLower] 存在 → 返回该值
5. 若 !spec → 返回 2
6. 若 args 中有 isCommand/isModule flag → 返回 3
7. 若 firstSubcommand 存在且匹配 spec.subcommands:
   - 子命令 args 中有 isCommand → 3
   - 子命令 args 中有 isVariadic → 2
   - 子命令有嵌套 subcommands → 4
   - 子命令无 args → 2（叶子子命令，避免过度具体）
   - 否则 → 3
8. 若 spec.args:
   - 有 isCommand → 根据位置返回 2 或 3
   - 无 subcommands 且 isVariadic → 1
   - 无 subcommands 且首参数非 optional → 2
9. 若 spec.args 中有 isDangerous → 3，否则 → 2
```

### 大小写处理

- `command` 统一转小写用于 `DEPTH_RULES` 键查找。
- `isKnownSubcommand` 中将 `arg` 和 `sub.name` 均转小写后比较，确保 PowerShell 的 `Git Status` 能匹配到 fig spec 的 `status`。

## 关键代码路径与文件引用

### 直接依赖（被 import）

| 文件 | 导入符号 | 作用 |
|------|---------|------|
| `src/utils/bash/registry.ts` | `CommandSpec` (type) | Fig spec 的类型定义 |

### 下游调用方

| 调用方文件 | 调用符号 | 场景 |
|-----------|---------|------|
| `src/utils/bash/prefix.ts` | `buildPrefix` | Bash 权限前缀提取：普通非包装命令的前缀构建 |
| `src/utils/powershell/staticPrefix.ts` | `buildPrefix`, `DEPTH_RULES` | PowerShell 权限前缀提取：外部命令的前缀构建与裸根守卫 |

### 调用链示例

**Bash 路径：**
```
src/utils/bash/prefix.ts:getCommandPrefixStatic()
  └── buildPrefix(cmd, args, spec)
        ├── calculateDepth(cmd, args, spec)
        │     ├── DEPTH_RULES
        │     └── findFirstSubcommand()
        └── shouldStopAtArg()
```

**PowerShell 路径：**
```
src/utils/powershell/staticPrefix.ts:extractPrefixFromElement()
  ├── getCommandSpec(nameLower)
  └── buildPrefix(name, cmd.args, spec)
        ├── calculateDepth()
        └── 位置完整性校验（PowerShell 独有）
```

## 依赖与外部交互

### Fig Spec 基础设施

- `src/utils/bash/registry.ts` 中的 `getCommandSpec()` 负责加载 spec：
  1. 先搜索本地 specs（`src/utils/bash/specs/index.js`）。
  2. 再动态导入 `@withfig/autocomplete/build/${command}.js`。
  3. 使用 `memoizeWithLRU` 缓存结果。
- `specPrefix.ts` 本身不直接加载 spec，只消费 `CommandSpec` 类型和结构。

### 与 PowerShell 的交互

PowerShell 的前缀提取器（`staticPrefix.ts`）在调用 `buildPrefix` 后会执行额外的**位置完整性校验**和**裸根守卫**：
- **位置完整性校验**：确保 `buildPrefix` 返回的每个词都能按顺序精确匹配原始 `cmd.args` 中的位置参数，防止单引号内空格被错误拆分（`git 'push origin'` → `git push origin` 的歧义）。
- **裸根守卫**：若 `buildPrefix` 只返回单个词（如 `git`），且该命令在 spec 中声明了子命令或存在于 `DEPTH_RULES` 中，则拒绝该前缀，避免生成过于宽泛的 `git:*` 规则。

这些补丁在 Bash 侧不存在（或存在于其他模块），体现了 PowerShell 引号处理和大小写不敏感特性带来的额外安全需求。

## 风险、边界与改进建议

### 风险

1. **`DEPTH_RULES` 的维护负担**
   硬编码表需要手动与 `@withfig/autocomplete` 包中的 spec 变化保持同步。若某个命令的 spec 新增了深层子命令结构（如 `docker` 新增 `docker compose buildx`），而 `DEPTH_RULES` 未及时更新，可能导致前缀过宽或过窄。

2. **`calculateDepth` 的隐式优先级复杂**
   深度决策涉及 `DEPTH_RULES`（复合键 → 命令键）→ spec 选项扫描 → 子命令分析 → `spec.args` 分析的多层 fallback。新增规则时容易误触更高优先级的分支，导致预期外的深度。

3. **`shouldStopAtArg` 的文件扩展名判断可能误判**
   ```ts
   const hasExtension = dotIndex > 0 && dotIndex < arg.length - 1 && !arg.substring(dotIndex + 1).includes(':')
   ```
   该逻辑会将有扩展名的参数视为文件路径并停止。对于像 `npm run build.js` 这样的命令，`build.js` 会被误判为文件路径而停止，前缀可能只保留 `npm run`。虽然这通常是安全的（避免将具体脚本名纳入规则），但也可能降低前缀的可用性。

4. **无 spec 时的默认深度为 2 可能过宽**
   对于不在 `DEPTH_RULES` 中且无法加载 Fig spec 的命令，`calculateDepth` 返回 2。这意味着 `unknown-tool deploy` 的前缀将是 `unknown-tool deploy`。若该命令的子命令体系实际上更深（如 `unknown-tool deploy prod`），则前缀可能过窄；反之若它无子命令概念，则 2 可能刚好。

5. **动态导入在 native/node 构建中的失效**
   文件顶部注释明确指出："dynamic imports don't work in native/node builds"。这意味着外部构建中大量 Fig spec 无法加载，系统严重依赖 `DEPTH_RULES` 的覆盖范围。未被覆盖的命令将获得默认深度 2，前缀质量下降。

### 边界

- **仅处理已拆分的参数数组**：`buildPrefix` 接收的是 `args: string[]`，不负责命令字符串的解析拆分。Bash 侧由 tree-sitter / `extractCommandArguments` 负责，PowerShell 侧由 AST parser 负责。
- **不处理管道 `|`**：管道命令的拆分和子命令提取在调用方（`prefix.ts`、`staticPrefix.ts`）完成，`specPrefix` 只处理单个命令元素。
- **flag 值跳过依赖 spec 准确性**：若 Fig spec 中某个 flag 的 `args` 标记缺失，`flagTakesArg` 会回退到启发式判断（"下一个参数不以 `-` 开头且不是已知子命令"），这在某些边缘场景（如 flag 值为负数 `-1`）可能误判。
- **大小写处理仅限比较**：`buildPrefix` 保留原始 `command` 和 `args` 的大小写输出，但内部比较时统一转小写。因此 `Git Status` 的前缀输出仍是 `Git Status`（保留用户输入大小写）。

### 改进建议

1. **将 `DEPTH_RULES` 的维护自动化**
   在 CI 中增加一个脚本，扫描 `@withfig/autocomplete` 包中所有 spec 的嵌套深度，与 `DEPTH_RULES` 做 diff 报警。或者至少维护一份注释说明每个规则的来源依据。

2. **为 `shouldStopAtArg` 增加模块名例外**
   对于 `python -m`、`node --eval` 等已知的 `isModule` / `isScript` 场景，当前已有 `-m` 的特殊处理。但像 `npm run build.js` 这种非路径但被误判为文件扩展名的情况，可考虑引入一个 `SCRIPT_EXTENSIONS` 白名单，仅当扩展名属于常见脚本类型（`.sh`, `.ps1`, `.py`, `.js`）时才停止，而 `.js` 在 `npm run` 上下文中实际上是任务名而非文件路径。
   
   更安全的做法：将扩展名判断与 `isScript` / `isCommand` 的 spec 标记联动，仅当 spec 未声明该位置为脚本时才按文件路径停止。

3. **增加 `buildPrefix` 和 `calculateDepth` 的单元测试**
   仓库中未找到针对 `specPrefix.ts` 的测试文件。建议补充覆盖：
   - 各 `DEPTH_RULES` 命令的深度计算；
   - flag 值跳过（`git -C /repo status` → `git status`）；
   - `python -c` 的特殊截断；
   - `isCommand` / `isModule` flag 的保留；
   - 文件路径 / URL 的停止行为；
   - 无 spec 时的默认深度；
   - 大小写不敏感的子命令匹配。

4. **改善 native/node 构建中的 spec 可用性**
   若动态导入在 native 构建中确实不可行，可考虑在构建时将常用 Fig spec 静态打包进产物（通过构建脚本扫描并生成 import 语句），而不是完全依赖 `DEPTH_RULES` 兜底。这能显著提升外部构建的前缀提取质量。

5. **统一 Bash 与 PowerShell 的裸根守卫**
   当前 PowerShell 侧有裸根守卫（拒绝单字前缀如 `git`），而 Bash 侧的 `prefix.ts` 注释中提到 "Bash's extractor doesn't gate this (bash/prefix.ts:363, separate fix)"。建议将裸根守卫下沉到 `specPrefix.ts` 的 `buildPrefix` 或 `calculateDepth` 中，使两条 Shell 路径共享同一安全策略，避免维护两份独立的补丁。

6. **为 `calculateDepth` 增加更详细的返回注释**
   该函数目前通过多个 `return` 语句在不同分支返回数字，但缺少统一的日志或注释说明"为什么这个命令的深度是 N"。在调试前缀过宽/过窄问题时，开发者需要反复阅读代码才能推断原因。建议在每个 `return` 前增加结构化注释或调试日志。
