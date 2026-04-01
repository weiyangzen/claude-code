# sedValidation.ts 研究文档

## 场景与职责

`sedValidation.ts` 是 BashTool 的 sed 命令专用安全验证模块，负责验证 sed 命令是否只执行安全的只读操作或受控的替换操作。由于 sed 是一个功能强大的流编辑器，它既可以用于安全的文本查看（如 `sed -n '1,10p'`），也可以用于危险的文件操作（如 `sed -i 's/foo/bar/'` 或执行外部命令），因此需要专门的验证逻辑。

### 核心职责

1. **sed 命令白名单验证**：只允许特定的安全 sed 模式
2. **行打印模式验证**：验证 `sed -n 'Np'` 或 `sed -n 'N,Mp'` 形式的只读行打印
3. **替换模式验证**：验证 `sed 's/pattern/replacement/flags'` 形式的安全替换
4. **危险操作黑名单**：检测并阻止 write (w/W)、execute (e/E) 等危险命令
5. **约束检查**：作为跨切面验证步骤，在权限系统中阻止危险的 sed 操作

### 在系统中的位置

```
BashTool.execute()
  └── bashToolHasPermission()
        ├── checkReadOnlyConstraints()
        ├── checkPathConstraints()
        │     └── sedCommandIsAllowedByAllowlist() [本文件 - 用于覆盖操作类型]
        └── checkSedConstraints() [本文件 - 危险操作检测]
```

### 安全模型

sed 命令根据模式分为两类：

1. **Pattern 1 - 行打印（只读）**：
   ```bash
   sed -n '5p' file.txt        # 打印第 5 行
   sed -n '1,10p' file.txt     # 打印 1-10 行
   sed -n '1p;2p;3p' file.txt  # 打印多行（分号分隔）
   ```

2. **Pattern 2 - 替换（可能写入）**：
   ```bash
   sed 's/foo/bar/g' file.txt              # 输出到 stdout（只读）
   sed -i 's/foo/bar/g' file.txt           # 原地编辑（写入）
   ```

---

## 功能点目的

### 1. 行打印模式验证 (isLinePrintingCommand)

**目的**：允许安全的行打印操作，这是 sed 的只读使用模式。

**允许的 flags**：
- `-n`, `--quiet`, `--silent`（必需）
- `-E`, `--regexp-extended`, `-r`（可选，扩展正则）
- `-z`, `--zero-terminated`（可选，NUL 分隔）
- `--posix`（可选，POSIX 模式）

**允许的表达式**：
- `p` - 打印所有行
- `Np` - 打印第 N 行（N 为数字）
- `N,Mp` - 打印 N 到 M 行
- 分号分隔的多个打印命令（`1p;2p;3p`）

**安全验证**：
```typescript
// 严格的打印命令白名单
export function isPrintCommand(cmd: string): boolean {
  if (!cmd) return false
  // 只匹配: p, 1p, 123p, 1,5p, 10,200p
  return /^(?:\d+|\d+,\d+)?p$/.test(cmd)
}
```

### 2. 替换模式验证 (isSubstitutionCommand)

**目的**：允许安全的替换操作，根据上下文决定是否允许文件写入。

**允许的 flags**：
- `-E`, `--regexp-extended`, `-r`（扩展正则）
- `--posix`（POSIX 模式）
- `-i`, `--in-place`（仅在 `allowFileWrites=true` 时允许）

**表达式验证**：
- 必须以 `s` 开头（替换命令）
- 必须使用 `/` 作为分隔符
- 必须恰好有 2 个分隔符（pattern 和 replacement）
- flags 只能是 `g`, `p`, `i`, `I`, `m`, `M` 和单个数字 `1-9`

### 3. 危险操作黑名单 (containsDangerousOperations)

**目的**：深度防御，即使白名单匹配也通过黑名单进行二次检查。

**阻止的操作**：

| 操作 | 模式 | 风险 |
|------|------|------|
| write (w) | `[address]w filename` | 文件写入 |
| Write (W) | `[address]W filename` | 文件写入（首行） |
| execute (e) | `[address]e [command]` | 任意代码执行 |
| Execute (E) | `[address]E` | 多行执行 |

**保守拒绝策略**：
- 非 ASCII 字符（Unicode 同形异义字符）
- 花括号（块命令）
- 换行符（多行命令）
- 注释（`#`）
- 否定操作符（`!`）
- 波浪号步进地址（GNU 扩展）
- 反斜杠技巧（替代分隔符）

### 4. 跨切面约束检查 (checkSedConstraints)

**目的**：作为权限系统的最终防线，阻止所有危险的 sed 操作。

**返回值**：
- `'passthrough'`：无危险操作或不是 sed 命令
- `'ask'`：检测到危险操作，需要用户确认

---

## 具体技术实现

### 关键数据结构

#### 验证选项
```typescript
interface SedValidationOptions {
  allowFileWrites?: boolean  // 是否允许 -i 原地编辑
}
```

### 核心验证流程

#### 1. 主验证入口：sedCommandIsAllowedByAllowlist()

