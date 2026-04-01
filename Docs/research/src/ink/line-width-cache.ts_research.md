# Research: src/ink/line-width-cache.ts

## 场景与职责

`line-width-cache.ts` 为 Ink 的文本测量提供行级缓存。在终端 UI 的每一帧渲染中，大量文本节点需要计算其在终端中的显示宽度（考虑 emoji、CJK、ANSI 转义码等）。由于已完成的多行内容在流式输出或滚动过程中通常保持不变，逐行缓存 `stringWidth` 结果可将调用量减少约 50 倍。

## 功能点目的

1. **加速重复测量**：同一行文本在相邻帧中宽度不变，直接命中缓存避免重新遍历 grapheme cluster。
2. **控制内存上限**：设置 `MAX_CACHE_SIZE = 4096`，防止在长时间运行或接收大量不同响应时缓存无限膨胀。
3. **透明回退**：缓存未命中时调用 `stringWidth` 计算，调用方无感知。

## 具体技术实现

```ts
const cache = new Map<string, number>()
const MAX_CACHE_SIZE = 4096

export function lineWidth(line: string): number {
  const cached = cache.get(line)
  if (cached !== undefined) return cached

  const width = stringWidth(line)

  if (cache.size >= MAX_CACHE_SIZE) {
    cache.clear()
  }

  cache.set(line, width)
  return width
}
```

- **缓存策略**：以完整行字符串为键，显示宽度为值。利用 `Map` 的 `get` 做 O(1) 查找。
- **淘汰策略**：到达上限时执行 `cache.clear()` 全量清空。注释说明"全清后一帧即可重新填充"，因为每帧涉及的行数通常远小于 4096。
- **依赖**：`stringWidth` 来自 `./stringWidth.js`，内部优先使用 `Bun.stringWidth`，回退到自定义 JS 实现。

## 关键代码路径与文件引用

- **被调用方**：
  - `src/ink/measure-text.ts:31` — `measureText()` 在逐行测量时调用 `lineWidth(line)`。
  - `src/ink/widest-line.ts:12` — `widestLine()` 在找最长行时调用 `lineWidth(line)`。
- **依赖方**：
  - `src/ink/stringWidth.ts` — 提供实际的字符串宽度计算。

## 依赖与外部交互

- 仅依赖 `./stringWidth.js`。
- 无外部配置或环境变量影响。

## 风险、边界与改进建议

- **风险**：以整行字符串为键，若行很长（如数千字符）会占用较多 Map 键内存；但 4096 条上限限制了最坏情况。
- **边界**：
  - 全清策略在刚好超过 4096 的帧会产生缓存抖动，导致该帧所有测量都回退到 `stringWidth`。
  - 不区分 ANSI 样式变化：同一文本带不同 ANSI 码会被视为不同键，这是正确行为（`stringWidth` 会 strip ANSI）。
- **改进建议**：
  1. 若 profiling 显示 4096 是瓶颈，可改为 LRU（如 `Map` 按插入顺序 + 淘汰最旧）而非全清，减少抖动。
  2. 对空字符串或纯 ASCII 短字符串可设置更短路径，避免 Map 查找开销（但现代引擎中 Map get 已足够快）。
  3. 可考虑将缓存与 `Output` 的 `charCache` 打通，避免同一行文本在输出阶段再次 tokenize。
