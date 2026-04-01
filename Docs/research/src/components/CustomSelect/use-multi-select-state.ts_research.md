# use-multi-select-state.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useMultiSelectState` 是一个 React Hook，专门用于管理**多选列表组件**的复杂状态。它是 CustomSelect 组件体系的核心状态管理器之一，与 `useSelectState`（单选）形成互补。

### 1.2 典型使用场景
- **多选项配置选择**：如 MCP 服务器配置选择、功能开关批量启用
- **标签/分类多选**：为对话或文件添加多个标签
- **批量操作前置选择**：选择多个文件进行批量处理
- **带输入框的多选**：支持选项中包含可编辑的输入字段（如自定义参数）

### 1.3 组件架构位置
```
SelectMulti.tsx (UI组件)
    ↓ 调用
useMultiSelectState.ts (状态管理) ←→ useSelectNavigation.ts (导航逻辑)
    ↓ 调用
useRegisterOverlay() (覆盖层注册)
useInput() (Ink 原始输入处理)
```

---

## 2. 功能点目的

### 2.1 核心功能清单

| 功能 | 目的 | 用户价值 |
|------|------|----------|
| 多值选择管理 | 维护 `selectedValues` 数组状态 | 支持同时选择多个选项 |
| 输入字段集成 | 管理 `inputValues` Map | 支持带输入框的选项类型 |
| 提交按钮控制 | `isSubmitFocused` 状态 | 区分选择态和提交态，防止误操作 |
| 键盘导航 | 继承 navigation 能力 | 纯键盘高效操作 |
| 数字快捷键 | 1-9 数字键直接切换 | 快速选择，提升效率 |
| 边界回调 | `onDownFromLastItem` / `onUpFromFirstItem` | 支持与其他组件的导航衔接 |

### 2.2 关键设计决策

**为何需要独立的提交按钮控制？**
- 多选场景下，Enter 键的行为存在歧义：是"切换当前选项选中状态"还是"提交所有选择"？
- 通过 `submitButtonText` 参数显式控制：
  - 提供 `submitButtonText`：显示提交按钮，Enter 在选项上切换选择，在按钮上提交
  - 不提供：Enter 直接提交，Space 切换选择

**输入字段值与选中状态的联动**
- 当输入框有值时，自动将该选项加入选中列表
- 当输入框清空时，自动从选中列表移除
- 这种设计确保用户输入的内容不会被遗漏

---

## 3. 具体技术实现

### 3.1 状态结构定义

```typescript
// 内部状态 (useState)
const [selectedValues, setSelectedValues] = useState<T[]>(defaultValue)
const [isSubmitFocused, setIsSubmitFocused] = useState(false)
const [inputValues, setInputValues] = useState<Map<T, string>>(initMap)
const [lastOptions, setLastOptions] = useState(options) // 用于检测变化

// 派生状态 (来自 useSelectNavigation)
interface MultiSelectState<T> extends SelectNavigation<T> {
  selectedValues: T[]
  inputValues: Map<T, string>
  isSubmitFocused: boolean
  updateInputValue: (value: T, inputValue: string) => void
  onCancel: () => void
}
```

### 3.2 关键流程详解

#### 3.2.1 选项变化时的状态重置

```typescript
// 行176-180
if (options !== lastOptions && !isDeepStrictEqual(options, lastOptions)) {
  setSelectedValues(defaultValue)
  setLastOptions(options)
}
```

**场景**：异步加载的选项数据在组件挂载后更新（如 `getAllMcpConfigs()` 返回）
**问题**：如果不重置，已勾选的选项可能对应新的选项列表中的不同项目
**解决**：使用 `isDeepStrictEqual` 深度比较，避免引用变化导致的误重置

#### 3.2.2 输入值更新与选中状态联动

```typescript
// 行217-244
const updateInputValue = useCallback((value: T, inputValue: string) => {
  // 1. 更新 inputValues Map
  setInputValues(prev => {
    const next = new Map(prev)
    next.set(value, inputValue)
    return next
  })

  // 2. 调用选项的 onChange 回调
  const option = options.find(opt => opt.value === value)
  if (option && option.type === 'input') {
    option.onChange(inputValue)
  }

  // 3. 自动同步选中状态
  updateSelectedValues(prev => {
    if (inputValue) {
      if (!prev.includes(value)) return [...prev, value]
      return prev
    } else {
      return prev.filter(v => v !== value)
    }
  })
}, [options, updateSelectedValues])
```

**设计要点**：
- 输入有值 → 自动选中
- 输入清空 → 自动取消选中
- 确保 UI 状态与数据状态一致

#### 3.2.3 键盘输入处理流程

