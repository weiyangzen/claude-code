# 研究文档：src/commands/help/help.tsx

## 场景与职责

`src/commands/help/help.tsx` 是 Claude Code 中 `/help` 命令的**实际执行体**。当用户在终端输入 `/help` 或在 PromptInput 中按 `?` 触发帮助时，命令系统通过懒加载动态导入本模块，并调用其导出的 `call` 函数。该函数的职责极为纯粹：

- 接收命令完成回调 `onDone`（在 `LocalJSXCommandCall` 中被称为 `onDone`，在 UI 层被当作 `onClose` 使用）；
- 从 `context.options` 中解构出当前会话可用的 `commands` 列表；
- 返回一个 React 元素 `<HelpV2 commands={commands} onClose={onDone} />`，由 REPL 通过 Ink 渲染到终端界面中。

本文件是**命令实现层**与**组件表现层**之间的薄胶合层（thin glue layer），本身不包含任何状态管理、副作用或交互逻辑。

## 功能点目的

1. **桥接命令协议与 React 组件**：将 `LocalJSXCommandCall` 的函数式调用约定（`async (onDone, context, args) => React.ReactNode`）转换为 `HelpV2` 的组件 props 约定。
2. **注入命令上下文**：把当前会话经过过滤后的完整 `commands` 数组传递给 `HelpV2`，使帮助弹窗能够实时展示内置命令、自定义技能、插件命令等。
3. **统一关闭语义**：将 `onDone` 直接透传为 `HelpV2` 的 `onClose`，确保用户通过 ESC、选择命令、或其他方式关闭帮助弹窗时，正确通知命令执行器完成本次 `/help` 调用。

## 具体技术实现

### 关键代码

```tsx
import * as React from 'react';
import { HelpV2 } from '../../components/HelpV2/HelpV2.js';
import type { LocalJSXCommandCall } from '../../types/command.js';

export const call: LocalJSXCommandCall = async (onDone, {
  options: {
    commands
  }
}) => {
  return <HelpV2 commands={commands} onClose={onDone} />;
}
```

### 类型系统

- `LocalJSXCommandCall`（定义于 `src/types/command.ts` 第 131–135 行）：
  ```ts
  export type LocalJSXCommandCall = (
    onDone: LocalJSXCommandOnDone,
    context: ToolUseContext & LocalJSXCommandContext,
    args: string,
  ) => Promise<React.ReactNode>
  ```
  本文件的 `call` 函数只使用了 `onDone` 和 `context.options.commands`，忽略了 `args`（`/help` 目前不接受参数）。

- `LocalJSXCommandOnDone`（`src/types/command.ts` 第 117–126 行）：
  ```ts
  export type LocalJSXCommandOnDone = (
    result?: string,
    options?: {
      display?: CommandResultDisplay
      shouldQuery?: boolean
      metaMessages?: string[]
      nextInput?: string
      submitNextInput?: boolean
    },
  ) => void
  ```
  `HelpV2` 内部会将 `onClose` 包装为 `close = () => onClose("Help dialog dismissed", { display: "system" })`，从而向命令系统报告：帮助弹窗已被关闭，结果以 `system` 级别显示（用户可见但不发送给模型）。

### 渲染与生命周期

