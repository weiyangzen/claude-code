# render-border.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`render-border.ts` 是 Ink 渲染系统的边框绘制模块，负责将 DOM 节点的边框样式渲染到终端输出缓冲区。它是 Ink 终端 UI 框架中视觉呈现层的关键组件，专门处理 Box 组件的边框渲染。

### 1.2 核心职责
- **边框渲染**：根据节点样式计算并绘制四边边框（上、下、左、右）
- **边框文本嵌入**：支持在顶部或底部边框中嵌入文本（如标题）
- **样式应用**：应用边框颜色、暗淡效果等视觉样式
- **自定义边框**：支持预定义边框样式（cli-boxes）和自定义虚线边框

### 1.3 使用场景
- Box 组件设置 `borderStyle` 属性时触发渲染
- 需要绘制带标题的面板、对话框、卡片等 UI 元素
- 需要区分不同视觉层级的容器（通过边框样式）

---

## 2. 功能点目的

### 2.1 BorderTextOptions - 边框文本配置
```typescript
export type BorderTextOptions = {
  content: string      // 预渲染字符串（含 ANSI 颜色码）
  position: 'top' | 'bottom'  // 文本位置
  align: 'start' | 'end' | 'center'  // 对齐方式
  offset?: number      // start/end 对齐时的偏移量
}
```

**设计目的**：允许在边框线上嵌入标题或状态文本，常见于：
- 面板标题（居中显示）
- 状态指示器（靠右显示）
- 标签（靠左显示）

### 2.2 CUSTOM_BORDER_STYLES - 自定义边框样式
```typescript
export const CUSTOM_BORDER_STYLES = {
  dashed: {
    top: '╌', left: '╎', right: '╎', bottom: '╌',
    topLeft: ' ', topRight: ' ', bottomLeft: ' ', bottomRight: ' '
  }
}
```

**设计目的**：cli-boxes 库不提供虚线边框，通过自定义 Unicode 字符实现轻量级虚线效果。

### 2.3 BorderStyle 类型联合
```typescript
export type BorderStyle =
  | keyof Boxes           // cli-boxes 预定义样式
  | keyof typeof CUSTOM_BORDER_STYLES  // 自定义样式
  | BoxStyle              // 用户自定义 BoxStyle 对象
```

**设计目的**：提供三层扩展性：
1. 使用内置样式（single, double, round 等）
2. 使用扩展样式（dashed）
3. 完全自定义字符

---

## 3. 具体技术实现

### 3.1 核心函数：renderBorder

```typescript
const renderBorder = (x: number, y: number, node: DOMNode, output: Output): void
```

**执行流程**：

1. **尺寸计算**（行 89-90）
   ```typescript
   const width = Math.floor(node.yogaNode!.getComputedWidth())
   const height = Math.floor(node.yogaNode!.getComputedHeight())
   ```
   从 Yoga 布局引擎获取计算后的宽高

2. **边框样式解析**（行 91-96）
   ```typescript
   const box = typeof node.style.borderStyle === 'string'
     ? (CUSTOM_BORDER_STYLES[...] ?? cliBoxes[...])
     : node.style.borderStyle
   ```
   支持字符串样式名或自定义 BoxStyle 对象

3. **颜色配置**（行 98-115）
   - 支持四边独立颜色：`borderTopColor`, `borderRightColor`, `borderBottomColor`, `borderLeftColor`
   - 回退到统一 `borderColor`
   - 支持暗淡效果：`borderDimColor` 及各边独立版本

4. **可见性控制**（行 117-120）
   ```typescript
   const showTopBorder = node.style.borderTop !== false
   ```
   通过 `borderTop`/`borderBottom`/`borderLeft`/`borderRight` 控制单边显示

5. **内容宽度计算**（行 122-125）
   ```typescript
   const contentWidth = Math.max(0, width - (showLeftBorder ? 1 : 0) - (showRightBorder ? 1 : 0))
   ```
   减去左右边框占用的 2 列

6. **顶部边框渲染**（行 127-153）
   - 构建边框线：`topLeft + top.repeat(contentWidth) + topRight`
   - 支持嵌入文本：调用 `embedTextInBorder` 计算文本位置
   - 应用颜色：调用 `styleBorderLine` 添加颜色和暗淡效果

7. **垂直边框渲染**（行 155-181）
   - 计算垂直边框高度（总高度减去上下边框）
   - 使用 `repeat()` 生成多行垂直边框
   - 分别处理左边框和右边框

8. **底部边框渲染**（行 183-209）
   - 与顶部边框逻辑相同
   - 同样支持文本嵌入

9. **输出写入**（行 211-227）
   - 按顺序写入：顶部 → 左垂直 → 右垂直 → 底部
   - 使用 `output.write(x, y, text)` 写入 Output 缓冲区

### 3.2 辅助函数：embedTextInBorder

**功能**：在边框线中嵌入文本，返回三段式结果 `[before, text, after]`

