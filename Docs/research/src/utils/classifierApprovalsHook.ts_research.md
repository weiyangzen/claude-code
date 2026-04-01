# src/utils/classifierApprovalsHook.ts 深入研究

## 场景与职责

`classifierApprovalsHook.ts` 是 `classifierApprovals.ts` 的 **React Hook 薄封装层**。它的唯一目的是：将 `classifierApprovals.ts` 中基于 `createSignal` 的 imperative store 桥接到 React 的 `useSyncExternalStore`，使 React 组件能够订阅分类器检查状态的变化。

该文件被刻意从 `classifierApprovals.ts` 中拆分出来，以避免纯状态消费者（如 `permissions.ts`、`toolExecution.ts`、`postCompactCleanup.ts`）在 print / non-interactive 模式下意外将 React 打包进依赖图。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `useIsClassifierChecking(toolUseID)` | React Hook，返回指定 tool use 是否正处于分类器检查中 |

## 具体技术实现

### useSyncExternalStore 桥接
```ts
import { useSyncExternalStore } from 'react'
import { isClassifierChecking, subscribeClassifierChecking } from './classifierApprovals.js'

export function useIsClassifierChecking(toolUseID: string): boolean {
  return useSyncExternalStore(
    subscribeClassifierChecking,
    () => isClassifierChecking(toolUseID),
  )
}
```
- `subscribeClassifierChecking` 作为 `useSyncExternalStore` 的 `subscribe` 参数。
- `() => isClassifierChecking(toolUseID)` 作为 `getSnapshot` 参数，每次 signal emit 时重新读取当前状态。
- `useSyncExternalStore` 保证在并发渲染（Concurrent Rendering）下的快照一致性，避免 tearing 问题。

## 关键代码路径与文件引用

```
src/components/messages/AssistantToolUseMessage.tsx
  └── useIsClassifierChecking(toolUseID)
      [渲染 assistant 的工具调用消息时，展示分类器检查中的加载动画/提示]
```

### 依赖模块
- `react` — `useSyncExternalStore`
- `src/utils/classifierApprovals.ts` — `isClassifierChecking`, `subscribeClassifierChecking`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| `classifierApprovals.ts` | 导入订阅函数与快照读取函数 | 底层状态存储与事件机制 |
| React 运行时 | `useSyncExternalStore` | 将外部 store 集成到组件渲染周期 |
| `AssistantToolUseMessage.tsx` | Hook 消费方 | 在 UI 中展示分类器检查状态 |

## 风险、边界与改进建议

### 风险
1. **Hook 过度渲染**：`isClassifierChecking` 的 snapshot 函数在每次 signal emit 时都会被所有订阅组件调用。若同时有大量 tool use 处于检查中，单次 `emit` 可能触发大量组件重新渲染。
2. **无记忆化**：当前未使用 `useCallback` 或 `useMemo` 包裹 snapshot 函数；虽然 `useSyncExternalStore` 内部会做优化，但频繁创建匿名函数在极端场景下仍有微小开销。
3. **单一 Hook 职责过窄**：目前仅暴露了 `useIsClassifierChecking`，若未来 UI 需要订阅分类器批准记录（如实时显示 "approved by classifier" 标签），需要新增 Hook，模块扩展性一般。

### 边界
- 该模块**仅用于 React 组件**；非 React 代码应直接导入 `classifierApprovals.ts`。
- `useIsClassifierChecking` 返回的是布尔值，不携带分类器类型、规则名、原因等额外信息；这些信息由 `UserToolSuccessMessage.tsx` 直接调用 `getClassifierApproval` / `getYoloClassifierApproval` 获取。
- 若 `feature('BASH_CLASSIFIER')` 与 `feature('TRANSCRIPT_CLASSIFIER')` 均关闭，`classifierApprovals.ts` 中的 `setClassifierChecking` 为 no-op，因此该 Hook 始终返回 `false`。

### 改进建议
1. **增加 `useClassifierApproval(toolUseID)` Hook**：若 `UserToolSuccessMessage` 未来需要响应式更新批准标签（而非仅在 mount 时读取），可在此模块中新增基于 `useSyncExternalStore` 的批准记录 Hook（前提是 `classifierApprovals.ts` 也为批准记录增加 signal）。
2. **记忆化 snapshot 函数**：
   ```ts
   const getSnapshot = useCallback(() => isClassifierChecking(toolUseID), [toolUseID])
   return useSyncExternalStore(subscribeClassifierChecking, getSnapshot)
   ```
   减少每次渲染创建新函数的开销。
3. **批量 emit 优化**：在 `classifierApprovals.ts` 中，若多个 tool use 的检查状态在短时间内连续变化，可考虑引入微任务批量化（microtask batching），将多次 `emit` 合并为一次，减少 React 重排次数。
4. **文档化拆分原因**：在 `classifierApprovals.ts` 顶部增加更醒目的注释，说明"所有 React 相关消费必须通过 classifierApprovalsHook.ts"，防止新开发者直接从 `classifierApprovals.ts` 引入 React 依赖到 print 模式。
