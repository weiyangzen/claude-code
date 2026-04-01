# CursorDeclarationContext.ts 研究文档

## 场景与职责

`CursorDeclarationContext` 是 Ink 框架中用于声明终端光标位置的 Context 系统，主要服务于：

1. **IME 输入支持**: 将终端光标定位到文本输入框的插入点，使 CJK（中日韩）输入法的预编辑文本能够正确显示在输入位置
2. **无障碍访问**: 让屏幕阅读器和屏幕放大镜能够跟踪原生光标位置，提升无障碍体验
3. **光标声明协调**: 解决多个组件同时声明光标位置时的冲突问题

这是一个低级别的基础设施组件，通常通过 `useDeclaredCursor` hook 使用，而不是直接使用 Context。

## 功能点目的

### 1. 光标声明类型

```typescript
export type CursorDeclaration = {
  readonly relativeX: number;  // 在声明节点内的列偏移（终端单元格宽度）
  readonly relativeY: number;  // 在声明节点内的行号
  readonly node: DOMElement;   // 提供绝对原点的 ink-box DOMElement
};
```

### 2. 安全的声明清除

```typescript
export type CursorDeclarationSetter = (
  declaration: CursorDeclaration | null,
  clearIfNode?: DOMElement | null,  // 条件清除参数
) => void;
```

**条件清除机制**: 
- 可选的 `clearIfNode` 参数使 `null` 成为条件清除
- 只有当当前声明的节点匹配 `clearIfNode` 时才清除
- 防止兄弟组件（如列表项）之间转移焦点时的竞争条件

### 3. 默认空实现

```typescript
const CursorDeclarationContext = createContext<CursorDeclarationSetter>(
  () => {},  // 默认空函数，避免未提供 Provider 时出错
);
```

## 具体技术实现

### 关键数据结构

```typescript
// 光标声明 - 相对坐标 + 参考节点
export type CursorDeclaration = {
  readonly relativeX: number;  // 节点内的列偏移
  readonly relativeY: number;  // 节点内的行偏移
  readonly node: DOMElement;   // Yoga 布局提供绝对原点的 DOM 节点
};

// 声明设置器 - 支持条件清除
export type CursorDeclarationSetter = (
  declaration: CursorDeclaration | null,
  clearIfNode?: DOMElement | null,
) => void;
```

### 关键流程

1. **Context 创建**
   - 使用 React 的 createContext 创建
   - 默认值为空函数，确保未包裹 Provider 时不会报错

2. **Provider 设置**
   - 在 App.tsx 中由 Ink 框架设置 Provider
   - 传入实际的 CursorDeclarationSetter 实现

3. **使用模式（通过 useDeclaredCursor hook）**
   ```typescript
   // 在组件中使用
   const setNode = useDeclaredCursor({
     line: 0,      // 相对于 Box 的行偏移
     column: 5,    // 相对于 Box 的列偏移
     active: true, // 是否激活声明
   });
   
   // 将 ref 附加到 Box
   <Box ref={setNode}>...</Box>
   ```

### 代码路径

```
CursorDeclarationContext.ts
├── 导入 React createContext
├── CursorDeclaration 类型定义
│   ├── relativeX: 相对列偏移
│   ├── relativeY: 相对行偏移
│   └── node: DOMElement 参考节点
├── CursorDeclarationSetter 类型定义
│   ├── declaration: 声明或 null
│   └── clearIfNode: 可选的条件清除参数
├── Context 创建（默认空函数）
└── 默认导出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `react` | createContext |
| `src/ink/dom.ts` | DOMElement 类型 |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/ink/hooks/use-declared-cursor.ts` | 主要消费者，提供声明光标的 hook |
| `src/ink/components/App.tsx` | 设置 Provider 值 |
| `src/ink/ink.tsx` | 读取声明并设置终端光标位置 |

### 集成流程

