# emitter.ts 深度研究文档

## 场景与职责

`EventEmitter` 是 Ink 中基于 Node.js 内置 `EventEmitter` 的扩展类，专门用于处理 Ink 的事件传播。与标准 Node EventEmitter 相比，它增加了对 Ink 自定义 `Event` 类的感知能力。

主要使用场景：
1. **全局输入事件分发**：`App.tsx` 使用它将输入事件广播给所有 `useInput` hook
2. **终端焦点事件**：处理终端窗口获得/失去焦点的事件
3. **挂起/恢复事件**：处理进程 SIGSTOP/SIGCONT 信号相关事件

## 功能点目的

### 1. stopImmediatePropagation 支持
标准 Node EventEmitter 没有内置的事件传播停止机制。`EventEmitter.emit` 方法检查事件对象是否调用了 `stopImmediatePropagation()`，如果调用了则停止调用后续监听器。

### 2. 无监听器数量限制
通过 `setMaxListeners(0)` 禁用默认的 10 个监听器警告。在 React 应用中，多个组件可能同时监听同一事件（如多个 `useInput` hooks），默认限制会导致虚假警告。

### 3. 错误事件特殊处理
错误事件（`'error'`）直接委托给父类，保持 Node.js 的错误处理语义（未处理的 error 事件会抛出）。

## 具体技术实现

### 类定义

```typescript
export class EventEmitter extends NodeEventEmitter {
  constructor() {
    super()
    this.setMaxListeners(0)  // 禁用监听器数量警告
  }

  override emit(type: string | symbol, ...args: unknown[]): boolean {
    // 错误事件直接委托
    if (type === 'error') {
      return super.emit(type, ...args)
    }

    const listeners = this.rawListeners(type)
    if (listeners.length === 0) {
      return false
    }

    // 检查第一个参数是否是 Ink 的 Event 实例
    const ccEvent = args[0] instanceof Event ? args[0] : null

    for (const listener of listeners) {
      listener.apply(this, args)

      // 如果事件调用了 stopImmediatePropagation，停止后续调用
      if (ccEvent?.didStopImmediatePropagation()) {
        break
      }
    }

    return true
  }
}
```

### 关键流程

1. **获取监听器列表**:
   ```typescript
   const listeners = this.rawListeners(type)
   ```
   使用 `rawListeners` 而非 `listeners`，返回原始监听器数组（包括 once 包装器）。

2. **事件类型检查**:
   ```typescript
   const ccEvent = args[0] instanceof Event ? args[0] : null
   ```
   检查第一个参数是否是 Ink 的 `Event` 实例，用于后续传播控制。

3. **顺序调用与传播控制**:
   ```typescript
   for (const listener of listeners) {
     listener.apply(this, args)
     if (ccEvent?.didStopImmediatePropagation()) {
       break
     }
   }
   ```
   按注册顺序调用监听器，每次调用后检查是否停止传播。

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/emitter.ts` - EventEmitter 类定义

### 使用位置

1. **App.tsx**（line 115）:
   ```typescript
   internal_eventEmitter = new EventEmitter()
   ```
   作为应用级事件总线，用于：
   - `'input'` 事件：`useInput` hook 监听
   - `'terminalfocus'` / `'terminalblur'` 事件：终端焦点变化
   - `'suspend'` / `'resume'` 事件：进程挂起/恢复

2. **use-input.ts**（lines 83-89）:
   ```typescript
   useEffect(() => {
     internal_eventEmitter?.on('input', handleData)
     return () => {
       internal_eventEmitter?.removeListener('input', handleData)
     }
   }, [internal_eventEmitter, handleData])
   ```

3. **StdinContext.ts**:
   通过 Context 将 EventEmitter 传递给子组件。

### 调用链示例
```
App.processKeysInBatch (App.tsx:444)
  → internal_eventEmitter.emit('input', event) (App.tsx:507)
    → EventEmitter.emit (emitter.ts:15)
      → listener.apply(this, args) (emitter.ts:30)
        → useInput.handleData (use-input.ts:69)
```

## 依赖与外部交互

### 依赖

1. **Node.js events 模块**:
   ```typescript
   import { EventEmitter as NodeEventEmitter } from 'events'
   ```

2. **event.ts**:
   ```typescript
   import { Event } from './event.js'
   ```
   用于 `stopImmediatePropagation` 检查。

### 被依赖

- `App.tsx` - 创建实例作为应用事件总线
- `ink.ts` - 导出供外部使用

### 与 DOM EventEmitter 的差异

| 特性 | Node EventEmitter | Ink EventEmitter | DOM EventTarget |
|-----|-------------------|------------------|-----------------|
| 捕获/冒泡 | 不支持 | 不支持（由 Dispatcher 处理） | 支持 |
| stopPropagation | 不支持 | 部分支持（stopImmediatePropagation） | 支持 |
| 事件对象 | 任意 | 支持 Ink Event | Event 实例 |
| 监听器限制 | 10（可配置） | 无限制 | 无限制 |

## 风险、边界与改进建议

### 风险点

1. **监听器顺序依赖**:
   - `useInput` hook 依赖注册顺序实现 `stopImmediatePropagation`
   - 如果 React 渲染顺序变化，可能导致行为不一致

2. **内存泄漏**:
   - 组件卸载时必须调用 `removeListener`
   - `use-input.ts` 的 `useEffect` 清理函数处理了这一点

3. **错误处理**:
   - 监听器异常会中断后续监听器调用
   - 与 Dispatcher 不同，没有 try-catch 包装

### 边界情况

1. **空事件对象**:
   ```typescript
   // 如果 emit 时传入 null/undefined
   const ccEvent = args[0] instanceof Event ? args[0] : null
   // ccEvent 为 null，stopImmediatePropagation 检查跳过
   ```

2. **非 Event 实例**:
   - 可以 emit 任意类型的事件
   - 只有 `Event` 实例支持传播控制

3. **错误事件**:
   - 特殊处理保持 Node.js 语义
   - 未监听的 error 事件会抛出

### 改进建议

1. **添加监听器异常捕获**:
   ```typescript
   for (const listener of listeners) {
     try {
       listener.apply(this, args)
     } catch (error) {
       logError(error)
       // 继续调用后续监听器
     }
     if (ccEvent?.didStopImmediatePropagation()) {
       break
     }
   }
   ```

2. **支持异步监听器**:
   ```typescript
   async emit(type: string, ...args: unknown[]): Promise<boolean> {
     // 支持 async 监听器
     for (const listener of listeners) {
       await listener.apply(this, args)
     }
   }
   ```

3. **添加调试信息**:
   ```typescript
   // 开发模式下记录事件流向
   if (process.env.NODE_ENV === 'development') {
     console.log(`[EventEmitter] ${String(type)} -> ${listeners.length} listeners`)
   }
   ```

4. ** once 监听器优化**:
   ```typescript
   // 当前 once 监听器由 Node 内部包装
   // 可考虑在 stopImmediatePropagation 时移除未调用的 once 监听器
   ```

5. **事件名称类型安全**:
   ```typescript
   // 使用类型定义允许的事件名称
   type InkEventMap = {
     'input': InputEvent
     'terminalfocus': TerminalFocusEvent
     'terminalblur': TerminalFocusEvent
     'suspend': void
     'resume': void
   }
   ```

### 测试建议

1. 测试 `stopImmediatePropagation` 正确停止后续监听器
2. 测试大量监听器时的性能
3. 测试监听器异常处理
4. 测试监听器添加/移除的正确性
5. 测试 once 监听器的行为
