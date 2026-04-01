# Research: src/ink/log-update.ts

## 场景与职责

`log-update.ts` 是 Ink 渲染管线中最复杂的模块之一，负责将两帧 `Frame`（前一帧 `prev` 与下一帧 `next`）的差异转换为可写入终端的 `Diff`（补丁数组）。它相当于终端版的"屏幕差异驱动更新引擎"，核心挑战在于：

- 终端光标只能相对移动，且处于未知起始位置；
- 内容可能溢出视口进入 scrollback，而 scrollback 中的行无法通过普通清屏/擦除指令修改；
- 需要最小化输出字节数以降低闪烁和延迟；
- 必须正确处理宽字符、超链接、ANSI 样式、硬件滚动（DECSTBM）等边界情况。

## 功能点目的

1. **增量更新（Diff）**：只输出发生变化的单元格，而非整屏重绘。
2. **全量回退（Full Reset）**：在 resize、scrollback 内容变化、shrinking 到视口以内等无法安全增量更新的场景，触发清屏后完整重绘。
3. **硬件滚动优化（DECSTBM）**：当 `ScrollBox` 的 `scrollTop` 变化且终端支持原子更新时，用 `CSI top;bot r` + `SU/SD` 代替逐行重写。
4. **光标恢复**：在主流屏幕（main screen）末尾正确恢复光标位置；在 alt-screen 中省略光标恢复以节省字节。
5. **宽字符补偿**：对终端 `wcwidth` 表可能遗漏的新 emoji（Unicode 12+）或 `VS16` 变体，插入 `cursorTo` + 空格填充，防止光标错位。

## 具体技术实现

### 核心类 `LogUpdate`

```ts
export class LogUpdate {
  private state: State
  constructor(private readonly options: Options) { ... }
  render(prev, next, altScreen = false, decstbmSafe = true): Diff
  renderPreviousOutput_DEPRECATED(prevFrame): Diff
  reset(): void
}
```

- `state.previousOutput` 追踪上一次输出字符串（已废弃，仅用于 `renderPreviousOutput_DEPRECATED`）。
- `render()` 是主入口，返回 `Diff`（`Patch[]`）。

### 关键流程

#### 1. Resize / 视口变化检测（行 142-147）
若 `next.viewport.height < prev.viewport.height` 或宽度变化，直接 `fullResetSequence_CAUSES_FLICKER(next, 'resize', ...)`。注释说明 resize 是罕见事件，预测新布局的复杂度不值得优化。

#### 2. DECSTBM 硬件滚动（行 165-185）
条件：
- `altScreen === true`
- `next.scrollHint` 存在
- `decstbmSafe === true`（调用方在无法保证原子序列时传 `false`，如不支持 DEC 2026 / BSU/ESU 的终端）

操作：
- 对 `prev.screen` 调用 `shiftRows(prev.screen, top, bottom, delta)` 模拟硬件滚动，使后续 diff 循环只看到"新滚入"的行。
- 生成补丁：`setScrollRegion(top+1, bottom+1) + (delta>0 ? csiScrollUp(delta) : csiScrollDown(-delta)) + RESET_SCROLL_REGION + CURSOR_HOME`。

#### 3. Scrollback 变化检测（行 199-248）
当内容高度 >= 视口高度且光标在底部时，前一帧的 cursor-restore 会在终端产生自动滚动，导致最上方一行进入 scrollback。若 diff 检测到 scrollback 区域内的行有变化，则必须 full reset（因为光标无法移动到 scrollback 中修改）。

计算：
```ts
const viewportY = prev.screen.height - prev.viewport.height
const scrollbackRows = viewportY + 1  // +1 为 cursor-restore 滚动额外推入的一行
```

#### 4. Shrinking 处理（行 250-283）
若 `next.screen.height < prev.screen.height`，需要清除底部多余的行。使用 `clear(N)` 指令（光标上移 N-1 行并擦除）。若需清除的行数超过视口高度，说明部分行在 scrollback 中，无法擦除，触发 full reset。

#### 5. Diff 循环（行 308-381）
使用 `diffEach(prev.screen, next.screen, callback)` 遍历所有变化的单元格。

跳过规则：
- 新增行（`growing && y >= prev.screen.height`）跳过，后续统一处理；
- `SpacerTail` / `SpacerHead`（宽字符的占位/折行标记）跳过；
- 空单元格且无前值（`isEmptyCellAt(next.screen, x, y) && !removed`）跳过，避免写尾部空格导致折行。

若变化发生在 `viewportY` 以上（scrollback 区），触发 `needsFullReset`。

写入逻辑：
- `moveCursorTo(screen, x, y)` 将虚拟光标定位到目标单元格；
- 处理 hyperlink 过渡（`transitionHyperlink`）；
- 处理 style 过渡（`stylePool.transition`）；
- `writeCellWithStyleStr` 写入字符并更新虚拟光标。

