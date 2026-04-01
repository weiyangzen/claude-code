# terminal-event.ts 深度研究文档

## 场景与职责

`TerminalEvent` 是 Ink 中所有 DOM 风格终端事件的基类，提供完整的事件传播和状态管理机制。它是 `Event` 类的扩展，增加了浏览器 Event API 的核心功能。

设计目标：
1. **DOM 兼容性**：模拟浏览器 Event API，降低学习成本
2. **传播控制**：支持捕获、冒泡、停止传播
3. **默认行为控制**：支持 preventDefault
4. **可扩展性**：为子类提供钩子（`_prepareForTarget`）

## 功能点目的

### 1. 事件基础属性
- `type`: 事件类型字符串（如 'keydown', 'click'）
- `timeStamp`: 事件创建时间（使用 `performance.now()`）
- `bubbles`: 是否冒泡
- `cancelable`: 是否可以取消默认行为

### 2. 事件目标追踪
- `target`: 事件目标节点（分发时设置）
- `currentTarget`: 当前处理节点（分发过程中变化）
- `eventPhase`: 当前阶段（'none' | 'capturing' | 'at_target' | 'bubbling'）

### 3. 传播控制
- `stopPropagation()`: 阻止事件继续传播
- `stopImmediatePropagation()`: 阻止当前节点其他监听器（继承自 Event）
- `preventDefault()`: 标记取消默认行为

### 4. 子类扩展钩子
`_prepareForTarget(target: EventTarget): void` - 子类可在每个处理器前执行设置

## 具体技术实现

### 类定义

```typescript
export class TerminalEvent extends Event {
  readonly type: string
  readonly timeStamp: number
  readonly bubbles: boolean
  readonly cancelable: boolean

  private _target: EventTarget | null = null
  private _currentTarget: EventTarget | null = null
  private _eventPhase: EventPhase = 'none'
  private _propagationStopped = false
  private _defaultPrevented = false

  constructor(type: string, init?: TerminalEventInit) {
    super()
    this.type = type
    this.timeStamp = performance.now()
    this.bubbles = init?.bubbles ?? true
    this.cancelable = init?.cancelable ?? true
  }
}
```

### 事件阶段

```typescript
type EventPhase = 'none' | 'capturing' | 'at_target' | 'bubbling'
```

阶段转换（由 Dispatcher 控制）：
```
'none' → 'capturing' → 'at_target' → 'bubbling' → 'none'
```

### 传播控制实现

```typescript
stopPropagation(): void {
  this._propagationStopped = true
}

override stopImmediatePropagation(): void {
  super.stopImmediatePropagation()  // 调用 Event 的基类方法
  this._propagationStopped = true
}
```

注意：`stopImmediatePropagation` 同时设置了：
- `Event._didStopImmediatePropagation`（供 EventEmitter 检查）
- `TerminalEvent._propagationStopped`（供 Dispatcher 检查）

### 默认行为控制

```typescript
preventDefault(): void {
  if (this.cancelable) {
    this._defaultPrevented = true
  }
}
```

`defaultPrevented` 是只读属性，Dispatcher 通过返回值判断是否被取消：
```typescript
dispatch(target: EventTarget, event: TerminalEvent): boolean {
  // ...
  return !event.defaultPrevented
}
```

### 内部方法（@internal）

这些方法由 Dispatcher 调用，不应对外暴露：

```typescript
_setTarget(target: EventTarget): void
_setCurrentTarget(target: EventTarget | null): void
_setEventPhase(phase: EventPhase): void
_isPropagationStopped(): boolean
_isImmediatePropagationStopped(): boolean
```

### EventTarget 类型

```typescript
export type EventTarget = {
  parentNode: EventTarget | undefined
  _eventHandlers?: Record<string, unknown>
}
```

这是 Ink DOM 元素的简化接口，与浏览器 EventTarget 不同。

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/terminal-event.ts` - TerminalEvent 类定义

### 使用位置

1. **Dispatcher**（`dispatcher.ts:9, 13, 19, 46, 87, 185`）:
   ```typescript
   import type { EventTarget, TerminalEvent } from './terminal-event.js'
   // 使用 TerminalEvent 作为所有事件的基类型
   ```

2. **ClickEvent**（`click-event.ts:10`）:
   ```typescript
   export class ClickEvent extends Event  // 注意：直接继承 Event，不是 TerminalEvent
   ```
   ClickEvent 直接继承 Event，因为它不需要 DOM 风格的传播。

3. **FocusEvent**（`focus-event.ts:11`）:
   ```typescript
   export class FocusEvent extends TerminalEvent
   ```

4. **KeyboardEvent**（`keyboard-event.ts:12`）:
   ```typescript
   export class KeyboardEvent extends TerminalEvent
   ```

### 继承层次

```
Event (event.ts)
  ├─ InputEvent (input-event.ts)  [不经过 Dispatcher]
  ├─ TerminalFocusEvent (terminal-focus-event.ts)  [不经过 Dispatcher]
  └─ TerminalEvent (terminal-event.ts)
       ├─ FocusEvent (focus-event.ts)
       ├─ KeyboardEvent (keyboard-event.ts)
       └─ [潜在的 PasteEvent, ResizeEvent 等]
