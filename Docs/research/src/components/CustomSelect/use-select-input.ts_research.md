# use-select-input.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useSelectInput` 是一个专门处理 Select 组件键盘输入的 React Hook。它作为**输入处理层**，桥接用户键盘操作与 Select 状态管理，支持单选和多选两种模式。

### 1.2 设计哲学
该 Hook 采用**分层处理策略**：
1. **Keybindings 层**：处理标准导航（上下箭头、Enter、Escape）
2. **Raw Input 层**：处理特殊输入（数字键、PageUp/PageDown、Tab、Space）

这种分层使得：
- 标准导航可通过配置化 keybindings 自定义
- 特殊输入保持固定的交互语义

### 1.3 典型使用场景
- **单选列表**：Select.tsx 中使用，处理标准选择输入
- **多选列表**：SelectMulti.tsx 中不使用（有自己的输入处理）
- **带输入框的选项**：特殊处理输入框模式下的键盘事件
- **图片选择模式**：与 Attachments 系统协作处理图片导航

### 1.4 组件架构位置
```
Select.tsx (UI组件)
    ↓ 调用
useSelectState.ts (状态管理)
    ↓ 调用
useSelectInput.ts (输入处理) ←→ useKeybindings() / useInput()
    ↓ 调用
useRegisterOverlay() (覆盖层注册)
```

---

## 2. 功能点目的

### 2.1 核心功能清单

| 功能 | 目的 | 用户价值 |
|------|------|----------|
| 双模式输入处理 | keybindings + useInput 分层 | 标准导航可配置，特殊输入固定 |
| 输入框模式检测 | 识别当前是否在输入框选项 | 在输入框内时改变键盘行为 |
| 图片选择集成 | 支持 Attachments 导航 | 在输入框内可浏览粘贴的图片 |
| 数字键选择 | 1-9 快速选择选项 | 高效选择，无需导航 |
| 禁用选择模式 | `disableSelection` 控制 | 纯浏览模式，防止误选 |
| 边界导航回调 | `onUpFromFirstItem` / `onDownFromLastItem` | 与其他组件导航衔接 |
| Tab 输入模式切换 | `onInputModeToggle` | 在选项和输入框间切换焦点 |

### 2.2 关键设计决策

**为何使用 useKeybindings + useInput 双轨制？**

```typescript
// useKeybindings 处理：标准导航（可配置）
useKeybindings({
  'select:next': () => state.focusNextOption(),
  'select:previous': () => state.focusPreviousOption(),
  'select:accept': () => { /* 选择当前项 */ },
  'select:cancel': () => state.onCancel?.(),
}, { context: 'Select' })

// useInput 处理：特殊输入（固定语义）
useInput((input, key) => {
  // PageUp/PageDown 翻页
  // Tab 切换输入模式
  // 数字键快速选择
  // Space 多选切换
})
```

优势：
1. **可配置性**：用户可通过 keybindings 配置自定义快捷键
2. **语义稳定**：数字键选择、翻页等行为保持一致
3. **优先级控制**：keybindings 处理完后，useInput 处理剩余输入

**输入框模式的特殊处理**

当聚焦在 `type: 'input'` 的选项上时：
- 数字键透传给 TextInput（输入数字而非选择选项）
- 上下箭头导航 Select（而非移动光标）
- 图片选择模式下，所有输入被抑制，交给 Attachments 处理

---

## 3. 具体技术实现

### 3.1 Props 接口定义

```typescript
interface UseSelectProps<T> {
  isDisabled?: boolean           // 禁用输入处理
  disableSelection?: boolean | 'numeric'  // 禁用选择（true=全禁，'numeric'=仅禁数字）
  state: SelectState<T>          // 来自 useSelectState 的状态
  options: OptionWithDescription<T>[]
  isMultiSelect?: boolean        // 是否多选（影响 Space 键行为）
  onUpFromFirstItem?: () => void // 从首项向上导航回调
  onDownFromLastItem?: () => void // 从末项向下导航回调
  onInputModeToggle?: (value: T) => void // Tab 切换输入模式
  inputValues?: Map<T, string>   // 输入框选项的当前值
  imagesSelected?: boolean       // 图片选择模式激活
  onEnterImageSelection?: () => boolean // 进入图片选择模式
}
```

