# NoSelect.tsx 研究文档

## 场景与职责

`NoSelect` 是 Ink 框架中的文本选择控制组件，用于在终端全屏模式下标记内容不可选择：

1. **选择排除**: 标记区域内的单元格不参与文本选择高亮和复制
2. **边距保护**: 支持 `fromLeftEdge` 模式，将排除区域从第 0 列延伸到 Box 的右边缘
3. ** gutters 支持**: 专门用于行号、diff 标记符、列表符号等边距内容的保护

NoSelect 仅在 `<AlternateScreen>` 的鼠标跟踪模式下生效，在主屏幕滚动回滚渲染中无操作。

## 功能点目的

### 1. 文本选择控制
- **选择高亮跳过**: NoSelect 区域内的单元格不会被选择高亮
- **复制内容排除**: 复制时跳过 NoSelect 区域的内容
- **视觉清晰**: 边距在选择过程中保持视觉不变，明确显示哪些内容会被复制

### 2. fromLeftEdge 模式
- **扩展排除区域**: 从第 0 列到 Box 右边缘的所有行都排除
- **缩进保护**: 用于嵌套在缩进容器中的内容（如 diff 中的工具消息行）
- **多行拖拽**: 防止多行拖拽时拾取中间行的容器前导缩进

### 3. 典型使用场景
```tsx
<Box flexDirection="row">
  <NoSelect fromLeftEdge><Text dimColor> 42 +</Text></NoSelect>
  <Text>const x = 1</Text>
</Box>
```

## 具体技术实现

### 关键数据结构

```typescript
type Props = Omit<BoxProps, 'noSelect'> & {
  /**
   * Extend the exclusion zone from column 0 to this box's right edge,
   * for every row this box occupies.
   * @default false
   */
  fromLeftEdge?: boolean;
};
```

### 关键流程

1. **属性解构**
   ```typescript
   const { children, fromLeftEdge, ...boxProps } = props;
   ```
   - 提取 children 和 fromLeftEdge
   - 其余属性传递给 Box

2. **noSelect 值确定**
   ```typescript
   const noSelectValue = fromLeftEdge ? "from-left-edge" : true;
   ```
   - fromLeftEdge 为 true 时: `"from-left-edge"`
   - 否则: `true`

3. **渲染 Box**
   ```typescript
   return <Box {...boxProps} noSelect={noSelectValue}>{children}</Box>;
   ```

### 代码路径

```
NoSelect.tsx
├── 导入依赖
│   ├── React
│   └── Box, BoxProps
├── Props 类型定义
│   ├── 继承 BoxProps（排除 noSelect）
│   └── fromLeftEdge?: boolean
├── NoSelect 函数组件
│   ├── 解构 props
│   ├── 确定 noSelect 值
│   └── 渲染 Box 并传递 noSelect
└── 导出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/ink/components/Box.tsx` | 底层容器，接收 noSelect 属性 |
| `src/ink/styles.ts` | Styles 类型定义 noSelect 属性 |

### noSelect 样式定义

```typescript
// src/ink/styles.ts
export type Styles = {
  // ... 其他属性
  
  /**
   * Exclude this box's cells from text selection in fullscreen mode.
   * Cells inside this region are skipped by both the selection highlight
   * and the copied text — useful for fencing off gutters (line numbers,
   * diff sigils) so click-drag over a diff yields clean copyable code.
   * Only affects alt-screen text selection; no-op otherwise.
   *
   * 'from-left-edge' extends the exclusion from column 0 to the box's
   * right edge for every row it occupies — this covers any upstream
   * indentation (tool message prefix, tree lines) so a multi-row drag
   * doesn't pick up leading whitespace from middle rows.
   */
  readonly noSelect?: boolean | 'from-left-edge'
}
```

### 渲染管线

```
NoSelect 组件
└── <Box noSelect={true | "from-left-edge"}>
    └── children

reconciler.ts
├── 设置 noSelect 属性到 DOM 节点
└── 应用到 Yoga 节点样式

render-node-to-output.ts
├── 渲染 ink-box 节点
├── 检查 node.style.noSelect
├── noSelect 为 true:
│   └── output.noSelect({ x, y, width, height })
└── noSelect 为 "from-left-edge":
    └── output.noSelect({ x: 0, y, width: boxX + width, height })

output.ts
├── 维护 noSelect 位图
└── 渲染时应用排除区域
```

