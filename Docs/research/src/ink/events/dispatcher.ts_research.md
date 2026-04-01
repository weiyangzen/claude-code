# dispatcher.ts 深度研究文档

## 场景与职责

`Dispatcher` 是 Ink 事件系统的核心调度器，负责实现 DOM 风格的事件传播机制。在 React 终端应用中，需要解决以下问题：

1. **事件优先级管理**：用户输入事件（键盘、点击）需要同步处理，而滚动、resize 可以异步
2. **捕获/冒泡阶段**：支持标准 DOM 事件的两阶段传播模型
3. **与 React Reconciler 集成**：事件处理需要与 React 的更新调度协调

Dispatcher 借鉴了 react-dom 的实现，将终端事件映射到 React 的优先级系统。

## 功能点目的

### 1. 两阶段事件传播
- **捕获阶段**：从根节点向下到目标节点
- **目标阶段**：事件到达目标节点
- **冒泡阶段**：从目标节点向上返回根节点

### 2. 事件优先级调度
- **DiscreteEventPriority**: 离散事件（键盘、点击、焦点），同步处理
- **ContinuousEventPriority**: 连续事件（resize、scroll、mousemove），可节流
- **DefaultEventPriority**: 默认优先级

### 3. 传播控制
- `stopPropagation()`: 阻止事件继续传播
- `stopImmediatePropagation()`: 阻止当前节点上其他监听器执行
- `preventDefault()`: 标记事件默认行为被取消

## 具体技术实现

### 核心数据结构

```typescript
// 调度监听器结构
type DispatchListener = {
  node: EventTarget           // 目标节点
  handler: (event: TerminalEvent) => void  // 处理器
  phase: 'capturing' | 'at_target' | 'bubbling'  // 阶段
}

// Dispatcher 状态
class Dispatcher {
  currentEvent: TerminalEvent | null = null
  currentUpdatePriority: number = DefaultEventPriority
  discreteUpdates: DiscreteUpdates | null = null  // 由 Reconciler 注入
}
```

### 事件收集算法

`collectListeners` 实现了 react-dom 的监听器累积模式：

```
遍历路径: target → root
├─ 捕获处理器: unshift (前置) → 根优先
└─ 冒泡处理器: push (追加) → 目标优先

结果顺序: [root-cap, ..., parent-cap, target-cap, target-bub, parent-bub, ..., root-bub]
```

代码实现（lines 46-79）：
```typescript
function collectListeners(target: EventTarget, event: TerminalEvent): DispatchListener[] {
  const listeners: DispatchListener[] = []
  let node: EventTarget | undefined = target
  
  while (node) {
    const isTarget = node === target
    
    // 捕获处理器前置
    if (captureHandler) {
      listeners.unshift({ node, handler: captureHandler, phase: isTarget ? 'at_target' : 'capturing' })
    }
    
    // 冒泡处理器追加
    if (bubbleHandler && (event.bubbles || isTarget)) {
      listeners.push({ node, handler: bubbleHandler, phase: isTarget ? 'at_target' : 'bubbling' })
    }
    
    node = node.parentNode
  }
  
  return listeners
}
```

### 事件处理器查找

`getHandler`（lines 19-34）通过 `HANDLER_FOR_EVENT` 映射表 O(1) 查找处理器：

```typescript
const mapping = HANDLER_FOR_EVENT[eventType]  // 如: { bubble: 'onClick', capture: undefined }
const propName = capture ? mapping.capture : mapping.bubble
return handlers[propName]
```

### 优先级映射

`getEventPriority`（lines 122-138）镜像 react-dom 的实现：

| 事件类型 | 优先级 | 说明 |
|---------|-------|------|
| keydown, keyup, click, focus, blur, paste | DiscreteEventPriority | 用户触发，需同步响应 |
| resize, scroll, mousemove | ContinuousEventPriority | 高频事件，可节流 |
| 其他 | DefaultEventPriority | 默认处理 |

### 三种分发模式

