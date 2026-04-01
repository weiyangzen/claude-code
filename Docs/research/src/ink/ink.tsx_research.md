# ink.tsx 深度研究文档

## 1. 场景与职责

`ink.tsx` 是 Ink 终端 UI 框架的核心引擎类，负责协调 React 组件树与终端输出之间的完整渲染管道。它是整个 Ink 实例的中央控制器，承担着以下核心职责：

- **React  reconciler 容器管理**：创建并维护 react-reconciler 的 FiberRoot 容器，同步 React 组件树到 Ink 的 DOM 抽象层
- **终端渲染调度**：通过节流机制（~60fps）调度渲染帧，协调 Yoga 布局计算、屏幕缓冲区和终端输出
- **双缓冲帧管理**：维护 front/back 双帧缓冲区，实现增量 diff 渲染
- **终端模式管理**：处理 alt-screen 切换、鼠标追踪、键盘扩展模式等终端状态
- **文本选择与剪贴板**：实现全屏模式下的文本选择、复制到剪贴板（OSC 52）
- **搜索高亮**：支持屏幕内容的搜索匹配高亮
- **输入事件分发**：将 stdin 输入事件分发到 React 组件树

该文件是 Ink 框架的"心脏"，所有渲染流程、终端交互和状态管理都通过 `Ink` 类进行协调。

## 2. 功能点目的

### 2.1 核心渲染循环

**目的**：将 React 组件树高效地渲染到终端，最小化输出字节数以降低终端模拟器的解析负担。

**实现机制**：
- 使用 `scheduleRender`（throttle 包装）控制渲染频率，约 16ms（60fps）间隔
- 通过 `onRender()` 方法执行完整渲染流程：Yoga 布局 → DOM 渲染 → 屏幕 diff → 终端输出
- 双缓冲机制：`frontFrame`（当前显示）和 `backFrame`（下一帧），通过交换实现增量更新

### 2.2 Alt-Screen 管理

**目的**：支持全屏终端应用（如 vim、less 风格），避免滚动历史被污染。

**实现机制**：
- `setAltScreenActive()` 控制 alt-screen 状态切换
- `resetFramesForAltScreen()` 在进入 alt-screen 时重置帧缓冲区为全屏空白
- 特殊处理：在 alt-screen 模式下，光标始终锚定到 (0,0)，每帧以 CSI H 开始
- 支持鼠标追踪（mode-1003）和焦点事件（DECSET 1004）

### 2.3 文本选择系统

**目的**：在全屏模式下提供类似终端原生选择的文本选择体验。

**实现机制**：
- `SelectionState` 对象跟踪选择状态（anchor/focus/dragging）
- 支持字符/单词/行三种选择模式（双击选词、三击选行）
- `applySelectionOverlay()` 在屏幕缓冲区上直接应用选择样式
- `copySelection()` 通过 OSC 52 序列将文本复制到系统剪贴板
- 处理滚动跟随：当内容滚动时，选择区域自动跟随（`shiftSelectionForFollow`）

### 2.4 搜索高亮

**目的**：在屏幕内容上高亮显示搜索匹配项。

**实现机制**：
- `setSearchHighlight()` 设置搜索查询词
- `applySearchHighlight()` 扫描屏幕缓冲区，对所有匹配单元格应用反色样式
- `setSearchPositions()` 支持基于位置的高亮（用于 VML 消息搜索）
- `scanElementSubtree()` 扫描 DOM 子树获取匹配位置

### 2.5 终端恢复与信号处理

**目的**：处理终端会话的生命周期事件（SIGCONT、resize、暂停/恢复）。

**实现机制**：
- `handleResume()`：SIGCONT 信号处理，重新进入 alt-screen 或重置主屏幕帧
- `handleResize()`：终端尺寸变化同步处理，触发 Yoga 重新布局
- `enterAlternateScreen()` / `exitAlternateScreen()`：外部编辑器（如 git commit）集成
- `reassertTerminalModes()`：长时间空闲后重新断言终端模式（处理 tmux attach/ssh reconnect）

### 2.6 原生光标定位

**目的**：将物理终端光标定位到输入框位置，支持 IME 预编辑文本和屏幕阅读器。

