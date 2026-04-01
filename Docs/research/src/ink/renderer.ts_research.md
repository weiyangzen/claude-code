# renderer.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`renderer.ts` 是 Ink 渲染系统的入口模块，负责协调 Yoga 布局计算、屏幕缓冲区管理和渲染流程控制。它是连接 React 协调器（reconciler）与底层终端输出的关键层。

### 1.2 核心职责
- **渲染器创建**：工厂函数创建配置好的渲染器实例
- **帧管理**：管理前后缓冲帧（front/back frame）的切换
- **布局验证**：验证 Yoga 布局计算结果的有效性
- **渲染协调**：调用 `renderNodeToOutput` 执行实际渲染
- **滚动优化**：处理 ScrollBox 的排水（drain）和滚动提示

### 1.3 使用场景
- Ink 实例初始化时创建渲染器
- 每帧渲染时调用渲染器生成新的帧数据
- 处理终端大小变化、内容更新等触发重绘的场景

---

## 2. 功能点目的

### 2.1 RenderOptions - 渲染选项

```typescript
export type RenderOptions = {
  frontFrame: Frame      // 前一帧（用于 blit 优化）
  backFrame: Frame       // 后一帧（渲染目标）
  isTTY: boolean         // 是否 TTY 模式
  terminalWidth: number  // 终端列数
  terminalRows: number   // 终端行数
  altScreen: boolean     // 是否备用屏幕模式
  prevFrameContaminated: boolean  // 前一帧是否被污染
}
```

**设计目的**：
- 封装渲染所需的全部上下文信息
- 支持双缓冲机制（front/back frame）
- 区分主屏幕和备用屏幕的不同处理逻辑

### 2.2 prevFrameContaminated - 帧污染标记

**设计目的**：
- 标记前一帧的屏幕缓冲区是否被修改（选择覆盖层）
- 标记是否被重置为空白（alt-screen 进入/resize/SIGCONT）
- 标记是否被重置为 0×0（forceRedraw）
- 污染时禁用 blit 优化，防止复制脏数据

### 2.3 Renderer 类型

```typescript
export type Renderer = (options: RenderOptions) => Frame
```

**设计目的**：
- 函数式接口，便于测试和模拟
- 纯函数语义：相同输入产生相同输出
- 无状态设计，所有状态通过参数传递

---

## 3. 具体技术实现

### 3.1 createRenderer 工厂函数（行 31-178）

**函数签名**：
```typescript
export default function createRenderer(
  node: DOMElement,      // Yoga 根节点
  stylePool: StylePool,  // 样式池（共享）
): Renderer
```

**闭包状态**（行 37）：
```typescript
let output: Output | undefined  // 跨帧复用的 Output 实例
```

**设计决策**：
- Output 实例跨帧复用，保留 charCache（字符缓存）
- 缓存 tokenize + grapheme clustering 结果
- 大多数行在帧间不变，缓存命中率高

### 3.2 渲染器内部逻辑详解

#### 阶段 1：解构选项（行 38-46）
```typescript
const { frontFrame, backFrame, isTTY, terminalWidth, terminalRows } = options
const prevScreen = frontFrame.screen
const backScreen = backFrame.screen
const charPool = backScreen.charPool
const hyperlinkPool = backScreen.hyperlinkPool
```

**关键设计**：
- pools 从 backFrame 读取（而非闭包捕获）
- 支持代际重置：pools 可能在帧间被替换

#### 阶段 2：布局验证（行 48-82）

**有效性检查**（行 52-61）：
```typescript
const computedHeight = node.yogaNode?.getComputedHeight()
const computedWidth = node.yogaNode?.getComputedWidth()

const hasInvalidHeight =
  computedHeight === undefined ||
  !Number.isFinite(computedHeight) ||
  computedHeight < 0
const hasInvalidWidth = /* 类似 */
```

**无效处理**（行 63-82）：
- 记录调试日志（`--debug` 标志可见）
- 返回空帧（0×0 屏幕）
- 防止 RangeError 崩溃

#### 阶段 3：高度计算与钳制（行 84-104）

```typescript
const width = Math.floor(node.yogaNode.getComputedWidth())
const yogaHeight = Math.floor(node.yogaNode.getComputedHeight())

// Alt-screen: 屏幕缓冲区高度 = 终端行数
const height = options.altScreen ? terminalRows : yogaHeight
```

**Alt-screen 高度钳制**（行 97-104）：
```typescript
if (options.altScreen && yogaHeight > terminalRows) {
  logForDebugging(
    `alt-screen: yoga height ${yogaHeight} > terminalRows ${terminalRows} — ` +
    `something is rendering outside <AlternateScreen>. Overflow clipped.`,
    { level: 'warn' }
  )
}
```

