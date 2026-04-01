# readOnlyCommandValidation.ts 研究文档

## 场景与职责

`readOnlyCommandValidation.ts` 是 Claude Code CLI 中 **shell 工具只读命令白名单** 的权威数据源与校验引擎。它为 `BashTool` 和 `PowerShellTool` 提供跨平台共享的命令安全配置，包括 git、gh、docker、rg、pyright 等大量 CLI 工具的安全标志定义，以及一个通用的 `validateFlags` 标志遍历器。该文件是安全边界的核心组成部分，直接决定哪些命令可以在无需用户确认的情况下自动执行。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `GIT_READ_ONLY_COMMANDS` | 定义所有只读 git 子命令（如 `git diff`、`git log`、`git status`）及其安全标志。 |
| `GH_READ_ONLY_COMMANDS` | 定义 ant 内部使用的 `gh` CLI 只读子命令（如 `gh pr view`、`gh issue list`）。 |
| `DOCKER_READ_ONLY_COMMANDS` | 定义 docker 只读命令（`docker logs`、`docker inspect`）。 |
| `RIPGREP_READ_ONLY_COMMANDS` | 定义 `rg` 只读搜索的安全标志。 |
| `PYRIGHT_READ_ONLY_COMMANDS` | 定义 `pyright` 类型检查器的安全标志。 |
| `EXTERNAL_READONLY_COMMANDS` | 跨 shell 通用的只读外部命令列表（如 `docker ps`）。 |
| `containsVulnerableUncPath(pathOrCommand)` | Windows 专用：检测命令或路径中是否包含可能触发 NTLM/Kerberos 凭证泄漏或 WebDAV 攻击的 UNC 路径。 |
| `validateFlags(tokens, startIndex, config, options)` | 通用标志遍历校验器，按配置的安全标志表逐个 token 检查，支持 `--` 处理、附值标志、捆绑短标志、xargs 目标命令检测等。 |
| `validateFlagArgument(value, argType)` | 按 `FlagArgType` 校验标志参数合法性。 |

## 具体技术实现

### 1. 安全配置结构

```ts
export type ExternalCommandConfig = {
  safeFlags: Record<string, FlagArgType>
  additionalCommandIsDangerousCallback?: (rawCommand: string, args: string[]) => boolean
  respectsDoubleDash?: boolean // 默认 true
}
```

- `safeFlags`: 键为标志名（如 `--stat`、`-n`），值为参数类型（`'none' | 'number' | 'string' | 'char' | '{}' | 'EOF'`）。
- `additionalCommandIsDangerousCallback`: 对位置参数或特殊语法做二次校验。
- `respectsDoubleDash`: 大多数工具遵守 POSIX `--` 结束标志解析；pyright 不遵守，设为 `false` 后 `validateFlags` 不会在 `--` 处停止。

### 2. validateFlags 核心算法

遍历 token 数组，从 `startIndex` 开始：

1. **xargs 特殊处理**: 若配置了 `xargsTargetCommands`，在遇到非标志 token 或 `--` 时，检查后续 token 是否在安全目标命令列表中；是则 `break`，否则 `return false`。
2. **`--` 处理**: 若 `respectsDoubleDash !== false`，遇到 `--` 后跳过后续校验，认为之后全是位置参数。
3. **标志解析**: 
   - 支持 `--flag=value` 附值形式。
   - 支持 `-A20` 这类数字附值短标志（仅对 `grep`/`rg` 开启）。
   - 支持 `-nr` 这类捆绑短标志，但 **要求捆绑内所有标志都必须是 `'none'` 类型**，防止参数消费不一致导致的解析差分攻击（如 `xargs -rI echo sh -c id` RCE）。
4. **参数校验**: 按 `FlagArgType` 校验；对 `string` 类型额外防御以 `-` 开头的参数（git `--sort` 的反向排序例外）。
5. **git 数字简写**: `-<number>` 视为等价的 `-n <number>`。

### 3. 多次安全修复痕迹

文件注释中记录了多次真实安全漏洞的修复：

- **git diff `-S/-G/-O`**: 之前设为 `'none'`，导致 `git diff -S -- --output=/tmp/pwned` 中 `-S` 被误认为无参数，解析差分造成任意文件写入。修复为 `'string'`。
- **xargs `-i`/`-e`**: GNU getopt 的 optional-attached-arg 语义与校验器不一致，导致目标命令判断错误，产生 RCE。已移除 `-i`/`-e`。
- **xargs 捆绑标志**: `xargs -rI echo sh -c id` 利用捆绑标志的解析差分绕过。修复为禁止捆绑中出现非 `'none'` 标志。
- **`-E=` 空附值**: `hasEquals` 与 `inlineValue` 分离，防止 `-E=` 被错误地消费下一个 token 作为参数。
- **tree `-R`**: 误以为只是“max depth rerun”，实际是带硬编码 `-o 00Tree.html` 的文件写入。已移除。
- **git tag / git branch**: 通过 `additionalCommandIsDangerousCallback` 阻止不带 `--list` 的位置参数（否则会创建 tag/branch）。
- **git reflog**: 阻止 `expire`、`delete`、`exists` 子命令。
- **gh 网络外泄**: `ghIsDangerousCallback` 阻止含 `://`、含 `@`、或三段式 `HOST/OWNER/REPO` 的 repo 参数。