### 3.2 关键流程详解

#### 3.2.1 输入框模式检测

```typescript
// 行103-107
const isInInput = useMemo(() => {
  const focusedOption = options.find(opt => opt.value === state.focusedValue)
  return focusedOption?.type === 'input'
}, [options, state.focusedValue])
```

**用途**：
- 决定是否抑制 keybindings 的导航处理
- 决定 useInput 中的输入处理方式

#### 3.2.2 Keybindings 处理器构建

```typescript
// 行112-164
const keybindingHandlers = useMemo(() => {
  const handlers: Record<string, () => void> = {}

  if (!isInInput) {
    // 非输入框模式：添加导航和选择处理器
    handlers['select:next'] = () => { /* ... */ }
    handlers['select:previous'] = () => { /* ... */ }
    handlers['select:accept'] = () => { /* ... */ }
  }

  if (state.onCancel) {
    handlers['select:cancel'] = () => state.onCancel!()
  }

  return handlers
}, [/* deps */])
```

**关键逻辑**：
- 在输入框内时，不注册 `select:next`/`select:previous`/`select:accept`
- 这样 j/k/Enter 等键会透传给 TextInput

#### 3.2.3 边界导航处理

```typescript
// 行116-135
handlers['select:next'] = () => {
  if (onDownFromLastItem) {
    const lastOption = options[options.length - 1]
    if (lastOption && state.focusedValue === lastOption.value) {
      onDownFromLastItem()  // 触发外部回调
      return
    }
  }
  state.focusNextOption()  // 正常导航
}
```

**设计**：
- 当提供 `onDownFromLastItem` 时，从最后一项向下导航会触发回调而非循环
- 用于与其他组件（如另一个列表）的导航衔接

#### 3.2.4 输入框模式下的键盘处理

```typescript
// 行187-231
if (currentIsInInput) {
  // 图片选择模式：抑制所有输入
  if (imagesSelected) return

  // DOWN 箭头进入图片选择模式
  if (key.downArrow && onEnterImageSelection?.()) {
    event.stopImmediatePropagation()
    return
  }

  // 上下箭头导航 Select（而非移动光标）
  if (key.downArrow || (key.ctrl && input === 'n')) {
    // ... 边界检查 ...
    state.focusNextOption()
    event.stopImmediatePropagation()
    return
  }

  // 其他键（包括数字）透传给 TextInput
  return
}
```

**关键行为**：
1. 图片选择模式激活时，完全抑制输入处理
2. DOWN 箭头尝试进入图片选择模式（如果有图片）
3. 上下箭头导航 Select，并阻止事件冒泡
4. 数字键直接返回，透传给 TextInput

#### 3.2.5 数字键选择逻辑

```typescript
// 行255-282
if (disableSelection !== 'numeric' && /^[0-9]+$/.test(normalizedInput)) {
  const index = parseInt(normalizedInput) - 1
  if (index >= 0 && index < state.options.length) {
    const selectedOption = state.options[index]!
    
    if (selectedOption.disabled === true) return
    
    if (selectedOption.type === 'input') {
      const currentValue = inputValues?.get(selectedOption.value) ?? ''
      
      if (currentValue.trim()) {
        // 有值：直接提交
        state.onChange?.(selectedOption.value)
        return
      }
      
      if (selectedOption.allowEmptySubmitToCancel) {
        // 允许空提交：直接提交
        state.onChange?.(selectedOption.value)
        return
      }
      
      // 无值：聚焦到输入框让用户输入
      state.focusOption(selectedOption.value)
      return
    }
    
    // 普通选项：直接提交
    state.onChange?.(selectedOption.value)
  }
}
```

