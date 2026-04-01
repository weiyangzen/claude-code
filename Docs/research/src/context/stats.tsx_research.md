# stats.tsx 研究文档

## 场景与职责

`stats.tsx` 为 Claude Code 提供了一套**运行时指标收集与持久化系统**。它既是一个 React Context（`StatsContext`），也是一个独立的命令式 store（`createStatsStore`）。主要职责包括：

1. **指标收集**：支持计数器（Counter）、仪表盘（Gauge）、计时器/直方图（Timer/Histogram）、集合（Set）四种指标类型。
2. **直方图 reservoir sampling**：对 Timer 指标使用 Algorithm R 进行 reservoir sampling（容量 1024），在内存受限的情况下仍能计算近似的 P50/P95/P99 分位数。
3. **会话结束时持久化**：通过监听 `process.on("exit", ...)`，在进程退出前将指标刷写到项目配置（`saveCurrentProjectConfig` 的 `lastSessionMetrics` 字段）。
4. **React 集成**：通过 `StatsProvider` 将 store 注入组件树，并提供 `useCounter`、`useGauge`、`useTimer`、`useSet` 等便捷 Hook。

## 功能点目的

### 1. `createStatsStore()`
返回一个 `StatsStore` 实例，包含以下方法：

| 方法 | 指标类型 | 说明 |
|------|----------|------|
| `increment(name, value=1)` | Counter | 累加数值 |
| `set(name, value)` | Gauge | 覆盖当前值 |
| `observe(name, value)` | Timer/Histogram | 记录样本，更新 reservoir、count、sum、min、max |
| `add(name, value)` | Set | 记录字符串去重集合，最终输出集合大小 |
| `getAll()` | — | 汇总所有指标为 `Record<string, number>` |

### 2. 直方图与 Reservoir Sampling
```ts
const RESERVOIR_SIZE = 1024

type Histogram = {
  reservoir: number[]
  count: number
  sum: number
  min: number
  max: number
}
```

- 当样本数 `< 1024` 时，直接推入 `reservoir`。
- 当样本数 `≥ 1024` 时，使用 Algorithm R：生成 `[0, count)` 随机整数 `j`，若 `j < 1024` 则替换 `reservoir[j]`。
- `getAll()` 时对 reservoir 排序，计算 P50/P95/P99。

### 3. `StatsProvider`
- 接收可选的 `store` prop，用于测试注入或外部预创建实例。
- 若未提供外部 store，则在首次渲染时通过 React Compiler 的 memo cache sentinel 调用 `createStatsStore()` 创建内部实例。
- 通过 `useEffect` 注册 `process.on("exit", flush)`，在退出时将 `store.getAll()` 写入当前项目配置的 `lastSessionMetrics`。

### 4. 便捷 Hook
| Hook | 返回 | 用途 |
|------|------|------|
| `useStats()` | `StatsStore` | 直接获取 store |
| `useCounter(name)` | `(value?: number) => void` | 绑定计数器增量回调 |
| `useGauge(name)` | `(value: number) => void` | 绑定仪表盘设置回调 |
| `useTimer(name)` | `(value: number) => void` | 绑定直方图观察回调 |
| `useSet(name)` | `(value: string) => void` | 绑定集合添加回调 |

## 具体技术实现

### Percentile 计算
```ts
function percentile(sorted: number[], p: number): number {
  const index = p / 100 * (sorted.length - 1)
  const lower = Math.floor(index)
  const upper = Math.ceil(index)
  if (lower === upper) return sorted[lower]!
  return sorted[lower]! + (sorted[upper]! - sorted[lower]!) * (index - lower)
}
```
采用线性插值法，适用于 reservoir 排序后的近似分位计算。

### 持久化流程
```ts
useEffect(() => {
  const flush = () => {
    const metrics = store.getAll()
    if (Object.keys(metrics).length > 0) {
      saveCurrentProjectConfig(current => ({
        ...current,
        lastSessionMetrics: metrics
      }))
    }
  }
  process.on("exit", flush)
  return () => process.off("exit", flush)
}, [store])
```

- 仅在 `store` 变化时重新注册监听器。
- 使用 `process.on("exit")` 而非 `beforeExit` 或 `SIGINT`，因为 Node.js 的 `exit` 事件是同步的，适合执行轻量级的配置写入。但这也意味着如果进程被 `SIGKILL` 终止，指标可能丢失。

