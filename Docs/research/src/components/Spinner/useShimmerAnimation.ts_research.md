# `src/components/Spinner/useShimmerAnimation.ts` 研究

本研究仅基于当前仓库可见的代码、配置类型、hooks、任务实现、调用链与测试文件检索结果完成；未把 `README`、`Docs`、`docs`、其他 Markdown 文档作为研究输入。

## 场景与职责

`useShimmerAnimation` 是一个专门用于计算文本 "shimmer"（扫光）动画索引的 React Hook。它的核心职责包括：

1. **驱动 shimmer 动画时钟**：基于 `useAnimationFrame` 提供一个按固定间隔 tick 的动画时间源。
2. **计算当前 glimmer 索引**：根据动画时间、消息宽度和 spinner 模式，计算扫光效果当前应该高亮的字符位置索引。
3. **停滞时取消订阅优化**：当 `isStalled` 为真时，向 `useAnimationFrame` 传入 `null` 以取消动画帧订阅，避免不必要的 CPU/渲染开销。
4. **支持不同模式的动画行为**：`requesting` 模式使用更快的扫光速度（50ms），其他模式使用较慢速度（200ms），且扫光方向相反。

该 Hook 被 `SpinnerAnimationRow` 和权限相关组件（`BashPermissionRequest`、`PermissionExplanation`）复用，是跨模块的共享动画原语。

## 功能点目的

- **性能优化**：通过向 `useAnimationFrame` 传 `null` 在停滞时取消订阅，避免 20fps 的定时器在不可见状态下持续触发。注释特别指出，即使调用方没有附加 `ref`（如条件渲染导致 JSX 未挂载），`useTerminalViewport` 的默认 `isVisible:true` 也无法触发视口暂停，因此 `isStalled ? null` 是唯一的停止机制。
- **模式差异化动画**：`requesting` 模式（向上发送请求）使用更快的扫光和正向循环，其他模式使用较慢的反向扫光，通过视觉差异强化用户对当前流状态的感知。
- **跨模块复用**：将 shimmer 动画逻辑从 `SpinnerAnimationRow` 中抽离出来，使权限解释、自动审批等待等非 spinner 页面也能使用相同的动画效果。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 函数签名

`src/components/Spinner/useShimmerAnimation.ts:6-10`

```typescript
export function useShimmerAnimation(
  mode: SpinnerMode,
  message: string,
  isStalled: boolean,
): [ref: (element: DOMElement | null) => void, glimmerIndex: number]
```

### 参数说明

| 参数 | 类型 | 说明 |
|-----|------|------|
| `mode` | `SpinnerMode` | 当前 spinner 流模式，决定 shimmer 速度和方向 |
| `message` | `string` | 需要应用 shimmer 效果的文本消息 |
| `isStalled` | `boolean` | 是否处于停滞状态，为真时取消动画订阅并返回固定索引 |

### 返回值

返回一个元组 `[ref, glimmerIndex]`：
- `ref`：用于绑定到 Ink `DOMElement` 的回调 ref，由 `useAnimationFrame` 提供。
- `glimmerIndex`：当前 shimmer 高亮窗口的中心索引，用于 `GlimmerMessage` 或 `ShimmerChar` 的渲染。

### 核心算法

`src/components/Spinner/useShimmerAnimation.ts:11-30`

```typescript
const glimmerSpeed = mode === 'requesting' ? 50 : 200
const [ref, time] = useAnimationFrame(isStalled ? null : glimmerSpeed)
const messageWidth = useMemo(() => stringWidth(message), [message])

if (isStalled) {
  return [ref, -100]
}

const cyclePosition = Math.floor(time / glimmerSpeed)
const cycleLength = messageWidth + 20

if (mode === 'requesting') {
  return [ref, (cyclePosition % cycleLength) - 10]
}
return [ref, messageWidth + 10 - (cyclePosition % cycleLength)]
```

#### 速度选择

- `requesting` 模式：`glimmerSpeed = 50ms`，与 `SpinnerAnimationRow` 的主动画时钟频率一致，扫光移动较快。
- 其他模式：`glimmerSpeed = 200ms`，扫光移动较慢，更柔和。

#### 周期计算

