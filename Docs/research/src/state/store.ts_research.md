# src/state/store.ts 研究文档

## 场景与职责

`store.ts` 是 Claude Code 的 **极简命令式状态存储实现**。它提供了一个零依赖、约 30 行的自定义 store，满足以下需求：

1. **持有单一状态树** (`state`) 并提供 `getState()` 读取。
2. **提供 `setState(updater)`** — 要求传入函数式 updater `(prev) => next`，并在状态实际变化时触发副作用与订阅通知。
3. **提供 `subscribe(listener)`** — 返回 unsubscribe 函数，供 `useSyncExternalStore` 或外部代码使用。

该 store 被 `AppState.tsx` 包装为 React Context，同时也被一些非 React 代码（如 `main.tsx` 的 headless 路径）直接使用。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `Store<T>` 类型 | 定义 store 的公共接口：`getState`、`setState`、`subscribe`。 |
| `createStore<T>` | 工厂函数，接收 `initialState` 与可选的 `onChange` 回调，返回一个符合 `Store<T>` 的对象。 |
| `Object.is` 相等性检查 | 在 `setState` 中比较 updater 返回的 `next` 与当前 `prev`，若相同则短路跳过，避免无意义的副作用与重渲染。 |
| `Set<Listener>` 订阅管理 | 维护 listeners 集合，在状态变化后同步遍历触发。 |

## 具体技术实现

### 1. 完整源码分析

```ts
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 2. 关键设计决策

- **函数式 updater**：强制不可变更新模式，调用方必须返回新对象；若 mutate `prev` 并返回同一引用，`Object.is` 会短路，变化不会传播。
- **`Object.is` 而非 `===`**：正确处理 `NaN`、`-0` / `+0` 等边界情况；对于对象引用比较，两者行为一致。
- **同步副作用**：`onChange` 在 `state = next` 之后、listener 通知之前执行。这保证了 `onChange` 中的逻辑（如 `onChangeAppState.ts`）读取 `getState()` 时已经拿到新值，而 React 组件在下一帧通过 `useSyncExternalStore` 拉取快照。
- **无批处理/无调度**：所有更新都是同步、立即生效的。没有 React 的 `setState` 批处理语义，也没有 Redux 的 action/middleware 概念。

### 3. 与 React 的集成

在 `AppState.tsx` 中：

```tsx
const [store] = useState(() => createStore(initialState ?? getDefaultAppState(), onChangeAppState))
// ...
export function useAppState(selector) {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

- `useSyncExternalStore` 要求 `subscribe` 返回 unsubscribe 函数，`store.ts` 完全满足。
- `get` 函数在每次 React 调度时同步调用，因此 `store.getState()` 必须足够快（当前实现只是返回闭包变量，O(1)）。

## 关键代码路径与文件引用

| 代码路径 | 说明 |
|----------|------|
| `createStore` (L10) | 被 `src/state/AppState.tsx` 用于创建 React 集成 store；被 `src/main.tsx` 用于 headless / SDK 模式下的命令式 store。 |
| `Store<T>` 类型 (L4) | 被 `AppStateStore.ts` 别名为 `AppStateStore = Store<AppState>`；也被各种工具类型引用。 |
| `setState` (L20) | 所有 AppState 变更的最终入口，包括组件通过 `useSetAppState`、工具函数直接调用 `store.setState`、以及 `main.tsx` 的初始化逻辑。 |
| `subscribe` (L28) | 被 `useSyncExternalStore` 消费，也用于少量非 React 代码（如 `print.ts` 的 headless 订阅）。 |

## 依赖与外部交互

### 直接依赖

- 无外部运行时依赖。仅使用 TypeScript 内置类型与 JavaScript 全局对象 (`Set`, `Object.is`)。

### 调用方（上游）

- `src/state/AppState.tsx`：创建 React 上下文 store。
- `src/main.tsx`：在 headless / SDK / 非交互式路径中直接创建并使用 store。
- `src/state/AppStateStore.ts`：导入 `Store` 类型。
- `src/cli/print.ts`（可能）：headless 模式下订阅 store 变化以输出流式结果。
- 大量工具函数和 Hook 通过 `useSetAppState` 或传入的 `setAppState` 间接调用 `store.setState`。

## 风险、边界与改进建议

### 风险

1. **无中间件/无日志**  
   当前实现过于精简，没有内置的 action 日志、时间旅行、或 devtools 集成。在调试复杂状态流时，开发者只能依赖 `onChangeAppState.ts` 中的日志，难以追踪具体是哪个调用方触发了变更。

2. **listener 同步遍历的稳定性**  若某个 listener 在回调中订阅或取消订阅，由于 `Set` 的遍历顺序在 ES2015+ 中是插入顺序且对并发修改具有一定容错性，但如果在遍历过程中有 listener 被删除，行为可能变得不确定。当前代码未做防御性复制。

3. **缺少错误隔离**  若某个 `listener()` 或 `onChange` 抛出异常，会中断后续 listener 的通知，可能导致部分组件未收到更新而处于陈旧状态。

4. **无批处理导致过度通知**  在单个事件处理函数中多次调用 `setState`，每次都会触发完整的 `onChange` + listeners 遍历。虽然 React 的 `useSyncExternalStore` 会在微任务边界做一定程度的批处理，但 `onChange` 中的副作用（如磁盘写入、网络通知）会被执行多次。

### 边界

- `setState` 只接受函数式 updater，不接受直接值。这是设计上的限制，强制调用方基于 prev 计算 next。
- `createStore` 的 `onChange` 只接收新旧状态，不接收任何 action/mutation 元数据，因此无法从回调中知道“是谁触发了这次变更”。
- 没有 `getInitialState` 或 `reset` 方法；重置状态需要调用方手动 `setState(() => getDefaultAppState())`。

### 改进建议

1. **增加 listener 遍历的容错性**  
   将 `for (const listener of listeners) listener()` 改为 `Array.from(listeners).forEach(listener => listener())`，或在外层加 try/catch 隔离单个 listener 的异常，避免单点失败导致级联通知丢失。

2. **在开发模式下增加变更追踪**  可在 `onChange` 参数之外，为 `createStore` 增加可选的 `onBeforeUpdate` / `onAfterUpdate` 钩子，或在开发构建中包装 `setState` 以打印调用栈，提升调试体验。

3. **评估引入轻量状态管理库**  若未来需求增长（如需要派生状态、action 日志、中间件），可评估迁移到 zustand（API 几乎一致，迁移成本极低）或 valtio。当前自定义实现与 zustand 的 `create` 接口高度兼容，迁移只需替换工厂函数。

4. **增加批处理支持**  可考虑在 `setState` 中引入微任务批处理：将变更和通知延迟到下一个 microtask，同一事件循环内的多次 `setState` 只触发一次通知。这能显著减少 `onChangeAppState` 中的副作用重复执行。
