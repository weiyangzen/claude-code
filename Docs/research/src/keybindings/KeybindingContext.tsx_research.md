# KeybindingContext.tsx 研究文档

## 场景与职责

`KeybindingContext.tsx` 是 Claude Code 键盘快捷键系统的核心 React Context 模块，负责在组件树中传递键盘绑定相关的状态和操作。它定义了键盘绑定系统的"依赖注入"机制，使得任何嵌套组件都能访问和操作用户自定义的键盘快捷键。

**核心职责：**
1. 创建和管理 `KeybindingContext` - 键盘绑定系统的全局状态容器
2. 提供 `KeybindingProvider` 组件 - 将绑定配置和状态注入 React 树
3. 提供 `useKeybindingContext` Hook - 供子组件访问上下文（强制要求 Provider 包裹）
4. 提供 `useOptionalKeybindingContext` Hook - 可选访问，返回 undefined 而非报错
5. 提供 `useRegisterKeybindingContext` Hook - 注册/注销活动上下文，支持上下文优先级

**架构位置：**
```
KeybindingSetup (Provider Setup)
    ↓
KeybindingProvider (本文件) → 提供 Context Value
    ↓
ChordInterceptor (拦截器)
    ↓
子组件 (通过 useKeybinding/useKeybindings 消费)
```

---

## 功能点目的

### 1. HandlerRegistration 类型
定义动作回调的注册结构：
```typescript
type HandlerRegistration = {
  action: string;           // 动作标识符，如 "app:toggleTodos"
  context: KeybindingContextName;  // 所属上下文，如 "Global", "Chat"
  handler: () => void;      // 回调函数
}
```

### 2. KeybindingContextValue 接口
Context 提供的完整功能集：

| 属性/方法 | 类型 | 用途 |
|-----------|------|------|
| `resolve` | `(input, key, contexts) => ChordResolveResult` | 解析按键输入到动作 |
| `setPendingChord` | `(pending) => void` | 更新和弦（组合键）等待状态 |
| `getDisplayText` | `(action, context) => string \| undefined` | 获取动作的快捷键显示文本 |
| `bindings` | `ParsedBinding[]` | 所有解析后的绑定（用于帮助显示） |
| `pendingChord` | `ParsedKeystroke[] \| null` | 当前等待中的和弦状态 |
| `activeContexts` | `Set<KeybindingContextName>` | 当前活动的上下文集合 |
| `registerActiveContext` | `(context) => void` | 注册活动上下文（组件挂载时调用） |
| `unregisterActiveContext` | `(context) => void` | 注销活动上下文（组件卸载时调用） |
| `registerHandler` | `(registration) => () => void` | 注册动作处理器，返回注销函数 |
| `invokeAction` | `(action) => boolean` | 调用指定动作的所有处理器（ChordInterceptor 使用） |

### 3. ProviderProps 接口
`KeybindingProvider` 接收的属性：
- `bindings`: 解析后的绑定配置
- `pendingChordRef`: Ref 对象，用于同步访问和弦状态（避免 React 状态延迟）
- `pendingChord`: 状态值，用于触发重新渲染（UI 更新）
- `setPendingChord`: 更新和弦状态的函数
- `activeContexts`: 当前活动上下文集合
- `registerActiveContext` / `unregisterActiveContext`: 上下文注册/注销函数
- `handlerRegistryRef`: Ref 到处理器注册表（ChordInterceptor 使用）

### 4. useRegisterKeybindingContext Hook
**用途：** 在组件挂载时注册一个键盘绑定上下文，使其绑定优先于 Global 绑定。

**典型用例：**
```tsx
function ThemePicker() {
  useRegisterKeybindingContext('ThemePicker')
  // 现在 ThemePicker 的 ctrl+t 绑定会覆盖 Global 的 todo 切换绑定
}
```

**实现机制：**
- 使用 `useLayoutEffect` 在挂载时注册，卸载时自动注销
- 支持 `isActive` 参数条件性注册

---

## 具体技术实现

### 1. KeybindingProvider 组件实现