**实现机制**：
- `setCursorDeclaration` 接收来自 `useDeclaredCursor` Hook 的光标声明
- `onRender()` 中根据声明的 DOM 节点位置和相对偏移计算绝对屏幕坐标
- 主屏幕和 alt-screen 使用不同的光标定位策略

## 3. 具体技术实现

### 3.1 关键数据结构

```typescript
// 帧结构（frame.ts）
type Frame = {
  readonly screen: Screen           // 屏幕缓冲区（packed Int32Array）
  readonly viewport: Size           // 视口尺寸
  readonly cursor: Cursor           // 光标位置
  readonly scrollHint?: ScrollHint  // DECSTBM 滚动优化提示
  readonly scrollDrainPending?: boolean  // 待处理滚动标志
}

// 屏幕缓冲区（screen.ts）
type Screen = Size & {
  cells: Int32Array                 // 打包的单元格数据 [charId, packed(styleId|hyperlinkId|width)]
  cells64: BigInt64Array            // 用于批量清除的 64 位视图
  charPool: CharPool                // 字符串驻留池
  hyperlinkPool: HyperlinkPool      // 超链接驻留池
  emptyStyleId: number
  damage: Rectangle | undefined     // 损坏区域跟踪
  noSelect: Uint8Array              // 不可选择单元格位图
  softWrap: Int32Array              // 软换行标记
}

// 选择状态（selection.ts）
type SelectionState = {
  anchor: Point | null              // 选择起点
  focus: Point | null               // 选择终点（拖动中）
  isDragging: boolean               // 是否正在拖动
  anchorSpan: { lo: Point; hi: Point; kind: 'word' | 'line' } | null  // 多击选择范围
  scrolledOffAbove: string[]        // 滚动出视口上方的文本累积
  scrolledOffBelow: string[]        // 滚动出视口下方的文本累积
  virtualAnchorRow?: number         // 虚拟锚点行（用于 clamp 恢复）
  virtualFocusRow?: number          // 虚拟焦点行
  lastPressHadAlt: boolean          // 上次按下是否有 Alt 修饰
}
```

### 3.2 关键流程

#### 3.2.1 渲染流程（onRender）

```
1. 检查 isUnmounted/isPaused，若 true 则直接返回
2. 清除 drainTimer（避免重复渲染）
3. flushInteractionTime()（更新交互时间戳）
4. 调用 renderer() 生成新帧：
   - Yoga 布局计算（已在 resetAfterCommit 中完成）
   - renderNodeToOutput() 将 DOM 渲染到 Output
   - Output.get() 返回 Screen 缓冲区
5. 处理滚动跟随（consumeFollowScroll）
6. 应用选择覆盖（applySelectionOverlay）
7. 应用搜索高亮（applySearchHighlight / applyPositionedHighlight）
8. 检查 layoutShifted / selActive / hlActive / prevFrameContaminated，决定全帧 damage
9. 调用 log.render(prevFrame, frame) 生成 diff patches
10. 交换 front/back 帧缓冲区
11. 定期重置 char/hyperlink pools（5 分钟间隔）
12. 处理 alt-screen 光标锚定和定位
13. 调用 writeDiffToTerminal() 输出到终端
14. 如有待处理滚动，设置 drainTimer 安排下一帧
15. 调用 onFrame 回调（性能分析）
```

#### 3.2.2 布局计算流程

```
React commit phase
  └── resetAfterCommit (reconciler.ts)
       └── rootNode.onComputeLayout() (ink.tsx)
            ├── 设置 Yoga root 宽度
            ├── yogaNode.calculateLayout()  // Yoga WASM 布局计算
            ├── 记录 Yoga 性能计数器
            └── 触发 onRender（通过 scheduleRender）
```

#### 3.2.3 输入事件处理流程

```
stdin 'readable' 事件
  └── App.handleReadable() (App.tsx)
       └── parseMultipleKeypresses() (parse-keypress.ts)
            └── processKeysInBatch()
                 ├── 键盘事件 → dispatchKeyboardEvent() → DOM 事件分发
                 ├── 鼠标事件 → handleMouseEvent()
                 │    ├── 选择更新（startSelection/updateSelection/finishSelection）
                 │    ├── 点击分发（dispatchClick）
                 │    └── 悬停分发（dispatchHover）
                 └── 终端响应 → querier.onResponse()
```

