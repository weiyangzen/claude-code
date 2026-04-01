# SpinnerAnimationRow.tsx 研究文档

## 场景与职责

`SpinnerAnimationRow.tsx` 是 Spinner 组件体系中**最核心的动画控制组件**，负责管理所有与动画时钟相关的状态和渲染。它将高频动画更新（50ms 周期）与低频应用状态更新解耦，显著提升渲染性能。

### 架构定位

```
SpinnerWithVerb (低频更新: ~25x/turn)
    ↓ 传递稳定引用
SpinnerAnimationRow (高频更新: ~383x/turn @ 50ms)
    ↓ 管理
├── SpinnerGlyph (旋转动画)
├── GlimmerMessage (微光效果)
└── 状态文本 (计时器、token 计数、thinking 状态)
```

### 核心职责

1. **动画时钟管理**: 通过 `useAnimationFrame(50)` 订阅 50ms 动画时钟
2. **性能隔离**: 将高频动画渲染与父组件的静态布局分离
3. **状态聚合**: 整合停滞检测、微光动画、token 计数、计时器等状态
4. **自适应布局**: 根据终端宽度动态决定显示哪些状态元素

---

## 功能点目的

### 1. 动画时钟驱动系统

**useAnimationFrame(50)**
- 订阅 50ms 间隔的动画时钟
- 当 `reducedMotion` 为 true 时传入 `null` 暂停动画
- 返回 `[viewportRef, time]`，其中 `time` 是累积的动画时间

### 2. 停滞检测集成

**useStalledAnimation**
```typescript
const { isStalled, stalledIntensity } = useStalledAnimation(
  time,
  currentResponseLength,
  hasActiveTools || leaderIsIdle,
  reducedMotion
);
```
- 3 秒无新 token 触发停滞检测
- 2 秒内从正常颜色渐变到红色
- 有活跃工具或 leader 空闲时重置计时器

### 3. 微光动画计算

**Glimmer Index 计算**
```typescript
const glimmerSpeed = mode === 'requesting' ? 50 : 200;
const cycleLength = glimmerMessageWidth + 20;
const cyclePosition = Math.floor(time / glimmerSpeed);
const glimmerIndex = reducedMotion ? -100 
  : isStalled ? -100 
  : mode === 'requesting' 
    ? cyclePosition % cycleLength - 10
    : glimmerMessageWidth + 10 - cyclePosition % cycleLength;
```

- **requesting 模式**: 微光从左向右移动（速度 50ms/步）
- **其他模式**: 微光从右向左移动（速度 200ms/步）
- **停滞时**: 隐藏微光（`glimmerIndex = -100`）

### 4. Tool-use 闪烁效果

```typescript
const flashOpacity = reducedMotion ? 0 
  : mode === 'tool-use' 
    ? (Math.sin(time / 1000 * Math.PI) + 1) / 2
    : 0;
```

- 使用正弦波生成 0-1 之间的平滑闪烁值
- 周期为 2 秒（`1000ms * π`）

### 5. Token 计数动画

**平滑递增算法**
```typescript
const gap = currentResponseLength - tokenCounterRef.current;
if (gap > 0) {
  let increment;
  if (gap < 70) {
    increment = 3;
  } else if (gap < 200) {
    increment = Math.max(8, Math.ceil(gap * 0.15));
  } else {
    increment = 50;
  }
  tokenCounterRef.current = Math.min(tokenCounterRef.current + increment, currentResponseLength);
}
```

- 小差距（<70）: 每次 +3，精细动画
- 中等差距（70-200）: 按比例递增
- 大差距（>200）: 每次 +50，快速追赶

### 6. Thinking 状态微光

```typescript
const THINKING_DELAY_MS = 3000;
const THINKING_GLOW_PERIOD_S = 2;
const thinkingElapsedSec = (time - THINKING_DELAY_MS) / 1000;
const thinkingOpacity = time < THINKING_DELAY_MS ? 0 
  : (Math.sin(thinkingElapsedSec * Math.PI * 2 / THINKING_GLOW_PERIOD_S) + 1) / 2;
```

- 延迟 3 秒后开始微光
- 2 秒周期正弦波动

### 7. 自适应宽度布局

**渐进式显示决策**
```typescript
const availableSpace = columns - messageWidth - 5;
let showThinking = wantsThinking && availableSpace > thinkingWidthValue;
// ...
const usedAfterThinking = showThinking ? thinkingWidthValue + sep : 0;
const showTimer = wantsTimerAndTokens && availableSpace > usedAfterThinking + timerWidth;
```

优先级：消息 > Thinking > 计时器 > Token 计数

---

## 具体技术实现

### 数据结构

