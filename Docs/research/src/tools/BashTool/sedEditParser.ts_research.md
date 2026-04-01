# sedEditParser.ts 研究文档

## 场景与职责

`sedEditParser.ts` 是 BashTool 的 sed 原地编辑命令解析器，专门用于解析和模拟 `sed -i`（in-place edit）命令的行为。该模块的核心目标是将 sed 的原地编辑操作转换为类似文件编辑器的渲染格式，使用户能够直观地看到文件修改内容。

### 核心职责

1. **sed 原地编辑命令识别**：判断一个命令是否为 `sed -i` 原地编辑操作
2. **编辑信息提取**：从 sed 命令中提取文件路径、搜索模式、替换字符串和 flags
3. **替换模拟**：在 JavaScript 中模拟 sed 的替换行为，生成修改后的文件内容
4. **BRE/ERE 转换**：将 sed 的基本正则表达式（BRE）转换为 JavaScript 的扩展正则表达式（ERE）

### 在系统中的位置

```
BashTool.execute()
  └── 执行 sed 命令
        └── 渲染层检测 sed 编辑
              └── sedEditParser.ts 解析并生成文件编辑视图
```

### 使用场景

当用户执行类似以下命令时：
```bash
sed -i 's/oldPattern/newText/g' file.txt
```

系统会：
1. 使用 `isSedInPlaceEdit()` 检测是否为原地编辑
2. 使用 `parseSedEditCommand()` 提取编辑信息
3. 读取原文件内容
4. 使用 `applySedSubstitution()` 模拟替换
5. 以文件编辑器的形式展示修改

---

## 功能点目的

### 1. 原地编辑命令识别 (isSedInPlaceEdit)

**目的**：快速判断一个命令是否为可解析的 sed 原地编辑操作。

**识别条件**：
- 命令以 `sed` 开头
- 包含 `-i` 或 `--in-place` 标志
- 包含有效的替换表达式（`s/pattern/replacement/flags`）
- 指定了目标文件路径

### 2. 编辑信息提取 (parseSedEditCommand)

**目的**：从 sed 命令中提取所有必要的编辑信息。

**提取字段**：
- `filePath`: 被编辑的文件路径
- `pattern`: 搜索正则表达式
- `replacement`: 替换字符串
- `flags`: 替换 flags（g, i, m, p 等）
- `extendedRegex`: 是否使用扩展正则（`-E` 或 `-r`）

**支持的 sed 语法**：
```bash
# 标准形式
sed -i 's/foo/bar/g' file.txt

# 使用 -e 标志
sed -i -e 's/foo/bar/g' file.txt

# 使用 --expression
sed -i --expression='s/foo/bar/g' file.txt

# macOS 备份后缀
sed -i '' 's/foo/bar/g' file.txt
sed -i.bak 's/foo/bar/g' file.txt

# 扩展正则
sed -i -E 's/[0-9]+/NUM/g' file.txt
```

### 3. 替换模拟 (applySedSubstitution)

**目的**：在 JavaScript 中准确模拟 sed 的替换行为。

**技术挑战**：
- sed 的 BRE 和 JavaScript 的 ERE 元字符转义规则相反
- sed 的特殊替换字符（`&`、 `\n`、 `\1` 等）需要转换
- 需要正确处理 flags（全局替换 `g`、忽略大小写 `i`、多行 `m`）

---

## 具体技术实现

### 关键数据结构

#### SedEditInfo 类型
```typescript
export type SedEditInfo = {
  filePath: string      // 被编辑的文件路径
  pattern: string       // 搜索正则表达式
  replacement: string   // 替换字符串
  flags: string         // 替换 flags（g, i, m, p, 1-9）
  extendedRegex: boolean // 是否使用扩展正则
}
```

### BRE 到 ERE 转换机制

sed 的 BRE（Basic Regular Expression）和 JavaScript 的 ERE（Extended Regular Expression）在元字符转义上有相反的规则：

| 含义 | BRE (sed) | ERE (JavaScript) |
|------|-----------|------------------|
| 一个或多个 | `\+` | `+` |
| 零个或一个 | `\?` | `?` |
| 或 | `\|` | `\|` |
| 分组 | `\(` `\)` | `(` `)` |

