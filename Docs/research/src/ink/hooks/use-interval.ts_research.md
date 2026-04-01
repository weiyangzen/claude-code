# use-interval.ts 深入研究

## 场景与职责

`use-interval.ts` 提供了两个基于共享时钟的定时器 Hook：`useAnimationTimer` 和 `useInterval`。与 `usehooks-ts` 中的 `useInterval` 不同，这些 Hook 基于 Ink 的共享 `ClockContext`，将所有定时器整合到单一的唤醒源，提高性能并确保时间同步。

## 功能点目的

### 1. useAnimationTimer
- 返回当前时钟时间，按给定间隔更新
- 作为非 keepAlive 订阅者，不会单独保持时钟运行
- 适用于纯时间计算（如闪烁位置、帧索引）

### 2. useInterval
- 基于共享时钟的间隔回调执行
- 支持暂停（传入 `null` 作为 intervalMs）
- 所有定时器共享一个时钟，减少唤醒次数

### 3. 性能优化
- 单一 setInterval 驱动所有订阅者
- 避免多个组件各自创建定时器
- 终端失焦时自动降低时钟频率

## 具体技术实现

### useAnimationTimer 实现

```typescript
export function useAnimationTimer(intervalMs: number): number {
  const clock = useContext(ClockContext)
  const [time, setTime] = useState(() => clock?.now() ?? 0)

  useEffect(() => {
    if (!clock) return

    let lastUpdate = clock.now()

    const onChange = (): void => {
      const now = clock.now()
      if (now - lastUpdate >= intervalMs) {
        lastUpdate = now
        setTime(now)
      }
    }

    // keepAlive: false — 不会单独保持时钟运行
    return clock.subscribe(onChange, false)
  }, [clock, intervalMs])

  return time
}
```

### useInterval 实现

```typescript
export function useInterval(
  callback: () => void,
  intervalMs: number | null,
): void {
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  const clock = useContext(ClockContext)

  useEffect(() => {
    if (!clock || intervalMs === null) return

    let lastUpdate = clock.now()

    const onChange = (): void => {
      const now = clock.now()
      if (now - lastUpdate >= intervalMs) {
        lastUpdate = now
        callbackRef.current()
      }
    }

    return clock.subscribe(onChange, false)
  }, [clock, intervalMs])
}
```

### 关键技术点

1. **keepAlive: false**：
   - 两个 Hook 都使用 `keepAlive: false`
   - 表示它们不会单独驱动时钟运行
   - 依赖其他 keepAlive 订阅者（如动画组件）来驱动时钟

2. **时间差检查**：
   - 使用 `now - lastUpdate >= intervalMs` 判断是否需要更新
   - 避免过于频繁的回调执行
   - 累积误差由时钟本身处理

3. **callbackRef 模式**：
   - `useInterval` 使用 ref 存储最新回调
   - 避免在回调变化时重新订阅
   - 订阅保持稳定，减少开销

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/ClockContext.tsx` | 提供共享时钟上下文 |

### 与 useAnimationFrame 的区别

| 特性 | useAnimationFrame | useAnimationTimer | useInterval |
|------|------------------|-------------------|-------------|
| 返回 | [ref, time] | time | void |
| keepAlive | true | false | false |
| 用途 | 驱动动画 | 获取时间 | 执行回调 |
| 视口感知 | 是 | 否 | 否 |

### Clock 接口

```typescript
export type Clock = {
  subscribe: (onChange: () => void, keepAlive: boolean) => () => void
  now: () => number
  setTickInterval: (ms: number) => void
}
```

### 时钟订阅机制

```typescript
// ClockContext.tsx 中的订阅逻辑
subscribe(onChange, keepAlive) {
  subscribers.set(onChange, keepAlive)
  updateInterval()  // 根据 keepAlive 状态决定是否启动 interval
  return () => {
    subscribers.delete(onChange)
    updateInterval()
  }
}
```

## 依赖与外部交互

### 与 ClockContext 的交互

1. **订阅流程**：
   - Hook 挂载时调用 `clock.subscribe(onChange, false)`
   - 返回清理函数在卸载时调用

2. **时间获取**：
   - 通过 `clock.now()` 获取当前时间
   - 时间值相对于时钟启动时间

3. **频率控制**：
   - 时钟频率由 `ClockProvider` 根据终端焦点状态调整
   - 聚焦时：16ms（约 60fps）
   - 失焦时：32ms（约 30fps）

### 与 useAnimationFrame 的协作

- `useAnimationFrame` 是 keepAlive 订阅者，驱动时钟运行
- `useAnimationTimer` 和 `useInterval` 是非 keepAlive 订阅者，被动接收 tick
- 当所有 keepAlive 订阅者都卸载后，时钟停止，非 keepAlive 订阅者也不再接收 tick

## 风险、边界与改进建议

### 潜在风险

1. **时钟停止**：
   - 如果没有 keepAlive 订阅者，时钟停止
   - 非 keepAlive 订阅者的回调不会执行
   - 可能导致 `useInterval` 看起来"不工作"

2. **累积误差**：
   - `lastUpdate = now` 的更新方式可能导致累积误差
   - 长时间运行后，实际间隔可能偏离设定值

3. **内存泄漏**：
   - 如果清理函数未正确执行，订阅可能残留

### 边界情况

1. **intervalMs = null**：
   - `useInterval` 暂停执行
   - 恢复时从当前时间继续

2. **时钟未就绪**：
   - `clock` 为 null 时 Hook 不执行任何操作
   - 需要确保组件在 ClockProvider 内使用

3. **极小 intervalMs**：
   - 小于时钟 tick 间隔的 intervalMs 实际上按 tick 间隔执行
   - 不会提高执行频率

### 改进建议

1. **添加 keepAlive 选项**：
   ```typescript
   useInterval(callback, intervalMs, { keepAlive: true })
   ```
   允许重要定时器保持时钟运行

2. **误差补偿**：
   ```typescript
   // 使用目标时间而非上次实际时间
   const targetTime = lastUpdate + intervalMs
   if (now >= targetTime) {
     lastUpdate = targetTime  // 而非 now
     callback()
   }
   ```

3. **添加立即执行选项**：
   ```typescript
   useInterval(callback, intervalMs, { immediate: true })
   ```
   挂载时立即执行一次回调

4. **性能监控**：
   - 在开发模式下跟踪订阅者数量
   - 警告过多的订阅者

### 测试建议

1. 测试时钟停止/启动场景
2. 测试长时间运行的定时精度
3. 测试多个 Hook 同时使用的性能
4. 测试暂停/恢复功能
5. 测试组件快速挂载/卸载
