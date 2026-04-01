# FuzzyPicker.tsx 研究文档

## 场景与职责

FuzzyPicker 是一个功能强大的模糊搜索选择器组件，用于在终端 UI 中提供类似 fzf 的交互式搜索体验。它支持实时搜索、键盘导航、预览面板和多种自定义操作。

核心职责：
- 提供实时搜索输入框，支持复杂的 Emacs 风格快捷键
- 显示可滚动的过滤后结果列表
- 支持键盘导航（上下箭头、Ctrl+P/N）
- 支持可选的预览面板（底部或右侧）
- 提供 Tab/Shift+Tab 的辅助操作
- 自适应终端尺寸，防止溢出

## 功能点目的

### 1. 搜索输入
- 使用 `useSearchInput` hook 提供丰富的文本输入体验
- 支持 Emacs 风格快捷键（Ctrl+A/E/B/F、Ctrl+K/U/W、Ctrl+Y 等）
- 支持 kill-ring（剪切板历史）和 yank（粘贴）
- 空查询时 Backspace 不会退出（`backspaceExitsOnEmpty: false`）

### 2. 列表导航
- **方向键**: ↑/↓ 或 Ctrl+P/N 导航
- **方向控制**: 支持 `direction='up'`（atuin 风格，items[0] 在底部）
- **自适应可见数量**: 根据终端高度自动调整显示的项目数
- **窗口滑动**: 当项目数超过可见区域时，自动滑动窗口

### 3. 预览面板
- **位置**: 支持 `bottom`（底部）或 `right`（右侧）
- **自适应**: 右侧预览需要足够的终端宽度
- **稳定性**: 布局结构不依赖 preview 的真值，避免切换时的布局跳动

### 4. 操作支持
- **主操作**: Enter 键选择当前项
- **辅助操作**: Tab 键执行自定义操作（如 `onTab` 提供的操作）
- **Shift+Tab**: 独立的 Shift+Tab 操作（可选）
- **取消**: Esc/Ctrl+C 取消选择

### 5. 紧凑模式
- 当终端宽度小于 120 列时启用
- 缩短标签（如 "navigate" → "nav"）
- 隐藏 Shift+Tab 提示

## 具体技术实现

### 关键数据结构

```typescript
type PickerAction<T> = {
  action: string;           // 提示标签，如 "mention"
  handler: (item: T) => void;
};

type Props<T> = {
  title: string;
  placeholder?: string;     // 默认 "Type to search…"
  initialQuery?: string;
  items: readonly T[];
  getKey: (item: T) => string;
  renderItem: (item: T, isFocused: boolean) => React.ReactNode;
  renderPreview?: (item: T) => React.ReactNode;
  previewPosition?: 'bottom' | 'right';  // 默认 'bottom'
  visibleCount?: number;    // 默认 8
  direction?: 'down' | 'up';  // 默认 'down'
  onQueryChange: (query: string) => void;
  onSelect: (item: T) => void;
  onTab?: PickerAction<T>;
  onShiftTab?: PickerAction<T>;
  onFocus?: (item: T | undefined) => void;
  onCancel: () => void;
  emptyMessage?: string | ((query: string) => string);
  matchLabel?: string;      // 状态行，如 "500+ matches"
  selectAction?: string;    // 默认 "select"
  extraHints?: React.ReactNode;
};
```

### 核心实现逻辑

1. **可见数量计算**
   ```tsx
   const DEFAULT_VISIBLE = 8;
   const CHROME_ROWS = 10;  // Pane + title + gaps + SearchBox + hints
   const MIN_VISIBLE = 2;
   
   const visibleCount = Math.max(
     MIN_VISIBLE, 
     Math.min(requestedVisible, rows - CHROME_ROWS - (matchLabel ? 1 : 0))
   );
   ```
   - 确保至少有 MIN_VISIBLE 个项目可见
   - 根据终端高度自动限制，防止溢出
   - 溢出会导致光标位置错乱和闪烁

