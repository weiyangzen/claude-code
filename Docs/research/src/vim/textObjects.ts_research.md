# src/vim/textObjects.ts 研究文档

## 场景与职责

本文件是 Claude Code 的 Vim 模式实现中的**文本对象（Text Object）查找模块**。它负责解析和查找 Vim 文本对象的范围，支持 `iw`/`aw`（单词）、`i"`/`a"`（引号）、`i(`/`a(`（括号）等常用文本对象。

**核心职责：**
- 查找单词对象（word/WORD）的范围
- 查找引号对象（quote）的范围
- 查找括号对象（bracket/paren）的范围
- 支持 inner（内部）和 around（包含边界）两种范围模式

**在 Vim 子系统中的位置：**
```
useVimInput.ts (Hook)
  → transitions.ts (状态机)
    → operators.ts (操作执行)
      → textObjects.ts (文本对象查找) ← 本文档
        → Cursor.ts (字符分类工具)
        → intl.ts (Unicode grapheme 处理)
```

## 功能点目的

### 1. 文本对象类型支持

**支持的文本对象：**

| 类型 | 说明 | 示例 |
|------|------|------|
| `w` | 小写 word（字母数字下划线序列） | `iw`, `aw` |
| `W` | 大写 WORD（非空白字符序列） | `iW`, `aW` |
| `"` | 双引号字符串 | `i"`, `a"` |
| `'` | 单引号字符串 | `i'`, `a'` |
| `` ` `` | 反引号字符串 | `` i` ``, `` a` `` |
| `(`, `)`, `b` | 圆括号 | `i(`, `a(`, `ib`, `ab` |
| `[`, `]` | 方括号 | `i[`, `a[` |
| `{`, `}`, `B` | 花括号 | `i{`, `a{`, `iB`, `aB` |
| `<`, `>` | 尖括号 | `i<`, `a<` |

### 2. 范围模式

**Inner 模式（`i` 前缀）：**
- 只包含对象内部的内容
- 例如：`i"` 选中引号内的文本，不包含引号本身
- `i(` 选中括号内的内容，不包含括号

**Around 模式（`a` 前缀）：**
- 包含对象及其边界标记
- 例如：`a"` 选中引号及其内的文本
- `a(` 选中括号及其内的内容，包含括号

### 3. 单词对象的特殊处理

**字符分类（Vim 语义）：**
- **Word 字符**：字母、数字、下划线（Unicode 支持）
- **空白字符**：空格、制表符等
- **标点字符**：其他所有非空白字符

**单词边界规则：**
1. 如果光标在 word 字符上，扩展到该 word 的边界
2. 如果光标在空白字符上，扩展到该空白区域的边界
3. 如果光标在标点字符上，扩展到该标点序列的边界

**Around 模式的空白处理：**
- 优先扩展到尾随的空白
- 如果没有尾随空白，则扩展到前导空白

## 具体技术实现

### 关键流程

#### 主入口函数 `findTextObject`
```
findTextObject(text, offset, objectType, isInner)
  ├── 如果是 'w' → findWordObject(text, offset, isInner, isVimWordChar)
  ├── 如果是 'W' → findWordObject(text, offset, isInner, isNonWhitespace)
  └── 如果是括号/引号类型
       ├── 如果是引号（开闭相同）→ findQuoteObject
       └── 否则 → findBracketObject
```

#### 单词对象查找流程
```
findWordObject(text, offset, isInner, isWordChar)
  ├── 将文本分割为 graphemes（Unicode 安全）
  ├── 确定光标所在的 grapheme 索引
  ├── 分类当前字符（word/空白/标点）
  ├── 向前/向后扩展到边界
  └── 如果是 around 模式，扩展到相邻空白
```

#### 引号对象查找流程
```
findQuoteObject(text, offset, quote, isInner)
  ├── 限制在当前行范围内
  ├── 找到行内所有引号位置
  ├── 配对引号（0-1, 2-3, 4-5...）
  └── 返回包含光标的配对范围
```

#### 括号对象查找流程
```
findBracketObject(text, offset, open, close, isInner)
  ├── 从光标位置向后查找匹配的 open 括号（考虑嵌套）
  ├── 从 open 括号后向前查找匹配的 close 括号（考虑嵌套）
  └── 返回范围（inner 排除括号本身）
```

### 数据结构

#### 文本对象范围
```typescript
export type TextObjectRange = { start: number; end: number } | null
```

#### 括号配对映射（行19-33）
```typescript
const PAIRS: Record<string, [string, string]> = {
  '(': ['(', ')'],
  ')': ['(', ')'],
  b: ['(', ')'],
  '[': ['[', ']'],
  ']': ['[', ']'],
  '{': ['{', '}'],
  '}': ['{', '}'],
  B: ['{', '}'],
  '<': ['<', '>'],
  '>': ['<', '>'],
  '"': ['"', '"'],
  "'": ["'", "'"],
  '`': ['`', '`'],
}
```

### 关键代码路径

**文件位置：** `/home/sansha/Github/claude-code-instructkr/src/vim/textObjects.ts`

**核心函数位置：**

| 函数 | 行号 | 说明 |
|------|------|------|
| `findTextObject` | 38-58 | 主入口函数 |
| `findWordObject` | 60-116 | 单词对象查找 |
| `findQuoteObject` | 118-147 | 引号对象查找 |
| `findBracketObject` | 149-186 | 括号对象查找 |

**关键代码片段：**

**Grapheme 安全处理（行66-82）：**
```typescript
// Pre-segment into graphemes for grapheme-safe iteration
const graphemes: Array<{ segment: string; index: number }> = []
for (const { segment, index } of getGraphemeSegmenter().segment(text)) {
  graphemes.push({ segment, index })
}

// Find which grapheme index the offset falls in
let graphemeIdx = graphemes.length - 1
for (let i = 0; i < graphemes.length; i++) {
  const g = graphemes[i]!
  const nextStart = i + 1 < graphemes.length ? graphemes[i + 1]!.index : text.length
  if (offset >= g.index && offset < nextStart) {
    graphemeIdx = i
    break
  }
}
```

**引号配对逻辑（行136-144）：**
```typescript
// Pair quotes correctly: 0-1, 2-3, 4-5, etc.
for (let i = 0; i < positions.length - 1; i += 2) {
  const qs = positions[i]!
  const qe = positions[i + 1]!
  if (qs <= posInLine && posInLine <= qe) {
    return isInner
      ? { start: lineStart + qs + 1, end: lineStart + qe }
      : { start: lineStart + qs, end: lineStart + qe + 1 }
  }
}
```

**括号嵌套处理（行156-182）：**
```typescript
// 查找 open 括号（向后搜索，考虑嵌套）
let depth = 0
let start = -1
for (let i = offset; i >= 0; i--) {
  if (text[i] === close && i !== offset) depth++
  else if (text[i] === open) {
    if (depth === 0) { start = i; break }
    depth--
  }
}

// 查找 close 括号（向前搜索，考虑嵌套）
depth = 0
let end = -1
for (let i = start + 1; i < text.length; i++) {
  if (text[i] === open) depth++
  else if (text[i] === close) {
    if (depth === 0) { end = i; break }
    depth--
  }
}
```

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `isVimPunctuation`, `isVimWhitespace`, `isVimWordChar` | `../utils/Cursor.js` | Vim 字符分类函数 |
| `getGraphemeSegmenter` | `../utils/intl.js` | Unicode grapheme 分割器 |

### 被调用方

| 模块 | 路径 | 调用方式 |
|------|------|----------|
| `operators.ts` | `./operators.js` | `import { findTextObject }` |

### 字符分类函数

**来自 `Cursor.ts`：**

```typescript
// Pre-compiled regex patterns for Vim word detection
export const VIM_WORD_CHAR_REGEX = /^[\p{L}\p{N}\p{M}_]$/u
export const WHITESPACE_REGEX = /\s/

export const isVimWordChar = (ch: string): boolean => VIM_WORD_CHAR_REGEX.test(ch)
export const isVimWhitespace = (ch: string): boolean => WHITESPACE_REGEX.test(ch)
export const isVimPunctuation = (ch: string): boolean =>
  ch.length > 0 && !isVimWhitespace(ch) && !isVimWordChar(ch)
```

**Unicode 属性说明：**
- `\p{L}` - 任意语言的字母
- `\p{N}` - 数字
- `\p{M}` - 标记字符（如重音符号）
- `u` 标志 - 启用 Unicode 属性转义

## 风险、边界与改进建议

### 已知边界情况

1. **引号配对的简化假设（行136）**
   ```typescript
   // Pair quotes correctly: 0-1, 2-3, 4-5, etc.
   ```
   - 假设引号总是成对出现（0-1, 2-3, ...）
   - 不处理转义引号（如 `"He said \"Hello\""`）
   - 不处理嵌套引号类型不同的情况

2. **光标在空白区域的单词对象（行97-100）**
   ```typescript
   } else if (isWs(graphemeIdx)) {
     // ...
     return { start: offsetAt(startIdx), end: offsetAt(endIdx) }
   }
   ```
   - 如果光标在空白上，直接返回空白范围
   - 忽略 `isInner` 参数（空白对象没有 inner/around 区别）

3. **括号查找的边界条件**
   - 如果光标在括号内部但前面没有匹配的 open 括号，返回 `null`
   - 如果找到 open 但没有找到 close，返回 `null`

4. **Unicode 处理性能**
   - 每次调用 `findWordObject` 都重新分割 graphemes
   - 对于大文本，这可能影响性能

### 潜在风险

1. **引号查找的行限制（行124-127）**
   ```typescript
   const lineStart = text.lastIndexOf('\n', offset - 1) + 1
   const lineEnd = text.indexOf('\n', offset)
   const effectiveEnd = lineEnd === -1 ? text.length : lineEnd
   ```
   - 引号对象只在当前行内查找
   - 跨行字符串（如某些编程语言支持）无法正确处理

2. **括号嵌套深度**
   - 使用简单的计数器处理嵌套
   - 极端嵌套情况下可能有问题（虽然实际中罕见）

3. **字符分类的一致性**
   - 依赖 `Cursor.ts` 中的正则表达式
   - 如果 `Cursor.ts` 的实现在不同版本间变化，行为可能不一致

### 改进建议

1. **支持转义引号**
   ```typescript
   // 建议添加
   function findQuoteWithEscape(text, quote, isInner) {
     // 处理转义引号，如 \"
   }
   ```

2. **缓存 grapheme 分割结果**
   - 对于频繁操作的文本，考虑缓存 grapheme 边界
   - 可以在 `MeasuredText` 类中实现

3. **添加更多文本对象类型**
   - `it`/`at` - HTML/XML 标签
   - `ip`/`ap` - 段落
   - `is`/`as` - 句子

4. **优化大文本性能**
   - 单词查找时，不需要分割整个文本的 graphemes
   - 可以从光标位置向两边扩展查找

5. **增强错误处理**
   - 当前返回 `null` 表示查找失败
   - 考虑添加错误原因信息，便于调试

### 测试建议

需要覆盖的边界情况：
- 光标在文本开头/结尾的单词对象
- 空引号（`""`）的处理
- 未闭合的括号
- 嵌套括号（`((()))`）
- Unicode 字符（emoji、CJK）的单词边界
- 混合空白字符（空格、制表符）的 around 模式

### 代码质量观察

1. **类型安全**
   - 使用 TypeScript 的严格类型
   - `TextObjectRange` 类型清晰表达可能返回 `null`

2. **代码组织**
   - 每个文本对象类型有独立的查找函数
   - 逻辑清晰，易于扩展

3. **注释质量**
   - 关键逻辑有注释说明
   - 引号配对逻辑有明确的注释解释配对规则
