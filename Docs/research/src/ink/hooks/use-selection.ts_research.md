# use-selection.ts 深入研究

## 场景与职责

`useSelection` 是 Ink 终端 UI 框架中提供文本选择功能的 Hook。它允许在全屏模式下进行文本选择、复制和操作，模拟终端原生的文本选择行为。

## 功能点目的

### 1. 文本选择操作
- 复制选中的文本到剪贴板
- 清除选择高亮
- 检查是否存在活动选择

### 2. 选择状态管理
- 获取原始可变选择状态（用于拖滚动）
- 订阅选择状态变化
- 支持选择锚点和焦点的移动

### 3. 滚动集成
- 选择随内容滚动而移动
- 捕获即将滚出视口的行
- 支持键盘滚动时的选择调整

### 4. 主题集成
- 设置选择高亮背景色
- 使用纯色背景替代 SGR-7 反转，保持语法高亮可读

## 具体技术实现

### useSelection 接口

```typescript
export function useSelection(): {
  copySelection: () => string
  copySelectionNoClear: () => string
  clearSelection: () => void
  hasSelection: () => boolean
  getState: () => SelectionState | null
  subscribe: (cb: () => void) => () => void
  shiftAnchor: (dRow: number, minRow: number, maxRow: number) => void
  shiftSelection: (dRow: number, minRow: number, maxRow: number) => void
  moveFocus: (move: FocusMove) => void
  captureScrolledRows: (firstRow: number, lastRow: number, side: 'above' | 'below') => void
  setSelectionBgColor: (color: string) => void
}
```

### useHasSelection 接口

```typescript
export function useHasSelection(): boolean
```

- 响应式选择存在状态
- 在选择创建或清除时重新渲染调用者
- 全屏模式外始终返回 false

### SelectionState 类型

```typescript
// src/ink/selection.ts
export type SelectionState = {
  anchor: Point | null           // 鼠标按下位置
  focus: Point | null            // 当前拖动位置
  isDragging: boolean            // 是否正在拖动
  anchorSpan: {                   // 单词/行模式
    lo: Point
    hi: Point
    kind: 'word' | 'line'
  } | null
  scrolledOffAbove: string[]     // 滚出视口上方的行
  scrolledOffBelow: string[]     // 滚出视口下方的行
  scrolledOffAboveSW: boolean[]  // 软换行标记（上方）
  scrolledOffBelowSW: boolean[]  // 软换行标记（下方）
  virtualAnchorRow?: number      // 预钳位锚点行
  virtualFocusRow?: number       // 预钳位焦点行
  lastPressHadAlt: boolean       // 上次按下是否有 Alt 修饰
}
```

### 核心实现逻辑

1. **获取 Ink 实例**：
   ```typescript
   useContext(StdinContext)
   const ink = instances.get(process.stdout)
   ```

2. **方法委托**：
   ```typescript
   return useMemo(() => {
     if (!ink) {
       return { /* no-op functions */ }
     }
     return {
       copySelection: () => ink.copySelection(),
       copySelectionNoClear: () => ink.copySelectionNoClear(),
       clearSelection: () => ink.clearTextSelection(),
       hasSelection: () => ink.hasTextSelection(),
       getState: () => ink.selection,
       subscribe: (cb) => ink.subscribeToSelectionChange(cb),
       shiftAnchor: (dRow, minRow, maxRow) => shiftAnchor(ink.selection, dRow, minRow, maxRow),
       shiftSelection: (dRow, minRow, maxRow) => ink.shiftSelectionForScroll(dRow, minRow, maxRow),
       moveFocus: (move) => ink.moveSelectionFocus(move),
       captureScrolledRows: (firstRow, lastRow, side) => 
         ink.captureScrolledRows(firstRow, lastRow, side),
       setSelectionBgColor: (color) => ink.setSelectionBgColor(color),
     }
   }, [ink])
   ```

3. **useHasSelection 实现**：
   ```typescript
   export function useHasSelection(): boolean {
     useContext(StdinContext)
     const ink = instances.get(process.stdout)
     return useSyncExternalStore(
       ink ? ink.subscribeToSelectionChange : NO_SUBSCRIBE,
       ink ? ink.hasTextSelection : ALWAYS_FALSE,
     )
   }
   ```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/StdinContext.ts` | 用于锚定到 App 子树 |
| `src/ink/instances.ts` | Ink 实例映射表 |
| `src/ink/selection.ts` | 选择逻辑实现（917 行） |

### selection.ts 核心函数

| 函数 | 作用 |
|------|------|
| `createSelectionState()` | 创建初始选择状态 |
| `startSelection()` | 开始选择（鼠标按下） |
| `updateSelection()` | 更新选择（拖动中） |
| `finishSelection()` | 完成选择（鼠标释放） |
| `clearSelection()` | 清除选择 |
| `selectWordAt()` | 双击选择单词 |
| `selectLineAt()` | 三击选择整行 |
| `extendSelection()` | 扩展单词/行模式选择 |
| `moveFocus()` | 键盘移动焦点 |
| `shiftAnchor()` | 拖动滚动时移动锚点 |
| `shiftSelection()` | 键盘滚动时移动整个选择 |
| `shiftSelectionForFollow()` | 自动跟随滚动时移动选择 |
| `hasSelection()` | 检查是否有选择 |
| `selectionBounds()` | 获取规范化的选择边界 |
| `isCellSelected()` | 检查单元格是否在选择内 |
| `getSelectedText()` | 提取选中的文本 |
| `captureScrolledRows()` | 捕获即将滚出的行 |
| `applySelectionOverlay()` | 应用选择覆盖层到屏幕 |
| `findPlainTextUrlAt()` | 查找纯文本 URL |

### 单词边界检测

```typescript
// Unicode 感知的单词字符匹配器
const WORD_CHAR = /[\p{L}\p{N}_/.\-+~\\]/u

