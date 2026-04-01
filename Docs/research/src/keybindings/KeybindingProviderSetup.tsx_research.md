# KeybindingProviderSetup.tsx 研究文档

## 场景与职责

`KeybindingProviderSetup.tsx` 是 Claude Code 键盘快捷键系统的顶层集成模块，负责将键盘绑定系统初始化并注入到应用的组件树中。它是连接配置加载、状态管理和事件处理的 orchestrator（编排器）。

**核心职责：**
1. **配置加载**：加载默认绑定和用户自定义绑定（`~/.claude/keybindings.json`）
2. **热重载支持**：监听配置文件变化，自动重新加载
3. **和弦状态管理**：管理多键组合（chord）的输入状态（如 `ctrl+x ctrl+k`）
4. **上下文跟踪**：维护活动上下文集合，支持上下文优先级
5. **警告通知**：将绑定配置错误通过通知系统展示给用户
6. **全局拦截**：通过 `ChordInterceptor` 组件拦截所有按键，处理和弦序列

**架构位置：**
```
AppStateProvider
    ↓
KeybindingSetup (本文件)
    ├── 加载绑定配置（默认 + 用户）
    ├── 初始化和弦状态管理
    ├── 设置文件监视（热重载）
    ├── 注册警告通知
    ↓
KeybindingProvider (Context)
    ↓
ChordInterceptor (按键拦截)
    ↓
应用子组件
```

---

## 功能点目的

### 1. KeybindingSetup 组件

**主要功能：**
- 同步加载初始绑定（`loadKeybindingsSyncWithWarnings`）
- 设置和弦状态（`pendingChordRef` + `pendingChord` state）
- 管理活动上下文（`activeContextsRef`）
- 初始化文件监视器（`initializeKeybindingWatcher`）
- 订阅配置变更（`subscribeToKeybindingChanges`）

**状态管理：**
```typescript
const [loadResult, setLoadResult] = useState<KeybindingsLoadResult>(...)
const [isReload, setIsReload] = useState(false)
const pendingChordRef = useRef<ParsedKeystroke[] | null>(null)
const [pendingChord, setPendingChordState] = useState<ParsedKeystroke[] | null>(null)
const chordTimeoutRef = useRef<NodeJS.Timeout | null>(null)
const handlerRegistryRef = useRef(new Map<string, Set<HandlerRegistration>>())
const activeContextsRef = useRef<Set<KeybindingContextName>>(new Set())
```

### 2. useKeybindingWarnings Hook

**用途：** 将绑定配置的警告/错误通过通知系统展示给用户。

**行为：**
- 错误和警告计数统计
- 生成提示消息（如 "Found 2 keybinding errors and 1 warning · /doctor for details"）
- 通过 `addNotification` 显示通知
- 60秒超时，错误用红色，警告用黄色

### 3. ChordInterceptor 组件

**核心职责：** 全局按键拦截器，处理和弦序列。

**为什么需要它：**
- 必须在子组件之前注册 `useInput`，确保优先处理
- 防止和弦的第二键（如 `ctrl+c r` 中的 `r`）被 PromptInput 捕获为文本输入

**处理流程：**
1. 收集所有 handler 注册的上下文 + 活动上下文 + Global
2. 调用 `resolveKeyWithChordState` 解析按键
3. 根据结果类型处理：
   - `chord_started`: 设置等待状态，阻止事件传播
   - `match`: 完成和弦，调用 handler，阻止传播
   - `chord_cancelled`: 清除等待状态，阻止传播
   - `unbound`: 清除等待状态，阻止传播
   - `none`: 不处理，允许其他 handler 尝试

### 4. 和弦超时机制

**配置：**
```typescript
const CHORD_TIMEOUT_MS = 1000  // 1秒超时
```

**行为：**
- 用户开始和弦后启动定时器
- 1秒内未完成，自动取消和弦状态
- 超时后 `pendingChordRef` 和 `pendingChord` 都设为 null

---

## 具体技术实现

### 1. 双轨状态管理（Ref + State）

**为什么需要双轨：**
```typescript
const pendingChordRef = useRef<ParsedKeystroke[] | null>(null)  // 同步访问
const [pendingChord, setPendingChordState] = useState<ParsedKeystroke[] | null>(null)  // 触发渲染
```

**使用场景：**
- **Ref**: 在 `resolve()` 中需要立即获取当前值（输入处理不能等待渲染周期）
- **State**: 触发重新渲染以更新 UI（如显示当前和弦状态）

**同步机制：**
```typescript
const setPendingChord = useCallback((pending: ParsedKeystroke[] | null) => {
  clearChordTimeout()
  if (pending !== null) {
    // 设置超时定时器
    chordTimeoutRef.current = setTimeout(..., CHORD_TIMEOUT_MS)
  }
  pendingChordRef.current = pending  // 立即更新 ref
  setPendingChordState(pending)       // 触发渲染
}, [clearChordTimeout])
```

