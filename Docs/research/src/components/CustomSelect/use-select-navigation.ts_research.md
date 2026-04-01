# use-select-navigation.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useSelectNavigation` 是 CustomSelect 组件体系的**核心导航引擎**，负责管理选项列表的所有导航状态和行为。它采用 React `useReducer` 实现，提供可预测的导航状态管理。

### 1.2 设计特点
- **不可变状态更新**：使用 reducer 模式确保状态变化可追踪
- **自动视口管理**：聚焦变化时自动滚动视口
- **循环导航支持**：默认在列表首尾间循环
- **选项变化自适应**：选项数组变化时自动重置状态

### 1.3 典型使用场景
- **单选列表**：Select.tsx 的基础导航能力
- **多选列表**：SelectMulti.tsx 的导航基础
- **长列表分页**：支持 PageUp/PageDown 翻页
- **程序化聚焦**：通过 `focusValue` prop 外部控制聚焦

### 1.4 组件架构位置
```
Select.tsx / SelectMulti.tsx
    ↓ 调用
useSelectState.ts / useMultiSelectState.ts
    ↓ 调用
useSelectNavigation.ts (导航核心)
    ↓ 使用
option-map.ts (选项索引结构)
```

---

## 2. 功能点目的

### 2.1 核心功能清单

| 功能 | 目的 | 用户价值 |
|------|------|----------|
| 选项聚焦 | 维护当前聚焦选项 | 键盘导航的视觉反馈 |
| 视口管理 | 维护可见选项范围 | 长列表的性能优化 |
| 循环导航 | 首尾选项间循环 | 快速到达任意位置 |
| 翻页导航 | PageUp/PageDown | 长列表快速浏览 |
| 程序化聚焦 | `focusValue` prop | 外部控制聚焦位置 |
| 选项变化适配 | 自动重置和恢复 | 异步数据更新无缝 |
| 聚焦验证 | 确保聚焦值有效 | 选项删除后自动恢复 |

### 2.2 关键设计决策

**为何使用 useReducer 而非 useState？**

```typescript
// useReducer 优势：
// 1. 复杂状态逻辑集中管理
// 2. 便于测试（纯函数 reducer）
// 3. 状态变化可追溯
const [state, dispatch] = useReducer(reducer<T>, initArg, createDefaultState)
```

导航涉及多个相关状态（focusedValue, visibleFromIndex, visibleToIndex），使用 reducer 可以将更新逻辑集中，避免分散的 setState 调用导致的不一致。

**OptionMap 数据结构**

```typescript
class OptionMap<T> extends Map<T, OptionMapItem<T>> {
  readonly first: OptionMapItem<T> | undefined
  readonly last: OptionMapItem<T> | undefined
}

interface OptionMapItem<T> {
  label: ReactNode
  value: T
  description?: string
  previous: OptionMapItem<T> | undefined
  next: OptionMapItem<T> | undefined
  index: number
}
```

- 使用双向链表结构支持 O(1) 的相邻导航
- 同时维护 Map 支持 O(1) 的值查找
- 保留 index 支持基于位置的导航

---

## 3. 具体技术实现

### 3.1 状态结构

```typescript
interface State<T> {
  optionMap: OptionMap<T>        // 选项映射（含链表结构）
  visibleOptionCount: number     // 可视选项数量
  focusedValue: T | undefined    // 当前聚焦值
  visibleFromIndex: number       // 可视范围起始
  visibleToIndex: number         // 可视范围结束
}

type Action<T> =
  | { type: 'focus-next-option' }
  | { type: 'focus-previous-option' }
  | { type: 'focus-next-page' }
  | { type: 'focus-previous-page' }
  | { type: 'set-focus'; value: T }
  | { type: 'reset'; state: State<T> }
```

### 3.2 Reducer 实现详解

#### 3.2.1 下一个选项 (focus-next-option)

```typescript
case 'focus-next-option': {
  if (state.focusedValue === undefined) return state

  const item = state.optionMap.get(state.focusedValue)
  if (!item) return state

  // 循环：最后一个的 next 指向 first
  const next = item.next || state.optionMap.first
  if (!next) return state

  // 循环到开头时，重置视口到起始
  if (!item.next && next === state.optionMap.first) {
    return {
      ...state,
      focusedValue: next.value,
      visibleFromIndex: 0,
      visibleToIndex: state.visibleOptionCount,
    }
  }

  // 需要滚动时调整视口
  const needsToScroll = next.index >= state.visibleToIndex
  if (!needsToScroll) {
    return { ...state, focusedValue: next.value }
  }

  const nextVisibleToIndex = Math.min(
    state.optionMap.size,
    state.visibleToIndex + 1,
  )
  const nextVisibleFromIndex = nextVisibleToIndex - state.visibleOptionCount

  return {
    ...state,
    focusedValue: next.value,
    visibleFromIndex: nextVisibleFromIndex,
    visibleToIndex: nextVisibleToIndex,
  }
}
```

