# event-handlers.ts 深度研究文档

## 场景与职责

`event-handlers.ts` 是 Ink 事件系统的类型定义和映射中心，负责：

1. **定义事件处理器类型**：为所有 Ink 组件提供类型安全的事件处理器类型
2. **事件类型到处理器属性的映射**：支持运行时 O(1) 查找事件对应的处理器
3. **识别事件处理器属性**：帮助 Reconciler 区分事件属性与普通 DOM 属性

该文件是连接 TypeScript 类型系统与运行时事件分发的桥梁。

## 功能点目的

### 1. 事件处理器类型定义
定义所有 Box 组件支持的事件处理器类型：
- `KeyboardEventHandler` - 键盘事件
- `FocusEventHandler` - 焦点事件
- `PasteEventHandler` - 粘贴事件
- `ResizeEventHandler` - 调整大小事件
- `ClickEventHandler` - 点击事件
- `HoverEventHandler` - 悬停事件（无参数）

### 2. 处理器属性接口
`EventHandlerProps` 定义了组件 props 中所有事件处理器：
- 支持捕获阶段处理器（`onXxxCapture`）
- 遵循 React/DOM 命名约定

### 3. 反向查找映射
`HANDLER_FOR_EVENT` 将事件类型字符串映射到处理器属性名：
```typescript
{
  keydown: { bubble: 'onKeyDown', capture: 'onKeyDownCapture' },
  focus: { bubble: 'onFocus', capture: 'onFocusCapture' },
  // ...
}
```

### 4. 事件属性集合
`EVENT_HANDLER_PROPS` 用于 Reconciler 快速判断属性是否为事件处理器。

## 具体技术实现

### 类型定义

```typescript
// 处理器类型别名
type KeyboardEventHandler = (event: KeyboardEvent) => void
type FocusEventHandler = (event: FocusEvent) => void
type PasteEventHandler = (event: PasteEvent) => void
type ResizeEventHandler = (event: ResizeEvent) => void
type ClickEventHandler = (event: ClickEvent) => void
type HoverEventHandler = () => void

// 组件 Props 中的事件处理器
export type EventHandlerProps = {
  onKeyDown?: KeyboardEventHandler
  onKeyDownCapture?: KeyboardEventHandler
  onFocus?: FocusEventHandler
  onFocusCapture?: FocusEventHandler
  onBlur?: FocusEventHandler
  onBlurCapture?: FocusEventHandler
  onPaste?: PasteEventHandler
  onPasteCapture?: PasteEventHandler
  onResize?: ResizeEventHandler
  onClick?: ClickEventHandler
  onMouseEnter?: HoverEventHandler
  onMouseLeave?: HoverEventHandler
}
```

### 运行时映射

```typescript
// 事件类型 -> 处理器属性名
export const HANDLER_FOR_EVENT: Record<
  string,
  { bubble?: keyof EventHandlerProps; capture?: keyof EventHandlerProps }
> = {
  keydown: { bubble: 'onKeyDown', capture: 'onKeyDownCapture' },
  focus: { bubble: 'onFocus', capture: 'onFocusCapture' },
  blur: { bubble: 'onBlur', capture: 'onBlurCapture' },
  paste: { bubble: 'onPaste', capture: 'onPasteCapture' },
  resize: { bubble: 'onResize' },
  click: { bubble: 'onClick' },
}

// 所有事件处理器属性名集合
export const EVENT_HANDLER_PROPS = new Set<string>([
  'onKeyDown', 'onKeyDownCapture',
  'onFocus', 'onFocusCapture',
  'onBlur', 'onBlurCapture',
  'onPaste', 'onPasteCapture',
  'onResize',
  'onClick',
  'onMouseEnter', 'onMouseLeave',
])
```

### 使用模式

1. **Dispatcher 查找处理器**（`dispatcher.ts:27-33`）:
   ```typescript
   const mapping = HANDLER_FOR_EVENT[eventType]
   const propName = capture ? mapping.capture : mapping.bubble
   return handlers[propName]
   ```

2. **Reconciler 识别事件属性**（`reconciler.ts:137-139, 447-449`）:
   ```typescript
   if (EVENT_HANDLER_PROPS.has(key)) {
     setEventHandler(node, key, value)
     return
   }
   ```

