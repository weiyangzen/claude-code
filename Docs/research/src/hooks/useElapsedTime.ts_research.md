# useElapsedTime.ts 研究文档

## 场景与职责

`useElapsedTime` 是一个用于**实时显示格式化经过时间**的 React Hook。它基于 `useSyncExternalStore` 实现，以指定的间隔（默认 1 秒）订阅时间变化，并将经过的毫秒数格式化为人类可读的字符串（如 "1m 23s"）。

该 Hook 被广泛应用于需要展示任务运行时长的 UI 组件中：
- `TeammateSpinnerLine.tsx`： teammate 任务运行时间
- `*DetailDialog.tsx`（Dream、InProcessTeammate、AsyncAgent、RemoteSession）：各类详情对话框中的耗时展示
- `WizardNavigationFooter.tsx`：向导流程中的步骤耗时

## 功能点目的

1. **实时更新经过时间**：当 `isRunning = true` 时，以固定间隔触发重新计算和渲染。

2. **支持暂停时间扣除**：通过 `pausedMs` 参数，可以扣除任务暂停期间的时间，得到“实际运行时间”。

3. **支持冻结结束时间**：通过 `endTime` 参数，可以在任务完成后冻结显示的时间。这样即使观众在任务结束很久后查看，也不会看到从 startTime 到当前时间的总时长。

4. **高效渲染**：使用 `useSyncExternalStore` 而非 `useState` + `setInterval`，避免不必要的 React 状态更新开销，且能更好地与 React 18 的并发特性配合。

## 具体技术实现

### 源码实现

```ts
import { useCallback, useSyncExternalStore } from 'react'
import { formatDuration } from '../utils/format.js'

export function useElapsedTime(
  startTime: number,
  isRunning: boolean,
  ms: number = 1000,
  pausedMs: number = 0,
  endTime?: number,
): string {
  const get = () =>
    formatDuration(Math.max(0, (endTime ?? Date.now()) - startTime - pausedMs))

  const subscribe = useCallback(
    (notify: () => void) => {
      if (!isRunning) return () => {}
      const interval = setInterval(notify, ms)
      return () => clearInterval(interval)
    },
    [isRunning, ms],
  )

  return useSyncExternalStore(subscribe, get, get)
}
```

### 设计要点

- **`useSyncExternalStore` 的使用**：
  - `subscribe`：注册一个 `setInterval`，当 `isRunning` 变化时自动重新订阅/清理
  - `getSnapshot`（`get`）：每次 React 需要读取值时调用，直接基于当前时间计算
  - `getServerSnapshot`（也是 `get`）：SSR 场景下使用相同的计算逻辑

- **时间计算**：
  ```ts
  (endTime ?? Date.now()) - startTime - pausedMs
  ```
  - 若 `endTime` 存在，使用 `endTime` 作为终点，时间冻结
  - 否则使用 `Date.now()`，实时推进
  - 减去 `pausedMs` 得到净运行时间
  - `Math.max(0, ...)` 防止负数（虽然正常场景不应出现）

- **`formatDuration`**（`src/utils/format.ts`）：
  - `< 1s`：显示为 `0.xs`（保留 1 位小数）
  - `< 1min`：显示为整数秒（如 `23s`）
  - `>= 1min`：显示为 `Xm Ys`，支持天、小时级展开
  - 处理了秒进位到分、分进位到时的边界情况

### 使用示例

在 `TeammateSpinnerLine.tsx` 中：
```ts
const elapsed = useElapsedTime(startTime, isRunning, 1000, pausedMs)
```

在 `DreamDetailDialog.tsx` 等详情页中：
```ts
const elapsed = useElapsedTime(task.startTime, task.status === 'running', 1000, task.pausedMs, task.endTime)
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useElapsedTime.ts` | 本 Hook 实现 |
| `src/utils/format.ts` | `formatDuration`、时间格式化工具 |
| `src/components/Spinner/TeammateSpinnerLine.tsx` | 调用方：teammate 任务运行时间 |
| `src/components/tasks/DreamDetailDialog.tsx` | 调用方：Dream 任务详情耗时 |
| `src/components/tasks/InProcessTeammateDetailDialog.tsx` | 调用方：In-process teammate 详情耗时 |
| `src/components/tasks/AsyncAgentDetailDialog.tsx` | 调用方：Async agent 详情耗时 |
| `src/components/tasks/RemoteSessionDetailDialog.tsx` | 调用方：Remote session 详情耗时 |
| `src/components/wizard/WizardNavigationFooter.tsx` | 调用方：向导页脚耗时 |

