# option-map.ts 研究文档

## 场景与职责

`option-map.ts` 实现了一个专门用于选项列表的数据结构 `OptionMap<T>`，继承自 JavaScript 原生的 `Map` 类。它为选择组件提供：

1. **O(1) 选项查找**：通过值快速定位选项
2. **双向链表结构**：支持选项间的快速前后导航
3. **索引维护**：每个选项维护其在列表中的索引位置

该数据结构是选择组件导航功能的基础，被 `use-select-navigation.ts` 使用。

## 功能点目的

### 1. 快速选项定位
通过 `Map` 的键值对存储，实现通过选项值（value）快速获取选项完整信息：
```typescript
const item = optionMap.get(focusedValue);
```

### 2. 双向导航支持
每个选项节点维护 `previous` 和 `next` 指针，支持：
- 向前导航（next）
- 向后导航（previous）
- 循环导航（首尾相连）

### 3. 索引信息维护
在构建时计算每个选项的索引，避免运行时重复计算：
```typescript
const item = {
  index,  // 0-based 索引
  // ...
}
```

### 4. 首尾快速访问
维护 `first` 和 `last` 指针，支持快速跳转到列表两端：
```typescript
optionMap.first  // 第一个选项
optionMap.last   // 最后一个选项
```

## 具体技术实现

### 数据结构定义

```typescript
type OptionMapItem<T> = {
  label: ReactNode;           // 显示标签
  value: T;                   // 选项值（作为 Map 的 key）
  description?: string;       // 可选描述
  previous: OptionMapItem<T> | undefined;  // 前一个节点
  next: OptionMapItem<T> | undefined;      // 后一个节点
  index: number;              // 在列表中的索引
}
```

### 类定义

```typescript
export default class OptionMap<T> extends Map<T, OptionMapItem<T>> {
  readonly first: OptionMapItem<T> | undefined
  readonly last: OptionMapItem<T> | undefined

  constructor(options: OptionWithDescription<T>[]) {
    // 构建双向链表和 Map
  }
}
```

### 构建算法

```typescript
constructor(options: OptionWithDescription<T>[]) {
  const items: Array<[T, OptionMapItem<T>]> = []
  let firstItem: OptionMapItem<T> | undefined
  let lastItem: OptionMapItem<T> | undefined
  let previous: OptionMapItem<T> | undefined
  let index = 0

  for (const option of options) {
    const item = {
      label: option.label,
      value: option.value,
      description: option.description,
      previous,           // 链接到前一个节点
      next: undefined,    // 将在下一次迭代中设置
      index,
    }

    if (previous) {
      previous.next = item  // 建立双向链接
    }

    firstItem ||= item    // 记录第一个节点
    lastItem = item       // 更新最后一个节点

    items.push([option.value, item])
    index++
    previous = item
  }

  super(items)            // 调用 Map 构造函数
  this.first = firstItem
  this.last = lastItem
}
```

### 时间复杂度分析

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| `get(value)` | O(1) | Map 原生操作 |
| `first` / `last` | O(1) | 直接属性访问 |
| `item.next` / `item.previous` | O(1) | 指针访问 |
| `item.index` | O(1) | 属性访问 |
| 构建 | O(n) | 单次遍历 |

## 关键代码路径与文件引用

### 直接依赖

| 文件 | 用途 |
|------|------|
| `react` | `ReactNode` 类型 |
| `./select.js` | `OptionWithDescription` 类型 |

### 被依赖文件

| 文件 | 用途 |
|------|------|
| `use-select-navigation.ts` | 核心使用者，用于导航状态管理 |

在 `use-select-navigation.ts` 中的使用：
```typescript
import OptionMap from './option-map.js'

// 在 createDefaultState 中创建
const optionMap = new OptionMap<T>(options)

// 在 reducer 中使用
const item = state.optionMap.get(state.focusedValue)
const next = item.next || state.optionMap.first  // 循环导航
```

## 依赖与外部交互

### 类型依赖

```typescript
import type { ReactNode } from 'react'
import type { OptionWithDescription } from './select.js'
```

### 与 use-select-navigation 的协作

```
OptionMap
├── 提供数据结构
│   ├── Map: value -> OptionMapItem
│   ├── 双向链表: previous/next
│   └── 首尾指针: first/last
└── 被 use-select-navigation 消费
    ├── 构建初始状态
    ├── 处理导航 action
    └── 计算可见选项
```

## 风险、边界与改进建议

### 风险

1. **内存占用**
   - 每个选项额外存储 `previous`/`next` 指针
   - 对于超大数据量（>10k 选项）可能有内存压力

2. **不可变性破坏**
   - `first` 和 `last` 是 `readonly` 但内部引用可变
   - 外部代码不应修改节点间的链接关系

3. **选项变更重建成本**
   - 每次选项变化都需要重建整个 OptionMap
   - 在 `use-select-navigation.ts` 中通过 `isDeepStrictEqual` 检测变化

### 边界情况

1. **空选项列表**
   - `first` 和 `last` 为 `undefined`
   - Map 为空

2. **单选项列表**
   - `first === last`
   - `previous` 和 `next` 均为 `undefined`

3. **重复值**
   - 依赖 Map 的去重特性，后出现的选项会覆盖先出现的
   - 实际使用中应保证选项值唯一

### 改进建议

1. **性能优化**
   - 对于静态选项列表，考虑使用 `WeakMap` 缓存
   - 添加选项变更的增量更新支持，避免全量重建

2. **类型安全**
   - 考虑添加运行时检查确保选项值唯一性
   - 添加对 `OptionWithDescription` 的验证

3. **功能扩展**
   - 可添加按索引快速查找的方法 `getByIndex(index)`
   - 支持选项分组（group）的层级结构

4. **代码改进**
   ```typescript
   // 建议添加边界检查方法
   has(value: T): boolean {
     return super.has(value)
   }
   
   // 建议添加安全获取方法
   getOrThrow(value: T): OptionMapItem<T> {
     const item = this.get(value)
     if (!item) throw new Error(`Option not found: ${value}`)
     return item
   }
   ```