3. **组件类型定义**（`Box.tsx:11-46`）:
   ```typescript
   export type Props = Except<Styles, 'textWrap'> & {
     onClick?: (event: ClickEvent) => void
     onFocus?: (event: FocusEvent) => void
     // ...
   }
   ```

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/event-handlers.ts` - 类型和映射定义

### 使用位置

1. **dispatcher.ts**（line 8）:
   ```typescript
   import { HANDLER_FOR_EVENT } from './event-handlers.js'
   ```
   用于 `getHandler` 函数查找处理器。

2. **reconciler.ts**（line 25）:
   ```typescript
   import { EVENT_HANDLER_PROPS } from './event-handlers.js'
   ```
   用于 `applyProp` 和 `commitUpdate` 识别事件属性。

3. **Box.tsx**（lines 3-9, 30-45）:
   ```typescript
   import type { ClickEvent } from '../events/click-event.js'
   import type { FocusEvent } from '../events/focus-event.js'
   import type { KeyboardEvent } from '../events/keyboard-event.js'
   ```
   使用事件类型定义组件 Props。

4. **hit-test.ts**（line 3）:
   ```typescript
   import type { EventHandlerProps } from './events/event-handlers.js'
   ```
   用于 `dispatchHover` 的类型检查。

### 类型依赖图
```
event-handlers.ts
  ├─ click-event.ts (ClickEvent)
  ├─ focus-event.ts (FocusEvent)
  ├─ keyboard-event.ts (KeyboardEvent)
  ├─ paste-event.ts (PasteEvent)  // 未在批次中
  └─ resize-event.ts (ResizeEvent)  // 未在批次中
```

## 依赖与外部交互

### 依赖

- `click-event.ts` - `ClickEvent` 类型
- `focus-event.ts` - `FocusEvent` 类型
- `keyboard-event.ts` - `KeyboardEvent` 类型
- `paste-event.ts` - `PasteEvent` 类型（未在批次中）
- `resize-event.ts` - `ResizeEvent` 类型（未在批次中）

### 被依赖

- `dispatcher.ts` - 使用 `HANDLER_FOR_EVENT`
- `reconciler.ts` - 使用 `EVENT_HANDLER_PROPS`
- `hit-test.ts` - 使用 `EventHandlerProps` 类型
- `Box.tsx` - 使用事件类型定义 Props

## 风险、边界与改进建议

### 风险点

1. **类型与运行时不同步**:
   - `EventHandlerProps` 类型与 `HANDLER_FOR_EVENT` / `EVENT_HANDLER_PROPS` 需要手动同步
   - 添加新事件时容易遗漏更新

2. **缺少严格类型检查**:
   ```typescript
   // HANDLER_FOR_EVENT 使用 string 作为键，而非字面量类型
   HANDLER_FOR_EVENT: Record<string, ...>
   // 可考虑使用更精确的类型
   ```

3. **悬停事件特殊处理**:
   - `onMouseEnter` / `onMouseLeave` 在 `EVENT_HANDLER_PROPS` 中
   - 但不在 `HANDLER_FOR_EVENT` 中（因为不通过 Dispatcher 分发）
   - 这种不一致可能导致混淆

### 边界情况

1. **大小写敏感**:
   - 事件类型字符串（如 `'keydown'`）是小写
   - 处理器属性名使用驼峰（如 `'onKeyDown'`）
   - 需要确保映射正确

2. **可选捕获处理器**:
   - `resize` 和 `click` 没有捕获阶段处理器
   - `HANDLER_FOR_EVENT` 中对应字段为 `undefined`

3. **类型兼容性**:
   ```typescript
   // HoverEventHandler 返回 void，与其他处理器不同
   type HoverEventHandler = () => void
   // 使用时需要注意
   ```

### 改进建议

1. **使用常量替代字符串字面量**:
   ```typescript
   export const EventType = {
     KEYDOWN: 'keydown',
     FOCUS: 'focus',
     // ...
   } as const
   
   export type EventType = typeof EventType[keyof typeof EventType]
   ```

2. **自动生成映射**:
   ```typescript
   // 使用 TypeScript 类型操作自动生成 HANDLER_FOR_EVENT
   type EventTypeToHandler = {
     keydown: { bubble: 'onKeyDown', capture: 'onKeyDownCapture' }
     // ...
   }
   ```

3. **统一事件处理**:
   ```typescript
   // 考虑将 hover 事件也纳入 Dispatcher 体系
   // 或者明确分离，避免混淆
   ```

4. **添加事件元数据**:
   ```typescript
   export const EVENT_METADATA: Record<string, {
     bubbles: boolean
     cancelable: boolean
     hasCapture: boolean
   }> = {
     keydown: { bubbles: true, cancelable: true, hasCapture: true },
     // ...
   }
   ```

5. **类型安全的事件发射**:
   ```typescript
   // 在 Dispatcher 中添加类型安全的方法
   dispatchEvent<T extends keyof EventTypeToHandler>(
     type: T,
     event: EventTypeToEvent[T]
   ): boolean
   ```

### 测试建议

1. 验证所有 `EventHandlerProps` 中的属性都在 `EVENT_HANDLER_PROPS` 中
2. 验证 `HANDLER_FOR_EVENT` 中的每个事件类型都有对应的处理器
3. 测试类型推断的正确性
4. 测试运行时映射的准确性
