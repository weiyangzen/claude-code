# src/utils/intl.ts 研究文档

## 场景与职责

`intl.ts` 是一个性能优化型工具模块，负责缓存昂贵的 `Intl` 构造函数实例，避免在代码库中重复创建。`Intl` API（如 `Intl.Segmenter`、`Intl.RelativeTimeFormat`、`Intl.Locale`）的构造函数调用成本约为 0.05–0.1ms，在频繁调用的场景（如文本截断、相对时间格式化、字符串宽度计算）下会累积成显著开销。

该模块采用**懒加载（lazy initialization）**策略，确保只有在真正需要时才支付实例化成本。

调用方分布广泛，包括：
- `src/utils/format.ts`：`getRelativeTimeFormat`, `getTimeZone`
- `src/ink/stringWidth.ts`：`getGraphemeSegmenter`
- `src/utils/truncate.ts`：`getGraphemeSegmenter`
- `src/utils/format.ts`：`getSystemLocaleLanguage`
- `src/bridge/bridgeStatusUtil.ts`
- `src/utils/earlyInput.ts`
- `src/hooks/useVimInput.ts`
- `src/vim/operators.ts`
- `src/vim/textObjects.ts`
- `src/utils/Cursor.ts`

## 功能点目的

### 1. `getGraphemeSegmenter` / `firstGrapheme` / `lastGrapheme`
提供基于 Unicode 标准 grapheme cluster（用户感知字符）的文本分割能力。这对正确处理 emoji（如 👨‍👩‍👧‍👦 = 1 个 grapheme）、组合字符（如 é = e + ́）至关重要。

- `firstGrapheme(text)`：提取字符串的第一个 grapheme。
- `lastGrapheme(text)`：提取字符串的最后一个 grapheme（通过遍历所有 segments）。

### 2. `getWordSegmenter`
提供基于 Unicode 标准的分词能力。当前代码库中似乎没有直接调用该函数，但作为基础设施保留。

### 3. `getRelativeTimeFormat`
缓存 `Intl.RelativeTimeFormat` 实例，按 `style:numeric` 组合作为 key。支持 `'long' | 'short' | 'narrow'` 风格和 `'always' | 'auto'` 数字模式。

### 4. `getTimeZone`
缓存系统时区字符串（通过 `Intl.DateTimeFormat().resolvedOptions().timeZone`）。在进程生命周期内不会改变，因此只需计算一次。

### 5. `getSystemLocaleLanguage`
缓存系统 locale 的语言子标签（如 `'en'`、`'ja'`）。使用 `new Intl.Locale(locale).language` 提取。注释特别说明：
- `null` = 尚未计算
- `undefined` = 已计算但不可用（如剥离了 ICU 的运行时环境）

这种区分避免了在 ICU 不可用的环境中每次调用都重试。

## 具体技术实现

### 懒加载模式
所有实例都通过模块级变量 + getter 函数实现懒加载：

```typescript
let graphemeSegmenter: Intl.Segmenter | null = null

export function getGraphemeSegmenter(): Intl.Segmenter {
  if (!graphemeSegmenter) {
    graphemeSegmenter = new Intl.Segmenter(undefined, { granularity: 'grapheme' })
  }
  return graphemeSegmenter
}
```

### LRU/Map 缓存
`RelativeTimeFormat` 使用 `Map` 缓存多组合实例：

```typescript
const rtfCache = new Map<string, Intl.RelativeTimeFormat>()

export function getRelativeTimeFormat(style, numeric) {
  const key = `${style}:${numeric}`
  let rtf = rtfCache.get(key)
  if (!rtf) {
    rtf = new Intl.RelativeTimeFormat('en', { style, numeric })
    rtfCache.set(key, rtf)
  }
  return rtf
}
```

注意：`RelativeTimeFormat` 硬编码使用 `'en'` 作为 locale。这是有意的设计——Claude Code 的 CLI 界面目前主要使用英语输出，统一使用 `'en'` 可以避免因系统 locale 不同导致的格式不一致。

