# `src/utils/Cursor.ts` 技术调研文档

## 1. 场景与职责

`src/utils/Cursor.ts` 是终端文本输入系统的核心引擎，负责在基于 Ink（React for terminal）的 UI 中实现类 Emacs/Vim 的文本编辑体验。该文件全长约 1530 行，主要服务于以下场景：

- **Prompt 输入**：`src/components/PromptInput/PromptInput.tsx` 渲染用户的主输入框，所有光标定位、选区、掩码（mask）显示均依赖 `Cursor` 的输出。
- **Hook 层状态管理**：`src/hooks/useTextInput.ts` 作为通用文本输入 Hook，通过不可变的 `Cursor` 实例驱动所有编辑操作；`src/hooks/useSearchInput.ts` 复用同一套机制实现搜索框；`src/hooks/useVimInput.ts` 则在 Vim 模式下包装 `Cursor`。
- **Vim 运动与操作解析**：`src/vim/motions.ts` 与 `src/vim/operators.ts` 调用 `Cursor` 的 `nextVimWord`、`prevVimWord`、`findCharacter`、`modifyText` 等方法，将 Vim 命令解析为具体的光标位移或文本变更。

其职责可归纳为三点：
1. **文本测量与折行**：基于终端列宽，将原始文本折行为视觉行（wrapped lines），并维护字符串偏移量（offset）与视觉坐标（line, column）之间的双向映射。
2. **光标移动**：支持字符级、词级、行级、逻辑行级（以 `\n` 分隔）及 Vim 语义（word / WORD / `f`/`F`/`t`/`T`）的多种移动方式。
3. **编辑与 Kill Ring**：提供插入、删除、替换等编辑操作，同时维护一个全局的 kill ring（剪切环），支持 Emacs 风格的 `yank`（粘贴）与 `yank-pop`（循环粘贴）。

---

## 2. 功能点目的

### 2.1 Unicode NFC 规范化

`MeasuredText` 在构造时立即执行 `text.normalize('NFC')`（第 1135 行）。NFC（Normalization Form C）将组合字符序列合并为单一码点，例如将拉丁字母 `e` + 组合重音符号 `́`（U+0301）合并为 `é`（U+00E9）。

**目的**：
- **统一偏移量语义**：如果输入中混有 NFD 与 NFC，同样的视觉字符可能对应 1 个或 2 个 UTF-16 码元，导致光标在字符内部“卡住”。NFC 规范化后，每个视觉字符的码元数量变得可预测。
- **保证 `Intl.Segmenter` 结果一致**：`Intl.Segmenter` 的切分结果依赖于码点序列；NFC 能消除因等价序列差异导致的边界不一致。
- **与显示宽度对齐**：`stringWidth` 和终端实际渲染均基于 NFC 后的码点计算宽度，避免光标位置与终端渲染错位。

### 2.2 不可变的 `Cursor` 模式

`Cursor` 类的所有公共方法（如 `left()`、`right()`、`insert()`、`nextWord()`）均返回一个新的 `Cursor` 实例，而非修改当前实例的 `offset`。

**目的**：
- **简化撤销/重做与状态管理**：调用方（如 `useTextInput.ts`）只需用新实例替换旧实例即可更新状态，无需深拷贝或追踪变更历史。
- **避免副作用链**：在 Vim 操作符（如 `dw`、`cw`）中，可以安全地基于原始光标计算目标位置，再生成最终光标，中间步骤不会影响外部状态。
- **React 兼容**：不可变对象天然适合 React 的 `useState`/`useReducer`，引用相等性检查可直接用于性能优化。

### 2.3 全局 Kill Ring

文件顶部以模块级变量定义了 `killRing: string[]`、`killRingIndex`、`lastActionWasKill`、`lastYankStart` 等状态（第 16–24 行），并提供 `pushToKillRing`、`yankPop`、`recordYank` 等函数。

**目的**：
- **跨输入框共享剪切历史**：用户在 Prompt 中删除的文本可以在搜索框中粘贴，实现类 Emacs 的全局 kill ring 体验。
- **连续删除累积**：当连续执行删除操作时（如多次 `Ctrl+K`），`pushToRing` 会将新删除的文本追加到同一条 kill ring 记录中，而不是产生多条碎片记录。
- **Yank-Pop 循环**：粘贴后按 `Alt+Y` 可循环替换为更早的 kill 记录。

