# ScrollBox.tsx 深度研究文档

## 场景与职责

`ScrollBox` 是 Ink 终端 UI 框架中实现**虚拟滚动容器**的核心组件。它解决在终端环境中渲染大量内容的性能问题——当消息列表、日志输出或代码内容超出可视区域时，ScrollBox 提供：

1. **视口裁剪 (Viewport Culling)**：只渲染可见区域内的子元素，避免渲染不可见内容
2. **平滑滚动**：支持滚轮事件、键盘导航和程序化滚动控制
3. **粘性底部 (Sticky Scroll)**：内容增长时自动保持滚动到底部（类似聊天应用的行为）
4. **虚拟滚动支持**：与 `useVirtualScroll` hook 配合，实现超大数据集的高效渲染

### 典型使用场景

- **消息列表**：`Messages.tsx`、`VirtualMessageList.tsx` 中渲染对话历史
- **代码/日志查看**：长文本内容的可滚动展示
- **设置面板**：`Config.tsx` 等配置界面的滚动区域
- **全屏布局**：`FullscreenLayout.tsx` 中的主内容区

---

## 功能点目的

### 1. Imperative Scroll API (命令式滚动接口)

通过 `useImperativeHandle` 暴露的 `ScrollBoxHandle` 提供以下能力：

| 方法 | 用途 |
|------|------|
| `scrollTo(y)` | 绝对位置滚动 |
| `scrollBy(dy)` | 相对滚动，支持累积增量 |
| `scrollToElement(el, offset)` | 滚动到指定元素（延迟到渲染时计算位置）|
| `scrollToBottom()` | 滚动到底部并启用粘性跟随 |
| `getScrollTop()` / `getScrollHeight()` | 获取滚动状态 |
| `isSticky()` | 检查是否处于粘性底部状态 |
| `subscribe(listener)` | 订阅滚动变化事件 |
| `setClampBounds(min, max)` | 设置虚拟滚动的边界限制 |

**设计关键**：`scrollToElement` 与 `scrollTo` 的区别——前者在渲染时才读取 Yoga 计算的位置，避免 React 异步渲染导致的位置过时问题。

### 2. 滚动状态管理

```typescript
// DOMElement 上的滚动相关属性
scrollTop?: number              // 当前滚动位置
pendingScrollDelta?: number     // 待处理的滚动增量（平滑滚动）
scrollClampMin/Max?: number     // 虚拟滚动边界
scrollHeight?: number           // 内容总高度
scrollViewportHeight?: number   // 视口高度
scrollViewportTop?: number      // 视口在屏幕上的绝对位置
stickyScroll?: boolean          // 是否粘性跟随
scrollAnchor?: { el, offset }   // 元素锚定滚动
```

### 3. 性能优化机制

- **微任务批处理**：`queueMicrotask` 合并多个 `scrollBy` 调用到单次渲染
- **滚动活动标记**：`markScrollActivity()` 通知后台任务暂停，避免与滚动竞争事件循环
- **脏标记系统**：`markDirty()` + `scheduleRenderFrom()` 触发 Ink 的节流渲染

---

## 具体技术实现

### 关键流程 1: 滚动事件处理

```typescript
function scrollMutated(el: DOMElement): void {
  markScrollActivity()      // 暂停后台任务 150ms
  markDirty(el)             // 标记节点需要重渲染
  markCommitStart()         // 标记提交开始（性能追踪）
  notify()                  // 通知订阅者
  
  // 微任务批处理：合并多次 scrollBy 调用
  if (renderQueuedRef.current) return
  renderQueuedRef.current = true
  queueMicrotask(() => {
    renderQueuedRef.current = false
    scheduleRenderFrom(el)  // 触发 Ink 渲染
  })
}
```

### 关键流程 2: 滚动增量累积与消耗

```typescript
// ScrollBox.tsx - 累积阶段
scrollBy(dy: number) {
  el.pendingScrollDelta = (el.pendingScrollDelta ?? 0) + Math.floor(dy)
  scrollMutated(el)
}

// render-node-to-output.ts - 消耗阶段
if (pending !== undefined && pending !== 0) {
  const pastClamp = haveClamp && ((pending < 0 && cur < cMin) || (pending > 0 && cur > cMax))
  const eff = pastClamp ? Math.min(4, innerHeight >> 3) : innerHeight
  cur += isXtermJsHost()
    ? drainAdaptive(node, pending, eff)   // VS Code: 自适应小步进
    : drainProportional(node, pending, eff) // 原生终端: 比例衰减
}
```

**双模式排水策略**：
- **xterm.js (VS Code)**：使用 `drainAdaptive`，小增量（≤5）立即完成，大增量分步处理，保证平滑动画
- **原生终端**：使用 `drainProportional`，每帧消耗剩余量的 ~75%，快速收敛

### 关键流程 3: 粘性底部跟随

```typescript
// 渲染时检查是否需要跟随
const sticky = node.stickyScroll ?? Boolean(node.attributes['stickyScroll'])
const prevMaxScroll = Math.max(0, prevScrollHeight - prevInnerHeight)
const grew = scrollHeight >= prevScrollHeight
const atBottom = sticky || (grew && scrollTopBeforeFollow >= prevMaxScroll)

if (atBottom && (node.pendingScrollDelta ?? 0) >= 0) {
  node.scrollTop = maxScroll  // 跟随到底部
  // ... 同步 sticky 标志
}
```

### 数据结构: 组件结构约定