**转换算法**（`applySedSubstitution` 函数中的 BRE 处理）：

```typescript
if (!sedInfo.extendedRegex) {
  jsPattern = sedInfo.pattern
    // 步骤 1: 保护字面量反斜杠
    .replace(/\\\\/g, BACKSLASH_PLACEHOLDER)
    // 步骤 2: 将转义的元字符替换为占位符
    .replace(/\\\+/g, PLUS_PLACEHOLDER)
    .replace(/\\\?/g, QUESTION_PLACEHOLDER)
    .replace(/\\\|/g, PIPE_PLACEHOLDER)
    .replace(/\\\(/g, LPAREN_PLACEHOLDER)
    .replace(/\\\)/g, RPAREN_PLACEHOLDER)
    // 步骤 3: 转义未转义的元字符（BRE 中是字面量）
    .replace(/\+/g, '\\+')
    .replace(/\?/g, '\\?')
    .replace(/\|/g, '\\|')
    .replace(/\(/g, '\\(')
    .replace(/\)/g, '\\)')
    // 步骤 4: 恢复占位符为 JS 元字符
    .replace(BACKSLASH_PLACEHOLDER_RE, '\\\\')
    .replace(PLUS_PLACEHOLDER_RE, '+')
    .replace(QUESTION_PLACEHOLDER_RE, '?')
    .replace(PIPE_PLACEHOLDER_RE, '|')
    .replace(LPAREN_PLACEHOLDER_RE, '(')
    .replace(RPAREN_PLACEHOLDER_RE, ')')
}
```

### 替换字符串处理

**sed 特殊字符转换**：
```typescript
const jsReplacement = sedInfo.replacement
  // 解转义 \/ 为 /
  .replace(/\\\//g, '/')
  // 保护转义的 &
  .replace(/\\&/g, ESCAPED_AMP_PLACEHOLDER)
  // 将 & 转换为 $&（JavaScript 中的完整匹配）
  .replace(/&/g, '$$&')
  // 恢复字面量 &
  .replace(new RegExp(ESCAPED_AMP_PLACEHOLDER, 'g'), '&')
```

### 命令解析流程

#### parseSedEditCommand 执行流程

1. **命令前缀检查**：
   ```typescript
   const sedMatch = trimmed.match(/^\s*sed\s+/)
   if (!sedMatch) return null
   ```

2. **Shell 命令解析**：
   ```typescript
   const parseResult = tryParseShellCommand(withoutSed)
   if (!parseResult.success) return null
   ```

3. **参数遍历**：
   - 检测 `-i` 或 `--in-place` 标志
   - 处理 macOS 备份后缀（`-i ''` 或 `-i.bak`）
   - 检测 `-E`、`-r`、 `--regexp-extended` 扩展正则标志
   - 提取 `-e` 或 `--expression` 表达式

4. **替换表达式解析**：
   - 仅支持 `/` 作为分隔符
   - 状态机解析：pattern → replacement → flags
   - 处理转义字符（`\/`）

5. **Flags 验证**：
   ```typescript
   const validFlags = /^[gpimIM1-9]*$/
   if (!validFlags.test(flags)) return null
   ```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `isSedInPlaceEdit()` | 40-43 | 检测是否为 sed 原地编辑命令 |
| `parseSedEditCommand()` | 49-238 | 解析 sed 编辑命令 |
| `applySedSubstitution()` | 244-322 | 应用 sed 替换到内容 |

### 占位符常量

| 常量 | 行号 | 用途 |
|------|------|------|
| `BACKSLASH_PLACEHOLDER` | 10 | 保护字面量反斜杠 |
| `PLUS_PLACEHOLDER` | 11 | BRE `\+` 占位符 |
| `QUESTION_PLACEHOLDER` | 12 | BRE `\?` 占位符 |
| `PIPE_PLACEHOLDER` | 13 | BRE `\|` 占位符 |
| `LPAREN_PLACEHOLDER` | 14 | BRE `\(` 占位符 |
| `RPAREN_PLACEHOLDER` | 15 | BRE `\)` 占位符 |

### 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `crypto` | `randomBytes` | 生成替换占位符的 salt |
| `../../utils/bash/shellQuote.js` | `tryParseShellCommand` | Shell 命令解析 |

