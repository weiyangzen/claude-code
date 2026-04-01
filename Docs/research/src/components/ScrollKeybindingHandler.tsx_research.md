# ScrollKeybindingHandler.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位

`ScrollKeybindingHandler` 是 Claude Code 全屏模式下的**核心滚动与选择处理组件**，负责协调：

1. **键盘滚动导航** - PgUp/PgDn、Ctrl+Home/End、方向键等
2. **鼠标滚轮加速** - 区分物理鼠标滚轮与触控板，自适应加速曲线
3. **文本选择系统** - 拖拽选择、复制、键盘扩展选择
4. **模态分页器** - Transcript 模式下的 less/tmux copy-mode 风格导航 (g/G, ctrl+u/d/b/f)
5. **拖拽自动滚动** - 选择拖拽到视口边缘时的自动滚动

### 1.2 使用场景

| 场景 | 功能 |
|------|------|
| 消息历史浏览 | PgUp/PgDn 滚动半屏，滚轮逐行滚动 |
| 大段代码阅读 | 模态分页器 (g/G 跳转到顶部/底部, ctrl+u/d 半页) |
| 文本复制 | 鼠标拖拽选择，自动复制到剪贴板 (OSC 52/tmux/pbcopy) |
| 搜索导航 | 配合 VirtualMessageList 的搜索高亮，Esc 清除选择 |
| 跨设备兼容 | 自动检测 xterm.js (VS Code) vs 原生终端，应用不同滚动曲线 |

### 1.3 调用位置

```
src/screens/REPL.tsx (2处挂载)
├── 主消息滚动区域 (scrollRef)
│   └── ScrollKeybindingHandler scrollRef={scrollRef} isActive={...} onScroll={composedOnScroll}
└── 模态框滚动区域 (modalScrollRef)
    └── ScrollKeybindingHandler scrollRef={modalScrollRef} isActive={...}
```

**挂载顺序关键**: ScrollKeybindingHandler 必须在 CancelRequestHandler 之前挂载，确保 Ctrl+C 在有选择时执行复制而非取消任务。

---

## 2. 功能点目的

### 2.1 滚轮加速系统

**问题背景**: 
- 不同终端发送滚轮事件的频率差异巨大
- Ghostty/iTerm2 预放大 3 倍，VS Code (xterm.js) 不预放大
- 物理鼠标滚轮 (~9 事件/秒) vs 触控板轻扫 (~30 事件/秒)

**解决方案**: 双路径加速系统

| 路径 | 检测方式 | 加速曲线 | 适用终端 |
|------|----------|----------|----------|
| Native | `!isXtermJs()` | 40ms 窗口线性斜坡 + 反弹检测 | iTerm2, Ghostty, kitty |
| xterm.js | `isXtermJs()` | 指数衰减 (150ms 半衰期) | VS Code, Cursor, Windsurf |

**关键常量**:
```typescript
// Native 路径
const WHEEL_ACCEL_WINDOW_MS = 40;   // 事件间隔 <40ms 时加速
const WHEEL_ACCEL_STEP = 0.3;       // 每次递增倍数
const WHEEL_ACCEL_MAX = 6;          // 最大加速倍数

// xterm.js 路径
const WHEEL_DECAY_HALFLIFE_MS = 150;  // 指数衰减半衰期
const WHEEL_DECAY_STEP = 5;           // 动量增量
const WHEEL_DECAY_CAP_FAST = 6;       // 快速事件上限
const WHEEL_DECAY_CAP_SLOW = 3;       // 慢速事件上限
```

### 2.2 鼠标滚轮模式检测

**问题**: 如何区分物理鼠标滚轮与触控板？

**启发式算法**:
1. **反弹检测**: 物理滚轮编码器在快速滚动时产生反向抖动 (28% 事件)，触控板不会
2. **爆发检测**: 触控板轻扫产生 100+ 个 <5ms 间隔事件，鼠标 ≤3 个
3. **空闲断开**: 1500ms 无事件时重置模式，允许设备切换

### 2.3 文本选择系统

**核心功能**:
- **拖拽选择**: 鼠标按下 → 移动 → 释放，支持字符/单词/行模式
- **键盘扩展**: Shift+方向键扩展选择，Shift+Home/End 跳转到行首/尾
- **自动复制**: 选择完成自动复制到剪贴板 (iTerm2 风格)
- **选择跟踪**: 滚动时选择锚点跟随内容移动