```typescript
export function sedCommandIsAllowedByAllowlist(
  command: string,
  options?: { allowFileWrites?: boolean },
): boolean
```

**执行流程**：
1. 提取 sed 表达式（`extractSedExpressions()`）
2. 检测是否有文件参数（`hasFileArgs()`）
3. 根据 `allowFileWrites` 选择验证模式：
   - `allowFileWrites=true`：只验证替换模式（Pattern 2）
   - `allowFileWrites=false`：验证行打印或替换模式
4. Pattern 2 拒绝分号分隔的命令
5. 对所有表达式执行黑名单检查（`containsDangerousOperations()`）

#### 2. 表达式提取：extractSedExpressions()

```typescript
export function extractSedExpressions(command: string): string[]
```

**处理逻辑**：
1. 移除 `sed ` 前缀
2. 检测危险 flag 组合（`-ew`, `-eW`, `-ee`, `-we`）
3. 使用 `tryParseShellCommand()` 解析参数
4. 提取 `-e` 和 `--expression=` 指定的表达式
5. 第一个非 flag 参数作为表达式（如果没有使用 `-e`）

#### 3. 文件参数检测：hasFileArgs()

```typescript
export function hasFileArgs(command: string): boolean
```

**检测逻辑**：
1. 解析命令参数
2. 跳过 `-e`, `--expression` 及其参数
3. 处理 glob 模式（如 `*.log`）作为文件参数
4. 第一个非 flag 参数是表达式（如果没有 `-e`）
5. 后续非 flag 参数是文件参数

#### 4. 危险操作检测：containsDangerousOperations()

**多层防御策略**：

**第一层 - 保守拒绝**：
```typescript
// 非 ASCII 字符（Unicode 同形异义）
if (/[^\x01-\x7F]/.test(cmd)) return true

// 花括号块
if (cmd.includes('{') || cmd.includes('}')) return true

// 换行符
if (cmd.includes('\n')) return true

// 注释（不在 s 命令后的 #）
const hashIndex = cmd.indexOf('#')
if (hashIndex !== -1 && !(hashIndex > 0 && cmd[hashIndex - 1] === 's')) return true
```

**第二层 - 地址和操作符**：
```typescript
// 否定操作符
if (/^!/.test(cmd) || /[/\d$]!/.test(cmd)) return true

// GNU 步进地址
if (/\d\s*~\s*\d|,\s*~\s*\d|\$\s*~\s*\d/.test(cmd)) return true

// 裸逗号（1,$ 的简写）
if (/^,/.test(cmd)) return true
```

**第三层 - 危险命令检测**：

Write 命令检测：
```typescript
if (
  /^[wW]\s*\S+/.test(cmd) ||                    // w file
  /^\d+\s*[wW]\s*\S+/.test(cmd) ||              // 1w file
  /^\/[^/]*\/[IMim]*\s*[wW]\s*\S+/.test(cmd) || // /pattern/w file
  /^\d+,\d+\s*[wW]\s*\S+/.test(cmd)             // 1,10w file
) {
  return true
}
```

Execute 命令检测：
```typescript
if (
  /^e/.test(cmd) ||                              // e cmd
  /^\d+\s*e/.test(cmd) ||                        // 1e
  /^\/[^/]*\/[IMim]*\s*e/.test(cmd) ||           // /pattern/e
  /^\d+,\d+\s*e/.test(cmd)                       // 1,10e
) {
  return true
}
```

替换 flags 检测：
```typescript
const substitutionMatch = cmd.match(/s([^\\\n]).*?\1.*?\1(.*?)$/)
if (substitutionMatch) {
  const flags = substitutionMatch[2] || ''
  if (flags.includes('w') || flags.includes('W')) return true  // 写入 flag
  if (flags.includes('e') || flags.includes('E')) return true  // 执行 flag
}
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `sedCommandIsAllowedByAllowlist()` | 247-301 | 主验证入口 |
| `isLinePrintingCommand()` | 44-117 | 行打印模式验证 |
| `isPrintCommand()` | 128-133 | 打印命令白名单检查 |
| `hasFileArgs()` | 307-379 | 文件参数检测 |
| `extractSedExpressions()` | 388-466 | 表达式提取 |
| `checkSedConstraints()` | 644-684 | 跨切面约束检查 |

### 内部辅助函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `isSubstitutionCommand()` | 142-238 | 替换模式验证 |
| `containsDangerousOperations()` | 473-629 | 危险操作黑名单 |
| `validateFlagsAgainstAllowlist()` | 13-35 | Flag 白名单验证 |

### 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../../Tool.js` | `ToolPermissionContext` | 权限上下文类型 |
| `../../utils/bash/commands.js` | `splitCommand_DEPRECATED` | 命令分割 |
| `../../utils/bash/shellQuote.js` | `tryParseShellCommand` | Shell 命令解析 |
| `../../utils/permissions/PermissionResult.js` | `PermissionResult` | 权限结果类型 |

---

## 依赖与外部交互

### 上游调用方

