# hit-test.ts 研究文档

## 场景与职责

`hit-test.ts` 实现 Ink 的鼠标点击检测和事件分发系统。它提供类似浏览器 DOM 的点击测试（hit-testing）和事件冒泡机制，支持鼠标交互的终端 UI。

### 核心职责

1. **点击测试（Hit Testing）**：找出包含指定屏幕坐标的 DOM 元素
2. **点击事件分发**：从最深命中节点向上冒泡分发 ClickEvent
3. **悬停事件管理**：跟踪鼠标移动，分发 mouseenter/mouseleave 事件
4. **点击聚焦**：自动将点击的元素设为焦点

### 使用场景

- 按钮点击交互
- 可点击列表项
- 超链接点击
- 鼠标悬停效果
- 拖拽选择的起始点检测

## 功能点目的

### 1. 点击测试（hitTest）

使用 `nodeCache` 中缓存的渲染矩形进行快速命中检测：
- 从根节点开始递归检测
- 使用屏幕坐标（已包含 scrollTop 偏移）
- 子节点反向遍历（后渲染的在顶层）
- 返回最深的命中节点

### 2. 点击事件分发（dispatchClick）

完整的点击事件处理流程：
1. 执行 hitTest 找到目标节点
2. 点击聚焦：找到最近的 focusable 祖先并聚焦
3. 创建 ClickEvent
4. 从目标节点向上冒泡
5. 每个有 onClick 的节点触发回调
6. 支持 `stopImmediatePropagation()` 停止冒泡

### 3. 悬停事件分发（dispatchHover）

跟踪鼠标移动，分发 enter/leave 事件：
1. 收集当前命中的所有有 hover 处理的节点
2. 与上一帧的悬停集合比较
3. 对离开的节点分发 mouseleave
4. 对新进入的节点分发 mouseenter
5. 更新悬停集合

### 4. 本地坐标计算

ClickEvent 包含相对于当前处理节点的本地坐标：
- `localCol = col - rect.x`
- `localRow = row - rect.y`
- 每个处理节点看到的坐标是相对于自身的

## 具体技术实现

### 点击测试

```typescript
export function hitTest(
  node: DOMElement,
  col: number,
  row: number,
): DOMElement | null {
  // 1. 获取缓存的渲染矩形
  const rect = nodeCache.get(node)
  if (!rect) return null
  
  // 2. 检查坐标是否在矩形内
  if (
    col < rect.x ||
    col >= rect.x + rect.width ||
    row < rect.y ||
    row >= rect.y + rect.height
  ) {
    return null
  }
  
  // 3. 反向遍历子节点（后渲染的在顶层）
  for (let i = node.childNodes.length - 1; i >= 0; i--) {
    const child = node.childNodes[i]!
    if (child.nodeName === '#text') continue
    const hit = hitTest(child, col, row)
    if (hit) return hit
  }
  
  // 4. 没有子节点命中，返回当前节点
  return node
}
```

### 点击事件分发

```typescript
export function dispatchClick(
  root: DOMElement,
  col: number,
  row: number,
  cellIsBlank = false,
): boolean {
  // 1. 执行 hit test
  let target: DOMElement | undefined = hitTest(root, col, row) ?? undefined
  if (!target) return false
  
  // 2. 点击聚焦
  if (root.focusManager) {
    let focusTarget: DOMElement | undefined = target
    while (focusTarget) {
      if (typeof focusTarget.attributes['tabIndex'] === 'number') {
        root.focusManager.handleClickFocus(focusTarget)
        break
      }
      focusTarget = focusTarget.parentNode
    }
  }
  
  // 3. 创建事件并冒泡
  const event = new ClickEvent(col, row, cellIsBlank)
  let handled = false
  while (target) {
    const handler = target._eventHandlers?.onClick as
      | ((event: ClickEvent) => void)
      | undefined
    if (handler) {
      handled = true
      // 计算本地坐标
      const rect = nodeCache.get(target)
      if (rect) {
        event.localCol = col - rect.x
        event.localRow = row - rect.y
      }
      handler(event)
      if (event.didStopImmediatePropagation()) return true
    }
    target = target.parentNode
  }
  return handled
}
```

### 悬停事件分发

```typescript
export function dispatchHover(
  root: DOMElement,
  col: number,
  row: number,
  hovered: Set<DOMElement>,
): void {
  // 1. 收集当前命中的 hoverable 节点
  const next = new Set<DOMElement>()
  let node: DOMElement | undefined = hitTest(root, col, row) ?? undefined
  while (node) {
    const h = node._eventHandlers as EventHandlerProps | undefined
    if (h?.onMouseEnter || h?.onMouseLeave) next.add(node)
    node = node.parentNode
  }
  
  // 2. 分发 mouseleave（离开的节点）
  for (const old of hovered) {
    if (!next.has(old)) {
      hovered.delete(old)
      // 跳过已卸载的节点
      if (old.parentNode) {
        ;(old._eventHandlers as EventHandlerProps | undefined)?.onMouseLeave?.()
      }
    }
  }
  
  // 3. 分发 mouseenter（新进入的节点）
  for (const n of next) {
    if (!hovered.has(n)) {
      hovered.add(n)
      ;(n._eventHandlers as EventHandlerProps | undefined)?.onMouseEnter?.()
    }
  }
}
```

