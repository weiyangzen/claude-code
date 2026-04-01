# wrap-text.ts 研究文档

## 场景与职责

`wrap-text.ts` 实现文本换行和截断功能，根据指定的最大宽度和换行策略处理文本。这是终端文本渲染的核心工具，支持多种换行模式以适应不同 UI 需求。

### 核心职责
1. **文本换行**: 将长文本按指定宽度换行
2. **文本截断**: 支持多种截断位置（开头/中间/结尾）
3. **ANSI 安全**: 正确处理包含 ANSI 转义序列的文本
4. **宽字符支持**: 正确处理 CJK、emoji 等宽字符

## 功能点目的

### 1. 换行策略 (`textWrap`)

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `wrap` | 硬换行，保留所有内容 | 多行文本显示 |
| `wrap-trim` | 硬换行并修剪行尾空格 | 紧凑布局 |
| `truncate` / `truncate-end` | 末尾截断显示省略号 | 空间有限 |
| `truncate-middle` | 中间截断显示省略号 | 保留首尾标识 |
| `truncate-start` | 开头截断显示省略号 | 保留末尾信息 |
| `end` / `middle` | 不处理（保留原样） | 特殊布局 |

### 2. 截断实现
使用 `sliceAnsi` 精确切片 ANSI 文本：
```typescript
function truncate(
  text: string,
  columns: number,
  position: 'start' | 'middle' | 'end',
): string
```

**算法**:
- **end**: 保留前 `columns-1` 个字符 + 省略号
- **start**: 省略号 + 保留后 `columns-1` 个字符
- **middle**: 前半部分 + 省略号 + 后半部分

### 3. 宽字符处理
```typescript
function sliceFit(text: string, start: number, end: number): string {
  const s = sliceAnsi(text, start, end)
  return stringWidth(s) > end - start ? sliceAnsi(text, start, end - 1) : s
}
```

处理边界跨越宽字符的情况：如果切片结果超出目标宽度，收紧边界重试。

## 具体技术实现

### 主函数
```typescript
export default function wrapText(
  text: string,
  maxWidth: number,
  wrapType: Styles['textWrap'],
): string
```

### 换行处理
```typescript
if (wrapType === 'wrap') {
  return wrapAnsi(text, maxWidth, { trim: false, hard: true })
}

if (wrapType === 'wrap-trim') {
  return wrapAnsi(text, maxWidth, { trim: true, hard: true })
}
```

使用 `wrapAnsi` 库（或 Bun 原生实现）进行 ANSI 安全的硬换行。

### 截断处理
```typescript
if (wrapType!.startsWith('truncate')) {
  let position: 'end' | 'middle' | 'start' = 'end'
  if (wrapType === 'truncate-middle') position = 'middle'
  if (wrapType === 'truncate-start') position = 'start'
  return truncate(text, maxWidth, position)
}
```

### 省略号常量
```typescript
const ELLIPSIS = '…'  // Unicode 省略号（非三个点）
```

### 截断算法细节
**End 截断**:
```typescript
return sliceFit(text, 0, columns - 1) + ELLIPSIS
```

**Start 截断**:
```typescript
const length = stringWidth(text)
return ELLIPSIS + sliceFit(text, length - columns + 1, length)
```

**Middle 截断**:
```typescript
const half = Math.floor(columns / 2)
return (
  sliceFit(text, 0, half) +
  ELLIPSIS +
  sliceFit(text, length - (columns - half) + 1, length)
)
```

## 关键代码路径与文件引用

### 依赖
```typescript
import sliceAnsi from '../utils/sliceAnsi.js'
import { stringWidth } from './stringWidth.js'
import type { Styles } from './styles.js'
import { wrapAnsi } from './wrapAnsi.js'
```

### 调用方
- **`dom.ts`**: `measureTextNode()` 在测量时换行文本
- **Text 组件**: 根据 `textWrap` 属性处理文本

### 使用示例
```typescript
import wrapText from './wrap-text.js'

// 硬换行
wrapText('Hello World', 5, 'wrap')  // 'Hello\nWorld'

// 截断
wrapText('Hello World', 8, 'truncate')  // 'Hello W…'
wrapText('Hello World', 8, 'truncate-middle')  // 'Hel…rld'
wrapText('Hello World', 8, 'truncate-start')  // '…o World'
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `utils/sliceAnsi.ts` | ANSI 安全文本切片 |
| `stringWidth.ts` | 计算显示宽度 |
| `styles.ts` | `Styles` 类型定义 |
| `wrapAnsi.ts` | 底层换行实现 |

### 外部交互
- **wrap-ansi npm 包**: Node 环境的换行实现
- **Bun.wrapAnsi**: Bun 运行时的原生优化实现

## 风险、边界与改进建议

### 已知风险
1. **性能**: `sliceAnsi` 和 `wrapAnsi` 都有一定计算开销
2. **精度**: 宽字符边界计算可能有 1 列误差
3. **缓存**: 无缓存机制，重复换行重复计算

### 边界情况
1. **maxWidth < 1**: 返回空字符串
2. **maxWidth === 1**: 仅返回省略号
3. **文本宽度 <= maxWidth**: 返回原文本
4. **空文本**: 返回空字符串
5. **纯 ANSI 序列**: 可能产生意外结果

### 改进建议
1. **缓存**: 对频繁换行的相同文本添加缓存
2. **词边界**: 支持按词边界换行（word-wrap）
3. **连字符**: 支持自动连字符断词
4. **性能优化**: 针对大文本优化切片算法
5. **测试覆盖**: 添加更多边界情况的单元测试

### 相关库
- wrap-ansi: 支持 ANSI 的文本换行
- slice-ansi: 支持 ANSI 的文本切片
- string-width: 计算字符串显示宽度