1. **触发**：用户在 PromptInput 输入 `/help` → `processSlashCommand.tsx` 匹配到 `help` 命令 → 调用 `command.load()` 动态导入本模块。
2. **执行**：`mod.call(onDone, context, args)` 被调用，返回 `<HelpV2 commands={commands} onClose={onDone} />`。
3. **挂载**：`processSlashCommand.tsx` 第 630–636 行调用 `setToolJSX({ jsx, shouldHidePromptInput: true, showSpinner: false, isLocalJSXCommand: true, isImmediate: false })`。
4. **REPL 渲染**：`src/screens/REPL.tsx` 检测到 `toolJSX.isLocalJSXCommand` 为真，将 `HelpV2` 渲染到 TUI 的合适位置（fullscreen 模式下进入 modal/scrollable，非 fullscreen 模式下 inline 渲染）。
5. **关闭**：用户在 `HelpV2` 中按 ESC 或浏览后退出 → `onDone("Help dialog dismissed", { display: "system" })` → `processSlashCommand.tsx` 的 `onDone` 回调解析为 `messages` 并结束 Promise → REPL 清除 `toolJSX`。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands/help/help.tsx` | **本文件**：`/help` 命令的执行体，导出 `call` 函数。 |
| `src/commands/help/index.ts` | 命令注册入口，通过 `load: () => import('./help.js')` 指向本文件编译产物。 |
| `src/components/HelpV2/HelpV2.tsx` | 被本文件导入的顶层帮助组件，负责绘制标签页（general / commands / custom-commands）、快捷键提示、命令列表等。 |
| `src/components/HelpV2/Commands.tsx` | `HelpV2` 的子组件，用于在 "commands" 和 "custom-commands" 标签页中渲染可选择的命令列表。 |
| `src/components/HelpV2/General.tsx` | `HelpV2` 的子组件，渲染 "general" 标签页中的简介文本和 `PromptInputHelpMenu` 快捷键一览。 |
| `src/components/PromptInput/PromptInput.tsx` | 提供 `?` 快捷触发帮助（第 855–858 行），以及 `helpOpen`/`setHelpOpen` 状态管理。 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `local-jsx` 命令执行器，负责调用 `load()`、`call()`、管理 `onDone` Promise 和 `setToolJSX`（第 551–655 行）。 |
| `src/screens/REPL.tsx` | 维护 `localJSXCommandRef` 和 `toolJSX` 状态，决定 local-jsx 组件的渲染层级与位置（第 1041–1141、4533–4604 行）。 |
| `src/types/command.ts` | 定义 `LocalJSXCommandCall`、`LocalJSXCommandOnDone`、`CommandResultDisplay` 等核心类型。 |

## 依赖与外部交互

### 上游依赖（调用方）

- **`processSlashCommand.tsx`**：唯一运行时调用本文件 `call` 函数的代码路径。它通过 `command.load()` 获取模块，再调用 `mod.call(onDone, context, args)`。
- **`src/commands.ts`** / **`getCommands()`**：提供 `context.options.commands`，即当前用户可见的完整命令列表（包含内置命令、技能、插件、工作流等，并已经过 `meetsAvailabilityRequirement` 和 `isCommandEnabled` 过滤）。

### 下游依赖（被调用方）

- **`HelpV2.tsx`**：接收 `commands`（`Command[]`）和 `onClose`（`() => void`）两个 props。
  - `commands` 被用于分类展示：内置命令（`builtinCommands`）、自定义命令（`customCommands`）、ant-only 命令（`antOnlyCommands`，当前被 `false &&` 硬编码禁用）。
  - `onClose` 被绑定到 ESC 快捷键（`help:dismiss`）和 "cancel" 提示上。

### 横向交互

- **快捷键系统**：`HelpV2` 内部通过 `useKeybinding('help:dismiss', close, { context: 'Help' })` 注册 ESC 关闭。该 action 在 `src/keybindings/schema.ts` 第 125 行定义，默认绑定在 `src/keybindings/defaultBindings.ts` 第 216–218 行的 `Help` context 中。
- **Ctrl+C / Ctrl+D 退出**：`HelpV2` 使用 `useExitOnCtrlCDWithKeybindings(close)` 钩子，确保用户在帮助弹窗打开时按 Ctrl+C 或 Ctrl+D 也能正确关闭弹窗并退出应用。
- **PromptInput 的 `?` 触发**：在 `PromptInput.tsx` 第 855 行，当输入框内容变为 `?` 时，会调用 `setHelpOpen(v => !v)` 打开一个轻量帮助提示（与 `/help` 命令渲染的 `HelpV2` 弹窗是两套 UI，但共享部分组件如 `PromptInputHelpMenu`）。

## 风险、边界与改进建议

### 风险

1. **`commands` 数组的体积与稳定性**：`call` 函数直接将 `context.options.commands` 传入 `HelpV2`。该数组可能包含数十到上百个命令对象（尤其是安装了多个插件或技能时）。`HelpV2` 内部使用 React Compiler 的 `_c` 缓存机制，但 `commands` 引用变化会导致大量 memo 失效和重新过滤计算（`builtInCommandNames`、`filter`、`sort`、`map`）。在快速连续打开/关闭帮助时，可能造成不必要的 CPU 开销。
2. **`args` 被完全忽略**：`call` 签名包含 `args: string`，但函数体未使用。若用户输入 `/help something`，系统不会给出任何参数错误提示，而是静默忽略参数并打开帮助。这与某些命令（如 `/config set`）的行为不一致，可能让用户误以为参数生效。
3. **无错误边界**：如果 `HelpV2` 或其子组件在渲染时抛出异常（例如 `commands` 中包含格式异常的命令对象），异常会向上冒泡到 `processSlashCommand.tsx` 的 `.catch()` 中，导致帮助弹窗直接消失，用户可能看不到任何反馈。

### 边界

- **非交互模式**：`processSlashCommand.tsx` 在 `isNonInteractiveSession` 为 `true` 时，会在 `mod.call()` 返回 JSX 后直接 `resolve({ messages: [], shouldQuery: false })`，不调用 `setToolJSX`。因此本文件返回的 `<HelpV2 />` 实际上会被丢弃。这是符合预期的——非交互环境没有 TUI 可以渲染。
- **Fullscreen vs Non-fullscreen**：`help` 未标记 `immediate: true`，在 fullscreen 模式下属于非立即 local-jsx 命令，会被渲染在 scrollable 区域（REPL.tsx 注释明确说明）。这意味着如果聊天记录很长，帮助弹窗会随滚动条移动，而不是像 `/btw` 那样固定在底部。
- **Remote 模式可用性**：`help` 被加入 `REMOTE_SAFE_COMMANDS`，在 `--remote` 模式下保留；但由于它是 `local-jsx` 类型，`isBridgeSafeCommand()` 返回 `false`，无法通过手机/网页端的 Remote Control bridge 调用。

### 改进建议

1. **参数校验与提示**：建议对 `args` 做简单校验。若 `args.trim().length > 0`，可在 `onDone` 中返回一条提示信息，例如 `"Help does not accept arguments. Try /help or ? for shortcuts."`，提升用户体验一致性。
2. **命令列表缓存优化**：`HelpV2` 内部对 `commands` 的分类过滤（builtin / custom / ant-only）在每次打开时都会重新执行。可考虑在 `help.tsx` 或 `HelpV2` 中使用 `useMemo`（或利用已有的 React Compiler 缓存）对分类结果做稳定化，减少重复计算。若 `commands` 引用频繁变化，可进一步在 `REPL` 层通过 `useMemo` 稳定传入 `PromptInput` 的 `commands` prop。
3. **增加错误边界**：在 `help.tsx` 返回的 JSX 外层包裹一个轻量的错误边界组件（Ink 生态中可用 `ErrorBoundary` 模式），捕获 `HelpV2` 渲染异常，并通过 `onDone` 返回友好的错误文本，避免静默失败。
4. **测试覆盖**：目前未找到针对 `help.tsx` 的测试。建议补充：
   - 单元测试：验证 `call` 函数返回的 React 元素类型为 `HelpV2`，且 props 正确传递；
   - 集成测试：模拟用户输入 `/help`，验证 `setToolJSX` 被调用且 `shouldHidePromptInput: true`；
   - 组件测试：验证 `HelpV2` 在 `commands` 为空、仅有内置命令、包含自定义技能等不同场景下的标签页渲染行为。
5. **考虑 `immediate` 标记**：如果产品希望帮助弹窗在 fullscreen 模式下像其他即时弹窗一样固定在底部（避免随聊天记录滚动），可在 `src/commands/help/index.ts` 中增加 `immediate: true`。但需同步评估对焦点管理、ESC 处理、以及 `useQueueProcessor` 中 `hasActiveLocalJsxUI` 判断的影响。
