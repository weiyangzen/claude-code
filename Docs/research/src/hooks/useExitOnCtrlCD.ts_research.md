# useExitOnCtrlCD.ts 研究文档

## 场景与职责

`useExitOnCtrlCD` 是 Claude Code TUI 中负责处理**应用退出快捷键**的核心 Hook。它实现了基于时间窗口的“双击”确认机制：
- 用户第一次按 `Ctrl+C` 或 `Ctrl+D` 时，显示提示信息（如 "Press Ctrl-C again to exit"）
- 如果在超时窗口内再次按下相同按键，则执行退出操作

该 Hook 被设计为**与具体的键绑定系统解耦**：它接收一个 `useKeybindingsHook` 注入参数，而不是直接导入 `useKeybindings`，这是为了避免 `useExitOnCtrlCD.ts` 与 `src/keybindings/` 模块之间产生循环依赖。

## 功能点目的

1. **双击确认退出**：防止用户误触 `Ctrl+C`（通常用于复制或中断）或 `Ctrl+D`（EOF）导致应用意外退出。

2. **支持中断回调（`onInterrupt`）**：`Ctrl+C` 首先尝试调用 `onInterrupt` 回调。如果回调返回 `true`，表示上层组件已处理该中断（如取消当前请求），不再进入双击退出逻辑；返回 `false` 才继续走双击退出流程。

3. **支持自定义退出行为（`onExit`）**：允许调用方传入自定义的退出处理函数，默认使用 Ink 的 `exit()`（`useApp`）。

4. **支持激活/失活控制（`isActive`）**：当嵌入式 `TextInput` 获得焦点时，可以将 `isActive` 设为 `false`，让 `TextInput` 自己处理 `Ctrl+C`/`Ctrl+D`，避免事件被父级 Dialog 重复处理。

## 具体技术实现

### 核心逻辑

```ts
export function useExitOnCtrlCD(
  useKeybindingsHook: UseKeybindingsHook,
  onInterrupt?: () => boolean,
  onExit?: () => void,
  isActive = true,
): ExitState {
  const { exit } = useApp()
  const [exitState, setExitState] = useState<ExitState>({ pending: false, keyName: null })

  const exitFn = useMemo(() => onExit ?? exit, [onExit, exit])

  // Double-press handler for ctrl+c
  const handleCtrlCDoublePress = useDoublePress(
    pending => setExitState({ pending, keyName: 'Ctrl-C' }),
    exitFn,
  )

  // Double-press handler for ctrl+d
  const handleCtrlDDoublePress = useDoublePress(
    pending => setExitState({ pending, keyName: 'Ctrl-D' }),
    exitFn,
  )

  const handleInterrupt = useCallback(() => {
    if (onInterrupt?.()) return // Feature handled it
    handleCtrlCDoublePress()
  }, [handleCtrlCDoublePress, onInterrupt])

  const handleExit = useCallback(() => {
    handleCtrlDDoublePress()
  }, [handleCtrlDDoublePress])

  const handlers = useMemo(
    () => ({
      'app:interrupt': handleInterrupt,
      'app:exit': handleExit,
    }),
    [handleInterrupt, handleExit],
  )

  useKeybindingsHook(handlers, { context: 'Global', isActive })

  return exitState
}
```

### 设计要点

- **依赖注入 `useKeybindingsHook`**：
  类型定义为：
  ```ts
  type UseKeybindingsHook = (
    handlers: Record<string, () => void>,
    options?: KeybindingOptions,
  ) => void
  ```
  这种注入方式使得 `useExitOnCtrlCD` 不直接依赖 `src/keybindings/useKeybinding.ts`，从而打破潜在的 import cycle。

- **`useDoublePress` 复用**：
  `Ctrl+C` 和 `Ctrl+D` 各自创建一个 `useDoublePress` 实例，独立维护 pending 状态和超时计时器。

- **`onInterrupt` 的优先级**：
  `app:interrupt` 绑定到 `handleInterrupt`，它先尝试 `onInterrupt?.()`。这允许 `REPL.tsx` 等调用方在 `Ctrl+C` 时优先取消当前查询，而不是退出应用。

- **`ExitState` 返回**：
  ```ts
  export type ExitState = {
    pending: boolean
    keyName: 'Ctrl-C' | 'Ctrl-D' | null
  }
  ```
  调用方可以根据 `exitState` 渲染提示文本（如 "Press Ctrl-C again to exit"）。

### 键绑定映射