### 3.3 终端协议与命令

#### 3.3.1 使用的 DEC/CSI/OSC 序列

```typescript
// DEC 私有模式（dec.ts）
ENTER_ALT_SCREEN = '\x1b[?1049h'      // 进入备用屏幕
EXIT_ALT_SCREEN = '\x1b[?1049l'      // 退出备用屏幕
ENABLE_MOUSE_TRACKING = '\x1b[?1003h' // 启用鼠标追踪（任何事件）
DISABLE_MOUSE_TRACKING = '\x1b[?1003l'
ENABLE_FOCUS_EVENTS = '\x1b[?1004h'  // 启用焦点事件
DISABLE_FOCUS_EVENTS = '\x1b[?1004l'
BSU = '\x1b[?2026h'                  // 同步更新开始（DEC 2026）
ESU = '\x1b[?2026l'                  // 同步更新结束

// CSI 序列（csi.ts）
CURSOR_HOME = '\x1b[H'               // 光标归位
CURSOR_POSITION = '\x1b[{row};{col}H' // 绝对定位（1-indexed）
CURSOR_MOVE = '\x1b[{dy};{dx}H'      // 相对移动
ERASE_SCREEN = '\x1b[2J'             // 擦除屏幕
SCROLL_UP = '\x1b[{n}S'              // 向上滚动
SCROLL_DOWN = '\x1b[{n}T'            // 向下滚动
SET_SCROLL_REGION = '\x1b[{top};{bottom}r'  // 设置滚动区域

// OSC 序列（osc.ts）
OSC_52_CLIPBOARD = '\x1b]52;c;{base64}\x07'  // 剪贴板设置
OSC_8_HYPERLINK = '\x1b]8;;{url}\x07'        // 超链接开始
```

#### 3.3.2 同步输出（DEC 2026）

当终端支持 DEC 2026 时，所有渲染输出被包裹在 BSU/ESU 之间：

```
BSU (Begin Synchronized Update)
  CSI H                    // 光标归位（alt-screen）
  [diff patches...]        // 所有屏幕更新
  CSI {row};1 H            // 光标定位到输入位置
ESU (End Synchronized Update)
```

这确保终端将整帧作为原子操作渲染，避免可见的撕裂或部分更新。

### 3.4 性能优化策略

1. **Blit 优化**：干净节点（dirty=false 且布局未变）从 prevScreen 直接复制，跳过重新渲染
2. **Damage 跟踪**：仅 diff 发生变化的区域，而非全屏
3. **DECSTBM 滚动**：纯滚动时发送硬件滚动序列（CSI S/T），而非重写内容
4. **字符缓存**：Output.charCache 缓存 tokenize + grapheme 聚类结果
5. **样式池驻留**：StylePool 使用整数 ID 表示样式组合，避免重复字符串操作
6. **节流渲染**：scheduleRender 使用 lodash throttle（16ms），合并快速状态更新

## 4. 关键代码路径与文件引用

### 4.1 核心类定义

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `ink.tsx` | Ink 主类 | `Ink` class, `Options`, `drainStdin()` |
| `renderer.ts` | 渲染器工厂 | `createRenderer()`, `Renderer` type |
| `reconciler.ts` | React reconciler | `reconciler`, `dispatcher`, `getOwnerChain()` |
| `frame.ts` | 帧类型定义 | `Frame`, `FrameEvent`, `Patch`, `emptyFrame()` |

### 4.2 DOM 与布局

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `dom.ts` | DOM 抽象 | `DOMElement`, `TextNode`, `createNode()`, `markDirty()` |
| `layout/node.ts` | Yoga 节点包装 | `LayoutNode`, `LayoutDisplay`, `LayoutMeasureMode` |
| `layout/engine.ts` | Yoga 引擎 | `createLayoutNode()` |

### 4.3 屏幕与输出

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `screen.ts` | 屏幕缓冲区 | `Screen`, `CharPool`, `StylePool`, `setCellAt()`, `blitRegion()` |
| `output.ts` | 渲染输出收集 | `Output`, `Operation`, `Clip` |
| `log-update.ts` | 终端 diff | `LogUpdate`, `VirtualScreen` |
| `terminal.ts` | 终端能力检测 | `SYNC_OUTPUT_SUPPORTED`, `isXtermJs()`, `writeDiffToTerminal()` |