```

注意：ClickEvent 直接继承 Event，因为它使用简化的冒泡机制（见 hit-test.ts）。

## 依赖与外部交互

### 依赖

- `event.ts` - `Event` 基类

### 被依赖

- `dispatcher.ts` - 核心使用者，依赖 TerminalEvent 的所有功能
- `focus-event.ts` - `FocusEvent` 继承
- `keyboard-event.ts` - `KeyboardEvent` 继承
- `click-event.ts` - 不继承，但使用 `EventTarget` 类型

### 与 Dispatcher 的协作

```
Dispatcher.dispatch()
  ├─ _setTarget()           设置事件目标
  ├─ collectListeners()     收集监听器
  │   └─ getHandler()       通过 HANDLER_FOR_EVENT 查找
  └─ processDispatchQueue() 执行监听器
      ├─ _setEventPhase()   更新阶段
      ├─ _setCurrentTarget() 更新当前目标
      ├─ _prepareForTarget() 子类钩子
      ├─ handler()          调用用户处理器
      ├─ _isPropagationStopped() 检查是否停止
      └─ _isImmediatePropagationStopped() 检查是否立即停止
```

## 风险、边界与改进建议

### 风险点

1. **ClickEvent 不一致**:
   - ClickEvent 直接继承 Event，不使用 TerminalEvent
   - 导致 hit-test.ts 中需要手动实现冒泡逻辑
   - 可能产生维护问题

2. **EventTarget 简化**:
   - 与浏览器 EventTarget 不兼容
   - 缺少 `addEventListener` / `removeEventListener`
   - 只能使用 React 风格的 `onXxx` 属性

3. **timeStamp 精度**:
   - 使用 `performance.now()`，在 Node.js 中可用
   - 但可能与环境时间不同步

4. **内存占用**:
   - 每个事件创建多个私有字段
   - 高频事件（如 mousemove）可能产生 GC 压力

### 边界情况

1. **不可取消事件**:
   ```typescript
   // cancelable: false 时 preventDefault 无效果
   new FocusEvent('blur', null)  // FocusEvent 设置 cancelable: false
   ```

2. **非冒泡事件**:
   ```typescript
   // bubbles: false 时只触发目标阶段
   ```

3. **空目标**:
   ```typescript
   // _setCurrentTarget(null) 在分发结束后调用
   // 用户代码不应在事件处理器外访问 currentTarget
   ```

4. **重复停止传播**:
   ```typescript
   // 多次调用 stopPropagation 无害
   stopPropagation(): void {
     this._propagationStopped = true  // 幂等
   }
   ```

### 改进建议

1. **统一 ClickEvent**:
   ```typescript
   // 考虑让 ClickEvent 也继承 TerminalEvent
   // 移除 hit-test.ts 中的手动冒泡逻辑
   ```

2. **添加事件池**:
   ```typescript
   // 高频事件（如 mousemove）可以复用事件对象
   static pool: TerminalEvent[] = []
   static acquire(type: string, init?: TerminalEventInit): TerminalEvent
   release(): void
   ```

3. **添加 composed 属性**:
   ```typescript
   // 支持 Shadow DOM 风格的事件穿透
   readonly composed: boolean
   ```

4. **添加 isTrusted 属性**:
   ```typescript
   // 区分用户触发和程序化触发
   readonly isTrusted: boolean
   ```

5. **支持自定义数据**:
   ```typescript
   // 允许附加任意数据
   readonly detail?: unknown
   ```

6. **优化内存**:
   ```typescript
   // 使用 Symbol 或 WeakMap 存储私有状态
   // 减少可枚举属性数量
   ```

### 测试建议

1. 测试事件阶段转换的正确性
2. 测试 stopPropagation 和 stopImmediatePropagation 的差异
3. 测试 preventDefault 与 cancelable 的交互
4. 测试 target/currentTarget 的变化
5. 测试子类 _prepareForTarget 钩子
6. 测试大量事件创建的性能