**关键逻辑**：
1. 获取当前项在 OptionMap 中的节点
2. 取 next 节点，如果不存在则循环到 first
3. 循环时重置视口到列表开头
4. 非循环时，如果 next 超出可视范围，滚动视口

#### 3.2.2 上一个选项 (focus-previous-option)

与 `focus-next-option` 对称，循环时指向 last，视口重置到列表末尾。

#### 3.2.3 翻页导航 (focus-next-page)

```typescript
case 'focus-next-page': {
  // 计算目标索引：当前索引 + 每页数量
  const targetIndex = Math.min(
    state.optionMap.size - 1,
    item.index + state.visibleOptionCount,
  )

  // 遍历找到目标节点（O(n)，但 n = visibleOptionCount，通常很小）
  let targetItem = state.optionMap.first
  while (targetItem && targetItem.index < targetIndex) {
    if (targetItem.next) targetItem = targetItem.next
    else break
  }

  // 调整视口使目标项可见
  const nextVisibleToIndex = Math.min(
    state.optionMap.size,
    targetItem.index + 1,
  )
  const nextVisibleFromIndex = Math.max(
    0,
    nextVisibleToIndex - state.visibleOptionCount,
  )

  return {
    ...state,
    focusedValue: targetItem.value,
    visibleFromIndex: nextVisibleFromIndex,
    visibleToIndex: nextVisibleToIndex,
  }
}
```

#### 3.2.4 设置聚焦 (set-focus)

```typescript
case 'set-focus': {
  // 避免重复聚焦
  if (state.focusedValue === action.value) return state

  const item = state.optionMap.get(action.value)
  if (!item) return state

  // 已在视口内，仅更新聚焦
  if (item.index >= state.visibleFromIndex && 
      item.index < state.visibleToIndex) {
    return { ...state, focusedValue: action.value }
  }

  // 需要滚动：最小滚动使项可见
  let nextVisibleFromIndex: number
  let nextVisibleToIndex: number

  if (item.index < state.visibleFromIndex) {
    // 在上方：滚动到顶部
    nextVisibleFromIndex = item.index
    nextVisibleToIndex = Math.min(
      state.optionMap.size,
      nextVisibleFromIndex + state.visibleOptionCount,
    )
  } else {
    // 在下方：滚动到底部
    nextVisibleToIndex = Math.min(state.optionMap.size, item.index + 1)
    nextVisibleFromIndex = Math.max(
      0,
      nextVisibleToIndex - state.visibleOptionCount,
    )
  }

  return {
    ...state,
    focusedValue: action.value,
    visibleFromIndex: nextVisibleFromIndex,
    visibleToIndex: nextVisibleToIndex,
  }
}
```

**最小滚动原则**：
- 目标在上方时，将其放在视口顶部
- 目标在下方时，将其放在视口底部
- 避免不必要的视口跳动

### 3.3 默认状态创建

```typescript
const createDefaultState = <T>({
  visibleOptionCount: customVisibleOptionCount,
  options,
  initialFocusValue,
  currentViewport,
}: Pick<UseSelectNavigationProps<T>, 'visibleOptionCount' | 'options'> & {
  initialFocusValue?: T
  currentViewport?: { visibleFromIndex: number; visibleToIndex: number }
}): State<T> => {
  const visibleOptionCount =
    typeof customVisibleOptionCount === 'number'
      ? Math.min(customVisibleOptionCount, options.length)
      : options.length

  const optionMap = new OptionMap<T>(options)
  
  // 确定初始聚焦值
  const focusedItem =
    initialFocusValue !== undefined && optionMap.get(initialFocusValue)
      ? optionMap.get(initialFocusValue)
      : optionMap.first
  const focusedValue = focusedItem ? initialFocusValue : optionMap.first?.value

  // 视口计算...
  // 如果提供了 currentViewport 且聚焦项在其中，保留视口
  // 否则调整视口以显示聚焦项
}
```

### 3.4 选项变化处理

```typescript
// 行526-544
const [lastOptions, setLastOptions] = useState(options)

if (options !== lastOptions && !isDeepStrictEqual(options, lastOptions)) {
  dispatch({
    type: 'reset',
    state: createDefaultState({
      visibleOptionCount,
      options,
      initialFocusValue: focusValue ?? state.focusedValue ?? initialFocusValue,
      currentViewport: {
        visibleFromIndex: state.visibleFromIndex,
        visibleToIndex: state.visibleToIndex,
      },
    }),
  })
  setLastOptions(options)
}
```

