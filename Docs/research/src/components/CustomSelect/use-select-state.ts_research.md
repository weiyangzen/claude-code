# use-select-state.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useSelectState` 是 CustomSelect 组件体系中**最基础的单选状态管理 Hook**。它封装了单选列表的核心状态逻辑，为 `Select.tsx` 组件提供简洁的状态接口。

### 1.2 设计特点
- **极简封装**：仅 157 行代码，职责单一清晰
- **组合优于继承**：复用 `useSelectNavigation` 的导航能力
- **受控/非受控兼容**：支持 `defaultValue` 和回调模式

### 1.3 典型使用场景
- **简单单选列表**：从多个选项中选择一项
- **带确认的选择**：选择后需要额外确认（通过 `onChange` 回调）
- **带取消功能的选择**：支持 Escape 取消（通过 `onCancel` 回调）

### 1.4 组件架构位置
```
Select.tsx (UI组件)
    ↓ 调用
useSelectState.ts (单选状态管理)
    ↓ 调用
useSelectNavigation.ts (导航核心)
```

与多选版本的对比：
```
SelectMulti.tsx
    ↓ 调用
useMultiSelectState.ts (多选状态管理，更复杂)
    ↓ 调用
useSelectNavigation.ts (共享导航核心)
```

---

## 2. 功能点目的

### 2.1 核心功能清单

| 功能 | 目的 | 用户价值 |
|------|------|----------|
| 单值选择 | 维护选中的单个值 | 标准单选交互 |
| 选择确认 | `selectFocusedOption` 方法 | 用户主动确认选择 |
| 导航集成 | 继承所有导航能力 | 完整的键盘导航 |
| 回调通知 | `onChange` / `onCancel` | 与父组件通信 |
| 默认值 | `defaultValue` | 非受控组件模式 |

### 2.2 与 useMultiSelectState 的差异

| 特性 | useSelectState | useMultiSelectState |
|------|----------------|---------------------|
| 选择值类型 | `T \| undefined` | `T[]` |
| 输入框支持 | 无 | 有（inputValues Map） |
| 提交按钮 | 无 | 有（isSubmitFocused） |
| 选择触发 | Enter 直接提交 | Space 切换，Enter 提交 |
| 代码复杂度 | 简单（157行） | 复杂（414行） |

---

## 3. 具体技术实现

### 3.1 状态结构

```typescript
// 内部状态
const [value, setValue] = useState<T | undefined>(defaultValue)

// 导航状态（来自 useSelectNavigation）
const navigation = useSelectNavigation<T>({
  visibleOptionCount,
  options,
  initialFocusValue: undefined,  // 单选默认不指定初始聚焦
  onFocus,
  focusValue,
})

// 组合后的返回值
interface SelectState<T> extends SelectNavigation<T> {
  value: T | undefined                    // 选中的值
  selectFocusedOption: () => void        // 确认选择方法
  onChange?: (value: T) => void          // 选择回调
  onCancel?: () => void                  // 取消回调
}
```

### 3.2 关键流程详解

#### 3.2.1 状态初始化

```typescript
// 行136
const [value, setValue] = useState<T | undefined>(defaultValue)
```

- 使用 `defaultValue` 初始化内部状态
- 组件作为非受控组件运行
- 选择后通过 `onChange` 回调通知父组件

#### 3.2.2 导航集成

```typescript
// 行138-144
const navigation = useSelectNavigation<T>({
  visibleOptionCount,
  options,
  initialFocusValue: undefined,  // 不指定初始聚焦，由 navigation 决定
  onFocus,
  focusValue,
})
```

**设计决策**：
- `initialFocusValue: undefined` - 让 navigation 使用默认逻辑（第一项）
- 不指定 `initialFocusValue` 为 `defaultValue`，因为聚焦和选择是独立概念

#### 3.2.3 选择确认

```typescript
// 行146-148
const selectFocusedOption = useCallback(() => {
  setValue(navigation.focusedValue)
}, [navigation.focusedValue])
```

**关键行为**：
- 将当前聚焦值设为选中值
- 不自动触发 `onChange`，由调用方决定何时通知
- 通常由 `useSelectInput` 在 Enter 键时调用

#### 3.2.4 返回值组合

```typescript
// 行150-156
return {
  ...navigation,           // 展开所有导航能力
  value,                   // 当前选中值
  selectFocusedOption,     // 确认选择方法
  onChange,                // 透传回调
  onCancel,                // 透传回调
}
```

**模式**：使用展开运算符继承 navigation 的所有属性和方法

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
use-select-state.ts
├── react (useCallback, useState)
├── ./select.ts
│   └── OptionWithDescription<T> 类型
└── ./use-select-navigation.ts
    └── useSelectNavigation() - 导航核心
```

### 4.2 核心代码路径

| 功能 | 行号范围 | 关键逻辑 |
|------|----------|----------|
| Props 类型 | 5-42 | UseSelectStateProps 接口 |
| 返回值类型 | 44-125 | SelectState 接口 |
| 状态初始化 | 136 | useState(defaultValue) |
| 导航集成 | 138-144 | useSelectNavigation 调用 |
| 选择方法 | 146-148 | selectFocusedOption |
| 返回值组合 | 150-156 | 对象展开组合 |

---

## 5. 依赖与外部交互

### 5.1 Props 详解

```typescript
interface UseSelectStateProps<T> {
  visibleOptionCount?: number    // 默认 5，可视选项数量
  options: OptionWithDescription<T>[]  // 选项列表
  defaultValue?: T               // 初始选中值
  onChange?: (value: T) => void  // 选择回调
  onCancel?: () => void          // 取消回调
  onFocus?: (value: T) => void   // 聚焦回调
  focusValue?: T                 // 外部控制聚焦
}
```

### 5.2 返回值详解

```typescript
interface SelectState<T> {
  // 来自 useSelectNavigation
  focusedValue: T | undefined
  focusedIndex: number
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
  
