# AlternateScreen.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`AlternateScreen` 是 Ink 终端 UI 框架中负责**全屏模式（Alternate Screen Buffer）**的核心组件。它通过 DEC 1049 终端控制序列切换终端的备用屏幕缓冲区，为 Claude Code 提供类 Vim/Less 的全屏交互体验。

### 1.2 使用场景
- **全屏模式（Fullscreen Mode）**：当 `CLAUDE_CODE_NO_FLICKER=0` 或满足特定条件时启用
- **临时覆盖层（Overlay）**：如 Ctrl+O 转录视图、权限对话框、命令输出等需要全屏展示的场景
- **文本选择与鼠标交互**：提供鼠标跟踪、文本选择、点击事件等高级交互能力

### 1.3 关键特性
- 进入/退出备用屏幕缓冲区（DEC 1049）
- 可选的 SGR 鼠标跟踪（滚轮 + 点击/拖拽）
- 高度约束为终端视口行数（无原生滚动回退）
- 与 Ink 实例状态同步（`setAltScreenActive`）
- 安全的信号退出处理（SIGCONT/SIGSTOP）

---

## 2. 功能点目的

### 2.1 备用屏幕缓冲区管理
| 功能 | 目的 |
|------|------|
| `ENTER_ALT_SCREEN` (DEC 1049h) | 切换到备用屏幕，保留主屏幕内容 |
| `EXIT_ALT_SCREEN` (DEC 1049l) | 返回主屏幕，恢复之前的终端状态 |
| `\x1B[2J\x1B[H` | 清屏并将光标归位（进入时初始化） |

### 2.2 鼠标跟踪控制
| 模式 | DEC 代码 | 功能 |
|------|----------|------|
| MOUSE_NORMAL | 1000 | 报告按钮按下/释放/滚轮 |
| MOUSE_BUTTON | 1002 | 增加拖拽事件（按钮按下时的移动） |
| MOUSE_ANY | 1003 | 增加无按钮移动（悬停检测） |
| MOUSE_SGR | 1006 | 使用 SGR 格式（CSI < btn;col;row M/m）替代传统 X10 |

鼠标事件最终转换为：
- `ParsedKey`（滚轮事件，通过 keybinding 系统处理）
- 选择状态更新（点击/拖拽，通过 `use-selection.ts` 暴露）

### 2.3 高度约束与布局
- 通过 `TerminalSizeContext` 获取终端行数 (`rows`)
- 使用 `Box` 组件设置 `height={rows}` 和 `flexShrink={0}`
- 强制内容高度等于视口高度，溢出通过 `overflow: scroll` 或 flexbox 处理
- 防止出现原生滚动回退（scrollback）行为

---

## 3. 具体技术实现

### 3.1 关键流程

#### 3.1.1 组件挂载流程（进入备用屏幕）
```
useInsertionEffect
  ↓
writeRaw(ENTER_ALT_SCREEN + "\x1B[2J\x1B[H" + ENABLE_MOUSE_TRACKING)
  ↓
ink?.setAltScreenActive(true, mouseTracking)
  ↓
渲染子组件（被 Box 高度约束）
```

#### 3.1.2 组件卸载流程（退出备用屏幕）
```
useInsertionEffect cleanup
  ↓
ink?.setAltScreenActive(false)
  ↓
ink?.clearTextSelection()
  ↓
writeRaw(DISABLE_MOUSE_TRACKING + EXIT_ALT_SCREEN)
```

### 3.2 数据结构

#### 3.2.1 Props 接口
```typescript
type Props = PropsWithChildren<{
  /** Enable SGR mouse tracking (wheel + click/drag). Default true. */
  mouseTracking?: boolean;
}>;
```

#### 3.2.2 React Compiler 缓存模式
组件使用 React Compiler（`_c`）进行自动记忆化：
- `$[0]` - `mouseTracking` 依赖
- `$[1]` - `writeRaw` 依赖
- `$[2]` - effect 函数缓存
- `$[3]` - effect 依赖数组缓存
- `$[4]` - `children` 依赖
- `$[5]` - `rows` 依赖
- `$[6]` - 渲染结果缓存

### 3.3 终端控制协议

