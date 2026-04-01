# src/native-ts/color-diff/index.ts 研究文档

> 研究范围：代码、脚本、配置、测试及必要实现上下文。基于仓库快照中可直接读取的源码与调用链进行分析。

---

## 1. 场景与职责

`src/native-ts/color-diff/index.ts`（999 行）是 Claude Code 中 **纯 TypeScript 实现的语法高亮与差异渲染模块**。它是原生 Rust NAPI 模块 `vendor/color-diff-src` 的完整 TS 移植版本，目的是在无法加载原生二进制（或构建系统选择 TS 路径）时，提供零原生依赖的降级实现。

该模块直接面向终端输出，职责包括：
- **Diff Hunk 渲染**：将 `diff` 格式的 patch hunk（带 `+`/`-`/` ` 标记的行）渲染为带语法高亮、词级差异背景色、行号、标记符的 ANSI 转义字符串数组。
- **普通代码块渲染**：为完整文件内容提供行号 + 语法高亮的终端输出（供 `HighlightedCode` 组件复用）。
- **主题与颜色模式适配**：根据终端能力（`COLORTERM`）与用户主题选择（dark/light/ansi/daltonized）输出 `truecolor`、`256 色` 或 `ANSI 16 色` 转义序列。
- **API 兼容层**：导出类型与函数签名与原生 NAPI 模块的 `.d.ts` 完全一致，使上层调用方无需感知底层实现是 Rust 还是 TS。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| `ColorDiff` 类 | 接收一个 `Hunk`（含 oldStart/newStart/lines 等），输出可直接写入终端的 ANSI 字符串数组。支持词级 diff 高亮（在行内标出具体修改的单词范围）。 |
| `ColorFile` 类 | 接收完整文件内容与路径，输出带行号和语法高亮的 ANSI 字符串数组。用于 `/view`、代码预览等非 diff 场景。 |
| `getSyntaxTheme()` | 返回当前映射到的语法主题名与来源信息。TS 版目前为 stub，仅做默认映射，不支持通过 `BAT_THEME` 切换外部主题。 |
| `getNativeModule()` | 将 `ColorDiff` / `ColorFile` / `getSyntaxTheme` 打包成与 NAPI 模块一致的对象，供动态加载器使用。 |
| `__test` 导出 | 暴露内部纯函数（`tokenize`、`wordDiffStrings`、`ansi256FromRgb` 等），供外部单元测试注入与验证。 |

---

## 3. 具体技术实现

### 3.1 模块加载策略：懒加载 highlight.js

源码第 24–43 行采用**延迟 `require`** 加载 `highlight.js`：

```ts
let cachedHljs: HLJSApi | null = null
function hljs(): HLJSApi {
  if (cachedHljs) return cachedHljs
  const mod = require('highlight.js')
  cachedHljs = 'default' in mod && mod.default ? mod.default : mod
  return cachedHljs!
}
```

**原因**：`highlight.js` 全量 bundle 会在 `require` 时注册 190+ 语言语法，带来约 50MB 堆内存与 100–200ms（macOS）/数倍（Windows）的初始化开销。若使用顶层 `import`，任何间接引用该模块的代码（包括测试 preload）都会在模块求值时支付此成本。Windows CI 上这曾导致 GC 暂停并触发 `beforeEach/afterEach` 超时（PR #24150）。懒加载将成本推迟到首次真正渲染时，与原生 NAPI 模块的 `dlopen` 懒加载策略保持一致。

### 3.2 颜色系统与 ANSI 转义

**核心类型**（第 75–78 行）：
- `Color = { r, g, b, a }`
- `Style = { foreground: Color, background: Color }`
- `Block = [Style, string]` —— 渲染的最小单元

**颜色模式检测**（`detectColorMode`，第 95–99 行）：
- 主题名包含 `ansi` → `ansi`
- 环境变量 `COLORTERM` 为 `truecolor` 或 `24bit` → `truecolor`
- 否则 → `color256`

