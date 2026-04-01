# shellRuleMatching.ts 深入研究

## 1. 场景与职责

`shellRuleMatching.ts` 是 Claude Code 权限系统的 Shell 命令匹配引擎，负责解析和匹配权限规则与实际的 Shell 命令。它提供了统一的规则解析、通配符匹配和权限建议生成功能，被 BashTool 和 PowerShellTool 共享使用。

### 核心职责

1. **规则解析**: 将权限规则字符串解析为结构化的规则对象（精确匹配、前缀匹配、通配符匹配）
2. **通配符匹配**: 实现安全的通配符模式匹配，支持转义字符
3. **权限建议生成**: 根据命令生成权限更新建议（精确命令、前缀规则）
4. **向后兼容**: 支持传统的 `:*` 前缀语法

### 使用场景

- **Bash 权限检查**: `bashPermissions.ts` 使用这些工具进行命令匹配
- **PowerShell 权限检查**: `powershellPermissions.ts` 复用相同的匹配逻辑
- **权限建议**: 当用户批准命令时，生成合适的权限规则建议
- **规则验证**: 验证用户输入的权限规则格式是否正确

---

## 2. 功能点目的

### 2.1 规则解析

```typescript
export function parsePermissionRule(permissionRule: string): ShellPermissionRule
```

**目的**: 将权限规则字符串解析为三种类型之一的结构化对象。

**支持的规则类型**:

| 类型 | 语法示例 | 描述 |
|------|----------|------|
| `exact` | `npm install` | 精确匹配整个命令 |
| `prefix` | `npm:*` | 匹配以 `npm ` 开头的命令（传统语法） |
| `wildcard` | `npm *` | 匹配包含通配符的模式 |

**解析优先级**:
1. 检查是否以 `:*` 结尾 → `prefix` 类型
2. 检查是否包含未转义的 `*` → `wildcard` 类型
3. 默认 → `exact` 类型

### 2.2 通配符匹配

```typescript
export function matchWildcardPattern(
  pattern: string,
  command: string,
  caseInsensitive = false,
): boolean
```

**目的**: 安全地将通配符模式与命令进行匹配。

**支持的通配符语法**:
- `*`: 匹配任意字符序列（包括空序列）
- `\*`: 匹配字面量 `*`
- `\\`: 匹配字面量 `\`

**特殊处理**:
- 尾随空格 + 通配符可选化: `git *` 同时匹配 `git` 和 `git add`
- 正则特殊字符转义: 自动转义 `.+?^${}()|[\]` 等字符
- `s` (dotAll) 标志: 使 `.` 匹配换行符，支持多行命令

### 2.3 通配符检测

```typescript
export function hasWildcards(pattern: string): boolean
```

**目的**: 检测模式是否包含未转义的通配符。

**逻辑**:
- 如果以 `:*` 结尾 → 不是通配符（是传统前缀语法）
- 遍历字符串，统计每个 `*` 前的反斜杠数量
- 如果反斜杠数量为偶数（包括 0），则该 `*` 是未转义的

### 2.4 前缀提取

```typescript
export function permissionRuleExtractPrefix(permissionRule: string): string | null
```

**目的**: 从传统的 `:*` 语法中提取前缀。

**示例**:
- `npm:*` → `npm`
- `git commit:*` → `git commit`
- `npm install` → `null`（不匹配）

### 2.5 权限建议生成

```typescript
export function suggestionForExactCommand(
  toolName: string,
  command: string,
): PermissionUpdate[]

export function suggestionForPrefix(
  toolName: string,
  prefix: string,
): PermissionUpdate[]
```

**目的**: 根据命令生成权限更新建议。

**建议目标**: 默认保存到 `localSettings`（本地设置，不提交到 git）

---

## 3. 具体技术实现

### 3.1 数据结构

**Shell 权限规则**（判别联合类型）:
```typescript
export type ShellPermissionRule =
  | { type: 'exact'; command: string }
  | { type: 'prefix'; prefix: string }
  | { type: 'wildcard'; pattern: string }