2. **键盘事件处理**
   ```tsx
   const handleKeyDown = (e: KeyboardEvent) => {
     // 上/下导航（支持 Ctrl+P/N）
     if (e.key === 'up' || e.ctrl && e.key === 'p') {
       step(direction === 'up' ? 1 : -1);
       return;
     }
     
     // Enter 选择
     if (e.key === 'return') {
       const selected = items[focusedIndex];
       if (selected) onSelect(selected);
       return;
     }
     
     // Tab 处理
     if (e.key === 'tab') {
       const tabAction = e.shift ? onShiftTab ?? onTab : onTab;
       if (tabAction) tabAction.handler(selected);
       else onSelect(selected);  // 无 onTab 时 Tab 等同于 Enter
     }
   };
   ```

3. **窗口滑动计算**
   ```tsx
   const windowStart = clamp(
     focusedIndex - visibleCount + 1, 
     0, 
     items.length - visibleCount
   );
   const visible = items.slice(windowStart, windowStart + visibleCount);
   ```
   - 确保 focused item 始终在可见区域内
   - 当列表滚动时，窗口跟随焦点移动

4. **列表渲染（List 子组件）**
   ```tsx
   function List({ visible, windowStart, focusedIndex, direction, ... }) {
     const rows = visible.map((item, i) => {
       const actualIndex = windowStart + i;
       const isFocused = actualIndex === focusedIndex;
       const atLowEdge = i === 0 && windowStart > 0;
       const atHighEdge = i === visible.length - 1 && windowStart + visibleCount < total;
       
       return (
         <ListItem 
           key={getKey(item)}
           isFocused={isFocused}
           showScrollUp={direction === "up" ? atHighEdge : atLowEdge}
           showScrollDown={direction === "up" ? atLowEdge : atHighEdge}
         >
           {renderItem(item, isFocused)}
         </ListItem>
       );
     });
     
     return <Box flexDirection={direction === "up" ? "column-reverse" : "column"}>{rows}</Box>;
   }
   ```
   - 使用 `column-reverse` 实现 atuin 风格的向上布局
   - 计算滚动指示器（showScrollUp/Down）的显示条件

5. **布局结构**
   ```tsx
   // 右侧预览布局
   <Box flexDirection="row" gap={2}>
     <Box flexDirection="column" flexShrink={0}>
       {listBlock}
       {matchLabel && <Text dimColor>{matchLabel}</Text>}
     </Box>
     {preview ?? <Box flexGrow={1} />}
   </Box>
   
   // 底部预览布局
   <Box flexDirection="column">
     {listBlock}
     {matchLabel && <Text dimColor>{matchLabel}</Text>}
     {preview}
   </Box>
   ```
   - 使用 `Box flexGrow={1}` 占位保持布局稳定
   - 避免 preview 切换时出现布局跳动

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/design-system/FuzzyPicker.tsx`

### 依赖文件

| 文件 | 用途 |
|------|------|
| `../../hooks/useSearchInput.js` | 搜索输入处理（含 Emacs 快捷键） |
| `../../hooks/useTerminalSize.js` | 终端尺寸获取 |
| `../../ink/events/keyboard-event.js` | 键盘事件类型 |
| `../../ink/layout/geometry.js` (clamp) | 数值限制工具 |
| `../../ink.js` (Box, Text, useTerminalFocus) | 布局组件和焦点状态 |
| `../SearchBox.js` | 搜索输入框 UI |
| `./Byline.js` | 快捷键提示连接 |
| `./KeyboardShortcutHint.js` | 键盘快捷键提示 |
| `./ListItem.js` | 列表项组件 |
| `./Pane.js` | 外层容器 |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/components/HistorySearchDialog.tsx` | 历史记录搜索 |
| `src/components/QuickOpenDialog.tsx` | 快速打开文件 |
| `src/components/GlobalSearchDialog.tsx` | 全局代码搜索 |

## 依赖与外部交互

### useSearchInput Hook

FuzzyPicker 依赖 `useSearchInput` 提供复杂的输入处理：

```tsx
const { query, cursorOffset } = useSearchInput({
  isActive: true,
  onExit: () => {},  // 由 handleKeyDown 处理
  onCancel,
  initialQuery,
  backspaceExitsOnEmpty: false,  // 防止持续按 Backspace 退出
});
```

