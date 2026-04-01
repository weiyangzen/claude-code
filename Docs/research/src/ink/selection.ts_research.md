# selection.ts 研究文档

## 场景与职责

`selection.ts` 是 Ink 终端 UI 框架的文本选择模块，负责管理全屏模式下的鼠标/键盘文本选择状态和行为。它实现了类似原生终端的文本选择体验。

**核心职责：**
1. **选择状态管理** - 跟踪锚点(anchor)、焦点(focus)、拖拽状态
2. **选择模式** - 支持字符模式、单词模式、行模式
3. **文本提取** - 从屏幕缓冲区提取选中的纯文本
4. **滚动跟踪** - 处理拖拽到屏幕边缘时的自动滚动
5. **软换行处理** - 正确处理自动换行的文本提取
6. **URL 检测** - 识别纯文本 URL（OSC 8 超链接的备用）
7. **选择高亮应用** - 将选择状态应用到屏幕缓冲区（视觉反馈）

**在架构中的位置：**
- 被 `ink.tsx` 主控制器管理生命周期和事件处理
- 被 `components/App.tsx` 处理鼠标事件
- 被 `hooks/useCopyOnSelect.ts` 监听选择变化
- 与 `screen.ts` 交互读取/修改屏幕缓冲区

---

## 功能点目的

### 1. 选择状态模型

**目的：** 精确跟踪用户的选择操作

**状态组成：**
```typescript
type SelectionState = {
  anchor: Point | null       // 鼠标按下位置
  focus: Point | null        // 当前拖拽位置（null 表示点击无拖拽）
  isDragging: boolean        // 是否正在拖拽
  anchorSpan: { lo, hi, kind } | null  // 单词/行模式的初始范围
  scrolledOffAbove: string[] // 滚动出视图上方的文本
  scrolledOffBelow: string[] // 滚动出视图下方的文本
  scrolledOffAboveSW: boolean[]  // 软换行标记（上方）
  scrolledOffBelowSW: boolean[]  // 软换行标记（下方）
  virtualAnchorRow?: number  // 钳制前的锚点行（用于滚动恢复）
  virtualFocusRow?: number   // 钳制前的焦点行
  lastPressHadAlt: boolean   // 上次点击是否有 Alt 键
}
```

**设计决策：**
- anchor/focus 模型支持双向选择（从后往前选）
- `focus === null` 表示点击无拖拽，不触发复制
- 虚拟行跟踪支持滚动后的位置恢复

### 2. 选择模式

**字符模式（默认）：**
- 逐字符选择
- 拖拽时精确到单元格

**单词模式（双击）：**
- 基于字符类别（字母/数字/标点/空格）
- 匹配 iTerm2 默认的单词字符集：`[\p{L}\p{N}_/.\-+~\\]`
- 拖拽时扩展到新位置的单词边界

**行模式（三击）：**
- 整行选择
- 拖拽时扩展到新行

### 3. 滚动跟踪

**目的：** 处理拖拽到屏幕边缘时的内容滚动

**机制：**
1. 拖拽到边缘触发 `scrollBy`，内容滚动
2. 锚点/焦点需要"跟随"文本移动
3. 滚动出视图的内容被捕获到 `scrolledOffAbove/Below`
4. 反向滚动时从累积器恢复内容

**虚拟行跟踪：**
```
场景：选择第 5 行 → PgDn 滚动 10 行
- 锚点从第 5 行钳制到第 0 行
- virtualAnchorRow = 5（记录真实位置）
- scrolledOffAbove 捕获第 5-9 行

后续 PgUp 滚动 -10 行：
- 从 virtualAnchorRow=5 计算新位置
- 正确恢复锚点到第 5 行
- 从 scrolledOffAbove 恢复内容
```

### 4. 软换行处理

**目的：** 复制时合并自动换行的逻辑行

**机制：**
- `screen.softWrap[row] > 0` 表示该行是上一行的延续
- `extractRowText()` 根据 softWrap 决定是否在行尾添加换行
- `joinRows()` 合并软换行相关的行

### 5. URL 检测

**目的：** 当单元格没有 OSC 8 超链接时，检测纯文本 URL

