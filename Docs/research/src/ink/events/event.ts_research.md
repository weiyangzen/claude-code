# event.ts 深度研究文档

## 场景与职责

`Event` 是 Ink 事件系统的基类，提供最基础的事件传播控制功能。它是所有 Ink 事件（`InputEvent`、`TerminalFocusEvent` 等）的祖先类。

设计目标：
1. **轻量级**：仅包含最核心的传播控制功能
2. **兼容性**：与 Node.js EventEmitter 和 DOM Event 都保持一定兼容
3. **可扩展性**：为子类提供基础，子类可添加特定功能

## 功能点目的

### 1. 立即传播停止
`stopImmediatePropagation()` 方法允许事件监听器阻止同一事件上其他监听器的执行。这是 Ink 事件系统的核心功能之一。

### 2. 状态查询
`didStopImmediatePropagation()` 方法供 EventEmitter 检查是否需要停止调用后续监听器。

## 具体技术实现

### 类定义

```typescript
export class Event {
  private _didStopImmediatePropagation = false

  didStopImmediatePropagation(): boolean {
    return this._didStopImmediatePropagation
  }

  stopImmediatePropagation(): void {
    this._didStopImmediatePropagation = true
  }
}
```

### 设计特点

1. **私有状态**：使用私有字段 `_didStopImmediatePropagation`，避免外部直接修改
2. **方法命名**：遵循 DOM Event API 命名约定（`stopImmediatePropagation`）
3. **简单性**：仅 11 行代码，功能单一明确

### 与 DOM Event 的对比

| 特性 | Ink Event | DOM Event |
|-----|-----------|-----------|
| stopPropagation | 不支持（由 Dispatcher 处理） | 支持 |
| stopImmediatePropagation | 支持 | 支持 |
| preventDefault | 不支持（由 TerminalEvent 处理） | 支持 |
| bubbles | 不支持（由 TerminalEvent 处理） | 支持 |
| target/currentTarget | 不支持（由 TerminalEvent 处理） | 支持 |

Ink 采用分层设计：
- `Event`：最基础，仅用于 EventEmitter 的传播控制
- `TerminalEvent`：DOM 风格，支持 target、currentTarget、冒泡等

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/event.ts` - Event 基类定义

### 使用位置

1. **EventEmitter**（`emitter.ts:27, 32`）:
   ```typescript
   const ccEvent = args[0] instanceof Event ? args[0] : null
   // ...
   if (ccEvent?.didStopImmediatePropagation()) {
     break
   }
   ```

2. **InputEvent**（`input-event.ts:192`）:
   ```typescript
   export class InputEvent extends Event {
     // ...
   }
   ```

3. **TerminalFocusEvent**（`terminal-focus-event.ts:12`）:
   ```typescript
   export class TerminalFocusEvent extends Event {
     // ...
   }
   ```

4. **TerminalEvent**（`terminal-event.ts:19`）:
   ```typescript
   export class TerminalEvent extends Event {
     // 继承并扩展
   }
   ```

### 继承层次
```
Event (event.ts)
  ├─ InputEvent (input-event.ts)
  ├─ TerminalFocusEvent (terminal-focus-event.ts)
  └─ TerminalEvent (terminal-event.ts)
       ├─ ClickEvent (click-event.ts)
       ├─ FocusEvent (focus-event.ts)
       └─ KeyboardEvent (keyboard-event.ts)
```

## 依赖与外部交互

### 依赖
无外部依赖，纯基础类。

### 被依赖

- `emitter.ts` - 检查 `didStopImmediatePropagation()`
- `input-event.ts` - `InputEvent` 继承
- `terminal-focus-event.ts` - `TerminalFocusEvent` 继承
- `terminal-event.ts` - `TerminalEvent` 继承
- `ink.ts` - 导出供外部使用

## 风险、边界与改进建议

### 风险点

1. **功能过于简单**:
   - 不包含 `stopPropagation`（阻止冒泡）
   - 这个职责由 `TerminalEvent` 和 `Dispatcher` 承担
   - 可能导致理解上的困惑

2. **与 DOM Event 不完全兼容**:
   - 如果用户期望 DOM Event 的完整 API，可能会失望
   - 文档需要明确说明差异

3. **instanceof 检查**:
   - `emitter.ts` 使用 `instanceof Event` 检查
   - 如果存在多个 Event 类副本（如打包问题），检查会失败

### 边界情况

1. **重复使用**:
   ```typescript
   // Event 实例是否可重用？
   const event = new Event()
   event.stopImmediatePropagation()
   // 重置？
   // 当前没有 reset 方法
   ```

2. **子类覆盖**:
   ```typescript
   // TerminalEvent 覆盖了 stopImmediatePropagation
   override stopImmediatePropagation(): void {
     super.stopImmediatePropagation()
     this._propagationStopped = true  // 同时设置冒泡停止
   }
   ```

3. **序列化**:
   - Event 实例不能被 JSON 序列化（私有字段）
   - 这在跨进程通信时需要考虑

### 改进建议

1. **添加 Symbol 标识**:
   ```typescript
   // 避免 instanceof 的多副本问题
   static readonly brand = Symbol.for('ink.Event')
   readonly [Event.brand] = true
   ```

2. **添加重置方法**（如果支持重用）:
   ```typescript
   reset(): void {
     this._didStopImmediatePropagation = false
   }
   ```

3. **添加类型守卫**:
   ```typescript
   export function isEvent(obj: unknown): obj is Event {
     return obj instanceof Event
   }
   ```

4. **文档改进**:
   - 明确说明与 DOM Event 的差异
   - 说明何时使用 Event vs TerminalEvent

5. **冻结实例**:
   ```typescript
   stopImmediatePropagation(): void {
     if (this._didStopImmediatePropagation) return
     this._didStopImmediatePropagation = true
     Object.freeze(this)  // 防止后续修改
   }
   ```

### 测试建议

1. 测试 `stopImmediatePropagation` 后 `didStopImmediatePropagation` 返回 true
2. 测试多次调用 `stopImmediatePropagation` 不会报错
3. 测试子类继承后的行为
4. 测试 `instanceof` 检查在各种场景下的可靠性