**Alpha 通道语义**（第 92–138 行）：
- `a === 0`：表示调色板索引，编码在 `.r` 中（bat 的 ansi-theme 约定）。根据索引值输出 `\x1b[30–37m`、`\x1b[90–97m` 或 `\x1b[38;5;{idx}m`。
- `a === 1`：终端默认色，输出 `\x1b[39m` / `\x1b[49m`。
- `a === 255`：正常 RGB 色，按模式输出 `\x1b[38;2;R;G;Bm` 或 `\x1b[38;5;{ansi256}m`。

**ANSI 256 近似算法**（`ansi256FromRgb`，第 101–127 行）：
- 手动实现了 Rust `ansi_colours::ansi256_from_rgb` 的移植。
- 使用 6×6×6 色立方体（`CUBE_LEVELS = [0, 95, 135, 175, 215, 255]`）与 24 级灰阶 ramp（232–255）分别计算候选色。
- 通过欧氏距离比较选择感知上更近的索引；对极端灰度值做边界修正（如 248,248,242 映射到立方体白 231 而非 ramp 顶 255）。

### 3.3 主题与语法作用域映射

**主题结构**（`Theme`，第 170–180 行）包含：
- 增删行的背景色（`addLine` / `deleteLine`）
- 增删词的背景色（`addWord` / `deleteWord`）
- 装饰色（`addDecoration` / `deleteDecoration`，用于 `+`/`-` 标记与行号）
- 前景色、背景色
- `scopes: Record<string, Color>` —— highlight.js scope 到终端颜色的映射表

**默认主题映射**（`defaultSyntaxThemeName`，第 182–186 行）：
- 暗色主题 → `Monokai Extended`
- 亮色主题 → `GitHub`
- ANSI 主题 → `ansi`

**Scope 颜色表**（第 188–243 行）：
- `MONOKAI_SCOPES` 与 `GITHUB_SCOPES` 中的颜色均通过测量 Rust 原生模块的 syntect 输出获得，确保 TS 降级实现与原生实现在视觉上尽量一致。
- 针对 highlight.js 将 `const`/`let`/`function`/`class` 等统一标记为 `keyword` 的问题，代码维护了一个 `STORAGE_KEYWORDS` 集合（第 248–265 行），在 `scopeColor()` 中将这些词重新映射到 `_storage` 的青色，以还原 syntect 的 `storage.type` 着色。

### 3.4 语言检测

`detectLanguage()`（第 422–451 行）模拟了 bat 的 `SyntaxMapping` + syntect 的扩展名查找：
- **文件名匹配**：`Dockerfile`、`Makefile`、`Rakefile`、`Gemfile`、`CMakeLists` 等。
- **扩展名匹配**：直接通过 `hljs().getLanguage(ext)` 查询。
- **Shebang / 首行检测**：处理 `#!/bin/bash`、`#!/usr/bin/env python`、`<?php`、`<?xml` 等。
- 若检测失败则返回 `null`，后续按纯文本渲染。

### 3.5 词级 Diff 算法

**分词策略**（`tokenize`，第 550–574 行）：
- 将字符串切分为三类 token：
  1. 单词/标识符运行：`/[\p{L}\p{N}_]/u`
  2. 空白运行：`/\s/`
  3. 单个标点/其他字符（按 codepoint 推进，支持 surrogate pairs）
- 该策略与 Rust 版 `diffWordsWithSpace` 的分词行为对齐。

**相邻行配对**（`findAdjacentPairs`，第 576–602 行）：
- 扫描 hunk 的 marker 数组，找到连续的 `-` 块及其后紧跟的连续 `+` 块。
- 按顺序一一配对（`min(delCount, addCount)`），返回 `[delIdx, addIdx]` 数组。
- **限制**：仅处理“`-` 块后紧跟 `+` 块”的最简局部模式；对于重排、插入上下文行、数量不等的复杂修改，会退化为整行 diff（无词级高亮）。

**差异计算与阈值**（`wordDiffStrings`，第 604–636 行）：
- 使用 `diffArrays(oldTokens, newTokens)`（来自 `diff` npm 包）计算 token 级差异。
- 统计变更字符长度 `changedLen` 与总长度 `totalLen`。
- 若 `changedLen / totalLen > CHANGE_THRESHOLD (0.4)`，则认为改动过大，放弃词级高亮，返回空范围数组。这避免了整行重写时满屏高亮造成的视觉噪音。