function charClass(c: string): 0 | 1 | 2 {
  if (c === ' ' || c === '') return 0  // 空白
  if (WORD_CHAR.test(c)) return 1      // 单词字符
  return 2                              // 其他
}
```

匹配 iTerm2 默认设置，使双击选择路径（如 `/usr/bin/bash`）时选中整个路径。

### 软换行处理

```typescript
function joinRows(lines: string[], text: string, sw: boolean | undefined): void {
  if (sw && lines.length > 0) {
    lines[lines.length - 1] += text  // 软换行：追加到前一行
  } else {
    lines.push(text)                  // 硬换行：新行
  }
}
```

### 虚拟行跟踪

```typescript
// shiftSelection 中的虚拟行跟踪
const vAnchor = (s.virtualAnchorRow ?? s.anchor.row) + dRow
const vFocus = (s.virtualFocusRow ?? s.focus.row) + dRow
```

用于处理键盘滚动时的选择调整，确保反向滚动时能正确恢复位置。

## 依赖与外部交互

### 与 Ink 实例的交互

1. **选择状态**：
   - `ink.selection` 是 Ink 实例的只读属性
   - 包含完整的 SelectionState

2. **剪贴板操作**：
   - `copySelection()` 复制并清除选择
   - `copySelectionNoClear()` 复制不清除（用于选择即复制）

3. **事件订阅**：
   - `subscribeToSelectionChange()` 订阅选择变化
   - 用于 `useHasSelection` 的响应式更新

### 与渲染流程的交互

```typescript
// 在 onRender 中
applySelectionOverlay(this.backFrame.screen, this.selection, this.stylePool);
```

选择覆盖层直接修改屏幕缓冲区的单元格样式，然后由正常的 diff 流程处理。

### 与滚动功能的交互

1. **拖动滚动**：
   - 拖动到视口边缘时自动滚动
   - `captureScrolledRows()` 捕获即将滚出的行
   - `shiftAnchor()` 调整锚点位置

2. **键盘滚动**：
   - PgUp/PgDn 时 `shiftSelection()` 移动整个选择
   - 选择随内容一起移动

3. **自动跟随**：
   - 内容追加时 `shiftSelectionForFollow()` 保持选择相对位置
   - 如果选择完全滚出视口则清除

## 风险、边界与改进建议

### 潜在风险

1. **内存使用**：
   - `scrolledOffAbove/Below` 数组可能累积大量文本
   - 长时间拖动滚动可能消耗大量内存

2. **性能问题**：
   - `applySelectionOverlay` 遍历选择范围内的每个单元格
   - 大范围选择可能影响渲染性能

3. **并发修改**：
   - SelectionState 是 mutable 的
   - 需要小心处理并发修改

### 边界情况

1. **无选择状态**：
   - 所有方法都有空检查
   - 返回安全默认值

2. **宽字符**：
   - SpacerTail/SpacerHead 单元格被跳过
   - 宽字符的头部包含完整字形

3. **noSelect 单元格**：
   - 边距、行号等标记为 noSelect 的单元格被跳过
   - 不会被复制，也不会高亮

4. **软换行**：
   - 自动换行的行在复制时合并
   - 保持逻辑行而非视觉行

5. **URL 检测**：
   - 纯文本 URL 检测使用 ASCII 字符集
   - 支持括号平衡检查

### 改进建议

1. **添加选择历史**：
   ```typescript
   useSelection({ history: true })
   // 支持撤销/重做选择
   ```

2. **支持块选择模式**：
   ```typescript
   moveFocus(move, { blockMode: true })
   ```

3. **改进 URL 检测**：
   - 支持更多 URL 方案
   - 支持国际化域名

4. **添加选择动画**：
   - 选择时的视觉反馈
   - 复制成功提示

5. **性能优化**：
   - 使用空间索引加速 `isCellSelected`
   - 延迟加载滚出视口的行

6. **添加选择统计**：
   ```typescript
   const { stats } = useSelection()
   // stats.charCount, stats.lineCount, stats.wordCount
   ```

### 测试建议

1. 测试双击/三击选择
2. 测试拖动滚动时的行捕获
3. 测试键盘滚动时的选择调整
4. 测试宽字符和 emoji 选择
5. 测试软换行行的复制
6. 测试 noSelect 单元格跳过
7. 测试纯文本 URL 检测
8. 测试虚拟行跟踪（PgUp/PgDn 往返）