**核心逻辑：**
```typescript
export function KeybindingProvider(props): React.ReactNode {
  const { bindings, pendingChordRef, pendingChord, setPendingChord, 
          activeContexts, registerActiveContext, unregisterActiveContext,
          handlerRegistryRef, children } = props;
  
  // 1. getDisplay 函数 - 缓存优化
  const getDisplay = useMemo(
    () => (action, context) => getBindingDisplayText(action, context, bindings),
    [bindings]
  );
  
  // 2. registerHandler - 处理器注册
  const registerHandler = useMemo(() => {
    return (registration) => {
      const registry = handlerRegistryRef.current;
      if (!registry.has(registration.action)) {
        registry.set(registration.action, new Set());
      }
      registry.get(registration.action).add(registration);
      // 返回注销函数
      return () => { /* 从 Set 中删除 */ };
    };
  }, [handlerRegistryRef]);
  
  // 3. invokeAction - 调用处理器（按上下文优先级）
  const invokeAction = useMemo(() => {
    return (action) => {
      const handlers = registry.get(action);
      for (const reg of handlers) {
        if (activeContexts.has(reg.context)) {
          reg.handler();
          return true; // 找到并执行了处理器
        }
      }
      return false;
    };
  }, [activeContexts, handlerRegistryRef]);
  
  // 4. resolve - 解析按键（委托给 resolver）
  const resolve = useMemo(
    () => (input, key, contexts) => 
      resolveKeyWithChordState(input, key, contexts, bindings, pendingChordRef.current),
    [bindings, pendingChordRef]
  );
  
  // 5. 组装 Context Value
  const value = { resolve, setPendingChord, getDisplayText: getDisplay, 
                  bindings, pendingChord, activeContexts, 
                  registerActiveContext, unregisterActiveContext,
                  registerHandler, invokeAction };
  
  return <KeybindingContext.Provider value={value}>{children}</KeybindingContext.Provider>;
}
```

**React Compiler 优化：**
- 使用 `_c(n)`（React Compiler 的缓存计数器）进行细粒度记忆化
- 每个派生值都有独立的缓存槽位（$[0], $[1], ...）
- 依赖变化时才重新计算

### 2. Handler Registry 数据结构

```typescript
Map<string, Set<HandlerRegistration>>
// 键：action 名称
// 值：该动作的所有注册处理器集合
```

**注册流程：**
1. 检查 action 是否已有 Set，没有则创建
2. 将 registration 添加到 Set
3. 返回注销函数（从 Set 删除，空 Set 时删除整个 entry）

**调用流程（invokeAction）：**
1. 获取 action 对应的所有 handlers
2. 遍历 handlers，检查 registration.context 是否在 activeContexts 中
3. 第一个匹配的 handler 被执行，返回 true
4. 无匹配返回 false

### 3. 上下文优先级系统

**优先级顺序（从高到低）：**
1. 通过 `useRegisterKeybindingContext` 注册的上下文
2. 组件自身的 context（如 useKeybinding 的 options.context）
3. "Global" 上下文

**实现机制：**
- `activeContexts` 是一个 `Set<KeybindingContextName>`
- 解析时传入的 contexts 数组按优先级排序
- resolver 按数组顺序查找匹配，第一个匹配的绑定获胜

---

## 关键代码路径与文件引用

### 核心类型定义（推断自 schema.ts）
```typescript
// 来自 schema.ts / types.js（构建时生成）
type KeybindingContextName = 
  | 'Global' | 'Chat' | 'Autocomplete' | 'Confirmation' | 'Help'
  | 'Transcript' | 'HistorySearch' | 'Task' | 'ThemePicker' | 'Settings'
  | 'Tabs' | 'Attachments' | 'Footer' | 'MessageSelector' | 'DiffDialog'
  | 'ModelPicker' | 'Select' | 'Plugin';

interface ParsedKeystroke {
  key: string;      // 键名，如 'k', 'enter', 'escape'
  ctrl: boolean;
  alt: boolean;
  shift: boolean;
  meta: boolean;    // Alt/Option
  super: boolean;   // Cmd/Win
}

type Chord = ParsedKeystroke[];

interface ParsedBinding {
  chord: Chord;
  action: string | null;  // null 表示解绑
  context: KeybindingContextName;
}
```

