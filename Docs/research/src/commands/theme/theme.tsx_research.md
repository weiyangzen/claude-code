# 研究文档：src/commands/theme/theme.tsx

## 场景与职责

`src/commands/theme/theme.tsx` 是 `/theme` slash 命令的**实际 UI 实现层**。当用户在 REPL 中输入 `/theme` 时，REPL 引擎通过 `local-jsx` 执行路径调用该模块导出的 `call` 函数，后者返回一个 Ink/React 组件树，在终端内渲染交互式主题选择器。该文件负责：

1. 挂载 `ThemePickerCommand` 组件；
2. 通过 `useTheme()` 与全局主题状态交互；
3. 处理用户"选择主题"和"取消"两种交互结果；
4. 通过 `onDone` 回调向 REPL 报告命令完成状态。

## 功能点目的

1. **渲染主题选择器**：将 `ThemePicker` 组件包装在 `Pane` 容器中，提供与 REPL 一致的命令面板视觉风格。
2. **应用主题变更**：用户确认选择后，调用 `setTheme(setting)` 立即切换全局主题，并返回成功消息。
3. **支持取消流程**：用户按 Esc 取消时，返回系统级消息（`display: 'system'`），不修改主题。
4. **符合 local-jsx 命令契约**：实现 `LocalJSXCommandCall` 签名，接收 `onDone` 和 `context`，返回 `React.ReactNode`。

## 具体技术实现

### 组件结构

```tsx
function ThemePickerCommand({ onDone }: Props): React.ReactNode {
  const [, setTheme] = useTheme()
  return (
    <Pane color="permission">
      <ThemePicker
        onThemeSelect={setting => {
          setTheme(setting)
          onDone(`Theme set to ${setting}`)
        }}
        onCancel={() => {
          onDone('Theme picker dismissed', { display: 'system' })
        }}
        skipExitHandling={true}
      />
    </Pane>
  )
}
```

### 命令入口

```tsx
export const call: LocalJSXCommandCall = async (onDone, _context) => {
  return <ThemePickerCommand onDone={onDone} />
}
```

- `call` 是 `local-jsx` 命令的标准导出，由 REPL 在匹配到 `/theme` 后异步调用。
- `_context` 被忽略，说明 `/theme` 不依赖当前工具上下文（如 `ToolUseContext` 中的 IDE 状态、MCP 配置等）。

### React Compiler 编译痕迹

文件源码经过 React Compiler（`react/compiler-runtime`）编译，所有 JSX 创建和函数闭包都被 memo cache（`$[n]`）包裹。例如：

- `onThemeSelect` 和 `onCancel` 回调仅在依赖（`onDone`、`setTheme`）变化时重新创建；
- `Pane` + `ThemePicker` 的 JSX 节点在依赖不变时直接复用缓存对象。

这对理解运行时行为很重要：虽然源码看起来像普通 React 组件，实际运行时是高度 memoized 的。

## 关键代码路径与文件引用

### 上游调用方

| 文件 | 作用 |
|------|------|
| `src/commands/theme/index.ts` | 懒加载入口 `load: () => import('./theme.js')`，指向本文件。 |
| `src/screens/REPL.tsx:3265` | `const jsx = await mod.call(onDone, context, commandArgs)`，`local-jsx` 命令的统一执行点。 |
| `src/screens/REPL.tsx:4533-4600` | 说明 `theme` 属于"非 immediate local-jsx"命令，渲染在可滚动区域内（与 `/btw` 等 immediate 命令区分）。 |

### 下游依赖

| 文件 | 作用 |
|------|------|
| `src/commands.ts` | 提供 `CommandResultDisplay` 类型（`'skip' \| 'system' \| 'user'`）。 |
| `src/components/design-system/Pane.tsx` | 提供 `Pane` 容器组件，用于绘制命令面板顶部分隔线并添加内边距。 |
| `src/components/ThemePicker.tsx` | 提供核心的交互式主题选择 UI（基于 `Select` 组件）。 |
| `src/ink.ts` | 重新导出 `useTheme`（实际来自 `ThemeProvider.tsx`）。 |
| `src/types/command.ts` | 提供 `LocalJSXCommandCall` 类型定义。 |

### 主题状态链路

