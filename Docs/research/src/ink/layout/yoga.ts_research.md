# src/ink/layout/yoga.ts 研究文档

## 场景与职责

`yoga.ts` 是 Ink 布局系统的**Yoga 适配器实现**，将 `node.ts` 中定义的 `LayoutNode` 接口适配到实际的 Yoga 布局引擎。它是连接抽象接口与具体实现的**桥梁**。

**核心定位**：
- 实现 `LayoutNode` 接口的 Yoga 版本
- 处理 Yoga 原生枚举与 Ink 枚举之间的**映射转换**
- 提供 Yoga 节点的**实例管理**
- 封装 Yoga 的复杂性和平台差异（WASM vs 纯 TS）

## 功能点目的

### 1. YogaLayoutNode 类

`YogaLayoutNode` 类实现了 `LayoutNode` 接口，是 Yoga 节点的包装器：

```typescript
export class YogaLayoutNode implements LayoutNode {
  readonly yoga: YogaNode  // 底层 Yoga 节点实例
  // ... 实现所有 LayoutNode 方法
}
```

### 2. 枚举映射

Yoga 使用数字枚举，Ink 使用字符串常量，需要双向映射：

```typescript
const EDGE_MAP: Record<LayoutEdge, Edge> = {
  all: Edge.All,
  horizontal: Edge.Horizontal,
  left: Edge.Left,
  // ...
}
```

支持的映射：
- `EDGE_MAP`: LayoutEdge → Yoga Edge
- `GUTTER_MAP`: LayoutGutter → Yoga Gutter
- 内联映射：FlexDirection、Align、Justify、Wrap、Overflow、PositionType

### 3. 测量函数适配

Yoga 的测量函数签名与 Ink 不同，需要适配：

```typescript
setMeasureFunc(fn: LayoutMeasureFunc): void {
  this.yoga.setMeasureFunc((w, wMode) => {
    const mode = wMode === MeasureMode.Exactly
      ? LayoutMeasureMode.Exactly
      : wMode === MeasureMode.AtMost
        ? LayoutMeasureMode.AtMost
        : LayoutMeasureMode.Undefined
    return fn(w, mode)  // 只传递 width 相关参数
  })
}
```

### 4. 工厂函数

```typescript
export function createYogaLayoutNode(): LayoutNode {
  return new YogaLayoutNode(Yoga.Node.create())
}
```

## 具体技术实现

### 类结构

```
YogaLayoutNode
├── 属性
│   └── yoga: YogaNode (readonly)
├── 树操作（委托给 yoga）
│   ├── insertChild → yoga.insertChild
│   ├── removeChild → yoga.removeChild
│   ├── getChildCount → yoga.getChildCount
│   └── getParent → yoga.getParent (包装为 YogaLayoutNode)
├── 布局计算（委托给 yoga）
│   ├── calculateLayout → yoga.calculateLayout (固定 LTR)
│   ├── setMeasureFunc → 适配后委托
│   ├── unsetMeasureFunc → yoga.unsetMeasureFunc
│   └── markDirty → yoga.markDirty
├── 布局读取（委托给 yoga）
│   ├── getComputedLeft/Top/Width/Height
│   └── getComputedBorder/Padding (通过 EDGE_MAP 转换)
├── 样式设置器（映射 + 委托）
│   ├── 尺寸设置（直接委托）
│   ├── Flex 设置（通过映射表）
│   ├── 对齐设置（通过映射表）
│   ├── 定位设置（通过 EDGE_MAP）
│   └── 间距设置（通过 EDGE_MAP/GUTTER_MAP）
└── 生命周期（委托给 yoga）
    ├── free → yoga.free
    └── freeRecursive → yoga.freeRecursive
```

### 枚举映射实现

**Edge/Gutter 映射**（使用预定义映射表）：
```typescript
const EDGE_MAP: Record<LayoutEdge, Edge> = { /* ... */ }
const GUTTER_MAP: Record<LayoutGutter, Gutter> = { /* ... */ }

setMargin(edge: LayoutEdge, value: number): void {
  this.yoga.setMargin(EDGE_MAP[edge]!, value)
}
```

**其他枚举映射**（使用内联 Record）：
```typescript
setFlexDirection(dir: LayoutFlexDirection): void {
  const map: Record<LayoutFlexDirection, FlexDirection> = {
    row: FlexDirection.Row,
    'row-reverse': FlexDirection.RowReverse,
    // ...
  }
  this.yoga.setFlexDirection(map[dir]!)
}
```

### 父节点包装

Yoga 返回原始节点，需要包装为 `YogaLayoutNode`：
```typescript
getParent(): LayoutNode | null {
  const p = this.yoga.getParent()
  return p ? new YogaLayoutNode(p) : null
}
```

**注意**：这会创建新的包装器实例，不是单例。如果频繁调用可能影响性能。

### 布局方向固定

```typescript
calculateLayout(width?: number, _height?: number): void {
  this.yoga.calculateLayout(width, undefined, Direction.LTR)
}
```

- 固定使用 LTR（从左到右）方向
- 高度参数被忽略（Yoga 内部计算）

## 关键代码路径与文件引用

### 被引用方

| 文件 | 引用内容 | 用途 |
|------|----------|------|
| `src/ink/layout/engine.ts` | `createYogaLayoutNode` | 工厂函数创建布局节点 |
| `src/ink/layout/node.ts` | 实现 `LayoutNode` 接口 | 类型契约 |