**剪贴板路径优先级**:
1. Native (macOS pbcopy, 本地无 SSH)
2. tmux load-buffer (tmux 会话内)
3. OSC 52 (SSH 回退)

### 2.4 模态分页器

当 `isModal=true` 时 (Transcript 模式)，启用 less/tmux copy-mode 风格导航：

| 按键 | 动作 |
|------|------|
| g | 跳转到顶部 |
| G | 跳转到底部 |
| j/k | 上/下一行 |
| Ctrl+u/d | 上/下半页 |
| Ctrl+b/f | 上/下整页 |
| Space | 下整页 |
| b | 上整页 |
| Ctrl+n/p | 下/上一行 (emacs 风格) |

---

## 3. 具体技术实现

### 3.1 核心数据结构

#### WheelAccelState (滚轮加速状态)
```typescript
export type WheelAccelState = {
  time: number;           // 最后事件时间戳
  mult: number;           // 当前加速倍数
  dir: 0 | 1 | -1;        // 当前方向 (0=无, 1=下, -1=上)
  xtermJs: boolean;       // 是否使用 xterm.js 路径
  frac: number;           // 小数累积 (xterm.js 路径)
  base: number;           // 基础行数/事件 (环境变量可配置)
  pendingFlip: boolean;   // 等待反弹确认
  wheelMode: boolean;     // 确认为物理鼠标模式
  burstCount: number;     // 连续 <5ms 事件计数
};
```

#### Props 接口
```typescript
type Props = {
  scrollRef: RefObject<ScrollBoxHandle | null>;  // ScrollBox 引用
  isActive: boolean;                              // 是否激活
  onScroll?: (sticky: boolean, handle: ScrollBoxHandle) => void;  // 滚动回调
  isModal?: boolean;                              // 启用模态分页器
};
```

### 3.2 关键流程

#### 3.2.1 滚轮事件处理流程

```
滚轮事件 (wheelup/wheeldown)
    ↓
useKeybindings 捕获
    ↓
scroll:lineUp / scroll:lineDown 处理
    ↓
initAndLogWheelAccel() (首次延迟初始化)
    ↓
computeWheelStep(state, dir, now)
    ├── Native 路径: 窗口斜坡 + 反弹检测
    └── xterm.js 路径: 指数衰减曲线
    ↓
scrollUp() / scrollDown()
    ↓
scrollRef.current.scrollBy(step) / scrollToBottom()
    ↓
onScroll?.(sticky, handle) 回调
```

#### 3.2.2 选择拖拽自动滚动流程

```
useDragToScroll hook
    ↓
selection.subscribe(check) 监听选择变化
    ↓
dragScrollDirection() 计算方向
    ├── focus.row < top → -1 (向上滚动)
    ├── focus.row > bottom → 1 (向下滚动)
    └── 否则 → 0 (停止)
    ↓
start(dir) 启动定时器 (50ms 间隔)
    ↓
tick() 每 50ms 执行
    ├── 检查 isDragging
    ├── 检查 pendingDelta === 0 (避免堆积)
    ├── captureScrolledRows() 捕获即将滚出的行
    ├── shiftAnchor() 移动锚点
    └── scrollBy(AUTOSCROLL_LINES)
```

#### 3.2.3 键盘页面跳转选择跟踪流程

```
scroll:pageUp / scroll:pageDown / scroll:top / scroll:bottom
    ↓
translateSelectionForJump(s, delta)
    ├── 检查选择是否在 ScrollBox 内容区域内
    ├── 计算实际滚动距离 (边界限制)
    ├── captureScrolledRows() 捕获滚出的行
    └── shiftSelection() 移动整个选择
    ↓
jumpBy(s, delta) / scrollTo()
    ↓
onScroll?.(sticky, s)
```

### 3.3 关键算法实现

#### 3.3.1 滚轮步长计算 (computeWheelStep)

