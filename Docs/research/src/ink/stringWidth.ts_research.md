# stringWidth.ts 研究文档

## 场景与职责

`stringWidth.ts` 是 Ink 终端渲染引擎的文本宽度计算模块，负责计算字符串在终端中的显示宽度。这是终端 UI 布局的基础，直接影响文本换行、对齐和容器尺寸计算。

**核心职责：**
1. **显示宽度计算** - 计算字符串在终端中占用的列数
2. **宽字符支持** - 正确处理 CJK、Emoji 等双宽字符
3. **ANSI 序列处理** - 忽略 ANSI 转义序列（不占用显示宽度）
4. **Grapheme Cluster 支持** - 正确处理 Unicode 组合字符
5. **性能优化** - 提供快速路径和 Bun 运行时优化

**在架构中的位置：**
- 被 `output.ts`、`wrap-text.ts`、`tabstops.ts` 等模块广泛使用
- 被 `line-width-cache.ts` 缓存结果
- 是 Yoga 布局引擎测量文本的基础

---

## 功能点目的

### 1. 显示宽度计算

**目的：** 确定字符串在终端中实际占用的列数

**问题复杂性：**
- ASCII 字符宽度为 1
- CJK 字符宽度通常为 2
- Emoji 宽度通常为 2（但国旗等特殊）
- ANSI 转义序列宽度为 0
- 组合字符（如带重音符号的字母）宽度为 1
- 零宽字符（如 ZWJ）宽度为 0

### 2. 运行时适配

**目的：** 利用 Bun 运行时的原生优化

**策略：**
- 检测 `Bun.stringWidth` 可用性
- 存在时使用原生实现（C 级性能）
- 不存在时回退到 JavaScript 实现

**Bun 选项：**
```typescript
const BUN_STRING_WIDTH_OPTS = { ambiguousIsNarrow: true }
// ambiguousIsNarrow: 将模糊宽度字符视为窄字符（符合西方语境）
```

### 3. 快速路径优化

**目的：** 减少常见情况的计算开销

**快速路径：**
1. **纯 ASCII** - 直接返回字符串长度（排除控制字符）
2. **简单 Unicode** - 无 Emoji、无变体选择器时逐字符计算
3. **完整处理** - 使用 grapheme segmenter 处理复杂情况

### 4. 特殊字符处理

**Emoji 处理：**
- 使用 `emoji-regex` 检测 Emoji
- 区域指示符（国旗）特殊处理：单个=1，成对=2
- 不完整的 keycap 序列（数字+VS16 无 U+20E3）宽度为 1

**零宽字符：**
- 控制字符（0x00-0x1f, 0x7f-0x9f）
- 零宽空格/连接符（0x200b-0x200d）
- 变体选择器（0xfe00-0xfe0f, 0xe0100-0xe01ef）
- 组合附加符号（多个 Unicode 范围）
- 印度语系组合标记
- 泰语/老挝语声调符号

---

## 具体技术实现

### 主函数结构

```typescript
export const stringWidth: (str: string) => number = bunStringWidth
  ? str => bunStringWidth(str, BUN_STRING_WIDTH_OPTS)
  : stringWidthJavaScript

function stringWidthJavaScript(str: string): number {
  // 1. 空字符串检查
  if (!str || str.length === 0) return 0
  
  // 2. 纯 ASCII 快速路径
  if (isPureAscii(str)) return countPrintableAscii(str)
  
  // 3. 去除 ANSI 序列
  if (str.includes('\x1b')) {
    str = stripAnsi(str)
    if (str.length === 0) return 0
  }
  
  // 4. 简单 Unicode 快速路径
  if (!needsSegmentation(str)) {
    return countSimpleUnicode(str)
  }
  
  // 5. 完整 grapheme cluster 处理
  return countGraphemeClusters(str)
}
```

### ASCII 快速路径

```typescript
function isPureAscii(str: string): boolean {
  for (let i = 0; i < str.length; i++) {
    const code = str.charCodeAt(i)
    // 检查非 ASCII 或 ANSI 转义（0x1b）
    if (code >= 127 || code === 0x1b) return false
  }
  return true
}

function countPrintableAscii(str: string): number {
  let width = 0
  for (let i = 0; i < str.length; i++) {
    const code = str.charCodeAt(i)
    if (code > 0x1f) width++  // 排除控制字符
  }
  return width
}
```

