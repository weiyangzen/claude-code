# renderPlaceholder.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`renderPlaceholder.ts` 是一个纯工具函数模块，负责处理文本输入框的占位符（placeholder）渲染逻辑。它独立于 React 组件生命周期，专注于视觉表现的计算。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 空输入框提示 | 当输入框为空时显示占位符文本（如 "How can Claude help?"） |
| 光标视觉反馈 | 在占位符上显示反色光标，提示用户输入位置 |
| 语音输入模式 | 录音时隐藏占位符文本，仅显示光标 |
| 终端焦点状态 | 根据终端是否获得焦点调整光标显示 |

### 1.3 调用方
- `src/components/BaseTextInput.tsx` - 基础文本输入组件

---

## 2. 功能点目的

### 2.1 占位符渲染
- **目的**：在空输入框中提供视觉提示，引导用户操作
- **实现**：使用 `chalk.dim()` 将占位符显示为暗淡颜色
- **条件**：仅在 `value.length === 0` 且提供了 `placeholder` 时显示

### 2.2 光标集成
- **目的**：在占位符上显示光标位置，提供视觉反馈
- **实现**：将占位符首字符反色显示（`chalk.inverse`）
- **条件**：需要同时满足 `showCursor && focus && terminalFocus`

### 2.3 语音模式支持
- **目的**：语音输入时简化视觉表现
- **实现**：`hidePlaceholderText` 标志完全隐藏占位符文本
- **用例**：录音状态下仅显示闪烁光标

---

## 3. 具体技术实现

### 3.1 类型定义

```typescript
type PlaceholderRendererProps = {
  placeholder?: string      // 占位符文本内容
  value: string             // 当前输入值
  showCursor?: boolean      // 是否显示光标
  focus?: boolean           // 输入框是否获得焦点
  terminalFocus: boolean    // 终端是否获得焦点
  invert?: (text: string) => string  // 反色函数（默认识 chalk.inverse）
  hidePlaceholderText?: boolean      // 是否隐藏占位符文本（语音模式）
}
```

### 3.2 核心算法

```typescript
export function renderPlaceholder({
  placeholder,
  value,
  showCursor,
  focus,
  terminalFocus = true,
  invert = chalk.inverse,
  hidePlaceholderText = false,
}: PlaceholderRendererProps): {
  renderedPlaceholder: string | undefined
  showPlaceholder: boolean
} {
  let renderedPlaceholder: string | undefined = undefined

  if (placeholder) {
    if (hidePlaceholderText) {
      // 语音模式：仅显示光标
      renderedPlaceholder =
        showCursor && focus && terminalFocus ? invert(' ') : ''
    } else {
      // 正常模式：暗淡显示占位符
      renderedPlaceholder = chalk.dim(placeholder)

      // 光标显示：首字符反色
      if (showCursor && focus && terminalFocus) {
        renderedPlaceholder =
          placeholder.length > 0
            ? invert(placeholder[0]!) + chalk.dim(placeholder.slice(1))
            : invert(' ')
      }
    }
  }

  // 仅在空值时显示占位符
  const showPlaceholder = value.length === 0 && Boolean(placeholder)

  return { renderedPlaceholder, showPlaceholder }
}
```

### 3.3 光标渲染逻辑详解

| 条件组合 | 结果 |
|----------|------|
| `showCursor=true, focus=true, terminalFocus=true` | 首字符反色 + 其余暗淡 |
| `showCursor=true, focus=true, terminalFocus=false` | 全部暗淡（无光标） |
| `showCursor=false` 或 `focus=false` | 全部暗淡（无光标） |
| `hidePlaceholderText=true` | 仅空格反色（语音模式） |

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

```
BaseTextInput.tsx
  └── renderPlaceholder()
      └── chalk.dim() / chalk.inverse()
```

### 4.2 依赖关系

```
renderPlaceholder.ts
└── chalk (npm package) - 终端样式处理
```

### 4.3 使用示例（来自 BaseTextInput.tsx）

```typescript
const {
  showPlaceholder,
  renderedPlaceholder
} = renderPlaceholder({
  placeholder: props.placeholder,
  value: props.value,
  showCursor: props.showCursor,
  focus: props.focus,
  terminalFocus,
  invert,
  hidePlaceholderText
})

// 渲染时
{showPlaceholder && props.placeholderElement 
  ? props.placeholderElement 
  : showPlaceholder && renderedPlaceholder 
    ? <Ansi>{renderedPlaceholder}</Ansi> 
    : <Ansi>{renderedValue}</Ansi>
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 版本 |
|------|------|------|
| `chalk` | 终端字符串样式 | ^5.x |

### 5.2 无外部状态依赖

该模块是纯函数：
- 不访问全局状态
- 不发起网络请求
- 不操作文件系统
- 仅依赖输入参数

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 空占位符 | `placeholder=""` 时渲染空字符串 | 外层组件控制 |
| 长占位符截断 | 终端宽度限制未处理 | 外层组件处理 |
| 多行占位符 | 未支持换行 | 单行的设计假设 |

### 6.2 边界条件

1. **空字符串值**：`value=""` 时显示占位符
2. **空格值**：`value=" "` 不显示占位符（长度 > 0）
3. **undefined placeholder**：安全返回 `undefined`
4. **零长度占位符**：`placeholder=""` 等效于无占位符

### 6.3 改进建议

1. **多行支持**：扩展以支持多行占位符的渲染
2. **动画光标**：支持闪烁动画（需配合 `useBlink` hook）
3. **截断提示**：当占位符超过终端宽度时显示省略号
4. **样式自定义**：支持更多样式选项（颜色、粗细等）

### 6.4 代码质量

- **优点**：
  - 纯函数，易于测试
  - 单一职责，逻辑清晰
  - 默认参数提供向后兼容
  
- **潜在改进**：
  - 添加 JSDoc 示例
  - 考虑提取光标渲染为独立函数
  - 类型定义可移至共享 types 文件
