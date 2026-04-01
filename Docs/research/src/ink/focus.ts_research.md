# focus.ts 研究文档

## 场景与职责

`focus.ts` 实现 Ink 的焦点管理系统，提供类似浏览器 DOM 的焦点行为。它管理焦点状态、焦点栈，并处理焦点事件的分发。

### 核心职责

1. **焦点状态管理**：跟踪当前焦点元素（activeElement）
2. **焦点栈维护**：保存焦点历史，支持焦点恢复
3. **焦点导航**：支持 Tab 键向前/向后移动焦点
4. **节点移除处理**：被聚焦节点移除时自动恢复焦点
5. **事件分发**：分发 focus/blur 事件

### 使用场景

- 表单输入框的焦点管理
- 按钮和可交互元素的焦点状态
- 键盘导航支持
- 焦点丢失后的自动恢复

## 功能点目的

### 1. FocusManager 类

纯状态管理器，不直接引用 DOM 树：
- `activeElement`：当前焦点元素
- `focusStack`：焦点历史栈（最大 32 个）
- `enabled`：焦点管理是否启用

### 2. 焦点栈机制

**栈操作**：
- 新焦点：旧焦点压栈
- 去重：压栈前移除栈中已有实例，防止 Tab 循环导致无限增长
- 溢出：超过 32 个时移除最旧的

**焦点恢复**：
- 当前焦点被移除时，从栈中弹出最近的仍挂载的元素
- 如果栈空，焦点设为 null

### 3. Tab 导航

**collectTabbable**：
- 从根节点 DFS 遍历
- 收集 `tabIndex >= 0` 的元素

**moveFocus**：
- 支持向前（Tab）和向后（Shift+Tab）导航
- 循环：到达末尾时回到开头

### 4. 节点移除处理

**handleNodeRemoved**：
1. 从焦点栈移除被删除节点及其子树中的节点
2. 检查 activeElement 是否在被删除的子树中
3. 分发 blur 事件
4. 从栈中恢复焦点

## 具体技术实现

### FocusManager 类

```typescript
export class FocusManager {
  activeElement: DOMElement | null = null
  private dispatchFocusEvent: (target: DOMElement, event: FocusEvent) => boolean
  private enabled = true
  private focusStack: DOMElement[] = []
  private readonly MAX_FOCUS_STACK = 32

  constructor(
    dispatchFocusEvent: (target: DOMElement, event: FocusEvent) => boolean,
  ) {
    this.dispatchFocusEvent = dispatchFocusEvent
  }

  focus(node: DOMElement): void {
    if (node === this.activeElement) return
    if (!this.enabled) return

    const previous = this.activeElement
    if (previous) {
      // 去重：防止 Tab 循环导致无限增长
      const idx = this.focusStack.indexOf(previous)
      if (idx !== -1) this.focusStack.splice(idx, 1)
      this.focusStack.push(previous)
      if (this.focusStack.length > MAX_FOCUS_STACK) this.focusStack.shift()
      
      this.dispatchFocusEvent(previous, new FocusEvent('blur', node))
    }
    
    this.activeElement = node
    this.dispatchFocusEvent(node, new FocusEvent('focus', previous))
  }

  blur(): void {
    if (!this.activeElement) return
    const previous = this.activeElement
    this.activeElement = null
    this.dispatchFocusEvent(previous, new FocusEvent('blur', null))
  }
}
```

### 焦点导航

```typescript
private moveFocus(direction: 1 | -1, root: DOMElement): void {
  if (!this.enabled) return

  const tabbable = collectTabbable(root)
  if (tabbable.length === 0) return

  const currentIndex = this.activeElement
    ? tabbable.indexOf(this.activeElement)
    : -1

  const nextIndex =
    currentIndex === -1
      ? direction === 1 ? 0 : tabbable.length - 1
      : (currentIndex + direction + tabbable.length) % tabbable.length

  const next = tabbable[nextIndex]
  if (next) this.focus(next)
}
```

### 节点移除处理

```typescript
handleNodeRemoved(node: DOMElement, root: DOMElement): void {
  // 1. 从焦点栈移除被删除节点及其子树中的节点
  this.focusStack = this.focusStack.filter(
    n => n !== node && isInTree(n, root),
  )

  // 2. 检查 activeElement 是否在被删除的子树中
  if (!this.activeElement) return
  if (this.activeElement !== node && isInTree(this.activeElement, root)) {
    return
  }

  // 3. 分发 blur 事件
  const removed = this.activeElement
  this.activeElement = null
  this.dispatchFocusEvent(removed, new FocusEvent('blur', null))

  // 4. 从栈中恢复焦点
  while (this.focusStack.length > 0) {
    const candidate = this.focusStack.pop()!
    if (isInTree(candidate, root)) {
      this.activeElement = candidate
      this.dispatchFocusEvent(candidate, new FocusEvent('focus', removed))
      return
    }
  }
}
```

### 辅助函数