  // 本 Hook 添加
  value: T | undefined
  selectFocusedOption: () => void
  onChange?: (value: T) => void
  onCancel?: () => void
}
```

### 5.3 与 useSelectInput 的协作

```typescript
// useSelectInput.ts 中的使用
interface UseSelectProps<T> {
  state: SelectState<T>  // 来自 useSelectState
  // ...
}

// 使用示例
handlers['select:accept'] = () => {
  if (disableSelection === true) return
  if (state.focusedValue === undefined) return
  
  const focusedOption = options.find(opt => opt.value === state.focusedValue)
  if (focusedOption?.disabled === true) return
  
  state.selectFocusedOption?.()  // 调用确认选择
  state.onChange?.(state.focusedValue)  // 触发回调
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1：选择后未通知父组件
```typescript
const selectFocusedOption = useCallback(() => {
  setValue(navigation.focusedValue)
}, [navigation.focusedValue])
```
- **行为**：`selectFocusedOption` 只更新内部状态，不调用 `onChange`
- **设计意图**：由调用方（如 useSelectInput）决定何时通知
- **潜在问题**：如果直接调用 `selectFocusedOption` 而不调用 `onChange`，父组件状态不一致

#### 风险2：value 与 focusedValue 的分离
- **设计**：选中值（value）和聚焦值（focusedValue）是独立的
- **场景**：用户导航到某选项但未确认（未按 Enter）
- **状态**：focusedValue 变化，value 保持不变
- **潜在问题**：UI 需要同时显示聚焦状态和选中状态，容易混淆

### 6.2 边界条件

| 边界条件 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| defaultValue 未提供 | value 初始为 undefined | 行136 |
| defaultValue 无效 | 保留，但 UI 不会高亮 | 行136 |
| options 为空 | navigation 处理，focusedValue 为 undefined | useSelectNavigation |
| selectFocusedOption 时 focusedValue 为 undefined | setValue(undefined) | 行147 |
| 频繁调用 selectFocusedOption | useCallback 保证稳定性 | 行146 |

### 6.3 改进建议

#### 建议1：添加受控模式支持
```typescript
// 当前：仅支持非受控模式（defaultValue）
// 改进：支持受控模式（value + onChange）
interface UseSelectStateProps<T> {
  // ...
  value?: T              // 受控值
  defaultValue?: T       // 非受控初始值
}

// 内部使用
const isControlled = value !== undefined
const [internalValue, setInternalValue] = useState(defaultValue)
const currentValue = isControlled ? value : internalValue
```

#### 建议2：选择时自动触发 onChange
```typescript
// 当前：调用方需要手动触发 onChange
// 改进：selectFocusedOption 内部触发
const selectFocusedOption = useCallback(() => {
  const newValue = navigation.focusedValue
  setValue(newValue)
  if (newValue !== undefined) {
    onChange?.(newValue)  // 自动触发
  }
}, [navigation.focusedValue, onChange])
```

**注意**：这会改变现有行为，需要评估影响。

#### 建议3：添加选择验证
```typescript
const selectFocusedOption = useCallback(() => {
  const focusedOption = options.find(opt => opt.value === navigation.focusedValue)
  
  // 验证选项是否可被选
  if (focusedOption?.disabled) return
  if (focusedOption?.type === 'input' && !inputValue?.trim()) return
  
  setValue(navigation.focusedValue)
}, [navigation.focusedValue, options])
```

#### 建议4：支持清除选择
```typescript
// 添加清除方法
const clearSelection = useCallback(() => {
  setValue(undefined)
}, [])

return {
  // ...
  clearSelection,
}
```

### 6.4 测试建议

应覆盖以下场景：
1. 默认值初始化
2. 选择确认（selectFocusedOption）
3. 回调触发（onChange / onCancel）
4. 与 navigation 的集成
5. 空列表处理
6. 无效默认值处理
7. 频繁选择操作

---

## 7. 关联文件速查

| 文件 | 关系 | 用途 |
|------|------|------|
| `Select.tsx` | 调用方 | 单选组件 UI |
| `use-select-navigation.ts` | 依赖 | 导航核心能力 |
| `use-select-input.ts` | 配合使用 | 输入处理，调用 selectFocusedOption |
| `select.ts` | 依赖 | OptionWithDescription 类型 |
| `use-multi-select-state.ts` | 兄弟 | 多选版本（更复杂） |

---

## 8. 代码简洁性分析

### 8.1 代码统计
- 总行数：157 行
- 有效代码：约 30 行（去除类型定义和空行）
- 复杂度：低

### 8.2 简洁的原因
1. **职责单一**：只管理"单选值"这一个状态
2. **复用导航**：导航逻辑完全委托给 useSelectNavigation
3. **无输入处理**：输入处理委托给 useSelectInput
4. **无副作用**：没有 useEffect，纯状态管理

### 8.3 与多选版本的对比

| 指标 | useSelectState | useMultiSelectState |
|------|----------------|---------------------|
| 行数 | 157 | 414 |
| useState | 1 个 | 4 个 |
| useCallback | 1 个 | 3 个 |
| useInput | 无 | 有（完整键盘处理） |
| 复杂度 | 低 | 高 |

这种差异体现了"单选"和"多选"在交互复杂度上的本质区别。
