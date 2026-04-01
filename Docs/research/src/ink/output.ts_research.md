# Research: src/ink/output.ts

## 场景与职责

`output.ts` 是 Ink 的**屏幕缓冲区构建器**。它将渲染树（DOM）产生的绘制指令（写文本、块拷贝、清除、裁剪、滚动偏移等）收集为 `Operation` 列表，然后在 `get()` 方法中按顺序执行这些操作，最终生成一帧完整的 `Screen`（二维单元格数组）。这个 `Screen` 随后会被 `log-update.ts` 与前一帧对比，生成终端差异补丁。

简言之，`Output` 负责把**高层绘制指令**转换为**低层屏幕像素（单元格）数据**。

## 功能点目的

1. **指令收集与延迟执行**：渲染树遍历过程中只记录轻量指令，避免在遍历的同时直接操作大数组。
2. **跨帧缓存加速**：`charCache` 缓存每行文本的 tokenize + grapheme cluster + style intern 结果，稳定态帧大部分文本行可直接命中。
3. **ANSI 文本解析**：正确处理带 ANSI 转义码和超链接的文本，将其拆分为带样式的 grapheme cluster。
4. **裁剪（Clip）支持**：实现 `overflow: hidden` 等盒模型的内容裁剪，支持嵌套裁剪区域求交。
5. **块拷贝（Blit）与滚动偏移（Shift）**：支持从旧屏幕直接拷贝未变化区域，以及模拟硬件滚动的行偏移。
6. **清除与 NoSelect**：处理节点移除后的残影清除，以及禁用文本选择的区域标记。

## 具体技术实现

### 类 `Output`

```ts
export default class Output {
  width: number
  height: number
  private readonly stylePool: StylePool
  private screen: Screen
  private readonly operations: Operation[] = []
  private charCache: Map<string, ClusteredChar[]> = new Map()

  constructor(options: Options) { ... }
  reset(width, height, screen): void
  blit(src, x, y, width, height): void
  shift(top, bottom, n): void
  clear(region, fromAbsolute?): void
  noSelect(region): void
  write(x, y, text, softWrap?): void
  clip(clip): void
  unclip(): void
  get(): Screen
}
```

### `get()` 方法的两遍执行流程

#### Pass 1：收集 `clear` 操作并扩展 damage（行 277-305）

遍历所有 `operations`，对 `clear` 类型：
- 计算与屏幕边界的交集；
- 将区域合并到 `screen.damage`（供后续 diff 使用）；
- 若 `fromAbsolute` 为 true，加入 `absoluteClears` 列表。

`absoluteClears` 的作用：绝对定位节点可能覆盖非兄弟子树，当它被移除时，后续兄弟节点的 `blit` 会从旧屏幕拷贝回该绝对节点的残影。`absoluteClears` 用于在 `blit` 时跳过这些被污染的行。

#### Pass 2：执行非 clear 操作（行 309-508）

按顺序处理：
- **`clip`**：将新 clip 与当前 clip 栈顶求交（`intersectClip`），压栈。
- **`unclip`**：弹栈。
- **`blit`**：使用 `blitRegion` 从 `src` 屏幕批量拷贝单元格到当前屏幕。会与当前 clip 求交，并跳过 `absoluteClears` 覆盖的完整行。
- **`shift`**：调用 `shiftRows(screen, top, bottom, n)` 在屏幕缓冲区内移动行。
- **`write`**：最复杂的操作，见下文。

#### Pass 3：执行 `noSelect`（行 515-520）

最后处理 `noSelect`，确保标记覆盖在 `blit` 和 `write` 之上。

### `write` 操作的详细流程

1. **按 `\n` 分割为 lines**（`text.split('\n')`）。
2. **Clip 处理**：
   - 若完全在 clip 外，直接 `continue`；
   - 水平裁剪：对每行用 `sliceAnsi` 截取可见区间，并处理宽字符跨越 clip 边界的回退（`to - 1` 重试）；
   - 垂直裁剪：用 `lines.slice(from, to)` 截取可见行；若首可见行是 soft-wrap 延续，需记录前一行内容结束位置。
3. **逐行写入**：
   - 调用 `writeLineToScreen(screen, line, x, lineY, screenWidth, stylePool, charCache)`；
   - 若 `softWrap` 存在，更新 `screen.softWrap` 位图。

### `writeLineToScreen` 详解

这是整个模块最热的循环，被刻意提取为独立函数以便 JIT 优化。

#### 1. `charCache` 查找或生成（行 642-651）

```ts
let characters = charCache.get(line)
if (!characters) {
  characters = reorderBidi(
    styledCharsWithGraphemeClustering(
      styledCharsFromTokens(tokenize(line)),
      stylePool,
    ),
  )
  charCache.set(line, characters)
}
```