**Native 路径**:
```typescript
// 反弹检测与处理
if (state.pendingFlip) {
  if (dir !== state.dir || now - state.time > WHEEL_BOUNCE_GAP_MAX_MS) {
    // 真实方向反转
    state.dir = dir;
    state.mult = state.base;
    return Math.floor(state.mult);
  }
  // 确认反弹 → 进入 wheelMode
  state.wheelMode = true;
}

// 方向变化 → 延迟确认
if (dir !== state.dir && state.dir !== 0) {
  state.pendingFlip = true;
  return 0;
}

// Wheel 模式: 指数衰减曲线
if (state.wheelMode) {
  const m = Math.pow(0.5, gap / WHEEL_DECAY_HALFLIFE_MS);
  const next = 1 + (state.mult - 1) * m + WHEEL_MODE_STEP * m;
  state.mult = Math.min(cap, next, state.mult + WHEEL_MODE_RAMP);
  return Math.floor(state.mult);
}

// Trackpad 模式: 40ms 窗口线性斜坡
if (gap > WHEEL_ACCEL_WINDOW_MS) {
  state.mult = state.base;
} else {
  state.mult = Math.min(cap, state.mult + WHEEL_ACCEL_STEP);
}
return Math.floor(state.mult);
```

**xterm.js 路径**:
```typescript
// 同批次事件 (<5ms) → 1 行/事件
if (sameDir && gap < WHEEL_BURST_MS) return 1;

// 方向反转或空闲 → 重置为 2
if (!sameDir || gap > WHEEL_DECAY_IDLE_MS) {
  state.mult = 2;
  state.frac = 0;
} else {
  // 指数衰减
  const m = Math.pow(0.5, gap / WHEEL_DECAY_HALFLIFE_MS);
  const cap = gap >= WHEEL_DECAY_GAP_MS ? WHEEL_DECAY_CAP_SLOW : WHEEL_DECAY_CAP_FAST;
  state.mult = Math.min(cap, 1 + (state.mult - 1) * m + WHEEL_DECAY_STEP * m);
}

// 小数累积
const total = state.mult + state.frac;
const rows = Math.floor(total);
state.frac = total - rows;
return rows;
```

#### 3.3.2 模态分页器动作映射

```typescript
export function modalPagerAction(input: string, key: Key): ModalPagerAction | null {
  // 特殊键 (方向键/Home/End)
  if (!key.ctrl && !key.shift) {
    if (key.upArrow) return 'lineUp';
    if (key.downArrow) return 'lineDown';
    if (key.home) return 'top';
    if (key.end) return 'bottom';
  }
  
  // Ctrl 组合键
  if (key.ctrl) {
    switch (input) {
      case 'u': return 'halfPageUp';
      case 'd': return 'halfPageDown';
      case 'b': return 'fullPageUp';
      case 'f': return 'fullPageDown';
      case 'n': return 'lineDown';
      case 'p': return 'lineUp';
    }
  }
  
  // 裸字母 (支持 key-repeat 批处理)
  const c = input[0];
  if (input !== c.repeat(input.length)) return null;
  
  if (c === 'G' || (c === 'g' && key.shift)) return 'bottom';
  if (key.shift) return null;
  switch (c) {
    case 'g': return 'top';
    case 'j': return 'lineDown';
    case 'k': return 'lineUp';
    case ' ': return 'fullPageDown';
    case 'b': return 'fullPageUp';
  }
}
```

### 3.4 与 ScrollBox 的协作

```typescript
// ScrollBoxHandle 接口 (ScrollBox.tsx)
export type ScrollBoxHandle = {
  scrollTo: (y: number) => void;           // 绝对滚动
  scrollBy: (dy: number) => void;          // 相对滚动 (累积到 pendingScrollDelta)
  scrollToBottom: () => void;              // 滚动到底部并启用 sticky
  getScrollTop: () => number;              // 当前滚动位置
  getPendingDelta: () => number;           // 待处理的滚动增量
  getScrollHeight: () => number;           // 内容总高度
  getViewportHeight: () => number;         // 视口高度
  getViewportTop: () => number;            // 视口顶部行号
  isSticky: () => boolean;                 // 是否粘附底部
  setClampBounds: (min, max) => void;      // 设置滚动边界
};
```

**滚动同步机制**:
- `scrollBy()` 将增量累积到 `pendingScrollDelta`，由渲染器在 `render-node-to-output.ts` 中异步消耗
- `scrollTo()` 直接设置 `scrollTop`，清除 `pendingScrollDelta`
- `scrollToBottom()` 设置 `stickyScroll` 属性，渲染器自动跟随内容增长

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
ScrollKeybindingHandler.tsx
├── React hooks
│   ├── useRef (wheelAccel, selection state)
│   ├── useEffect (drag-to-scroll subscription)
│   └── useKeybindings (键盘绑定)
├── ink 系统
│   ├── useInput (原始输入处理)
│   ├── useSelection (文本选择)
│   └── ScrollBoxHandle (滚动控制)
├── 上下文
│   └── useNotifications (复制提示)
└── 工具函数
    ├── isXtermJs() (终端检测)
    ├── getClipboardPath() (剪贴板路径)
    └── logForDebugging() (调试日志)