### ClickEvent 结构

```typescript
export class ClickEvent extends Event {
  readonly col: number           // 屏幕列（0-indexed）
  readonly row: number           // 屏幕行（0-indexed）
  localCol = 0                   // 相对于处理节点的列
  localRow = 0                   // 相对于处理节点的行
  readonly cellIsBlank: boolean  // 点击的单元格是否空白
}
```

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/hit-test.ts`
- **导出函数**：
  - `hitTest` - 点击测试
  - `dispatchClick` - 点击事件分发
  - `dispatchHover` - 悬停事件分发

### 依赖关系

**被导入**：
- `./dom.js` - DOMElement 类型
- `./events/click-event.js` - ClickEvent
- `./events/event-handlers.js` - EventHandlerProps
- `./node-cache.js` - nodeCache

**导入使用**：
```typescript
import type { DOMElement } from './dom.js'
import { ClickEvent } from './events/click-event.js'
import type { EventHandlerProps } from './events/event-handlers.js'
import { nodeCache } from './node-cache.js'
```

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `hitTest` | 点击测试 | 18-41 |
| `dispatchClick` | 点击事件分发 | 49-89 |
| `dispatchHover` | 悬停事件分发 | 102-130 |

### 相关文件

- `src/ink/dom.ts` - DOMElement 定义，_eventHandlers、focusManager
- `src/ink/node-cache.ts` - nodeCache，CachedLayout
- `src/ink/events/click-event.ts` - ClickEvent 定义
- `src/ink/events/event-handlers.ts` - EventHandlerProps
- `src/ink/events/event.ts` - Event 基类（stopImmediatePropagation）
- `src/ink/focus.ts` - FocusManager.handleClickFocus

## 依赖与外部交互

### 与渲染管线的交互

```
渲染到屏幕
    ↓
更新 nodeCache（每个节点的屏幕矩形）
    ↓
鼠标事件到达
    ↓
hitTest() 使用 nodeCache 进行命中检测
    ↓
分发事件到组件
```

### 与事件系统的交互

```
终端鼠标事件
    ↓
解析为 (col, row)
    ↓
dispatchClick / dispatchHover
    ↓
hitTest 找到目标
    ↓
冒泡分发到事件处理器
    ↓
组件回调执行
```

### 与 Focus 系统的交互

```
dispatchClick
    ↓
找到 focusable 祖先
    ↓
focusManager.handleClickFocus()
    ↓
更新 activeElement
    ↓
分发 focus/blur 事件
```

## 风险、边界与改进建议

### 已知风险

1. **nodeCache 同步**：
   - hitTest 依赖 nodeCache 中的渲染矩形
   - 如果缓存未更新，命中检测可能错误

2. **性能问题**：
   - 每次鼠标事件都遍历 DOM 树
   - 深层树可能有性能影响
   - 可考虑空间索引优化

3. **事件顺序**：
   - hover 和 click 事件顺序需要小心处理
   - 快速移动可能导致 enter/leave 不匹配

4. **空白单元格点击**：
   - `cellIsBlank` 参数需要外部提供
   - 依赖于屏幕缓冲区的状态

### 边界情况

1. **无命中**：hitTest 返回 null，dispatchClick 返回 false
2. **无事件处理器**：dispatchClick 返回 false（未处理）
3. **停止冒泡**：handler 调用 `stopImmediatePropagation()`
4. **节点移除**：hover 处理中检查 `old.parentNode`
5. **焦点管理器缺失**：跳过点击聚焦逻辑

### 改进建议

1. **性能优化**：
   - 实现空间索引（如 R-tree 或网格）
   - 缓存上一次的命中结果
   - 批量处理鼠标移动事件
   
   ```typescript
   // 示例：简单缓存
   let lastHit: { col: number; row: number; node: DOMElement | null }
   ```

2. **功能扩展**：
   - 支持右键点击（context menu）
   - 支持双击检测
   - 支持拖拽起始检测
   - 添加点击区域（hit region）API

3. **精度改进**：
   - 考虑字符宽度（CJK/emoji）
   - 支持不规则点击区域

4. **错误处理**：
   - 添加循环引用检测
   - 处理异常的 nodeCache 状态

5. **测试覆盖**：
   - 各种树结构的命中测试
   - 事件冒泡顺序验证
   - 边界坐标（0、最大值）测试
   - 快速鼠标移动场景