在 Claude Code 的键绑定系统中：
- `app:interrupt` 默认映射到 `ctrl+c`
- `app:exit` 默认映射到 `ctrl+d`

这些映射定义在键绑定配置中（`src/keybindings/` 相关配置），`useExitOnCtrlCD` 只关心 action 名称，不关心具体按键。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useExitOnCtrlCD.ts` | 本 Hook 实现 |
| `src/hooks/useExitOnCtrlCDWithKeybindings.ts` | 标准封装层，注入 `useKeybindings` |
| `src/hooks/useDoublePress.ts` | 双击逻辑复用 |
| `src/ink/hooks/use-app.ts` | `useApp`，提供 Ink 的 `exit()` |
| `src/keybindings/useKeybinding.ts` | `useKeybindings`，实际的键绑定注册实现 |
| `src/keybindings/types.ts` | `KeybindingContextName` 类型 |
| `src/screens/REPL.tsx` | 通过 `useExitOnCtrlCDWithKeybindings` 使用 |
| `src/components/design-system/Dialog.tsx` | 大量 Dialog 组件使用 `useExitOnCtrlCDWithKeybindings` |

## 依赖与外部交互

### 内部依赖
- **React**：`useCallback`、`useMemo`、`useState`
- **Ink**：`useApp` 获取退出函数
- **自定义 Hook**：`useDoublePress`

### 外部交互
无直接外部交互。退出行为最终通过 Ink 的 `exit()` 或调用方自定义的 `onExit` 实现，可能触发进程退出。

## 风险、边界与改进建议

### 风险与边界

1. **`onInterrupt` 与 `handleCtrlCDoublePress` 的竞态**：`onInterrupt` 是同步调用的，如果它返回 `true`，`handleCtrlCDoublePress` 不会执行。但如果 `onInterrupt` 内部有异步操作（如发送取消请求），它仍然返回 `true`，此时用户可能看不到任何反馈，因为 `exitState.pending` 没有被设置。

2. **`isActive` 的传递一致性**：注释中提到 "Set false while an embedded TextInput is focused"，但实际使用中，很多 Dialog 组件并没有正确传递 `isActive` 给 `useExitOnCtrlCDWithKeybindings`，可能导致 `Ctrl+C` 在 TextInput 中同时触发输入框的复制/取消和 Dialog 的退出提示。

3. **硬编码的 action 名称**：`'app:interrupt'` 和 `'app:exit'` 是硬编码字符串。如果键绑定系统中的 action 名称变更，这里会静默失效。

4. **`useKeybindingsHook` 的类型限制**：注入的 Hook 类型要求 handler 返回 `void`，而 `onInterrupt` 返回 `boolean`。这没问题，因为 `handleInterrupt` 包装了 `onInterrupt` 的返回值。但如果未来需要支持 Promise 类型的 handler，类型定义需要更新。

5. **没有长按/连击保护**：如果用户以极快速度连续按三次 `Ctrl+C`，行为是：第一次触发 `onInterrupt`（如果存在），第二次开始 firstPress，第三次触发 doublePress 退出。这在某些场景下可能过于敏感。

### 改进建议

1. **统一 `isActive` 管理**：建议为所有包含 `TextInput` 的 Dialog 提供一个高阶组件或 Hook 封装，自动检测 `TextInput` 焦点状态并设置 `isActive = false`，减少遗漏。

2. **支持 `onInterrupt` 的异步反馈**：如果 `onInterrupt` 执行了异步取消操作，可以考虑让 `useExitOnCtrlCD` 在 `onInterrupt` 返回 `true` 时短暂设置 `exitState = { pending: true, keyName: 'Ctrl-C' }` 并显示 "Cancelling..."，提升用户体验。

3. **Action 名称常量化**：将 `'app:interrupt'` 和 `'app:exit'` 提取为常量（如 `ACTION_INTERRUPT = 'app:interrupt'`），放在 `src/keybindings/constants.ts` 中，避免魔法字符串。

4. **增加退出前确认回调**：某些场景下（如有未保存的编辑、后台任务运行中），退出前需要额外确认。可以增加一个可选的 `onBeforeExit` 回调，返回 `Promise<boolean>`，在真正调用 `exitFn` 前执行：
   ```ts
   onBeforeExit?: () => boolean | Promise<boolean>
   ```

5. **与 `useTextInput.ts` 的 Esc 双击统一**：`useTextInput.ts` 中也有独立的 `useDoublePress` 处理 Esc 清空输入。可以考虑将这些终端编辑行为抽象为统一的 "double-press action" 模式，便于维护和扩展。