**算法：**
1. 从点击位置向左右扩展，匹配 URL 字符集
2. 查找最近的 scheme 锚点（http://, https://, file://）
3. 剥离尾部标点（考虑括号平衡）
4. 验证点击位置在 URL 范围内

---

## 具体技术实现

### 单词边界检测

```typescript
// Unicode-aware 单词字符
const WORD_CHAR = /[\p{L}\p{N}_.\/\-+~\\]/u

function charClass(c: string): 0 | 1 | 2 {
  if (c === ' ' || c === '') return 0  // 空格
  if (WORD_CHAR.test(c)) return 1      // 单词字符
  return 2                              // 其他标点
}

function wordBoundsAt(screen, col, row): { lo, hi } | null {
  // 1. 如果点击在 SpacerTail，回退到 Wide 字符头部
  // 2. 获取起始字符类别
  // 3. 向左扩展直到类别变化或 noSelect
  // 4. 向右扩展直到类别变化或 noSelect
  // 5. 跳过 SpacerTail 单元格
}
```

### 选择扩展逻辑

```typescript
function extendSelection(s, screen, col, row): void {
  if (!s.isDragging || !s.anchorSpan) return
  
  // 获取当前鼠标位置的单词/行范围
  if (s.anchorSpan.kind === 'word') {
    mLo = wordBoundsAt(...).lo ?? col
    mHi = wordBoundsAt(...).hi ?? col
  } else {
    mLo = { col: 0, row }
    mHi = { col: width - 1, row }
  }
  
  // 确定选择方向
  if (mHi < s.anchorSpan.lo) {
    // 鼠标在锚点范围之前：向后扩展
    s.anchor = s.anchorSpan.hi
    s.focus = mLo
  } else if (mLo > s.anchorSpan.hi) {
    // 鼠标在锚点范围之后：向前扩展
    s.anchor = s.anchorSpan.lo
    s.focus = mHi
  } else {
    // 鼠标在锚点范围内：仅选择锚点范围
    s.anchor = s.anchorSpan.lo
    s.focus = s.anchorSpan.hi
  }
}
```

### 滚动位移计算

```typescript
function shiftSelection(s, dRow, minRow, maxRow, width): void {
  // 1. 计算虚拟行位置（考虑之前的钳制）
  const vAnchor = (s.virtualAnchorRow ?? s.anchor.row) + dRow
  const vFocus = (s.virtualFocusRow ?? s.focus.row) + dRow
  
  // 2. 检查是否两端都移出同一边界
  if ((vAnchor < minRow && vFocus < minRow) || 
      (vAnchor > maxRow && vFocus > maxRow)) {
    clearSelection(s)  // 选择完全移出视图，清除
    return
  }
  
  // 3. 计算债务（超出边界的距离）
  const oldAboveDebt = max(0, minRow - oldMin)
  const newAboveDebt = max(0, minRow - min(vAnchor, vFocus))
  
  // 4. 债务减少时，从累积器恢复行
  if (newAboveDebt < oldAboveDebt) {
    s.scrolledOffAbove.length -= (oldAboveDebt - newAboveDebt)
  }
  
  // 5. 债务增加时，新行会被 captureScrolledRows 捕获
  
  // 6. 钳制到视图边界
  s.anchor = shift(s.anchor, vAnchor)
  s.focus = shift(s.focus, vFocus)
  
  // 7. 更新虚拟行（如果仍在边界外）
  s.virtualAnchorRow = vAnchor < minRow || vAnchor > maxRow ? vAnchor : undefined
}
```

### 文本提取