该 hook 支持：
- 光标移动（箭头键、Ctrl+B/F、Home/End）
- 文本编辑（Backspace、Delete、Ctrl+D/H）
- Kill 操作（Ctrl+K/U/W，删除行/单词）
- Yank 操作（Ctrl+Y/Meta+Y，粘贴）
- 单词跳转（Ctrl/Meta + 箭头）

### 终端焦点系统

```tsx
const isTerminalFocused = useTerminalFocus();
```

- 用于 SearchBox 的显示状态
- 终端失焦时显示非活跃状态样式

### 列表项交互

通过 `renderItem` 和 `renderPreview` 回调，调用方完全控制列表项和预览的渲染：

```tsx
<FuzzyPicker
  items={files}
  getKey={f => f.path}
  renderItem={(file, isFocused) => (
    <Text color={isFocused ? 'suggestion' : undefined}>
      {file.name}
    </Text>
  )}
  renderPreview={file => (
    <Text dimColor>{file.preview}</Text>
  )}
/>
```

## 风险、边界与改进建议

### 潜在风险

1. **终端高度溢出**
   - 如果计算错误，picker 可能超出终端高度
   - 会导致光标位置错乱和闪烁
   - `CHROME_ROWS` 常量需要与 SearchBox、Pane 的实际高度保持同步

2. **性能问题**
   - `items` 数组很大时，每次查询变化都会重新渲染整个列表
   - 虽然使用 React Compiler 优化，但仍需注意大数据量场景

3. **键盘事件冲突**
   - `useSearchInput` 和 `handleKeyDown` 都处理键盘事件
   - 通过 `onExit: () => {}` 确保 Enter 由 handleKeyDown 处理

4. **方向逻辑复杂性**
   - `direction='up'` 时，视觉方向和数组索引方向相反
   - 滚动指示器的逻辑需要根据方向翻转

### 边界情况

1. **空列表**
   - 显示 `emptyMessage`（支持字符串或函数）
   - ListItem 保持高度占位，避免布局跳动

2. **单项目**
   - 正常渲染，无滚动指示器
   - 导航快捷键仍然有效

3. **终端尺寸过小**
   - 至少保证 MIN_VISIBLE=2 个项目可见
   - 如果终端高度小于 CHROME_ROWS + MIN_VISIBLE，可能显示异常

4. **查询变化时的焦点重置**
   ```tsx
   useEffect(() => {
     onQueryChange(query);
     setFocusedIndex(0);  // 查询变化时重置到第一项
   }, [query]);
   ```

5. **items 长度变化**
   ```tsx
   useEffect(() => {
     setFocusedIndex(i => clamp(i, 0, items.length - 1));
   }, [items.length]);
   ```
   - 确保 focusedIndex 不会越界

### 改进建议

1. **虚拟化长列表**
   - 对于非常大的列表，可以实现虚拟滚动
   - 只渲染可见区域内的项目

2. **搜索防抖**
   - 当前 `onQueryChange` 在每次按键时立即触发
   - 可以添加防抖优化，减少不必要的过滤计算

3. **多选支持**
   ```typescript
   multiSelect?: boolean;
   selectedKeys?: Set<string>;
   onSelectionChange?: (keys: Set<string>) => void;
   ```

4. **分组支持**
   ```typescript
   groups?: { label: string; items: T[] }[];
   ```
   - 支持按组显示项目

5. **快捷键自定义**
   - 允许调用方自定义导航快捷键
   - 支持 Vim 风格（j/k）或 Emacs 风格（Ctrl+N/P）切换

6. **搜索高亮**
   - 在 renderItem 中提供匹配的高亮信息
   - 显示匹配的部分

7. **异步加载**
   ```typescript
   loading?: boolean;
   hasMore?: boolean;
   onLoadMore?: () => void;
   ```
   - 支持无限滚动加载

### 测试注意事项

- 测试 `direction='up'` 时的导航方向
- 测试终端高度变化时的可见数量调整
- 测试空列表和单项目列表的渲染
- 验证滚动指示器的正确显示
- 测试 Tab/Shift+Tab 的各种组合
- 验证预览面板切换时的布局稳定性
- 测试快速输入时的性能和响应