### 2.4 Grapheme Cluster 处理

通过 `Intl.Segmenter`（`granularity: 'grapheme'`）将文本切分为用户感知的“字符”（grapheme cluster），例如 emoji 家庭组合 `👨‍👩‍👧‍👦` 会被视为一个整体单元。

**目的**：
- **光标不进入组合字符内部**：按左右方向键时，光标以 grapheme 为步长移动，避免停留在 ZWJ（零宽连接符）或肤色修饰符之间。
- **显示宽度与偏移量精确对应**：CJK 字符、emoji、阿拉伯文组合等复杂脚本在终端中可能占据 1 或 2 个单元格，grapheme 级切分是正确计算列宽的前提。

### 2.5 `[Image #N]` Chip 原子性

当用户粘贴图片引用时，输入框会显示为 `[Image #1]`、`[Image #2]` 等标记。`Cursor` 通过正则 `/\[Image #\d+\]/` 识别这些 chip，并在移动和删除时将其视为不可分割的原子单元。

**目的**：
- **防止用户光标进入 chip 内部**：`left()` 和 `right()` 会检测 chip 边界并直接跳过（第 304–319 行）。
- **删除操作不破坏 chip 结构**：`deleteTokenBefore()`、`snapOutOfImageRef()` 等方法确保用户不会只删除 chip 的一部分，从而避免生成无效的图片引用标记。

---

## 3. 具体技术实现

### 3.1 核心类结构

#### `MeasuredText`

`MeasuredText` 是 `Cursor` 的底层依赖，负责所有与文本几何相关的计算：

- **构造与规范化**（第 1125–1137 行）：
  ```ts
  constructor(text: string, readonly columns: number) {
    this.text = text.normalize('NFC')
    this.navigationCache = new Map()
  }
  ```
  传入的 `columns` 为终端可用列宽（不含光标占位）。`Cursor.fromText()` 在创建 `MeasuredText` 时会传入 `columns - 1`，为光标预留一列空间（第 168–169 行）。

- **懒加载折行**（`wrappedLines` getter，第 1143–1148 行）：
  调用 `wrapAnsi(this.text, this.columns, { hard: true, trim: false })` 将文本按硬折行规则切分。随后通过 `measureWrappedText()`（第 1293–1369 行）将折行结果映射回原始字符串偏移量，生成 `WrappedLine[]`。每个 `WrappedLine` 记录：
  - `text`：该视觉行的字符串内容
  - `startOffset`：在原始文本中的起始偏移
  - `isPrecededByNewline`：是否紧跟换行符（决定前导空格是否被 trim）
  - `endsWithNewline`：是否以换行符结尾

- **Grapheme 边界缓存**（`getGraphemeBoundaries()`，第 1150–1160 行）：
  使用 `getGraphemeSegmenter().segment(this.text)` 遍历所有切分段，将每个段的 `index` 存入数组，最后追加 `this.text.length`。后续 `nextOffset` / `prevOffset` 通过在该数组上进行**二分查找**（`binarySearchBoundary`，第 1198–1230 行）来快速定位下一个或上一个 grapheme 边界，时间复杂度为 O(log n)。

- **词边界缓存**（`getWordBoundaries()`，第 1168–1189 行）：
  使用 `getWordSegmenter().segment(this.text)` 切分，返回 `{ start, end, isWordLike }` 数组。该缓存对 CJK 文本尤为重要，因为 `Intl.Segmenter` 会将每个汉字识别为独立的词。

- **偏移量与视觉坐标双向映射**（第 1386–1481 行）：
  - `getOffsetFromPosition({ line, column })`：先定位到对应 `WrappedLine`，再通过 `displayWidthToStringIndex` 将显示列宽转换为字符串索引，最后加上 `startOffset`。
  - `getPositionFromOffset(offset)`：遍历 `wrappedLines`，找到包含该偏移量的行，再计算其在行内的显示列宽。对于折行产生的非首行，会扣除被 `trimStart()` 去掉的前导空格宽度。

#### `Cursor`

`Cursor` 本身只持有三个只读字段：

