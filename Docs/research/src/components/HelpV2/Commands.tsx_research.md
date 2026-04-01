# Commands.tsx 研究文档

## 场景与职责

`Commands.tsx` 是 Claude Code 帮助系统（HelpV2）中的**命令列表展示子组件**。它在 `/help` 弹窗的 "commands" 和 "custom-commands" 两个标签页中被复用，负责将原始的 `Command[]` 数据转换为可交互、可滚动、带格式化的终端 UI 列表。核心职责包括：命令去重与排序、描述文本截断、根据终端尺寸计算分页高度，以及与上层 `Tabs` 的焦点系统协同工作。

## 功能点目的

1. **命令列表渲染**：接收 `commands` 数组，输出一个仅用于浏览（不可选择）的垂直列表。
2. **数据清洗**：对同名命令去重（保留第一个），并按命令名字母顺序排序，保证列表稳定可预测。
3. **描述格式化**：调用 `formatDescriptionWithSource` 为每个命令附加来源标注（如 plugin、bundled、workflow 等），再使用 `truncate` 按可用列宽截断，防止终端换行错乱。
4. **尺寸自适应**：根据传入的 `columns` 和 `maxHeight` 动态计算每行最大宽度和可见选项数，确保列表在有限空间内完整呈现。
5. **焦点桥接**：通过 `useTabHeaderFocus` 实现与 `Tabs` 组件的键盘焦点互通——当用户在列表第一项按「上」时，焦点回到标签页头部。

## 具体技术实现

### 关键流程

```
props.commands
  → filter (Set 去重)
  → sort (localeCompare 按 name 排序)
  → map (构造 Select options)
  → render Select
```

### 数据结构

- **Props**：
  - `commands: Command[]` — 原始命令数组
  - `maxHeight: number` — 列表可用最大高度（行数）
  - `columns: number` — 终端列数
  - `title: string` — 列表上方显示的标题
  - `onCancel: () => void` — 取消/关闭回调
  - `emptyMessage?: string` — 列表为空时的提示文案

- **Select Option 结构**：
  ```ts
  {
    label: `/${cmd.name}`,
    value: cmd.name,
    description: truncate(formatDescriptionWithSource(cmd), maxWidth, true)
  }
  ```

### 核心计算逻辑

- `maxWidth = Math.max(1, columns - 10)`：为索引、边距预留约 10 列后，剩余宽度用于描述文本。
- `visibleCount = Math.max(1, Math.floor((maxHeight - 10) / 2))`：基于经验公式估算可见选项数，假设每个选项平均占 2 行高度，并预留 10 行给标题和其他 UI。

### 渲染协议

使用 `Select` 组件（来自 `../CustomSelect/select.js`）并传入以下关键属性：

| 属性 | 值 | 说明 |
|------|-----|------|
| `disableSelection` | `true` | 仅浏览，Enter 不会触发选择 |
| `hideIndexes` | `true` | 隐藏左侧数字序号 |
| `layout` | `"compact-vertical"` | 标签与描述垂直堆叠，紧凑排列 |
| `onUpFromFirstItem` | `focusHeader` | 第一项按上箭头时焦点回到 Tab 头部 |
| `isDisabled` | `headerFocused` | Tab 头部获得焦点时禁用列表键盘事件 |

### React Compiler 缓存

文件已被 React Compiler 编译，使用 `_c(n)` 进行细粒度 memo 缓存。所有派生数据（options、JSX 节点）均通过缓存数组槽位进行依赖比较，避免不必要的重渲染。

## 关键代码路径与文件引用

- **本文件**：`src/components/HelpV2/Commands.tsx`
- **调用方**：`src/components/HelpV2/HelpV2.tsx`（在 "commands" / "custom" Tab 中渲染）
- **依赖文件**：
  - `src/commands.ts` — `Command` 类型定义、`formatDescriptionWithSource`
  - `src/ink.ts` — `Box`、`Text`
  - `src/utils/truncate.ts`（经 `src/utils/format.ts` 重导出）— `truncate`
  - `src/components/CustomSelect/select.tsx` — `Select` 组件
  - `src/components/design-system/Tabs.tsx` — `useTabHeaderFocus`

## 依赖与外部交互

| 依赖 | 交互方式 | 说明 |
|------|----------|------|
| `Commands.tsx` → `Tabs.tsx` | `useTabHeaderFocus()` | 读取/控制 Tab 头部焦点状态 |
| `Commands.tsx` → `select.tsx` | Props 传递 | 将命令数据转为 `OptionWithDescription[]` 后渲染 |
| `Commands.tsx` → `commands.ts` | 导入类型与工具函数 | `Command`、`formatDescriptionWithSource` |
| `Commands.tsx` → `format.ts` | 导入 `truncate` | 对描述文本做终端宽度截断 |

## 风险、边界与改进建议

1. **硬编码尺寸公式**：`maxHeight - 10` 和 `/ 2` 是经验值，在极小终端（如高度 < 12 行）或超大字体场景下，`visibleCount` 可能过小甚至为 1，导致用户体验下降。建议通过实际测量 `SelectOption` 高度来动态计算。

2. **去重静默丢失**：同名命令仅保留第一个，没有任何日志或 UI 提示告知用户有重复命令被隐藏。若插件与内置命令冲突，用户可能困惑为何某些命令未出现。

3. **无空状态差异化处理**：虽然支持 `emptyMessage`，但组件本身不区分 "加载中" 和 "真正为空" 两种状态，调用方需自行保证数据就绪后再渲染。

4. **缺少单元测试**：目录下无测试文件，命令去重、排序、截断、visibleCount 计算等逻辑均缺乏自动化覆盖。

5. **React Compiler 编译产物可读性差**：源码已被编译为低层级缓存代码，直接调试或热修复难度较高，建议保留原始 `.tsx` 源映射以便开发调试。
