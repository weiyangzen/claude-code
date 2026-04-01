# 研究文档：src/utils/fpsTracker.ts

## 场景与职责

`fpsTracker.ts` 提供了一个极简的**帧率指标追踪器**，用于在 UI 渲染过程中记录每次渲染耗时，并计算平均帧率（average FPS）和低 1% 帧率（low 1% FPS）。它主要服务于性能监控场景，如 cost tracker 和 React 组件的性能指标采集。

该模块 intentionally 保持零依赖（除 `performance.now()` 外），可以在主线程、worker 或测试环境中即插即用。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `FpsTracker` | 类，记录渲染耗时并计算 FPS 指标。 |
| `FpsMetrics` | 类型，`{ averageFps: number; low1PctFps: number }`。 |
| `record(durationMs)` | 记录单次渲染耗时，并记录当前 `performance.now()` 时间戳。 |
| `getMetrics()` | 计算并返回平均 FPS 和低 1% FPS；若数据不足返回 `undefined`。 |

## 具体技术实现

### 类结构

```ts
export class FpsTracker {
  private frameDurations: number[] = []
  private firstRenderTime: number | undefined
  private lastRenderTime: number | undefined

  record(durationMs: number): void {
    const now = performance.now()
    if (this.firstRenderTime === undefined) {
      this.firstRenderTime = now
    }
    this.lastRenderTime = now
    this.frameDurations.push(durationMs)
  }
}
```

- `record` 接收的是**单次渲染耗时**（毫秒），而非帧间隔。调用方（如 React 渲染性能 hook）负责测量并传入。
- `firstRenderTime` 和 `lastRenderTime` 记录的是调用 `record` 时的 wall-clock 时间，用于计算总时间跨度。

### 指标计算（`getMetrics`）

```ts
getMetrics(): FpsMetrics | undefined {
  if (this.frameDurations.length === 0 || totalTimeMs <= 0) {
    return undefined
  }

  const totalFrames = this.frameDurations.length
  const averageFps = totalFrames / (totalTimeMs / 1000)

  const sorted = this.frameDurations.slice().sort((a, b) => b - a)
  const p99Index = Math.max(0, Math.ceil(sorted.length * 0.01) - 1)
  const p99FrameTimeMs = sorted[p99Index]!
  const low1PctFps = p99FrameTimeMs > 0 ? 1000 / p99FrameTimeMs : 0

  return {
    averageFps: Math.round(averageFps * 100) / 100,
    low1PctFps: Math.round(low1PctFps * 100) / 100,
  }
}
```

- **Average FPS**：`总帧数 / 总时间(秒)`。注意这里的“总时间”是 `lastRenderTime - firstRenderTime`，而不是各帧耗时的累加。这意味着如果两次渲染之间有明显的空闲间隔，average FPS 会被拉低。
- **Low 1% FPS**：将帧耗时按降序排列，取第 1 百分位（p99）的耗时，然后 `1000 / p99FrameTimeMs` 换算成 FPS。这个指标反映的是最卡顿的那 1% 帧的表现。
- 结果保留两位小数。

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/cost-tracker.ts` | `FpsTracker` | 成本追踪器中的性能监控。 |
| `src/costHook.ts` | `FpsTracker` | React hook 中的性能数据采集。 |
| `src/dialogLaunchers.tsx` | `FpsTracker` | 对话框启动性能监控。 |
| `src/context/fpsMetrics.tsx` | `FpsMetrics` (type) | FPS 指标 React Context。 |
| `src/interactiveHelpers.tsx` | `FpsTracker` | 交互辅助函数中的性能追踪。 |
| `src/components/App.tsx` | `FpsTracker` | 应用根组件渲染性能。 |
| `src/main.tsx` | `FpsTracker` | 入口性能初始化。 |
| `src/replLauncher.tsx` | `FpsTracker` | REPL 启动性能。 |

### 被调用方

- Web API `performance.now()`：获取高精度时间戳。

## 依赖与外部交互

- 无网络依赖。
- 无持久化。
- 无外部库依赖。
- 依赖全局 `performance` 对象；在 Node.js 测试环境中可能需要 `global.performance = { now: Date.now }` 的 polyfill。

## 风险、边界与改进建议

### 风险

1. **内存泄漏**：`frameDurations` 数组会无限增长，直到 `FpsTracker` 实例被垃圾回收。若实例生命周期极长（如应用级别的单例）且 `record` 被频繁调用（如每帧都记录），数组可能积累数百万条目。
2. **Average FPS 计算歧义**：`totalTimeMs` 使用的是两次 `record` 调用之间的 wall-clock 时间，而非实际渲染耗时总和。如果调用方在渲染结束后立即 `record`，且两次渲染间隔很短，该指标近似正确；但如果间隔包含大量空闲时间，average FPS 会被低估。
3. **Low 1% 命名与实现的反向性**：指标名为 `low1PctFps`，但计算方式是基于**最高**的 1% 帧耗时（p99 耗时）。这在游戏/图形行业中是常见做法（即 "1% low" = 最差的 1% 帧），但命名对非专业人士可能产生困惑。
4. **零耗时帧**：若传入 `durationMs = 0`，`p99FrameTimeMs` 可能为 0，导致 `low1PctFps` 被计算为 0（代码中有 `> 0` 保护，返回 0）。

### 边界

- 不主动限制 `frameDurations` 数组长度，调用方需自行决定何时创建新实例或重置。
- 不区分渲染帧和逻辑帧，所有传入的 `durationMs` 都被平等对待。
- 无自动采样或节流机制。

### 改进建议

1. **数组长度上限**：增加 `maxSamples` 参数（如 10000），当 `frameDurations.length` 超过阈值时，采用滑动窗口或分桶聚合，避免无限增长。
2. **更精确的 average FPS**：提供两种 average 计算方式：
   - `wallClockAverageFps`（当前实现）
   - `renderTimeAverageFps`（`totalFrames / (sum(durations) / 1000)`）
3. **更丰富的百分位**：除 1% low 外，可补充 `p50`（中位数）和 `p95`，形成更完整的性能画像。
4. **测试覆盖**：补充对大量样本、零耗时、以及 `performance.now()` 非单调递增边界情况的单元测试。