```typescript
function getSelectedText(s, screen): string {
  const lines: string[] = []
  
  // 1. 添加 scrolledOffAbove 的行
  for (i in s.scrolledOffAbove) {
    joinRows(lines, s.scrolledOffAbove[i], s.scrolledOffAboveSW[i])
  }
  
  // 2. 添加视图中选择范围内的行
  for (row from start.row to end.row) {
    const text = extractRowText(screen, row, rowStart, rowEnd)
    joinRows(lines, text, sw[row] > 0)
  }
  
  // 3. 添加 scrolledOffBelow 的行
  for (i in s.scrolledOffBelow) {
    joinRows(lines, s.scrolledOffBelow[i], s.scrolledOffBelowSW[i])
  }
  
  return lines.join('\n')
}

function extractRowText(screen, row, colStart, colEnd): string {
  // 1. 根据 softWrap 确定内容结束位置
  const contentEnd = screen.softWrap[row + 1] || 0
  const lastCol = contentEnd > 0 ? min(colEnd, contentEnd - 1) : colEnd
  
  // 2. 收集字符（跳过 noSelect 和 spacer）
  let line = ''
  for (col from colStart to lastCol) {
    if (noSelect[idx] === 1) continue
    const cell = cellAt(screen, col, row)
    if (cell.width === SpacerTail || cell.width === SpacerHead) continue
    line += cell.char
  }
  
  // 3. 软换行行保留尾部空格，其他行去除
  return contentEnd > 0 ? line : line.replace(/\s+$/, '')
}
```

### URL 检测

```typescript
function findPlainTextUrlAt(screen, col, row): string | undefined {
  // 1. 获取 URL 字符范围（ASCII 可打印字符，排除特定分隔符）
  const URL_BOUNDARY = new Set([...'<>"\'` '])
  
  // 2. 从点击位置向左右扩展
  let lo = col, hi = col
  while (lo > 0 && isUrlChar(cellAt(lo-1))) lo--
  while (hi < width-1 && isUrlChar(cellAt(hi+1))) hi++
  
  // 3. 提取 token 并查找 scheme
  const token = extractChars(lo, hi)
  const schemeRe = /(?:https?|file):\/\//g
  
  // 4. 找到包含点击位置的最右侧 scheme
  let urlStart = -1, urlEnd = token.length
  for (let m; (m = schemeRe.exec(token)); ) {
    if (m.index > clickIdx) { urlEnd = m.index; break }
    urlStart = m.index
  }
  
  // 5. 剥离尾部标点（考虑括号平衡）
  let url = token.slice(urlStart, urlEnd)
  while (url.length > 0) {
    const last = url.at(-1)
    if ('.,;:!?'.includes(last)) { url = url.slice(0, -1); continue }
    // 检查括号平衡...
  }
  
  // 6. 验证点击位置在 URL 内
  if (clickIdx >= urlStart + url.length) return undefined
  return url
}
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 位置 | 说明 |
|------|------|------|
| `createSelectionState()` | line 65-77 | 创建初始选择状态 |
| `startSelection()` | line 79-98 | 开始选择（鼠标按下）|
| `updateSelection()` | line 100-114 | 更新焦点（鼠标移动）|
| `finishSelection()` | line 116-120 | 结束拖拽（鼠标释放）|
| `clearSelection()` | line 122-134 | 清除选择 |
| `selectWordAt()` | line 240-254 | 双击选择单词 |
| `selectLineAt()` | line 368-380 | 三击选择整行 |
| `extendSelection()` | line 389-421 | 扩展选择（单词/行模式拖拽）|
| `moveFocus()` | line 442-450 | 键盘移动焦点 |
| `shiftSelection()` | line 470-565 | 键盘滚动时位移选择 |
| `shiftAnchor()` | line 573-602 | 仅位移锚点（拖拽滚动）|
| `shiftSelectionForFollow()` | line 625-674 | 自动跟随滚动时位移 |
| `hasSelection()` | line 676-678 | 检查是否有有效选择 |
| `selectionBounds()` | line 684-692 | 获取规范化的选择范围 |
| `isCellSelected()` | line 698-710 | 检查单元格是否被选中 |
| `getSelectedText()` | line 773-795 | 提取选中文本 |
| `captureScrolledRows()` | line 813-875 | 捕获滚动出视图的行 |
| `applySelectionOverlay()` | line 893-917 | 应用选择高亮到屏幕 |
| `findPlainTextUrlAt()` | line 272-359 | 检测纯文本 URL |

### 导入依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `./layout/geometry.js` | clamp | 数值钳制 |
| `./screen.js` | CellWidth, cellAt, cellAtIndex, type Screen, type StylePool, setCellStyleId | 屏幕操作 |

