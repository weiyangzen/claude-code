# Box.tsx 研究文档

## 场景与职责

`Box` 是 Ink 框架中最基础、最核心的布局组件，相当于浏览器中的 `<div style="display: flex">`。它基于 Yoga 布局引擎实现 Flexbox 布局系统，负责：

1. **布局容器**：作为所有 UI 元素的容器，支持 Flexbox 布局属性（flexDirection、flexGrow、flexShrink、flexWrap 等）
2. **事件处理**：支持鼠标点击、键盘事件、焦点事件、鼠标悬停等交互
3. **样式应用**：支持 margin、padding、gap、border、overflow 等丰富的样式属性
4. **焦点管理**：通过 tabIndex 和 autoFocus 支持键盘导航和焦点控制

Box 是构建任何 Ink 应用的基石，几乎所有其他组件（如 Button、ScrollBox、NoSelect 等）都基于 Box 构建。

## 功能点目的

### 1. Flexbox 布局支持
- **flexDirection**: 定义主轴方向（row、column、row-reverse、column-reverse）
- **flexGrow/flexShrink**: 控制子元素在空间分配中的伸缩行为
- **flexWrap**: 控制子元素是否换行（nowrap、wrap、wrap-reverse）
- **alignItems/alignSelf/justifyContent**: 控制交叉轴和主轴上的对齐方式

### 2. 事件系统
- **onClick**: 鼠标左键点击事件（仅在 AlternateScreen 中有效）
- **onFocus/onBlur**: 焦点获取/失去事件，支持捕获阶段（onFocusCapture/onBlurCapture）
- **onKeyDown**: 键盘按下事件，支持捕获阶段（onKeyDownCapture）
- **onMouseEnter/onMouseLeave**: 鼠标进入/离开事件

### 3. 焦点管理
- **tabIndex**: Tab 键导航顺序，>=0 参与 Tab 循环，-1 仅支持程序化聚焦
- **autoFocus**: 组件挂载时自动获取焦点
- **ref**: 支持获取 DOMElement 引用进行程序化操作

### 4. 样式系统
- **Margin/Padding**: 支持统一设置（margin/padding）和单边设置（marginTop 等）
- **Gap**: 支持统一间隙（gap）和行列间隙（columnGap/rowGap）
- **Overflow**: 支持 visible、hidden、scroll 三种溢出处理
- **Border**: 支持边框样式、颜色、单边控制
- **Position**: 支持 absolute/relative 定位

### 5. 开发时警告
通过 `warn.ifNotInteger` 对 margin、padding、gap 等属性进行整数校验，帮助开发者发现潜在问题。

## 具体技术实现

### 关键数据结构

```typescript
// Props 类型定义（基于 Styles，排除 textWrap）
export type Props = Except<Styles, 'textWrap'> & {
  ref?: Ref<DOMElement>;
  tabIndex?: number;
  autoFocus?: boolean;
  onClick?: (event: ClickEvent) => void;
  onFocus?: (event: FocusEvent) => void;
  onFocusCapture?: (event: FocusEvent) => void;
  onBlur?: (event: FocusEvent) => void;
  onBlurCapture?: (event: FocusEvent) => void;
  onKeyDown?: (event: KeyboardEvent) => void;
  onKeyDownCapture?: (event: KeyboardEvent) => void;
  onMouseEnter?: () => void;
  onMouseLeave?: () => void;
};
```

### 关键流程

1. **属性解构与默认值处理**
   - 使用 React Compiler 的 `_c` 函数进行记忆化
   - 为 flexWrap、flexDirection、flexGrow、flexShrink 设置默认值
   - 验证 margin、padding、gap 相关属性为整数

2. **样式合并**
   - 将 flex 相关属性与传入的 style 合并
   - 处理 overflowX/overflowY 的默认值（从 overflow 继承或 'visible'）

3. **渲染**
   - 渲染为 `<ink-box>` 自定义元素
   - 传递所有事件处理器和样式属性

### 代码路径