### 渲染实现

```typescript
// src/ink/render-node-to-output.ts
if (node.style.noSelect) {
  const boxX = Math.floor(x);
  const fromEdge = node.style.noSelect === 'from-left-edge';
  output.noSelect({
    x: fromEdge ? 0 : boxX,
    y: Math.floor(y),
    width: fromEdge ? boxX + Math.floor(width) : Math.floor(width),
    height: Math.floor(height),
  });
}
```

## 依赖与外部交互

### 运行时依赖

1. **Box 组件**: 作为底层容器
2. **React Compiler**: 使用 `_c` 函数进行自动记忆化
3. **输出系统**: output.noSelect 方法标记排除区域

### Props 交互

| Prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| fromLeftEdge | boolean | false | 是否从第 0 列开始排除 |
| ...boxProps | BoxProps | - | 所有 Box 的属性都支持 |

### 选择系统交互

```
用户拖拽选择
├── 起始位置确定
├── 拖拽过程中
│   ├── 检查每个单元格的 noSelect 位图
│   ├── noSelect 区域: 跳过高亮
│   └── 普通区域: 正常高亮
└── 复制时
    ├── 遍历选择区域
    ├── 跳过 noSelect 单元格
    └── 生成复制文本
```

## 风险、边界与改进建议

### 已知风险

1. **仅在 AlternateScreen 有效**: 在主屏幕模式下无效果，开发者可能误解
2. **嵌套行为**: 多层 NoSelect 嵌套时的行为可能不符合预期
3. **与绝对定位**: 绝对定位元素与 NoSelect 的交互需要仔细测试

### 边界情况

1. **Box 尺寸为 0**: 如果 Box 没有尺寸，noSelect 区域也为 0
2. **溢出内容**: 子元素溢出 Box 时，溢出部分不受 noSelect 保护
3. **负坐标**: 绝对定位导致负坐标时的处理

### 改进建议

1. **开发警告**: 检测是否在 AlternateScreen 外使用
   ```typescript
   if (process.env.NODE_ENV === 'development') {
     // 检查是否在 AlternateScreen 上下文中
   }
   ```

2. **嵌套优化**: 优化嵌套 NoSelect 的处理逻辑

3. **调试模式**: 添加视觉指示器显示 noSelect 区域（开发环境）
   ```typescript
   if (process.env.CLAUDE_CODE_DEBUG_NOSELECT) {
     // 渲染边框或背景色标识 noSelect 区域
   }
   ```

4. **部分选择**: 考虑支持更细粒度的选择控制（如每字符级别）

5. **文档增强**: 增加更多使用示例和最佳实践

### 代码质量

- 组件简单，职责明确
- 使用 React Compiler 自动记忆化
- 类型定义清晰

### 测试建议

- 测试 fromLeftEdge true/false 的渲染差异
- 测试选择高亮是否正确跳过 noSelect 区域
- 测试复制内容是否正确排除 noSelect 区域
- 测试嵌套 NoSelect 的行为
- 测试与滚动、绝对定位的交互

### 使用模式

```tsx
// 基本使用 - 行号边距
<Box flexDirection="row">
  <NoSelect><Text dimColor>  1 </Text></NoSelect>
  <Text>import React from 'react';</Text>
</Box>

// fromLeftEdge - diff 展示
<Box flexDirection="row">
  <NoSelect fromLeftEdge>
    <Text color="green">+ </Text>
  </NoSelect>
  <Text>const x = 1;</Text>
</Box>

// 组合使用
<Box flexDirection="column">
  {lines.map((line, i) => (
    <Box key={i} flexDirection="row">
      <NoSelect fromLeftEdge>
        <Text dimColor>{String(i + 1).padStart(4)} </Text>
        <Text color={line.type === 'add' ? 'green' : 'red'}>
          {line.type === 'add' ? '+' : '-'}
        </Text>
      </NoSelect>
      <Text>{line.content}</Text>
    </Box>
  ))}
</Box>
```