```typescript
// 行247-404: useInput 回调
useInput((input, key, event) => {
  // 1. 输入标准化（全角转半角）
  const normalizedInput = normalizeFullWidthDigits(input)
  
  // 2. 检测是否在输入框内
  const isInInput = focusedOption?.type === 'input'
  
  // 3. 输入框内只允许导航键
  if (isInInput) {
    const isAllowedKey = key.upArrow || key.downArrow || key.escape || 
                         key.tab || key.return || 
                         (key.ctrl && (input === 'n' || input === 'p' || key.return))
    if (!isAllowedKey) return
  }
  
  // 4. 各类键盘事件处理...
}, { isActive: !isDisabled })
```

**输入框模式特殊处理**：
- 在输入框内时，普通字符输入直接透传给 TextInput
- 只允许特定导航键（上下箭头、Escape、Tab、Enter、Ctrl+N/P）

### 3.3 键盘映射表

| 按键 | 常规模式 | 输入框模式 | 提交按钮聚焦 |
|------|----------|------------|--------------|
| ↑ / Ctrl+P / k | 上一个选项 | 上一个选项 | 返回最后选项 |
| ↓ / Ctrl+N / j | 下一个选项 | 下一个选项 | 触发 onDownFromLastItem |
| Tab | 下一个（可进入提交按钮） | 允许 | - |
| Shift+Tab | 上一个 | 允许 | 返回最后选项 |
| Enter | 提交/切换选择 | Ctrl+Enter 提交 | 提交 |
| Space | 切换选择 | 透传输入 | - |
| 1-9 | 切换对应索引选项 | 透传输入 | - |
| Escape | 取消 | 取消 | 取消 |
| PageUp/PageDown | 翻页 | - | - |

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
use-multi-select-state.ts
├── react (useCallback, useState)
├── util (isDeepStrictEqual)
├── ../../context/overlayContext.js
│   └── useRegisterOverlay() - 注册为覆盖层
├── ../../ink/events/input-event.js
│   └── InputEvent 类型
├── ../../ink.js
│   └── useInput() - 原始输入处理
├── ../../utils/stringUtils.js
│   ├── normalizeFullWidthDigits() - 全角数字转半角
│   └── normalizeFullWidthSpace() - 全角空格转半角
├── ./select.js
│   └── OptionWithDescription<T> 类型
└── ./use-select-navigation.js
    └── useSelectNavigation() - 导航状态管理