**复杂逻辑解析**：
- 输入类型选项有值时：直接提交（用户可 Tab 进入编辑）
- 输入类型选项无值但允许空提交：直接提交
- 输入类型选项无值且不允许空提交：聚焦到输入框
- 普通选项：直接提交

### 3.3 键盘映射表

| 按键 | 常规模式 | 输入框模式 | 图片选择模式 |
|------|----------|------------|--------------|
| j/k 或 ↑/↓ | 导航（keybindings） | 导航 Select | 抑制（Attachments 处理） |
| Enter | 选择（keybindings） | 透传 | 抑制 |
| Escape | 取消（keybindings） | 透传 | 抑制 |
| Tab | 切换输入模式 | 切换输入模式 | 抑制 |
| 1-9 | 选择对应选项 | 透传输入 | 抑制 |
| Space | 多选切换 | 透传输入 | 抑制 |
| PageUp/Dn | 翻页 | - | 抑制 |
| ↓ (在输入框) | - | 尝试进入图片选择 | - |

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
use-select-input.ts
├── react (useMemo)
├── ../../context/overlayContext.js
│   └── useRegisterOverlay()
├── ../../ink/events/input-event.js
│   └── InputEvent 类型
├── ../../ink.js
│   ├── useInput()
│   └── Key 类型
├── ../../keybindings/useKeybinding.js
│   ├── useKeybinding()
│   └── useKeybindings()
├── ../../utils/stringUtils.js
│   ├── normalizeFullWidthDigits()
│   └── normalizeFullWidthSpace()
├── ./select.js
│   └── OptionWithDescription<T> 类型
└── ./use-select-state.js
    └── SelectState<T> 类型
```

### 4.2 核心代码路径

| 功能 | 行号范围 | 关键逻辑 |
|------|----------|----------|
| Props 定义 | 13-84 | UseSelectProps 接口 |
| 覆盖层注册 | 99-101 | useRegisterOverlay |
| 输入框检测 | 103-107 | isInInput useMemo |
| Keybindings 构建 | 112-164 | handlers 构建逻辑 |
| Keybindings 注册 | 166-169 | useKeybindings 调用 |
| useInput 回调 | 173-286 | 完整输入处理 |
| Tab 处理 | 181-185 | 输入模式切换 |
| 输入框模式 | 187-231 | 特殊输入处理 |
| 翻页 | 233-239 | PageUp/PageDown |
| Space 多选 | 242-253 | 多选切换 |
| 数字键选择 | 255-282 | 1-9 选择逻辑 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖详解

#### 5.1.1 useRegisterOverlay
```typescript
useRegisterOverlay('select', !!state.onCancel)
```
- **条件注册**：仅在 `state.onCancel` 存在时注册
- **目的**：有取消功能的 Select 才需要拦截 Escape 键

#### 5.1.2 useKeybindings
```typescript
useKeybindings(keybindingHandlers, {
  context: 'Select',
  isActive: !isDisabled,
})
```
- **上下文**：'Select'，可从 keybindings 配置中读取自定义绑定
- **动态激活**：通过 `isActive` 控制

#### 5.1.3 useInput
```typescript
useInput(handleInput, { isActive: !isDisabled })
```
- **优先级**：在 keybindings 之后处理
- **用途**：处理 keybindings 不处理的特殊输入

### 5.2 与 SelectState 的交互

```typescript
interface SelectState<T> {
  focusedValue: T | undefined
  visibleFromIndex: number
  options: OptionWithDescription<T>[]
  focusNextOption: () => void
  focusPreviousOption: () => void
  focusNextPage: () => void
  focusPreviousPage: () => void
  focusOption: (value: T | undefined) => void
  selectFocusedOption?: () => void
  onChange?: (value: T) => void
  onCancel?: () => void
}
```

**使用方式**：
- 读取状态：`state.focusedValue`, `state.visibleFromIndex`
- 调用方法：`state.focusNextOption()`, `state.onChange?.(value)`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1：图片选择模式的输入抑制
```typescript
// 行190
if (imagesSelected) return
```
- **风险**：完全抑制输入可能导致某些快捷键无法使用
- **场景**：用户在图片选择模式下想按 Escape 退出整个 Select
- **当前行为**：Escape 由 Attachments 处理，可退出图片选择模式

#### 风险2：数字键与输入框的冲突
- **风险**：在输入框内输入数字时，如果输入框失去焦点再获得，可能触发选择
- **当前处理**：输入框模式下直接返回，数字键透传
- **边界**：快速输入多位数时，每个数字是独立事件，不会累积

#### 风险3：Tab 键的循环焦点
- **风险**：Tab 键在输入框和普通选项间切换，但没有视觉指示器
- **依赖**：父组件提供 `onInputModeToggle` 回调

### 6.2 边界条件

| 边界条件 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| options 为空 | find 返回 undefined，isInInput 为 false | 行104-106 |
| focusedValue 为 undefined | 各种操作都有保护检查 | 分散在各处 |
| disableSelection = true | select:accept 处理器直接返回 | 行137 |
| disableSelection = 'numeric' | 数字键选择被禁用 | 行256 |
| 输入框选项 disabled | 数字键选择时检查 | 行262-264 |
| imagesSelected 变化 | 通过 props 传入，响应式处理 | 行77, 190 |

### 6.3 改进建议

#### 建议1：添加连续数字输入支持
```typescript
// 当前：每次按键独立处理
// 改进：支持输入 "12" 选择第12项
const [pendingNumber, setPendingNumber] = useState('')

