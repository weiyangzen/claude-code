# frame.ts 研究文档

## 场景与职责

`frame.ts` 定义 Ink 渲染系统的帧数据结构和屏幕清除决策逻辑。它是渲染管线中连接布局计算、屏幕缓冲区和输出优化的关键类型定义模块。

### 核心职责

1. **帧类型定义**：定义 `Frame` 类型，包含屏幕缓冲区、视口、光标状态
2. **Patch 类型系统**：定义所有可能的屏幕更新操作类型
3. **帧事件类型**：定义帧性能指标和闪烁检测
4. **屏幕清除决策**：根据当前帧和前一帧决定是否需要清屏

## 功能点目的

### 1. Frame 类型

表示一帧的完整渲染状态：
```typescript
export type Frame = {
  readonly screen: Screen        // 屏幕缓冲区（字符和样式）
  readonly viewport: Size        // 终端视口尺寸
  readonly cursor: Cursor        // 光标位置和可见性
  readonly scrollHint?: ScrollHint | null  // DECSTBM 滚动优化提示
  readonly scrollDrainPending?: boolean     // 滚动待处理标志
}
```

### 2. Patch 类型系统

定义屏幕更新的原子操作：
- `stdout`：直接输出内容
- `clear`：清除指定行数
- `clearTerminal`：清屏（带原因）
- `cursorHide/Show`：光标显示控制
- `cursorMove/To`：光标移动
- `carriageReturn`：回车
- `hyperlink`：超链接开始
- `styleStr`：样式切换字符串

### 3. 帧性能指标（FrameEvent）

用于性能分析和优化：
- **各阶段耗时**：renderer、diff、optimize、write、yoga、commit
- **Yoga 统计**：访问节点数、测量次数、缓存命中
- **闪烁检测**：resize、offscreen、clear 原因

### 4. 屏幕清除决策

`shouldClearScreen` 函数决定何时需要清屏：
- **resize**：终端尺寸变化
- **offscreen**：当前或前一帧内容超出视口

## 具体技术实现

### Frame 创建

```typescript
export function emptyFrame(
  rows: number,
  columns: number,
  stylePool: StylePool,
  charPool: CharPool,
  hyperlinkPool: HyperlinkPool,
): Frame {
  return {
    screen: createScreen(0, 0, stylePool, charPool, hyperlinkPool),
    viewport: { width: columns, height: rows },
    cursor: { x: 0, y: 0, visible: true },
  }
}
```

### Patch 类型定义

```typescript
export type Patch =
  | { type: 'stdout'; content: string }
  | { type: 'clear'; count: number }
  | {
      type: 'clearTerminal'
      reason: FlickerReason
      debug?: { triggerY: number; prevLine: string; nextLine: string }
    }
  | { type: 'cursorHide' }
  | { type: 'cursorShow' }
  | { type: 'cursorMove'; x: number; y: number }
  | { type: 'cursorTo'; col: number }
  | { type: 'carriageReturn' }
  | { type: 'hyperlink'; uri: string }
  | { type: 'styleStr'; str: string }
```

### 帧性能指标

```typescript
export type FrameEvent = {
  durationMs: number
  phases?: {
    renderer: number      // DOM → yoga → screen
    diff: number          // screen diff → Patch[]
    optimize: number      // patch 合并/去重
    write: number         // Patch[] → ANSI → stdout
    patches: number       // 优化前 patch 数量
    yoga: number          // calculateLayout
    commit: number        // React reconcile
    yogaVisited: number   // layoutNode 调用次数
    yogaMeasured: number  // measureFunc 调用次数
    yogaCacheHits: number // _hasL 缓存命中
    yogaLive: number      // Yoga 节点存活数
  }
  flickers: Array<{
    desiredHeight: number
    availableHeight: number
    reason: FlickerReason
  }>
}
```

### 屏幕清除决策