**设计原因**：
- `<AlternateScreen>` 包裹的子树应该填满终端高度
- 如果内容渲染为兄弟节点（bug），yogaHeight > terminalRows
- 钳制防止虚拟/物理光标失步

#### 阶段 4：屏幕与 Output 准备（行 105-112）

```typescript
const screen = backScreen ?? createScreen(width, height, stylePool, charPool, hyperlinkPool)

if (output) {
  output.reset(width, height, screen)
} else {
  output = new Output({ width, height, stylePool, screen })
}
```

**设计决策**：
- 优先复用 backScreen（双缓冲）
- Output.reset 清理操作队列，保留 charCache
- charCache 大小限制：>16384 时清空

#### 阶段 5：重置渲染状态（行 114-116）

```typescript
resetLayoutShifted()
resetScrollHint()
resetScrollDrainNode()
```

**作用**：
- 清除上一帧的布局位移标记
- 清除滚动提示（将在渲染过程中重新计算）
- 清除滚动排水节点

#### 阶段 6：绝对定位移除处理（行 128-135）

```typescript
const absoluteRemoved = consumeAbsoluteRemovedFlag()
renderNodeToOutput(node, output, {
  prevScreen: absoluteRemoved || options.prevFrameContaminated ? undefined : prevScreen,
})
```

**设计原因**：
- 绝对定位节点可能覆盖非兄弟节点（如覆盖 ScrollBox）
- 移除后，blit 会恢复已移除节点的像素
- 检测到绝对定位移除时，禁用 blit

#### 阶段 7：获取渲染结果（行 137）

```typescript
const renderedScreen = output.get()
```

**副作用**：
- 执行所有排队操作（write/blit/clear/shift）
- 填充 Screen 缓冲区
- 计算 damage 区域

#### 阶段 8：滚动排水处理（行 139-144）

```typescript
const drainNode = getScrollDrainNode()
if (drainNode) markDirty(drainNode)
```

**设计原因**：
- 渲染清除了 scrollbox.dirty，下一帧根 blit 会跳过该子树
- markDirty 向上遍历祖先，确保下一帧继续处理
- 在渲染后执行，避免被 renderNodeToOutput 的 dirty 清除覆盖

#### 阶段 9：返回帧数据（行 146-176）

```typescript
return {
  scrollHint: options.altScreen ? getScrollHint() : null,
  scrollDrainPending: drainNode !== null,
  screen: renderedScreen,
  viewport: {
    width: terminalWidth,
    height: options.altScreen ? terminalRows + 1 : terminalRows,
  },
  cursor: {
    x: 0,
    y: options.altScreen ? Math.max(0, Math.min(screen.height, terminalRows) - 1) : screen.height,
    visible: !isTTY || screen.height === 0,
  },
}
```

**关键设计**：

1. **scrollHint**：仅 alt-screen 有效（行 147）
   - 主屏幕使用 log-update 的增量更新，不需要硬件滚动

2. **viewport.height 特殊处理**（行 160）：
   ```typescript
   height: options.altScreen ? terminalRows + 1 : terminalRows
   ```
   - Alt-screen：假 viewport 高度 = rows + 1
   - 防止 `shouldClearScreen` 的 `screen.height >= viewport.height` 触发
   - Alt-screen 内容正好 `rows` 高，但从不滚动

3. **光标位置**（行 162-175）：
   - Alt-screen：钳制在视口内（`terminalRows - 1`）
   - 防止 `screen.height === terminalRows` 时触发 LF 滚动
   - 主屏幕：光标在内容底部（`screen.height`）

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

```
ink.tsx (Ink 类)
    ↓
onRender() 调度渲染
    ↓
renderer({ frontFrame, backFrame, ...options })  // createRenderer 返回的函数
    ↓
  ├─ renderNodeToOutput(root, output, { prevScreen })
  │     ↓
  │   递归渲染 DOM 树
  │
  ├─ output.get() → Screen
  │
  └─ 返回 Frame { screen, viewport, cursor, scrollHint, scrollDrainPending }
```

### 4.2 关键依赖

| 文件 | 用途 |
|------|------|
| `./dom.js` | DOMElement 类型，markDirty |
| `./frame.js` | Frame 类型定义 |
| `./node-cache.js` | consumeAbsoluteRemovedFlag |
| `./output.js` | Output 类 |
| `./render-node-to-output.js` | 核心渲染逻辑 |
| `./screen.js` | createScreen, StylePool |
| `../utils/debug.js` | logForDebugging |

### 4.3 类型定义详解

```typescript
export type RenderOptions = {
  frontFrame: Frame           // 前一帧（只读引用）
  backFrame: Frame            // 后一帧（将被修改）
  isTTY: boolean              // 影响光标可见性
  terminalWidth: number       // 终端宽度（列）
  terminalRows: number        // 终端高度（行）
  altScreen: boolean          // 影响高度计算和光标位置
  prevFrameContaminated: boolean  // 影响 blit 优化
}

export type Renderer = (options: RenderOptions) => Frame
```

