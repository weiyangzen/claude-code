# Research: `src/utils/powershell/staticPrefix.ts`

## 场景与职责

`staticPrefix.ts` 是 PowerShell 权限对话框中 **"Yes, and don't ask again for: ___"** 可编辑输入的**静态前缀提取器**。它的职责是：

1. 在用户收到 PowerShell 命令的权限询问时，自动给出一个最合理的 wildcard 前缀建议（如 `git status:*`、`Get-Process:*`）；
2. 对于复合命令（`Get-Process; git status && npm test`），为每个子命令提取前缀，并通过**词对齐最长公共前缀（LCP）**进行合并，减少用户需要授权的条目数；
3. 过滤掉已自动放行（read-only / allowlisted）的子命令，避免对无害命令生成无意义的建议；
4. 严格阻止对危险 cmdlet（如 `Invoke-Expression`、`Start-Process`）和路径型调用生成过于宽泛的 wildcard 规则，防止一次授权后永久绕过安全校验。

该文件在架构上**镜像了 Bash 的前缀提取器**（`src/utils/bash/prefix.ts`），但底层使用 PowerShell AST 解析器（`parser.ts`）而非 tree-sitter，因为 PowerShell 的命令元素拆分、引号处理、参数绑定逻辑与 Bash 截然不同。

## 功能点目的

### 1. `getCommandPrefixStatic`
对单条 PowerShell 命令提取一个前缀字符串（或 `null`）。流程：
- 调用 `parsePowerShellCommand` 获取 AST；
- 使用 `getAllCommands` 找到第一个 `CommandAst`；
- 调用 `extractPrefixFromElement` 进行实际提取。

### 2. `getCompoundCommandPrefixesStatic`
对复合命令（含 `;`、`&&`、`||` 等）提取多个前缀，支持：
- `excludeSubcommand` 回调过滤（如跳过 read-only 子命令）；
- 按根命令分组后做词对齐 LCP 合并；
- 若 LCP 退化为单个词且该根命令具有 fig spec 子命令结构，则**丢弃整组**（避免生成 `git:*` 这种过于宽泛的规则）。

### 3. `extractPrefixFromElement`
核心的单命令前缀提取逻辑，包含多层安全门控：
- `nameType === 'application'` → 拒绝（路径型调用，如 `.\script.ps1`）；
- `NEVER_SUGGEST.has(name.toLowerCase())` → 拒绝（危险 cmdlet）；
- `nameType === 'cmdlet'` → 直接返回 cmdlet 名称（PowerShell cmdlet 无子命令概念）；
- 外部命令 → 检查命令名和参数是否全为静态字面量（`StringConstant` / `Parameter`），然后调用 fig-spec 驱动的 `buildPrefix`；
- `buildPrefix` 后做**位置完整性校验**：确保 prefix 中的每个词都能按顺序精确匹配到原始 `cmd.args` 中的某个位置参数，防止单引号内空格被错误拆分为多个词（如 `git 'push origin'` 被拆成 `git push origin`）；
- **裸根守卫**：若 prefix 只有单个词且该命令在 fig spec 中声明了子命令（或有 `DEPTH_RULES` 条目），则拒绝（避免 `git:*`、`npm:*` 等过于宽泛的规则）。

### 4. `wordAlignedLCP`
词对齐最长公共前缀。比较时不截断单词内部：
- `["npm run test", "npm run lint"]` → `"npm run"`
- 大小写不敏感（PowerShell 本身不区分大小写），但输出保留第一组的原始大小写。

## 具体技术实现

### 4.1 外部命令的前缀深度计算（fig spec 复用）
PowerShell 的外部命令（`git`、`npm`、`kubectl` 等）与 shell 无关，因此 `staticPrefix.ts` 直接复用 Bash 的 fig spec 基础设施：
- `getCommandSpec(nameLower)`：异步加载 `@withfig/autocomplete` spec（LRU 缓存）；
- `buildPrefix(name, args, spec)`：根据 spec 的 `subcommands`、`options`、`args` 定义，跳过全局 flag 及其值，找到有意义的子命令深度。

例如：
```powershell
git -C /repo status --short
```
`buildPrefix` 知道 `-C` 接受一个值，因此跳过 `/repo`，将 `status` 识别为子命令，`--short` 是 flag 被截断，最终前缀为 `git status`。

### 4.2 位置完整性校验（Post-buildPrefix Word Integrity）
这是 PowerShell 侧独有的安全补丁（bash/prefix.ts 没有等价逻辑）。`buildPrefix` 返回的字符串是空格拼接的，但 PowerShell 的 AST 解析器对**单引号字符串**会保留内部空格作为单个参数值（`parser.ts` 中 `isStringLiteral && ce.value != null ? ce.value : ce.text`）。