```

### 4.2 关键文件引用

| 文件 | 用途 |
|------|------|
| `src/ink/components/ScrollBox.tsx` | ScrollBoxHandle 类型定义，滚动核心 |
| `src/ink/hooks/use-selection.ts` | 选择系统 Hook 封装 |
| `src/ink/selection.ts` | 选择状态操作 (shiftSelection, captureScrolledRows) |
| `src/ink/terminal.ts` | `isXtermJs()` 终端检测 |
| `src/ink/render-node-to-output.ts` | 滚动消耗 (drainAdaptive/drainProportional) |
| `src/ink/termio/osc.ts` | `getClipboardPath()`, `setClipboard()` |
| `src/keybindings/useKeybinding.ts` | `useKeybindings()` Hook |
| `src/keybindings/defaultBindings.ts` | 默认按键绑定 (scroll:pageUp 等) |
| `src/hooks/useCopyOnSelect.ts` | 自动复制 Hook |
| `src/context/notifications.tsx` | 通知系统 |

### 4.3 关键代码行号

```typescript
// ScrollKeybindingHandler.tsx
// 滚轮加速常量: 41-104
// Props 类型定义: 13-26
// WheelAccelState 类型: 142-170
// shouldClearSelectionOnKey: 115-120
// selectionFocusMoveForKey: 132-141
// computeWheelStep: 176-297
// readScrollSpeedBase: 305-310
// initWheelAccel: 314-326
// ScrollKeybindingHandler 组件: 359-623
// useDragToScroll hook: 637-792
// dragScrollDirection: 810-825
// jumpBy: 834-847
// scrollDown: 852-865
// scrollUp: 873-882
// modalPagerAction: 900-960
// applyModalPagerAction: 969-1011
```

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

| 依赖 | 类型 | 说明 |
|------|------|------|
| `react` | 核心 | React hooks (useRef, useEffect) |
| `../ink.js` | 内部 | Ink 核心 (useInput, Key 类型) |
| `../context/notifications.js` | 内部 | 通知系统 |
| `../hooks/useCopyOnSelect.js` | 内部 | 自动复制 |
| `../keybindings/useKeybinding.js` | 内部 | 按键绑定 |
| `../utils/debug.js` | 内部 | 调试日志 |

### 5.2 环境变量

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_SCROLL_SPEED` | 基础滚动速度倍率 (默认 1, 范围 0-20) |
| `TERM_PROGRAM` | 终端检测 (vscode, iTerm.app, ghostty 等) |
| `TMUX` | tmux 会话检测 |
| `SSH_CONNECTION` | SSH 连接检测 (影响剪贴板路径) |

### 5.3 与父组件的交互

**REPL.tsx 中的集成**:
```typescript
// 滚动回调组合
const composedOnScroll = useCallback((sticky: boolean, handle: ScrollBoxHandle) => {
  lastUserScrollRef.current = Date.now();
  if (!sticky) {
    onScrollAway(handle);  // 触发 "N new messages" 药丸显示
  } else {
    onRepin();  // 清除药丸
  }
  maybeLoadOlder(handle);  // 无限滚动加载历史
  disarmSearch();  // 退出搜索模式
}, [onScrollAway, onRepin, maybeLoadOlder, disarmSearch]);

// 组件挂载
<ScrollKeybindingHandler 
  scrollRef={scrollRef}
  isActive={isFullscreenEnvEnabled() && (centeredModal != null || !focusedInputDialog)}
  onScroll={centeredModal ? undefined : composedOnScroll}
/>
```

### 5.4 与 VirtualMessageList 的协作