### 4. UNC 路径检测

```ts
export function containsVulnerableUncPath(pathOrCommand: string): boolean
```

仅在 Windows 平台生效，检测 8 种模式：

1. `\\server\share` 反斜杠 UNC
2. `//server/share` 正斜杠 UNC（排除 `://` URL）
3. `/\\server` 混合分隔符
4. `\\/server` 反向混合分隔符
5. `@SSL@\d+` / `@\d+@SSL` WebDAV SSL/端口模式
6. `DavWWWRoot` Windows WebDAV 重定向标记
7. IPv4 地址前缀 `\\1.2.3.4\`
8. IPv6 地址前缀 `\\[2001:db8::1]\`

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/readOnlyCommandValidation.ts` | 本文件 | 白名单与校验引擎。 |
| `src/tools/BashTool/readOnlyValidation.ts` | 调用方 | 导入 `GIT_READ_ONLY_COMMANDS`、`GH_READ_ONLY_COMMANDS`、`validateFlags`、`containsVulnerableUncPath` 等。 |
| `src/tools/PowerShellTool/readOnlyValidation.ts` | 调用方 | 导入共享命令配置与 `validateFlags`。 |
| `src/utils/permissions/pathValidation.ts` | 调用方 | 可能引用 `containsVulnerableUncPath` 做路径校验。 |
| `src/utils/permissions/filesystem.ts` | 调用方 | 可能引用 UNC 检测。 |
| `src/utils/platform.ts` | 被调用 | `getPlatform()` 用于 UNC 检测的平台判断。 |

## 依赖与外部交互

- **Node.js 内置**: 无（纯逻辑文件）。
- **内部模块**: `../platform.js`。
- **无外部网络/进程交互**。

## 风险、边界与改进建议

### 风险

1. **解析差分持续存在**: `validateFlags` 是对真实命令行解析器的近似模拟，任何未覆盖的 GNU getopt / cobra / PowerShell 参数绑定语义差异都可能成为新的绕过点。历史已多次证明这一点。
2. **白名单膨胀**: 随着支持的 CLI 工具增多，文件体积已达 68KB，维护成本上升；新增标志时容易误将危险标志标记为安全。
3. **callback 逻辑重复**: `git tag`、`git branch`、`git reflog` 等均有独立的 `additionalCommandIsDangerousCallback`，存在复制粘贴风险；未来新增 git 子命令时可能遗漏 callback。
4. **UNC 检测的正则绕过**: 虽然覆盖了 8 种模式，但攻击者可能使用 Unicode 同形异义字符、零宽字符或 PowerShell 的多种路径表示法绕过正则。

### 边界

- `validateFlags` 只校验标志，不校验位置参数的内容（除非 callback 介入）。
- 不支持动态加载外部白名单文件，所有配置必须在编译期确定。
- `EXTERNAL_READONLY_COMMANDS` 仅包含 `docker ps`、`docker images`，跨平台通用命令覆盖极少。
- `PYRIGHT_READ_ONLY_COMMANDS` 显式设置 `respectsDoubleDash: false`，因为 pyright 把 `--` 当文件路径。

### 改进建议

1. **引入真实 parser 替代近似模拟**: 对关键命令（git、gh、docker）使用其官方 CLI parser（如 libgit2 参数解析、cobra 的 `pflag`）或至少使用完整的 shlex + getopt 模拟器，消除解析差分。
2. **配置与代码分离**: 将 `GIT_READ_ONLY_COMMANDS`、`GH_READ_ONLY_COMMANDS` 等迁移到 JSON/YAML 配置文件中，通过构建时校验生成 TypeScript 类型，降低维护难度并支持自动化审计。
3. **统一 callback 框架**: 为 git 子命令提供通用的“位置参数模式匹配”框架（如只允许 `-l` 后的 pattern、禁止裸位置参数），减少重复代码。
4. **增加模糊测试**: 对 `validateFlags` 引入基于 property-based testing 的 fuzzing，随机生成命令字符串并与真实工具（在隔离环境中）的解析结果对比，自动发现解析差分。
5. **UNC 检测增强**: 引入规范化步骤（如 NFC 归一化、去除零宽字符）后再做正则匹配，提升防御深度。
6. **体积拆分**: 将不同工具的白名单拆分为独立子模块（如 `gitReadOnlyCommands.ts`、`ghReadOnlyCommands.ts`），改善构建缓存和代码审查体验。
