# use-animation-frame.ts 深入研究

## 场景与职责

`useAnimationFrame` 是 Ink 终端 UI 框架中用于实现同步动画的核心 Hook。它解决了终端应用中动画组件的以下关键问题：

1. **动画同步**：多个动画实例共享同一个时钟源，确保所有动画保持同步
2. **性能优化**：当组件离开视口时自动暂停动画，减少不必要的渲染
3. **终端失焦处理**：终端失去焦点时自动降低动画频率，节省资源
4. **可暂停/恢复**：支持通过传入 `null` 来暂停动画，恢复时从当前时间继续

## 功能点目的

### 1. 共享时钟机制
- 通过 `ClockContext` 获取全局时钟实例
- 所有使用此 Hook 的组件订阅同一个时钟，确保动画帧同步
- 时钟仅在至少有一个 `keepAlive` 订阅者时才运行

### 2. 视口感知
- 使用 `useTerminalViewport` 检测组件是否在终端视口内
- 当组件滚动出视口时自动停止订阅，避免无效计算
- 重新进入视口时自动恢复

### 3. 智能节流
- 支持自定义更新间隔（默认 16ms，约 60fps）
- 通过时间差检查避免过于频繁的更新

## 具体技术实现

### 关键数据结构

```typescript
// 返回类型
[ref: (element: DOMElement | null) => void, time: number]
```

- `ref`: 回调 ref，用于附加到动画元素
- `time`: 当前动画时间（毫秒），从时钟启动开始计算

### 核心流程

1. **初始化阶段**：
   ```typescript
   const clock = useContext(ClockContext)
   const [viewportRef, { isVisible }] = useTerminalViewport()
   const [time, setTime] = useState(() => clock?.now() ?? 0)
   ```

2. **激活状态计算**：
   ```typescript
   const active = isVisible && intervalMs !== null
   ```
   只有同时满足可见且 intervalMs 不为 null 时才激活

3. **订阅时钟**：
   ```typescript
   useEffect(() => {
     if (!clock || !active) return
     
     let lastUpdate = clock.now()
     
     const onChange = (): void => {
       const now = clock.now()
       if (now - lastUpdate >= intervalMs!) {
         lastUpdate = now
         setTime(now)
       }
     }
     
     // keepAlive: true 表示此订阅会驱动时钟运行
     return clock.subscribe(onChange, true)
   }, [clock, intervalMs, active])
   ```

### 时间更新逻辑

- 使用 `lastUpdate` 记录上次更新时间
- 每次时钟 tick 时检查时间差是否达到 `intervalMs`
- 达到阈值时才更新 `time` 状态，避免不必要的 React 重渲染

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/ClockContext.tsx` | 提供共享时钟上下文，包含 `createClock` 工厂函数 |
| `src/ink/hooks/use-terminal-viewport.ts` | 提供视口检测能力 |
| `src/ink/dom.ts` | 定义 `DOMElement` 类型 |

### 调用方示例

```typescript
function Spinner() {
  const [ref, time] = useAnimationFrame(120)
  const frame = Math.floor(time / 120) % FRAMES.length
  return <Box ref={ref}>{FRAMES[frame]}</Box>
}
```

### ClockContext 关键实现

```typescript
// src/ink/components/ClockContext.tsx
export type Clock = {
  subscribe: (onChange: () => void, keepAlive: boolean) => () => void
  now: () => number
  setTickInterval: (ms: number) => void
}
```

- `subscribe`: 订阅时钟变化，`keepAlive` 为 true 时表示此订阅会保持时钟运行
- `now`: 获取当前时间（相对于时钟启动时间）
- `setTickInterval`: 设置 tick 间隔，用于失焦时降低频率

## 依赖与外部交互

### 与 ClockContext 的交互

1. **订阅机制**：
   - 使用 `Map<() => void, boolean>` 存储订阅者和 keepAlive 状态
   - 当有任何 keepAlive 订阅者时，启动 `setInterval`
   - 最后一个 keepAlive 订阅者取消时，清除 interval

2. **时间同步**：
   - 所有订阅者在同一个 tick 中收到相同的 `tickTime`
   - 暂停时返回实时时间，避免返回过时的 tickTime

### 与 useTerminalViewport 的交互

- `useTerminalViewport` 返回 `[ref, entry]`
- `entry.isVisible` 表示元素是否在视口内
- 视口检测基于 Yoga 布局计算，考虑滚动偏移

### 与终端焦点的交互

- `ClockProvider` 使用 `useTerminalFocus` 监听终端焦点状态
- 失焦时自动将 tick 间隔从 16ms 调整为 32ms（`BLURRED_TICK_INTERVAL_MS`）
- 这种调整是全局的，影响所有动画

## 风险、边界与改进建议

### 潜在风险

1. **时钟漂移**：长时间运行的动画可能累积时间误差
2. **内存泄漏**：如果 `clock.subscribe` 的清理函数未正确调用，可能导致内存泄漏
3. **并发问题**：多个组件同时设置不同的 `intervalMs` 时，时钟频率由最后一个设置决定

### 边界情况

1. **intervalMs = null**：动画暂停，时间冻结在最后一个值
2. **时钟未就绪**：`clock` 为 null 时返回时间 0
3. **快速切换**：频繁切换可见/不可见状态可能导致频繁的订阅/取消订阅

### 改进建议

1. **添加防抖**：对于频繁切换可见性的场景，可考虑添加防抖机制
2. **支持 RAF 模式**：对于需要更流畅动画的场景，可考虑支持类似 requestAnimationFrame 的模式
3. **性能监控**：添加开发模式下的性能监控，检测异常的订阅/取消频率
4. **文档完善**：添加更多使用示例，特别是关于复杂动画场景的最佳实践

### 测试建议

1. 测试时钟同步：多个组件使用不同 interval 时的时间一致性
2. 测试视口切换：快速滚动时的行为
3. 测试暂停/恢复：传入 null 后恢复的时间连续性
4. 测试内存泄漏：大量组件挂载/卸载后的内存占用
