# Button.tsx 研究文档

## 场景与职责

`Button` 是 Ink 框架中的交互式按钮组件，提供高层次的按钮抽象，封装了：

1. **交互状态管理**: 管理 focused（焦点）、hovered（悬停）、active（激活）三种状态
2. **多模态触发**: 支持点击（鼠标）、键盘（Enter/Space）多种触发方式
3. **渲染委托**: 通过 render prop 模式让调用者根据状态自定义渲染内容
4. **无障碍支持**: 支持 tabIndex、autoFocus 等焦点管理属性

Button 是构建可交互 UI 的基础组件，适用于需要用户点击或键盘操作的场景。

## 功能点目的

### 1. 状态管理
- **focused**: 按钮当前是否拥有焦点
- **hovered**: 鼠标是否悬停在按钮上
- **active**: 按钮是否处于激活状态（按下后 100ms 内）

### 2. 触发机制
- **键盘触发**: 支持 Enter 和 Space 键触发 onAction
- **鼠标触发**: 点击时触发 onAction
- **视觉反馈**: 触发时设置 active 状态 100ms，提供视觉反馈

### 3. 渲染委托
- 支持 render prop 模式：`children` 可以是函数 `(state: ButtonState) => React.ReactNode`
- 调用者可以根据 focused/hovered/active 状态自定义渲染样式
- 如果不提供函数，则直接渲染 children

### 4. 焦点管理
- **tabIndex**: 默认为 0，可通过 props 自定义
- **autoFocus**: 支持挂载时自动获取焦点
- **ref**: 支持获取底层 Box 的 DOMElement 引用

## 具体技术实现

### 关键数据结构

```typescript
// 按钮状态
export type ButtonState = {
  focused: boolean;
  hovered: boolean;
  active: boolean;
};

// Props 定义
export type Props = Except<Styles, 'textWrap'> & {
  ref?: Ref<DOMElement>;
  onAction: () => void;  // 触发回调（必需）
  tabIndex?: number;
  autoFocus?: boolean;
  children: ((state: ButtonState) => React.ReactNode) | React.ReactNode;
};
```

### 关键流程

1. **状态初始化**
   ```typescript
   const [isFocused, setIsFocused] = useState(false);
   const [isHovered, setIsHovered] = useState(false);
   const [isActive, setIsActive] = useState(false);
   const activeTimer = useRef<ReturnType<typeof setTimeout> | null>(null);
   ```

2. **清理副作用**
   - 使用 useEffect 在组件卸载时清理 activeTimer
   - 防止内存泄漏

3. **键盘事件处理**
   ```typescript
   const handleKeyDown = (e: KeyboardEvent) => {
     if (e.key === "return" || e.key === " ") {
       e.preventDefault();
       setIsActive(true);
       onAction();
       // 100ms 后清除 active 状态
       activeTimer.current = setTimeout(setIsActive, 100, false);
     }
   };
   ```

4. **鼠标事件处理**
   - `handleClick`: 直接调用 onAction
   - `handleFocus/handleBlur`: 更新 focused 状态
   - `handleMouseEnter/handleMouseLeave`: 更新 hovered 状态

5. **内容渲染**
   - 根据状态创建 ButtonState 对象
   - 如果 children 是函数，调用它传入状态；否则直接使用 children

### 代码路径

```
Button.tsx
├── 导入依赖（React、Box、事件类型等）
├── ButtonState 类型定义
├── Props 类型定义
├── Button 函数组件
│   ├── 状态初始化（useState）
│   ├── Timer 引用（useRef）
│   ├── 清理副作用（useEffect）
│   ├── 事件处理器（handleKeyDown、handleClick 等）
│   ├── 内容渲染（根据状态决定）
│   └── 渲染 Box 组件
└── 默认导出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/ink/components/Box.tsx` | 底层布局容器，继承其样式能力 |
| `src/ink/dom.ts` | DOMElement 类型 |
| `src/ink/events/click-event.ts` | ClickEvent 类型 |
| `src/ink/events/focus-event.ts` | FocusEvent 类型 |
| `src/ink/events/keyboard-event.ts` | KeyboardEvent 类型 |
| `src/ink/styles.ts` | Styles 类型 |

### 调用方

Button 组件被广泛使用于：
- 各种工具 UI（如 BashTool、BriefTool 等）
- 交互式对话框和确认框
- 导航和菜单组件

### 继承关系

```
Button
├── Props 继承自 Box（Except<Styles, 'textWrap'>）
├── 渲染 Box 组件
└── 将事件处理器传递给 Box
```

## 依赖与外部交互

### 运行时依赖

1. **React Hooks**: useState、useEffect、useRef、useCallback
2. **React Compiler**: 使用 `_c` 函数进行自动记忆化
3. **Box 组件**: 作为底层容器，继承其布局和事件能力

### Props 交互

| Prop | 交互目标 | 说明 |
|------|----------|------|
| onAction | 调用者 | 按钮被触发时的回调（必需） |
| children | 渲染系统 | 普通节点或 render prop 函数 |
| ref | Box/DOMElement | 转发到 Box 组件 |
| tabIndex/autoFocus | Box | 焦点管理属性转发 |
| ...style | Box | 所有 Box 的样式属性都支持 |

### 事件流

```
用户交互
├── 键盘（Enter/Space）
│   └── handleKeyDown
│       ├── setIsActive(true)
│       ├── onAction()
│       └── setTimeout(setIsActive, 100, false)
├── 点击
│   └── handleClick
│       └── onAction()
├── 焦点变化
│   ├── handleFocus -> setIsFocused(true)
│   └── handleBlur -> setIsFocused(false)
└── 鼠标悬停
    ├── handleMouseEnter -> setIsHovered(true)
    └── handleMouseLeave -> setIsHovered(false)
```

## 风险、边界与改进建议

### 已知风险

1. **active 状态竞态**: 如果用户在 100ms 内多次触发，timer 可能被覆盖，导致 active 状态无法正确清除
2. **onAction 必需但未标记**: onAction 是功能必需的 props，但类型定义中没有标记为必需（虽然实现中假设它存在）
3. **React Compiler 依赖**: 与 Box 一样，依赖 React Compiler 的记忆化

### 边界情况

1. **快速连续触发**: 用户快速按 Enter 或连续点击时，active 状态的定时器管理可能出现问题
2. **焦点与悬停同时**: focused 和 hovered 可以同时为 true，调用者需要考虑这种组合状态的样式
3. **ref 转发**: ref 被转发到 Box 组件，不是直接到 DOMElement，调用者需要了解这一层间接

### 改进建议

1. **竞态处理**: 在设置新 timer 前清除旧的 timer，确保 active 状态正确管理
   ```typescript
   if (activeTimer.current) {
     clearTimeout(activeTimer.current);
   }
   activeTimer.current = setTimeout(...);
   ```

2. **Props 验证**: 考虑在开发环境对 onAction 进行运行时检查，确保其存在且为函数

3. **状态合并**: 考虑提供默认的按钮样式（虽然当前设计是 intentionally unstyled）

4. **长按支持**: 可以考虑添加 onLongPress 支持，用于需要长按触发的场景

5. **禁用状态**: 当前没有 disabled 状态支持，需要调用者在 onAction 中自行处理或根据状态渲染不同样式

### 代码质量

- 使用 React Compiler 进行记忆化，性能较好
- 事件处理器使用 useCallback 模式（通过 React Compiler 自动处理）
- 清理逻辑完善，避免内存泄漏