### 2. 活动上下文管理

**注册/注销：**
```typescript
const registerActiveContext = useCallback((context: KeybindingContextName) => {
  activeContextsRef.current.add(context)
}, [])

const unregisterActiveContext = useCallback((context: KeybindingContextName) => {
  activeContextsRef.current.delete(context)
}, [])
```

**为什么用 Ref 而非 State：**
- 输入处理需要同步访问当前活动上下文
- 不能等待 React 渲染周期

### 3. 热重载实现

**初始化流程：**
```typescript
useEffect(() => {
  // 1. 初始化文件监视器（幂等）
  void initializeKeybindingWatcher()
  
  // 2. 订阅配置变更
  const unsubscribe = subscribeToKeybindingChanges(result => {
    setIsReload(true)
    setLoadResult(result)
    logForDebugging(`[keybindings] Reloaded: ...`)
  })
  
  return () => {
    unsubscribe()
    clearChordTimeout()
  }
}, [clearChordTimeout])
```

**关键特性：**
- `initializeKeybindingWatcher` 是幂等的（通过 `initialized` 标志）
- 只有外部用户（Anthropic 员工）启用自定义绑定时才启用监视
- 变更时自动更新绑定和警告

### 4. ChordInterceptor 详细逻辑

```typescript
function ChordInterceptor({ bindings, pendingChordRef, setPendingChord, 
                            activeContexts, handlerRegistryRef }) {
  const handleInput = useMemo(() => (input, key, event) => {
    // 1. 滚轮事件且不在和弦中，直接返回
    if ((key.wheelUp || key.wheelDown) && pendingChordRef.current === null) return
    
    // 2. 收集所有相关上下文
    const registry = handlerRegistryRef.current
    const handlerContexts = new Set()
    for (const handlers of registry.values()) {
      for (const reg of handlers) {
        handlerContexts.add(reg.context)
      }
    }
    const contexts = [...handlerContexts, ...activeContexts, "Global"]
    
    // 3. 记录是否已在和弦中
    const wasInChord = pendingChordRef.current !== null
    
    // 4. 解析按键
    const result = resolveKeyWithChordState(input, key, contexts, bindings, pendingChordRef.current)
    
    // 5. 根据结果处理
    switch (result.type) {
      case "chord_started":
        setPendingChord(result.pending)
        event.stopImmediatePropagation()
        break
      case "match":
        setPendingChord(null)
        if (wasInChord) {
          // 在和弦中完成的匹配，直接调用 handler
          const contextsSet = new Set(contexts)
          const handlers = registry.get(result.action)
          for (const reg of handlers) {
            if (contextsSet.has(reg.context)) {
              reg.handler()
              event.stopImmediatePropagation()
              break
            }
          }
        }
        break
      case "chord_cancelled":
      case "unbound":
        setPendingChord(null)
        event.stopImmediatePropagation()
        break
      case "none":
        // 不处理，允许其他 handler 尝试
    }
  }, [bindings, pendingChordRef, ...])
  
  useInput(handleInput)
  return null
}
```

**关键设计：**
- 使用 `stopImmediatePropagation()` 阻止事件继续传播
- 只有在 `wasInChord` 为 true 时才直接调用 handler（单键绑定由 useKeybinding 处理）
- 滚轮事件特殊处理，避免干扰滚动

---

## 关键代码路径与文件引用

### 核心常量
```typescript
const CHORD_TIMEOUT_MS = 1000  // 和弦超时时间（毫秒）
```

### 关键依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../context/notifications.js` | `useNotifications` | 显示绑定警告通知 |
| `../ink.js` | `Key`, `useInput` | 键盘事件处理 |
| `../ink/events/input-event.js` | `InputEvent` | 输入事件类型 |
| `../utils/array.js` | `count` | 统计错误/警告数量 |
| `../utils/debug.js` | `logForDebugging` | 调试日志 |
| `../utils/stringUtils.js` | `plural` | 复数形式处理 |
| `./KeybindingContext.js` | `KeybindingProvider` | Context Provider |
| `./loadUserBindings.js` | `initializeKeybindingWatcher`, `loadKeybindingsSyncWithWarnings`, `subscribeToKeybindingChanges` | 绑定加载和监视 |
| `./resolver.js` | `resolveKeyWithChordState` | 按键解析 |
| `./types.js` | `KeybindingContextName`, `ParsedBinding`, `ParsedKeystroke` | 类型定义 |
| `./validate.js` | `KeybindingWarning` | 警告类型 |

### 类型定义

```typescript
type Props = {
  children: React.ReactNode
}

type HandlerRegistration = {
  action: string
  context: KeybindingContextName
  handler: () => void
}
```