```ts
readonly measuredText: MeasuredText
readonly offset: number
readonly selection: number  // 当前未使用，预留用于未来选区功能
```

所有移动方法均基于 `offset` 计算新的偏移量，然后 `new Cursor(this.measuredText, newOffset, 0)`。

**关键方法分类**：

| 类别 | 方法名 | 说明 |
|------|--------|------|
| 基础移动 | `left()`, `right()`, `up()`, `down()` | 以 grapheme 或视觉行为单位移动 |
| 行首行尾 | `startOfLine()`, `endOfLine()`, `startOfCurrentLine()`, `firstNonBlankInLine()` | 基于视觉行（wrapped line） |
| 逻辑行移动 | `startOfLogicalLine()`, `endOfLogicalLine()`, `upLogicalLine()`, `downLogicalLine()`, `firstNonBlankInLogicalLine()` | 基于 `\n` 分隔的逻辑行 |
| 标准词移动 | `nextWord()`, `prevWord()`, `endOfWord()` | 基于 `Intl.Segmenter` 的 `word` 粒度 |
| Vim 词移动 | `nextVimWord()`, `prevVimWord()`, `endOfVimWord()` | 基于 `\p{L}\p{N}\p{M}_` 正则的 Vim 语义 |
| Vim WORD 移动 | `nextWORD()`, `prevWORD()`, `endOfWORD()` | 基于非空白字符序列 |
| 编辑 | `insert()`, `del()`, `backspace()`, `modifyText()` | 均返回新 `Cursor` |
| 删除到行首/行尾 | `deleteToLineStart()`, `deleteToLineEnd()`, `deleteToLogicalLineEnd()` | 返回 `{ cursor, killed }`，用于 kill ring |
| 按词删除 | `deleteWordBefore()`, `deleteWordAfter()` | 结合 `snapOutOfImageRef` 处理 chip |
| Token 删除 | `deleteTokenBefore()` | 专门处理 `[Image #N]`、`[Pasted text #N]` 等原子标记 |
| Vim 字符查找 | `findCharacter(char, type, count)` | 支持 `f/F/t/T` 及计数前缀 |
| 渲染与视口 | `render()`, `getViewportStartLine()`, `getViewportCharOffset()`, `getViewportCharEnd()` | 生成带光标反显、掩码、ghost text 的终端字符串 |

### 3.2 渲染流程（`render` 方法）

`render()`（第 203–299 行）是 `Cursor` 与终端显示之间的桥梁：

1. **视口裁剪**：根据 `maxVisibleLines` 计算 `startLine` 与 `endLine`，采用“光标居中”策略（`half = Math.floor(maxVisibleLines / 2)`），当总行数超过可视行数时，尽量让光标位于视口中间。

2. **掩码处理**：若传入 `mask` 字符（用于密码或 OAuth token 输入）：
   - 对最后一行，仅保留末尾最多 6 个字符可见，其余用掩码字符替换（第 226–233 行）。
   - 对前面的折行，全部掩码（第 234–238 行）。

3. **光标行分割**：对光标所在行，使用 `getGraphemeSegmenter().segment(displayText)` 单遍遍历，按显示宽度累加，将字符串切分为 `beforeCursor`、`atCursor`、`afterCursor` 三部分（第 250–269 行）。这比原先的两遍方案更高效，且天然保证光标不会落在 grapheme 内部。

4. **Ghost Text 处理**：若当前光标在文本末尾且存在 ghost text（如 AI 建议的补全内容），将 ghost text 的第一个 grapheme 放入反显光标中，剩余部分以 `dim` 样式追加（第 274–289 行）。

5. **ANSI 反显**：通过调用方传入的 `invert` 函数对 `atCursor` 或 ghost text 首字符进行 ANSI 反色输出。

### 3.3 Kill Ring 状态机

Kill ring 的实现位于模块顶层，不隶属于任何类：

```ts
const KILL_RING_MAX_SIZE = 10
let killRing: string[] = []
let killRingIndex = 0
let lastActionWasKill = false
let lastYankStart = 0
let lastYankLength = 0
let lastActionWasYank = false
```

