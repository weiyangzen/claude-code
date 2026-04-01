# notifications.tsx 研究文档

## 场景与职责

`notifications.tsx` 实现了 Claude Code 的**应用级通知队列系统**。它基于 `AppState` 的 `notifications` 切片（`{ current: Notification | null, queue: Notification[] }`），提供：

1. **`addNotification`** — 将通知加入队列或立即展示（支持优先级、折叠、失效、超时）。
2. **`removeNotification`** — 按 key 移除当前展示或队列中的通知。
3. **`getNext`** — 从队列中按优先级选出下一个待展示通知。

该系统用于在底部状态栏（`PromptInputFooter` 区域）显示临时提示，如 IDE 连接状态、自动更新结果、外部编辑器快捷方式提示、环境钩子变更等。

## 功能点目的

### 1. 通知类型
```ts
type Priority = 'low' | 'medium' | 'high' | 'immediate'

type Notification =
  | { key: string; text: string; color?: keyof Theme; priority: Priority; timeoutMs?: number; invalidates?: string[]; fold?: ... }
  | { key: string; jsx: React.ReactNode; priority: Priority; timeoutMs?: number; invalidates?: string[]; fold?: ... }
```

- **`key`**：唯一标识，用于去重和移除。
- **`invalidates`**：当新通知到达时，自动清除具有指定 key 的现有通知（队列中和当前展示的都清除）。
- **`fold`**：若队列或当前展示中已存在相同 `key` 的通知，调用 `fold(accumulator, incoming)` 合并为一条通知（类似 `Array.reduce`）。
- **`priority`**：`immediate` > `high` > `medium` > `low`，队列按优先级出队。
- **`timeoutMs`**：默认 8000ms，超时后自动清除当前通知并尝试出队下一条。

### 2. `useNotifications` Hook
返回 `{ addNotification, removeNotification }`，内部通过 `useAppStateStore()` 和 `useSetAppState()` 读写全局状态。

### 3. `processQueue`
私有回调，负责：
- 当 `current === null` 时，从 `queue` 中按优先级取出下一条通知设为 `current`。
- 为 `current` 设置 `setTimeout`，超时后清除并再次触发 `processQueue`。

## 具体技术实现

### 优先级与出队
```ts
const PRIORITIES: Record<Priority, number> = {
  immediate: 0, high: 1, medium: 2, low: 3
}

export function getNext(queue: Notification[]): Notification | undefined {
  if (queue.length === 0) return undefined
  return queue.reduce((min, n) =>
    PRIORITIES[n.priority] < PRIORITIES[min.priority] ? n : min
  )
}
```

### `addNotification` 的关键分支

#### A. `immediate` 优先级
- **立即中断**：清除现有 `currentTimeoutId`，将新通知直接设为 `current`。
- **旧通知回队**：将之前的 `current`（如果存在且非 immediate）推到队列头部。
- **过滤失效**：从队列中移除被 `invalidates` 指定的通知。
- **设置新超时**：为新的 immediate 通知启动独立超时。

#### B. 非 immediate + 存在 `fold`
- **匹配 current**：若 `current.key === notif.key`，调用 `fold(current, notif)` 生成新通知替换 `current`，并**重置超时**。
- **匹配 queue**：若队列中存在相同 key，调用 `fold(queued, notif)` 替换队列项。

#### C. 普通去重添加
- 若 key 已存在于 `current` 或 `queue` 中，直接忽略（`shouldAdd = false`）。
- 若新通知 `invalidates` 命中 `current`，立即清除 `current` 并过滤队列。
- 否则将通知追加到队列尾部（过滤掉被 invalidates 的项和已有的 immediate 项）。
- 最后调用 `processQueue()` 尝试展示。

### `removeNotification`
- 若 key 匹配 `current`，清除超时并将 `current` 设为 `null`。
- 过滤队列中匹配项。
- 调用 `processQueue()` 尝试展示下一条。