**智能重置策略**：
1. 尝试保留当前聚焦值（如果新选项中存在）
2. 尝试保留当前视口（如果聚焦值仍在视口内）
3. 使用 `isDeepStrictEqual` 避免引用变化导致的误重置

### 3.5 聚焦验证

```typescript
// 行592-602
const validatedFocusedValue = useMemo(() => {
  if (state.focusedValue === undefined) return undefined
  
  const exists = options.some(opt => opt.value === state.focusedValue)
  if (exists) return state.focusedValue
  
  // 回退到第一个选项
  return options[0]?.value
}, [state.focusedValue, options])
```

**场景**：选项在渲染过程中变化，但 reducer 的 reset action 还未处理
**解决**：使用 useMemo 验证聚焦值，无效时回退到首项

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
use-select-navigation.ts
├── react (useCallback, useEffect, useMemo, useReducer, useRef, useState)
├── util (isDeepStrictEqual)
├── ./option-map.ts
│   └── OptionMap<T> 类
└── ./select.ts
    └── OptionWithDescription<T> 类型
```

### 4.2 核心代码路径

| 功能 | 行号范围 | 关键逻辑 |
|------|----------|----------|
| State 类型 | 13-38 | State<T> 接口定义 |
| Action 类型 | 40-72 | Action<T> 联合类型 |
| Reducer | 74-330 | 完整 reducer 实现 |
| focus-next-option | 76-126 | 下一个选项逻辑 |
| focus-previous-option | 128-180 | 上一个选项逻辑 |
| focus-next-page | 182-229 | 下一页逻辑 |
| focus-previous-page | 231-272 | 上一页逻辑 |
| reset | 274-276 | 重置状态 |
| set-focus | 278-328 | 设置聚焦逻辑 |
| Props 类型 | 332-359 | UseSelectNavigationProps |
| 返回值类型 | 361-422 | SelectNavigation 接口 |
| 默认状态创建 | 424-503 | createDefaultState 函数 |
| Hook 实现 | 505-653 | useSelectNavigation 主体 |
| Reducer 初始化 | 512-520 | useReducer 调用 |
| 选项变化检测 | 526-544 | isDeepStrictEqual 比较 |
| Action creators | 546-577 | dispatch 包装函数 |
| visibleOptions | 579-586 | 可视选项计算 |
| 聚焦验证 | 588-609 | validatedFocusedValue |
| isInInput | 604-609 | 输入框检测 |
| onFocus 回调 | 614-618 | useEffect 触发 onFocus |
| focusValue 响应 | 621-628 | 程序化聚焦 |
| focusedIndex | 631-637 | 1-based 索引计算 |

---

## 5. 依赖与外部交互

### 5.1 OptionMap 详解

```typescript
// option-map.ts
class OptionMap<T> extends Map<T, OptionMapItem<T>> {
  readonly first: OptionMapItem<T> | undefined
  readonly last: OptionMapItem<T> | undefined

  constructor(options: OptionWithDescription<T>[]) {
    // 构建双向链表
    for (const option of options) {
      const item = {
        label: option.label,
        value: option.value,
        description: option.description,
        previous,           // 指向前一项
        next: undefined,    // 指向下一项（后续设置）
        index,
      }
      // 维护 first/last 引用
      // 添加到 Map
    }
  }
}
```

**设计优势**：
- Map 提供 O(1) 的值查找
- 链表提供 O(1) 的相邻导航
- index 支持基于位置的计算

### 5.2 与父组件的交互

```typescript
interface UseSelectNavigationProps<T> {
  visibleOptionCount?: number    // 默认 5
  options: OptionWithDescription<T>[]
  initialFocusValue?: T          // 初始聚焦
  onFocus?: (value: T) => void   // 聚焦回调
  focusValue?: T                 // 外部控制聚焦
}