### 导出符号

```typescript
export { KeybindingSetup }  // 主组件
```

---

## 依赖与外部交互

### 1. 上游依赖（输入）

**来自 loadUserBindings.ts：**
- `loadKeybindingsSyncWithWarnings()`: 同步加载绑定和警告
- `initializeKeybindingWatcher()`: 初始化文件监视
- `subscribeToKeybindingChanges()`: 订阅配置变更

**来自 notifications context：**
- `addNotification()`: 添加警告通知
- `removeNotification()`: 移除通知

**来自 resolver.ts：**
- `resolveKeyWithChordState()`: 核心解析逻辑

### 2. 下游输出（消费）

**传递给 KeybindingProvider：**
- `bindings`: 解析后的绑定
- `pendingChordRef` / `pendingChord`: 和弦状态
- `setPendingChord`: 状态更新函数
- `activeContexts`: 活动上下文
- `registerActiveContext` / `unregisterActiveContext`: 上下文管理
- `handlerRegistryRef`: 处理器注册表

**ChordInterceptor 消费：**
- 使用 `useInput` 拦截所有按键
- 调用 `resolveKeyWithChordState` 解析
- 通过 `handlerRegistryRef` 调用匹配的 handler

### 3. 生命周期管理

**挂载时：**
1. 同步加载绑定（`loadKeybindingsSyncWithWarnings`）
2. 初始化文件监视器
3. 订阅配置变更
4. 显示任何加载警告

**配置变更时：**
1. 调用 `setIsReload(true)`
2. 更新 `loadResult`
3. 触发重新渲染，新绑定生效

**卸载时：**
1. 取消订阅
2. 清除和弦超时定时器

---

## 风险、边界与改进建议

### 1. 已知风险

**ChordInterceptor 顺序依赖：**
- 必须在组件树中位于其他使用 `useInput` 的组件之前
- 如果顺序错误，和弦的第二键会被其他组件捕获
- 当前通过 JSX 结构保证顺序（Interceptor 在 children 之前）

**Ref 与 State 同步风险：**
- `pendingChordRef` 和 `pendingChord` 必须始终保持同步
- 任何直接修改 ref 而不调用 `setPendingChord` 的代码都会导致不一致

**内存泄漏：**
- `handlerRegistryRef` 中的 handlers 需要正确注销
- 依赖组件卸载时调用 `registerHandler` 返回的注销函数

**定时器泄漏：**
- 组件卸载时通过 cleanup 函数清除超时
- 但如果在超时回调执行前快速挂载/卸载，可能有竞态条件

### 2. 边界情况

**空绑定：**
- 用户删除 `keybindings.json` 后，使用默认绑定
- 文件不存在时静默回退

**无效配置：**
- JSON 解析错误时显示错误通知
- 继续使用之前的有效绑定

**和弦中断：**
- 用户按 Escape 取消和弦
- 超时自动取消
- 按非和弦键取消

**多上下文冲突：**
- 同一 action 在多个上下文中有 handler 时，优先级可能不明确
- 依赖 contexts 数组的顺序

### 3. 改进建议

**代码组织：**
```typescript
// 建议：将 ChordInterceptor 提取为独立文件
// 当前内联在 KeybindingProviderSetup.tsx 中，文件较大（~400行）
```

**类型安全：**
- `activeContextsRef.current` 直接传递，类型为 `Set<KeybindingContextName>`
- 但 ProviderProps 中 `activeContexts` 也是同样的类型
- 建议明确区分 Ref 和 State 的使用场景

**性能优化：**
```typescript
// 当前每次渲染都重建 contexts 数组
const contexts = [...handlerContexts, ...activeContexts, "Global"]

// 建议：缓存 handlerContexts，只在注册变化时更新
```

**错误处理：**
- `initializeKeybindingWatcher()` 返回的 Promise 被忽略（`void`）
- 建议添加错误处理，监视器初始化失败时通知用户

**调试支持：**
- 当前只有 `logForDebugging` 输出
- 建议添加更详细的和弦状态追踪，便于调试用户问题

### 4. 测试建议

- 测试和弦超时机制（1秒内完成 vs 超时）
- 测试热重载（修改 keybindings.json 后自动生效）
- 测试多上下文优先级（注册上下文 vs Global）
- 测试 ChordInterceptor 顺序（确保在 children 之前）
- 测试内存泄漏（大量注册/注销 handler）
- 测试无效配置处理（错误通知显示）

### 5. 已知问题

**ESLint 例外：**
```typescript
// eslint-disable-next-line custom-rules/prefer-use-keybindings
import { type Key, useInput } from '../ink.js'
```
- ChordInterceptor 必须使用 `useInput` 而非 `useKeybinding`
- 这是设计需求，不是代码异味