```typescript
export type SpinnerAnimationRowProps = {
  // 动画输入
  mode: SpinnerMode;
  reducedMotion: boolean;
  hasActiveTools: boolean;
  responseLengthRef: React.RefObject<number>;

  // 消息（回合内稳定）
  message: string;
  messageColor: keyof Theme;
  shimmerColor: keyof Theme;
  overrideColor?: keyof Theme | null;

  // 计时器 refs（稳定引用）
  loadingStartTimeRef: React.RefObject<number>;
  totalPausedMsRef: React.RefObject<number>;
  pauseStartTimeRef: React.RefObject<number | null>;

  // 显示标志
  spinnerSuffix?: string | null;
  verbose: boolean;
  columns: number;

  // 队友相关
  hasRunningTeammates: boolean;
  teammateTokens: number;
  foregroundedTeammate: InProcessTeammateTaskState | undefined;
  leaderIsIdle?: boolean;

  // Thinking 状态
  thinkingStatus: 'thinking' | number | null;
  effortSuffix: string;
};
```

### 关键常量

```typescript
const SEP_WIDTH = stringWidth(' · ');           // 分隔符宽度: 3
const THINKING_BARE_WIDTH = stringWidth('thinking'); // 9
const SHOW_TOKENS_AFTER_MS = 30_000;            // 30秒后显示token计数

// Thinking 微光颜色
const THINKING_INACTIVE = { r: 153, g: 153, b: 153 };
const THINKING_INACTIVE_SHIMMER = { r: 185, g: 185, b: 185 };
```

### 时间计算逻辑

**经过时间计算**
```typescript
const now = Date.now();
const elapsedTimeMs = pauseStartTimeRef.current !== null
  ? pauseStartTimeRef.current - loadingStartTimeRef.current - totalPausedMsRef.current
  : now - loadingStartTimeRef.current - totalPausedMsRef.current;
```

**回合起始时间追踪**
```typescript
const derivedStart = now - elapsedTimeMs;
const turnStartRef = useRef(derivedStart);
if (!hasRunningTeammates || derivedStart < turnStartRef.current) {
  turnStartRef.current = derivedStart;
}
```

- 用于队友场景：leader 的 elapsedTimeMs 可能跳动，但 turnStartRef 保持最早开始时间

### 子组件：SpinnerModeGlyph

```typescript
function SpinnerModeGlyph({ mode }: { mode: SpinnerMode }) {
  switch (mode) {
    case "tool-input":
    case "tool-use":
    case "responding":
    case "thinking":
      return <Box width={2}><Text dimColor>{figures.arrowDown}</Text></Box>;
    case "requesting":
      return <Box width={2}><Text dimColor>{figures.arrowUp}</Text></Box>;
  }
}
```

- 输入模式（requesting）: 向上箭头
- 输出/处理模式: 向下箭头

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `figures` | 默认导入 | 终端图形符号（箭头） |
| `../../ink/stringWidth.js` | `stringWidth` | 字符宽度计算 |
| `../../ink.js` | `Box`, `Text`, `useAnimationFrame` | Ink 组件和动画钩子 |
| `../../tasks/InProcessTeammateTask/types.js` | `InProcessTeammateTaskState` | 队友状态类型 |
| `../../utils/format.js` | `formatDuration`, `formatNumber` | 格式化工具 |
| `../../utils/ink.js` | `toInkColor` | 颜色转换 |
| `../../utils/theme.js` | `Theme` | 主题类型 |
| `../design-system/Byline.js` | `Byline` | 内联元数据显示 |
| `./GlimmerMessage.js` | `GlimmerMessage` | 消息微光效果 |
| `./SpinnerGlyph.js` | `SpinnerGlyph` | 旋转动画 |
| `./types.js` | `SpinnerMode` | 模式类型 |
| `./useStalledAnimation.js` | `useStalledAnimation` | 停滞检测钩子 |
| `./utils.js` | `interpolateColor`, `toRGBColor` | 颜色工具 |

### 被调用方

| 文件路径 | 使用方式 |
|---------|---------|
| `Spinner.tsx` (推测) | 主 Spinner 组件调用 |
| `REPL.tsx` (推测) | REPL 界面集成 |

### 源码位置

```
src/components/Spinner/
├── SpinnerAnimationRow.tsx   # 本文件
├── SpinnerGlyph.tsx          # 旋转动画子组件
├── GlimmerMessage.tsx        # 消息微光子组件
├── useStalledAnimation.ts    # 停滞检测钩子
├── utils.ts                  # 工具函数
└── types.ts                  # 类型定义（缺失）
```

---

## 依赖与外部交互

### 核心依赖详解

**1. useAnimationFrame**
```typescript
const [viewportRef, time] = useAnimationFrame(reducedMotion ? null : 50);
```
- 来自 `../../ink.js`
- 50ms 间隔调用，累积时间作为 `time` 返回
- 传入 `null` 时暂停动画（用于 reduced motion）

