# readOnlyValidation.ts 研究文档

## 场景与职责

`readOnlyValidation.ts` 是 BashTool 的核心安全验证模块，负责判断一个 Bash 命令是否为**只读操作**。该模块是 Claude Code CLI 安全架构的关键组成部分，确保 AI 助手不会未经用户许可就执行文件写入、代码执行或网络请求等危险操作。

### 核心职责

1. **只读命令验证**：判断命令是否只读取文件/数据而不修改系统状态
2. **复合命令处理**：处理包含 `&&`、`||`、`|` 等操作符的复合命令
3. **Git 安全沙箱**：防止通过 Git 命令执行恶意钩子代码的逃逸攻击
4. **命令白名单管理**：维护一个详尽的只读命令及其安全参数的白名单
5. **路径验证集成**：与 `pathValidation.ts` 协作验证文件路径安全性

### 在系统中的位置

```
BashTool.execute()
  └── bashToolHasPermission() [bashPermissions.ts]
        ├── checkReadOnlyConstraints() [本文件]
        ├── checkPathConstraints() [pathValidation.ts]
        └── checkSedConstraints() [sedValidation.ts]
```

---

## 功能点目的

### 1. 统一命令验证配置系统 (COMMAND_ALLOWLIST)

**目的**：通过声明式配置替代硬编码的验证逻辑，提高安全性和可维护性。

**设计原则**：
- 每个命令定义其安全的 flags 及参数类型
- 支持自定义回调函数进行额外验证
- 支持 POSIX `--` 结束选项标记的处理

**参数类型系统**：
```typescript
type FlagArgType = 
  | 'none'      // 无参数 (--color, -n)
  | 'number'    // 整数参数 (--context=3)
  | 'string'    // 字符串参数 (--relative=path)
  | 'char'      // 单字符 (分隔符)
  | '{}'        // 仅字面量 "{}"
  | 'EOF'       // 仅字面量 "EOF"
```

### 2. 复合命令安全验证

**目的**：防止通过复合命令绕过安全检查。

**防御场景**：
- `cd /malicious/dir && git status` - 防止切换到包含恶意 Git 钩子的目录
- `mkdir -p hooks && echo 'evil' > hooks/pre-commit && git status` - 防止创建 Git 内部文件后执行 Git
- `cat file | xargs cat` - 防止通过 xargs 执行 UNC 路径导致的网络请求

### 3. Git 沙箱逃逸防护

**目的**：防止攻击者通过操纵 Git 内部文件执行任意代码。

**防护机制**：
- 检测当前目录是否为裸 Git 仓库（bare repo）
- 阻止同时包含 `cd` 和 `git` 的复合命令
- 检测命令是否写入 Git 内部路径（HEAD、objects/、refs/、hooks/）

### 4. 变量扩展攻击防护

**目的**：防止通过 `$VAR` 变量扩展绕过 flag 验证。

**攻击示例**：
```bash
# 攻击：$Z 在验证时看起来不是以 - 开头，但运行时展开为 --output=/tmp/pwned
git diff "$Z--output=/tmp/pwned"
```

**防护措施**：在 `isCommandSafeViaFlagParsing()` 中检查所有 token 是否包含 `$` 字符。

### 5. Windows UNC 路径防护

**目的**：防止通过 UNC 路径（如 `\\server\share`）触发 NTLM/Kerberos 凭证泄露或 WebDAV 攻击。

---

## 具体技术实现

### 关键数据结构

#### CommandConfig 配置结构
```typescript
type CommandConfig = {
  safeFlags: Record<string, FlagArgType>  // 安全 flags 映射
  regex?: RegExp                           // 额外正则验证
  additionalCommandIsDangerousCallback?: ( // 自定义危险检测回调
    rawCommand: string,
    args: string[],
  ) => boolean
  respectsDoubleDash?: boolean            // 是否尊重 POSIX -- (默认 true)
}
```

#### 命令白名单示例 (fd 命令)
```typescript
const FD_SAFE_FLAGS: Record<string, FlagArgType> = {
  '-H': 'none',
  '--hidden': 'none',
  '-d': 'number',
  '--max-depth': 'number',
  '-t': 'string',
  '--type': 'string',
  // SECURITY: -x/--exec 和 -X/--exec-batch 被故意排除
  // 它们会为每个搜索结果执行任意命令
}
```

### 关键流程

#### 1. 主验证入口：checkReadOnlyConstraints()

