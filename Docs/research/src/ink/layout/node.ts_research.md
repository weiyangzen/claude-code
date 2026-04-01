# src/ink/layout/node.ts 研究文档

## 场景与职责

`node.ts` 是 Ink 布局系统的**核心抽象层**，定义了与具体布局引擎无关的 `LayoutNode` 接口。它是上层代码（DOM、样式、渲染）与底层 Yoga 实现之间的**契约（Contract）**。

**核心定位**：
- 定义布局节点的**完整接口契约**
- 提供**类型安全**的布局常量枚举（Edge、Align、Justify 等）
- 解耦上层代码与 Yoga 具体实现
- 支持未来替换布局引擎（如需要）

## 功能点目的

### 1. 布局常量枚举

定义了 Yoga/CSS Flexbox 规范中的所有布局常量：

| 常量 | 用途 | 对应 CSS |
|------|------|----------|
| `LayoutEdge` | 边距/边框/定位边 | margin-left, padding-top 等 |
| `LayoutGutter` | Flex 间隙轴 | gap, column-gap, row-gap |
| `LayoutDisplay` | 显示模式 | display: flex/none |
| `LayoutFlexDirection` | 主轴方向 | flex-direction |
| `LayoutAlign` | 交叉轴对齐 | align-items, align-self |
| `LayoutJustify` | 主轴对齐 | justify-content |
| `LayoutWrap` | 换行模式 | flex-wrap |
| `LayoutPositionType` | 定位模式 | position: relative/absolute |
| `LayoutOverflow` | 溢出处理 | overflow: visible/hidden/scroll |
| `LayoutMeasureMode` | 测量模式 | Yoga 内部使用 |

### 2. LayoutNode 接口

定义了布局节点的完整操作集：

**树操作**：
- `insertChild(child, index)` / `removeChild(child)` - 子节点管理
- `getChildCount()` / `getParent()` - 树遍历

**布局计算**：
- `calculateLayout(width?, height?)` - 执行布局计算
- `setMeasureFunc(fn)` / `unsetMeasureFunc()` - 设置/移除测量函数
- `markDirty()` - 标记需要重新布局

**布局读取**：
- `getComputedLeft/Top/Width/Height()` - 获取计算后的布局值
- `getComputedBorder/Padding(edge)` - 获取边距值

**样式设置**（完整 Flexbox 属性支持）：
- 尺寸：`setWidth/Height/MinWidth/MinHeight/MaxWidth/MaxHeight`（支持数值、百分比、auto）
- Flex：`setFlexDirection/Grow/Shrink/Basis/Wrap`
- 对齐：`setAlignItems/Self/JustifyContent`
- 定位：`setPositionType/Position/PositionPercent`
- 间距：`setMargin/Padding/Border/Gap`
- 其他：`setDisplay/Overflow`

**生命周期**：
- `free()` / `freeRecursive()` - 释放节点资源

### 3. 测量函数类型

```typescript
export type LayoutMeasureFunc = (
  width: number,
  widthMode: LayoutMeasureMode,
) => { width: number; height: number }
```

用于文本节点等需要自定义测量逻辑的场景。

## 具体技术实现

### 类型定义模式

使用 TypeScript 的 `const` 对象 + 类型推导模式：

```typescript
export const LayoutEdge = {
  All: 'all',
  Horizontal: 'horizontal',
  // ...
} as const
export type LayoutEdge = (typeof LayoutEdge)[keyof typeof LayoutEdge]
```

**优势**：
- 运行时值为字符串，便于调试
- 编译时类型检查
- 避免传统 enum 的反向映射问题

### LayoutNode 接口设计

```
LayoutNode (接口)
├── 树操作
│   ├── insertChild(child, index)
│   ├── removeChild(child)
│   ├── getChildCount()
│   └── getParent()
├── 布局计算
│   ├── calculateLayout(width?, height?)
│   ├── setMeasureFunc(fn)
│   ├── unsetMeasureFunc()
│   └── markDirty()
├── 布局读取
│   ├── getComputedLeft/Top/Width/Height()
│   └── getComputedBorder/Padding(edge)
├── 样式设置器（~40 个方法）
│   ├── 尺寸设置（数值/百分比/auto）
│   ├── Flex 设置
│   ├── 对齐设置
│   ├── 定位设置
│   └── 间距设置
└── 生命周期
    ├── free()
    └── freeRecursive()
```