攻击示例：
```powershell
git 'push origin' push origin
```
`args = ['push origin', 'push', 'origin']`，`buildPrefix` 可能输出 `git push origin`。如果直接建议 `git push origin:*`，则用户实际运行 `git push origin --force`（3 个独立 argv）也会被匹配——但用户从未真正批准过 `push origin` 这个子命令组合，而是批准了一个被错误拆分的单引号参数。

校验算法：
```ts
let argIdx = 0
for (const word of prefix.split(' ').slice(1)) {
  if (word.includes('\\')) return null
  while (argIdx < cmd.args.length) {
    const a = cmd.args[argIdx]!
    if (a === word) break
    if (a.startsWith('-')) {
      argIdx++
      // 若 spec 表明该 flag 接受值，则再跳过一个参数
      if (spec?.options && /* flag takes value */) argIdx++
      continue
    }
    return null // 位置参数不匹配 → buildPrefix 拆分了某个 arg
  }
  if (argIdx >= cmd.args.length) return null
  argIdx++
}
```

### 4.3 裸根守卫（Bare-Root Guard）
若 `buildPrefix` 没有找到子命令（空参数或只有全局 flag），则返回单字前缀（如 `git`）。对于具有子命令结构的 CLI，这会生成 `git:*`，从而自动放行 `git push --force` 等任意子命令。

守卫逻辑：
```ts
if (
  !prefix.includes(' ') &&
  (spec?.subcommands?.length || DEPTH_RULES[nameLower])
) {
  return null
}
```
`DEPTH_RULES` 覆盖了无 spec 或动态导入失败的场景（`gcloud`、`aws`、`kubectl`、`az` 等）。

### 4.4 复合命令的 LCP 合并与二次守卫
`getCompoundCommandPrefixesStatic` 在分组 LCP 后，再次检查：
```ts
if (lcpWordCount <= 1) {
  const rootSpec = await getCommandSpec(rootLower)
  if (rootSpec?.subcommands?.length || DEPTH_RULES[rootLower]) {
    continue // 丢弃该组
  }
}
```
这是为了防止 `git add` + `git commit` 的 LCP 退化为 `git` 后，仍然建议 `git:*`。

## 关键代码路径与文件引用

| 导出符号 | 消费者 | 用途 |
|---------|--------|------|
| `getCommandPrefixStatic` | `src/utils/powershell/staticPrefix.ts` 内部 | 单命令前缀提取（`getCompoundCommandPrefixesStatic` 的单命令 fallback） |
| `getCompoundCommandPrefixesStatic` | `src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx` | 权限对话框的 `editablePrefix` 初始化 |

### 调用链详情
```
PowerShellPermissionRequest.tsx
  └── getCompoundCommandPrefixesStatic(command, element => isAllowlistedCommand(element, element.text))
        └── parsePowerShellCommand(command)
        └── getAllCommands(parsed)
        └── extractPrefixFromElement(cmd)
              ├── NEVER_SUGGEST (dangerousCmdlets.ts)
              ├── getCommandSpec(nameLower)  (bash/registry.js)
              ├── buildPrefix(name, args, spec)  (shell/specPrefix.js)
              └── 位置完整性校验 + 裸根守卫
```

## 依赖与外部交互

### 入依赖
- `src/utils/powershell/parser.js`：
  - `parsePowerShellCommand`：AST 解析入口；
  - `getAllCommands`：扁平化获取所有 `CommandAst`；
  - `ParsedCommandElement` 类型定义。
- `src/utils/powershell/dangerousCmdlets.js` → `NEVER_SUGGEST`：危险 cmdlet 黑名单，用于前缀建议门控。
- `src/utils/bash/registry.js` → `getCommandSpec`：fig spec 加载器（与 Bash 共享）。
- `src/utils/shell/specPrefix.js` → `buildPrefix`、`DEPTH_RULES`：fig-spec 驱动的前缀深度计算（与 Bash 共享）。
- `src/utils/stringUtils.js` → `countCharInString`：计算 LCP 中的空格数以判断词数。

### 出依赖
- `src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx`：UI 层消费 `getCompoundCommandPrefixesStatic` 的结果来初始化可编辑前缀输入框。

## 风险、边界与改进建议