```typescript
export function checkReadOnlyConstraints(
  input: z.infer<typeof BashTool.inputSchema>,
  compoundCommandHasCd: boolean,
): PermissionResult
```

**执行流程**：
1. 使用 `tryParseShellCommand()` 解析命令
2. 调用 `bashCommandIsSafe_DEPRECATED()` 进行基础安全检查
3. 检测 Windows UNC 路径
4. **复合命令安全检测**：
   - 检查是否同时包含 `cd` 和 `git`
   - 检查是否为裸 Git 仓库
   - 检查是否写入 Git 内部路径
5. 检查 Git 命令是否在沙箱外的目录执行
6. 使用 `splitCommand_DEPRECATED()` 分割复合命令
7. 对每个子命令调用 `isCommandReadOnly()`

#### 2. 命令只读验证：isCommandReadOnly()

```typescript
function isCommandReadOnly(command: string): boolean
```

**执行流程**：
1. 处理 `2>&1` 标准错误重定向
2. 检测 UNC 路径
3. 检测未引用的 glob 字符和变量扩展
4. 调用 `isCommandSafeViaFlagParsing()` 进行 flag 验证
5. 回退到 `READONLY_COMMAND_REGEXES` 正则匹配

#### 3. Flag 解析验证：isCommandSafeViaFlagParsing()

```typescript
export function isCommandSafeViaFlagParsing(command: string): boolean
```

**核心安全逻辑**：
1. **变量扩展防护**：检查所有 token 是否包含 `$` 字符
2. **Brace Expansion 防护**：检查 `{` 和 `,` 或 `..` 的组合
3. **Token 解析**：使用 `tryParseShellCommand()` 解析命令
4. **多词命令匹配**：支持 "git diff"、"git stash list" 等
5. **Flag 验证**：调用 `validateFlags()` 验证每个 flag
6. **回调验证**：执行 `additionalCommandIsDangerousCallback`

#### 4. Git ls-remote 特殊防护

```typescript
// 在 isCommandSafeViaFlagParsing() 中
if (tokens[0] === 'git' && tokens[1] === 'ls-remote') {
  for (let i = 2; i < tokens.length; i++) {
    const token = tokens[i]
    if (token && !token.startsWith('-')) {
      // 拒绝 HTTP/HTTPS URL
      if (token.includes('://')) return false
      // 拒绝 SSH URL
      if (token.includes('@') || token.includes(':')) return false
    }
  }
}
```

### 安全关键代码路径

#### xargs 特殊处理
```typescript
// 当找到安全的目标命令时停止验证 flags
if (
  options?.xargsTargetCommands &&
  options.commandName === 'xargs' &&
  (!token.startsWith('-') || token === '--')
) {
  if (token === '--' && i + 1 < tokens.length) {
    i++
    token = tokens[i]
  }
  if (token && options.xargsTargetCommands.includes(token)) {
    break  // 停止验证，目标命令本身必须无危险 flags
  }
  return false
}
```

#### 组合 Flag 安全验证
```typescript
// 处理组合 flags 如 -nE 或 -Er
if (flag.startsWith('-') && !flag.startsWith('--') && flag.length > 2) {
  for (let j = 1; j < flag.length; j++) {
    const singleFlag = '-' + flag[j]
    const flagType = config.safeFlags[singleFlag]
    if (!flagType) return false
    // SECURITY: 组合 flags 中的所有 flag 必须是 'none' 类型
    // 防止参数消耗导致的解析差异
    if (flagType !== 'none') return false
  }
}
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `checkReadOnlyConstraints()` | 1876-1990 | 主验证入口 |
| `isCommandSafeViaFlagParsing()` | 1246-1408 | Flag 解析验证 |
| `containsUnquotedExpansion()` | 1600-1669 | 未引用扩展检测 |

### 命令白名单

| 常量 | 行号 | 说明 |
|------|------|------|
| `COMMAND_ALLOWLIST` | 128-1137 | 主要只读命令白名单 |
| `ANT_ONLY_COMMAND_ALLOWLIST` | 1141-1199 | Ant 用户专用命令（含网络访问） |
| `READONLY_COMMANDS` | 1432-1503 | 简单只读命令列表 |
| `READONLY_COMMAND_REGEXES` | 1509-1570 | 正则表达式验证模式 |
| `SAFE_TARGET_COMMANDS_FOR_XARGS` | 1232-1239 | xargs 安全目标命令 |

### 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../../utils/shell/readOnlyCommandValidation.js` | `GIT_READ_ONLY_COMMANDS`, `validateFlags` | 共享 Git 命令配置和 flag 验证 |
| `./bashSecurity.js` | `bashCommandIsSafe_DEPRECATED` | 基础安全验证 |
| `./pathValidation.js` | `COMMAND_OPERATION_TYPE`, `PATH_EXTRACTORS` | 路径提取和操作类型 |
| `./sedValidation.js` | `sedCommandIsAllowedByAllowlist` | Sed 命令验证 |
| `../../utils/bash/shellQuote.js` | `tryParseShellCommand` | Shell 命令解析 |
| `../../utils/bash/commands.js` | `splitCommand_DEPRECATED`, `extractOutputRedirections` | 命令分割和重定向提取 |

