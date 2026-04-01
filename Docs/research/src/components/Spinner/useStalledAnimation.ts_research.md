# `src/components/Spinner/useStalledAnimation.ts` 研究

本研究仅基于当前仓库可见的代码、配置类型、hooks、任务实现、调用链与测试文件检索结果完成；未把 `README`、`Docs`、`docs`、其他 Markdown 文档作为研究输入。

## 场景与职责

`useStalledAnimation` 是一个用于检测流式响应是否"停滞"并计算相应视觉强度的 React Hook。它的核心职责包括：

1. **检测 token 流停滞**：通过监控 `currentResponseLength` 的变化，判断自上次收到新 token 以来已经过去了多久。
2. **计算停滞强度（stalled intensity）**：当停滞时间超过 3 秒阈值后，在随后的 2 秒内将强度从 0 线性渐变到 1，用于驱动 spinner 和消息文本向红色过渡。
3. **平滑过渡**：在非 reduced motion 模式下，通过动画帧 tick 逐步逼近目标强度，避免颜色突兀跳变。
4. **抑制误报**：当 `hasActiveTools` 为真时，强制将停滞时间视为 0，避免工具执行期间触发虚假停滞状态。
5. **初始状态处理**：当 `currentResponseLength` 为 0 时，以组件挂载时间作为计时起点，确保请求发送后尚未收到首个 token 时也能正确显示等待状态。

该 Hook 是 spinner 视觉反馈系统中最重要的"状态感知"层之一，直接影响用户对模型响应速度的感知。

## 功能点目的

- **显式暴露延迟**：当模型超过 3 秒没有继续输出 token 时，spinner 和消息文本会逐渐变红，向用户传达"当前响应可能卡住或变慢"的信息。
- **避免工具执行误报**：工具调用期间通常不会有新 token，但系统仍在正常工作。`hasActiveTools` 参数确保这段时间不会触发停滞红色。
- **可访问性支持**：`reducedMotion` 模式下跳过平滑过渡，直接使用计算出的目标强度，避免持续的颜色插值动画对敏感用户造成不适。
- **与父级动画时钟解耦**：该 Hook 不自己维护 `setInterval` 或 `requestAnimationFrame`，而是接收父组件的 `time` 参数（来自 `useAnimationFrame`），确保动画速度与终端焦点状态同步（终端失焦时 Ink 的动画帧会减速）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 函数签名

`src/components/Spinner/useStalledAnimation.ts:6-14`

```typescript
export function useStalledAnimation(
  time: number,
  currentResponseLength: number,
  hasActiveTools = false,
  reducedMotion = false,
): {
  isStalled: boolean
  stalledIntensity: number
}
```

### 参数说明

| 参数 | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `time` | `number` | - | 父组件的动画时钟时间（毫秒），通常来自 `useAnimationFrame` |
| `currentResponseLength` | `number` | - | 当前响应的字符长度，用于检测是否有新 token |
| `hasActiveTools` | `boolean` | `false` | 是否有正在执行的工具，为真时重置停滞计时 |
| `reducedMotion` | `boolean` | `false` | 是否开启减少动画模式 |

### 内部 Refs

`src/components/Spinner/useStalledAnimation.ts:15-19`

```typescript
const lastTokenTime = useRef(time)
const lastResponseLength = useRef(currentResponseLength)
const mountTime = useRef(time)
const stalledIntensityRef = useRef(0)
const lastSmoothTime = useRef(time)
```

- `lastTokenTime`：上次检测到新 token 时的动画时钟时间。
- `lastResponseLength`：上一次的响应长度，用于与当前值比较。
- `mountTime`：组件首次挂载时的动画时钟时间，用于 `currentResponseLength === 0` 的初始计时。
- `stalledIntensityRef`：当前平滑过渡后的强度值。
- `lastSmoothTime`：上次执行平滑过渡计算时的动画时钟时间。

### 新 token 检测与重置

`src/components/Spinner/useStalledAnimation.ts:22-27`

```typescript
if (currentResponseLength > lastResponseLength.current) {
  lastTokenTime.current = time
  lastResponseLength.current = currentResponseLength
  stalledIntensityRef.current = 0
  lastSmoothTime.current = time
}
```

当 `currentResponseLength` 严格大于上一次记录的值时，认为收到了新 token，重置所有与停滞相关的计时和强度。

### 停滞时间计算

`src/components/Spinner/useStalledAnimation.ts:30-38`

```typescript
let timeSinceLastToken: number
if (hasActiveTools) {
  timeSinceLastToken = 0
  lastTokenTime.current = time
} else if (currentResponseLength > 0) {
  timeSinceLastToken = time - lastTokenTime.current
} else {
  timeSinceLastToken = time - mountTime.current
}
```