// 在数字键处理中
if (/^[0-9]$/.test(normalizedInput)) {
  const newPending = pendingNumber + normalizedInput
  const index = parseInt(newPending) - 1
  
  if (index < state.options.length) {
    // 设置延迟确认定时器
    setPendingNumber(newPending)
    setTimeout(() => {
      setPendingNumber('')
      // 执行选择
    }, 300)
  }
}
```

#### 建议2：优化输入框模式检测
```typescript
// 当前：每次渲染都遍历查找
const isInInput = useMemo(() => {
  const focusedOption = options.find(opt => opt.value === state.focusedValue)
  return focusedOption?.type === 'input'
}, [options, state.focusedValue])

// 改进：SelectState 直接提供 isInInput
// 避免重复计算，已在 useSelectNavigation 中计算过
```

#### 建议3：添加键盘事件日志
```typescript
// 用于调试复杂的键盘交互
useEffect(() => {
  if (process.env.DEBUG_SELECT_INPUT) {
    console.log('SelectInput:', { input, key, isInInput, imagesSelected })
  }
}, [input, key])
```

#### 建议4：统一输入框退出行为
```typescript
// 当前：Tab 切换输入模式，但 Escape 行为不一致
// 建议：添加明确的退出输入模式快捷键
handlers['select:exitInputMode'] = () => {
  if (isInInput && onInputModeToggle) {
    onInputModeToggle(state.focusedValue!)
  }
}
```

### 6.4 测试建议

应覆盖以下场景：
1. 输入框模式下各种键盘输入
2. 图片选择模式的输入抑制
3. 数字键选择的各种边界（禁用选项、输入框选项）
4. disableSelection 的不同值
5. 边界导航回调的触发
6. Tab 键切换输入模式
7. 全角/半角数字输入

---

## 7. 关联文件速查

| 文件 | 关系 | 用途 |
|------|------|------|
| `Select.tsx` | 调用方 | 单选组件 UI |
| `use-select-state.ts` | 依赖 | 单选状态管理 |
| `use-multi-select-state.ts` | 不直接使用 | 多选有自己的输入处理 |
| `select-input-option.tsx` | 配合 | 输入类型选项渲染 |
| `select-option.tsx` | 配合 | 普通选项渲染 |
| `overlayContext.tsx` | 依赖 | 覆盖层注册 |
| `useKeybinding.ts` | 依赖 | keybindings 系统 |
| `use-input.ts` | 依赖 | 原始输入处理 |
| `stringUtils.ts` | 依赖 | 字符串标准化 |
