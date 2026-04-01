# Research: src/ink/optimizer.ts

## 场景与职责

`optimizer.ts` 是 Ink 渲染管线中的**补丁压缩器**。`log-update.ts` 生成的 `Diff`（终端补丁数组）在复杂场景下可能包含大量冗余指令：连续的 `cursorMove`、重复的 `styleStr`、成对的 `cursorHide`/`cursorShow`、空 `stdout` 等。该模块在 `Diff` 写入终端前执行单遍优化，减少最终输出字节数，降低终端解析器和渲染管线的压力。

## 功能点目的

1. **减少输出字节数**：合并同类补丁，消除无操作补丁。
2. **保持语义正确性**：优化规则经过严格验证，确保合并后终端状态不变。
3. **低开销**：单遍线性扫描，时间复杂度 O(n)，不引入复杂数据结构。

## 具体技术实现

```ts
export function optimize(diff: Diff): Diff {
  if (diff.length <= 1) return diff

  const result: Diff = []
  let len = 0

  for (const patch of diff) {
    // 1. 跳过 no-ops
    if (type === 'stdout' && patch.content === '') continue
    if (type === 'cursorMove' && patch.x === 0 && patch.y === 0) continue
    if (type === 'clear' && patch.count === 0) continue

    // 2. 尝试与 result 最后一项合并
    if (len > 0) {
      const last = result[lastIdx]!

      // 合并连续 cursorMove
      if (type === 'cursorMove' && lastType === 'cursorMove') {
        result[lastIdx] = { type: 'cursorMove', x: last.x + patch.x, y: last.y + patch.y }
        continue
      }

      // 覆盖连续 cursorTo（只有最后一个有效）
      if (type === 'cursorTo' && lastType === 'cursorTo') {
        result[lastIdx] = patch
        continue
      }

      // 拼接相邻 styleStr
      if (type === 'styleStr' && lastType === 'styleStr') {
        result[lastIdx] = { type: 'styleStr', str: last.str + patch.str }
        continue
      }

      // 去重连续 hyperlink（URI 相同）
      if (type === 'hyperlink' && lastType === 'hyperlink' && patch.uri === last.uri) {
        continue
      }

      // 抵消 cursorHide + cursorShow 对
      if ((type === 'cursorShow' && lastType === 'cursorHide') ||
          (type === 'cursorHide' && lastType === 'cursorShow')) {
        result.pop()
        len--
        continue
      }
    }

    result.push(patch)
    len++
  }

  return result
}
```

### 优化规则详解

| 规则 | 说明 | 安全依据 |
|------|------|----------|
| 跳过空 `stdout` | `content === ''` 不产生任何终端效果 | 无副作用 |
| 跳过零移动 `cursorMove` | `(0,0)` 不改变光标位置 | 无副作用 |
| 跳过零计数 `clear` | `count === 0` 不清除任何行 | 无副作用 |
| 合并 `cursorMove` | 向量相加 | 相对移动满足加法结合律 |
| 覆盖 `cursorTo` | 后者绝对定位覆盖前者 | 终端只响应最后一个 `CHA` |
| 拼接 `styleStr` | 字符串拼接 | `styleStr` 是 ANSI 过渡序列，连续发送等价于合并发送 |
| 去重 `hyperlink` | 相同 URI 的连续超链接指令冗余 | 超链接状态不变 |
| 抵消光标显隐对 | `hide` + `show` 或 `show` + `hide` 成对删除 | 最终光标可见性不变 |

### 关于 `styleStr` 的安全注释

代码中特别说明：`styleStr` 是**过渡差分**（`diffAnsiCodes(from, to)`），不是单纯的"设置样式"。因此不能随意丢弃前一个 `styleStr`，因为它的 undo-codes 可能不被后一个覆盖。例如 `[\e[49m, \e[2m]` 若丢弃 `\e[49m`，背景重置会泄漏到后续 `\e[2J/\e[2K`（BCE 行为）。所以只允许**拼接**，不允许**丢弃**。

## 关键代码路径与文件引用

- **调用方**：
  - `src/ink/ink.tsx:26` — `import { optimize } from './optimizer.js'`，在 `onRender` 中将 `log.render()` 产生的 `Diff` 传入 `optimize()`。
- **输入类型**：
  - `src/ink/frame.js` — `Diff` 类型定义（`Patch[]`）。

## 依赖与外部交互

- 仅依赖 `./frame.js` 的 `Diff` 类型。
- 无运行时外部依赖，纯函数，无状态。

## 风险、边界与改进建议

- **风险**：当前优化是**局部贪心**的，只合并相邻同类补丁。若中间隔了一个无关补丁（如 `cursorMove` 被 `styleStr` 隔开），则无法合并。对于高密度 diff，这种非全局最优可能导致仍有冗余。
- **边界**：
  - `styleStr` 拼接在极端长序列下可能生成很长的单个字符串，但终端通常能处理 KB 级 ANSI 序列。
  - `cursorMove` 合并后若 `x`/`y` 值很大，依赖终端正确解析多参数 CSI 序列；所有目标终端均支持。
- **改进建议**：
  1. **引入更多规则**：例如相邻的 `stdout` 内容拼接（当它们之间没有光标移动时），可进一步减少小片段输出。
  2. **两趟优化**：第一趟做局部合并，第二趟扫描非相邻但可抵消的补丁（如 `styleStr` 后紧跟一个重置到相同状态的 `styleStr`）。
  3. 可添加 `optimize` 前后的 patch 数量/字节数统计，集成到 `FrameEvent` 的 `phases.optimize` 中，便于持续监控优化收益。
  4. 考虑将 `result` 数组预分配容量（`new Array(diff.length)`）以减少动态扩容，虽然现代引擎对此优化已较好。