---

## 依赖与外部交互

### 上游调用方

该模块通常被渲染层调用，用于将 sed 原地编辑命令转换为文件编辑视图。

### 下游依赖

1. **shellQuote.ts**: 提供 `tryParseShellCommand()` 用于解析 shell 命令
2. **文件系统**: 需要读取原文件内容以应用替换

---

## 风险、边界与改进建议

### 已知限制

#### 1. 分隔符限制
**当前实现**：仅支持 `/` 作为替换分隔符。

**sed 支持的替代分隔符**：sed 允许使用任意字符作为分隔符（如 `s#foo#bar#g`、`s|foo|bar|g`），但本模块为了简化只支持 `/`。

**影响**：使用非 `/` 分隔符的 sed 命令无法被解析，将回退到标准命令输出显示。

#### 2. 多文件编辑
**当前实现**：仅支持单个文件的编辑。

```typescript
// 代码中明确拒绝多文件
if (filePath !== null) {
  // More than one file - not supported for simple rendering
  return null
}
```

#### 3. 复杂表达式限制
**当前实现**：仅支持简单的替换表达式（`s/pattern/replacement/flags`）。

**不支持的 sed 特性**：
- 多命令表达式（`sed -e 'cmd1' -e 'cmd2'`）
- 地址范围（`sed '1,10s/foo/bar/'`）
- 分支和标签
- 保持空间操作（h, H, g, G, x）
- 读取和写入文件（r, w）
- 执行外部命令（e）

#### 4. Glob 模式拒绝
```typescript
if (typeof token === 'object' && token !== null && 'op' in token && token.op === 'glob') {
  // Glob patterns are too complex for this simple parser
  return null
}
```

### 安全风险

#### 1. 正则表达式注入
**风险**：sed 的 pattern 可能包含恶意构造的正则表达式，导致 ReDoS（正则表达式拒绝服务）。

**当前防护**：
- 使用 `new RegExp()` 构造时捕获异常
- 限制 flags 为安全的字符集

**建议改进**：
```typescript
// 添加正则表达式复杂度检查
function checkRegexComplexity(pattern: string): boolean {
  // 检查嵌套量词、过度重复等
  const nestedQuantifiers = /[+*?]\{.*\}|[+*?][+*?]/
  return !nestedQuantifiers.test(pattern)
}
```

#### 2. 替换字符串注入
**风险**：`replacement` 中的 `$` 序列可能被恶意利用。

**当前防护**：
- 使用随机 salt 的占位符防止注入
- 正确处理 `$$`（JavaScript 中的字面量 `$`）

### 改进建议

#### 1. 支持更多分隔符
```typescript
// 支持任意分隔符
const substMatch = expression.match(/^s([^\\\n])/)
if (!substMatch) return null
const delimiter = substMatch[1]
const pattern = new RegExp(`^s${delimiter}(.*?)${delimiter}(.*?)${delimiter}(.*?)$`)
```

#### 2. 支持多个 `-e` 表达式
```typescript
// 收集所有表达式
const expressions: string[] = []
// ... 在参数遍历中收集所有 -e 表达式
// 按顺序应用每个替换
```

#### 3. 增强错误处理
```typescript
// 提供更详细的错误信息
export type ParseError = 
  | { type: 'NOT_SED_COMMAND' }
  | { type: 'NO_IN_PLACE_FLAG' }
  | { type: 'INVALID_EXPRESSION', reason: string }
  | { type: 'UNSUPPORTED_FEATURE', feature: string }

export function parseSedEditCommand(command: string): 
  | { success: true; data: SedEditInfo }
  | { success: false; error: ParseError }
```

#### 4. 性能优化
- 缓存编译后的正则表达式
- 对大文件使用流式处理

#### 5. 测试覆盖建议
- BRE 和 ERE 的各种组合
- 特殊字符的转义序列
- 边界情况（空 pattern、空 replacement）
- 各种 flags 组合

### 代码质量建议

1. **类型安全**：`args` 数组的类型可以更加精确
2. **常量提取**：flags 验证正则可以提取为命名常量
3. **文档注释**：添加更多 JSDoc 注释说明边界情况