```typescript
// VirtualMessageList.tsx
// ScrollKeybindingHandler 的 onScroll 触发 disarmSearch
export type JumpHandle = {
  // ...
  disarmSearch: () => void;  // 清除搜索位置高亮
};
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 滚轮检测误判

**风险**: 反弹检测依赖物理滚轮产生反向事件，某些高端鼠标 (如 MX Master 电磁滚轮) 在自由旋转模式下可能不产生反弹。

**缓解**: 
- 爆发检测作为备用 (100+ <5ms 事件 = 触控板)
- 1500ms 空闲断开允许手动重置

#### 6.1.2 剪贴板写入失败

**风险**: OSC 52 在 iTerm2 默认禁用，tmux 需要 `allow-passthrough on`。

**缓解**:
- 显示反馈消息告知用户剪贴板路径
- 本地 macOS 优先使用 pbcopy

#### 6.1.3 选择漂移

**风险**: 快速滚动时选择锚点可能漂移，特别是拖拽到视口边缘时。

**缓解**:
- `captureScrolledRows` 在滚动前捕获行内容
- `virtualAnchorRow` 跟踪虚拟位置
- `shiftSelection` 的债务计算确保正确弹出累积行

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 内容未填满视口 | `scroll:lineUp/Down` 返回 `false`，事件冒泡给子组件 |
| 滚轮到达底部 | 自动重新启用 sticky 模式 (`scrollDown` 返回 `true`) |
| 滚轮到达顶部 | 强制 `scrollTo(0)`，清除 pending delta 防止负累积 |
| 选择跨越 footer | `translateSelectionForJump` 检查两端点，拒绝移动跨边界选择 |
| 模态框显示时 | `isActive` 或 `onScroll` 被抑制，防止滚动干扰 |

### 6.3 改进建议

#### 6.3.1 可配置滚轮曲线

当前 `CLAUDE_CODE_SCROLL_SPEED` 只影响基础倍率，建议增加：
```typescript
// 建议添加
const CLAUDE_CODE_SCROLL_CURVE = 'linear' | 'exponential' | 'adaptive';
const CLAUDE_CODE_SCROLL_SENSITIVITY = number;  // 触控板专用灵敏度
```

#### 6.3.2 选择持久化

当前选择会在以下情况丢失：
- 滚动到选择完全离开视口
- 新的 assistant 消息到达并触发 compact

建议：
- 将选择范围存储为消息-relative 坐标
- compact 后恢复选择 (如果消息仍在)

#### 6.3.3 多指触控板手势

当前只处理滚轮事件，建议增加：
- 双指轻扫惯性滚动检测
- 捏合缩放 (调整字体大小)

#### 6.3.4 测试覆盖

建议增加以下测试：
- 滚轮加速曲线的单元测试 (模拟不同事件间隔)
- 反弹检测的单元测试 (模拟物理鼠标抖动)
- 选择跟踪的集成测试 (拖拽 → 滚动 → 复制)
- 模态分页器的端到端测试

### 6.4 性能考虑

| 方面 | 现状 | 建议 |
|------|------|------|
| 滚轮事件 | 每个事件触发 React 调度 | 已在 `scrollBy` 中合并，pendingDelta 批量处理 |
| 选择渲染 | 每帧全屏 damage | 必要，因为选择覆盖层修改 cell style |
| 自动滚动定时器 | 50ms setInterval | 合理，与 16ms 渲染帧率匹配 |
| 剪贴板写入 | 异步，无阻塞 | 正确，使用 `void` 忽略 Promise |

---

## 7. 附录

### 7.1 调试技巧

```typescript
// 启用滚轮调试日志
// 在 initAndLogWheelAccel 中已包含日志
// 查看 XTVERSION 检测结果:
logForDebugging(`wheel accel: ${xtermJs ? 'decay (xterm.js)' : 'window (native)'} · base=${base}`);

// 手动检查当前终端类型
import { isXtermJs } from '../ink/terminal.js';
console.log('isXtermJs:', isXtermJs());
```

### 7.2 相关文档

- `docs/research/terminal-scroll-*` - 滚轮调优研究文档
- `scroll-copy-mode-design.md` - 团队记忆文档 (搜索导航设计)

### 7.3 版本历史

| 日期 | 变更 |
|------|------|
| 2026-03 |  Boris 滚轮调优：添加反弹检测、wheelMode、xterm.js 自适应曲线 |
| 2026-03-15 | Tom 决议：模态分页器 ctrl+u/d 在 transcript 模式下工作 |
| 2026-03-17 | Boris 鼠标滚轮测量：28% 反弹率，~9 事件/秒 |
| 2026-03-18 | j/k 重新添加 (Tom 决议，撤销 3/16 移除) |

---

*文档生成时间: 2026-04-01*
*研究范围: src/components/ScrollKeybindingHandler.tsx 及其直接依赖*
