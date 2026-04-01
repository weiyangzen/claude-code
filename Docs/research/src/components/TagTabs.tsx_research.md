# TagTabs.tsx 研究文档

## 场景与职责

`TagTabs.tsx` 是 Claude Code 终端 UI 中用于**标签过滤导航**的展示组件。它出现在用户通过 `LogSelector` 浏览可恢复会话（resume sessions）时，当会话列表中存在多个标签（tag）时，提供一组水平排列的标签页，让用户按标签快速筛选会话。组件左侧显示 "Resume" 或 "Resume (All Projects)" 提示，右侧显示当前选中标签及可切换的标签页；当标签过多超出可用宽度时，会以箭头 + 数字的形式提示隐藏的标签数量。

## 功能点目的

1. **标签分页与筛选入口**：将唯一的标签列表渲染为可交互的标签页（包含一个固定的 "All" 页），支持通过外部 `selectedIndex` 控制当前选中项。
2. **宽度自适应与溢出处理**：根据 `availableWidth` 动态计算每个标签的显示宽度，对超长标签进行截断，并在总宽度超出时只显示一个以选中标签为中心的滑动窗口，左右以箭头提示隐藏数量。
3. **精确字符宽度计算**：使用 `stringWidth` 计算终端实际显示宽度（正确处理 Unicode、emoji、CJK 等宽字符），确保布局计算与终端渲染一致。
4. **键盘操作提示**：右侧始终显示 `(tab to cycle)` 或带隐藏数量的提示，引导用户通过 Tab 键循环切换标签。

## 具体技术实现

### 关键流程

1. **常量定义与宽度预算**
   - 定义了 `ALL_TAB_LABEL = 'All'`、`TAB_PADDING = 2`（左右各一个空格）、`HASH_PREFIX_LENGTH = 1`（非 All 标签前加 `#`）。
   - 计算左右提示区的最坏情况宽度：`LEFT_ARROW_WIDTH`（`← NN `）和 `RIGHT_HINT_WIDTH_WITH_COUNT` / `RIGHT_HINT_WIDTH_NO_COUNT`。
   - 可用标签区宽度 = `availableWidth - resumeLabelWidth - rightHintWidth - 2`（间隙）。

2. **标签宽度计算**
   - `getTabWidth(tab, maxWidth)`：All 标签固定宽度；其他标签用 `stringWidth` 计算实际宽度，并受 `maxSingleTabWidth` 限制（至少为 `maxTabsWidth / 2`）。
   - `truncateTag(tag, maxWidth)`：在预留 `TAB_PADDING + HASH_PREFIX_LENGTH` 后，用 `truncateToWidth` 截断标签文本。

3. **滑动窗口算法**
   - 若所有标签总宽度 ≤ `maxTabsWidth`，全部显示。
   - 否则以 `safeSelectedIndex` 为中心，先计入该标签宽度，然后交替向左右扩展，直到再扩展会超出 `effectiveMaxWidth = maxTabsWidth - LEFT_ARROW_WIDTH` 为止。
   - 最终得到 `startIndex` 和 `endIndex`，并计算 `hiddenLeft` / `hiddenRight`。

4. **渲染**
   - 使用 `Box` + `Text`（Ink 组件）水平排列。
   - 选中标签以 `backgroundColor="suggestion" color="inverseText" bold` 高亮。
   - 非 All 标签渲染为 `#{truncateTag(...)}`，All 标签直接渲染文本。

### 数据结构

- `Props`：
  - `tabs: string[]` — 标签字符串数组（不含 "All"）。
  - `selectedIndex: number` — 当前选中索引（外部控制）。
  - `availableWidth: number` — 可用于渲染的总列数。
  - `showAllProjects?: boolean` — 是否显示 "Resume (All Projects)" 而非 "Resume"。

### 协议/命令

- 无网络协议或子进程命令。
- 纯展示组件，交互逻辑（Tab 键切换）由父组件 `LogSelector` 通过 `useInput` 处理并传入新的 `selectedIndex`。

## 关键代码路径与文件引用

- **本文件**：`src/components/TagTabs.tsx`
- **父组件/调用方**：`src/components/LogSelector.tsx`（通过 `TagTabs` 渲染标签栏，并管理 `selectedTagIndex` 状态）
- **依赖工具**：
  - `src/ink/stringWidth.ts` — 终端显示宽度计算（Bun.stringWidth 或 JS fallback）。
  - `src/utils/format.ts`（通过 `./truncate.js` 导出）— `truncateToWidth` 截断函数。
  - `src/ink.js` — `Box`、`Text` 组件。

## 依赖与外部交互

| 依赖 | 路径 | 用途 |
|------|------|------|
| `stringWidth` | `src/ink/stringWidth.ts` | 精确计算字符串在终端中的显示宽度 |
| `Box`, `Text` | `src/ink.js` | Ink 布局与文本渲染 |
| `truncateToWidth` | `src/utils/format.ts`（re-export from `./truncate.js`） | 按显示宽度截断字符串 |
| `LogSelector` | `src/components/LogSelector.tsx` | 唯一调用方，传入标签列表与选中索引 |

## 风险、边界与改进建议

### 风险与边界

1. **宽度计算与渲染不一致**：`stringWidth` 和 `truncateToWidth` 必须与 Ink/终端的实际渲染逻辑严格一致。若终端字体或 Ink 版本变更导致宽度偏差，可能出现标签重叠或提前换行。
2. **极端窄终端**：当 `availableWidth` 极小（< 约 30 列）时，`maxTabsWidth` 可能为负值，代码中未显式兜底，虽然 `Math.max(0, ...)` 在部分地方使用，但整体布局可能异常。
3. **标签数量爆炸**：`MAX_OVERFLOW_DIGITS = 2` 假设隐藏标签数 ≤ 99。若标签超过 99 个，右侧提示宽度计算会低估，导致布局错位。
4. **无键盘交互内聚**：标签切换的键盘处理分散在 `LogSelector` 中，增加了跨文件理解成本。

### 改进建议

1. **增加最小宽度保护**：在计算 `maxTabsWidth` 和 `maxSingleTabWidth` 时增加更明确的 `Math.max(0, ...)` 或最小合理值断言，防止负数宽度。
2. **动态溢出位数**：将 `MAX_OVERFLOW_DIGITS` 改为根据实际 `tabs.length` 动态计算，避免 99+ 标签时的宽度低估。
3. **提取滑动窗口逻辑为独立纯函数**：当前窗口算法与渲染耦合，可提取为 `getVisibleTabWindow(tabs, tabWidths, selectedIndex, maxWidth)` 纯函数，便于单元测试。
4. **添加单元测试**：针对 `getTabWidth`、`truncateTag` 和滑动窗口算法补充测试，覆盖 CJK、emoji、超长标签、窄终端等场景。