### 3.6 渲染管线（Highlight 变换流水线）

对于 `ColorDiff` 与 `ColorFile`，单条逻辑行的渲染遵循同一变换流水线（第 648–826 行）：

1. **`highlightLine()`** —— 调用 `hljs().highlight()` 将代码行解析为 `Block[]`。若语言未知或 hljs emitter 形状不匹配，则回退到默认样式。
2. **`removeNewlines()`** —— 将 Block 中的 `\n` 拆分并过滤空串，防止高亮器返回的多行 token 破坏行结构。
3. **`applyBackground()`** —— 根据词级 diff 的 `Range[]`，将 Block 中对应片段的背景色从 `lineBg` 切换为 `wordBg`。
4. **`wrapText()`** —— 按终端宽度 `effectiveWidth` 做硬折行。使用 `stringWidth()` 计算显示宽度，支持 emoji、CJK、组合字符。折行时保留每个片段的样式；对变更行（`+`/`-`）在折行后补空格以延伸背景色到右边缘。
5. **`dimContent()`** —— 仅在 `ansi` 模式下对删除行整体加 `\x1b[2m`（dim），使删除行在有限颜色空间中更易区分。
6. **`addMarker()`** —— 在每一行最左侧插入 `+`/`-`/` ` 标记符，使用对应装饰色与行背景色。
7. **`addLineNumber()`** —— 在标记符右侧插入右对齐行号（` ${lineNumber.padStart(maxDigits)} `）。对上下文行（` `）与无标记行使用 `\x1b[2m` dim（除非 `fullDim`）。
8. **`intoLines()`** —— 将 `Block[][]` 转换为最终的 ANSI 转义字符串数组。

**`ColorDiff.render()` 特有逻辑**（第 842–932 行）：
- 第一遍扫描：为 hunk 的每一行分配 `marker` 与 `lineNumber`（`+` 行使用 newLine 计数器，`-` 行使用 oldLine 计数器，` ` 行两者同步递增）。
- 第二遍扫描：仅在 `!dim` 时计算词级 diff；删除行（`-`）不做语法高亮（直接按默认样式输出），新增/上下文行做高亮。
- `effectiveWidth = max(1, width - maxDigits - 2 - 1)`，为行号、标记符、两侧空格预留列宽。

**`ColorFile.render()` 特有逻辑**（第 935–968 行）：
- 将完整代码按 `\n` 拆分，丢弃 trailing empty line（与 Rust `.lines()` 行为对齐）。
- 每行独立高亮，无 marker，行号连续递增。
- `effectiveWidth = max(1, width - maxDigits - 2)`，无 marker 列，因此少减 1。

---

## 4. 关键代码路径与文件引用

### 4.1 目标文件内部关键路径

| 功能 | 行号范围 | 说明 |
|------|----------|------|
| 懒加载 hljs | 24–43 | `require('highlight.js')` 延迟加载与 interop 处理 |
| 公共 API 类型 | 52–69 | `Hunk`、`SyntaxTheme`、`NativeModule` |
| 颜色模式检测 | 95–99 | `detectColorMode` |
| ANSI 256 转换 | 101–127 | `ansi256FromRgb` |
| ANSI 转义生成 | 129–145 | `colorToEscape` |
| 主题构建 | 282–362 | `buildTheme`（dark/light/ansi/daltonized） |
| 语言检测 | 422–451 | `detectLanguage` |
| 词级 diff 分词 | 550–574 | `tokenize` |
| 相邻行配对 | 576–602 | `findAdjacentPairs` |
| 词级范围计算 | 604–636 | `wordDiffStrings` |
| 渲染流水线 | 648–826 | `removeNewlines` / `wrapText` / `addLineNumber` / `applyBackground` / `intoLines` |
| Diff 渲染入口 | 842–932 | `ColorDiff.render()` |
| 文件渲染入口 | 935–968 | `ColorFile.render()` |
| 测试钩子 | 990–999 | `__test` 导出 |

### 4.2 上游调用链