### 4.4 选择与交互

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `selection.ts` | 文本选择逻辑 | `SelectionState`, `startSelection()`, `applySelectionOverlay()` |
| `searchHighlight.ts` | 搜索高亮 | `applySearchHighlight()` |
| `render-to-screen.ts` | 位置扫描高亮 | `scanPositions()`, `applyPositionedHighlight()` |
| `focus.ts` | 焦点管理 | `FocusManager`, `getFocusManager()` |
| `hit-test.ts` | 点击测试 | `dispatchClick()`, `dispatchHover()` |

### 4.5 组件与上下文

| 文件 | 职责 | 关键导出 |
|------|------|----------|
| `components/App.tsx` | 根组件 | `App` class, `handleMouseEvent()` |
| `components/CursorDeclarationContext.ts` | 光标声明 | `CursorDeclaration`, `CursorDeclarationSetter` |
| `useTerminalNotification.ts` | 终端写入上下文 | `TerminalWriteProvider` |

### 4.6 关键代码路径示例

**路径 1：初始渲染**
```
new Ink(options)
  └── constructor()
       ├── 创建 DOM root (dom.createNode)
       ├── 创建 FocusManager
       ├── 创建 reconciler container
       └── 绑定事件处理器
```

**路径 2：React 更新 → 终端输出**
```
React state update
  └── reconciler commit
       └── resetAfterCommit()
            ├── onComputeLayout() → Yoga layout
            └── onRender() / scheduleRender()
                 └── Ink.onRender()
                      ├── renderer() → Frame
                      ├── apply overlays (selection/search)
                      ├── log.render() → Diff
                      └── writeDiffToTerminal()
```

**路径 3：鼠标选择**
```
stdin mouse event
  └── App.handleReadable()
       └── handleMouseEvent()
            ├── startSelection() / updateSelection() / finishSelection()
            └── onSelectionChange() → scheduleRender()
```

## 5. 依赖与外部交互

### 5.1 外部依赖

| 包名 | 用途 | 关键 API |
|------|------|----------|
| `react` / `react-reconciler` | React 核心 | `createReconciler`, `ConcurrentRoot`, `updateContainerSync` |
| `auto-bind` | 方法自动绑定 | `autoBind(this)` |
| `lodash-es` | 节流/空函数 | `throttle()`, `noop` |
| `signal-exit` | 进程退出处理 | `onExit()` |
| `@alcalzone/ansi-tokenize` | ANSI 序列解析 | `tokenize()`, `styledCharsFromTokens()`, `diffAnsiCodes()` |
| `yoga-layout` (native-ts) | Flexbox 布局 | `YogaNode`, `calculateLayout()` |

### 5.2 标准库依赖

- `fs`: 同步写入（unmount 时的终端清理）
- `util`: `format()` 用于 console patch
- Node.js 流: `stdin`/`stdout`/`stderr` 的 `ReadStream`/`WriteStream`

### 5.3 环境交互

| 环境变量 | 用途 |
|----------|------|
| `CLAUDE_CODE_DEBUG_REPAINTS` | 启用重绘调试日志 |
| `CLAUDE_CODE_ACCESSIBILITY` | 无障碍模式（显示光标） |
| `CLAUDE_CODE_DISABLE_MOUSE` | 禁用鼠标点击 |
| `TERM_PROGRAM` | 终端类型检测（iTerm.app, vscode 等） |
| `TMUX` | tmux 检测（禁用 DEC 2026） |
| `WT_SESSION` | Windows Terminal 检测 |

### 5.4 进程信号

| 信号 | 处理 |
|------|------|
| `SIGCONT` | `handleResume()` - 恢复终端状态 |
| `exit` (via signal-exit) | `unmount()` - 清理终端并退出 |

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 Yoga WASM 内存管理

**风险**：Yoga 节点使用 WASM 内存，若 `freeRecursive()` 与引用清理不同步可能导致 use-after-free 或内存泄漏。

