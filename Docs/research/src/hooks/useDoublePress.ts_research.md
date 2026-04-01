# useDoublePress.ts 研究文档

## 场景与职责

`useDoublePress` 是一个通用的 React Hook，用于实现**基于时间窗口的“双击”逻辑**。在终端 TUI 应用中，很多操作不能简单通过单次按键触发（因为容易误触），需要用户在一定时间内连续按两次相同按键才执行最终动作，第一次按键仅给出提示。

该 Hook 被广泛应用于：
- **退出确认**：`useExitOnCtrlCD.ts` 中 Ctrl+C / Ctrl+D 的双击退出
- **输入清空**：`useTextInput.ts` 中 Esc 的双击清空输入、Ctrl+C 的双击退出编辑
- **PromptInput**：某些自定义交互中的双击确认

## 功能点目的

1. **时间窗口内的双击检测**：定义一个超时时间（默认 800ms），若两次调用发生在该窗口内，则触发 `onDoublePress`。

2. **首次按压反馈**：第一次按压时调用 `onFirstPress`（可选），并通过 `setPending(true)` 通知外部进入“待确认”状态。

3. **超时自动重置**：若第二次按压未在超时窗口内发生，自动调用 `setPending(false)` 清除待确认状态。

4. **内存安全**：组件卸载时自动清理 `setTimeout`，避免内存泄漏和闭包 stale 问题。

## 具体技术实现

### 核心逻辑

```ts
export const DOUBLE_PRESS_TIMEOUT_MS = 800

export function useDoublePress(
  setPending: (pending: boolean) => void,
  onDoublePress: () => void,
  onFirstPress?: () => void,
): () => void {
  const lastPressRef = useRef<number>(0)
  const timeoutRef = useRef<NodeJS.Timeout | undefined>(undefined)

  const clearTimeoutSafe = useCallback(() => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current)
      timeoutRef.current = undefined
    }
  }, [])

  useEffect(() => {
    return () => { clearTimeoutSafe() }
  }, [clearTimeoutSafe])

  return useCallback(() => {
    const now = Date.now()
    const timeSinceLastPress = now - lastPressRef.current
    const isDoublePress =
      timeSinceLastPress <= DOUBLE_PRESS_TIMEOUT_MS && timeoutRef.current !== undefined

    if (isDoublePress) {
      clearTimeoutSafe()
      setPending(false)
      onDoublePress()
    } else {
      onFirstPress?.()
      setPending(true)
      clearTimeoutSafe()
      timeoutRef.current = setTimeout(
        (setPending, timeoutRef) => {
          setPending(false)
          timeoutRef.current = undefined
        },
        DOUBLE_PRESS_TIMEOUT_MS,
        setPending,
        timeoutRef,
      )
    }

    lastPressRef.current = now
  }, [setPending, onDoublePress, onFirstPress, clearTimeoutSafe])
}
```

### 设计要点

- **`timeoutRef.current !== undefined` 作为状态标志**：不仅判断时间差，还检查当前是否有一个待处理的 timeout。这防止了在 timeout 已经触发（pending 被重置为 false）后，用户再次按键被误判为双击。

- **`setPending` 和 `timeoutRef` 作为参数传入 callback**：`setTimeout` 的回调显式接收 `setPending` 和 `timeoutRef` 作为参数，而不是依赖闭包。这是一种防御性编程，确保即使闭包中的引用变得 stale，timeout 触发时也能正确操作到最新的状态和 ref。

- **`lastPressRef` 记录时间戳**：使用 `Date.now()` 而非计数器，逻辑直观且不受系统时间微调影响（在 800ms 尺度下可忽略）。

### 使用模式

在 `useExitOnCtrlCD.ts` 中：
```ts
const handleCtrlCDoublePress = useDoublePress(
  pending => setExitState({ pending, keyName: 'Ctrl-C' }),
  exitFn,
)
```