---

## 5. 依赖与外部交互

### 5.1 内部模块依赖图

```
renderer.ts
├── dom.ts
├── frame.ts
├── node-cache.ts
├── output.ts
├── render-node-to-output.ts
│   ├── node-cache.ts
│   ├── output.ts
│   └── ...
├── screen.ts
└── ../utils/debug.ts
```

### 5.2 与 Ink 主类的交互

```typescript
// ink.tsx 中的使用
private renderer: Renderer

constructor(options: Options) {
  // ...
  this.renderer = createRenderer(this.rootNode, this.stylePool)
}

private onRender = () => {
  const frame = this.renderer({
    frontFrame: this.frontFrame,
    backFrame: this.backFrame,
    isTTY: this.options.stdout.isTTY || false,
    terminalWidth: this.terminalColumns,
    terminalRows: this.terminalRows,
    altScreen: this.altScreenActive,
    prevFrameContaminated: this.prevFrameContaminated,
  })
  // ...处理 frame，交换缓冲...
}
```

### 5.3 双缓冲机制

```
帧 N:   frontFrame (显示) ← 前一帧
        backFrame  (渲染目标) → 渲染后成为 frontFrame

帧 N+1: frontFrame (显示) = 前一帧的 backFrame
        backFrame  (新的渲染目标) = 前一帧的 frontFrame
```

**交换逻辑**（ink.tsx）：
```typescript
[this.frontFrame, this.backFrame] = [this.backFrame, this.frontFrame]
```

---

## 6. 风险、边界与改进建议

### 6.1 已知边界情况

1. **无效布局值处理**（行 52-82）
   - Yoga 计算前 `getComputedHeight()` 返回 NaN
   - 负数或 Infinity 值会导致 RangeError
   - 返回空帧作为降级处理

2. **Alt-screen 高度溢出**（行 97-104）
   - 内容渲染在 `<AlternateScreen>` 外时触发
   - 记录警告日志帮助诊断
   - 溢出内容被静默裁剪

3. **光标位置边界**（行 170-172）
   ```typescript
   y: options.altScreen
     ? Math.max(0, Math.min(screen.height, terminalRows) - 1)
     : screen.height
   ```
   - Alt-screen：确保光标在视口内
   - 防止最后一行的光标恢复触发 LF

### 6.2 潜在风险

| 风险 | 严重程度 | 说明 |
|------|----------|------|
| Output 状态累积 | 中 | charCache 永不清理，可能内存增长 |
| 布局验证遗漏 | 低 | 某些无效 Yoga 状态可能未捕获 |
| 竞态条件 | 低 | prevFrameContaminated 标记的同步 |
| 性能退化 | 低 | 大内容时 layoutShifted 导致全量渲染 |

### 6.3 改进建议

1. **缓存大小限制**
   ```typescript
   // 当前：charCache 在 Output.reset 中清理
   // 建议：更细粒度的缓存策略
   if (output.charCache.size > MAX_CACHE_SIZE) {
     // LRU 清理或全量清空
   }
   ```

2. **布局验证增强**
   ```typescript
   // 建议：添加更多 Yoga 状态检查
   if (node.yogaNode.getDisplay() === LayoutDisplay.None) {
     // 提前返回优化
   }
   ```

3. **性能监控**
   ```typescript
   // 建议：添加渲染阶段计时
   const t0 = performance.now()
   renderNodeToOutput(...)
   const renderTime = performance.now() - t0
   if (renderTime > SLOW_RENDER_THRESHOLD) {
     logForDebugging(`Slow render: ${renderTime.toFixed(2)}ms`)
   }
   ```

4. **错误恢复**
   ```typescript
   // 建议：渲染失败时的降级策略
   try {
     renderNodeToOutput(...)
   } catch (e) {
     logError('Render failed, returning empty frame', e)
     return emptyFrame(...)
   }
   ```

5. **类型安全**
   ```typescript
   // 当前：多处使用 ! 非空断言
   // 建议：更严格的运行时检查
   const yogaNode = node.yogaNode
   if (!yogaNode) {
     throw new Error('Yoga node required for rendering')
   }
   ```

### 6.4 测试建议

- **单元测试**：
  - 无效 Yoga 尺寸的降级行为
  - altScreen 高度钳制
  - prevFrameContaminated 对 blit 的影响
  - 光标位置计算

- **集成测试**：
  - 双缓冲正确性
  - 滚动排水触发
  - 帧数据完整性

- **性能测试**：
  - 大内容渲染时间
  - 缓存命中率
  - 内存使用增长