**缓解**：`cleanupYogaNode()` 在 `freeRecursive()` 前调用 `clearYogaNodeReferences()` 清除所有引用。

#### 6.1.2 终端状态同步失败

**风险**：tmux attach/detach、ssh reconnect、笔记本睡眠/唤醒可能导致终端模式（鼠标追踪、alt-screen）与 Ink 内部状态不一致。

**缓解**：
- `reassertTerminalModes()` 在 >5s stdin 空闲后重新断言模式
- `handleResume()` 处理 SIGCONT 信号
- `handleResize()` 在尺寸变化时重置帧并重新进入 alt-screen

#### 6.1.3 宽字符（CJK/Emoji）处理

**风险**：终端 wcwidth 与 Ink 的 stringWidth 不一致可能导致光标位置漂移。

**缓解**：
- 使用 `CellWidth` 枚举显式标记宽字符和 spacer 单元格
- `needsWidthCompensation()` 检测需要补偿的新 emoji
- `writeCellWithStyleStr()` 中处理宽字符边界情况

#### 6.1.4 选择状态与滚动竞争

**风险**：快速滚动时，选择状态的虚拟行跟踪可能与实际屏幕内容不同步。

**缓解**：
- `virtualAnchorRow`/`virtualFocusRow` 跟踪预 clamp 位置
- `captureScrolledRows()` 在滚动前捕获即将离开视口的文本
- 双向债务跟踪（above/below debt）确保累积器长度正确

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 终端尺寸为 0x0 | 使用默认 80x24，Yoga 布局后 clamp 到有效范围 |
| 内容高度超过视口 | `shouldClearScreen()` 返回 'offscreen'，触发全帧重置 |
| 快速连续 resize | 同步处理（非 debounce），跳过相同尺寸事件 |
| 在编辑器中按 Ctrl+Z | `handleSuspend()` 保存 raw mode 计数，SIGCONT 恢复 |
| 双击后拖动 | `anchorSpan` 保持原始单词/行范围，拖动扩展到新位置 |
| 滚动时选择 | `shiftSelectionForFollow()` 使选择跟随文本移动 |

### 6.3 改进建议

#### 6.3.1 性能优化

1. **SIMD diff**：利用 Bun 的 `Bun.indexOfFirstDifference` 加速屏幕缓冲区比较
2. **Web Worker Yoga**：将 Yoga 布局计算移到 Worker 线程（当前在主线程）
3. **更激进的 blit**：当前 blit 仅用于干净节点，可探索部分脏节点的增量 blit

#### 6.3.2 可靠性增强

1. **终端能力运行时探测**：当前依赖环境变量，可增加更多 CSI 查询（如 DA1）
2. **Yoga 内存池**：减少 WASM 边界 crossing 的分配开销
3. **选择状态持久化**：支持跨会话的选择恢复（如 tmux 重连后）

#### 6.3.3 功能扩展

1. **矩形选择**：当前仅支持线性选择，可增加块选择模式（Alt+拖动）
2. **多光标支持**：类似 VS Code 的多光标编辑
3. **图像协议**：支持 Sixel/iTerm 图像协议在终端显示图片

#### 6.3.4 代码组织

1. **拆分 Ink 类**：当前 `ink.tsx` 超过 1700 行，可将选择管理、搜索高亮、终端模式管理拆分为独立模块
2. **类型安全**：增加更多运行时类型检查，特别是 Yoga 节点有效性
3. **测试覆盖**：增加集成测试覆盖 resize、SIGCONT、快速滚动等边界场景

### 6.4 调试工具

当前已提供的调试手段：

| 机制 | 启用方式 | 输出 |
|------|----------|------|
| 重绘调试 | `CLAUDE_CODE_DEBUG_REPAINTS=1` | 全帧重置的组件链 |
| Commit 日志 | `CLAUDE_CODE_COMMIT_LOG=path` | 每帧的 reconcile/layout/render 时间 |
| 调试日志 | `--debug` flag | stdin 事件、终端模式变更 |
| 高写入比警告 | 自动 | blit/write 比例异常时警告 |

建议增加：
- 帧时间线可视化（导出 Chrome DevTools 格式）
- Yoga 布局树 dump
- 屏幕缓冲区前后对比工具
