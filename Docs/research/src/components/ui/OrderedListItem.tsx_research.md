# OrderedListItem.tsx 深度研究文档

## 场景与职责

OrderedListItem 是 OrderedList 的子组件，用于渲染单个有序列表项。它负责显示编号标记和列表内容，与 OrderedList 配合形成完整的有序列表组件体系。

**核心使用场景：**
- 作为 OrderedList 的子元素使用（`OrderedList.Item` 语法）
- 在终端 UI 中展示带编号标记的列表项
- 支持多行内容的垂直布局

**设计目标：**
- 从 Context 获取编号标记，保持与父级列表一致
- 支持复杂的嵌套内容（多行文本、其他组件等）
- 使用 dimColor 样式使编号标记视觉层级降低

---

## 功能点目的

### 1. 编号标记显示
- 通过 `OrderedListItemContext` 获取当前项的编号标记
- 使用 `Text` 组件的 `dimColor` 属性使标记呈现暗淡效果
- 标记与内容之间保持适当间距

### 2. 内容布局
- 使用 `Box` 组件实现水平布局（gap={1}）
- 内容区域使用垂直布局（flexDirection="column"）
- 支持任意 React 节点作为子内容

### 3. React Compiler 优化
- 代码经过 React Compiler 编译
- 使用缓存机制避免不必要的重渲染

---

## 具体技术实现

### 关键数据结构

```typescript
// Context 导出
export const OrderedListItemContext = createContext({
  marker: ''  // 当前项的编号标记（如 "1.", "1.1."）
});

// Props 定义
type OrderedListItemProps = {
  children: ReactNode;
};
```

### 核心渲染流程

**1. Context 消费（第 15-17 行）**
```javascript
const { marker } = useContext(OrderedListItemContext);
```
- 从父级 OrderedList 注入的 Context 获取编号

**2. 标记渲染（第 19-25 行）**
```javascript
if ($[0] !== marker) {
  t1 = <Text dimColor={true}>{marker}</Text>;
  $[0] = marker;
  $[1] = t1;
} else {
  t1 = $[1];
}
```
- 使用 React Compiler 缓存标记渲染结果
- `dimColor={true}` 使编号呈现暗淡效果，降低视觉干扰

**3. 内容渲染（第 26-33 行）**
```javascript
if ($[2] !== children) {
  t2 = <Box flexDirection="column">{children}</Box>;
  $[2] = children;
  $[3] = t2;
} else {
  t2 = $[3];
}
```
- 将子内容包裹在垂直布局的 Box 中
- 支持多行文本、其他组件等复杂内容

**4. 整体布局（第 34-43 行）**
```javascript
if ($[4] !== t1 || $[5] !== t2) {
  t3 = <Box gap={1}>{t1}{t2}</Box>;
  $[4] = t1;
  $[5] = t2;
  $[6] = t3;
} else {
  t3 = $[6];
}
```
- 水平布局，gap={1} 在标记和内容之间添加一个空格间距
- 整体结构：`[标记] [内容]`

---

## 关键代码路径与文件引用

### 内部依赖
| 路径 | 用途 |
|------|------|
| `react/compiler-runtime` | React Compiler 缓存机制 |
| `react` | 核心 React API（createContext, useContext, type ReactNode） |
| `../../ink.js` | Ink 框架（Box, Text 组件） |

### 外部调用方
| 路径 | 使用场景 |
|------|----------|
| `src/components/ui/OrderedList.tsx` | 导入并作为 Item 属性挂载（第 4、69 行） |
| `src/components/Onboarding.tsx` | 通过 OrderedList.Item 使用（第 73、82 行） |

### 调用示例
```tsx
// 方式 1：通过 OrderedList.Item 使用
<OrderedList>
  <OrderedList.Item>
    <Text>列表项内容</Text>
  </OrderedList.Item>
</OrderedList>

// 方式 2：直接导入使用（不推荐，缺少 Context）
import { OrderedListItem } from './ui/OrderedListItem.js';
<OrderedListItem>内容</OrderedListItem>
```

