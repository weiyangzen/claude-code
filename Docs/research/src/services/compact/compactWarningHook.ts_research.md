# compactWarningHook.ts 深度研究文档

## 场景与职责

`compactWarningHook.ts` 是一个轻量级的 React Hook 模块，用于在 UI 层订阅上下文压缩警告的抑制状态。它是 `compactWarningState.ts` 的 React 绑定层，遵循关注点分离原则，将纯状态管理与 React 框架解耦。

该模块的主要使用场景是在 `TokenWarning.tsx` 组件中，用于控制"上下文即将耗尽"警告的显示行为。

## 功能点目的

### 1. React 状态订阅
- 使用 `useSyncExternalStore` Hook 订阅 `compactWarningStore` 的状态变化
- 提供类型安全的布尔值状态访问

### 2. 框架解耦
- 将 React 依赖限制在此单一模块中
- 确保 `compactWarningState.ts` 可被非 React 环境（如打印模式启动路径）安全导入

## 具体技术实现

### 核心实现

```typescript
import { useSyncExternalStore } from 'react'
import { compactWarningStore } from './compactWarningState.js'

export function useCompactWarningSuppression(): boolean {
  return useSyncExternalStore(
    compactWarningStore.subscribe,    // 订阅函数
    compactWarningStore.getState,     // 获取当前状态
  )
}
```

### 设计模式

该模块采用 **Adapter Pattern**（适配器模式）：

```
┌─────────────────────┐
│  compactWarningState │  ← 纯状态管理（框架无关）
│  (Store 实现)        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  compactWarningHook  │  ← React 适配层
│  (Hook 封装)         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  TokenWarning.tsx    │  ← UI 组件
│  (消费者)            │
└─────────────────────┘
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `react` | `useSyncExternalStore` |
| `./compactWarningState.js` | `compactWarningStore` |

### 外部调用方

| 调用方 | 路径 | 用途 |
|-------|------|------|
| `TokenWarning.tsx` | `src/components/TokenWarning.tsx` | 控制警告显示抑制 |

### 调用代码片段

```typescript
// src/components/TokenWarning.tsx
import { useCompactWarningSuppression } from '../services/compact/compactWarningHook.js'

function TokenWarning({ tokenUsage, model }) {
  const isSuppressed = useCompactWarningSuppression()
  
  // 如果警告被抑制，不显示警告 UI
  if (isSuppressed) {
    return null
  }
  
  // 正常显示警告...
}
```

## 依赖与外部交互

### React API

| API | 用途 |
|-----|------|
| `useSyncExternalStore` | 订阅外部 store 的标准 React Hook |

### Store 接口契约

`compactWarningStore` 必须实现以下接口：

```typescript
interface CompactWarningStore {
  subscribe(callback: () => void): () => void  // 返回取消订阅函数
  getState(): boolean                           // 返回当前抑制状态
}
```

## 风险、边界与改进建议

### 已知风险

1. **无风险**: 该模块实现极其简单，仅做一层转发，无复杂逻辑

### 边界条件

| 场景 | 行为 |
|------|------|
| Store 未初始化 | React 会抛出错误（但实际情况中不会发生） |
| 组件卸载 | `useSyncExternalStore` 自动处理取消订阅 |

### 改进建议

1. **当前设计合理**: 该模块职责单一，无需修改

2. **潜在扩展**（如需求变更）:
   ```typescript
   // 如果需要支持选择器
   export function useCompactWarningSuppression<T>(
     selector: (state: boolean) => T = identity
   ): T {
     return useSyncExternalStore(
       compactWarningStore.subscribe,
       () => selector(compactWarningStore.getState()),
     )
   }
   ```

3. **文档维护**: 如 `compactWarningState.ts` 的接口变更，需同步更新此模块