### 被调用方

| 模块 | 调用内容 | 说明 |
|------|----------|------|
| `ink.tsx` | 几乎所有函数 | 主控制器管理选择生命周期 |
| `components/App.tsx` | startSelection, updateSelection, finishSelection, selectWordAt, selectLineAt, extendSelection, clearSelection | 鼠标事件处理 |
| `hooks/useCopyOnSelect.ts` | hasSelection, getSelectedText | 自动复制 |
| `components/ScrollKeybindingHandler.tsx` | moveFocus, shiftSelection | 键盘选择 |
| `components/PromptInput/PromptInputFooterLeftSide.tsx` | hasSelection | 显示选择提示 |

---

## 依赖与外部交互

### 与 screen.ts 的交互

1. **读取单元格**
   - `cellAt()` / `cellAtIndex()` - 获取单元格内容和宽度
   - 检查 `screen.noSelect` 跳过边距区域
   - 检查 `screen.softWrap` 处理换行

2. **应用高亮**
   - `setCellStyleId()` - 修改单元格样式
   - `stylePool.withSelectionBg()` - 获取选择背景样式

### 与 ink.tsx 的交互

1. **事件处理**
   - 鼠标事件通过 App.tsx 转发到 ink.tsx 的选择处理方法
   - 键盘事件在 ScrollKeybindingHandler.tsx 处理

2. **滚动集成**
   - `shiftAnchor()` - 拖拽到边缘滚动时调用
   - `shiftSelection()` - 键盘滚动时调用
   - `shiftSelectionForFollow()` - 自动跟随滚动时调用
   - `captureScrolledRows()` - 滚动前捕获行内容

3. **渲染集成**
   - `applySelectionOverlay()` 在每帧渲染后调用
   - 与搜索高亮、当前匹配高亮共享样式叠加机制

### 与 useCopyOnSelect.ts 的交互

- 监听选择状态变化
- 调用 `getSelectedText()` 获取文本
- 使用 `clipboardy` 写入剪贴板

---

## 风险、边界与改进建议

### 已知风险

1. **状态复杂性**
   - 虚拟行、累积器、anchorSpan 等多重回滚机制复杂
   - 容易在边界情况下出现 highlight ≠ copy 的不一致

2. **性能问题**
   - `getSelectedText()` 每帧可能被调用多次
   - 大选择范围时字符串拼接开销大

3. **URL 检测局限**
   - 仅支持 ASCII URL 字符
   - 不支持 IDN（国际化域名）
   - 括号平衡算法可能误判

4. **多字节字符处理**
   - `wordBoundsAt` 中的字符类别判断基于单字符
   - 复杂 grapheme cluster 可能被错误分割

### 边界情况

1. **空选择**
   - 点击无拖拽时 `focus === null`，`hasSelection()` 返回 false

2. **单单元格选择**
   - anchor === focus 时仍视为有效选择

3. **反向选择**
   - anchor 在 focus 之后，`selectionBounds()` 会自动交换

4. **滚动到边界**
   - 两端同时移出同一边界时清除选择
   - 避免产生单单元格幽灵高亮

5. **软换行与硬换行混合**
   - `joinRows()` 正确处理连续软换行
   - 软换行后的硬换行产生空行

### 改进建议

1. **性能优化**
   - 缓存 `getSelectedText()` 结果，仅在状态变化时重新计算
   - 使用字符串数组+join 替代重复拼接
   - 延迟提取 scrolledOff 行，仅在需要复制时处理

2. **功能增强**
   - 支持矩形/块选择模式（Alt+拖拽）
   - 支持多选区（Ctrl/Cmd+点击）
   - 支持选择历史（可撤销的选择变化）
   - 更智能的 URL 检测（支持更多 scheme、IDN）

3. **代码质量**
   - 将 URL 检测提取到独立模块
   - 添加更多单元测试覆盖复杂滚动场景
   - 使用类型 branded types 区分行列坐标

4. **可访问性**
   - 添加屏幕阅读器通知
   - 支持键盘导航的可见指示器
   - 选择变化时提供音频反馈选项