### 分段检测

```typescript
function needsSegmentation(str: string): boolean {
  for (const char of str) {
    const cp = char.codePointAt(0)!
    
    // Emoji 范围
    if (cp >= 0x1f300 && cp <= 0x1faff) return true
    if (cp >= 0x2600 && cp <= 0x27bf) return true  // 杂项符号
    if (cp >= 0x1f1e6 && cp <= 0x1f1ff) return true  // 区域指示符
    
    // 变体选择器、ZWJ
    if (cp >= 0xfe00 && cp <= 0xfe0f) return true
    if (cp === 0x200d) return true  // ZWJ
  }
  return false
}
```

### Grapheme Cluster 处理

```typescript
function countGraphemeClusters(str: string): number {
  let width = 0
  
  for (const { segment: grapheme } of getGraphemeSegmenter().segment(str)) {
    // 检查 Emoji
    EMOJI_REGEX.lastIndex = 0
    if (EMOJI_REGEX.test(grapheme)) {
      width += getEmojiWidth(grapheme)
      continue
    }
    
    // 非 Emoji grapheme：取第一个非零宽字符的宽度
    for (const char of grapheme) {
      const codePoint = char.codePointAt(0)!
      if (!isZeroWidth(codePoint)) {
        width += eastAsianWidth(codePoint, { ambiguousAsWide: false })
        break
      }
    }
  }
  
  return width
}
```

### Emoji 宽度计算

```typescript
function getEmojiWidth(grapheme: string): number {
  const first = grapheme.codePointAt(0)!
  
  // 区域指示符（国旗）
  if (first >= 0x1f1e6 && first <= 0x1f1ff) {
    let count = 0
    for (const _ of grapheme) count++
    return count === 1 ? 1 : 2  // 单个=1，成对=2
  }
  
  // 不完整的 keycap 序列
  if (grapheme.length === 2) {
    const second = grapheme.codePointAt(1)
    if (second === 0xfe0f && isDigitOrHashOrStar(first)) {
      return 1  // 如 "3\uFE0F" 无 U+20E3
    }
  }
  
  return 2  // 默认 Emoji 宽度
}
```

### 零宽字符检测

```typescript
function isZeroWidth(codePoint: number): boolean {
  // 快速路径：常见可打印范围
  if (codePoint >= 0x20 && codePoint < 0x7f) return false
  if (codePoint >= 0xa0 && codePoint < 0x0300) return codePoint === 0x00ad
  
  // 控制字符
  if (codePoint <= 0x1f || (codePoint >= 0x7f && codePoint <= 0x9f)) return true
  
  // 零宽字符
  if ((codePoint >= 0x200b && codePoint <= 0x200d) ||  // ZW space/joiner
      codePoint === 0xfeff ||  // BOM
      (codePoint >= 0x2060 && codePoint <= 0x2064)) return true
  
  // 变体选择器
  if ((codePoint >= 0xfe00 && codePoint <= 0xfe0f) ||
      (codePoint >= 0xe0100 && codePoint <= 0xe01ef)) return true
  
  // 组合附加符号（多个范围）...
  // 印度语系、泰语/老挝语、阿拉伯语格式化...
  
  return false
}
```

---

## 关键代码路径与文件引用

### 导出内容

| 名称 | 类型 | 位置 | 说明 |
|------|------|------|------|
| `stringWidth` | function | line 220-222 | 主导出，自动选择实现 |

### 导入依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `emoji-regex` | default | Emoji 检测 |
| `get-east-asian-width` | eastAsianWidth | 东亚字符宽度 |
| `strip-ansi` | default | 去除 ANSI 序列 |
| `../utils/intl.js` | getGraphemeSegmenter | Grapheme 分段 |

### 被调用方

| 模块 | 说明 |
|------|------|
| `wrap-text.ts` | 文本换行计算 |
| `tabstops.ts` | Tab 制表位对齐 |
| `line-width-cache.ts` | 行宽度缓存 |
| `output.ts` | 输出宽度计算 |
| `render-border.ts` | 边框渲染 |
| `components/TagTabs.tsx` | 标签页布局 |
| `components/CustomSelect/select.tsx` | 选择器布局 |
| `components/HistorySearchDialog.tsx` | 搜索对话框 |
| `bridge/bridgeUI.ts` | UI 桥接 |
| `utils/sliceAnsi.ts` | ANSI 字符串切片 |
| `utils/truncate.ts` | 文本截断 |
| `utils/terminal.ts` | 终端工具 |
| `utils/ansiToPng.ts` | PNG 导出 |
| `utils/markdown.ts` | Markdown 处理 |
| `utils/Cursor.ts` | 光标定位 |
| `commands/copy/copy.tsx` | 复制功能 |