```

### 4.2 核心代码路径

| 功能 | 行号范围 | 关键逻辑 |
|------|----------|----------|
| Props 定义 | 14-94 | UseMultiSelectStateProps 接口 |
| 状态定义 | 96-151 | MultiSelectState 接口 |
| 状态初始化 | 169-191 | useState 初始化 |
| 选项变化检测 | 176-180 | isDeepStrictEqual 比较 |
| 导航集成 | 203-211 | useSelectNavigation 调用 |
| 覆盖层注册 | 215 | useRegisterOverlay |
| 输入值更新 | 217-244 | updateInputValue 实现 |
| 键盘处理 | 247-404 | useInput 回调 |
| Tab 处理 | 270-293 | 提交按钮焦点控制 |
| 箭头导航 | 296-341 | 上下箭头及边界处理 |
| 翻页 | 344-352 | PageUp/PageDown |
| Enter/Space | 355-382 | 选择/提交逻辑 |
| 数字键 | 385-395 | 1-9 快捷选择 |
| Escape | 398-401 | 取消处理 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖详解

#### 5.1.1 useRegisterOverlay
```typescript
useRegisterOverlay('multi-select')
```
- **目的**：向全局覆盖层系统注册当前多选组件
- **作用**：确保 CancelRequestHandler 不会拦截 Escape 键（当多选激活时，Escape 应关闭多选而非取消请求）
- **来源**：`src/context/overlayContext.tsx`

#### 5.1.2 useSelectNavigation
```typescript
const navigation = useSelectNavigation<T>({
  visibleOptionCount,
  options,
  initialFocusValue: initialFocusLast ? options[options.length - 1]?.value : undefined,
  onFocus,
  focusValue,
})
```
- **提供能力**：
  - `focusedValue` - 当前聚焦值
  - `visibleFromIndex` / `visibleToIndex` - 可视范围
  - `visibleOptions` - 当前可视选项
  - `focusNextOption()` / `focusPreviousOption()` - 导航方法
  - `focusNextPage()` / `focusPreviousPage()` - 翻页方法
  - `focusOption()` - 指定聚焦

#### 5.1.3 useInput
```typescript
useInput(handler, { isActive: !isDisabled })
```
- **来源**：`src/ink/hooks/use-input.ts`
- **功能**：Ink 框架提供的原始键盘输入处理
- **特点**：
  - 使用 `useLayoutEffect` 同步启用 raw mode
  - 支持 `stopImmediatePropagation()` 阻止事件冒泡

### 5.2 与父组件的交互

```typescript
// 回调接口
interface UseMultiSelectStateProps<T> {
  onChange?: (values: T[]) => void    // 选择变化通知
  onCancel: () => void                // 取消操作
  onFocus?: (value: T) => void        // 聚焦通知
  onSubmit?: (values: T[]) => void    // 提交操作
  onDownFromLastItem?: () => void     // 从最后一项向下导航
  onUpFromFirstItem?: () => void      // 从第一项向上导航
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1：选项变化时的选中状态重置
```typescript
// 行176-180
if (options !== lastOptions && !isDeepStrictEqual(options, lastOptions)) {
  setSelectedValues(defaultValue)
  setLastOptions(options)
}
```
- **风险**：深度比较 `isDeepStrictEqual` 对于大型选项列表可能有性能问题
- **场景**：选项数量 > 1000 且频繁更新时
- **缓解**：目前选项列表通常较小（配置项、标签等）

#### 风险2：输入框状态下的键盘冲突
- **风险**：在输入框内按数字键，预期是输入数字，但可能被解释为选择选项
- **当前处理**：输入框内只允许特定导航键，数字键透传
- **边界**：`hideIndexes` 为 true 时，数字键不应触发选择（已处理，行385）

#### 风险3：提交按钮与选项的边界导航
- **风险**：提交按钮聚焦时，按上箭头应返回最后选项，但逻辑复杂容易出错
- **当前处理**：行329-331 显式处理

### 6.2 边界条件

| 边界条件 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| options 为空 | navigation 会处理，focusedValue 为 undefined | useSelectNavigation |
| defaultValue 包含无效值 | 保留，但 UI 不会高亮不存在的选项 | 行169 |
| 输入框选项无初始值 | inputValues Map 中无该键，显示空 | 行183-191 |
| 同时按多个修饰键 | 按代码顺序处理，先匹配的先执行 | 行247-404 |
| 快速连续按键 | useInput 保证事件顺序，状态更新按 React 批次 | - |

### 6.3 改进建议

#### 建议1：添加选项变化前的确认机制
```typescript
// 当前：直接重置
setSelectedValues(defaultValue)

// 改进：如果当前有选择，提供保留或重置的选项
// 或通过配置参数控制行为
```

#### 建议2：优化数字键处理
```typescript
// 当前：全局监听数字键
if (!hideIndexes && /^[0-9]+$/.test(normalizedInput)) {
  // ...
}

// 改进：支持多位数快速选择（如输入 "12" 选择第12项）
// 需要添加延迟确认机制
```

#### 建议3：添加全选/清空快捷键
```typescript
// 建议添加：
// Ctrl+A - 全选
// Ctrl+Shift+A - 清空选择
// 需要评估与系统快捷键的冲突
```

#### 建议4：性能优化
```typescript
// 对于大型列表，考虑：
// 1. 虚拟化可视区域外的选项
// 2. 使用 useMemo 缓存 visibleOptions 计算
// 3. 延迟加载选项描述
```

#### 建议5：类型安全增强
```typescript
// 当前：selectedValues 是 T[]
// 改进：如果 T 是复杂对象，考虑使用唯一标识符而非对象引用
// 避免 isDeepStrictEqual 的频繁调用
```

### 6.4 测试建议

应覆盖以下场景：
1. 选项异步加载后的状态重置
2. 输入框内各种键盘输入
3. 提交按钮的显示/隐藏逻辑
4. 边界导航（onDownFromLastItem / onUpFromFirstItem）
5. 数字键在 hideIndexes 不同值下的行为
6. 全角/半角输入转换
7. 快速连续按键的响应

---

## 7. 关联文件速查

| 文件 | 关系 | 用途 |
|------|------|------|
| `SelectMulti.tsx` | 调用方 | 多选组件 UI |
| `use-select-navigation.ts` | 依赖 | 导航状态管理 |
| `use-select-state.ts` | 兄弟 | 单选状态管理（参考实现） |
| `select.tsx` | 依赖 | 单选组件 UI |
| `select-input-option.tsx` | 配合 | 输入类型选项渲染 |
| `select-option.tsx` | 配合 | 普通选项渲染 |
| `option-map.ts` | 间接依赖 | 选项索引结构 |
| `overlayContext.tsx` | 依赖 | 覆盖层注册 |
| `stringUtils.ts` | 依赖 | 字符串标准化 |
| `use-input.ts` | 依赖 | 原始输入处理 |