interface SelectNavigation<T> {
  focusedValue: T | undefined
  focusedIndex: number           // 1-based，0 表示无聚焦
  visibleFromIndex: number
  visibleToIndex: number
  visibleOptions: Array<OptionWithDescription<T> & { index: number }>
  isInInput: boolean
  focusNextOption: () => void
  focusPreviousOption: () => void
  focusNextPage: () => void
  focusPreviousPage: () => void
  focusOption: (value: T | undefined) => void
  options: OptionWithDescription<T>[]
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1：翻页导航的性能
```typescript
// focus-next-page 中
let targetItem = state.optionMap.first
while (targetItem && targetItem.index < targetIndex) {
  if (targetItem.next) targetItem = targetItem.next
  else break
}
```
- **复杂度**：O(visibleOptionCount)
- **风险**：visibleOptionCount 很大时（如 100）可能卡顿
- **缓解**：通常 visibleOptionCount 较小（5-10）

#### 风险2：isDeepStrictEqual 的性能
```typescript
if (options !== lastOptions && !isDeepStrictEqual(options, lastOptions)) {
  // 重置状态
}
```
- **风险**：大型选项列表（>1000项）的深度比较开销大
- **场景**：频繁更新的大型列表
- **缓解**：选项列表通常较小

#### 风险3：聚焦验证的竞态
```typescript
// 行592-602
const validatedFocusedValue = useMemo(() => {
  const exists = options.some(opt => opt.value === state.focusedValue)
  // ...
}, [state.focusedValue, options])
```
- **风险**：options 变化后、reducer reset 前，validatedFocusedValue 可能指向不存在的值
- **当前处理**：使用 useMemo 提供临时回退
- **潜在问题**：短时间内可能显示不正确的聚焦状态

### 6.2 边界条件

| 边界条件 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| options 为空 | optionMap.first/last 为 undefined，focusedValue 为 undefined | createDefaultState |
| visibleOptionCount > options.length | 取 Math.min | 行433-436 |
| initialFocusValue 无效 | 回退到第一项 | 行440-441 |
| 循环导航（在最后一项按↓） | next 指向 first，重置视口 | 行88-102 |
| 循环导航（在第一项按↑） | previous 指向 last，重置视口 | 行147-159 |
| focusValue 频繁变化 | 每次变化都 dispatch set-focus | 行621-628 |
| 选项变化时保留视口 | 传递 currentViewport | 行534-539 |

### 6.3 改进建议

#### 建议1：添加虚拟滚动支持
```typescript
// 对于超大型列表，考虑虚拟滚动
interface VirtualScrollConfig {
  itemHeight: number
  overscan: number
}

// 只渲染可视区域内的选项
const visibleOptions = useMemo(() => {
  const start = Math.max(0, visibleFromIndex - overscan)
  const end = Math.min(options.length, visibleToIndex + overscan)
  return options.slice(start, end).map((opt, i) => ({
    ...opt,
    index: start + i,
  }))
}, [options, visibleFromIndex, visibleToIndex, overscan])
```

#### 建议2：优化翻页导航
```typescript
// 当前：线性遍历
// 改进：OptionMap 添加按索引查找方法
class OptionMap<T> extends Map<T, OptionMapItem<T>> {
  at(index: number): OptionMapItem<T> | undefined {
    // 从 first 开始遍历，但可优化为跳表或缓存
    let item = this.first
    while (item && item.index < index) {
      item = item.next
    }
    return item
  }
}
```

#### 建议3：添加搜索/过滤集成
```typescript
// 支持动态过滤后的导航
interface UseSelectNavigationProps<T> {
  // ...
  filter?: (option: OptionWithDescription<T>) => boolean
}

// 在 reducer 中考虑过滤后的索引映射
```

#### 建议4：添加动画支持
```typescript
// 视口滚动时添加平滑过渡
const [isScrolling, setIsScrolling] = useState(false)

const focusNextOption = useCallback(() => {
  setIsScrolling(true)
  dispatch({ type: 'focus-next-option' })
  setTimeout(() => setIsScrolling(false), 150)
}, [])

// 返回值中包含 isScrolling，UI 层可据此添加动画
```

#### 建议5：优化选项变化检测
```typescript
// 当前：深度比较整个数组
// 改进：使用选项的唯一标识符比较
const optionIds = useMemo(() => 
  options.map(o => getOptionId(o)).join(','),
[options])

// 比较 id 字符串而非深度比较
if (optionIds !== lastOptionIds) {
  // 重置状态
}
```

### 6.4 测试建议

应覆盖以下场景：
1. 各种导航操作（上下、翻页、循环）
2. 视口边界（刚好在边界内/外）
3. 选项变化（添加、删除、重排）
4. 程序化聚焦（focusValue prop）
5. 初始聚焦（initialFocusValue）
6. onFocus 回调触发时机
7. 空列表处理
8. 单选项列表
9. 频繁快速导航
10. 选项变化时的视口保留

---

## 7. 关联文件速查

| 文件 | 关系 | 用途 |
|------|------|------|
| `option-map.ts` | 依赖 | 选项索引数据结构 |
| `select.ts` | 依赖 | OptionWithDescription 类型 |
| `Select.tsx` | 调用方 | 单选组件 |
| `SelectMulti.tsx` | 调用方 | 多选组件 |
| `use-select-state.ts` | 调用方 | 单选状态管理 |
| `use-multi-select-state.ts` | 调用方 | 多选状态管理 |
| `use-select-input.ts` | 配合使用 | 输入处理 |