ScrollBox 依赖特定的子元素结构：

```jsx
<ink-box overflowY="scroll">     {/* ScrollBox 外框 - 固定高度 */}
  <Box flexGrow={1} flexShrink={0}> {/* 内容包装器 - 自然高度 */}
    {children}                    {/* 实际内容 */}
  </Box>
</ink-box>
```

- `flexShrink: 0` 防止内容包装器被压缩
- `overflow: scroll` 在 Yoga 层面阻止容器扩展以适应内容

---

## 关键代码路径与文件引用

### 核心文件

| 文件 | 职责 |
|------|------|
| `ScrollBox.tsx` | 组件实现、命令式 API、状态管理 |
| `render-node-to-output.ts` | 滚动渲染、排水算法、视口裁剪 |
| `dom.ts` | `DOMElement` 类型定义、滚动相关属性 |
| `styles.ts` | `overflow`/`overflowX`/`overflowY` 样式处理 |

### 调用链

```
用户滚轮事件
  ↓
ScrollKeybindingHandler.tsx (解析为 wheelup/wheeldown)
  ↓
useKeybinding.ts / 直接调用
  ↓
scrollBoxRef.current.scrollBy(dy)
  ↓
ScrollBox.tsx: scrollMutated()
  ↓
scheduleRenderFrom() → Ink 渲染循环
  ↓
render-node-to-output.ts: renderNodeToOutput()
  ↓
排水算法处理 pendingScrollDelta
  ↓
视口裁剪 + 内容渲染
```

### 关键配置常量

```typescript
// render-node-to-output.ts
const SCROLL_MIN_PER_FRAME = 4        // 原生终端最小每帧滚动行数
const SCROLL_INSTANT_THRESHOLD = 5    // xterm.js 立即完成阈值
const SCROLL_HIGH_PENDING = 12        // xterm.js 高速滚动阈值
const SCROLL_STEP_MED = 2             // 中等速度步进
const SCROLL_STEP_HIGH = 3            // 高速步进
const SCROLL_MAX_PENDING = 30         // 最大累积滚动量
```

---

## 依赖与外部交互

### 直接依赖

```typescript
import { markScrollActivity } from '../../bootstrap/state.js'  // 滚动活动标记
import type { DOMElement } from '../dom.js'                     // DOM 类型
import { markDirty, scheduleRenderFrom } from '../dom.js'       // 渲染触发
import { markCommitStart } from '../reconciler.js'              // 性能标记
import type { Styles } from '../styles.js'                      // 样式类型
import Box from './Box.js'                                      // 布局容器
```

### 外部交互

1. **与 `useVirtualScroll` 的协作**：
   - `setClampBounds()` 由 `useVirtualScroll` 调用，设置当前挂载子元素的范围
   - 防止快速滚动时显示空白区域

2. **与 `render-node-to-output.ts` 的协作**：
   - 读取 `pendingScrollDelta` 进行排水
   - 计算 `scrollHeight` 和视口边界
   - 执行视口裁剪 (`renderScrolledChildren`)

3. **与 `bootstrap/state.ts` 的协作**：
   - `markScrollActivity()` 设置 150ms 的滚动状态
   - 后台任务（IDE poll、LSP poll、GCS fetch）检查 `getIsScrollDraining()` 并跳过

---

## 风险、边界与改进建议

### 已知风险

1. **滚动竞争条件**：
   - 快速连续调用 `scrollTo` 和 `scrollBy` 可能导致 `pendingScrollDelta` 和直接赋值冲突
   - 代码通过 `scrollAnchor = undefined` 在 `scrollBy` 中取消锚定来缓解

2. **虚拟滚动边界情况**：
   - 当 `scrollTop` 超出 `scrollClampMin/Max` 时，渲染被限制但 `scrollTop` 继续更新
   - 需要 React 提交后才能恢复同步，期间可能显示边缘内容

3. **xterm.js 检测延迟**：
   - `isXtermJs()` 依赖异步 XTVERSION 探测，早期滚动事件可能使用错误算法
   - 使用 `process.env.TERM_PROGRAM === 'vscode'` 作为同步回退

### 边界情况处理

| 场景 | 处理方式 |
|------|----------|
| 内容高度为 0 | `maxScroll = 0`，滚动位置保持 0 |
| 视口高度 ≥ 内容高度 | 无需滚动，`stickyScroll` 无效果 |
| 滚动到负数位置 | `Math.max(0, y)` 钳制 |
| 滚动超出最大位置 | `Math.min(cur, maxScroll)` 钳制 |
| 相反方向滚动抵消 | `pendingScrollDelta` 自然累加/抵消 |

### 改进建议

1. **平滑滚动动画**：
   - 当前 `scrollToElement` 是即时跳转，可考虑添加可选的缓动动画
   - 需要解决与 `useVirtualScroll` 范围更新的同步问题（参见代码注释）

2. **滚动性能监控**：
   - 添加滚动帧率监控，检测排水算法是否跟得上输入事件
   - 当前依赖 `markCommitStart()` 的基础性能追踪

3. **测试覆盖**：
   - 虚拟滚动边界条件的单元测试
   - xterm.js vs 原生终端的排水行为差异测试
   - 粘性跟随在内容收缩时的行为测试

4. **代码简化**：
   - `scrollMutated` 中的 `markCommitStart()` 调用目的不明确，可能需要文档说明或移除
   - 考虑将排水算法抽象为可配置策略，便于针对不同终端优化