## 依赖与外部交互

### 内部依赖
- **React**：`useCallback`、`useSyncExternalStore`
- **工具函数**：`formatDuration`

### 无外部交互
该 Hook 是纯本地计算，不涉及网络、文件系统或外部进程。

## 风险、边界与改进建议

### 风险与边界

1. **`setInterval` 的精度问题**：JavaScript 的 `setInterval` 在事件循环繁忙时可能漂移。对于需要精确到毫秒级的计时（如竞速、严格超时），1 秒间隔的精度足够，但如果未来需要更精细的显示（如 100ms 刷新），会增加渲染频率。

2. **`Date.now()` 与系统时间**：`get` 函数每次读取都调用 `Date.now()`，如果用户调整系统时间（如 NTP 同步、手动修改），显示的时间会突然跳变。对于长时间运行的任务，这种跳变可能让用户困惑。

3. **`pausedMs` 的维护责任在调用方**：Hook 本身不追踪暂停区间，只接收一个最终的 `pausedMs` 数值。调用方需要自行累加所有暂停时间，如果计算错误，显示的时间也会错误。

4. **`useSyncExternalStore` 的 `subscribe` 返回清理函数**：当前实现中，当 `isRunning` 从 `true` → `false` 时，`subscribe` 被重新调用，旧 interval 被清理，新 subscribe 立即返回空清理函数。这是正确的，但如果 `ms` 频繁变化（如从 1000 变到 500），会导致 interval 的频繁重建。

5. **SSR 水合一致性**：`getServerSnapshot` 和 `getSnapshot` 使用相同的 `Date.now()`。在 SSR 场景下，服务端渲染时的时间与客户端水合时的时间可能不同，导致水合不匹配（hydration mismatch）。虽然 Claude Code 是终端应用，不涉及浏览器 SSR，但如果该 Hook 被复用到其他渲染目标（如 Web），需要注意。

### 改进建议

1. **使用 `performance.now()` 替代 `Date.now()`**：
   - `performance.now()` 不受系统时间调整影响，且精度更高（微秒级）。
   - 需要相应地将 `startTime`、`endTime`、`pausedMs` 也改为基于 `performance.now()` 的单调时钟。
   - 示例：
     ```ts
     const get = () =>
       formatDuration(Math.max(0, (endTime ?? performance.now()) - startTime - pausedMs))
     ```

2. **支持自定义格式化选项**：当前强制使用 `formatDuration` 的默认行为。可以暴露一个可选的 `formatter` 参数，让调用方自定义显示格式：
   ```ts
   export function useElapsedTime(
     startTime: number,
     isRunning: boolean,
     ms: number = 1000,
     pausedMs: number = 0,
     endTime?: number,
     formatter: (ms: number) => string = formatDuration,
   ): string
   ```

3. **自动暂停追踪（可选）**：可以考虑增加一个更高级的 Hook（如 `useElapsedTimeWithPause`），内部维护 `isRunning` 状态变化的时间戳，自动计算 `pausedMs`，减轻调用方负担。

4. **节流/防抖优化**：如果 `ms` 设置得很小（如 100ms），且该 Hook 被大量组件同时使用，会导致大量 `setInterval`。可以考虑使用一个全局的共享 ticker（如基于 `requestAnimationFrame` 或全局 `setInterval`），各组件订阅同一 ticker，减少定时器数量。

5. **增加 `mostSignificantOnly` 选项透传**：`formatDuration` 支持 `mostSignificantOnly` 选项（如只显示 `1m` 而不显示 `1m 23s`），但 `useElapsedTime` 没有透传。可以通过增加可选参数来支持：
   ```ts
   formatOptions?: Parameters<typeof formatDuration>[1]
   ```