- `cycleLength = messageWidth + 20`，意味着扫光高亮窗口会在消息文本前后各延伸 10 个字符宽度的空白区域，形成"进入-穿过-离开"的完整动画周期。
- `cyclePosition = Math.floor(time / glimmerSpeed)`，表示从动画开始到现在经历了多少个 glimmer tick。

#### 方向与索引计算

- `requesting` 模式：`(cyclePosition % cycleLength) - 10`
  - 正向移动，从 `-10` 开始逐渐增大，穿过消息文本，最后到达 `messageWidth + 10`。
- 其他模式：`messageWidth + 10 - (cyclePosition % cycleLength)`
  - 反向移动，从 `messageWidth + 10` 开始逐渐减小，穿过消息文本，最后到达 `-10`。

#### 停滞状态处理

- 当 `isStalled` 为 `true` 时：
  - `useAnimationFrame(null)` 取消订阅，停止接收动画帧。
  - 提前返回 `glimmerIndex = -100`，该值远小于任何可能的字符索引，确保 `GlimmerMessage` 中的 `shimmerStart >= messageWidth || shimmerEnd < 0` 条件成立，从而不渲染任何 shimmer 高亮效果。

### `useAnimationFrame` 的行为

`useAnimationFrame` 来自 `../../ink.js`，是 Ink 框架提供的 Hook：
- 当传入数字时，以该数字为间隔（毫秒）触发回调，返回当前累计时间 `time`。
- 当传入 `null` 时，取消订阅，不再触发回调。
- 返回的 `ref` 用于绑定到 DOM 元素，Ink 内部可能用它来做视口可见性检测（`useTerminalViewport`）。

## 关键代码路径与文件引用

- `src/components/Spinner/useShimmerAnimation.ts:1-4`
  - 导入依赖：`useMemo`、`stringWidth`、`useAnimationFrame`、`SpinnerMode`。
- `src/components/Spinner/useShimmerAnimation.ts:6-10`
  - Hook 函数签名与参数定义。
- `src/components/Spinner/useShimmerAnimation.ts:11-13`
  - `glimmerSpeed` 模式分支与 `useAnimationFrame` 调用。
- `src/components/Spinner/useShimmerAnimation.ts:14`
  - `messageWidth` 的 `useMemo` 缓存。
- `src/components/Spinner/useShimmerAnimation.ts:16-22`
  - 停滞状态短路返回。
- `src/components/Spinner/useShimmerAnimation.ts:24-30`
  - `cyclePosition`、`cycleLength` 计算与方向分支。
- `src/components/Spinner/SpinnerAnimationRow.tsx:132-138`
  - `SpinnerAnimationRow` 内部也内联了类似的 shimmer 计算逻辑，但直接使用共享的 `time` 而不是调用 `useShimmerAnimation`。
- `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx:36-44`
  - 外部复用 `useShimmerAnimation` 的位置之一。
- `src/components/permissions/PermissionExplanation.tsx:10-39`
  - 外部复用 `useShimmerAnimation` 的位置之二。

## 依赖与外部交互

### 直接依赖

- `react`：`useMemo`。
- `../../ink/stringWidth.js`：`stringWidth`，计算消息文本的显示宽度。
- `../../ink.js`：`useAnimationFrame`、`DOMElement` 类型。
- `./types.js`：`SpinnerMode` 类型（**注意：当前源码中 `types.js`/`types.ts` 缺失**）。

### 外部交互

- **被 `SpinnerAnimationRow` 间接替代**：`SpinnerAnimationRow.tsx` 没有直接调用 `useShimmerAnimation`，而是内联了几乎相同的计算逻辑（`src/components/Spinner/SpinnerAnimationRow.tsx:132-138`），因为它已经拥有共享的 `time` 变量。这导致同一套算法存在两处实现，增加了维护成本。
- **被权限组件直接复用**：`BashPermissionRequest` 和 `PermissionExplanation` 直接导入并使用 `useShimmerAnimation`，用于在等待用户审批时显示 shimmer 动画文本。

## 风险、边界与改进建议

### 1. 算法重复实现（DRY 原则违反）