**2. useStalledAnimation**
```typescript
const { isStalled, stalledIntensity } = useStalledAnimation(
  time,
  currentResponseLength,
  hasActiveTools || leaderIsIdle,
  reducedMotion
);
```
- 基于动画时钟的停滞检测
- 3 秒阈值，2 秒渐变

**3. 队友状态系统**
```typescript
type InProcessTeammateTaskState = {
  identity: {
    agentName: string;
    color: string;
  };
  progress?: {
    tokenCount: number;
  };
  isIdle: boolean;
};
```

### 数据流

```
Parent Component
    ↓ Props
SpinnerAnimationRow
    ├─ useAnimationFrame(50) → time
    ├─ useStalledAnimation → { isStalled, stalledIntensity }
    ├─ useMemo → glimmerMessageWidth
    ├─ useRef → tokenCounterRef, turnStartRef
    ↓
计算派生值:
    ├─ frame (旋转帧)
    ├─ glimmerIndex (微光位置)
    ├─ flashOpacity (闪烁强度)
    ├─ timerText (格式化时间)
    ├─ totalTokens (token 总数)
    ├─ thinkingShimmerColor (thinking 微光色)
    ↓
渲染子组件:
    ├─ <SpinnerGlyph />
    ├─ <GlimmerMessage />
    └─ 状态文本
```

---

## 风险、边界与改进建议

### 已知风险

1. **类型文件缺失**
   - `src/components/Spinner/types.ts` 不存在
   - `SpinnerMode` 类型定义缺失
   - `InProcessTeammateTaskState` 类型可能也需要确认

2. **复杂度过高**
   - 单文件承担过多职责（动画、布局、状态聚合）
   - 265 行代码，逻辑分支复杂

3. **魔法数字过多**
   ```typescript
   const gap < 70        // 小差距阈值
   const gap < 200       // 中差距阈值
   const SHOW_TOKENS_AFTER_MS = 30_000  // 30秒
   const THINKING_DELAY_MS = 3000       // 3秒
   const THINKING_GLOW_PERIOD_S = 2     // 2秒
   ```

4. **时间计算依赖 refs**
   - 使用 refs 存储时间状态，可能导致时序问题
   - 暂停/恢复逻辑复杂，容易出错

### 边界情况

| 场景 | 行为 |
|-----|------|
| `columns` 过小 | 渐进隐藏 thinking、timer、tokens |
| `reducedMotion = true` | 暂停所有动画，使用静态值 |
| `leaderIsIdle = true` | 抑制停滞检测 |
| `foregroundedTeammate` 存在 | 显示队友名称和 token 计数 |
| `thinkingStatus` 为数字 | 显示 "thought for Xs" |

### 改进建议

1. **类型文件恢复**
   ```typescript
   // types.ts
   export type SpinnerMode = 'requesting' | 'thinking' | 'tool-use' | 'tool-input' | 'responding';
   export type InProcessTeammateTaskState = { ... };
   ```

2. **常量集中管理**
   ```typescript
   // constants.ts
   export const ANIMATION = {
     FRAME_INTERVAL_MS: 50,
     GLIMMER_SPEED_REQUESTING: 50,
     GLIMMER_SPEED_OTHER: 200,
     STALLED_THRESHOLD_MS: 3000,
     STALLED_FADE_DURATION_MS: 2000,
     SHOW_TOKENS_AFTER_MS: 30000,
     THINKING_DELAY_MS: 3000,
     THINKING_GLOW_PERIOD_S: 2,
   } as const;
   ```

3. **逻辑拆分**
   - 提取 `useGlimmerAnimation` 钩子
   - 提取 `useTokenCounter` 钩子
   - 提取 `useThinkingShimmer` 钩子

4. **性能优化**
   - 使用 `useMemo` 缓存更多计算结果
   - 考虑使用 CSS 动画替代 React 状态驱动

5. **测试覆盖**
   - 时间计算逻辑单元测试
   - 布局决策边界测试
   - 动画状态转换测试

---

## 附录：编译后代码特点

### React Compiler 优化模式

**多条件缓存检查**
```typescript
if ($[0] !== flashOpacity || $[1] !== message || $[2] !== messageColor || ...) {
  // 重新计算
} else {
  // 复用缓存
}
```

**早期返回优化**
```typescript
bb0: {
  // 计算逻辑
  if (condition) {
    t2 = <Component />;
    break bb0;
  }
}
```

**分段渲染缓存**
字素分割、宽度计算、分段字符串都被独立缓存，形成多级缓存策略。

### 源码结构

原始源码约 230 行，编译后约 265 行，增加：
- 缓存数组操作
- 条件分支显式化
- Source map 注释
