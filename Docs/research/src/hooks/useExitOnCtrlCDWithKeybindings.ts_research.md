# useExitOnCtrlCDWithKeybindings.ts 研究文档

## 场景与职责

`useExitOnCtrlCDWithKeybindings` 是 `useExitOnCtrlCD` 的**标准便利封装层（convenience hook）**。它的唯一职责是将 `useExitOnCtrlCD` 与项目内实际的键绑定系统 `useKeybindings` 连接起来，避免每个调用方都手动传递 `useKeybindings` 作为注入参数。

由于 `useExitOnCtrlCD.ts` 不直接导入 `src/keybindings/useKeybinding.ts`（为了防止循环依赖），这个封装层提供了一个**即拿即用**的 API，是绝大多数组件处理 `Ctrl+C` / `Ctrl+D` 双击退出的入口。

## 功能点目的

1. **消除依赖注入的样板代码**：调用方无需关心 `useKeybindingsHook` 参数，直接调用 `useExitOnCtrlCDWithKeybindings` 即可。

2. **保持模块解耦**：`useExitOnCtrlCD.ts` 继续不依赖键绑定模块，循环依赖风险仍被隔离在封装层中。

3. **统一退出行为接口**：与 `useExitOnCtrlCD` 保持相同的参数签名（`onExit`、`onInterrupt`、`isActive`）和返回值（`ExitState`）。

## 具体技术实现

### 源码实现

```ts
import { useKeybindings } from '../keybindings/useKeybinding.js'
import { type ExitState, useExitOnCtrlCD } from './useExitOnCtrlCD.js'

export type { ExitState }

export function useExitOnCtrlCDWithKeybindings(
  onExit?: () => void,
  onInterrupt?: () => boolean,
  isActive?: boolean,
): ExitState {
  return useExitOnCtrlCD(useKeybindings, onInterrupt, onExit, isActive)
}
```

### 设计要点

- **参数顺序调整**：注意到 `useExitOnCtrlCD` 的签名是 `(useKeybindingsHook, onInterrupt?, onExit?, isActive?)`，而 `useExitOnCtrlCDWithKeybindings` 的签名是 `(onExit?, onInterrupt?, isActive?)`。这里 `onExit` 和 `onInterrupt` 的顺序被调换了——`onExit` 被放在了前面。

  这种设计可能是因为：
  - 大多数调用方只关心自定义退出行为（`onExit`），而不需要处理中断（`onInterrupt`）
  - 将更常用的参数放在前面，减少传入 `undefined` 占位的情况

- **类型重导出**：`export type { ExitState }` 使得调用方可以直接从本文件导入类型，无需再导入 `useExitOnCtrlCD.ts`。

### 使用模式

在 `Dialog.tsx` 中：
```ts
const { pending, keyName } = useExitOnCtrlCDWithKeybindings(onExit, onInterrupt, isActive)
```

在 `REPL.tsx` 中：
```ts
const exitState = useExitOnCtrlCDWithKeybindings(undefined, onInterrupt)
```

在各类命令/设置组件中（如 `Settings.tsx`、`ModelPicker.tsx`、`TrustDialog.tsx` 等）：
```ts
useExitOnCtrlCDWithKeybindings()
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useExitOnCtrlCDWithKeybindings.ts` | 本封装层 |
| `src/hooks/useExitOnCtrlCD.ts` | 底层双击退出逻辑 |
| `src/keybindings/useKeybinding.ts` | 注入的 `useKeybindings` |
| `src/screens/REPL.tsx` | 调用方：主屏幕退出处理 |
| `src/components/design-system/Dialog.tsx` | 调用方：通用对话框退出处理 |
| `src/components/Settings/Settings.tsx` | 调用方：设置页面 |
| `src/components/ModelPicker.tsx` | 调用方：模型选择器 |
| `src/components/TrustDialog/TrustDialog.tsx` | 调用方：信任对话框 |
| `src/components/Onboarding.tsx` | 调用方： onboarding 流程 |
| `src/components/HelpV2/HelpV2.tsx` | 调用方：帮助页面 |
| `src/commands/plugin/PluginSettings.tsx` | 调用方：插件设置 |

## 依赖与外部交互

### 内部依赖
- **React Hook**：`useExitOnCtrlCD`
- **键绑定系统**：`useKeybindings`

### 无外部交互
该文件是纯封装层，不涉及网络、文件系统或外部进程。

## 风险、边界与改进建议

### 风险与边界

1. **参数顺序与底层不一致**：`useExitOnCtrlCDWithKeybindings(onExit?, onInterrupt?, isActive?)` 与 `useExitOnCtrlCD(useKeybindingsHook, onInterrupt?, onExit?, isActive?)` 中 `onExit` 和 `onInterrupt` 的顺序相反。这可能导致：
   - 开发者如果同时接触两个 API，容易混淆参数顺序
   - 某些调用方可能错误地将中断处理函数传给了 `onExit` 位置

2. **无额外功能增值**：该文件除了转发调用外没有任何额外逻辑。如果只是为了省一行代码，其价值有限；但从架构角度看，它确实承担了"连接两个模块"的职责。

3. **循环依赖风险仍在**：虽然 `useExitOnCtrlCD.ts` 本身不依赖键绑定模块，但 `useExitOnCtrlCDWithKeybindings.ts` 同时依赖两者。如果未来 `useKeybindings` 或其依赖链反向依赖了 `useExitOnCtrlCDWithKeybindings`，循环依赖仍然会发生。

4. **调用方泛滥**：从 grep 结果看，有数十个组件直接调用 `useExitOnCtrlCDWithKeybindings`。如果未来需要统一修改退出行为（如增加全局退出确认），需要修改的入口点很多。

### 改进建议

1. **统一参数顺序**：建议将 `useExitOnCtrlCDWithKeybindings` 的参数顺序与底层 `useExitOnCtrlCD` 对齐，即 `(onInterrupt?, onExit?, isActive?)`。虽然这会是一次 breaking change，但能显著降低心智负担。如果担心 breaking change，可以改为接收一个 options 对象：
   ```ts
   export function useExitOnCtrlCDWithKeybindings(options?: {
     onExit?: () => void
     onInterrupt?: () => boolean
     isActive?: boolean
   }): ExitState
   ```

2. **考虑合并为一个 Hook**：如果循环依赖问题可以通过其他方式解决（如将键绑定类型定义提取到更底层的包），可以直接让 `useExitOnCtrlCD` 内部导入 `useKeybindings`，消除这个中间层。

3. **增加全局退出守卫**：可以在封装层中增加一个可选的全局退出前检查（如是否有未保存的变更、是否有运行中的任务），统一处理所有调用方的退出确认逻辑，而不是让每个调用方自己实现。

4. **文档化使用规范**：由于调用方众多，建议在代码注释或 AGENTS.md 中明确说明：
   - 何时应该使用 `useExitOnCtrlCDWithKeybindings`
   - 何时应该直接使用 `useExitOnCtrlCD`
   - `isActive` 的正确设置方式（特别是包含 `TextInput` 的组件）