- **`pushToKillRing(text, direction)`**（第 26–49 行）：
  - 若 `lastActionWasKill` 为 true，则将新文本追加（`append`）或前置（`prepend`）到 `killRing[0]`。
  - 否则，将新文本 `unshift` 到数组头部；若超过 `KILL_RING_MAX_SIZE`（10），则 `pop` 掉最旧的记录。
  - 每次 push 都会重置 `lastActionWasYank = false`。

- **`recordYank(start, length)`**（第 80–85 行）：在 `yank` 成功后记录插入位置与长度，并将 `killRingIndex` 重置为 0。

- **`yankPop()`**（第 91–103 行）：
  - 仅当 `lastActionWasYank` 为 true 且 `killRing.length > 1` 时才允许执行。
  - `killRingIndex = (killRingIndex + 1) % killRing.length`，返回对应的文本以及原始 yank 的 `start`/`length`，供调用方替换。

### 3.4 `[Image #N]` Chip 的原子性实现

Chip 的识别依赖三个正则方法：

- `imageRefEndingAt(offset)`（第 325–328 行）：检查 `offset` 是否恰好落在一个 `[Image #N]` 的末尾。
- `imageRefStartingAt(offset)`（第 330–333 行）：检查 `offset` 是否恰好落在一个 `[Image #N]` 的开头。
- `snapOutOfImageRef(offset, toward)`（第 340–351 行）：遍历文本中所有 `[Image #N]` 匹配项，若 `offset` 严格位于某个 chip 内部（`offset > start && offset < end`），则将其吸附到 `start` 或 `end`。

这些辅助方法被以下公共方法调用：
- `left()` / `right()`：直接跳过 chip。
- `deleteWordBefore()` / `deleteWordAfter()`：在计算目标词边界前，先调用 `snapOutOfImageRef`，确保不会只删除 chip 的一部分。
- `deleteTokenBefore()`：额外支持 `[Pasted text #N]`、`[...Truncated text #N +50 lines...]` 等变体（第 960–961 行）。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件内部关键路径

| 功能 | 起始行号 | 方法/变量 |
|------|----------|-----------|
| Kill Ring 定义 | 16 | `KILL_RING_MAX_SIZE`, `killRing` |
| Kill Ring Push | 26 | `pushToKillRing` |
| Yank-Pop | 91 | `yankPop` |
| NFC 规范化 | 1135 | `this.text = text.normalize('NFC')` |
| Grapheme 边界 | 1150 | `getGraphemeBoundaries()` |
| 词边界 | 1168 | `getWordBoundaries()` |
| 二分查找边界 | 1198 | `binarySearchBoundary()` |
| 折行测量 | 1293 | `measureWrappedText()` |
| 光标构造 | 151 | `Cursor` class |
| 渲染 | 203 | `render()` |
| 左右移动 | 301 | `left()`, `right()` |
| Chip 识别 | 325 | `imageRefEndingAt()`, `imageRefStartingAt()` |
| Chip 吸附 | 340 | `snapOutOfImageRef()` |
| 标准词移动 | 561 | `nextWord()`, `endOfWord()`, `prevWord()` |
| Vim 词移动 | 658 | `nextVimWord()`, `endOfVimWord()`, `prevVimWord()` |
| Vim WORD 移动 | 774 | `nextWORD()`, `endOfWORD()`, `prevWORD()` |
| 编辑核心 | 845 | `modifyText()` |
| 删除到行首/行尾 | 880 | `deleteToLineStart()`, `deleteToLineEnd()` |
| Token 删除 | 937 | `deleteTokenBefore()` |
| Vim 字符查找 | 1062 | `findCharacter()` |
| 视口计算 | 172 | `getViewportStartLine()` |

### 4.2 上游依赖文件

- **`src/ink/stringWidth.ts`**：提供 `stringWidth()`，优先使用 `Bun.stringWidth`（带 `ambiguousIsNarrow: true`），回退到基于 `eastAsianWidth`、`emoji-regex`、`strip-ansi` 的 JS 实现。被 `MeasuredText` 和 `Cursor.render()` 大量调用。
- **`src/ink/wrapAnsi.ts`**：提供 ANSI 感知的文本折行，优先使用 `Bun.wrapAnsi`，回退到 `wrap-ansi` npm 包。被 `measureWrappedText()` 调用。
- **`src/utils/intl.ts`**：懒加载并缓存 `Intl.Segmenter` 实例（grapheme 与 word 粒度），以及 `firstGrapheme`、`lastGrapheme` 等辅助函数。避免每次调用都重新构造昂贵的 `Intl.Segmenter` 对象。

