# focus-event.ts 深度研究文档

## 场景与职责

`FocusEvent` 是 Ink 中处理焦点变化的事件类，用于：

1. **组件焦点管理**：当焦点在可聚焦元素之间移动时触发
2. **焦点历史追踪**：通过 `relatedTarget` 知道焦点来自/去往哪个元素
3. **焦点状态同步**：与 React 的受控组件模式配合

在终端 UI 中，焦点管理面临特殊挑战：
- 没有原生的 "tabindex" 概念，需要手动实现
- 需要模拟浏览器的焦点环（focus ring）行为
- 焦点变化需要与键盘导航（Tab/Shift+Tab）配合

## 功能点目的

### 1. 焦点事件类型
- `'focus'`：元素获得焦点时触发
- `'blur'`：元素失去焦点时触发

### 2. 相关目标追踪
`relatedTarget` 属性表示：
- 在 `'focus'` 事件中：之前获得焦点的元素（如果有）
- 在 `'blur'` 事件中：即将获得焦点的元素（如果有）

### 3. 冒泡支持
与浏览器不同，Ink 的 FocusEvent 支持冒泡，允许父组件监听子组件的焦点变化。

## 具体技术实现

### 类定义

```typescript
export class FocusEvent extends TerminalEvent {
  readonly relatedTarget: EventTarget | null

  constructor(
    type: 'focus' | 'blur',
    relatedTarget: EventTarget | null = null,
  ) {
    super(type, { bubbles: true, cancelable: false })
    this.relatedTarget = relatedTarget
  }
}
```

### 关键特性

1. **继承 TerminalEvent**：
   - 获得 DOM 风格的事件属性（target、currentTarget、eventPhase）
   - 支持 `stopPropagation()` 和 `preventDefault()`

2. **事件初始化**：
   ```typescript
   super(type, { bubbles: true, cancelable: false })
   ```
   - `bubbles: true`：事件会冒泡，父组件可以监听
   - `cancelable: false`：焦点事件不能被取消（与浏览器行为一致）

3. **相关目标**：
   - 类型为 `EventTarget | null`
   - 可选参数，默认为 `null`

### 使用模式

1. **焦点切换**（`focus.ts:27-42`）:
   ```typescript
   focus(node: DOMElement): void {
     const previous = this.activeElement
     if (previous) {
       this.dispatchFocusEvent(previous, new FocusEvent('blur', node))
     }
     this.activeElement = node
     this.dispatchFocusEvent(node, new FocusEvent('focus', previous))
   }
   ```

2. **焦点移除**（`focus.ts:44-50`）:
   ```typescript
   blur(): void {
     if (!this.activeElement) return
     const previous = this.activeElement
     this.activeElement = null
     this.dispatchFocusEvent(previous, new FocusEvent('blur', null))
   }
   ```

3. **节点移除恢复**（`focus.ts:57-82`）:
   ```typescript
   handleNodeRemoved(node: DOMElement, root: DOMElement): void {
     // ...
     this.dispatchFocusEvent(removed, new FocusEvent('blur', null))
     // ...
     this.dispatchFocusEvent(candidate, new FocusEvent('focus', removed))
   }
   ```

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/focus-event.ts` - FocusEvent 类定义

### 使用位置

1. **focus.ts**（line 2）:
   ```typescript
   import { FocusEvent } from './events/focus-event.js'
   ```
   在 `FocusManager` 中创建和分发焦点事件。

2. **Box.tsx**（line 7, 31-34）:
   ```typescript
   import type { FocusEvent } from '../events/focus-event.js'
   // ...
   onFocus?: (event: FocusEvent) => void
   onBlur?: (event: FocusEvent) => void
   ```
   组件 Props 类型定义。

3. **event-handlers.ts**（line 2, 8, 25-28, 49-50）:
   ```typescript
   import type { FocusEvent } from './focus-event.js'
   type FocusEventHandler = (event: FocusEvent) => void
   // ...
   onFocus?: FocusEventHandler
   onBlur?: FocusEventHandler
   ```

### 调用链
```
FocusManager.focus (focus.ts:27)
  → new FocusEvent('focus', previous) (focus-event.ts:14)
  → dispatchDiscrete (dispatcher.ts:207)
    → dispatch (dispatcher.ts:185)
      → collectListeners (dispatcher.ts:46)
      → processDispatchQueue (dispatcher.ts:87)
        → handler(event)  // onFocus/onBlur