### `lastGrapheme` 的实现细节
```typescript
export function lastGrapheme(text: string): string {
  if (!text) return ''
  let last = ''
  for (const { segment } of getGraphemeSegmenter().segment(text)) {
    last = segment
  }
  return last
}
```

该实现时间复杂度为 O(n)，对于超长字符串（如数 MB 的日志输出）效率不高。但调用方通常只在短文本（如光标位置、单行输入）上使用，因此实际性能可接受。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/intl.ts:13-20` | `getGraphemeSegmenter` |
| `src/utils/intl.ts:26-31` | `firstGrapheme` |
| `src/utils/intl.ts:37-44` | `lastGrapheme` |
| `src/utils/intl.ts:46-51` | `getWordSegmenter` |
| `src/utils/intl.ts:56-67` | `getRelativeTimeFormat` |
| `src/utils/intl.ts:72-77` | `getTimeZone` |
| `src/utils/intl.ts:84-94` | `getSystemLocaleLanguage` |

## 依赖与外部交互

### 外部依赖
- 纯 JavaScript 运行时内置 `Intl` API，无第三方依赖。

### 调用方
- `src/utils/format.ts`：相对时间格式化、时区显示
- `src/ink/stringWidth.ts`： grapheme 宽度计算
- `src/utils/truncate.ts`：文本截断
- `src/bridge/bridgeStatusUtil.ts`：状态工具提示
- `src/utils/earlyInput.ts`：早期输入处理
- `src/hooks/useVimInput.ts` / `src/vim/operators.ts` / `src/vim/textObjects.ts` / `src/utils/Cursor.ts`：Vim 模式光标移动和文本对象操作

## 风险、边界与改进建议

### 风险与边界
1. **`lastGrapheme` 的线性扫描**：对于极长字符串，遍历所有 grapheme 来获取最后一个字符效率低下。可以考虑使用反向迭代（但 `Intl.Segmenter` 目前不支持反向分割）。
2. **硬编码 `'en'` locale**：`getRelativeTimeFormat` 固定使用英语。如果未来 CLI 需要完全本地化（i18n），需要重构为动态 locale。
3. **`Intl` 不可用环境**：`getSystemLocaleLanguage` 已经处理了 `Intl.Locale` 不可用的场景，但 `getGraphemeSegmenter` 和 `getRelativeTimeFormat` 没有类似的 try/catch。在某些精简 Node.js 运行时（如某些 Docker 镜像剥离了 ICU 数据）中，这些调用可能抛出 `RangeError: Invalid language tag` 或类似错误。
4. **无缓存上限**：`rtfCache` 理论上最多有 `3 styles × 2 numerics = 6` 个条目，所以实际上不会溢出。但如果有更多参数组合加入，需要注意边界。
5. **线程/进程安全**：缓存的 `Intl` 实例在单线程的 Node.js/Bun 事件循环中是安全的，但如果未来引入 Worker Threads，每个 Worker 会有自己的模块实例和缓存，这是预期行为。

### 改进建议
1. **为 `lastGrapheme` 添加长度限制**：在调用 `getGraphemeSegmenter().segment(text)` 前，若 `text.length` 超过某个阈值（如 10KB），可以回退到 `text.slice(-4)`（大多数 grapheme 不会超过 4 个 UTF-16 code units），避免线性扫描巨型字符串。
2. **统一 locale 配置**：引入一个 `DEFAULT_LOCALE` 常量或从配置中读取，为未来的 i18n 做准备。
3. **增强健壮性**：为 `getGraphemeSegmenter` 和 `getRelativeTimeFormat` 添加 try/catch，在 `Intl` 不可用时提供简单的回退实现（如基于 `Array.from(text)` 或手动时间格式化）。
4. **导出缓存清理函数**：在测试环境中，有时需要重置模块级缓存。可以导出 `_resetIntlCacheForTesting()` 函数，便于单元测试隔离。
