# get-max-width.ts 研究文档

## 场景与职责

`get-max-width.ts` 是一个简单的工具函数，用于计算 Yoga 布局节点的内容宽度（content width）。它是文本换行和布局计算的基础工具。

### 核心职责

1. **内容宽度计算**：从 Yoga 节点的计算宽度中减去 padding 和 border
2. **文本换行支持**：为文本测量提供可用宽度
3. **布局一致性**：确保文本换行与 Yoga 布局结果一致

## 功能点目的

### 1. 内容宽度计算

Yoga 的 `getComputedWidth()` 返回的是包含 padding 和 border 的总宽度。对于文本换行，需要计算实际可用的内容宽度：

```
内容宽度 = 计算宽度 - 左padding - 右padding - 左border - 右border
```

### 2. 文本换行场景

在 `measureTextNode`（dom.ts）中使用：
- 获取 Yoga 节点分配的宽度
- 计算可用于文本的实际宽度
- 传递给 `wrapText` 进行换行

### 3. 布局一致性警告

文档注释中明确警告：返回值可能比父容器更宽。这是标准 CSS/Yoga 行为：
- 在 column 方向的 flex 父节点中，width 是 cross axis
- `align-items: stretch` 不会将子节点收缩到小于其固有尺寸
- 文本节点可能溢出

## 具体技术实现

### 实现代码

```typescript
import { LayoutEdge, type LayoutNode } from './layout/node.js'

const getMaxWidth = (yogaNode: LayoutNode): number => {
  return (
    yogaNode.getComputedWidth() -
    yogaNode.getComputedPadding(LayoutEdge.Left) -
    yogaNode.getComputedPadding(LayoutEdge.Right) -
    yogaNode.getComputedBorder(LayoutEdge.Left) -
    yogaNode.getComputedBorder(LayoutEdge.Right)
  )
}

export default getMaxWidth
```

### 计算逻辑

1. **获取计算宽度**：`yogaNode.getComputedWidth()`
2. **减去水平 padding**：Left 和 Right
3. **减去水平 border**：Left 和 Right
4. **返回内容宽度**：可用于文本的最大宽度

### 与 Yoga 的交互

Yoga 的两遍测量过程：
1. **AtMost 遍**：确定宽度，结果可能较宽
2. **Exactly 遍**：确定高度，使用实际约束

`getComputedWidth()` 反映 AtMost 结果，而 `getComputedHeight()` 反映 Exactly 结果。

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/get-max-width.ts`
- **导出函数**：`getMaxWidth`（默认导出）

### 依赖关系

**被导入**：
- `./layout/node.js` - LayoutNode 类型和 LayoutEdge 常量

**导入使用**：
```typescript
import { LayoutEdge, type LayoutNode } from './layout/node.js'
```

### 使用位置

通过代码搜索，该函数被以下模块使用：

| 模块 | 用途 |
|------|------|
| `render-node-to-output.ts` | 文本渲染时获取可用宽度 |
| `dom.ts` | `measureTextNode` 中计算文本宽度 |

### 相关文件

- `src/ink/layout/node.ts` - LayoutNode 接口和 LayoutEdge 定义
- `src/ink/layout/engine.ts` - Yoga 布局引擎实现
- `src/ink/dom.ts` - measureTextNode 实现
- `src/ink/render-node-to-output.ts` - 文本渲染

## 依赖与外部交互

### 调用时机

```
Yoga 布局计算完成
    ↓
需要测量文本
    ↓
getMaxWidth(yogaNode)
    ↓
wrapText(text, maxWidth, textWrap)
    ↓
measureText(wrappedText, maxWidth)
    ↓
返回尺寸给 Yoga
```

### 与 wrapText 的交互

```typescript
// 在 render-node-to-output.ts 中
const maxWidth = getMaxWidth(node.yogaNode!)
const wrappedText = wrapText(text, maxWidth, textWrap)
```

## 风险、边界与改进建议

### 已知风险

1. **宽度溢出**：
   - 返回值可能大于父容器宽度
   - 调用方需要自行处理溢出（如 clamp）

2. **负值风险**：
   - 如果 padding + border > width，返回负值
   - 可能导致后续计算错误

3. **Yoga 节点状态**：
   - 必须在 `calculateLayout` 后调用
   - 否则返回未定义行为

### 边界情况

1. **零宽度**：返回负的 padding/border 总和
2. **未布局节点**：Yoga 可能返回 0 或 NaN
3. **百分比尺寸**：Yoga 已计算为绝对值

### 改进建议

1. **防御性编程**：
   - 添加负值保护
   - 验证 Yoga 节点状态
   
   ```typescript
   const getMaxWidth = (yogaNode: LayoutNode): number => {
     const width = yogaNode.getComputedWidth()
     const padding = 
       yogaNode.getComputedPadding(LayoutEdge.Left) +
       yogaNode.getComputedPadding(LayoutEdge.Right)
     const border =
       yogaNode.getComputedBorder(LayoutEdge.Left) +
       yogaNode.getComputedBorder(LayoutEdge.Right)
     return Math.max(0, width - padding - border)
   }
   ```

2. **错误处理**：
   - 添加开发模式警告
   - 记录异常值用于调试

3. **性能优化**：
   - 缓存计算结果（如果 Yoga 节点未变化）
   - 但 Yoga 布局是每帧重新计算，缓存收益有限

4. **文档完善**：
   - 添加使用示例
   - 明确前置条件（必须在 layout 后调用）

5. **测试覆盖**：
   - 各种 padding/border 组合的测试
   - 边界值（0、负数、极大值）测试
   - 与 Yoga 布局结果的一致性验证