#### 6. Growing 处理（行 403-412）
对新增行调用 `renderFrameSlice(screen, next, prev.screen.height, next.screen.height, stylePool)`，直接输出新行。由于终端在底部会自动滚动，新行自然推入视口。

#### 7. 光标恢复（行 423-451）
- **alt-screen**：不恢复光标，下一帧以 `CSI H`（home）开始，相对移动从 (0,0) 算起。
- **main screen + cursor.y >= screen.height**：用 `\r` + `\n` 创建新行，因为光标移动指令无法创建新行。
- **main screen + cursor 在内容区内**：`moveCursorTo` 直接定位。

### `renderFrameSlice` / `renderFrame`

用于 full reset 或 growing 时的整段渲染。逐行、逐列遍历，使用 `visibleCellAtIndex` 快速跳过空白单元格和可优化的前景色-only 空格。每行结束显式重置 style/hyperlink 并输出 `\r\n`。

### `writeCellWithStyleStr`

内联了 `txn` 逻辑以避免闭包分配。关键逻辑：
- 宽字符在视口边缘（`px + 2 >= threshold`）时跳过写入，防止折行错位；
- `needsWidthCompensation` 检测新 emoji / VS16，插入 `cursorTo` + 空格 + 回写，强制修正旧终端的 `wcwidth`。

### `VirtualScreen`

内部类，追踪虚拟光标位置和已生成的 `diff` 数组。`txn` 方法接受一个函数，返回 `[patches, delta]`，自动追加补丁并更新光标。

### `needsWidthCompensation`

检测两类字符：
1. Unicode 12.0-15.0 新增符号（`U+1FA70-U+1FAFF`、`U+1FB00-U+1FBFF`）；
2. 文本默认 emoji + `U+FE0F`（如 ⚔️、☠️、❤️）。

### `moveCursorTo`

处理 pending wrap 状态（光标 x >= 视口宽度）：先用 `\r` 解除 pending wrap，再发相对移动指令。跨行移动时统一使用 `\r` + `cursorMove`，避免 `CUD` 在视口底部被 margin 截断。

## 关键代码路径与文件引用

- **调用方**：
  - `src/ink/ink.tsx:23` — `Ink` 类持有 `LogUpdate` 实例，在 `onRender` 中调用 `log.render(prev, next, altScreen, decstbmSafe)`。
- **依赖模块**：
  - `@alcalzone/ansi-tokenize` — `ansiCodesToString`, `diffAnsiCodes` 用于样式序列生成；
  - `src/ink/screen.js` — `Cell`, `CellWidth`, `cellAt`, `diffEach`, `shiftRows`, `visibleCellAtIndex` 等；
  - `src/ink/termio/csi.js` — `CURSOR_HOME`, `csiScrollUp`, `csiScrollDown`, `RESET_SCROLL_REGION`, `setScrollRegion`；
  - `src/ink/termio/osc.js` — `LINK_END`, `oscLink` 用于超链接；
  - `src/ink/frame.js` — `Diff`, `FlickerReason`, `Frame` 类型；
  - `src/ink/layout/geometry.js` — `Point` 类型；
  - `src/utils/debug.js` — `logForDebugging` 用于慢渲染告警。

## 依赖与外部交互

- 无直接 I/O，纯计算模块。输入为两帧 `Frame` 和配置，输出为 `Diff` 数组。
- `options.isTTY` 决定部分行为（如非 TTY 时直接 `renderFullFrame`）。
- `decstbmSafe` 由 `ink.tsx` 根据终端能力（`SYNC_OUTPUT_SUPPORTED`）传入。

## 风险、边界与改进建议

- **风险**：`fullResetSequence_CAUSES_FLICKER` 在 scrollback 变化、resize、shrinking 等场景会清屏重绘，虽然正确但可能产生可见闪烁。`ink.tsx` 会统计并上报 `flickers`。
- **边界**：
  - DECSTBM 硬件滚动要求 `altScreen && decstbmSafe`，在不支持原子更新的终端上回退到 diff 循环，字节数更多但无中间态。
  - `viewportY` 的计算对 `cursorRestoreScroll` 的 +1 修正非常敏感，错误会导致光标偏移 1 行。
  - 宽字符补偿依赖硬编码的 Unicode 范围，未来新增 emoji 需要手动扩展。
- **改进建议**：
  1. 对 `renderFullFrame`（非 TTY 路径）可探索增量输出，当前每帧都输出完整内容，在日志重定向场景下效率低。
  2. `diffEach` 的回调中频繁创建闭包，可进一步内联或改为迭代器模式减少 GC。
  3. 慢渲染日志阈值固定为 50ms，可考虑根据终端尺寸动态调整。
  4. 宽字符补偿列表可改为从 Unicode 数据自动生成，减少维护成本。