三种情况：
1. **`hasActiveTools` 为真**：强制 `timeSinceLastToken = 0`，并同步 `lastTokenTime`，确保工具执行期间不会进入停滞状态。
2. **`currentResponseLength > 0`**：正常计算自上次 token 以来的时间差。
3. **`currentResponseLength === 0`**：以挂载时间为起点计算等待时间，适用于请求已发送但尚未收到任何响应的阶段。

### 停滞状态与目标强度

`src/components/Spinner/useStalledAnimation.ts:42-45`

```typescript
const isStalled = timeSinceLastToken > 3000 && !hasActiveTools
const intensity = isStalled
  ? Math.min((timeSinceLastToken - 3000) / 2000, 1)
  : 0
```

- **阈值**：3000ms（3 秒）。超过此时间没有新 token 即判定为停滞。
- **渐变窗口**：2000ms（2 秒）。从判定为停滞开始，强度在 2 秒内从 0 线性增长到 1。
- **上限**：`Math.min(..., 1)` 确保强度不超过 1。

### 平滑过渡算法

`src/components/Spinner/useStalledAnimation.ts:48-67`

```typescript
if (!reducedMotion && (intensity > 0 || stalledIntensityRef.current > 0)) {
  const dt = time - lastSmoothTime.current
  if (dt >= 50) {
    const steps = Math.floor(dt / 50)
    let current = stalledIntensityRef.current
    for (let i = 0; i < steps; i++) {
      const diff = intensity - current
      if (Math.abs(diff) < 0.01) {
        current = intensity
        break
      }
      current += diff * 0.1
    }
    stalledIntensityRef.current = current
    lastSmoothTime.current = time
  }
} else {
  stalledIntensityRef.current = intensity
  lastSmoothTime.current = time
}
```

- **步长控制**：每 50ms 执行一次平滑计算。如果两次 tick 间隔超过 50ms，会按 `Math.floor(dt / 50)` 计算需要补足的步数。
- **指数逼近**：每一步向目标强度移动剩余差距的 10%（`diff * 0.1`），形成平滑的指数衰减逼近效果。
- **提前终止**：当差距小于 0.01 时直接跳到目标值，避免无限微动。
- **reducedMotion 短路**：直接同步 `stalledIntensityRef.current = intensity`，无过渡动画。

### 最终返回值

`src/components/Spinner/useStalledAnimation.ts:70-74`

```typescript
const effectiveIntensity = reducedMotion
  ? intensity
  : stalledIntensityRef.current

return { isStalled, stalledIntensity: effectiveIntensity }
```

- `reducedMotion` 模式下返回即时计算的 `intensity`。
- 正常模式下返回经过平滑过渡后的 `stalledIntensityRef.current`。

## 关键代码路径与文件引用

- `src/components/Spinner/useStalledAnimation.ts:1`
  - 导入 `useRef` from `react`。
- `src/components/Spinner/useStalledAnimation.ts:6-14`
  - Hook 函数签名与返回类型定义。
- `src/components/Spinner/useStalledAnimation.ts:15-19`
  - 五个 `useRef` 的声明与初始化。
- `src/components/Spinner/useStalledAnimation.ts:22-27`
  - 新 token 检测与重置逻辑。
- `src/components/Spinner/useStalledAnimation.ts:30-38`
  - `timeSinceLastToken` 的三分支计算。
- `src/components/Spinner/useStalledAnimation.ts:42-45`
  - `isStalled` 判定与目标 `intensity` 计算。
- `src/components/Spinner/useStalledAnimation.ts:48-67`
  - 平滑过渡算法（50ms 步长、指数逼近）。
- `src/components/Spinner/useStalledAnimation.ts:70-74`
  - `reducedMotion` 分支与最终返回。
- `src/components/Spinner/SpinnerAnimationRow.tsx:127-130`
  - `SpinnerAnimationRow` 调用 `useStalledAnimation` 的位置，传入 `hasActiveTools || leaderIsIdle` 作为抑制条件。
- `src/components/Spinner/SpinnerGlyph.tsx:49-68`
  - 使用 `stalledIntensity` 进行颜色插值的位置。
- `src/components/Spinner/GlimmerMessage.tsx:87-132`
  - 使用 `stalledIntensity` 将消息文本变红的位置。

## 依赖与外部交互

### 直接依赖

- `react`：`useRef`。

### 外部交互

- **父组件 `SpinnerAnimationRow`**：是唯一调用方。它从 `useAnimationFrame(50)` 获取 `time`，并将 `responseLengthRef.current` 作为 `currentResponseLength` 传入。同时传入 `hasActiveTools || leaderIsIdle` 来抑制误报。
- **消费者 `SpinnerGlyph` 和 `GlimmerMessage`**：接收 `stalledIntensity` 返回值，分别用于：
  - `SpinnerGlyph`：将旋转字符的颜色从主题色插值到错误红色。
  - `GlimmerMessage`：将整个消息文本的颜色插值到错误红色。
