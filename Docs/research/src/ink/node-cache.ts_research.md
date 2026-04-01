# Research: src/ink/node-cache.ts

## 场景与职责

`node-cache.ts` 维护 Ink 渲染树中各节点的**布局缓存**与**清除状态**。在每一帧渲染中，`render-node-to-output.ts` 会将每个 `DOMElement` 的 Yoga 计算边界（x, y, width, height, top）写入 `nodeCache`；后续帧若节点未变（clean subtree），可直接复用这些边界进行 `blit`（块拷贝），避免重新遍历子树。同时，当子节点被移除或尺寸缩小时，需要记录其旧边界以便在下一帧清除残留像素，`pendingClears` 即用于此目的。

## 功能点目的

1. **布局缓存（`nodeCache`）**：存储每个节点的屏幕坐标矩形，供 blit、hit-test、清除残留使用。
2. **待清除区域（`pendingClears`）**：记录被移除子节点的旧边界，下一帧渲染时在这些区域写空白单元格。
3. **绝对定位移除标记（`absoluteNodeRemoved`）**：当绝对定位节点被移除时，设置全局标志，通知渲染器禁用 blit 优化一帧。因为绝对节点可能覆盖非兄弟子树，blit 会从旧屏幕拷贝回被覆盖的内容，导致"幽灵"残影。

## 具体技术实现

```ts
export type CachedLayout = {
  x: number
  y: number
  width: number
  height: number
  top?: number  // yoga-local getComputedTop，用于 ScrollBox 视口裁剪优化
}

export const nodeCache = new WeakMap<DOMElement, CachedLayout>()
export const pendingClears = new WeakMap<DOMElement, Rectangle[]>()

let absoluteNodeRemoved = false

export function addPendingClear(parent, rect, isAbsolute): void {
  const existing = pendingClears.get(parent)
  if (existing) existing.push(rect)
  else pendingClears.set(parent, [rect])
  if (isAbsolute) absoluteNodeRemoved = true
}

export function consumeAbsoluteRemovedFlag(): boolean {
  const had = absoluteNodeRemoved
  absoluteNodeRemoved = false
  return had
}
```

- **数据结构**：
  - `WeakMap<DOMElement, CachedLayout>`：键为 DOM 节点，值为布局矩形。使用 WeakMap 避免阻止节点被垃圾回收。
  - `WeakMap<DOMElement, Rectangle[]>`：键为父节点，值为该父节点下所有待清除的子区域列表。
- **状态管理**：`absoluteNodeRemoved` 是模块级布尔标志，每帧由 `renderer.ts` 调用 `consumeAbsoluteRemovedFlag()` 读取并复位。

## 关键代码路径与文件引用

- **写入 `nodeCache`**：
  - `src/ink/render-node-to-output.ts` — 在渲染每个节点时调用 `nodeCache.set(node, { x, y, width, height, top })`。
- **读取 `nodeCache`**：
  - `src/ink/hit-test.ts:23` — `hitTest()` 用 `nodeCache.get(node)` 做点击测试。
  - `src/ink/renderer.ts` — 在判断 clean subtree 是否可 blit 时读取缓存。
  - `src/ink/render-node-to-output.ts` — 读取旧缓存对比以决定是否 layout shifted。
- **`pendingClears` 写入**：
  - `src/ink/dom.ts` — `removeChildNode` 或节点缩小时调用 `addPendingClear()`。
- **`pendingClears` 读取**：
  - `src/ink/render-node-to-output.ts` — 在渲染父节点时读取并执行清除。
- **`consumeAbsoluteRemovedFlag` 调用**：
  - `src/ink/renderer.ts:4` — 每帧渲染开始时检查，若曾为 true 则禁用根级 blit。
- **Ink 主类使用**：
  - `src/ink/ink.tsx:25` — `import { nodeCache } from './node-cache.js'`（用于 `findOwnerChainAtRow` 等调试功能）。

## 依赖与外部交互

- 依赖 `./dom.js` 的 `DOMElement` 类型和 `./layout/geometry.js` 的 `Rectangle` 类型。
- 无外部 I/O，纯内存状态管理。

## 风险、边界与改进建议

- **风险**：`WeakMap` 的键是对象引用，若测试代码或热重载场景下频繁替换 DOM 节点对象，缓存会失效，导致 blit 优化率下降。
- **边界**：
  - `pendingClears` 只记录矩形，不记录被移除节点的样式；清除时统一写空白单元格（`styleId = none`）。若被移除节点带有背景色，清除后背景会消失，这是预期行为。
  - `absoluteNodeRemoved` 是全局单标志，若同一帧有多个绝对节点被移除，只需设置一次 true；但无法区分是哪些节点，因此渲染器会保守地禁用整棵树的 blit。
- **改进建议**：
  1. 可考虑将 `nodeCache` 的 `top` 字段统一纳入，使 ScrollBox 的视口裁剪逻辑更一致。
  2. `pendingClears` 目前只收集矩形，未来若需要支持渐变或复杂背景清除，可能需要存储更多上下文。
  3. 对 `absoluteNodeRemoved` 的保守策略，可优化为只标记受影响的子树范围（如收集被覆盖节点的最小公共祖先），减少不必要的全树重绘。
  4. 增加调试模式统计：记录每帧 `nodeCache` 命中/未命中率和 `pendingClears` 数量，帮助分析 blit 效率。