1. **pathValidation.ts**: 
   - 调用 `sedCommandIsAllowedByAllowlist()` 判断 sed 是否为只读
   - 用于覆盖 `COMMAND_OPERATION_TYPE` 中的 'write' 为 'read'

2. **readOnlyValidation.ts**:
   - 在 `COMMAND_ALLOWLIST.sed` 中使用 `sedCommandIsAllowedByAllowlist` 作为回调

3. **bashPermissions.ts**:
   - 调用 `checkSedConstraints()` 进行跨切面验证

### 下游依赖

1. **shellQuote.ts**: 提供 `tryParseShellCommand()` 用于解析 shell 命令
2. **commands.ts**: 提供 `splitCommand_DEPRECATED()` 用于分割复合命令

---

## 风险、边界与改进建议

### 已知限制

#### 1. 分隔符限制
**当前实现**：替换命令只支持 `/` 作为分隔符。

**sed 实际能力**：sed 支持任意字符作为分隔符（`s#foo#bar#`、`s|foo|bar|` 等）。

**影响**：使用非 `/` 分隔符的合法替换命令会被拒绝。

**代码位置**：
```typescript
// 第 199 行
const substitutionMatch = expr.match(/^s\/(.*?)$/)
if (!substitutionMatch) {
  return false
}
```

#### 2. 表达式数量限制
**当前实现**：
- Pattern 1（行打印）允许多个表达式（分号分隔）
- Pattern 2（替换）只接受单个表达式

```typescript
// 第 185-187 行
if (expressions.length !== 1) {
  return false
}
```

#### 3. Flag 组合限制
**当前实现**：flags 验证使用简单的字符白名单。

```typescript
const allowedFlagChars = /^[gpimIM]*[1-9]?[gpimIM]*$/
```

**限制**：不支持所有 sed 实现的各种扩展 flags。

### 安全风险

#### 1. 解析器差异风险
**风险描述**：验证器和实际 sed 对复杂表达式的解析可能存在差异。

**现有防护**：
- 保守拒绝策略（非 ASCII、花括号、换行等）
- 严格的模式白名单

**残余风险**：某些边缘情况的转义序列可能解析不一致。

#### 2. 同形异义字符攻击
**防护代码**：
```typescript
// 第 484-486 行
if (/[^\x01-\x7F]/.test(cmd)) {
  return true
}
```

**说明**：拒绝所有非 ASCII 字符，防止使用 Unicode 同形异义字符（如全角 `ｗ`、小写大写 `ᴡ`）绕过检测。

#### 3. 替代分隔符绕过
**风险**：如果攻击者找到方法使用替代分隔符绕过 `/` 的检测。

**建议**：扩展验证以支持或明确拒绝替代分隔符。

### 改进建议

#### 1. 支持替代分隔符
```typescript
// 检测并支持任意分隔符
function parseSubstitution(expr: string): { pattern: string; replacement: string; flags: string; delimiter: string } | null {
  const match = expr.match(/^s(.)/)
  if (!match) return null
  const delimiter = match[1]
  // 构建动态正则来解析三个分隔符之间的内容
  const pattern = new RegExp(`^s${delimiter}(.*?)${delimiter}(.*?)${delimiter}(.*?)$`)
  // ...
}
```

#### 2. 增强错误报告
```typescript
export type SedValidationError = 
  | { type: 'INVALID_EXPRESSION'; details: string }
  | { type: 'DANGEROUS_COMMAND'; command: string }
  | { type: 'UNSUPPORTED_FLAG'; flag: string }

export function sedCommandIsAllowedByAllowlist(
  command: string,
  options?: { allowFileWrites?: boolean },
): { allowed: true } | { allowed: false; error: SedValidationError }
```

#### 3. 更精确的地址解析
当前对地址范围的检测使用简单正则，可以改进为更精确的解析器：
```typescript
// 当前实现（简单正则）
if (/^\d+,\d+\s*[wW]\s*\S+/.test(cmd)) return true

// 改进：先解析地址，再检查命令
const address = parseAddress(cmd)
if (address && isWriteCommand(cmd.slice(address.end))) return true
```

#### 4. 性能优化
- 缓存正则表达式编译结果
- 对常见模式使用快速路径

#### 5. 测试覆盖建议
- 各种分隔符组合
- 边界情况（空 pattern、空 replacement）
- 复杂的地址范围
- Unicode 同形异义字符
- 转义序列的各种组合

### 代码质量建议

1. **魔法数字提取**：将各种正则模式提取为命名常量
2. **类型安全**：改进 `args` 数组的类型定义
3. **文档完善**：添加更多边界情况的注释说明
4. **错误日志**：在拒绝时记录具体原因用于调试

### 安全审计检查清单

新增 sed 支持时检查：
- [ ] 新模式是否只读？
- [ ] 是否可能执行任意代码？
- [ ] 是否可能写入文件？
- [ ] 是否可能泄露敏感信息？
- [ ] 是否有解析器差异风险？
- [ ] 是否测试了各种转义序列？