```
src/components/StructuredDiff.tsx
  └── import { expectColorDiff } from './StructuredDiff/colorDiff.js'
        └── src/components/StructuredDiff/colorDiff.ts
              └── import { ColorDiff, ColorFile, getSyntaxTheme } from 'color-diff-napi'
                    └── (构建系统别名) src/native-ts/color-diff/index.ts

src/components/HighlightedCode.tsx
  └── import { expectColorFile } from './StructuredDiff/colorDiff.js'
        └── (同上)

src/components/ThemePicker.tsx
  └── import { getColorModuleUnavailableReason, getSyntaxTheme } from './StructuredDiff/colorDiff.js'
        └── (同上)
```

### 4.3 同层依赖文件

| 文件 | 用途 |
|------|------|
| `src/ink/stringWidth.ts` | 终端显示宽度计算（Bun.stringWidth 优先，JS fallback 处理 emoji/CJK/组合字符） |
| `src/utils/log.ts` | 错误日志收集与上报（`logError`），用于 hljs emitter 形状不匹配时的降级告警 |

---

## 5. 依赖与外部交互

### 5.1 运行时依赖（npm 包）

- **`diff`**：`diffArrays` 用于 token 级差异计算。
- **`highlight.js`**：语法高亮核心库。通过 `require()` 懒加载，全量 bundle 含 190+ 语言。

> 注：当前仓库快照中未包含 `package.json`，上述依赖关系从源码 `import` / `require` 语句推断得出。

### 5.2 本地模块依赖

- `../../ink/stringWidth.js` → `src/ink/stringWidth.ts`
- `../../utils/log.js` → `src/utils/log.ts`

### 5.3 环境变量交互