### 关键依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../ink.js` | `type Key` | Ink 键盘事件类型 |
| `./resolver.js` | `ChordResolveResult`, `getBindingDisplayText`, `resolveKeyWithChordState` | 按键解析逻辑 |
| `./types.js` | `KeybindingContextName`, `ParsedBinding`, `ParsedKeystroke` | 类型定义 |

### 导出符号

```typescript
export { KeybindingProvider }           // Provider 组件
export { useKeybindingContext }         // 强制上下文 Hook
export { useOptionalKeybindingContext } // 可选上下文 Hook
export { useRegisterKeybindingContext } // 上下文注册 Hook
```

---

## 依赖与外部交互

### 1. 上游依赖（输入）

**来自 KeybindingProviderSetup.tsx：**
- `bindings`: 合并后的默认 + 用户绑定
- `pendingChordRef` / `pendingChord`: 和弦状态（ref + state 双轨）
- `setPendingChord`: 状态更新函数
- `activeContexts`: 活动上下文集合
- `handlerRegistryRef`: 处理器注册表 Ref

**来自 resolver.js：**
- `resolveKeyWithChordState`: 核心解析函数
- `getBindingDisplayText`: 获取显示文本

### 2. 下游消费（输出）

**被 useKeybinding.ts 消费：**
```typescript
const keybindingContext = useOptionalKeybindingContext()
// 使用：resolve(), setPendingChord(), registerHandler()
```

**被 ChordInterceptor（KeybindingProviderSetup.tsx 内）消费：**
```typescript
// 使用：invokeAction(), handlerRegistryRef
```

**被任意组件消费：**
```typescript
useRegisterKeybindingContext('ThemePicker')
```

### 3. 与 Ink 的交互

- 使用 Ink 的 `Key` 类型（来自 `../ink.js`）
- 不直接使用 `useInput`，由下游组件（useKeybinding）处理

---

## 风险、边界与改进建议

### 1. 已知风险

**React Compiler 依赖：**
- 代码使用了 React Compiler 的 `_c()` 函数进行缓存
- 如果编译器版本不兼容，可能导致运行时错误
- 缓存槽位硬编码（$[0] 到 $[23]），扩展时需小心

**Context 性能：**
- `activeContexts` 变化会触发大量重新渲染
- 虽然使用了记忆化，但频繁上下文切换仍可能影响性能

**Handler Registry 内存泄漏：**
- 如果组件卸载时未正确调用注销函数，可能导致内存泄漏
- 依赖 useEffect 的 cleanup 机制

### 2. 边界情况

**空 Context 访问：**
- `useKeybindingContext` 在 Provider 外使用会抛出错误
- `useOptionalKeybindingContext` 返回 null，需调用方处理

**Handler 调用优先级：**
- 同一 action 多个 handler 时，只有第一个匹配上下文的会被执行
- 无明确优先级控制（如注册顺序）

**和弦状态同步：**
- `pendingChordRef` 和 `pendingChord` 双轨机制复杂
- 需要确保两者始终同步

### 3. 改进建议

**代码清晰度：**
```typescript
// 建议：将复杂的 useMemo 提取为独立 hooks
function useResolve(bindings, pendingChordRef) { ... }
function useRegisterHandler(handlerRegistryRef) { ... }
function useInvokeAction(activeContexts, handlerRegistryRef) { ... }
```

**类型安全：**
- `types.js` 文件在源码中不存在（可能是构建时生成）
- 建议明确类型定义位置，或添加类型声明文件

**性能优化：**
- 考虑将 `bindings` 按 context 分组，减少遍历
- 使用 `useCallback` 替代部分 `useMemo` 简化逻辑

**错误处理：**
- `invokeAction` 返回 boolean 表示是否找到 handler
- 建议添加调试日志帮助追踪未触发的绑定

### 4. 测试建议

- 测试 Provider 外使用 `useKeybindingContext` 是否抛出正确错误
- 测试 handler 注册/注销的生命周期
- 测试多个相同 action 不同 context 的 handler 优先级
- 测试 `useRegisterKeybindingContext` 的 `isActive` 参数