```

**转义占位符**（模块级常量，避免重复编译）:
```typescript
const ESCAPED_STAR_PLACEHOLDER = '\x00ESCAPED_STAR\x00'
const ESCAPED_BACKSLASH_PLACEHOLDER = '\x00ESCAPED_BACKSLASH\x00'
const ESCAPED_STAR_PLACEHOLDER_RE = new RegExp(ESCAPED_STAR_PLACEHOLDER, 'g')
const ESCAPED_BACKSLASH_PLACEHOLDER_RE = new RegExp(ESCAPED_BACKSLASH_PLACEHOLDER, 'g')
```

### 3.2 通配符匹配算法

```typescript
export function matchWildcardPattern(
  pattern: string,
  command: string,
  caseInsensitive = false,
): boolean {
  // 1. 去除首尾空白
  const trimmedPattern = pattern.trim()

  // 2. 处理转义序列
  let processed = ''
  let i = 0
  while (i < trimmedPattern.length) {
    const char = trimmedPattern[i]
    if (char === '\\' && i + 1 < trimmedPattern.length) {
      const nextChar = trimmedPattern[i + 1]
      if (nextChar === '*') {
        processed += ESCAPED_STAR_PLACEHOLDER
        i += 2
        continue
      } else if (nextChar === '\\') {
        processed += ESCAPED_BACKSLASH_PLACEHOLDER
        i += 2
        continue
      }
    }
    processed += char
    i++
  }

  // 3. 转义正则特殊字符
  const escaped = processed.replace(/[.+?^${}()|[\]\\'"]/g, '\\$&')

  // 4. 将未转义的 * 转换为 .* 
  const withWildcards = escaped.replace(/\*/g, '.*')

  // 5. 将占位符转换回转义的字面量
  let regexPattern = withWildcards
    .replace(ESCAPED_STAR_PLACEHOLDER_RE, '\\*')
    .replace(ESCAPED_BACKSLASH_PLACEHOLDER_RE, '\\\\')

  // 6. 特殊处理：单通配符且以 ' .*' 结尾的模式
  // 使 'git *' 同时匹配 'git' 和 'git add'
  const unescapedStarCount = (processed.match(/\*/g) || []).length
  if (regexPattern.endsWith(' .*') && unescapedStarCount === 1) {
    regexPattern = regexPattern.slice(0, -3) + '( .*)?'
  }

  // 7. 创建正则并匹配
  const flags = 's' + (caseInsensitive ? 'i' : '')
  const regex = new RegExp(`^${regexPattern}$`, flags)
  return regex.test(command)
}
```

### 3.3 规则解析算法

```typescript
export function parsePermissionRule(permissionRule: string): ShellPermissionRule {
  // 1. 检查传统 :* 前缀语法
  const prefix = permissionRuleExtractPrefix(permissionRule)
  if (prefix !== null) {
    return { type: 'prefix', prefix }
  }

  // 2. 检查新通配符语法
  if (hasWildcards(permissionRule)) {
    return { type: 'wildcard', pattern: permissionRule }
  }

  // 3. 默认为精确匹配
  return { type: 'exact', command: permissionRule }
}
```

### 3.4 通配符检测算法

```typescript
export function hasWildcards(pattern: string): boolean {
  // 传统前缀语法不算通配符
  if (pattern.endsWith(':*')) {
    return false
  }

  // 检查每个 * 是否被转义
  for (let i = 0; i < pattern.length; i++) {
    if (pattern[i] === '*') {
      // 统计前面的反斜杠数量
      let backslashCount = 0
      let j = i - 1
      while (j >= 0 && pattern[j] === '\\') {
        backslashCount++
        j--
      }
      // 偶数个反斜杠 = 未转义
      if (backslashCount % 2 === 0) {
        return true
      }
    }
  }
  return false
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 导出函数和类型

| 名称 | 行号 | 类型 | 描述 |
|------|------|------|------|
| `ShellPermissionRule` | 25-38 | type | Shell 权限规则联合类型 |
| `permissionRuleExtractPrefix` | 43-48 | function | 提取传统前缀 |
| `hasWildcards` | 54-78 | function | 检测通配符 |
| `matchWildcardPattern` | 90-154 | function | 通配符匹配 |
| `parsePermissionRule` | 159-184 | function | 规则解析 |
| `suggestionForExactCommand` | 189-206 | function | 精确命令建议 |
| `suggestionForPrefix` | 211-228 | function | 前缀规则建议 |

### 4.2 依赖关系

```
src/utils/permissions/shellRuleMatching.ts
├── 导入:
│   └── src/utils/permissions/PermissionUpdateSchema.ts (PermissionUpdate)
│
├── 被导入:
│   ├── src/tools/BashTool/bashPermissions.ts (主要调用方)
│   │   ├── parsePermissionRule → bashPermissionRule
│   │   ├── matchWildcardPattern → matchWildcardPattern
│   │   ├── permissionRuleExtractPrefix → permissionRuleExtractPrefix
│   │   ├── suggestionForExactCommand → sharedSuggestionForExactCommand
│   │   └── suggestionForPrefix → sharedSuggestionForPrefix
│   └── src/tools/PowerShellTool/powershellPermissions.ts (复用)
```

---

## 5. 依赖与外部交互

### 5.1 类型依赖

| 类型 | 来源 | 用途 |
|------|------|------|
| `PermissionUpdate` | `PermissionUpdateSchema.ts` | 权限更新结构 |

### 5.2 调用方

| 模块 | 用途 |
|------|------|
| `bashPermissions.ts` | Bash 命令权限匹配和建议生成 |
| `powershellPermissions.ts` | PowerShell 命令权限匹配 |

---

## 6. 风险、边界与改进建议

### 6.1 安全风险

| 风险 | 描述 | 评估 |
|------|------|------|
| ReDoS | 通配符模式转换为正则，可能存在 ReDoS 风险 | 低风险 - 模式来自用户配置，且长度有限 |
| 转义绕过 | 复杂的转义序列可能导致意外匹配 | 低风险 - 已使用占位符方法处理 |
| 换行注入 | 多行命令可能绕过某些检查 | 已缓解 - 使用 `s` 标志使 `.` 匹配换行符 |

### 6.2 边界情况

1. **空模式**: `matchWildcardPattern('', '')` 返回 `true`（空模式匹配空字符串）
2. **全通配符**: `*` 匹配任何字符串（包括空字符串）
3. **连续通配符**: `**` 等价于 `*`（因为 `.*.*` = `.*`）
4. **特殊字符**: 某些 Unicode 字符可能在正则转义时产生意外行为
5. **极长模式**: 超长模式（>10KB）可能导致性能问题

### 6.3 已知限制

1. **不区分大小写选项**: 只在通配符匹配中支持，前缀和精确匹配不支持
2. **不支持字符类**: 不支持 `[abc]` 或 `[:digit:]` 等 POSIX 字符类
3. **不支持锚点**: 不支持 `^` 和 `$` 锚点（总是全字符串匹配）
4. **单通配符特殊处理**: 只有单通配符且以 ` *` 结尾时才应用可选化逻辑

### 6.4 改进建议

#### 6.4.1 功能增强

1. **字符类支持**: 添加 POSIX 字符类支持
   ```typescript
   // 将 [:digit:] 转换为 \d
   pattern = pattern.replace(/\[:digit:\]/g, '\\d')
   ```

2. **单词边界**: 支持 `\b` 单词边界匹配
   ```typescript
   // 使 "npm\b" 匹配 "npm install" 但不匹配 "npmrc"
   ```

3. **否定匹配**: 支持 `!` 前缀表示否定
   ```typescript
   export type ShellPermissionRule =
     | { type: 'exact'; command: string; negate?: boolean }
     | { type: 'prefix'; prefix: string; negate?: boolean }
     | { type: 'wildcard'; pattern: string; negate?: boolean }
   ```

4. **大小写敏感选项**: 为前缀和精确匹配添加大小写选项
   ```typescript
   export function matchPrefix(
     prefix: string,
     command: string,
     caseInsensitive = false,
   ): boolean
   ```

#### 6.4.2 性能优化

1. **正则缓存**: 缓存编译后的正则表达式
   ```typescript
   const regexCache = new Map<string, RegExp>()
   
   export function matchWildcardPattern(pattern: string, command: string): boolean {
     let regex = regexCache.get(pattern)
     if (!regex) {
       regex = compilePattern(pattern)
       regexCache.set(pattern, regex)
     }
     return regex.test(command)
   }
   ```

2. **提前退出**: 对于明显的非匹配情况提前返回
   ```typescript
   // 如果命令长度明显小于模式的最小长度，直接返回 false
   const minLength = pattern.replace(/\*/g, '').length
   if (command.length < minLength) return false
   ```

3. **编译时优化**: 为常用模式提供预编译版本

#### 6.4.3 安全加固

1. **模式长度限制**: 限制模式长度防止 ReDoS
   ```typescript
   const MAX_PATTERN_LENGTH = 1000
   if (pattern.length > MAX_PATTERN_LENGTH) {
     throw new Error('Pattern too long')
   }
   ```

2. **正则复杂度分析**: 检测可能导致灾难性回溯的模式
   ```typescript
   function isSafePattern(pattern: string): boolean {
     // 检测 (a+)+ 等危险模式
   }
   ```

3. **超时保护**: 为匹配操作添加超时
   ```typescript
   function matchWithTimeout(pattern: string, command: string, timeoutMs: number): boolean
   ```

#### 6.4.4 代码质量

1. **单元测试**: 添加全面的边界情况测试
   ```typescript
   describe('matchWildcardPattern', () => {
     it('should handle empty pattern', () => { ... })
     it('should handle consecutive wildcards', () => { ... })
     it('should handle special regex characters', () => { ... })
     it('should handle unicode', () => { ... })
   })
   ```

2. **模糊测试**: 使用模糊测试发现边界情况
   ```typescript
   // 使用 fast-check 等库进行属性测试
   ```

3. **文档**: 添加更多示例和边缘情况说明

4. **类型安全**: 使用 branded types 区分不同类型的字符串
   ```typescript
   type Pattern = string & { __brand: 'pattern' }
   type Command = string & { __brand: 'command' }
   ```

---

## 7. 总结

`shellRuleMatching.ts` 是 Claude Code 权限系统的 Shell 命令匹配引擎，提供了统一的规则解析、通配符匹配和权限建议生成功能。其设计考虑了安全性和向后兼容性，通过占位符方法安全地处理转义字符，并支持传统的 `:*` 前缀语法。

关键设计亮点：
- **安全转义**: 使用占位符方法避免转义字符在正则转换中的问题
- **向后兼容**: 支持传统的 `:*` 前缀语法
- **智能匹配**: 单通配符模式支持可选尾随参数（`git *` 匹配 `git` 和 `git add`）
- **多行支持**: 使用 `s` 标志支持多行命令匹配

主要风险点：
- 正则转换可能引入 ReDoS 风险（虽然风险较低）
- 某些复杂的转义序列可能产生意外行为
- 极长模式可能导致性能问题

该模块虽然代码量不大，但在权限系统的核心匹配逻辑中起着关键作用。其稳定性和正确性直接影响权限规则的匹配结果，进而影响用户体验和系统安全。
