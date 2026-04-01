# use-declared-cursor.ts 深入研究

## 场景与职责

`useDeclaredCursor` 是 Ink 终端 UI 框架中用于声明式控制终端物理光标位置的高级 Hook。它解决了终端应用中以下关键问题：

1. **IME 输入法支持**：终端模拟器在物理光标位置渲染 IME 预编辑文本，将光标定位到文本输入框的插入点可使 CJK 输入内联显示
2. **无障碍支持**：屏幕阅读器和屏幕放大器跟踪原生光标位置，正确声明光标位置可让这些工具跟随输入
3. **多组件协调**：处理多个组件同时声明光标位置时的冲突解决

## 功能点目的

### 1. 光标位置声明
- 组件声明其内部的光标应该出现在终端的哪个位置
- 位置是相对于声明节点的 Yoga 布局计算的

### 2. 激活状态管理
- 支持条件性声明（通过 `active` 参数）
- 非激活时自动清理声明

### 3. 安全清理
- 组件卸载时自动清理声明
- 防止兄弟组件间的声明冲突

## 具体技术实现

### 接口定义

```typescript
export function useDeclaredCursor({
  line,
  column,
  active,
}: {
  line: number      // 节点内的行号
  column: number    // 节点内的列号（终端单元格宽度）
  active: boolean   // 是否激活声明
}): (element: DOMElement | null) => void
```

### 核心数据结构

```typescript
// CursorDeclarationContext 定义
export type CursorDeclaration = {
  readonly relativeX: number    // 节点内的 X 偏移
  readonly relativeY: number    // 节点内的 Y 偏移
  readonly node: DOMElement     // 声明的 DOM 节点
}

export type CursorDeclarationSetter = (
  declaration: CursorDeclaration | null,
  clearIfNode?: DOMElement | null,
) => void
```

### 关键实现逻辑

1. **节点引用管理**：
   ```typescript
   const nodeRef = useRef<DOMElement | null>(null)
   const setNode = useCallback((node: DOMElement | null) => {
     nodeRef.current = node
   }, [])
   ```

2. **声明更新（每次渲染）**：
   ```typescript
   useLayoutEffect(() => {
     const node = nodeRef.current
     if (active && node) {
       setCursorDeclaration({ relativeX: column, relativeY: line, node })
     } else {
       setCursorDeclaration(null, node)
     }
   })
   ```
   注意：没有依赖数组，确保每次渲染都重新声明

3. **卸载清理**：
   ```typescript
   useLayoutEffect(() => {
     return () => {
       setCursorDeclaration(null, nodeRef.current)
     }
   }, [setCursorDeclaration])
   ```

### 节点身份检查机制

清理时使用条件清除（`clearIfNode` 参数），处理两种风险场景：

1. **Memo 化组件场景**：
   - 一个 memo 化的激活实例在其他地方（如 Footer 中的搜索输入框）
   - 当前非激活实例重新渲染时，不应清除 memo 实例的声明

2. **兄弟组件切换场景**：
   - 焦点在列表项之间移动
   - 当焦点移动方向与兄弟顺序相反时，新失活项的 effect 在新激活项的 set 之后运行
   - 没有节点检查，它会覆盖兄弟的声明

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/CursorDeclarationContext.ts` | 定义光标声明上下文类型和默认值 |
| `src/ink/dom.ts` | 定义 `DOMElement` 类型 |

### 消费方（ink.tsx）

```typescript
// ink.tsx 中的相关代码
private cursorDeclaration: CursorDeclaration | null = null;

setCursorDeclaration: CursorDeclarationSetter = (declaration, clearIfNode) => {
  if (declaration === null && clearIfNode) {
    // 条件清除：只有当前声明属于指定节点时才清除
    if (this.cursorDeclaration?.node === clearIfNode) {
      this.cursorDeclaration = null;
    }
  } else {
    this.cursorDeclaration = declaration;
  }
  this.scheduleRender();
};
```

### 渲染时应用

```typescript
// 在 render 过程中
if (this.cursorDeclaration) {
  const { node, relativeX, relativeY } = this.cursorDeclaration;
  // 计算绝对位置并移动物理光标
  // ...
}
```

## 依赖与外部交互

### 与 Ink 主类的交互

1. **声明设置**：通过 Context 调用 Ink 实例的 `setCursorDeclaration` 方法
2. **渲染触发**：声明变化时触发重新渲染
3. **位置计算**：渲染时基于 Yoga 布局计算绝对位置

### 时序保证

注释中详细说明了时序：
- ref 附加和 useLayoutEffect 都在 React 的 layout 阶段触发
- 在 `resetAfterCommit` 调用 `scheduleRender` 之后
- `scheduleRender` 通过 `queueMicrotask` 延迟 `onRender`
- 因此 `onRender` 在 layout effects 提交后运行，在第一帧读取新的声明

测试环境使用 `onImmediateRender`（同步，无 microtask），因此测试需要显式调用 `ink.onRender()`。

## 风险、边界与改进建议

### 潜在风险

1. **时序依赖**：依赖 React 的 layout effect 时序，如果 React 内部实现变化可能受影响
2. **竞争条件**：多个组件同时声明时的竞争条件（已通过节点检查缓解）
3. **内存泄漏**：如果 ref 清理不当，可能导致 DOM 节点引用无法释放

### 边界情况

1. **节点未附加**：如果 ref 回调未被调用或传入 null，声明不会生效
2. **Yoga 布局未计算**：如果节点尚未完成布局计算，位置可能不正确
3. **快速切换**：组件快速挂载/卸载可能导致声明状态不一致

### 改进建议

1. **添加调试模式**：
   ```typescript
   // 开发模式下显示光标声明边界
   if (process.env.DEBUG_CURSOR) {
     console.log('Cursor declaration:', { line, column, active, node: nodeRef.current })
   }
   ```

2. **支持多光标**：
   - 当前只支持单光标声明
   - 可考虑支持多光标场景（如多选编辑器）

3. **自动检测行/列**：
   - 当前需要手动指定 line/column
   - 可考虑基于子组件自动检测（如 TextInput 组件内部计算）

4. **动画支持**：
   - 支持光标平滑移动动画
   - 在声明切换时提供视觉反馈

### 测试建议

1. 测试兄弟组件切换：确保焦点在列表项间移动时声明正确
2. 测试 memo 组件：验证 memo 化组件的声明不会被意外清除
3. 测试快速切换：验证快速挂载/卸载场景下的稳定性
4. 测试 IME 输入：验证 CJK 输入时的光标位置正确性