### 引用详情

**`src/ink/layout/engine.ts`** (第 2, 5 行):
```typescript
import { createYogaLayoutNode } from './yoga.js'
// ...
return createYogaLayoutNode()
```

### 依赖的 Yoga 模块

**`src/native-ts/yoga-layout/index.js`**:
```typescript
import Yoga, {
  Align, Direction, Display, Edge, FlexDirection, Gutter,
  Justify, MeasureMode, Overflow, PositionType, Wrap,
  type Node as YogaNode,
} from 'src/native-ts/yoga-layout/index.js'
```

这是项目内嵌的**纯 TypeScript Yoga 实现**，不是外部 npm 包。

## 依赖与外部交互

### 导入依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/native-ts/yoga-layout/index.js` | Yoga 默认导出和所有枚举 | 底层布局引擎 |
| `./node.js` | `LayoutNode` 接口和所有类型 | 实现契约 |

### 与纯 TS Yoga 的关系

```
src/native-ts/yoga-layout/index.js (纯 TS Yoga 实现)
    ↑ 导入
yoga.ts (适配器)
    ↑ 创建
engine.ts (工厂)
    ↑ 使用
Ink DOM/渲染系统
```

**关键特性**：
- 使用纯 TypeScript 实现的 Yoga（`native-ts/yoga-layout`）
- 无需 WASM 加载，同步可用
- 无线性内存管理复杂性

### Yoga 实例管理

```typescript
// 注释说明：TS yoga-layout 是同步的，不需要预加载/交换/重置机制
export function createYogaLayoutNode(): LayoutNode {
  return new YogaLayoutNode(Yoga.Node.create())
}
```

## 风险、边界与改进建议

### 风险点

1. **类型断言风险**：
   ```typescript
   insertChild(child: LayoutNode, index: number): void {
     this.yoga.insertChild((child as YogaLayoutNode).yoga, index)
   }
   ```
   如果传入非 `YogaLayoutNode` 实现会崩溃。

2. **父节点包装器重复创建**：`getParent()` 每次调用都创建新包装器，可能导致内存抖动。

3. **硬编码 LTR**：不支持 RTL（从右到左）布局，对于阿拉伯语/希伯来语场景有限制。

4. **非空断言滥用**：多处使用 `!` 操作符：
   ```typescript
   this.yoga.setMargin(EDGE_MAP[edge]!, value)
   ```
   如果 `LayoutEdge` 新增值而映射表未更新，会运行时错误。

### 边界情况

1. **空值处理**：`calculateLayout` 中 `undefined` 转换为 `NaN` 由 Yoga 内部处理
2. **百分比值**：字符串百分比（如 `"50%"`）在 `styles.ts` 中解析为数字后传入
3. **测量模式转换**：Yoga 的 `MeasureMode` 与 Ink 的 `LayoutMeasureMode` 单向映射

### 改进建议

1. **类型安全检查**：
   ```typescript
   private static assertYogaNode(node: LayoutNode): YogaLayoutNode {
     if (!(node instanceof YogaLayoutNode)) {
       throw new TypeError('Expected YogaLayoutNode instance')
     }
     return node
   }
   ```

2. **父节点缓存**：
   ```typescript
   private parentCache: LayoutNode | null | undefined = undefined
   
   getParent(): LayoutNode | null {
     if (this.parentCache === undefined) {
       const p = this.yoga.getParent()
       this.parentCache = p ? new YogaLayoutNode(p) : null
     }
     return this.parentCache
   }
   ```

3. **映射表验证**：
   ```typescript
   // 在模块加载时验证映射完整性
   const ALL_LAYOUT_EDGES: LayoutEdge[] = ['all', 'horizontal', /* ... */]
   for (const edge of ALL_LAYOUT_EDGES) {
     if (!(edge in EDGE_MAP)) {
       throw new Error(`Missing EDGE_MAP entry for: ${edge}`)
     }
   }
   ```

4. **RTL 支持**：
   ```typescript
   // 添加方向参数（如果需要）
   calculateLayout(width?: number, height?: number, direction: Direction = Direction.LTR): void {
     this.yoga.calculateLayout(width, height, direction)
   }
   ```

5. **性能优化 - 批量样式设置**：
   ```typescript
   // 当前每个样式设置都立即调用 yoga.markDirty()
   // 可以考虑批量设置模式
   batchStyles(fn: (node: this) => void): void {
     // 暂停 dirty 传播，批量设置后统一标记
   }
   ```

6. **内存泄漏防护**：
   ```typescript
   // 添加弱引用跟踪，帮助检测未释放的节点
   private static nodeCount = 0
   constructor() {
     YogaLayoutNode.nodeCount++
     if (process.env.NODE_ENV === 'development') {
       console.log(`YogaLayoutNode created, total: ${YogaLayoutNode.nodeCount}`)
     }
   }
   free(): void {
     YogaLayoutNode.nodeCount--
     this.yoga.free()
   }
   ```

7. **文档完善**：为每个方法添加 JSDoc，特别是 Yoga 特定行为
   ```typescript
   /**
    * 计算布局。注意：高度参数被忽略，Yoga 根据内容和约束自动计算。
    * 固定使用 LTR 方向。
    */
   calculateLayout(width?: number, _height?: number): void
   ```