### 风险
1. **异步前缀提取的竞态条件**：`PowerShellPermissionRequest.tsx` 使用 `useEffect` 异步调用 `getCompoundCommandPrefixesStatic`。如果用户在 `useEffect` resolve 之前手动编辑了前缀输入框（`hasUserEditedPrefix.current = true`），则异步结果会被丢弃。这是正确行为，但如果在高延迟环境（如 Windows Defender 导致 pwsh spawn 缓慢）下，用户可能看到输入框从原始命令 "闪烁" 到提取前缀的过程。
2. **位置完整性校验的 flag 值跳过依赖 spec**：当 spec 未加载或 `spec.options` 不存在时，代码走 `fail-safe` 路径——**不跳过 flag 的值**。这可能导致某些 flag 后面的位置参数被误判为不匹配，从而返回 `null`。虽然这是安全方向（不生成过于宽泛的规则），但会降低前缀建议的命中率。
3. **LCP 合并丢弃整组**：当用户运行 `git add file1 && git commit -m "msg"` 时，LCP 退化为 `git`，整组被丢弃，用户不会收到任何前缀建议。这比建议 `git:*` 更安全，但 UX 上用户可能需要手动输入规则。
4. **反斜杠拒绝**：位置完整性校验中 `if (word.includes('\\')) return null` 会拒绝所有含 Windows 路径的前缀（如 `git -C C:\repo status`）。这是为了避免过度具体的死规则，但也会导致常见 Windows 工作流缺少前缀建议。

### 边界
- **仅处理第一个 `CommandAst`**：`getCommandPrefixStatic` 使用 `.find(cmd => cmd.elementType === 'CommandAst')`，若管道以非命令表达式开头（如 `"$env:SECRET" | Write-Output`），则第一个元素是 `CommandExpressionAst`，会被跳过，最终可能返回 `{ commandPrefix: null }`。
- **单命令 vs 复合命令的行为差异**：`getCommandPrefixStatic` 对单命令做裸根守卫；`getCompoundCommandPrefixesStatic` 在 LCP 后再做一次守卫。两者守卫条件略有不同（后者还检查 `DEPTH_RULES[rootLower]`），但总体一致。
- **nameType === 'unknown' 的通过性**：对于既非 cmdlet 也非 application 的命令（如 `git` 在 PowerShell 中分类为 `unknown`，因为不含 `-` 也不含 `.\`），只要不在 `NEVER_SUGGEST` 中且通过 fig spec 提取，就可以生成前缀。这是正确行为，因为外部可执行文件在 PowerShell 中就是 `unknown`。

### 改进建议
1. **缓存 fig spec 查找结果**：`extractPrefixFromElement` 对每个外部命令都会调用 `getCommandSpec` 和 `buildPrefix`。在 `getCompoundCommandPrefixesStatic` 的循环中，同一命令可能出现多次（如 `git status && git log`）。虽然 `getCommandSpec` 本身有 LRU，但 `buildPrefix` 没有。可考虑在函数作用域内对 `(name, args, spec)` 做轻量级 memo。
2. **改善反斜杠路径的处理**：当前直接 `return null` 过于粗暴。可考虑将反斜杠路径中的空格问题与路径本身解耦：仅当反斜杠路径中包含空格时才拒绝（因为那是真正可能导致 `buildPrefix` 拆分歧义的情况），或者将反斜杠统一替换为正斜杠后再做位置校验。
3. **为无 spec 的外部命令提供启发式前缀**：当 `getCommandSpec` 返回 `null` 且不在 `DEPTH_RULES` 中时，`buildPrefix` 默认只返回命令名本身（因为 `calculateDepth` 在 `!spec` 时返回 2，但无子命令时 `buildPrefix` 循环不 push 任何 arg，最终仍是单命令名）。对于无 spec 的命令，可考虑采用更保守的启发式：取第一个非 flag 位置参数作为前缀（如 `mytool deploy` → `mytool deploy`），而不是直接返回 `mytool` 然后被裸根守卫拒绝。需要评估误报风险。
4. **增加单元测试覆盖**：仓库中未找到针对 `staticPrefix.ts` 的测试文件。建议补充测试用例覆盖：
   - 基本 cmdlet 前缀提取（`Get-Process` → `Get-Process`）；
   - 危险 cmdlet 拒绝（`Invoke-Expression` → `null`）；
   - 外部命令 fig spec 提取（`git status` → `git status`）；
   - 单引号空格防拆分（`git 'push origin'` → `null` 或正确前缀）；
   - 复合命令 LCP 合并与丢弃（`git add && git commit` → 无建议）；
   - 反斜杠路径行为（`git -C C:\repo status` → `null`）。
5. **与 Bash 前缀提取器的进一步统一**：`src/utils/bash/prefix.ts` 中的 `getCommandPrefixStatic` 支持 wrapper 命令递归（如 `nice git status`），而 PowerShell 版本没有。虽然 PowerShell 中 `nice` 等 wrapper 不常见，但 `cmd /c`、`pwsh -c` 等嵌套调用在 PowerShell 中是合法的。当前 `staticPrefix.ts` 对 `pwsh -c "..."` 只会提取 `pwsh`，而 `NEVER_SUGGEST` 已包含 `pwsh`，因此直接拒绝。这是安全的，但 UX 上用户无法为嵌套命令生成前缀规则。可考虑在未来引入轻量级的 wrapper 递归支持。