---

## 依赖与外部交互

### 上游调用方

1. **bashPermissions.ts**: `checkReadOnlyConstraints()` 是主要的调用入口
2. **speculation.ts**: 用于推测性验证，提前判断命令是否只读

### 下游依赖

1. **readOnlyCommandValidation.ts**: 共享命令配置和 flag 验证逻辑
2. **bashSecurity.ts**: 基础安全验证（命令注入、重定向等）
3. **pathValidation.ts**: 路径验证和 Git 沙箱检测
4. **sedValidation.ts**: Sed 命令的特殊验证

### 外部工具依赖

- **shell-quote**: 用于解析 shell 命令的库
- **tree-sitter**: 用于 AST 级别的命令分析

---

## 风险、边界与改进建议

### 已知风险

#### 1. Parser Differential 风险
**风险描述**：验证器和实际 shell 对命令的解析可能存在差异，导致验证通过但执行危险。

**现有防护**：
- 变量扩展检测（`$` 字符检查）
- Brace expansion 检测（`{` + `,` 或 `..`）
- 引号状态跟踪

**残余风险**：复杂的引号转义序列可能仍存在解析差异。

#### 2. xargs 参数消耗攻击
**风险描述**：GNU getopt 的组合 flag 语义与验证器可能不一致。

**防护措施**：
- 组合 flags 中的所有 flag 必须是 'none' 类型
- 使用 `SAFE_TARGET_COMMANDS_FOR_XARGS` 限制目标命令

#### 3. Git 沙箱逃逸
**风险描述**：通过创建 Git 内部文件（hooks/pre-commit）执行任意代码。

**防护措施**：
- 检测裸 Git 仓库
- 阻止 cd + git 复合命令
- 检测写入 Git 内部路径

### 边界限制

1. **命令复杂度限制**：`MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50`，超过此限制的复合命令需要手动批准
2. **只读定义**：某些命令（如 `sed -i`）在技术上会修改文件，但通过 `sedValidation.ts` 的特殊处理允许
3. **平台差异**：Windows 上 UNC 路径检查更严格，且 xargs 被完全禁用

### 改进建议

#### 1. 增强 Parser Differential 防护
```typescript
// 建议：增加对 ANSI-C 引用的检测 ($'...')
// 当前代码注释中提到这是潜在风险
if (/\$'[^']*'/.test(command)) {
  // ANSI-C quoting 可以编码任意字符
  return { behavior: 'ask', ... }
}
```

#### 2. 改进 Git 沙箱检测
- 考虑使用文件系统监控来检测 Git 内部文件的创建
- 增加对 `.git` 目录权限的检查

#### 3. 优化性能
- `splitCommand_DEPRECATED()` 在复杂命令上可能有指数级增长
- 考虑缓存解析结果

#### 4. 增强可观测性
- 增加更多安全事件日志
- 提供用户可见的安全检查说明

#### 5. 代码重构建议
- `isCommandReadOnly()` 和 `isCommandSafeViaFlagParsing()` 的边界可以更清晰
- 考虑将正则表达式验证迁移到声明式配置

### 安全审计要点

1. **新增命令到白名单时**：
   - 检查所有 flags 是否只读
   - 验证 flags 的参数类型
   - 测试组合 flags 的行为
   - 检查 `--` 处理逻辑

2. **修改 flag 验证逻辑时**：
   - 测试 GNU getopt 的边界情况
   - 验证变量扩展防护
   - 测试 brace expansion 防护

3. **Git 相关修改时**：
   - 测试裸仓库检测
   - 测试复合命令检测
   - 验证 Git 内部路径检测