#### 3.3.1 DEC 私有模式序列（dec.ts）
```typescript
// 进入备用屏幕（带清屏）
export const ENTER_ALT_SCREEN = decset(DEC.ALT_SCREEN_CLEAR);  // CSI ?1049h

// 退出备用屏幕
export const EXIT_ALT_SCREEN = decreset(DEC.ALT_SCREEN_CLEAR); // CSI ?1049l

// 启用鼠标跟踪（组合模式）
export const ENABLE_MOUSE_TRACKING = 
  decset(DEC.MOUSE_NORMAL) +   // 1000
  decset(DEC.MOUSE_BUTTON) +   // 1002
  decset(DEC.MOUSE_ANY) +      // 1003
  decset(DEC.MOUSE_SGR);       // 1006

// 禁用鼠标跟踪（逆序）
export const DISABLE_MOUSE_TRACKING = 
  decreset(DEC.MOUSE_SGR) +
  decreset(DEC.MOUSE_ANY) +
  decreset(DEC.MOUSE_BUTTON) +
  decreset(DEC.MOUSE_NORMAL);
```

#### 3.3.2 CSI 序列（csi.ts）
```typescript
// 清屏
export const ERASE_SCREEN = csi(2, 'J');  // CSI 2J

// 光标归位
export const CURSOR_HOME = csi('H');      // CSI H

// 光标定位（1-indexed）
export function cursorPosition(row: number, col: number): string {
  return csi(row, col, 'H');  // CSI row;col H
}
```

### 3.4 Ink 实例状态同步

#### 3.4.1 setAltScreenActive 实现（ink.tsx）
```typescript
setAltScreenActive(active: boolean, mouseTracking = false): void {
  if (this.altScreenActive === active) return;
  this.altScreenActive = active;
  this.altScreenMouseTracking = active && mouseTracking;
  if (active) {
    this.resetFramesForAltScreen();  // 重置帧缓冲区
  } else {
    this.repaint();                   // 重绘主屏幕
  }
}
```

#### 3.4.2 关键状态标志
- `altScreenActive`: 控制渲染器的光标 Y 轴钳制（防止滚动）
- `altScreenMouseTracking`: SIGCONT 恢复时决定是否重新启用鼠标跟踪
- `prevFrameContaminated`: 标记前一帧是否被污染（选择覆盖层/重置）

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
AlternateScreen.tsx
├── react (useContext, useInsertionEffect)
├── ../instances.js
│   └── Map<NodeJS.WriteStream, Ink> - Ink 实例管理
├── ../termio/dec.js
│   ├── ENTER_ALT_SCREEN/EXIT_ALT_SCREEN (DEC 1049)
│   ├── ENABLE_MOUSE_TRACKING/DISABLE_MOUSE_TRACKING (DEC 1000/1002/1003/1006)
│   └── decset/decreset 辅助函数
├── ../useTerminalNotification.js
│   └── TerminalWriteContext - 原始终端写入
├── ./Box.js
│   └── 布局容器（height={rows}, flexShrink={0}）
└── ./TerminalSizeContext.js
    └── 终端尺寸（rows/columns）
```

### 4.2 调用方（Callers）

#### 4.2.1 REPL.tsx（主要调用方）
```typescript
// 全屏模式启用时包裹主内容
return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
  {mainReturn}
</AlternateScreen>;

// 转录视图启用时包裹转录内容
return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
  {transcriptReturn}
</AlternateScreen>;
```

调用位置：
- 行 4485-4487：转录视图分支
- 行 5000-5002：主全屏分支

#### 4.2.2 其他调用方
- `thinkback.tsx` - 思考回溯功能
- `PromptInput.tsx` - 提示输入（特定模式）
- `AnimatedClawd.tsx` - 动画 Logo
- `FullscreenLayout.tsx` - 全屏布局（间接使用）

### 4.3 被调用方（Callees）

#### 4.3.1 Ink 类关键方法（ink.tsx）
| 方法 | 用途 |
|------|------|
| `setAltScreenActive()` | 标记备用屏幕状态 |
| `clearTextSelection()` | 清除文本选择 |
| `resetFramesForAltScreen()` | 重置帧缓冲区为空白 |
| `repaint()` | 主屏幕重绘 |
| `reenterAltScreen()` | SIGCONT 后重新进入 |

#### 4.3.2 渲染器处理（renderer.ts）
```typescript
// Alt-screen 高度钳制
const height = options.altScreen ? terminalRows : yogaHeight;