## 关键代码路径与文件引用

### 被引用方（核心依赖）

| 文件 | 引用内容 | 用途 |
|------|----------|------|
| `src/ink/layout/yoga.ts` | `LayoutNode` 等全部类型 | 实现 YogaLayoutNode 类 |
| `src/ink/layout/engine.ts` | `LayoutNode` | 工厂函数返回类型 |
| `src/ink/styles.ts` | `LayoutNode`, 所有常量枚举 | 将 React 样式应用到布局节点 |
| `src/ink/dom.ts` | `LayoutDisplay`, `LayoutMeasureMode` | DOM 元素类型定义和测量 |
| `src/ink/render-node-to-output.ts` | `LayoutDisplay`, `LayoutEdge` | 渲染时读取布局值 |
| `src/ink/get-max-width.ts` | `LayoutEdge`, `LayoutNode` | 计算内容宽度 |
| `src/ink/reconciler.ts` | `LayoutDisplay` | 显示/隐藏实例 |

### 引用详情

**`src/ink/styles.ts`** (第 1-12 行):
```typescript
import {
  LayoutAlign,
  LayoutDisplay,
  LayoutEdge,
  LayoutFlexDirection,
  LayoutGutter,
  LayoutJustify,
  type LayoutNode,
  LayoutOverflow,
  LayoutPositionType,
  LayoutWrap,
} from './layout/node.js'
```
这是最主要的消费者，将 React 组件的样式属性转换为 Yoga 节点操作。

**`src/ink/dom.ts`** (第 4 行):
```typescript
import { LayoutDisplay, LayoutMeasureMode } from './layout/node.js'
```

用于 `measureTextNode` 函数中的测量模式判断。

## 依赖与外部交互

### 无运行时依赖

该文件**纯类型定义**：
- 不导入任何其他模块
- 只有类型和常量定义
- 无实际可执行代码

### 与 Yoga 的关系

```
node.ts (接口定义)
    ↑ 实现
yoga.ts (YogaLayoutNode 类)
    ↑ 使用
engine.ts (工厂)
    ↑ 创建
dom.ts (DOM 元素)
    ↑ 关联
styles.ts (样式应用)
```

## 风险、边界与改进建议

### 风险点

1. **接口膨胀**：`LayoutNode` 接口有 40+ 个方法，维护成本高
2. **与 Yoga 紧耦合**：虽然意图是抽象，但接口设计完全匹配 Yoga API
3. **测量函数限制**：`LayoutMeasureFunc` 只接收 width，与 Yoga 的完整签名（width, widthMode, height, heightMode）不完全一致

### 边界情况

1. **空实现风险**：如果实现类未正确实现所有方法，运行时可能出错
2. **类型转换**：`yoga.ts` 中使用 `(child as YogaLayoutNode).yoga` 进行类型断言，有潜在风险

### 改进建议

1. **接口拆分**：将 `LayoutNode` 拆分为多个小接口
   ```typescript
   interface LayoutTreeNode { /* 树操作 */ }
   interface LayoutMeasurable { /* 测量相关 */ }
   interface LayoutStyleable { /* 样式设置 */ }
   interface LayoutNode extends LayoutTreeNode, LayoutMeasurable, LayoutStyleable {}
   ```

2. **测量函数增强**：考虑支持完整的 Yoga 测量签名
   ```typescript
   export type LayoutMeasureFunc = (
     width: number,
     widthMode: LayoutMeasureMode,
     height: number,
     heightMode: LayoutMeasureMode,
   ) => Size
   ```

3. **添加文档注释**：当前文件头只有简短注释，建议添加 JSDoc
   ```typescript
   /**
    * 标记节点及其祖先需要重新布局
    * 通常在样式变更或子节点变更后调用
    */
   markDirty(): void
   ```

4. **验证辅助**：添加运行时类型检查辅助函数
   ```typescript
   export function isLayoutNode(obj: unknown): obj is LayoutNode {
     return obj !== null && 
            typeof obj === 'object' &&
            'calculateLayout' in obj &&
            'getComputedWidth' in obj
   }
   ```

5. **版本兼容性**：如果 Yoga 升级导致 API 变化，考虑添加版本适配层
