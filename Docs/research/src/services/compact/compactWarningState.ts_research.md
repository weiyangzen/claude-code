# compactWarningState.ts 深度研究文档

## 场景与职责

`compactWarningState.ts` 实现了上下文压缩警告抑制状态的纯状态管理模块。它使用自定义的 `createStore` 工厂创建简单的发布-订阅状态存储，用于控制"上下文即将耗尽"警告的显示时机。

该模块的核心业务逻辑是：**在成功压缩后短暂抑制警告显示**，因为压缩后的准确 token 计数需要等待下一次 API 响应才能获取，期间显示的警告可能基于过时的估算值。

## 功能点目的

### 1. 警告抑制状态管理
- 跟踪当前是否应抑制上下文警告的显示
- 默认状态为 `false`（不抑制，正常显示警告）

### 2. 状态变更 API
- **`suppressCompactWarning`**: 在成功压缩后调用，抑制警告
- **`clearCompactWarningSuppression`**: 在新的压缩尝试开始时调用，清除抑制

### 3. 框架无关设计
- 不依赖 React 或其他 UI 框架
- 可被 Node.js 环境、打印模式启动路径等非 React 环境安全导入

## 具体技术实现

### 核心实现

```typescript
import { createStore } from '../../state/store.js'

// 创建布尔值 store，默认 false（不抑制）
export const compactWarningStore = createStore<boolean>(false)

/** 抑制压缩警告 */
export function suppressCompactWarning(): void {
  compactWarningStore.setState(() => true)
}

/** 清除压缩警告抑制 */
export function clearCompactWarningSuppression(): void {
  compactWarningStore.setState(() => false)
}
```

### Store 工厂接口

```typescript
// src/state/store.ts (推断)
interface Store<T> {
  subscribe(callback: () => void): () => void  // 订阅变化，返回取消订阅
  getState(): T                                 // 获取当前状态
  setState(updater: (prev: T) => T): void       // 更新状态
}
```

### 状态流转图

```
┌─────────────┐     clearCompactWarningSuppression()      ┌─────────────┐
│  SUPPRESSED │ ─────────────────────────────────────────> │   ACTIVE    │
│   (true)    │                                            │   (false)   │
└─────────────┘                                            └─────────────┘
       ▲                                                           │
       └───────────────────────────────────────────────────────────┘
                    suppressCompactWarning()
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `../../state/store.js` | `createStore` 工厂函数 |

### 外部调用方

| 调用方 | 路径 | 调用函数 |
|-------|------|---------|
| `microCompact.ts` | `src/services/compact/microCompact.ts:26,359,511` | `suppressCompactWarning`, `clearCompactWarningSuppression` |
| `compact.ts` (command) | `src/commands/compact/compact.ts:16,75,115` | `suppressCompactWarning` |
| `compactWarningHook.ts` | `src/services/compact/compactWarningHook.ts:2` | `compactWarningStore` |

### 调用时序

```
1. 新的压缩尝试开始
   └─> clearCompactWarningSuppression()  // 确保警告可显示
       
2. 压缩成功完成
   └─> suppressCompactWarning()          // 抑制警告直到下次 API 响应
   
3. 下次 API 响应后
   └─> （token 计数更新，警告基于新值显示）
   
4. 循环回到步骤 1
```

## 依赖与外部交互

### Store 工厂

`createStore` 是一个轻量级的状态管理工厂，提供类似 Zustand 的 API：

```typescript
// 伪代码展示 createStore 的行为
function createStore<T>(initialState: T) {
  let state = initialState
  const listeners = new Set<() => void>()
  
  return {
    subscribe: (cb: () => void) => {
      listeners.add(cb)
      return () => listeners.delete(cb)
    },
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      state = updater(state)
      listeners.forEach(cb => cb())
    },
  }
}
```

## 风险、边界与改进建议

### 已知风险

1. **状态持久化风险**: 
   - 如果 `clearCompactWarningSuppression` 未被正确调用，警告可能持续被抑制
   - 实际代码中在 `microcompactMessages` 开头调用，确保每次尝试前重置

2. **无超时机制**:
   - 警告抑制没有自动过期机制
   - 依赖后续流程正确调用 `clearCompactWarningSuppression`

### 边界条件

| 场景 | 行为 |
|------|------|
| 多次调用 `suppressCompactWarning` | 幂等，状态保持 `true` |
| 多次调用 `clearCompactWarningSuppression` | 幂等，状态保持 `false` |
| 无订阅者时状态变更 | 正常更新状态，无回调触发 |
| 并发状态更新 | 由 `createStore` 内部保证原子性 |

### 改进建议

1. **添加自动过期**（可选）:
   ```typescript
   let suppressionTimeout: NodeJS.Timeout | null = null
   
   export function suppressCompactWarning(durationMs = 30000): void {
     compactWarningStore.setState(() => true)
     
     // 自动清除
     if (suppressionTimeout) clearTimeout(suppressionTimeout)
     suppressionTimeout = setTimeout(() => {
       clearCompactWarningSuppression()
     }, durationMs)
   }
   ```

2. **添加调试日志**（开发模式）:
   ```typescript
   export function suppressCompactWarning(): void {
     if (process.env.DEBUG_COMPACT) {
       console.log('[CompactWarning] Suppressed')
     }
     compactWarningStore.setState(() => true)
   }
   ```

3. **当前设计保持简单**: 该模块职责单一，当前实现已满足需求，不建议过度工程化