// Alt-screen 视口高度伪装（rows + 1 防止 clearScreen 触发）
height: options.altScreen ? terminalRows + 1 : terminalRows,

// 光标 Y 轴钳制（防止最后一行滚动）
y: options.altScreen
  ? Math.max(0, Math.min(screen.height, terminalRows) - 1)
  : screen.height,
```

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

#### 5.1.1 终端能力检测
```typescript
// fullscreen.ts
export function isMouseTrackingEnabled(): boolean {
  return isFullscreenEnvEnabled() && !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_MOUSE);
}

export function isFullscreenEnvEnabled(): boolean {
  // ants 默认启用，外部用户默认禁用
  const defaultValue = "external" === 'ant' ? false : true;
  return !isEnvTruthy(process.env.CLAUDE_CODE_NO_FLICKER, defaultValue);
}
```

#### 5.1.2 Tmux 检测与提示
```typescript
// fullscreen.ts
export async function maybeGetTmuxMouseHint(): Promise<string | undefined> {
  if (!isTmux() || isTmuxMouseEnabled()) return undefined;
  return "tmux: mouse off — wheel won't scroll. Run `tmux set -g mouse on` to enable.";
}
```

### 5.2 事件系统集成

#### 5.2.1 鼠标事件处理链
```
终端 SGR 鼠标报告
  ↓
parse-keypress.ts (parseMultipleKeypresses)
  ↓
App.tsx handleMouseEvent
  ↓
Ink.dispatchClick / dispatchHover / handleSelectionDrag
  ↓
React 组件 onClick / onMouseEnter / onMouseLeave
```

#### 5.2.2 选择状态管理（selection.ts）
```typescript
export type SelectionState = {
  anchor: Point | null;           // 选择起点
  focus: Point | null;            // 选择终点（拖拽中）
  isDragging: boolean;            // 是否正在拖拽
  anchorSpan: { lo: Point; hi: Point; kind: 'word' | 'line' } | null;
  scrolledOffAbove: string[];     // 滚出视口上方的文本
  scrolledOffBelow: string[];     // 滚出视口下方的文本
  // ... 软换行、虚拟位置等
};
```

### 5.3 信号与生命周期处理

#### 5.3.1 SIGCONT 处理（ink.tsx）
```typescript
private handleResume = () => {
  if (this.altScreenActive) {
    this.reenterAltScreen();  // 重新进入 + 清屏 + 启用鼠标
    return;
  }
  // 主屏幕处理...
};
```

#### 5.3.2 信号退出清理（ink.tsx unmount）
```typescript
if (this.altScreenActive) {
  writeSync(1, EXIT_ALT_SCREEN);  // 先退出备用屏幕
}
writeSync(1, DISABLE_MOUSE_TRACKING);
// ... 其他清理
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 终端兼容性问题
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| iTerm2 闪烁 | `?1049h` 即使已在备用屏幕也会清屏 | 避免重复发送 ENTER_ALT_SCREEN |
| tmux 鼠标 | 默认 `mouse off` 导致滚轮失效 | 检测并显示提示 |
| xterm.js 链接 | VS Code 终端同时处理 OSC 8 链接 | 检测 TERM_PROGRAM 并跳过 |
| SSH 重连 | 终端重置 DEC 私有模式 | stdin 间隔检测 + 模式重断言 |

#### 6.1.2 竞态条件
- **Resize 事件**：快速连续 resize 可能触发多次帧重置
- **Unmount 竞态**：信号退出时组件卸载可能无法执行清理
- **SIGCONT 恢复**：终端状态可能已被 shell 修改

### 6.2 边界情况

#### 6.2.1 高度溢出处理
```typescript
// renderer.ts 防御性钳制
if (options.altScreen && yogaHeight > terminalRows) {
  logForDebugging(
    `alt-screen: yoga height ${yogaHeight} > terminalRows ${terminalRows}`,
    { level: 'warn' }
  );
}
```

