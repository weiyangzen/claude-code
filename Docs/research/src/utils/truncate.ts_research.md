# 研究文档：src/utils/truncate.ts

## 场景与职责

本模块提供**终端显示宽度感知的字符串截断、路径截断与文本换行**功能。与普通的 `String.prototype.slice` 不同，它需要正确处理：

- CJK 字符（占 2 列）、emoji（可能占 2 列）、组合字符（占 1 列但由多个 code point 组成）；
- 文件路径的中间截断（保留文件名和目录前缀）；
- 换行处的单线截断；
- Grapheme cluster 边界（避免将 emoji 或 surrogate pair 切成乱码）。

这些能力被广泛应用于终端 UI 渲染、插件市场浏览、文件路径显示、格式化输出等场景。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `truncatePathMiddle(path, maxLength)` | 中间截断文件路径，优先保留末尾文件名和开头目录上下文。 |
| `truncateToWidth(text, maxWidth)` | 从尾部截断字符串，使其显示宽度不超过 `maxWidth`，并追加 `…`。 |
| `truncateStartToWidth(text, maxWidth)` | 从头部截断字符串，保留尾部，并前置 `…`。 |
| `truncateToWidthNoEllipsis(text, maxWidth)` | 尾部截断但不追加省略号，供调用方自行拼接分隔符。 |
| `truncate(str, maxWidth, singleLine?)` | 通用截断入口，支持在首个换行处截断。 |
| `wrapText(text, width)` | 按显示宽度将文本拆分为多行数组，不破坏 grapheme。 |

## 具体技术实现

### 1. 显示宽度测量

- 使用 `src/ink/stringWidth.js` 提供的 `stringWidth(segment)` 函数，该函数基于 East Asian Width 数据计算字符在终端中的列宽（而非 Unicode code point 数量）。
- 这是本模块与纯文本截断工具的核心区别：它面向**终端列数**而非字符数。

### 2. Grapheme 安全分割

- 使用 `src/utils/intl.ts` 的 `getGraphemeSegmenter()` 获取 `Intl.Segmenter({ granularity: 'grapheme' })`。
- 所有截断和换行操作均迭代 grapheme segment，确保不会在 emoji、组合字符、surrogate pair 中间切断。

### 3. 路径中间截断算法 `truncatePathMiddle`

输入：`path`, `maxLength`

1. 若路径宽度 `<= maxLength`，直接返回。
2. 若 `maxLength <= 0`，返回 `…`。
3. 若 `maxLength < 5`，回退到普通尾部截断（`truncateToWidth`）。
4. 提取 `filename`（最后一个 `/` 及其后内容）和 `directory`（前面部分）。
5. 若 `filenameWidth >= maxLength - 1`，说明文件名本身太长，对**整个路径**做头部截断（`truncateStartToWidth`）。
6. 否则计算目录部分可用宽度：`availableForDir = maxLength - 1 - filenameWidth`（减 1 为省略号预留空间）。
7. 对 `directory` 调用 `truncateToWidthNoEllipsis` 截断到可用宽度，然后拼接为：`truncatedDir + '…' + filename`。

### 4. 尾部截断 `truncateToWidth`

- 若文本宽度 `<= maxWidth`，直接返回。
- 若 `maxWidth <= 1`，返回 `…`。
- 迭代 grapheme segments，累加宽度，当 `width + segWidth > maxWidth - 1` 时停止（预留 1 列给省略号）。
- 返回 `result + '…'`。

### 5. 头部截断 `truncateStartToWidth`

- 类似尾部截断，但从字符串末尾向前迭代。
- 记录能容纳下的起始 segment 索引 `startIdx`。
- 返回 `'…' + segments.slice(startIdx).join('')`。

### 6. 通用截断 `truncate`

- 若 `singleLine === true`：找到第一个 `\n`，截取之前的内容；若截取的宽度加上省略号仍超 `maxWidth`，则再调用 `truncateToWidth`；否则直接返回 `result + '…'`。
- 若 `singleLine === false` 或没有换行：直接调用 `truncateToWidth`。

