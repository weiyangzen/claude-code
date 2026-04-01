# DiffFileList.tsx 研究文档

## 场景与职责

`DiffFileList` 是 Claude Code diff 查看功能中的文件列表展示组件。它是 `DiffDialog` 在 list 视图模式下的核心子组件，负责：

1. **文件列表渲染** - 展示变更文件的列表，包含文件名和变更统计
2. **选中状态可视化** - 通过指针符号（`›`）和颜色高亮当前选中文件
3. **分页/虚拟滚动** - 当文件数超过 5 个时，实现滑动窗口展示
4. **文件变更统计展示** - 显示每个文件的增删行数、二进制标记、截断标记等
5. **终端宽度适配** - 动态计算路径显示宽度，超长路径进行头部截断

该组件作为纯展示组件，不管理状态，通过 props 接收数据和选中索引。

## 功能点目的

### 1. 滑动窗口分页
- **目的**：在有限的终端空间内展示大量文件，保持 UI 简洁
- **实现**：固定展示最多 `MAX_VISIBLE_FILES = 5` 个文件
- **行为**：选中项始终保持在可视区域内，窗口随选中项移动

### 2. 路径截断显示
- **目的**：适应不同终端宽度，确保统计信息始终可见
- **实现**：使用 `truncateStartToWidth` 从路径头部截断，保留文件名
- **计算**：`maxPathWidth = Math.max(20, columns - 16 - 3 - 4)`

### 3. 变更统计展示
每个文件项右侧显示：
- **增删行数**：`+{linesAdded}`（绿色）和 `-{linesRemoved}`（红色）
- **特殊状态标记**：
  - `untracked`：未跟踪文件
  - `Binary file`：二进制文件
  - `Large file modified`：大文件（>1MB）
  - `(truncated)`：行数超过 400 行限制

### 4. 分页指示器
当文件总数超过 5 个时：
- **上方指示器**：`↑ {n} more files`（当上方有隐藏文件时）
- **下方指示器**：`↓ {n} more files`（当下方有隐藏文件时）
- **目的**：告知用户列表可滚动，并提供上下文信息

## 具体技术实现

### Props 接口定义

```typescript
type Props = {
  files: DiffFile[];      // 文件列表
  selectedIndex: number;  // 当前选中项的索引
};

// DiffFile 定义（来自 useDiffData.ts）
type DiffFile = {
  path: string;           // 文件路径
  linesAdded: number;     // 新增行数
  linesRemoved: number;   // 删除行数
  isBinary: boolean;      // 是否为二进制文件
  isLargeFile: boolean;   // 是否为大文件
  isTruncated: boolean;   // 是否被截断
  isNewFile?: boolean;    // 是否为新文件
  isUntracked?: boolean;  // 是否为未跟踪文件
};
```

### 滑动窗口算法

```typescript
const MAX_VISIBLE_FILES = 5;

// 计算可视窗口的起止索引
const { startIndex, endIndex } = useMemo(() => {
  if (files.length <= MAX_VISIBLE_FILES) {
    return { startIndex: 0, endIndex: files.length };
  }
  
  // 尝试将选中项放在窗口中间
  let start = Math.max(0, selectedIndex - Math.floor(MAX_VISIBLE_FILES / 2));
  let end = start + MAX_VISIBLE_FILES;
  
  // 边界调整：确保不越界
  if (end > files.length) {
    end = files.length;
    start = Math.max(0, end - MAX_VISIBLE_FILES);
  }
  
  return { startIndex: start, endIndex: end };
}, [files.length, selectedIndex]);
```

### 文件项渲染（FileItem 子组件）

```typescript
function FileItem({ file, isSelected, maxPathWidth }) {
  // 路径截断
  const displayPath = truncateStartToWidth(file.path, maxPathWidth);
  
  // 选中指示器
  const pointer = isSelected ? figures.pointer + ' ' : '  ';
  const line = `${pointer}${displayPath}`;
  
  return (
    <Box flexDirection="row">
      <Text bold={isSelected} color={isSelected ? 'background' : undefined} inverse={isSelected}>
        {line}
      </Text>
      <Box flexGrow={1} />  {/* 弹性填充，将统计信息推到右侧 */}
      <FileStats file={file} isSelected={isSelected} />
    </Box>
  );
}
```

### 统计信息渲染（FileStats 子组件）

优先级顺序（互斥）：
1. `isUntracked` → 显示 "untracked"
2. `isBinary` → 显示 "Binary file"
3. `isLargeFile` → 显示 "Large file modified"
4. 正常文件 → 显示 `+{added}` `-{removed}`，可选 `(truncated)`

```typescript
function FileStats({ file, isSelected }) {
  if (file.isUntracked) {
    return <Text dimColor={!isSelected} italic>untracked</Text>;
  }
  if (file.isBinary) {
    return <Text dimColor={!isSelected} italic>Binary file</Text>;
  }
  if (file.isLargeFile) {
    return <Text dimColor={!isSelected} italic>Large file modified</Text>;
  }
  
  return (
    <Text>
      {file.linesAdded > 0 && (
        <Text color="diffAddedWord" bold={isSelected}>+{file.linesAdded}</Text>
      )}
      {file.linesAdded > 0 && file.linesRemoved > 0 && ' '}
      {file.linesRemoved > 0 && (
        <Text color="diffRemovedWord" bold={isSelected}>-{file.linesRemoved}</Text>
      )}
      {file.isTruncated && (
        <Text dimColor={!isSelected}> (truncated)</Text>
      )}
    </Text>
  );
}
```

