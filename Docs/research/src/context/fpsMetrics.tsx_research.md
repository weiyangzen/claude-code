# fpsMetrics.tsx 研究文档

## 场景与职责

`fpsMetrics.tsx` 为 Claude Code 的终端 UI 提供**帧率指标（FPS Metrics）的 React 上下文传递通道**。由于 FPS 追踪器（`FpsTracker`）实例通常创建在 React 树之外（如 `REPL.tsx` 或 `cost-tracker.ts`），该 Context 的职责是将一个**只读的 getter 函数**注入到组件树中，供需要显示或上报帧率数据的组件消费。

核心设计原则：
- **不持有状态**：Context value 是一个稳定的函数引用 `() => FpsMetrics | undefined`，因此 Provider 重渲染不会导致消费者重渲染。
- **懒求值**：消费者仅在需要时才调用 getter 读取最新指标。

## 功能点目的

### 1. `FpsMetricsProvider`
接收 `getFpsMetrics` 函数和 `children`，将其作为 Context value 向下传递。实现极简，无额外逻辑。

### 2. `useFpsMetrics`
返回当前 Context 中的 getter 函数。调用方可按需获取 `{ averageFps, low1PctFps }`。

## 具体技术实现

### 类型定义
```ts
type FpsMetricsGetter = () => FpsMetrics | undefined

// 来自 ../utils/fpsTracker.js
type FpsMetrics = {
  averageFps: number
  low1PctFps: number
}
```

### 关键流程
1. **实例化与注入**：
   - `src/screens/REPL.tsx` 创建 `FpsTracker` 实例，并在渲染循环中调用 `tracker.record(durationMs)` 记录每帧耗时。
   - `REPL.tsx` 将 `() => tracker.getMetrics()` 作为 `getFpsMetrics` prop 传给 `App` 组件。
   - `App.tsx` 用 `FpsMetricsProvider` 包裹整个应用树：
     ```tsx
     <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
       <StatsProvider store={stats}>...</StatsProvider>
     </FpsMetricsProvider>
     ```

2. **消费端读取**：
   - `src/cost-tracker.ts`（或相关成本追踪模块）通过 `useFpsMetrics()` 获取 getter，在合适的时机读取 FPS 数据用于性能分析或遥测上报。

### 编译产物特征
文件已被 React Compiler 编译，`_c(3)` 缓存了 Provider 的 JSX 创建结果，确保 `children` 和 `getFpsMetrics` 未变化时不重新创建 `<FpsMetricsContext.Provider>` 节点。

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/fpsMetrics.tsx` | 本文件，定义 Context、Provider、Hook |
| `src/utils/fpsTracker.ts` | 依赖类型 `FpsMetrics` 和 `FpsTracker` 类 |
| `src/components/App.tsx` | 顶层组装，将 `getFpsMetrics` 注入 Provider |
| `src/screens/REPL.tsx` | FPS 追踪器实例化方，传入 getter |
| `src/cost-tracker.ts` | 消费者之一，读取 FPS 指标 |

## 依赖与外部交互

- **React**：`createContext`、`useContext`。
- **`../utils/fpsTracker.js`**：类型依赖，运行时无直接耦合。
- **无 AppState 依赖**：完全独立于应用状态管理。

## 风险、边界与改进建议

### 风险与边界
1. **Getter 返回 undefined 的常态**：在应用启动初期或 tracker 尚未记录任何帧时，`getFpsMetrics()` 会返回 `undefined`。消费者必须做好空值处理，否则可能产生 `NaN` 或崩溃。
2. **无订阅/无实时性**：该 Context 只提供“拉取”能力，不提供订阅。如果消费者想在 UI 上实时展示 FPS，需要自行建立轮询或依赖其他机制。
3. **编译产物与源码差异**：React Compiler 编译后的产物中，条件缓存逻辑覆盖了原本简单的 JSX 返回，调试时容易误判为存在复杂状态逻辑。

### 改进建议
1. **考虑转换为订阅模式**：如果未来需要在 UI 常驻显示 FPS（如 Debug 浮层），可将 Context value 改为 `FpsMetrics | undefined` 并结合 `useSyncExternalStore` 订阅 tracker 变化，但需权衡重渲染成本。
2. **增加 JSDoc 提示**：在 `useFpsMetrics` 上标注“返回的是 getter 函数，调用后才得到指标”，降低新开发者的理解成本。
3. **统一性能指标入口**：若后续增加更多性能指标（如内存、渲染耗时），可将 `fpsMetrics.tsx` 泛化为 `PerformanceMetricsContext`，避免 Context 碎片化。