### 模块级 `currentTimeoutId`
文件顶部维护一个模块级变量 `let currentTimeoutId: NodeJS.Timeout | null = null`，用于：
- 在 `immediate` 到达时清除前一个超时。
- 在 `fold` 重置超时时清除旧超时。
- 在 `removeNotification` 清除当前通知时取消超时。

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/notifications.tsx` | 本文件，通知队列核心逻辑 |
| `src/state/AppStateStore.ts` | 定义 `AppState.notifications` 切片结构 |
| `src/state/AppState.tsx` | 提供 `useAppStateStore`、`useSetAppState` |
| `src/components/PromptInput/Notifications.tsx` | **主要渲染方**，读取 `notifications.current` 并渲染为底部状态栏文本/JSX |
| `src/components/PromptInput/PromptInput.tsx` | 组装 `Notifications` 组件到输入区底部 |
| `src/hooks/notifs/*.tsx` | 大量通知生产者（如 `useUpdateNotification.ts`、`useFastModeNotification.tsx` 等） |
| `src/utils/collapseBackgroundBashNotifications.ts` | 利用 `fold` 机制合并后台 Bash 通知 |

## 依赖与外部交互

- **React**：`useCallback`、`useEffect`。
- **`src/state/AppState.js`**：强依赖 `useAppStateStore` 和 `useSetAppState`。
- **`../utils/theme.js`**：类型依赖 `Theme`（用于 `color?: keyof Theme`）。
- **无直接 UI 依赖**：本文件纯逻辑，不渲染任何 JSX。

## 风险、边界与改进建议

### 风险与边界
1. **模块级 `currentTimeoutId` 的并发风险**：
   - 该变量是单例，假设整个应用只有一个通知队列实例。如果未来出现多个 `AppStateProvider`（虽然 `AppState.tsx` 已禁止嵌套），或测试并行运行，模块级变量会导致超时互相覆盖。
2. **`setTimeout` 闭包传递 `setAppState` 和 `processQueue`**：
   - 为了处理 React 闭包陈旧问题，代码将 `setAppState`、`nextKey`、`processQueue` 作为参数显式传入 `setTimeout` 回调。这增加了代码复杂度，且容易在重构时遗漏参数。
3. **`getNext` 使用 `reduce` 而非排序/堆**：
   - 每次 `processQueue` 都 O(n) 扫描队列。虽然通知队列通常很短，但在极端情况下（如批量后台任务通知）可能成为微性能瓶颈。
4. **`fold` 无默认实现**：
   - 调用方必须显式提供 `fold` 函数。若忘记提供且重复发送相同 key 的通知，第二次会被静默丢弃（去重逻辑），这可能不符合预期。
5. **mount-only effect 的 eslint 忽略**：
   ```ts
   // biome-ignore lint/correctness/useExhaustiveDependencies: mount-only effect
   useEffect(() => { ... }, [])
   ```
   该 effect 在组件挂载时检查初始队列长度并触发 `processQueue`。若 `store` 在挂载后立刻变化（理论上不应发生），可能漏处理。

### 改进建议
1. **将 `currentTimeoutId` 纳入状态或 ref**：
   - 使用 `useRef` 或将其存入 `AppState.notifications.timeoutId` 中，消除模块级单例，提升可测试性和多实例安全性。
2. **引入优先队列数据结构**：
   - 若通知量增长，可用最小堆（min-heap）替代 `reduce`，将 `getNext` 优化到 O(log n)。
3. **为 `fold` 提供默认行为**：
   - 当通知带 `key` 且未提供 `fold` 时，默认用新通知覆盖旧通知（或合并文本），而不是直接丢弃。这可通过在 `addNotification` 入口处补充默认 `fold` 实现完成。
4. **统一通知生产入口**：
   - 当前大量 `useXxxNotification` 钩子分散在 `src/hooks/notifs/` 中，建议建立统一的“通知事件总线”或中间件，便于集中监控、采样和 A/B 测试。
5. **增加单元测试覆盖**：
   - 重点测试 `immediate` 中断、`fold` 合并、`invalidates` 级联清除、超时链式触发等边界场景。