在 `useTextInput.ts` 中：
```ts
const handleCtrlC = useDoublePress(
  show => { onExitMessage?.(show, 'Ctrl-C') },
  () => onExit?.(),
  () => {
    if (originalValue) {
      onChange('')
      setOffset(0)
      onHistoryReset?.()
    }
  },
)
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useDoublePress.ts` | 本 Hook 实现 |
| `src/hooks/useExitOnCtrlCD.ts` | 调用方：Ctrl+C / Ctrl+D 双击退出 |
| `src/hooks/useTextInput.ts` | 调用方：Esc 双击清空、Ctrl+C 双击退出编辑、Ctrl+D 双击退出 |
| `src/components/SessionBackgroundHint.tsx` | 调用方：背景提示中的双击交互 |
| `src/components/PromptInput/PromptInput.tsx` | 调用方：Prompt 输入中的双击行为 |

## 依赖与外部交互

### 内部依赖
- **React**：`useCallback`、`useEffect`、`useRef`
- **Node 类型**：`NodeJS.Timeout`

### 无外部交互
该 Hook 是纯本地状态管理，不涉及网络、文件系统或外部进程。

## 风险、边界与改进建议

### 风险与边界

1. **`setPending` 的调用方需保证幂等**：`setTimeout` 回调和双击成功路径都会调用 `setPending(false)`。如果调用方（如 React `setState`）不处理幂等，理论上不会出问题，但需确保 `setPending` 不会触发副作用。

2. **`onFirstPress` 在每次“单按”时都会触发**：如果用户连续快速按键（如三连击），第一次是 firstPress，第二次是 doublePress，第三次又会变成 firstPress。这在某些场景下可能不符合直觉（用户可能期望第三次也是 doublePress）。

3. **时间窗口固定为 800ms**：`DOUBLE_PRESS_TIMEOUT_MS` 是硬编码常量，无法根据不同按键或用户偏好调整。对于某些用户来说 800ms 可能太短或太长。

4. **`setTimeout` 参数传递的复杂性**：虽然显式传递参数避免了闭包 stale 问题，但也增加了代码阅读难度。且 `timeoutRef` 作为对象引用传入 `setTimeout` 回调是安全的（因为 ref 对象本身不变），但初学者可能困惑。

5. **没有最大按压次数限制**：如果用户以略大于 800ms 的间隔持续按键，会永远处于“first press → timeout → first press”循环，不会触发 double press。

### 改进建议

1. **参数化超时时间**：将 `DOUBLE_PRESS_TIMEOUT_MS` 作为可选参数暴露给调用方，例如：
   ```ts
   export function useDoublePress(
     setPending: (pending: boolean) => void,
     onDoublePress: () => void,
     onFirstPress?: () => void,
     timeoutMs: number = DOUBLE_PRESS_TIMEOUT_MS,
   ): () => void
   ```

2. **支持“连击”模式**：增加一个可选的 `mode` 参数，支持 `n-press`（如三击）模式，或支持在 doublePress 后重置计时器，使连续按压都能触发。

3. **简化 `setTimeout` 回调**：可以使用 `useRef` 保存最新的 `setPending` 回调，然后在 `setTimeout` 中直接读取 ref，避免通过参数传递函数和 ref：
   ```ts
   const setPendingRef = useRef(setPending)
   setPendingRef.current = setPending
   // in setTimeout:
   setPendingRef.current(false)
   ```
   这样代码更简洁，且同样避免 stale closure。

4. **增加按键去重（可选）**：如果未来需要区分不同按键来源的双击（如 Ctrl+C 和 Ctrl+D 不应互相影响），当前每个调用方都独立创建 `useDoublePress` 实例，已经天然隔离。文档中应强调这一点。

5. **单元测试覆盖**：建议增加对以下场景的测试：
   - 单次按键后超时
   - 双击在窗口内触发
   - 组件卸载后 timeout 不触发
   - 连续三次按键的行为