```
useDeclaredCursor hook
├── 调用 useContext(CursorDeclarationContext) 获取 setter
├── useLayoutEffect 中调用 setter
│   ├── active 为 true: setter({ relativeX, relativeY, node })
│   └── active 为 false: setter(null, node)  // 条件清除
└── 返回 ref callback 用于附加到 Box

App.tsx
└── <CursorDeclarationContext.Provider value={onCursorDeclaration}>
    └── 传入由 ink.tsx 提供的 setter

ink.tsx
├── 维护 cursorDeclaration 状态
├── 每帧渲染后检查声明
└── 将终端光标移动到声明位置
```

## 依赖与外部交互

### 运行时依赖

1. **React**: createContext
2. **Ink 核心**: DOMElement 类型定义

### 交互流程

```
文本输入组件
├── useDeclaredCursor({ line, column, active })
│   └── 返回 setNode ref callback
├── <Box ref={setNode}>
│   └── ref 附加时 nodeRef.current = DOMElement
└── useLayoutEffect
    ├── active && node: setter({ relativeX, relativeY, node })
    └── !active: setter(null, node)  // 条件清除

Ink 渲染管线
├── 渲染完成后
├── ink.tsx 检查 cursorDeclaration
└── 输出终端转义序列定位光标
```

### 条件清除机制详解

```typescript
// 场景：列表项 A 失去焦点，列表项 B 获得焦点
// 如果没有条件清除，执行顺序可能导致问题：

// 情况 1：B 的 effect 先于 A 的 cleanup
A.useLayoutEffect cleanup: setter(null)     // 清除 A 的声明
B.useLayoutEffect: setter({ node: B })      // 设置 B 的声明
// 结果：B 的声明生效 ✓

// 情况 2：A 的 cleanup 后于 B 的 effect（focus 移动方向相反）
B.useLayoutEffect: setter({ node: B })      // 设置 B 的声明
A.useLayoutEffect cleanup: setter(null)     // 错误地清除 B 的声明！
// 结果：没有光标声明 ✗

// 解决方案：条件清除
A.useLayoutEffect cleanup: setter(null, ANode)  // 只有当前是 A 时才清除
B.useLayoutEffect: setter({ node: B })
// 结果：B 的声明保留 ✓
```

## 风险、边界与改进建议

### 已知风险

1. **时序依赖**: 声明在 useLayoutEffect 中设置，依赖于 React 的渲染时序
2. **竞争条件**: 虽然条件清除解决了大部分问题，但复杂的焦点转移场景仍可能有问题
3. **坐标计算**: relativeX/relativeY 由调用者计算，如果计算错误会导致光标位置不正确

### 边界情况

1. **多个活跃声明**: 如果多个组件同时设置 active=true，后设置的会覆盖先设置的
2. **节点未挂载**: 如果 ref 尚未附加到 Box，node 为 null，声明不会生效
3. **Provider 未设置**: 如果组件树未包裹 Provider，setter 是空函数，声明静默失败

### 改进建议

1. **声明优先级**: 考虑添加优先级机制，重要的输入框（如搜索框）可以覆盖其他声明
   ```typescript
   type CursorDeclaration = {
     readonly relativeX: number;
     readonly relativeY: number;
     readonly node: DOMElement;
     readonly priority?: number;  // 新增
   };
   ```

2. **调试支持**: 在开发环境添加警告，当多个组件同时声明光标时提示

3. **自动计算坐标**: 考虑提供工具函数自动计算光标在 Box 内的坐标，减少调用者负担

4. **声明队列**: 考虑使用队列管理声明，支持动画过渡效果

5. **类型安全**: 考虑使用更严格的类型确保 relativeX/relativeY 为非负整数

### 测试建议

- 测试焦点在兄弟组件间快速切换时的行为
- 测试组件卸载时的清理行为
- 测试 Provider 未设置时的降级行为
- 测试多个活跃声明的竞争情况

### 相关配置

- `CLAUDE_CODE_ACCESSIBILITY`: 设置后保持光标可见，与光标声明配合使用
