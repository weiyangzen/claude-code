# widest-line.ts 研究文档

## 场景与职责

`widest-line.ts` 计算多行文本中最宽行的显示宽度。这是文本布局的关键工具函数，用于确定文本块的固有宽度。

### 核心职责
1. **行宽度计算**: 遍历所有行，找出最大显示宽度
2. **ANSI 安全**: 正确处理包含 ANSI 转义序列的文本
3. **缓存利用**: 使用 `line-width-cache` 加速重复计算

## 功能点目的

### 1. 最宽行检测
```typescript
export function widestLine(string: string): number
```

将文本按 `\n` 分割，计算每行的显示宽度，返回最大值。

**应用场景**:
- 确定文本组件的固有宽度
- 计算 Box 的最小内容宽度
- 文本对齐时的参考宽度

### 2. 显示宽度计算
使用 `lineWidth()` 函数（带缓存的 `stringWidth`）计算每行宽度：
- 正确处理 ANSI 转义序列（不计入宽度）
- 正确处理宽字符（CJK、emoji 占 2 列）
- 利用缓存加速重复行的计算

## 具体技术实现

### 核心算法
```typescript
export function widestLine(string: string): number {
  let maxWidth = 0
  let start = 0

  while (start <= string.length) {
    const end = string.indexOf('\n', start)
    const line =
      end === -1 ? string.substring(start) : string.substring(start, end)

    maxWidth = Math.max(maxWidth, lineWidth(line))

    if (end === -1) break
    start = end + 1
  }

  return maxWidth
}
```

### 处理流程
1. 初始化 `maxWidth = 0`
2. 遍历文本，按 `\n` 分割
3. 对每行调用 `lineWidth()` 计算显示宽度
4. 更新最大值
5. 返回 `maxWidth`

### 边界处理
- **空文本**: 返回 0
- **单行文本**: 返回该行的显示宽度
- **末尾换行**: 正确处理末尾 `\n`（不计算空行）
- **连续换行**: 正确处理空行（宽度为 0）

## 关键代码路径与文件引用

### 依赖
```typescript
import { lineWidth } from './line-width-cache.js'
```

### 调用方
- **`measure-text.ts`**: 测量文本尺寸
- **`output.ts`**: 计算文本写入的宽度需求
- **布局代码**: 确定文本节点固有宽度

### 使用示例
```typescript
import { widestLine } from './widest-line.js'

const text = 'Hello\nWorld!\n🎉'
const width = widestLine(text)  // 返回 7（World! 的宽度，emoji 占 2）
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `line-width-cache.ts` | 带缓存的行宽度计算 |

### 外部交互
- **stringWidth**: 通过 `line-width-cache` 间接使用

## 风险、边界与改进建议

### 已知风险
1. **性能**: 大文本需要遍历所有字符查找换行符
2. **内存**: 临时字符串创建（`substring`）有 GC 压力
3. **缓存失效**: `lineWidth` 缓存可能占用较多内存

### 边界情况
1. **空字符串**: 返回 0
2. **仅换行符**: `"\n\n"` 返回 0（空行宽度为 0）
3. **无换行符**: 单行情形，等效于 `lineWidth(string)`
4. **ANSI 序列**: 转义序列不计入宽度

### 改进建议
1. **索引优化**: 使用 `String.prototype.split` 或正则可能更快
2. **零拷贝**: 避免 `substring`，使用索引传递
3. **流式处理**: 大文本支持流式处理
4. **提前退出**: 如果已知最大可能宽度，提前退出
5. **并行计算**: 极长文本可考虑分片并行

### 性能考虑
- **时间复杂度**: O(n)，n 为文本长度
- **空间复杂度**: O(1) 额外空间（不考虑临时字符串）
- **缓存收益**: 重复行计算从 O(m) 降到 O(1)，m 为行长度