---

## 依赖与外部交互

### 与外部库的交互

1. **emoji-regex**
   - 用于检测 Emoji 字符
   - 每次使用前重置 `lastIndex = 0`

2. **get-east-asian-width**
   - 计算东亚字符显示宽度
   - 使用 `ambiguousAsWide: false` 选项

3. **strip-ansi**
   - 去除 ANSI 转义序列
   - 在计算宽度前处理

4. **Intl.Segmenter** (通过 intl.js)
   - 用于 grapheme cluster 分割
   - 支持复杂 Unicode 字符

### 与 line-width-cache.ts 的交互

- `line-width-cache.ts` 缓存 `stringWidth` 的计算结果
- 避免重复计算相同字符串
- 显著提高渲染性能

### 与 wrap-text.ts 的交互

- 文本换行时计算每行宽度
- 确定换行位置
- 处理宽字符的边界情况

---

## 风险、边界与改进建议

### 已知风险

1. **Bun 版本差异**
   - `Bun.stringWidth` 行为可能随版本变化
   - 选项 `ambiguousIsNarrow` 需要验证兼容性

2. **emoji-regex 性能**
   - 正则表达式测试在热路径中
   - 每次重置 `lastIndex` 可能有开销

3. **Grapheme Segmenter 兼容性**
   - `Intl.Segmenter` 需要较新的 Node/Bun 版本
   - 旧环境需要 polyfill

4. **复杂脚本处理**
   - 注释提到 Devanagari 等复杂脚本的宽度计算可能与终端不一致
   - 可能导致布局错位

### 边界情况

1. **空字符串**
   - 返回 0

2. **纯 ANSI 序列**
   - `stripAnsi` 后长度为 0，返回 0

3. **控制字符**
   - ASCII 控制字符（0x00-0x1f）不计入宽度
   - 但某些终端可能显示为 `^X` 形式

4. **不完整的 Emoji 序列**
   - 如单独的 VS16（\uFE0F）
   - 按零宽字符处理

5. **组合字符**
   - 基础字符 + 多个组合标记
   - 整体宽度等于基础字符宽度

### 改进建议

1. **性能优化**
   - 使用 `Intl.Segmenter` 的 `granularity: 'grapheme'` 替代正则检测 Emoji
   - 缓存 `needsSegmentation` 的结果
   - 对于已知纯 ASCII 的调用点，提供 `stringWidthAscii` 快速函数

2. **准确性改进**
   - 添加终端检测，根据终端类型调整宽度计算
   - 支持配置 `ambiguousAsWide`（东亚用户可能需要）
   - 添加更多 Emoji 变体处理（肤色、性别修饰符）

3. **测试覆盖**
   - 添加全面的单元测试，覆盖各种 Unicode 范围
   - 与常见终端（iTerm2、Windows Terminal、VS Code）对比验证
   - 测试性能边界（超长字符串）

4. **代码质量**
   - 将 `isZeroWidth` 中的魔法数字提取为命名常量
   - 使用 TypeScript 的 bigint 处理大码点
   - 添加 JSDoc 说明各 Unicode 范围

5. **功能扩展**
   - 支持双向文本（Bidi）的宽度计算
   - 支持垂直书写模式的宽度概念
   - 提供 `stringWidthMax` 函数（计算多行字符串的最大宽度）

### 已知问题

1. **与终端的不一致**
   ```typescript
   // 代码注释中提到：
   // Bun.stringWidth=2 matches terminal cell allocation, which is what
   // we need for cursor positioning — the JS fallback's grapheme-cluster 
   // width of 1 would desync Ink's layout from the terminal.
   ```
   - 某些复杂 grapheme cluster 在终端中可能占用 2 列
   - JavaScript 回退实现可能计算为 1

2. **缓存一致性**
   - `line-width-cache.ts` 缓存结果
   - 如果 `stringWidth` 实现变化，需要清空缓存

3. **内存使用**
   - `EMOJI_REGEX` 是全局正则
   - 频繁调用可能累积状态