### 7. 文本换行 `wrapText`

- 逐 grapheme segment 累加宽度到当前行。
- 若当前行宽度 + 新 segment 宽度 `<= width`，则追加；否则将当前行推入结果数组，新 segment 开启新行。
- 最后将剩余内容推入数组。
- **注意**：该实现是硬切（greedy），不考虑单词边界。

## 关键代码路径与文件引用

- **主实现**：`src/utils/truncate.ts`（179 行）
- **显示宽度计算**：`src/ink/stringWidth.js`（`stringWidth`）
- **Grapheme 分割器**：`src/utils/intl.ts`（`getGraphemeSegmenter`）
- **主要调用方**：`src/utils/format.ts`（大量格式化函数调用 `truncateToWidth`、`truncatePathMiddle` 等）
- **调用方（插件市场）**：`src/commands/plugin/DiscoverPlugins.tsx`、`src/commands/plugin/BrowseMarketplace.tsx`（`truncateToWidth`）
- **调用方（REPL 路径显示）**：`src/screens/REPL.tsx` 等通过 `format.ts` 间接使用

## 依赖与外部交互

- **`../ink/stringWidth.js`**：`stringWidth` 函数。
- **`./intl.js`**：`getGraphemeSegmenter` 函数。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **`stringWidth` 与终端字体不一致**：`stringWidth` 基于 Unicode EastAsianWidth 规范，但用户终端实际使用的字体可能对某些字符（如新版 emoji、Nerd Font 图标、组合字符序列）有不同的宽度渲染。这可能导致截断后的文本在实际终端中仍然超宽或欠宽。
2. **`truncatePathMiddle` 的 Windows 路径分隔符**：函数使用 `lastIndexOf('/')` 提取文件名。在 Windows 上，若路径使用反斜杠 `\`，文件名提取会失败，导致整个路径被当作目录处理。虽然项目其他部分通常会将路径规范化为 POSIX 分隔符，但这是一个潜在边界。
3. **省略号宽度假设**：所有截断函数假设省略号 `…`（U+2026）占 1 列宽度。这在绝大多数终端成立，但某些旧版终端或特殊字体可能将其渲染为 2 列。

### 边界

- **`wrapText` 硬切单词**：换行可能在单词中间发生，对于可读性要求高的文本（如自然语言）不够优雅。当前使用场景主要是代码/路径/命令输出，硬切可接受。
- **不支持 ANSI 转义序列**：若输入文本包含颜色转义序列（如 `\x1b[31m`），`stringWidth` 可能将其计入宽度，导致截断位置偏移。调用方通常会在着色前先截断，因此实际影响较小。
- **仅支持水平截断**：无垂直截断（如限制行数）功能。

### 改进建议

1. **跨平台路径分隔符**：将 `lastIndexOf('/')` 替换为 `lastIndexOf(path.sep)` 或同时检查 `'/'` 和 `'\\'`，以原生支持 Windows 路径格式。
2. **单词边界换行**：为 `wrapText` 增加可选的 `breakWords: boolean` 参数。当为 `false` 时，优先在空格/标点处换行，仅当单个单词超过 `width` 时才硬切。
3. **ANSI-aware 截断**：引入对 ANSI escape sequence 的识别，在计算宽度时跳过这些序列，或在截断后保持序列闭合（避免颜色泄漏到后续文本）。
4. **多行高度限制**：增加 `truncateLines(text, maxLines, lineWidth)` 函数，支持在限制行数的同时对最后一行做尾部截断并追加 `…`。
5. **RTL（从右到左）文本支持**：当前算法隐式假设 LTR 布局。对于阿拉伯语/希伯来语混合路径，显示逻辑可能更复杂，可考虑集成 `Intl.Segmenter` 的 `granularity: 'word'` 来辅助边界判断。