### 编译产物特征
`StatsProvider` 和四个便捷 Hook 都已被 React Compiler 编译。`StatsProvider` 使用 `Symbol.for("react.memo_cache_sentinel")` 来懒初始化 `createStatsStore()`，确保内部 store 在组件生命周期内只创建一次。

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/stats.tsx` | 本文件，指标收集、React Context、Hook |
| `src/utils/config.ts` | `saveCurrentProjectConfig`，持久化目标 |
| `src/components/App.tsx` | 顶层注入 `StatsProvider`，可选传入外部 `stats` store |
| `src/components/Stats.tsx` | `/stats` 命令 UI，读取并展示 `lastSessionMetrics` 等历史数据 |
| `src/components/StatusLine.tsx` | 可能消费 `lastSessionMetrics` 展示简短统计 |
| `src/buddy/CompanionSprite.tsx` | 可能通过 stats Hook 记录 companion 交互 |
| `src/hooks/useManagePlugins.ts` | 使用 `useCounter` 记录插件操作 |
| `src/hooks/useScheduledTasks.ts` | 使用 `useCounter` 记录任务调度 |
| `src/hooks/useTeammateViewAutoExit.ts` | 使用 `useCounter` 记录自动退出 |
| `src/hooks/useRemoteSession.ts` | 使用 `useCounter` 记录远程会话事件 |
| `src/hooks/useSettings.ts` | 使用 `useCounter` 记录设置变更 |
| `src/hooks/useVoiceIntegration.tsx` | 使用 `useCounter` 记录语音模式使用 |
| `src/hooks/useReplBridge.tsx` | 使用 `useCounter` 记录桥接事件 |
| `src/hooks/useInboxPoller.ts` | 使用 `useCounter` 记录轮询事件 |
| `src/hooks/useGlobalKeybindings.tsx` | 使用 `useCounter` 记录快捷键使用 |
| `src/hooks/useSessionBackgrounding.ts` | 使用 `useCounter` 记录会话后台化 |
| `src/hooks/useSkillImprovementSurvey.ts` | 使用 `useCounter` 记录调查展示 |
| `src/commands/*`（如 `bridge.tsx`、`ide.tsx`、`model.tsx` 等） | 各命令组件使用便捷 Hook 记录命令调用次数 |
| `src/components/permissions/*.tsx` | 权限相关组件使用 stats Hook 记录用户决策 |

## 依赖与外部交互

- **React**：`createContext`、`useContext`、`useEffect`、`useMemo`（编译后表现为 memo cache）。
- **`../utils/config.js`**：强依赖 `saveCurrentProjectConfig` 实现持久化。
- **`process`**：Node.js 全局对象，用于注册 `exit` 事件监听器。
- **无外部网络依赖**：纯本地内存计算 + 本地配置写入。

## 风险、边界与改进建议

### 风险与边界
1. **`process.on("exit")` 的同步限制**：
   - `exit` 事件处理器中不能执行异步操作。`saveCurrentProjectConfig` 内部涉及文件 I/O（`writeFileSync` 或带锁的同步写入），虽然当前实现是同步的，但如果未来重构为异步，指标持久化将失效。
2. **`SIGKILL` / 崩溃导致数据丢失**：
   - 如果进程被强制终止或发生未捕获异常崩溃，`exit` 事件不会触发，整个会话的指标将完全丢失。
3. **`Math.random()` 的 reservoir sampling 非加密随机**：
   - Algorithm R 使用 `Math.random()`，对于指标采样来说足够，但在极端并发或安全敏感场景下，其随机性分布可能不够均匀。
4. **`getAll()` 的 O(n log n) 排序开销**：
   - 每次调用 `getAll()` 都会对所有直方图的 reservoir 数组进行 `slice().sort()`。如果 reservoir 接近 1024 且 `getAll()` 被频繁调用（如每帧），可能成为性能瓶颈。
5. **`useCounter` 等 Hook 的闭包稳定性**：
   - 编译后的 Hook 将 `name` 和 `store` 加入 memo cache。如果调用方动态变化 `name`（如基于 props），会重新创建回调函数，可能导致下游 effect 重新执行。

### 改进建议
1. **增加崩溃安全持久化**：
   - 除了 `process.on("exit")`，可定期（如每 30 秒或每 N 次指标更新）调用 `saveCurrentProjectConfig` 进行增量刷写，降低崩溃丢失风险。但需控制写盘频率，避免损坏 SSD 或触发配置锁竞争。
2. **为 `getAll()` 引入缓存**：
   - 在 `StatsStore` 内部维护一个 `lastGetAllResult` 和 `dirty` 标志，仅在指标有变化时才重新计算排序和分位数，避免重复计算。
3. **支持标签（Labels/Dimensions）**：
   - 当前指标名是纯字符串（如 `"plugin_install"`）。若未来需要按插件 ID、模型名称等维度拆分，可引入 `Record<string, string>` 标签系统，生成复合 key（如 `plugin_install{pluginId=foo}`）。
4. **提取持久化策略**：
   - 将 `process.on("exit")` 的注册逻辑从 `StatsProvider` 中抽离为一个可选的 `useStatsPersistence(statsStore)` Hook，让测试环境和 SDK 嵌入场景能够灵活关闭持久化。
5. **增加指标导出接口**：
   - 除了写入本地项目配置，可增加一个 `exportToJSON()` 或 `toOpenTelemetry()` 接口，便于集成到外部可观测性平台（如 Datadog、Prometheus）。
6. **单元测试覆盖 reservoir sampling**：
   - 对 `observe` + `getAll` 的 P50/P95/P99 计算进行大量随机数据测试，验证 Algorithm R 的近似精度是否在可接受范围内。
