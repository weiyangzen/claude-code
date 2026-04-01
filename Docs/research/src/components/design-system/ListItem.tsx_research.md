# ListItem.tsx 研究文档

## 场景与职责

ListItem 是一个用于选择类 UI（下拉菜单、多选框、菜单）的列表项组件。它处理列表项的常见交互模式，包括焦点指示、选择状态、滚动提示等。

核心职责：
- 显示焦点指示器（❯ 指针）
- 显示选择状态（✓ 勾选标记）
- 显示滚动提示（↓↑ 箭头）
- 根据状态自动应用颜色样式
- 支持可选的描述文本
- 支持光标位置声明（用于终端光标定位）

## 功能点目的

### 1. 状态指示器
- **焦点指示器**: 当 `isFocused=true` 时显示 `figures.pointer`（❯）
- **选择指示器**: 当 `isSelected=true` 时显示 `figures.tick`（✓）
- **滚动提示**: 当 `showScrollUp/Down=true` 时显示上下箭头

### 2. 自动样式
- 根据 `isFocused`、`isSelected`、`disabled` 状态自动应用颜色
- 支持禁用 `styled` 模式，允许自定义子元素样式
- 颜色映射：
  - `disabled` → `inactive`
  - `isSelected` → `success`
  - `isFocused` → `suggestion`

### 3. 光标声明
- 支持通过 `useDeclaredCursor` 声明终端光标位置
- 用于 IME 输入和屏幕阅读器支持
- 可通过 `declareCursor=false` 禁用（当子元素自己声明光标时）

### 4. 描述文本
- 支持在主内容下方显示描述文本
- 自动添加左侧缩进（paddingLeft=2）
- 使用 `inactive` 颜色显示

## 具体技术实现

### 关键数据结构

```typescript
type ListItemProps = {
  /** Whether this item is currently focused (keyboard selection) */
  isFocused: boolean;
  /** Whether this item is selected (chosen/checked). @default false */
  isSelected?: boolean;
  /** The content to display for this item */
  children: ReactNode;
  /** Optional description text displayed below the main content */
  description?: string;
  /** Show a down arrow indicator instead of pointer. @default false */
  showScrollDown?: boolean;
  /** Show an up arrow indicator instead of pointer. @default false */
  showScrollUp?: boolean;
  /** Whether to apply automatic styling. @default true */
  styled?: boolean;
  /** Whether this item is disabled. @default false */
  disabled?: boolean;
  /** Whether to declare the terminal cursor position. @default true */
  declareCursor?: boolean;
};
```

### 核心渲染逻辑

1. **指示器渲染**
   ```tsx
   function renderIndicator() {
     if (disabled) return <Text> </Text>;           // 空格占位
     if (isFocused) return <Text color="suggestion">{figures.pointer}</Text>;
     if (showScrollDown) return <Text dimColor>{figures.arrowDown}</Text>;
     if (showScrollUp) return <Text dimColor>{figures.arrowUp}</Text>;
     return <Text> </Text>;                          // 默认空格
   }
   ```
   - 优先级: disabled > focused > scrollDown > scrollUp > 默认
   - 始终返回相同宽度的内容，保持布局对齐

2. **文本颜色计算**
   ```tsx
   function getTextColor() {
     if (disabled) return "inactive";
     if (!styled) return undefined;  // 不应用自动样式
     if (isSelected) return "success";
     if (isFocused) return "suggestion";
   }
   ```

3. **光标声明**
   ```tsx
   const cursorRef = useDeclaredCursor({
     line: 0,
     column: 0,
     active: isFocused && !disabled && declareCursor !== false
   });
   ```
   - 仅在聚焦、非禁用、允许声明时激活
   - 声明位置为 (0, 0)，即相对于 Box 的左上角

4. **布局结构**
   ```tsx
   <Box ref={cursorRef} flexDirection="column">
     <Box flexDirection="row" gap={1}>
       {indicator}           {/* ❯ 或 ✓ 或 ↓↑ 或空格 */}
       {content}             {/* 主内容（可能包裹在 Text 中） */}
       {checkmark}           {/* 选择标记（右侧） */}
     </Box>
     {description && (
       <Box paddingLeft={2}>
         <Text color="inactive">{description}</Text>
       </Box>
     )}
   </Box>
   ```

### figures 图标

使用 `figures` npm 包提供跨平台一致的 Unicode 图标：
- `figures.pointer` → ❯（焦点指示器）
- `figures.tick` → ✓（选择标记）
- `figures.arrowDown` → ↓（向下滚动提示）
- `figures.arrowUp` → ↑（向上滚动提示）

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/design-system/ListItem.tsx`

### 依赖文件

| 文件 | 用途 |
|------|------|
| `figures` (npm) | 跨平台 Unicode 图标 |
| `../../ink/hooks/use-declared-cursor.js` | 光标位置声明 |
| `../../ink.js` (Box, Text) | 布局组件 |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/components/design-system/FuzzyPicker.tsx` | 模糊选择器的列表项 |
| `src/components/CustomSelect/select-option.tsx` | 自定义选择器的选项 |
| `src/commands/bridge/bridge.tsx` | Bridge 命令的列表 |
| `src/components/memory/MemoryFileSelector.tsx` | 内存文件选择器 |
| `src/components/ui/OrderedList.tsx` | 有序列表 |