### 4.3 下游调用文件

- **`src/hooks/useTextInput.ts`**：主文本输入 Hook，维护 `Cursor` 实例状态，处理键盘事件（方向键、Ctrl 组合键、Home/End 等），并调用 `Cursor` 的方法更新文本。
- **`src/hooks/useVimInput.ts`**：在 Vim 模式下拦截按键，将普通模式/插入模式的命令映射为 `Cursor` 操作。
- **`src/vim/motions.ts`**：解析 Vim 运动命令（如 `w`、`b`、`e`、`$`、`0`、`gg`、`G`、`f{char}`），通过 `Cursor` 的方法计算目标位置。
- **`src/vim/operators.ts`**：解析 Vim 操作符（如 `d`、`c`、`y`），结合 motion 结果调用 `modifyText` 或 kill ring 相关函数。
- **`src/components/PromptInput/PromptInput.tsx`**：接收 `Cursor.render()` 的输出字符串，通过 Ink 的 `<Text>` 组件渲染到终端。
- **`src/hooks/useSearchInput.ts`**：搜索输入框，复用 `useTextInput` 或直接与 `Cursor` 交互。

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

- **`Bun.stringWidth` / `Bun.wrapAnsi`**：在 Bun 运行时提供原生、高性能的字符串宽度计算与 ANSI 折行。若不可用则回退到 npm 实现。
- **`Intl.Segmenter`**：现代 JavaScript 国际化 API，用于 grapheme 和 word 切分。Node.js ≥ 16（with ICU）或 Bun 均支持。
- **`emoji-regex`**、`get-east-asian-width`**、`strip-ansi`**、`wrap-ansi`**：仅在非 Bun 环境下作为回退依赖使用。

### 5.2 数据流交互

```
用户按键
    ↓
useTextInput.ts / useVimInput.ts
    ↓
Cursor.left() / insert() / nextVimWord() / modifyText() 等
    ↓
MeasuredText (NFC + 折行 + 边界缓存)
    ↓
stringWidth.ts / wrapAnsi.ts / intl.ts
    ↓
新 Cursor 实例返回给 Hook
    ↓
PromptInput.tsx 调用 cursor.render() → Ink <Text>
```

### 5.3 Kill Ring 的全局共享

Kill ring 作为模块级变量，天然被所有导入 `Cursor.ts` 的模块共享。这意味着：
- Prompt 输入框、搜索框、任何未来的文本输入框共用同一份 kill ring。
- 连续删除的累积判断（`lastActionWasKill`）也是全局的：在一个输入框删除后切换到另一个输入框继续删除，**不会**被识别为同一次连续删除（因为通常伴随焦点切换导致的其他按键或状态重置）。

---

## 6. 风险、边界与改进建议

### 6.1 全局 Kill Ring 的副作用风险

**风险**：Kill ring 的模块级状态意味着所有输入字段共享剪切历史。虽然这在 Emacs 风格中是预期行为，但在多用户并发场景（如测试并行运行）或同一进程内多个独立会话时，可能导致意外的数据交叉污染。

**建议**：
- 将 kill ring 封装为一个可注入的 `KillRing` 类实例，由 `useTextInput` 在 Hook 初始化时创建并传入 `Cursor` 或相关编辑函数。
- 保留全局单例作为默认行为，但允许调用方通过上下文（Context）或 props 传入隔离的实例，以支持测试并行化和多会话隔离。

### 6.2 不可变 `Cursor` 的内存与性能权衡

**风险**：每次按键都创建新的 `Cursor` 和潜在的新的 `MeasuredText`（当文本变更时）。`MeasuredText` 内部会重新计算 `wrappedLines` 和 grapheme 边界，对于超长文本（如粘贴数千行代码）可能存在延迟。

**建议**：
- `MeasuredText` 已实现了懒加载缓存（`wrappedLines`、`graphemeBoundaries`、`wordBoundariesCache`、`navigationCache`），这是良好的第一步。
- 对于仅移动光标而不修改文本的操作（如 `left()`、`nextWord()`），`measuredText` 引用不变，所有缓存均可复用，性能开销极小。
- 可考虑对 `MeasuredText` 引入持久化数据结构（如增量更新折行结果），但当前复杂度下收益可能不明显，建议先通过性能分析确认瓶颈。