```typescript
// 收集可聚焦元素
function collectTabbable(root: DOMElement): DOMElement[] {
  const result: DOMElement[] = []
  walkTree(root, result)
  return result
}

function walkTree(node: DOMElement, result: DOMElement[]): void {
  const tabIndex = node.attributes['tabIndex']
  if (typeof tabIndex === 'number' && tabIndex >= 0) {
    result.push(node)
  }
  for (const child of node.childNodes) {
    if (child.nodeName !== '#text') walkTree(child, result)
  }
}

// 检查节点是否在树中
function isInTree(node: DOMElement, root: DOMElement): boolean {
  let current: DOMElement | undefined = node
  while (current) {
    if (current === root) return true
    current = current.parentNode
  }
  return false
}
```

### 根节点访问

```typescript
// 获取根节点（持有 FocusManager 的节点）
export function getRootNode(node: DOMElement): DOMElement {
  let current: DOMElement | undefined = node
  while (current) {
    if (current.focusManager) return current
    current = current.parentNode
  }
  throw new Error('Node is not in a tree with a FocusManager')
}

// 获取 FocusManager
export function getFocusManager(node: DOMElement): FocusManager {
  return getRootNode(node).focusManager!
}
```

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/focus.ts`
- **导出类**：`FocusManager`
- **导出函数**：`getRootNode`、`getFocusManager`

### 依赖关系

**被导入**：
- `./dom.js` - DOMElement 类型
- `./events/focus-event.js` - FocusEvent

**导入使用**：
```typescript
import type { DOMElement } from './dom.js'
import { FocusEvent } from './events/focus-event.js'
```

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `FocusManager.focus` | 设置焦点 | 27-42 |
| `FocusManager.blur` | 移除焦点 | 44-50 |
| `FocusManager.handleNodeRemoved` | 处理节点移除 | 57-82 |
| `FocusManager.handleAutoFocus` | 自动聚焦 | 84-86 |
| `FocusManager.handleClickFocus` | 点击聚焦 | 88-92 |
| `FocusManager.moveFocus` | 焦点导航 | 110-131 |
| `collectTabbable` | 收集可聚焦元素 | 134-138 |
| `walkTree` | DFS 遍历 | 140-151 |
| `isInTree` | 检查节点归属 | 153-160 |
| `getRootNode` | 获取根节点 | 166-173 |
| `getFocusManager` | 获取 FocusManager | 179-181 |

### 相关文件

- `src/ink/dom.ts` - DOMElement 定义，focusManager 属性
- `src/ink/events/focus-event.ts` - FocusEvent 定义
- `src/ink/events/dispatcher.ts` - 事件分发器
- `src/ink/reconciler.ts` - 调用 handleNodeRemoved

## 依赖与外部交互

### 与 DOM 的交互

```
DOMElement (ink-root)
    ↓
focusManager: FocusManager
    ↓
通过 parentNode 链访问
```

### 与 Reconciler 的交互

```
Reconciler 移除节点
    ↓
focusManager.handleNodeRemoved()
    ↓
更新焦点状态
    ↓
分发 blur/focus 事件
```

### 与事件系统的交互

```
键盘事件（Tab/Shift+Tab）
    ↓
focusManager.focusNext/Previous()
    ↓
collectTabbable() 获取候选
    ↓
focus() 设置新焦点
    ↓
dispatchFocusEvent() 分发事件
```

## 风险、边界与改进建议

### 已知风险

1. **栈大小限制**：
   - 最大 32 个，可能丢失更早的焦点历史
   - 复杂应用可能需要更大的栈

2. **isInTree 遍历**：
   - 每次检查都需要遍历到根
   - 深层树可能有性能影响

3. **焦点恢复延迟**：
   - 节点移除后异步恢复焦点
   - 可能导致短暂的"无焦点"状态

4. **tabIndex 处理**：
   - 仅支持数字类型
   - 不支持字符串或其他值

### 边界情况

1. **焦点管理禁用**：`enabled = false` 时所有焦点操作无效果
2. **重复聚焦同一元素**：直接返回，不触发事件
3. **空可聚焦列表**：`collectTabbable` 返回空数组时导航无效果
4. **根节点移除**：如果根节点被移除，getRootNode 抛出错误

### 改进建议

1. **性能优化**：
   - 缓存可聚焦元素列表，避免每次遍历
   - 使用 MutationObserver 模式监听变化
   
   ```typescript
   // 示例：缓存机制
   private tabbableCache: DOMElement[] | null = null
   invalidateCache() { this.tabbableCache = null }
   ```

2. **功能扩展**：
   - 支持 `tabIndex="-1"` 的可编程聚焦
   - 实现 `focusin`/`focusout` 事件（冒泡版本）
   - 添加 `focusVisible` 状态（区分鼠标和键盘聚焦）

3. **API 改进**：
   - 添加 `focusFirst()`、`focusLast()` 快捷方法
   - 支持焦点区域（FocusScope）概念
   - 添加焦点历史导航（Alt+Tab 风格）

4. **错误处理**：
   - 更优雅地处理根节点移除
   - 添加焦点循环检测和警告

5. **测试覆盖**：
   - 焦点栈溢出的边界测试
   - 复杂树结构的导航测试
   - 节点移除和焦点恢复的时机测试
