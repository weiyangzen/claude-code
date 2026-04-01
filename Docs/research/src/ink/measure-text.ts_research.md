# Research: src/ink/measure-text.ts

## 场景与职责

`measure-text.ts` 是 Ink 文本布局的核心测量函数，负责计算一段纯文本在指定最大宽度约束下的**显示宽度**和**视觉行数**。该函数被 Yoga 的自定义 `measureFunc` 调用，用于在 Flex 布局中确定 `Text` 节点的固有尺寸（intrinsic size）。由于终端文本可能包含换行符、ANSI 码、宽字符（CJK、emoji），且需要支持软换行（word-wrap），测量的准确性直接影响整个 UI 的布局结果。

## 功能点目的

1. **单遍测量**：在一次遍历中同时计算最大宽度和总高度，避免分别调用 `widestLine` 和 `countVisualLines` 的两次扫描。
2. **软换行高度计算**：当文本宽度超过 `maxWidth` 时，按 `Math.ceil(w / maxWidth)` 计算所需行数。
3. **无换行模式支持**：`maxWidth <= 0` 或 `Infinity` 时，每行只算 1 个视觉行。
4. **零分配行分割**：使用 `indexOf('\n', start)` + `substring` 代替 `split('\n')`，减少大文本时的数组分配。

## 具体技术实现

```ts
function measureText(text: string, maxWidth: number): Output {
  if (text.length === 0) return { width: 0, height: 0 }

  const noWrap = maxWidth <= 0 || !Number.isFinite(maxWidth)
  let height = 0, width = 0, start = 0

  while (start <= text.length) {
    const end = text.indexOf('\n', start)
    const line = end === -1 ? text.substring(start) : text.substring(start, end)

    const w = lineWidth(line)
    width = Math.max(width, w)

    if (noWrap) {
      height++
    } else {
      height += w === 0 ? 1 : Math.ceil(w / maxWidth)
    }

    if (end === -1) break
    start = end + 1
  }

  return { width, height }
}
```

- **`lineWidth(line)`**：来自 `./line-width-cache.js`，内部调用 `stringWidth` 并缓存结果。
- **`noWrap`**：在循环外判断，避免每次迭代重复计算。
- **空行处理**：`w === 0 ? 1` 保证空行也占 1 行高度（如连续换行 `\n\n`）。

## 关键代码路径与文件引用

- **被调用方**：
  - `src/ink/dom.ts:5` — `measureText` 被导入并用于 Yoga `measureFunc` 的文本测量。
- **依赖方**：
  - `src/ink/line-width-cache.ts` — 提供 `lineWidth`。
  - `src/ink/stringWidth.ts` — `lineWidth` 的底层实现。

## 依赖与外部交互

- 依赖 `./line-width-cache.js`。
- 无外部状态，纯函数，线程安全。

## 风险、边界与改进建议

- **风险**：`Math.ceil(w / maxWidth)` 是一种**均匀截断**近似，假设字符宽度均匀分布。对于包含宽字符和 ANSI 码的文本，实际换行位置由 `wrap-text.ts` 决定，可能与这里的估算不一致，导致 Yoga 计算的高度与实际渲染高度存在偏差。
- **边界**：
  - 不处理 `\r\n`，只按 `\n` 分割；若输入含 `\r` 会将其视为普通字符（宽度 0 或 1，取决于 `stringWidth`）。
  - 不处理制表符 `\t` 的展开；`stringWidth` 可能将其视为宽度 0 或 1，而实际渲染中 `output.ts` 会展开为 8 空格。
  - 对包含大量 ANSI 样式的文本，`lineWidth` 会 strip ANSI 后计算，但 `wrap-text.ts` 在换行时需要保留 ANSI，两者逻辑分离可能导致布局漂移。
- **改进建议**：
  1. **统一换行逻辑**：让 `measureText` 与 `wrap-text.ts` 共享同一套宽度累加器，确保 Yoga 测量高度与实际渲染高度一致。当前分离是历史原因，也是 Yoga measureFunc 被频繁调用时的性能权衡。
  2. **处理 `\r\n`**：可在分割前统一将 `\r\n` 替换为 `\n`。
  3. **制表符处理**：在测量阶段也按 8 字符 tab stop 估算，减少与实际渲染的偏差。
  4. 对超长文本（如 MB 级日志），可考虑采样或截断测量，避免 `indexOf` 在超大字符串上线性扫描的 CPU 开销。