- **与 `leaderIsIdle` 的协同**：`SpinnerAnimationRow` 将 `leaderIsIdle` 与 `hasActiveTools` 合并传入，确保 leader 已经完成当前 turn、只剩 teammate 在运行时，leader 的 spinner 不会误变红。

## 风险、边界与改进建议

### 1. Render 阶段直接修改 Ref

`src/components/Spinner/useStalledAnimation.ts:22-27` 和 `30-38` 在 Hook 的函数体（即 render 阶段）中直接修改了多个 `useRef` 的 `.current`。虽然 ref 修改不会触发重渲染，但在 React 的并发模式（Concurrent Mode）或严格模式（Strict Mode）下，render 阶段可能被中断或重复执行，导致时间戳记录不准确。

**风险**：
- 在并发渲染中，如果 render 被丢弃（discarded），ref 的修改不会被回滚，可能导致 `lastTokenTime` 被错误更新。

**建议**：
- 将 ref 修改逻辑封装到 `useEffect` 中，确保只在 commit 阶段执行。或者使用 `useMemo` 配合状态提升，将时间戳管理从 render 阶段移出。

### 2. `time` 参数的时钟来源假设

该 Hook 强烈依赖父组件传入的 `time` 来自 `useAnimationFrame`。如果调用方错误地传入了 `Date.now()` 或其他非同步时钟，平滑过渡算法会失效（因为 `dt` 可能突然跳到很大或很小）。

**建议**：
- 在 Hook 的 JSDoc 中明确标注：`time` 必须是来自 `useAnimationFrame` 的单调递增时间戳，不能是 wall-clock 时间。
- 或者增加一个开发环境断言，检测 `time` 是否出现倒退。

### 3. `hasActiveTools` 与 `leaderIsIdle` 的合并传入

`SpinnerAnimationRow` 将 `hasActiveTools || leaderIsIdle` 合并后传入 `useStalledAnimation`，这意味着 Hook 内部无法区分"因为有活跃工具"和"因为 leader 空闲"两种抑制原因。如果未来需要针对这两种情况显示不同的 UI 反馈（例如工具执行时显示工具图标，leader 空闲时显示 idle 文本），当前接口会限制扩展。

**建议**：
- 考虑将 `hasActiveTools` 和 `leaderIsIdle` 作为两个独立参数传入，让 Hook 保留更多上下文信息。
- 或者将抑制逻辑上提到 `SpinnerAnimationRow`，让 `useStalledAnimation` 只负责纯粹的停滞检测。

### 4. 3 秒阈值和 2 秒渐变窗口的硬编码

`3000ms` 和 `2000ms` 是硬编码的魔法数字，没有通过配置或设置暴露给用户。虽然这对大多数场景是合理的，但在某些网络环境或模型配置下，用户可能希望调整敏感度。

**建议**：
- 评估是否需要将这两个值迁移到设置系统中（如 `spinnerStallThresholdMs` 和 `spinnerStallFadeMs`）。
- 如果保持硬编码，建议提取为具名常量并添加注释说明选择依据。

### 5. 平滑过渡的步长算法在极端帧率下的行为

当终端失焦时，`useAnimationFrame` 的 tick 间隔可能从 50ms 跳到 1000ms 以上。此时 `steps = Math.floor(dt / 50)` 可能达到 20+，循环 20 次做指数逼近。虽然计算量很小，但在极端情况下（如 tick 间隔数秒），`current += diff * 0.1` 的多次迭代会迅速收敛到目标值，视觉上可能表现为"突然跳红"。

**建议**：
- 考虑对 `steps` 设置上限（如 `Math.min(steps, 10)`），或者改用基于 `dt` 的连续插值公式（如 `current = lerp(current, intensity, 1 - Math.exp(-dt / tau))`），使过渡效果与时间差解耦。

### 6. 缺少自动化测试

`useStalledAnimation` 是一个纯逻辑非常清晰的 Hook，但当前未发现任何直接测试。以下高价值测试场景缺失：
- 新 token 到达时重置停滞状态。
- 3 秒阈值前后的 `isStalled` 切换。
- 2 秒渐变窗口内的强度线性增长。
- `hasActiveTools` 抑制逻辑。
- `reducedMotion` 模式下的即时切换。
- 长时间停滞后强度上限为 1。

**建议**：
- 使用 `@testing-library/react` 或自定义 render hook 工具编写测试。
- 由于算法依赖 `time` 参数，测试时可以通过控制 `time` 的输入值来模拟时间推进，无需真实的 `setTimeout`。