```

## 依赖与外部交互

### 依赖

- `terminal-event.ts` - `TerminalEvent` 基类
- `terminal-event.ts` - `EventTarget` 类型

### 被依赖

- `focus.ts` - `FocusManager` 使用
- `event-handlers.ts` - 类型定义
- `Box.tsx` - 组件 Props 类型
- `ink.ts` - 导出（通过 focus.ts 间接）

### 与 FocusManager 的协作

```typescript
// focus.ts
export class FocusManager {
  activeElement: DOMElement | null = null
  private dispatchFocusEvent: (target: DOMElement, event: FocusEvent) => boolean

  constructor(dispatchFocusEvent: ...) {
    this.dispatchFocusEvent = dispatchFocusEvent
  }

  focus(node: DOMElement): void {
    // ...
    this.dispatchFocusEvent(node, new FocusEvent('focus', previous))
  }
}
```

`FocusManager` 负责：
1. 维护当前焦点元素 (`activeElement`)
2. 管理焦点历史栈 (`focusStack`)
3. 在适当时机创建和分发 `FocusEvent`

## 风险、边界与改进建议

### 风险点

1. **循环焦点**:
   - 如果两个组件互相设置对方为焦点，可能导致无限循环
   - `FocusManager.focus` 有检查 `if (node === this.activeElement) return`，防止简单循环

2. **节点移除时的焦点**:
   - 如果焦点元素被移除，需要从栈中恢复焦点
   - `handleNodeRemoved` 处理这种情况，但可能有竞态条件

3. **冒泡副作用**:
   - 焦点事件冒泡可能导致父组件意外处理
   - 需要使用 `stopPropagation()` 控制

### 边界情况

1. **relatedTarget 为 null**:
   - 首次聚焦或完全失去焦点时
   - 处理器需要处理这种情况

2. **同一元素多次聚焦**:
   ```typescript
   // 以下代码不会触发事件
   focusManager.focus(nodeA)
   focusManager.focus(nodeA)  // 无操作
   ```

3. **捕获阶段**:
   - 当前 `FocusEvent` 支持捕获阶段处理器
   - 但实际使用较少

4. **焦点与选择**:
   - 终端中焦点和文本选择是独立的
   - 需要确保两者不冲突

### 改进建议

1. **添加焦点原因**:
   ```typescript
   type FocusReason = 'click' | 'tab' | 'script' | 'restore'
   
   constructor(
     type: 'focus' | 'blur',
     relatedTarget: EventTarget | null = null,
     reason?: FocusReason
   )
   ```

2. **焦点委托**:
   ```typescript
   // 支持焦点委托模式
   readonly delegateTarget?: EventTarget
   ```

3. **焦点路径**:
   ```typescript
   // 记录焦点变化的完整路径
   readonly path: EventTarget[]
   ```

4. **异步焦点**:
   ```typescript
   // 支持异步焦点确认
   waitForFocus(): Promise<void>
   ```

5. **焦点可见性**:
   ```typescript
   // 焦点元素是否在可视区域内
   readonly isVisible: boolean
   ```

### 测试建议

1. 测试焦点切换的顺序（blur 先于 focus）
2. 测试 relatedTarget 的正确性
3. 测试节点移除时的焦点恢复
4. 测试焦点栈溢出（MAX_FOCUS_STACK = 32）
5. 测试 Tab 循环导航
6. 测试捕获和冒泡阶段的处理器调用顺序
