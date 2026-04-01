# use-terminal-focus.ts 深入研究

## 场景与职责

`useTerminalFocus` 是 Ink 终端 UI 框架中用于检测终端焦点状态的 Hook。它利用 DECSET 1004 焦点报告功能，让终端在获得或失去焦点时发送转义序列，从而实现对终端焦点状态的响应式跟踪。

## 功能点目的

### 1. 焦点状态检测
- 检测终端窗口是否获得焦点
- 支持终端焦点报告（DECSET 1004）
- 自动过滤焦点事件，不传递给 `useInput`

### 2. 响应式更新
- 焦点变化时自动重新渲染组件
- 使用 React Context 实现状态共享
- 避免不必要的重渲染

### 3. 无障碍支持
- 为需要焦点感知的组件提供状态
- 支持基于焦点的 UI 调整（如暂停动画）

## 具体技术实现

### 代码实现

```typescript
import { useContext } from 'react'
import TerminalFocusContext from '../components/TerminalFocusContext.js'

export function useTerminalFocus(): boolean {
  const { isTerminalFocused } = useContext(TerminalFocusContext)
  return isTerminalFocused
}
```

### 返回类型

```typescript
boolean  // true 表示终端有焦点（或焦点状态未知）
```

### TerminalFocusContext 定义

```typescript
// src/ink/components/TerminalFocusContext.tsx
export type TerminalFocusContextProps = {
  readonly isTerminalFocused: boolean
  readonly terminalFocusState: TerminalFocusState
}

export type TerminalFocusState = 'focused' | 'blurred' | 'unknown'

const TerminalFocusContext = createContext<TerminalFocusContextProps>({
  isTerminalFocused: true,
  terminalFocusState: 'unknown'
})
```

### TerminalFocusProvider 实现

```typescript
export function TerminalFocusProvider({ children }) {
  const isTerminalFocused = useSyncExternalStore(
    subscribeTerminalFocus,
    getTerminalFocused
  )
  const terminalFocusState = useSyncExternalStore(
    subscribeTerminalFocus,
    getTerminalFocusState
  )
  
  const value = useMemo(() => ({
    isTerminalFocused,
    terminalFocusState
  }), [isTerminalFocused, terminalFocusState])
  
  return (
    <TerminalFocusContext.Provider value={value}>
      {children}
    </TerminalFocusContext.Provider>
  )
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/TerminalFocusContext.tsx` | 提供焦点状态上下文 |
| `src/ink/terminal-focus-state.ts` | 焦点状态管理和订阅 |

### 焦点状态管理

```typescript
// src/ink/terminal-focus-state.ts（推断）

let focusState: TerminalFocusState = 'unknown'
const listeners = new Set<() => void>()

export function subscribeTerminalFocus(callback: () => void): () => void {
  listeners.add(callback)
  return () => listeners.delete(callback)
}

export function getTerminalFocused(): boolean {
  return focusState === 'focused' || focusState === 'unknown'
}

export function getTerminalFocusState(): TerminalFocusState {
  return focusState
}

// 内部：处理焦点事件
function handleFocusEvent(focused: boolean) {
  focusState = focused ? 'focused' : 'blurred'
  listeners.forEach(cb => cb())
}
```

### DECSET 1004 焦点报告

终端启用焦点报告：
```
ESC [ ? 1004 h  // 启用焦点事件
ESC [ ? 1004 l  // 禁用焦点事件
```

焦点事件序列：
```
ESC [ I  // 终端获得焦点
ESC [ O  // 终端失去焦点
```

### 使用示例

```typescript
import { useTerminalFocus } from 'ink'

const FocusAwareComponent = () => {
  const isFocused = useTerminalFocus()
  
  return (
    <Text color={isFocused ? 'green' : 'gray'}>
      {isFocused ? 'Terminal is focused' : 'Terminal is blurred'}
    </Text>
  )
}
```

### 在 ClockContext 中的使用

```typescript
// src/ink/components/ClockContext.tsx
export function ClockProvider({ children }) {
  const [clock] = useState(() => createClock(FRAME_INTERVAL_MS))
  const focused = useTerminalFocus()
  
  useEffect(() => {
    // 失焦时降低时钟频率
    clock.setTickInterval(focused ? FRAME_INTERVAL_MS : BLURRED_TICK_INTERVAL_MS)
  }, [clock, focused])
  
  return (
    <ClockContext.Provider value={clock}>
      {children}
    </ClockContext.Provider>
  )
}
```

## 依赖与外部交互

### 与终端的交互

1. **启用焦点报告**：
   - Ink 主类在初始化时发送 DECSET 1004 h
   - 启用后终端发送焦点事件

2. **事件处理**：
   - 焦点事件由 Ink 的输入处理层拦截
   - 更新内部焦点状态
   - 不传递给 `useInput`（自动过滤）

3. **禁用焦点报告**：
   - 应用退出时发送 DECSET 1004 l
   - 避免影响终端的后续使用

### 与 useSyncExternalStore 的交互

```typescript
const isTerminalFocused = useSyncExternalStore(
  subscribeTerminalFocus,   // 订阅函数
  getTerminalFocused        // 获取快照
)
```

- 使用 React 18 的 `useSyncExternalStore` API
- 确保服务端渲染兼容性
- 防止 tearing 问题

### 与 ClockContext 的协作

- `ClockProvider` 使用 `useTerminalFocus` 监听焦点变化
- 失焦时自动降低动画时钟频率（16ms → 32ms）
- 节省资源，减少后台 CPU 使用

## 风险、边界与改进建议

### 潜在风险

1. **终端不支持**：
   - 旧终端可能不支持 DECSET 1004
   - 焦点状态将保持 'unknown'

2. **SSH/Tmux 会话**：
   - 焦点事件可能在多层终端间传递
   - 可能出现意外的焦点状态

3. **快速切换**：
   - 快速焦点切换可能产生事件堆积
   - 可能导致不必要的重渲染

### 边界情况

1. **焦点状态未知**：
   - 初始状态为 'unknown'
   - `isTerminalFocused` 返回 true（保守策略）

2. **Provider 外部使用**：
   - 返回默认值 `{ isTerminalFocused: true, terminalFocusState: 'unknown' }`
   - 不会抛出错误

3. **非 TTY 环境**：
   - 焦点报告不可用
   - 状态保持 'unknown'

### 改进建议

1. **添加焦点变化回调**：
   ```typescript
   useTerminalFocus({
     onFocus: () => console.log('Focused'),
     onBlur: () => console.log('Blurred')
   })
   ```

2. **暴露完整状态**：
   ```typescript
   const { isFocused, state } = useTerminalFocusDetailed()
   // state: 'focused' | 'blurred' | 'unknown'
   ```

3. **添加防抖**：
   ```typescript
   // 防止快速切换导致的闪烁
   const isFocused = useDebounce(useTerminalFocus(), 100)
   ```

4. **支持强制值**：
   ```typescript
   // 测试或特殊场景
   useTerminalFocus({ forceValue: true })
   ```

5. **添加焦点历史**：
   ```typescript
   const { isFocused, focusHistory } = useTerminalFocus({ history: true })
   // focusHistory: Array<{ state: boolean, timestamp: number }>
   ```

### 测试建议

1. 测试终端获得/失去焦点
2. 测试不支持的终端（应返回 unknown）
3. 测试 Provider 外部使用
4. 测试与 ClockContext 的集成
5. 测试快速焦点切换
6. 测试 SSH/tmux 环境