`useShimmerAnimation.ts` 和 `SpinnerAnimationRow.tsx:132-138` 中存在几乎完全相同的 shimmer 索引计算逻辑。`SpinnerAnimationRow` 之所以内联实现，是因为它已经从 `useAnimationFrame(50)` 获取了 `time`，不想再引入一个额外的 Hook 调用。

**风险**：
- 如果未来需要调整 shimmer 速度、周期长度或方向逻辑，必须同时修改两处代码，容易遗漏。

**建议**：
- 将 `glimmerIndex` 的计算逻辑提取为一个纯函数，例如 `computeGlimmerIndex(mode, time, messageWidth)`，供 `useShimmerAnimation` 和 `SpinnerAnimationRow` 共同使用。
- 或者重构 `SpinnerAnimationRow` 使其直接调用 `useShimmerAnimation`，将 `time` 作为参数传入（但这会改变 Hook 的调用规则，需要谨慎）。

### 2. `types.js` 缺失的连锁影响

`useShimmerAnimation.ts` 导入 `SpinnerMode`  from `./types.js`，但该文件在当前源码目录中不存在。这会导致：
- TypeScript 类型检查失败。
- 构建系统可能报错或回退到 `any` 类型。
- 维护者无法直接查看 `SpinnerMode` 的合法取值。

**建议**：
- 立即恢复 `src/components/Spinner/types.ts`，至少包含：
  ```typescript
  export type SpinnerMode = 'requesting' | 'thinking' | 'responding' | 'tool-input' | 'tool-use'
  ```

### 3. `isStalled` 取消订阅的边界情况

注释提到："if the caller never attaches `ref` (e.g. conditional JSX), `useTerminalViewport` stays at its initial `isVisible:true` and the viewport-pause never kicks in, so this is the only stop mechanism."

这意味着 `useShimmerAnimation` 的停滞停止机制是最后一道防线。但如果调用方在 `isStalled` 从 `false` 变为 `true` 之前从未挂载过该 Hook（例如组件被条件渲染完全跳过了），则不存在订阅需要取消。这个设计是合理的，但需要确保所有调用方都正确传递 `isStalled` 状态。

**建议**：
- 在 `useShimmerAnimation` 的 JSDoc 中补充说明：`isStalled` 不仅是视觉状态，也是性能开关，调用方必须准确传递。

### 4. `messageWidth` 缓存的粒度

`messageWidth` 使用 `useMemo(() => stringWidth(message), [message])` 缓存。由于 `message` 在单个 turn 内通常是稳定的，这个缓存是有效的。但如果 `message` 频繁变化（例如逐字增长的流式文本），`useMemo` 的重新计算开销与直接调用 `stringWidth` 相差无几。

**建议**：
- 当前实现已经足够好，无需改动。但如果未来需要为极长消息（如数百字符）做 shimmer，可以考虑将 `messageWidth` 的计算也下沉到 `useAnimationFrame` 的 tick 回调中按需计算，避免 render 阶段重复执行。

### 5. `-100` 魔法数字的语义不透明

`isStalled` 时返回 `glimmerIndex = -100`，这个值的唯一目的是让 `GlimmerMessage` 的边界检查条件 `shimmerEnd < 0` 成立。但 `-100` 是一个没有明确语义关联的魔法数字，新维护者可能不理解为什么不是 `-1` 或 `Number.MIN_SAFE_INTEGER`。

**建议**：
- 定义一个具名常量，例如 `const HIDDEN_GLIMMER_INDEX = -100`，并在注释中说明其用途："A value far outside any valid character index to ensure no shimmer segment is rendered."
- 或者与 `GlimmerMessage` 协商一个更明确的协议，例如 `null` 或 `undefined` 表示 "无 shimmer"。

### 6. 缺少 Hook 级别的单元测试

当前未发现针对 `useShimmerAnimation` 的单元测试。该 Hook 的纯计算部分（`cyclePosition`、`cycleLength`、方向分支）非常适合做纯函数测试，但动画帧相关的部分需要 `@testing-library/react-hooks` 或类似工具。

**建议**：
- 提取纯计算逻辑后，为其编写单元测试，覆盖：
  - `requesting` 模式的正向扫光索引序列。
  - 非 `requesting` 模式的反向扫光索引序列。
  - `isStalled` 状态下的返回值。
  - 空消息（`messageWidth = 0`）时的周期长度和索引行为。