| 文件 | 作用 |
|------|------|
| `src/components/design-system/ThemeProvider.tsx` | 维护 `themeSetting`、`previewTheme`、`currentTheme`，提供 `useTheme()` / `usePreviewTheme()` / `useThemeSetting()`。 |
| `src/utils/theme.ts` | 定义 `ThemeName`、`ThemeSetting`、`THEME_NAMES`、`getTheme()` 以及 6 套具体颜色主题（dark/light/daltonized/ansi 等）。 |
| `src/utils/config.ts` | `saveGlobalConfig()` 持久化用户选择的主题到全局配置。 |

## 依赖与外部交互

### 状态交互

- `useTheme()` 返回 `[currentTheme, setThemeSetting]`。`ThemePickerCommand` 只使用第二个元素（setter），在用户选择时立即调用 `setTheme(setting)` 更新全局状态。
- `ThemePicker` 内部使用 `usePreviewTheme()` 实现"焦点预览"（鼠标/键盘悬停时临时切换主题），而 `ThemePickerCommand` 本身不直接参与预览逻辑。

### 与 REPL 的契约

- `onDone(result, options)` 是 REPL 注入的回调，用于通知命令生命周期结束。
- 选择成功时：`onDone('Theme set to ${setting}')`，默认 `display` 为 `'user'`（显示在对话流中）。
- 取消时：`onDone('Theme picker dismissed', { display: 'system' })`，以系统消息样式显示。
- `skipExitHandling={true}` 告知 `ThemePicker` 不要自行处理 `Ctrl+C` 退出逻辑，因为 REPL 的 `local-jsx` 执行框架已经接管了生命周期。

### 视觉层级

- `Pane color="permission"` 使用主题中的 `permission` 颜色绘制顶部分隔线，与 `/permissions`、 `/config` 等本地命令保持一致的视觉语言。
- 在 `FullscreenLayout` 的 modal slot 中渲染时，`Pane` 会检测到 `useIsInsideModal()` 为 `true`，从而隐藏自己的顶部分隔线，避免双重边框。

## 风险、边界与改进建议

### 风险

1. **硬编码的 `display: 'system'`**：取消消息被固定为系统显示级别。如果未来产品需求希望取消时不显示任何消息，需要修改源码；目前无法通过配置调整。
2. **`skipExitHandling` 与嵌套调用的耦合**：`ThemePickerCommand` 强制 `skipExitHandling={true}`，这意味着如果未来有场景需要 `ThemePicker` 独立运行（不通过 REPL 的 local-jsx 路径），`Ctrl+C` 行为可能不符合预期。
3. **编译后代码的可维护性**：由于文件经过 React Compiler 编译，原始 TSX 结构被大量 memo cache 逻辑包裹。手动调试或打补丁时需要对照 source map 或原始源码，直接阅读编译产物容易误判依赖关系。

### 边界

- **无参数解析**：`/theme` 不接受任何命令参数。用户无法通过 `/theme dark` 直接跳过交互；REPL 会将 `args` 传入 `call`，但本实现完全忽略 `_context` 和 `args`。
- **无错误处理**：`setTheme` 和 `onDone` 都是同步调用，没有 try/catch。若 `setTheme` 内部抛出异常（理论上极少），异常会冒泡到 REPL 的 `local-jsx` 执行逻辑中。
- **Bridge 安全**：`theme` 被加入 `REMOTE_SAFE_COMMANDS`，但 `isBridgeSafeCommand()` 对 `local-jsx` 类型统一返回 `false`，因此从 mobile/web bridge 发送的 `/theme` 实际上会被阻止。

### 改进建议

1. **支持命令行参数快速切换**：可扩展 `call` 函数解析 `args`（如 `"dark"`），若参数有效则直接 `setTheme(args)` 并跳过 `ThemePicker` 渲染，提升效率。
2. **抽离纯逻辑层**：将 `setTheme + onDone` 的回调组合提取为可测试的纯函数，便于单元测试而不必挂载 Ink 渲染器。
3. **增加取消时的静默选项**：允许通过 `args` 或配置控制取消时是否发送消息，减少对话流中的噪音。
4. **类型安全增强**：当前 `_context` 被完全忽略，可考虑显式解构出不需要的字段并标注原因，或利用 `Omit` 类型声明一个更精简的 `call` 签名，防止未来上下文扩展时产生隐式依赖。