### 相关组件

| 组件 | 关系 |
|------|------|
| FuzzyPicker | 主要调用方，使用 ListItem 渲染搜索结果 |
| CustomSelect | 使用 ListItem 作为选择选项的基础 |

## 依赖与外部交互

### useDeclaredCursor Hook

```tsx
import { useDeclaredCursor } from '../../ink/hooks/use-declared-cursor.js';
```

该 hook 用于声明终端光标应该停放的位置：
- 终端模拟器在物理光标位置渲染 IME 预编辑文本
- 屏幕阅读器和放大镜跟踪原生光标
- 将光标停放在文本输入的插入点位置，使 CJK 输入内联显示

参数：
- `line`: 相对于 Box 的行号（0 开始）
- `column`: 相对于 Box 的列号（0 开始）
- `active`: 是否激活此声明

返回：
- ref 回调函数，需要附加到 Box 组件

### figures 包

```tsx
import figures from 'figures';
```

提供跨平台一致的 Unicode 符号：
- 在 Windows 上自动回退到 ASCII 字符
- 在支持 Unicode 的终端上显示美观的符号

### 主题颜色

ListItem 使用以下主题颜色：
- `suggestion`: 焦点状态（蓝色系）
- `success`: 选择状态（绿色系）
- `inactive`: 禁用状态和描述文本（灰色系）

## 风险、边界与改进建议

### 潜在风险

1. **光标声明冲突**
   - 如果多个 ListItem 同时声明光标，可能产生冲突
   - 通过 `active` 条件和节点身份检查缓解
   - 但快速切换焦点时仍可能有短暂竞争

2. **图标宽度**
   - 假设所有图标都是单宽度字符
   - 在某些终端或字体中，Unicode 图标可能是双宽度
   - 可能导致布局错位

3. **React Compiler 缓存**
   - 代码经过 React Compiler 编译，包含大量缓存逻辑
   - `renderIndicator` 和 `getTextColor` 被缓存为函数
   - 需要确保依赖数组完整

### 边界情况

1. **所有状态同时开启**
   - `isFocused=true`, `isSelected=true`, `disabled=true`
   - 优先级: disabled > selected > focused
   - 指示器显示为空格（因为 disabled）

2. **滚动指示器与焦点冲突**
   - `isFocused=true` 和 `showScrollDown=true`
   - 焦点指示器优先（❯ 而不是 ↓）

3. **空 children**
   - 组件不验证 children 是否存在
   - 空 children 会渲染为空的 Text 或 Box

4. **长描述文本**
   - 描述文本不换行，可能超出容器宽度
   - 需要父组件控制宽度

5. **styled=false 时的颜色**
   - 不应用任何自动颜色
   - 子元素需要自行处理样式

### 改进建议

1. **支持多行内容**
   ```typescript
   multiline?: boolean;
   maxLines?: number;
   ```
   - 允许内容折行显示

2. **支持右侧操作**
   ```typescript
   action?: ReactNode;
   // 渲染在列表项最右侧
   ```

3. **支持图标前缀**
   ```typescript
   icon?: string | ReactNode;
   // 在主内容前显示图标
   ```

4. **支持嵌套缩进**
   ```typescript
   depth?: number;
   // 根据深度自动计算缩进
   ```

5. **支持悬停状态**
   ```typescript
   isHovered?: boolean;
   // 鼠标悬停时的样式
   ```

6. **描述文本换行**
   ```typescript
   descriptionWrap?: 'nowrap' | 'wrap' | 'truncate';
   ```

7. **性能优化**
   - 对于长列表，考虑使用 React.memo
   - 避免不必要的重渲染

8. **类型安全的颜色**
   ```typescript
   customColor?: keyof Theme;
   // 覆盖自动颜色计算
   ```

### 测试注意事项

- 测试所有状态组合（focused、selected、disabled 的各种组合）
- 验证滚动指示器与焦点指示器的优先级
- 测试光标声明的激活/禁用逻辑
- 验证描述文本的缩进和颜色
- 测试 styled=false 时的行为
- 验证 figures 图标在不同平台的显示
- 测试快速切换焦点时的光标声明

### 与 FuzzyPicker 的协作

FuzzyPicker 是 ListItem 的主要调用方：

```tsx
// FuzzyPicker.tsx
<ListItem
  key={getKey(item)}
  isFocused={isFocused}
  showScrollUp={direction === "up" ? atHighEdge : atLowEdge}
  showScrollDown={direction === "up" ? atLowEdge : atHighEdge}
  styled={false}  // FuzzyPicker 自己处理样式
>
  {renderItem(item, isFocused)}
</ListItem>
```

FuzzyPicker 设置 `styled=false`，因为：
1. `renderItem` 回调由调用方提供
2. 调用方可能有自己的样式逻辑
3. ListItem 只负责指示器和布局

这种分离允许灵活定制，同时保持一致的指示器行为。