**算法**：
1. 计算文本长度（使用 `stringWidth` 处理宽字符）
2. 文本过长时截断到边框长度
3. 根据对齐方式计算位置：
   - `center`: `(borderLength - textLength) / 2`
   - `start`: `offset + 1`（+1 考虑角落字符）
   - `end`: `borderLength - textLength - offset - 1`
4. 确保位置有效（1 到 `borderLength - textLength - 1` 之间）
5. 构建三段：before（边框字符 + 填充）、text、after（填充 + 边框字符）

### 3.3 辅助函数：styleBorderLine

**功能**：应用颜色和暗淡效果到边框线

```typescript
function styleBorderLine(line: string, color: Color | undefined, dim: boolean | undefined): string
```

**处理流程**：
1. 调用 `applyColor(line, color)` 应用前景色
2. 如果 `dim` 为 true，使用 `chalk.dim()` 添加暗淡效果

---

## 4. 关键代码路径与文件引用

### 4.1 调用路径

```
Box 组件设置 borderStyle
    ↓
render-node-to-output.ts (行 1206)
    ↓
renderBorder(x, y, node, output)
    ↓
output.write() → Output 缓冲区
```

### 4.2 关键文件引用

| 文件 | 用途 |
|------|------|
| `cli-boxes` | 提供预定义边框样式（single, double, round 等） |
| `./colorize.js` | `applyColor` 函数，应用颜色到文本 |
| `./dom.js` | `DOMNode` 类型定义 |
| `./output.js` | `Output` 类型，渲染输出缓冲区 |
| `./stringWidth.js` | `stringWidth` 函数，计算字符串显示宽度（处理宽字符） |
| `./styles.js` | `Color` 类型定义 |

### 4.3 代码位置详情

- **渲染触发点**：`render-node-to-output.ts` 行 1203-1206
  ```typescript
  // Render border AFTER children to ensure it's not overwritten by child
  // clearing operations.
  renderBorder(x, y, node, output)
  ```
  注意：边框在子元素之后渲染，防止子元素清除操作覆盖边框

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 版本/类型 | 用途 |
|------|----------|------|
| `chalk` | npm | 终端颜色处理、暗淡效果 |
| `cli-boxes` | npm | 预定义边框字符集 |

### 5.2 内部模块依赖

```typescript
import chalk from 'chalk'
import cliBoxes, { type Boxes, type BoxStyle } from 'cli-boxes'
import { applyColor } from './colorize.js'
import type { DOMNode } from './dom.js'
import type Output from './output.js'
import { stringWidth } from './stringWidth.js'
import type { Color } from './styles.js'
```

### 5.3 数据流交互

```
┌─────────────────┐
│   DOMNode       │  ← 包含 borderStyle, borderColor 等样式属性
│   (from Yoga)   │  ← 包含计算后的 width, height
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  renderBorder   │  ← 解析样式，构建边框字符
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  Output.write   │  ← 写入渲染操作队列
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  Screen (cells) │  ← 最终屏幕缓冲区
└─────────────────┘
```

---

## 6. 风险、边界与改进建议

### 6.1 已知边界情况

1. **文本过长截断**（行 45-47）
   ```typescript
   if (textLength >= borderLength - 2) {
     return ['', text.substring(0, borderLength), '']
   }
   ```
   当文本长度超过边框长度时，直接截断而不显示省略号，可能导致信息丢失

2. **零宽高处理**
   依赖调用方（render-node-to-output.ts）在 height=0 时提前返回，本模块不做检查

3. **负坐标处理**
   绝对定位节点在 `render-node-to-output.ts` 中行 448-450 进行 y<0 的钳制，本模块假设坐标已规范化

### 6.2 潜在风险

| 风险 | 严重程度 | 说明 |
|------|----------|------|
| 非等宽字体显示错位 | 中 | 依赖终端使用等宽字体，边框字符与空格宽度必须一致 |
| 宽字符文本对齐问题 | 低 | `stringWidth` 已处理，但某些终端对特定 Unicode 支持不一致 |
| 颜色代码嵌套 | 低 | `borderText.content` 已包含 ANSI 码时，外层颜色可能冲突 |

### 6.3 改进建议

1. **添加省略号支持**
   ```typescript
   // 当前：直接截断
   // 建议：text.substring(0, borderLength - 1) + '…'
   ```

2. **支持更多边框样式**
   - 添加点线边框（dotted）
   - 支持粗线边框（thick）

3. **优化文本嵌入算法**
   - 当前使用简单截断，可考虑支持滚动文本或自动缩小字体

4. **添加边框圆角与文本同时存在的处理**
   - 当前角落字符在文本嵌入时可能被覆盖，需要更精细的冲突检测

5. **类型安全增强**
   ```typescript
   // 当前 BoxStyle 来自 cli-boxes，但类型定义较宽松
   // 建议：更严格的运行时验证
   ```

### 6.4 测试建议

- 边界测试：width=1, height=1 时的行为
- Unicode 测试：CJK 字符、Emoji 作为边框文本
- 颜色测试：各种颜色格式（hex, rgb, ansi256, ansi）
- 组合测试：borderText + 单边禁用（borderTop: false）