#### 6.2.2 非 TTY 环境
- `isTTY` 为 false 时跳过所有终端控制序列
- `writeRaw` 上下文可能为 null（防御性检查）

#### 6.2.3 嵌套 AlternateScreen
- React reconciler 会复用现有实例（相同组件类型）
- 嵌套会导致外层卸载时退出备用屏幕，破坏内层状态
- **当前设计假设**：全局单例，不嵌套

### 6.3 改进建议

#### 6.3.1 短期优化
1. **增加嵌套检测**：开发模式下警告嵌套 AlternateScreen
2. **终端能力缓存**：缓存 `isXtermJs()` 等检测结果避免重复计算
3. **鼠标事件批处理**：高频率鼠标移动事件合并处理

#### 6.3.2 中期改进
1. **备用屏幕状态查询**：通过 DECRQM 查询实际终端状态，与内部状态对比
2. **渐进式增强**：根据终端支持的鼠标模式动态调整（1000 vs 1002 vs 1003）
3. **选择持久化**：跨会话保存/恢复文本选择状态

#### 6.3.3 长期架构
1. **终端抽象层**：将 DEC/CSI/OSC 序列封装为更高级的 Terminal API
2. **多终端支持**：支持同时渲染到多个输出流（tmux 多 pane 场景）
3. **无障碍增强**：与屏幕阅读器更好的集成（光标声明已部分支持）

### 6.4 调试与监控

#### 6.4.1 关键日志点
```typescript
// ink.tsx - 调试备用屏幕转换
logForDebugging(`[alt-screen] ${active ? 'enter' : 'exit'}`);

// renderer.ts - 高度溢出警告
logForDebugging(`alt-screen: yoga height ${yogaHeight} > terminalRows ${terminalRows}`);

// App.tsx - 鼠标事件
logForDebugging(`XTVERSION: terminal identified as "${r.name}"`);
```

#### 6.4.2 性能指标
通过 `onFrame` 回调监控：
- `renderer`: DOM → Yoga → 屏幕缓冲区
- `diff`: 屏幕差异计算
- `write`: ANSI 序列写入
- `patches`: 每帧补丁数量

---

## 7. 附录

### 7.1 相关文件索引

| 文件路径 | 职责 |
|----------|------|
| `src/ink/components/AlternateScreen.tsx` | 本组件 |
| `src/ink/ink.tsx` | Ink 核心类，状态管理 |
| `src/ink/renderer.ts` | 渲染器，高度钳制 |
| `src/ink/termio/dec.ts` | DEC 私有模式序列 |
| `src/ink/termio/csi.ts` | CSI 控制序列 |
| `src/ink/termio/ansi.ts` | ANSI 基础定义 |
| `src/ink/selection.ts` | 文本选择逻辑 |
| `src/ink/hit-test.ts` | 鼠标点击检测 |
| `src/ink/components/App.tsx` | 事件处理 |
| `src/ink/components/Box.tsx` | 布局容器 |
| `src/ink/components/ScrollBox.tsx` | 滚动容器 |
| `src/ink/components/NoSelect.tsx` | 选择排除 |
| `src/ink/hooks/use-selection.ts` | 选择 Hook |
| `src/screens/REPL.tsx` | 主要调用方 |
| `src/components/FullscreenLayout.tsx` | 全屏布局 |
| `src/utils/fullscreen.ts` | 全屏工具函数 |

### 7.2 终端控制序列参考

| 序列 | 代码 | 功能 |
|------|------|------|
| CSI ?1049h | ENTER_ALT_SCREEN | 进入备用屏幕（带保存） |
| CSI ?1049l | EXIT_ALT_SCREEN | 退出备用屏幕（带恢复） |
| CSI ?1000h | MOUSE_NORMAL | 基本鼠标报告 |
| CSI ?1002h | MOUSE_BUTTON | 按钮事件跟踪 |
| CSI ?1003h | MOUSE_ANY | 所有移动事件 |
| CSI ?1006h | MOUSE_SGR | SGR 编码格式 |
| CSI 2J | ERASE_SCREEN | 清屏 |
| CSI H | CURSOR_HOME | 光标归位 |

---

*文档生成时间: 2026-04-01*
*研究范围: src/ink/components/AlternateScreen.tsx 及其直接依赖*
