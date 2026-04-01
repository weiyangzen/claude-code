# tabstops.ts 研究文档

## 场景与职责

`tabstops.ts` 实现制表符（Tab）展开功能，将文本中的 `\t` 字符转换为适当数量的空格。这是终端文本渲染的关键预处理步骤，确保制表符在不同位置正确对齐。

### 核心职责
1. **Tab 展开**: 将 `\t` 转换为计算数量的空格
2. **列对齐**: 维护 8 列间隔的制表位（POSIX 默认）
3. **ANSI 安全**: 正确处理包含转义序列的文本

## 功能点目的

### 1. 制表位计算
使用 8 列间隔（与 Ghostty、大多数终端一致）：
```typescript
const DEFAULT_TAB_INTERVAL = 8
```

计算逻辑：
```
spaces = interval - (current_column % interval)
```

### 2. 流式处理
通过 tokenizer 处理文本，区分：
- **转义序列**: 原样保留，不增加列计数
- **普通字符**: 按显示宽度增加列计数
- **换行符**: 重置列计数为 0
- **制表符**: 展开为计算数量的空格

## 具体技术实现

### 核心算法
```typescript
export function expandTabs(text: string, interval = DEFAULT_TAB_INTERVAL): string
```

处理流程：
1. 快速路径：不含 `\t` 的文本直接返回
2. Tokenize 文本，分离 ANSI 序列和普通文本
3. 遍历 token，维护当前列位置
4. 遇到 `\t` 时计算到下一个制表位的空格数
5. 遇到 `\n` 时重置列计数

### 列位置跟踪
```typescript
let column = 0
for (const part of parts) {
  if (part === '\t') {
    const spaces = interval - (column % interval)
    result += ' '.repeat(spaces)
    column += spaces
  } else if (part === '\n') {
    result += part
    column = 0  // 重置
  } else {
    result += part
    column += stringWidth(part)  // 使用显示宽度
  }
}
```

### ANSI 序列处理
```typescript
if (token.type === 'sequence') {
  result += token.value  // 直接追加，不影响列计数
}
```

转义序列被识别为独立 token，不参与列位置计算。

## 关键代码路径与文件引用

### 依赖
```typescript
import { stringWidth } from './stringWidth.js'
import { createTokenizer } from './termio/tokenize.js'
```

### 调用方
- **`dom.ts`**: `measureTextNode()` 在测量文本尺寸前展开 Tab
- **`output.ts`**: 渲染时处理 Tab 字符（内联展开）

### 使用场景
```typescript
// dom.ts 中的使用
const text = expandTabs(rawText)
const dimensions = measureText(text, width)
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `stringWidth.ts` | 计算字符显示宽度（处理宽字符、emoji） |
| `termio/tokenize.ts` | 分离 ANSI 转义序列和普通文本 |

## 风险、边界与改进建议

### 已知风险
1. **硬编码间隔**: 8 列间隔不可配置，某些环境可能需要不同设置
2. **性能**: 每行文本都进行 tokenize，高频调用有开销
3. **递归展开**: 如果输入包含已展开的空格，不会重新处理

### 边界情况
1. **空文本**: 返回空字符串
2. **无 Tab 文本**: 快速路径直接返回原字符串
3. **Tab 在行尾**: 可能展开到下一制表位，导致行延长
4. **宽字符**: `stringWidth` 正确处理 CJK、emoji 的 2 列宽度

### 改进建议
1. **可配置间隔**: 支持通过环境变量或选项自定义制表位间隔
2. **缓存优化**: 对重复文本缓存展开结果
3. **延迟展开**: 仅在需要测量或渲染时展开，避免重复处理
4. **混合 Tab/空格**: 考虑支持弹性制表位（elastic tabstops）

### 相关标准
- POSIX 规定制表位间隔为 8 列
- Ghostty 的 `Tabstops.zig` 实现（代码注释提及的参考）