```
Box.tsx
├── 导入依赖（React、类型定义、warn 模块）
├── Props 类型定义
├── Box 函数组件
│   ├── 属性解构与默认值
│   ├── 整数校验（warn.ifNotInteger）
│   ├── 样式合并
│   └── 渲染 <ink-box>
└── 默认导出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/ink/styles.ts` | Styles 类型定义和样式应用函数 |
| `src/ink/dom.ts` | DOMElement 类型定义 |
| `src/ink/events/click-event.ts` | ClickEvent 类型 |
| `src/ink/events/focus-event.ts` | FocusEvent 类型 |
| `src/ink/events/keyboard-event.ts` | KeyboardEvent 类型 |
| `src/ink/warn.ts` | 开发时警告工具 |

### 被调用方

Box 组件被以下组件/文件广泛使用：

- `src/ink/components/Button.tsx` - 按钮组件基于 Box
- `src/ink/components/NoSelect.tsx` - 不可选择区域基于 Box
- `src/ink/components/ScrollBox.tsx` - 滚动容器基于 Box
- `src/ink/components/ErrorOverview.tsx` - 错误展示使用 Box
- 以及大量业务组件（BashTool、BriefTool 等）

### 渲染管线

Box 组件渲染的 `<ink-box>` 元素在 reconciler 中被处理：

1. `src/ink/reconciler.ts` - createInstance 创建 DOM 节点
2. `src/ink/dom.ts` - createNode 创建 ink-box 元素
3. `src/ink/styles.ts` - applyStyles 应用样式到 Yoga 节点
4. `src/ink/render-node-to-output.ts` - 渲染节点到输出

## 依赖与外部交互

### 运行时依赖

1. **React Compiler**: 使用 `_c` 函数进行自动记忆化，减少不必要的重渲染
2. **Yoga Layout**: 通过 `styles.ts` 将 Flexbox 属性应用到 Yoga 节点
3. **事件系统**: 通过 `_eventHandlers` 存储事件处理器，由 `events/dispatcher.ts` 调度

### Props 交互

| Prop | 交互目标 | 说明 |
|------|----------|------|
| ref | DOMElement | 获取对底层 DOM 节点的引用 |
| tabIndex/autoFocus | FocusManager | 参与焦点管理和 Tab 导航 |
| onClick | Dispatcher | 鼠标点击时由 dispatcher 触发 |
| onKeyDown | Dispatcher | 键盘事件由 dispatcher 触发 |
| onFocus/onBlur | Dispatcher | 焦点事件由 dispatcher 触发 |
| style | Yoga Node | 通过 styles.ts 应用到布局引擎 |

## 风险、边界与改进建议

### 已知风险

1. **React Compiler 依赖**: 代码使用 React Compiler 的 `_c` 函数进行记忆化，如果编译器配置变更或升级，可能影响性能优化
2. **事件仅在 AlternateScreen 有效**: onClick、onMouseEnter/onMouseLeave 等鼠标事件仅在 `<AlternateScreen>` 中有效，开发者容易误解
3. **整数校验仅在开发时**: warn.ifNotInteger 只在开发环境生效，生产环境不会检查

### 边界情况

1. **overflow 处理**: overflowX/overflowY 的默认值处理逻辑需要注意，它们会从 overflow 继承，默认为 'visible'
2. **flexShrink 默认值**: 默认为 1，与 CSS 标准一致，但可能与某些预期不同
3. **事件冒泡**: ClickEvent 支持冒泡，需要调用 stopImmediatePropagation() 阻止

### 改进建议

1. **文档增强**: 增加更多关于事件仅在 AlternateScreen 有效的警告注释
2. **类型安全**: 考虑使用更严格的类型来区分 AlternateScreen 内外可用的事件
3. **性能优化**: 当前的整数校验在每次渲染时都执行，可以考虑只在开发环境且 props 变化时执行
4. **测试覆盖**: 增加对边界情况（如 overflow 组合、flex 属性交互）的测试

### 相关配置

- `CLAUDE_CODE_DEBUG_REPAINTS`: 设置后可启用重绘调试，查看组件渲染情况
- `CLAUDE_CODE_ACCESSIBILITY`: 设置后保持光标可见，用于无障碍访问