流程：
- `tokenize(line)`（`@alcalzone/ansi-tokenize`）→ ANSI token 数组；
- `styledCharsFromTokens` → `StyledChar[]`；
- `styledCharsWithGraphemeClustering` → 按样式 run 分组，再用 `Intl.Segmenter` 拆 grapheme，预计算 `styleId` / `hyperlink` / `width`；
- `reorderBidi` → 对双向文本进行重排序。

#### 2. 热循环写入单元格（行 655-794）

遍历每个 `ClusteredChar`：
- **C0 控制字符处理**（`codePoint <= 0x1f`）：
  - `\t`（0x09）：展开为 8 空格 tab stop；
  - `\x1b`（ESC）：跳过未识别的转义序列（CSI、OSC、DCS、ST 等），防止光标状态错乱；
  - `\r`、backspace、bell 等：跳过。
- **零宽字符**：`width === 0` 则跳过，不占用单元格。
- **宽字符边缘处理**：若宽字符在屏幕最后一列放不下（`offsetX + 2 > screenWidth`），写入 `SpacerHead`（空白占位），与终端行为一致。
- **正常写入**：`setCellAt(screen, offsetX, y, { char, styleId, width, hyperlink })`，`offsetX` 增加 1 或 2。

### `styledCharsWithGraphemeClustering` 详解

优化点：
- **按样式 run 分组**：一行 80 字符若只有 3 个样式变化，只需 3 次 `stylePool.intern` 和 hyperlink 提取，而非 80 次。
- **Hyperlink 过滤**：`extractHyperlinkFromStyles` 从 ANSI token 中提取 OSC 8 URL；`filterOutHyperlinkStyles` 将 OSC 8 相关 ANSI 码从样式数组中移除，避免样式字符串污染 `stylePool`。

### `intersectClip`

```ts
function intersectClip(parent: Clip | undefined, child: Clip): Clip
```

规则：`undefined` 表示该轴无边界，取另一方的边界；若双方都有边界，取更紧的约束（`max` of mins, `min` of maxes）。若结果为空（`x1 >= x2` 或 `y1 >= y2`），被裁剪的 write 将完全丢弃。

## 关键代码路径与文件引用

- **调用方**：
  - `src/ink/renderer.ts:5` — `createRenderer` 持有 `Output` 实例，每帧调用 `output.get()` 获取 `Screen`。
  - `src/ink/render-to-screen.ts:7` — 搜索渲染也使用 `Output`。
  - `src/ink/ink.tsx:27` — `Ink` 类直接导入 `Output`。
- **被调用方（渲染树到指令）**：
  - `src/ink/render-node-to-output.ts` — 遍历 DOM 树，调用 `output.write()`、`output.blit()`、`output.clear()` 等。
  - `src/ink/render-border.ts` — 绘制边框时调用 `output.write()`。
- **依赖模块**：
  - `@alcalzone/ansi-tokenize` — tokenize、styledCharsFromTokens；
  - `src/ink/screen.js` — `blitRegion`、`setCellAt`、`shiftRows`、`resetScreen` 等；
  - `src/ink/bidi.js` — `reorderBidi`；
  - `src/ink/stringWidth.js` — 计算 grapheme 宽度；
  - `src/ink/sliceAnsi.js` — ANSI 安全字符串截取；
  - `src/utils/intl.js` — `getGraphemeSegmenter`；
  - `src/utils/debug.js` — `logForDebugging`。

## 依赖与外部交互

- 无直接 I/O，纯内存计算。
- `charCache` 大小超过 16384 时会在 `reset()` 中清空，防止内存无限增长。

## 风险、边界与改进建议

- **风险**：`charCache` 以完整行字符串（含 ANSI）为键，对样式频繁变化的动态内容（如彩虹文本、进度条）缓存命中率极低，且占用大量内存。
- **边界**：
  - `text.split('\n')` 在 write 阶段分配数组，对超大文本（如一次性写入数千行）有 GC 压力。
  - `sliceAnsi` 在 clip 边界处理宽字符时有一次重试逻辑（`to - 1`），对极窄 clip（宽度为 1）仍可能溢出。
  - C0 控制字符的 ESC 跳过逻辑是手写状态机，虽覆盖常见序列，但无法保证对所有终端序列都安全。
- **改进建议**：
  1. **charCache 键优化**：可对纯文本使用内容哈希或子串引用，减少长 ANSI 行的键内存。
  2. **write 阶段避免 split**：对无 clip 的简单场景，可直接用 `indexOf('\n')` 循环代替 `split`，与 `measure-text.ts` 一致。
  3. **宽字符 clip 回退**：当前只重试一次，若 `to - 1` 仍跨宽字符边界（理论上不可能，因为宽字符占 2 列），会静默溢出。可改为循环重试或提前计算截断点。
  4. 对 `blitCells / writeCells` 的调试日志阈值（`>1000 && writeCells > blitCells`）可配置化，便于在不同终端尺寸下诊断性能。
  5. `reorderBidi` 对非双向文本是 no-op，但函数调用开销仍在。可在 `styledCharsWithGraphemeClustering` 后检测是否存在 RTL 字符，条件调用 `reorderBidi`。