```typescript
export function shouldClearScreen(
  prevFrame: Frame,
  frame: Frame,
): FlickerReason | undefined {
  // 1. 检查尺寸变化
  const didResize =
    frame.viewport.height !== prevFrame.viewport.height ||
    frame.viewport.width !== prevFrame.viewport.width
  if (didResize) return 'resize'

  // 2. 检查内容溢出
  const currentFrameOverflows = frame.screen.height >= frame.viewport.height
  const previousFrameOverflowed =
    prevFrame.screen.height >= prevFrame.viewport.height
  if (currentFrameOverflows || previousFrameOverflowed) return 'offscreen'

  return undefined
}
```

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/frame.ts`
- **导出类型**：
  - `Frame` - 帧数据结构
  - `FrameEvent` - 帧性能事件
  - `Patch` - 屏幕更新操作
  - `Diff` - Patch 数组
  - `FlickerReason` - 闪烁原因
- **导出函数**：
  - `emptyFrame` - 创建空帧
  - `shouldClearScreen` - 清屏决策

### 依赖关系

**被导入**：
- `./cursor.js` - Cursor 类型
- `./layout/geometry.js` - Size 类型
- `./render-node-to-output.js` - ScrollHint 类型
- `./screen.js` - Screen、StylePool、CharPool、HyperlinkPool 类型

**导入使用**：
```typescript
import type { Cursor } from './cursor.js'
import type { Size } from './layout/geometry.js'
import type { ScrollHint } from './render-node-to-output.js'
import {
  type CharPool,
  createScreen,
  type HyperlinkPool,
  type Screen,
  type StylePool,
} from './screen.js'
```

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `emptyFrame` | 创建空帧 | 22-34 |
| `shouldClearScreen` | 决定是否需要清屏 | 105-124 |

### 相关文件

- `src/ink/screen.ts` - Screen 类型和缓冲区管理
- `src/ink/cursor.ts` - Cursor 类型
- `src/ink/layout/geometry.ts` - Size 类型
- `src/ink/render-node-to-output.ts` - ScrollHint 定义
- `src/ink/log-update.ts` - Patch 消费和终端输出
- `src/ink/optimizer.ts` - Patch 优化

## 依赖与外部交互

### 渲染管线中的位置

```
布局计算 (Yoga)
    ↓
渲染到屏幕缓冲区 (render-node-to-output.ts)
    ↓
创建 Frame { screen, viewport, cursor }
    ↓
shouldClearScreen() - 清屏决策
    ↓
Diff 计算 (log-update.ts)
    ↓
Patch 优化 (optimizer.ts)
    ↓
Patch → ANSI 序列
    ↓
stdout 输出
```

### 与性能监控的交互

```
渲染各阶段
    ↓
记录时间戳
    ↓
生成 FrameEvent
    ↓
回调到 ink.tsx 的 onFrame
    ↓
性能分析和监控
```

## 风险、边界与改进建议

### 已知风险

1. **闪烁检测局限**：
   - 仅检测 resize 和 offscreen 两种原因
   - 其他原因导致的闪烁无法捕获

2. **性能指标精度**：
   - 使用 `Date.now()` 或 `performance.now()`，精度有限
   - 极短操作可能测量不准确

3. **内存使用**：
   - `FrameEvent` 包含大量可选字段
   - 未启用性能监控时仍分配对象

### 边界情况

1. **零尺寸帧**：`emptyFrame` 创建 0x0 的 screen
2. **相同帧比较**：`shouldClearScreen` 处理相同尺寸帧
3. **Patch 类型安全**：运行时需确保 Patch 类型正确

### 改进建议

1. **类型安全**：
   - 使用 branded types 区分不同 ID 类型
   - 添加 Patch 类型的运行时验证
   
   ```typescript
   type StyleId = number & { __brand: 'StyleId' }
   type CharId = number & { __brand: 'CharId' }
   ```

2. **性能优化**：
   - 延迟创建 FrameEvent 的 phases 对象
   - 使用对象池重用 Patch 数组

3. **功能扩展**：
   - 添加更多闪烁原因检测
   - 支持帧率限制和自适应渲染
   - 添加渲染预算（render budget）概念

4. **调试工具**：
   - 添加 Frame 可视化工具
   - Patch 序列的回放和检查
   - 性能瓶颈自动分析

5. **测试覆盖**：
   - `shouldClearScreen` 的各种边界条件
   - Patch 序列的正确性验证
   - 性能指标计算的准确性