### 6.3 `Intl.Segmenter` 的可用性边界

**风险**：在精简 ICU 的 Node.js 构建环境中，`Intl.Segmenter` 可能不存在。当前代码未对此做显式降级处理，会导致运行时抛出 `Intl.Segmenter is not a constructor`。

**建议**：
- 在 `src/utils/intl.ts` 的 `getGraphemeSegmenter()` 中增加兼容性检查：若 `Intl.Segmenter` 不可用，回退到基于 `Array.from(str)` 或 `[@stdlib/string-grapheme-cluster-break](https://github.com/stdlib-js/string-grapheme-cluster-break)` 的简化实现，并打印一次降级警告。
- 对于 word 切分，可回退到基于 `\b` 或空格的正则，虽然 CJK 支持会下降，但至少能保证基本功能可用。

### 6.4 `[Image #N]` Chip 的正则硬编码

**风险**：`imageRefEndingAt`、`imageRefStartingAt`、`deleteTokenBefore` 中均硬编码了 `/\[Image #\d+\]/` 或类似的正则。若未来 chip 格式变更（如支持多语言前缀、额外属性），需要修改多处代码，容易遗漏。

**建议**：
- 将 chip 的正则模式提取为集中的常量或配置对象（如 `CHIP_PATTERNS = { image: /\[Image #\d+\]/, pasted: /\[Pasted text #\d+...\]/ }`）。
- 进一步抽象出 `ChipRegistry` 概念，允许调用方注册新的原子 token 类型，使 `Cursor` 无需感知具体的业务标记格式。

### 6.5 `selection` 字段的未完成状态

**风险**：`Cursor` 构造函数接受 `selection` 参数（第 156 行），但当前所有方法在返回新 `Cursor` 时均将其硬编码为 `0`。这意味着选区功能尚未实现，但接口已暴露，可能导致未来实现时的大量重构。

**建议**：
- 若近期无选区计划，可考虑移除 `selection` 参数以简化接口；
- 若计划支持，应尽早设计选区与 `modifyText` 的交互语义（如选中状态下输入替换选区内容、Shift+方向键扩展选区等），避免接口债务累积。

### 6.6 `modifyText` 中 NFC 的重复规范化

**风险**：`modifyText` 在计算新光标位置时，对 `insertString` 单独调用 `.normalize('NFC')`（第 857 行）。虽然 `MeasuredText` 构造时也会对整个文本做 NFC，但这里对插入字符串单独规范化是为了正确计算插入后的偏移量。若插入字符串本身已包含组合字符，双重规范化是安全的（幂等），但增加了不必要的计算。

**建议**：
- 由于 NFC 是幂等的，当前逻辑在正确性上无虞。若需优化，可在 `modifyText` 中跳过单独规范化，改为在 `Cursor.fromText` 创建 `MeasuredText` 后，通过 `measuredText.text.length` 反推新偏移量。但需注意：若插入字符串导致与前后文本产生新的组合（如 `e` + 插入 `́` 合并为 `é`），偏移量计算会更复杂。因此当前方案是简单且安全的，建议保留。

### 6.7 折行算法与 `wrapAnsi` 的耦合

**风险**：`measureWrappedText()` 依赖 `wrapAnsi` 的硬折行输出，然后通过 `indexOf(text, searchOffset)` 将折行结果映射回原始文本偏移量。如果 `wrapAnsi` 的实现细节（如对 ANSI 序列的处理、对零宽字符的截断行为）与 `stringWidth` 不完全一致，可能导致 `startOffset` 计算错误或抛出 `"Failed to find wrapped line in text"`。

**建议**：
- 增加对 `measureWrappedText` 的单元测试覆盖，特别是包含 ANSI 颜色码、emoji、CJK、以及混合 `\n` 与自动折行的边界情况。
- 考虑在 `indexOf` 失败时提供更详细的诊断信息（如失败的行内容、searchOffset 值），以便于排查 `wrapAnsi` 版本升级带来的不兼容问题。
