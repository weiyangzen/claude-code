# styles.ts 研究文档

## 场景与职责

`styles.ts` 是 Ink 终端 UI 框架的样式系统核心模块，负责将 React 组件声明式样式（CSS-like）转换为 Yoga 布局引擎可理解的底层指令。它是连接组件层与布局引擎的桥梁，实现了 Flexbox 布局在终端环境中的完整支持。

### 核心职责
1. **样式类型定义**：定义完整的 TypeScript 类型系统，包括颜色、文本样式、布局样式
2. **样式应用**：将样式对象转换为 Yoga 布局节点的具体方法调用
3. **单位转换**：处理百分比、数值、自动等多种尺寸单位
4. **边框管理**：支持自定义边框样式和单边控制

## 功能点目的

### 1. 颜色系统 (`Color` 类型)
支持多种颜色格式，满足终端颜色表达需求：
- **RGB**: `rgb(255,128,0)` - 真彩色支持
- **Hex**: `#FF8000` - 标准十六进制
- **ANSI256**: `ansi256(208)` - 256色终端兼容
- **ANSI**: `ansi:red`, `ansi:redBright` - 16色基础调色板

### 2. 文本样式 (`TextStyles`)
独立于 ANSI 字符串的结构化文本样式：
```typescript
type TextStyles = {
  color?: Color           // 前景色
  backgroundColor?: Color // 背景色
  dim?: boolean          // 暗淡效果
  bold?: boolean         // 粗体
  italic?: boolean       // 斜体
  underline?: boolean    // 下划线
  strikethrough?: boolean // 删除线
  inverse?: boolean      // 反色
}
```

### 3. 布局样式 (`Styles`)
完整的 Flexbox 布局属性支持：
- **定位**: `position`, `top`, `bottom`, `left`, `right` (绝对/相对定位)
- **盒模型**: `margin`, `padding`, `gap` (含 X/Y 简写)
- **Flex 属性**: `flexGrow`, `flexShrink`, `flexDirection`, `flexWrap`, `flexBasis`
- **对齐**: `alignItems`, `alignSelf`, `justifyContent`
- **尺寸**: `width`, `height`, `min/max` 约束 (支持百分比)
- **溢出**: `overflow`, `overflowX`, `overflowY` (visible/hidden/scroll)
- **边框**: `borderStyle`, `borderColor`, 单边控制
- **特殊**: `opaque` (填充背景), `noSelect` (文本选择排除)

### 4. 文本换行 (`textWrap`)
支持多种换行策略：
- `wrap`: 硬换行，保留所有内容
- `wrap-trim`: 硬换行并修剪行尾空格
- `truncate-end/middle/start`: 截断显示省略号

## 具体技术实现

### 样式应用流程

```typescript
const styles = (node: LayoutNode, style: Styles, resolvedStyle?: Styles): void
```

主函数按固定顺序调用各专项应用函数：
1. `applyPositionStyles` - 定位类型和边距
2. `applyOverflowStyles` - 溢出行为控制
3. `applyMarginStyles` - 外边距（含简写展开）
4. `applyPaddingStyles` - 内边距（含简写展开）
5. `applyFlexStyles` - Flexbox 属性
6. `applyDimensionStyles` - 宽高尺寸
7. `applyDisplayStyles` - 显示/隐藏
8. `applyBorderStyles` - 边框（依赖 resolvedStyle 处理增量更新）
9. `applyGapStyles` - 行列间距

### 关键实现细节

#### 百分比处理
```typescript
if (typeof style.width === 'string') {
  node.setWidthPercent(Number.parseInt(style.width, 10))
}
```
百分比值通过模板字面量类型 `${number}%` 约束，解析时去除 `%` 符号。

#### 简写属性展开
```typescript
if ('marginX' in style) {
  node.setMargin(LayoutEdge.Horizontal, style.marginX ?? 0)
}
```
`marginX` 同时设置 `marginLeft` 和 `marginRight`，在 Yoga 层使用 `LayoutEdge.Horizontal` 常量。

#### 边框增量更新
```typescript
const resolved = resolvedStyle ?? style
if ('borderStyle' in style) {
  // 使用 resolved 获取完整的边框状态
}
```
支持 React 的增量更新模式：当只传递变更的属性时，通过 `resolvedStyle` 获取完整状态计算边框。

#### 溢出与滚动
```typescript
if (y === 'scroll' || x === 'scroll') {
  node.setOverflow(LayoutOverflow.Scroll)
}
```
`scroll` 值在布局层和渲染层有不同含义：布局层阻止子元素扩展容器，渲染层启用 `scrollTop` 虚拟滚动。

## 关键代码路径与文件引用

### 类型依赖
```typescript
// 布局节点类型
import { LayoutNode, LayoutEdge, LayoutGutter, ... } from './layout/node.js'
// 边框样式类型
import type { BorderStyle, BorderTextOptions } from './render-border.js'
```

### 被调用方
- **`dom.ts`**: `setStyle()` 调用 `styles()` 应用样式到 Yoga 节点
- **`render-border.ts`**: 使用 `Styles` 类型中的边框相关属性
- **组件层**: `Box`, `Text` 等组件通过 `styles` 属性传递样式对象

### 常量映射
| CSS 值 | Yoga 常量 |
|--------|-----------|
| `flex-start` | `LayoutAlign.FlexStart` |
| `center` | `LayoutAlign.Center` |
| `flex-end` | `LayoutAlign.FlexEnd` |
| `stretch` | `LayoutAlign.Stretch` |
| `space-between` | `LayoutJustify.SpaceBetween` |
| `row` | `LayoutFlexDirection.Row` |
| `nowrap` | `LayoutWrap.NoWrap` |
| `absolute` | `LayoutPositionType.Absolute` |

## 依赖与外部交互

### 内部依赖
| 文件 | 用途 |
|------|------|
| `layout/node.ts` | Yoga 布局节点接口和常量 |
| `render-border.ts` | 边框样式类型定义 |

### 外部库
- **Yoga**: 底层 Flexbox 布局引擎（通过 `layout/node.ts` 封装）

## 风险、边界与改进建议

### 已知风险
1. **类型安全**: `Styles` 类型非常庞大，容易遗漏属性校验
2. **性能**: 每次样式变更都触发完整 Yoga 树重新计算
3. **单位混淆**: 百分比和像素数值在类型上无法区分（都是 `number`）

### 边界情况
1. **负值处理**: 部分属性（如 margin）支持负值，但 Yoga 可能有约束
2. **百分比基数**: 百分比计算依赖父节点尺寸，在根节点可能行为不一致
3. **边框冲突**: 当 `borderStyle` 和单边属性同时变更时，需要 `resolvedStyle` 协调

### 改进建议
1. **性能优化**: 实现样式对象浅比较，避免未变更样式的重复应用
2. **类型增强**: 使用 branded types 区分像素和百分比数值
3. **文档生成**: 从 JSDoc 自动生成样式属性参考文档
4. **验证层**: 添加运行时样式值验证（如颜色格式检查）
5. **CSS 兼容**: 考虑支持更多 CSS 单位（rem, vw/vh 等）

### 测试建议
- 边界值测试（0, 负数, 极大值）
- 百分比计算精度验证
- 简写属性与单独属性的优先级测试
- 增量更新时 resolvedStyle 的正确性