| 变量 | 作用 |
|------|------|
| `COLORTERM` | 决定颜色模式。`truecolor` / `24bit` → truecolor，否则 fallback 到 256 色。 |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` / `BAT_THEME` | `getSyntaxTheme()` 读取用于诊断展示，但 TS 版目前不根据该变量实际切换主题（stub）。 |
| `process.env`（通过 `colorDiff.ts`） | `CLAUDE_CODE_SYNTAX_HIGHLIGHT` 若被设为 falsy 值（如 `0`、`false`），`colorDiff.ts` 会返回 `null`，使上层完全禁用语法高亮并回退到 Fallback 组件。 |

### 5.4 构建/别名交互

源码中 `colorDiff.ts` 以 `'color-diff-napi'` 作为模块标识符导入，而仓库快照中**未找到** `tsconfig.json`、`vite.config.ts`、`bunfig.toml` 或 `package.json` 等显式配置该别名的文件。该映射关系由仓库外部的构建系统（或上层 monorepo 配置）负责解析。`src/native-ts/color-diff/index.ts` 通过保持与原生模块完全一致的类型签名，确保别名切换时零调用方改动。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

1. **懒加载 `require` 的 ESM 兼容性**
   - 代码使用 `require('highlight.js')` 进行懒加载。在纯 ESM（无 bundler 转换）环境下可能抛出 `ReferenceError: require is not defined`。虽然当前项目使用 bun 构建，但若未来迁移到严格 ESM Node 环境，需要改为动态 `import()`。

2. **hljs emitter 形状版本敏感**
   - `hasRootNode()` 对 `result.emitter` 做运行时类型守卫。若 highlight.js 大版本升级改变了内部 emitter 结构（如 `_emitter` vs `emitter`、`scope` vs `kind`），语法高亮会静默降级为灰度文本。目前通过 `logError` 仅记录一次错误，但用户侧无感知。

3. **`prefixContent` 接口悬空**
   - `ColorDiff` 构造器接收 `prefixContent`（用于多行字符串/注释的上下文预热），但 `render()` 中仅执行 `void this.prefixContent`，未实际使用。这导致跨行语法状态（如多行注释、模板字符串）无法延续，高亮准确性相比 Rust 版（可能支持状态预热）存在退化。

4. **词级 diff 配对策略过于简单**
   - `findAdjacentPairs` 只处理“连续 `-` 后紧跟连续 `+`”的局部模式。对于以下常见场景会完全退化为行级 diff：
     - 删除与新增数量不等（如删 2 增 3）
     - 上下文行插入在删除与新增之间
     - 代码块重排（移动）
   - 这导致复杂 patch 的词级高亮覆盖率显著下降。

5. **`dim=true` 时完全关闭词级 diff**
   - 当 `dim=true`（如“被拒绝的修改预览”）时，词级 diff 被整体跳过。虽然视觉上更克制，但信息密度也显著下降，用户难以快速定位具体改动点。

6. **`wrapText` 的强制前进可能溢出**
   - 当某字符的显示宽度大于剩余可用宽度且当前行尚无内容时，`wrapText` 会强制切分一个 codepoint 并放入当前行（第 686–691 行）。这会导致该行实际宽度超过 `effectiveWidth`，可能破坏终端布局对齐。

7. **`stringWidth` 平台差异**
   - `src/ink/stringWidth.ts` 优先使用 `Bun.stringWidth`，在 Node 环境回退到 JS 实现。两者对复杂脚本（如 Devanagari 连字）的宽度计算存在差异（注释中明确提到 JS fallback 可能返回 1 而 Bun 返回 2），可能导致折行位置不一致。

### 6.2 边界条件

- **空 hunk / 空文件**：`maxLineNumber` 使用 `Math.max(0, ...)` 保证至少为 0；`effectiveWidth` 使用 `Math.max(1, ...)` 保证至少为 1，避免除以零或负宽度。
- **未知语言**：`highlightLine` 捕获异常并回退到默认样式；`detectLanguage` 返回 `null` 时同样走默认样式。
- **超宽终端 / 极窄终端**：`wrapText` 在极窄终端（`width <= gutterWidth`）时由调用方（`StructuredDiff.tsx`）防御性跳过 gutter 分割；`ColorDiff.render` 内部仍尝试折行。
- **ANSI 模式颜色匮乏**：ANSI 主题下 `scopes` 映射表远小于 truecolor 主题，大量 scope 会 fallback 到 `theme.foreground`，导致高亮层次明显减少。

### 6.3 改进建议

1. **补全 `prefixContent` 状态预热**
   - 虽然 highlight.js 默认是 Stateless per call，但可以通过在 `render()` 前对 `prefixContent` 的每一行调用 `highlightLine()` 来“预热”语言检测与潜在的多行解析器状态（若未来 hljs 版本支持多行模式），或至少保持与 Rust 版的行为对齐。

2. **优化词级 diff 配对算法**
   - 将 `findAdjacentPairs` 升级为基于 LCS（最长公共子序列）或 Myers diff 的全局配对策略，允许非 1:1 的删除/新增映射，并支持跳过上下文行。可显著提升复杂 patch 的词级高亮覆盖率。

3. **增加单元测试覆盖**
   - 当前仓库快照中未找到针对该模块的 `*.test.*` 文件。建议为以下内部函数补充测试（通过 `__test` 导出注入）：
     - `tokenize`：验证 Unicode、emoji、surrogate pairs 的分词正确性。
     - `wordDiffStrings`：验证 40% 阈值行为、范围计算准确性。
     - `ansi256FromRgb`：与 Rust `ansi_colours` 的输出做对照采样测试。
     - `wrapText`：验证 CJK、emoji、ANSI 序列存在时的折行位置。
     - `detectLanguage`：验证 shebang、扩展名、文件名映射。

4. **细化 `dim=true` 的词级 diff 策略**
   - 可考虑在 `dim=true` 时仍计算词级 diff，但使用更淡化的装饰色（如降低饱和度或仅使用下划线），在保持视觉克制的同时保留信息密度。

5. **构建配置内聚化**
   - 将 `color-diff-napi` → `src/native-ts/color-diff/index.ts` 的别名映射以注释或文档形式固化到仓库可见的配置中（如 `tsconfig.json` paths 或 `bunfig.toml`），降低新成员理解成本。

6. **主题 stub 补全**
   - 若 `BAT_THEME` 或 `CLAUDE_CODE_SYNTAX_HIGHLIGHT` 指向已知主题，可在 `getSyntaxTheme()` 中真正返回对应主题名，而非始终返回默认值。这需要维护一个 bat 主题名到 hljs 主题的映射表。

---

*文档生成时间：2026-04-01*  
*基于文件版本：src/native-ts/color-diff/index.ts（999 行）*
