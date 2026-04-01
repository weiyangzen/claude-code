# src/ink/layout/geometry.ts 研究文档

## 场景与职责

`geometry.ts` 是 Ink 布局系统的**几何工具库**，提供基础几何类型定义和计算函数。它独立于布局引擎，为渲染、裁剪、碰撞检测等场景提供通用的几何运算能力。

**核心定位**：
- 提供**纯数学几何抽象**，与布局引擎解耦
- 支持**矩形运算**（并集、裁剪、边界检测）
- 支持**边距运算**（Edges 的创建、合并、解析）
- 作为渲染系统和布局系统的共享基础设施

## 功能点目的

### 1. 基础几何类型

```typescript
export type Point = { x: number; y: number }
export type Size = { width: number; height: number }
export type Rectangle = Point & Size  // { x, y, width, height }
```

**目的**：
- 统一坐标系统表示
- 支持 TypeScript 类型推导和检查
- `Rectangle` 使用交叉类型，兼具位置和尺寸

### 2. 边距类型与操作

```typescript
export type Edges = { top: number; right: number; bottom: number: left: number }
```

**功能**：
- `edges(all)` / `edges(vertical, horizontal)` / `edges(top, right, bottom, left)`：多态边距创建
- `addEdges(a, b)`：边距相加（用于计算总内边距+边框）
- `resolveEdges(partial)`：将部分边距解析为完整边距（默认值 0）
- `ZERO_EDGES`：零边距常量

### 3. 矩形运算

```typescript
unionRect(a, b)      // 两个矩形的并集（最小包围盒）
clampRect(rect, size) // 将矩形裁剪到指定尺寸范围内
withinBounds(size, point) // 检查点是否在尺寸范围内
```

### 4. 数值工具

```typescript
clamp(value, min?, max?)  // 数值限制在范围内
```

## 具体技术实现

### 关键数据结构

```
Point ──┬──► Rectangle
        │
Size ───┘

Edges ──► top/right/bottom/left 数值
```

### 边距函数重载实现

```typescript
// 使用 TypeScript 函数重载实现 CSS-like 边距语法
export function edges(all: number): Edges                           // edges(5)
export function edges(vertical: number, horizontal: number): Edges  // edges(5, 10)
export function edges(t: number, r: number, b: number, l: number): Edges  // edges(1,2,3,4)
export function edges(a: number, b?: number, c?: number, d?: number): Edges {
  if (b === undefined) {
    return { top: a, right: a, bottom: a, left: a }
  }
  if (c === undefined) {
    return { top: a, right: b, bottom: a, left: b }
  }
  return { top: a, right: b, bottom: c, left: d! }
}
```

### 矩形并集算法

```
unionRect(a, b):
  minX = min(a.x, b.x)
  minY = min(a.y, b.y)
  maxX = max(a.x + a.width, b.x + b.width)
  maxY = max(a.y + a.height, b.y + b.height)
  return { x: minX, y: minY, width: maxX - minX, height: maxY - minY }
```

### 矩形裁剪算法

```
clampRect(rect, size):
  minX = max(0, rect.x)
  minY = max(0, rect.y)
  maxX = min(size.width - 1, rect.x + rect.width - 1)
  maxY = min(size.height - 1, rect.y + rect.height - 1)
  return { x: minX, y: minY, width: max(0, maxX - minX + 1), height: max(0, maxY - minY + 1) }
```

## 关键代码路径与文件引用

### 被引用方

| 文件 | 引用内容 | 用途 |
|------|----------|------|
| `src/ink/render-node-to-output.ts` | `Rectangle` | 绝对定位元素的矩形缓存 |
| `src/ink/node-cache.ts` | `Rectangle` | 节点缓存的矩形存储 |

### 引用详情

**`render-node-to-output.ts`** (第 5 行):
```typescript
import type { Rectangle } from './layout/geometry.js'
```

用于记录绝对定位元素在上一帧的位置，用于滚动优化和 blit 修复。

## 依赖与外部交互

### 无外部依赖

该文件是**纯工具模块**：
- 不依赖任何其他 Ink 模块
- 不依赖 Yoga 布局引擎
- 仅使用 TypeScript/JavaScript 原生 API

### 导出清单

| 导出 | 类型 | 用途 |
|------|------|------|
| `Point` | type | 二维坐标 |
| `Size` | type | 尺寸 |
| `Rectangle` | type | 矩形区域 |
| `Edges` | type | 四边边距 |
| `edges` | function | 创建边距（多态） |
| `addEdges` | function | 边距相加 |
| `ZERO_EDGES` | constant | 零边距 |
| `resolveEdges` | function | 解析部分边距 |
| `unionRect` | function | 矩形并集 |
| `clampRect` | function | 矩形裁剪 |
| `withinBounds` | function | 点边界检测 |
| `clamp` | function | 数值裁剪 |

## 风险、边界与改进建议

### 风险点

1. **坐标系混淆**：`Rectangle` 使用 `x/y` 表示左上角，但在终端渲染中 `y` 是行索引，需要确保调用方理解坐标系
2. **溢出问题**：`clampRect` 中 `maxX - minX + 1` 在极端大值时可能溢出（但终端尺寸通常很小，实际风险低）

### 边界情况

1. **零尺寸矩形**：`clampRect` 返回 `{ width: 0, height: 0 }` 当矩形完全在边界外
2. **负尺寸输入**：函数不验证输入，调用方需确保 width/height 非负
3. **NaN 处理**：未处理 NaN 输入，可能传播错误

### 改进建议

1. **添加验证**：
   ```typescript
   function assertNonNegative(n: number, name: string): void {
     if (n < 0 || isNaN(n)) throw new Error(`${name} must be non-negative`)
   }
   ```

2. **不可变类型**：考虑使用 `readonly` 增强类型安全
   ```typescript
   export type Point = { readonly x: number; readonly y: number }
   ```

3. **更多几何运算**：
   - `intersectRect`：矩形交集（用于碰撞检测）
   - `containsRect`：包含检测
   - `inflateRect`：矩形膨胀/收缩

4. **与 Yoga 集成**：考虑提供从 Yoga 计算值到 `Rectangle` 的转换辅助函数
   ```typescript
   export function fromYogaNode(node: LayoutNode): Rectangle {
     return {
       x: node.getComputedLeft(),
       y: node.getComputedTop(),
       width: node.getComputedWidth(),
       height: node.getComputedHeight()
     }
   }
   ```

5. **单元测试**：这些纯函数非常适合单元测试，建议添加：
   - 边距多态组合测试
   - 矩形并集边界情况
   - 裁剪的边界条件