### React Compiler 优化

代码经过 React Compiler 编译，特点：
- 使用 `_c(n)` 创建固定大小的 memoization cache 数组
- 每个 JSX 元素和计算结果都有独立的 cache slot
- 通过比较依赖项（如 `files`, `selectedIndex`, `columns`）决定是否复用

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `src/hooks/useDiffData.ts` | `DiffFile` 类型定义 |
| `src/hooks/useTerminalSize.ts` | 获取终端宽度（columns） |
| `src/utils/format.ts` | `truncateStartToWidth` 路径截断 |
| `src/utils/stringUtils.ts` | `plural` 复数格式化 |
| `src/ink.ts` | Ink 组件（Box, Text） |
| `figures` | 终端图形符号（pointer: `›`） |

### 被调用方

| 文件路径 | 调用场景 |
|---------|---------|
| `src/components/diff/DiffDialog.tsx` | DiffDialog 在 viewMode='list' 时渲染 DiffFileList |

### 类型依赖

| 来源 | 类型 |
|-----|------|
| `src/hooks/useDiffData.ts` | `DiffFile` |
| `figures` 包 | 终端符号 |

## 依赖与外部交互

### 终端尺寸响应
通过 `useTerminalSize` hook 监听终端宽度变化：
- 动态计算 `maxPathWidth`
- 触发重新渲染以适应新宽度

### 路径截断算法
使用 `truncateStartToWidth`（来自 `src/utils/format.ts`）：
- 从路径头部截断，保留文件名部分
- 使用 `…` 前缀表示截断
- 基于字符宽度计算（考虑全角字符）

### 颜色主题
使用 Ink 的 `color` 属性：
- `diffAddedWord`：新增行颜色（通常为绿色）
- `diffRemovedWord`：删除行颜色（通常为红色）
- `background`：选中项背景色
- `dimColor`：非选中/次要信息颜色

## 风险、边界与改进建议

### 已知风险

1. **路径截断计算偏差**：
   - `maxPathWidth = Math.max(20, columns - 16 - 3 - 4)` 的魔法数字假设
   - 16（统计信息最大宽度）、3（padding）、4（预留边距）
   - 风险：不同语言/字体的统计信息实际宽度可能与假设不符

2. **长文件名覆盖统计**：
   - 当路径截断后仍过长，可能挤压统计信息显示空间
   - 当前使用 `flexGrow={1}` 弹性填充，但最小宽度限制可能失效

3. **大量文件性能**：
   - 虽然只渲染 5 个可见项，但 `files.map()` 在每次渲染时仍遍历所有文件
   - 当文件数 > 1000 时，可能成为性能瓶颈

### 边界情况

| 场景 | 处理行为 |
|-----|---------|
| `files.length === 0` | 显示 "No changed files" |
| `files.length <= 5` | 禁用分页，显示全部文件 |
| `selectedIndex` 越界 | 由父组件 DiffDialog 保证有效性 |
| 终端宽度 < 30 | `maxPathWidth` 被限制为 20，路径大幅截断 |
| 文件名为空字符串 | 正常渲染（显示空路径） |
| `linesAdded` 和 `linesRemoved` 同时为 0 | 不显示统计数字（仅显示截断标记如适用） |

### 改进建议

1. **布局优化**：
   - 使用更精确的宽度计算，考虑实际字符显示宽度（East Asian Width）
   - 实现自适应列宽：路径和统计信息根据内容动态分配空间
   - 添加文件图标/类型指示器（如 📄、📁、⚙️）

2. **交互增强**：
   - 支持按首字母快速跳转（如按 's' 跳到以 'src' 开头的文件）
   - 添加文件排序选项（按路径、按变更大小、按类型）
   - 支持多选（用于批量 stage/unstage）

3. **性能优化**：
   - 使用 `useMemo` 缓存 `visibleFiles` 的渲染结果
   - 对于超大量文件（>1000），考虑使用虚拟滚动而非简单切片
   - 延迟加载文件统计信息（当前是一次性计算）

4. **可访问性**：
   - 增加选中文件的 ARIA 标签
   - 支持屏幕阅读器朗读文件变更统计
   - 提供高对比度模式下的颜色替代方案

5. **代码结构**：
   - 将 `MAX_VISIBLE_FILES` 提取为可配置参数
   - `FileItem` 和 `FileStats` 可提取为独立文件，便于单元测试
   - 魔法数字（16, 3, 4）提取为命名常量并添加注释

6. **功能扩展**：
   - 显示文件类型图标或扩展名标记
   - 添加文件状态指示（modified, added, deleted, renamed）
   - 支持展开/折叠目录（树形视图）
   - 集成文件预览（hover 时显示 diff 片段）