---

## 依赖与外部交互

### Context 协作机制

**OrderedListItemContext**（本文件定义）
```typescript
export const OrderedListItemContext = createContext({ marker: '' });
```
- 由 OrderedList 提供值
- 包含当前项的完整编号标记

**OrderedListContext**（在 OrderedList.tsx 中定义）
- 用于嵌套列表时传递父级标记前缀
- OrderedListItem 不直接使用，由 OrderedList 内部使用

### 与 OrderedList 的协作流程

```
OrderedList (父组件)
  ├── 统计子元素数量
  ├── 计算编号宽度
  ├── 为每个子元素生成标记（如 "1.", "2."）
  ├── 通过 OrderedListItemContext.Provider 注入标记
  │
  └── OrderedListItem (子组件)
        ├── 通过 useContext 获取标记
        ├── 渲染暗淡的标记文本
        └── 渲染内容区域
```

### React Compiler 缓存策略

| 缓存槽位 | 存储内容 | 依赖项 |
|----------|----------|--------|
| $[0], $[1] | 标记渲染结果 | marker |
| $[2], $[3] | 内容渲染结果 | children |
| $[4], $[5], $[6] | 整体布局 | 标记 + 内容 |

---

## 风险、边界与改进建议

### 已知限制

1. **独立使用无效**
   - 必须在 OrderedList 内部使用才能获取正确的标记
   - 独立使用时 marker 为空字符串，不显示编号

2. **内容高度限制**
   - 内容区域使用 flexDirection="column"
   - 过长的内容可能导致列表项过高，影响可读性

3. **标记样式固定**
   - 当前使用固定的 dimColor 样式
   - 不支持自定义标记颜色或样式

### 边界情况

| 场景 | 行为 |
|------|------|
| 无 children | 渲染空的垂直 Box，保留标记 |
| 独立使用（无 Context） | marker 为空字符串，不显示编号 |
| 嵌套 OrderedList | 子列表项显示完整路径（如 "1.1."） |

### 改进建议

1. **支持自定义标记样式**
   ```typescript
   type OrderedListItemProps = {
     children: ReactNode;
     markerColor?: string;
     markerDimColor?: boolean;
   };
   ```

2. **添加列表项前缀图标支持**
   ```typescript
   type OrderedListItemProps = {
     children: ReactNode;
     icon?: ReactNode;
   };
   // 渲染：[标记] [图标] [内容]
   ```

3. **支持列表项禁用状态**
   ```typescript
   type OrderedListItemProps = {
     children: ReactNode;
     disabled?: boolean;
   };
   // disabled 时整体使用 dimColor
   ```

4. **优化独立使用体验**
   - 添加警告或回退机制
   - 或者支持独立使用时显示默认标记

5. **添加无障碍属性**
   ```tsx
   <Box gap={1} role="listitem">
     <Text dimColor aria-hidden="true">{marker}</Text>
     <Box flexDirection="column">{children}</Box>
   </Box>
   ```

6. **支持标记对齐方式**
   - 当前标记右对齐（通过父级的 padStart 实现）
   - 可考虑支持左对齐或居中对齐

---

## 源码映射说明

文件包含 base64 编码的 source map，指向原始 TypeScript 源码 `OrderedListItem.tsx`。调试时可使用 source map 还原原始代码位置。

原始源码结构（从 source map 还原）：
```typescript
import React, { createContext, type ReactNode, useContext } from 'react'
import { Box, Text } from '../ink.js'

export const OrderedListItemContext = createContext({ marker: '' })

type OrderedListItemProps = {
  children: ReactNode
}

export function OrderedListItem({
  children,
}: OrderedListItemProps): React.ReactNode {
  const { marker } = useContext(OrderedListItemContext)

  return (
    <Box gap={1}>
      <Text dimColor>{marker}</Text>
      <Box flexDirection="column">{children}</Box>
    </Box>
  )
}
```