1. **dispatch**（lines 185-201）：标准分发，使用当前优先级
2. **dispatchDiscrete**（lines 207-218）：离散事件，使用 `discreteUpdates` 包装
3. **dispatchContinuous**（lines 224-232）：连续事件，临时提升优先级

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/dispatcher.ts` - Dispatcher 类定义

### 使用位置

1. **Reconciler 集成**（`reconciler.ts:187, 411-510`）:
   ```typescript
   export const dispatcher = new Dispatcher()
   dispatcher.discreteUpdates = reconciler.discreteUpdates.bind(reconciler)
   ```

2. **Focus 管理**（`focus.ts:234`）:
   ```typescript
   new FocusManager((target, event) => dispatcher.dispatchDiscrete(target, event))
   ```

3. **键盘事件分发**（`ink.tsx` 中通过 `dispatchKeyboardEvent`）

### 调用链示例
```
FocusManager.focus (focus.ts:27)
  → dispatchDiscrete (dispatcher.ts:207)
    → discreteUpdates (reconciler 注入)
      → dispatch (dispatcher.ts:185)
        → collectListeners (dispatcher.ts:46)
        → processDispatchQueue (dispatcher.ts:87)
```

## 依赖与外部交互

### 依赖

1. **react-reconciler/constants.js**:
   - `DiscreteEventPriority`, `ContinuousEventPriority`, `DefaultEventPriority`, `NoEventPriority`

2. **event-handlers.js**:
   - `HANDLER_FOR_EVENT` - 事件类型到处理器属性名的映射

3. **terminal-event.js**:
   - `TerminalEvent`, `EventTarget` 类型

4. **utils/log.js**:
   - `logError` - 处理器异常捕获后的日志记录

### 被依赖

- `reconciler.ts` - 创建全局 dispatcher 实例
- `focus.ts` - 使用 dispatcher 分发焦点事件
- `ink.tsx` - 键盘事件分发

### React Reconciler 集成

Dispatcher 与 Reconciler 通过以下方式协作：

```typescript
// reconciler.ts 注入 discreteUpdates
dispatcher.discreteUpdates = reconciler.discreteUpdates.bind(reconciler)

// Reconciler 读取 dispatcher 状态
getCurrentUpdatePriority: () => dispatcher.currentUpdatePriority
resolveUpdatePriority: () => dispatcher.resolveEventPriority()
resolveEventType: () => dispatcher.currentEvent?.type ?? null
resolveEventTimeStamp: () => dispatcher.currentEvent?.timeStamp ?? -1.1
```

这种设计允许 React 根据当前处理的事件类型自动调整更新优先级。

## 风险、边界与改进建议

### 风险点

1. **循环导入风险**:
   - Dispatcher 需要 Reconciler 的 `discreteUpdates`
   - Reconciler 需要 Dispatcher 的实例
   - 通过延迟注入（lines 158-160 注释说明）打破循环

2. **异常处理**:
   - `processDispatchQueue` 捕获处理器异常（line 108-110）
   - 但仅记录日志，可能导致事件处理状态不一致

3. **内存泄漏**:
   - `currentEvent` 在 `dispatch` 结束后恢复（line 199）
   - 如果处理器抛出异常，`finally` 块确保清理

### 边界情况

1. **空监听器列表**:
   - `collectListeners` 可能返回空数组
   - `processDispatchQueue` 直接返回，无性能开销

2. **stopPropagation 与 stopImmediatePropagation**:
   - `stopPropagation` 阻止向父节点传播（line 98-100）
   - `stopImmediatePropagation` 立即停止所有处理（line 94-96）
   - 两者都检查，确保正确行为

3. **事件重入**:
   - `previousEvent` 保存（line 186）
   - 支持事件处理器中触发新事件

### 改进建议

1. **添加事件统计**:
   ```typescript
   // 用于调试性能问题
   eventCount: Map<string, number>
   totalDispatchTime: number
   ```

2. **支持 once 监听器**:
   ```typescript
   // 类似 addEventListener 的 once 选项
   onClickOnce?: (event: ClickEvent) => void
   ```

3. **异步处理器支持**:
   ```typescript
   // 允许处理器返回 Promise
   async processDispatchQueue(listeners, event) {
     for (const listener of listeners) {
       await listener.handler(event)
     }
   }
   ```

4. **事件委托优化**:
   ```typescript
   // 当前每个节点单独存储处理器
   // 可考虑在根节点统一委托，减少内存占用
   ```

5. **更细粒度的优先级**:
   ```typescript
   // 区分用户输入和程序化触发
   UserInputPriority
   ProgrammaticPriority
   ```

### 测试建议

1. 测试捕获/冒泡顺序的正确性
2. 测试 stopPropagation/stopImmediatePropagation 的行为差异
3. 测试事件重入场景
4. 测试大量监听器时的性能
5. 测试异常处理器不影响后续事件
