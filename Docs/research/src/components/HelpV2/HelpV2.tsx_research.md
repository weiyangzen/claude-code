# HelpV2.tsx 研究文档

## 场景与职责

`HelpV2.tsx` 是 Claude Code `/help` 命令的**主 UI 组件**，负责渲染一个模态化的帮助对话框。它聚合了所有可用命令（内置命令、自定义命令、技能命令等），按分类组织到多个标签页中，并提供完整的键盘导航、关闭交互和终端尺寸自适应能力。它是用户了解产品功能、浏览命令列表的核心入口。

## 功能点目的

1. **命令分类展示**：将传入的 `Command[]` 分为 "general"（通用介绍）、"commands"（内置命令）、"custom-commands"（自定义命令）三类，以标签页形式呈现。
2. **键盘交互闭环**：
   - 绑定 `help:dismiss` 快捷键用于关闭帮助窗口；
   - 支持双击 `Ctrl+C`/`Ctrl+D` 退出；
   - 标签页头部与内容区之间可通过上下箭头切换焦点。
3. **终端自适应**：根据终端行数动态计算帮助弹窗的最大高度，确保不遮挡过多 REPL 内容。
4. **命令过滤与隐藏**：自动过滤掉 `isHidden` 命令，并区分内置命令与非内置命令。

## 具体技术实现

### 组件结构与数据流

```tsx
HelpV2(props: { onClose, commands })
  → useTerminalSize()           // 获取 rows/columns
  → useIsInsideModal()          // 判断是否处于模态槽位
  → builtInCommandNames()       // 获取内置命令名集合
  → filter/split commands       // 分为 builtin / custom / antOnly
  → Tabs / Tab / Pane           // 渲染标签页容器
    → General                   // general 标签
    → Commands                  // commands 标签
    → Commands                  // custom-commands 标签
```

### 命令分类逻辑

```ts
const builtinNames = builtInCommandNames()
const builtinCommands = commands.filter(cmd => builtinNames.has(cmd.name) && !cmd.isHidden)
const customCommands  = commands.filter(cmd => !builtinNames.has(cmd.name) && !cmd.isHidden)
const antOnlyCommands = [] // 硬编码为空数组，相关 Tab 已禁用
```

- `builtInCommandNames()` 来自 `src/commands.ts`，是一个 memoized 的 `Set<string>`，包含所有内置命令的 `name` 和 `aliases`。
- 任何 `isHidden === true` 的命令都会被过滤掉，不在帮助中展示。

### 尺寸计算

- `maxHeight = Math.floor(rows / 2)`：帮助弹窗高度最多占终端行数的一半。
- `insideModal = useIsInsideModal()`：若在全屏布局的 modal slot 中渲染，则高度由外层控制，不额外限制；否则将 `maxHeight` 传给外层 `Box` 的 `height` 属性。

### 键盘与关闭交互

| 功能 | 实现方式 | 说明 |
|------|----------|------|
| 关闭帮助 | `useKeybinding("help:dismiss", close, { context: "Help" })` | 解析用户配置的关闭快捷键（默认 Esc） |
| 双击退出 | `useExitOnCtrlCDWithKeybindings(close)` | 提供 "再按一次退出" 的防误触机制 |
| 快捷键显示 | `useShortcutDisplay("help:dismiss", "Help", "esc")` | 在底部显示当前生效的关闭快捷键提示 |

### Tabs 配置

- 使用 `Tabs` 组件（来自 `../design-system/Tabs.js`）管理标签页。
- `defaultTab="general"`：默认打开通用介绍页。
- `color="professionalBlue"`：统一使用专业蓝主题色。
- 标题动态显示 `Claude Code v${MACRO.VERSION}`，其中 `MACRO.VERSION` 为构建时注入的宏变量。

### 标签页详情

1. **general**：`<General />` 静态介绍页。
2. **commands**：`<Commands commands={builtinCommands} ... title="Browse default commands:" />`
3. **custom**：`<Commands commands={customCommands} ... title="Browse custom commands:" emptyMessage="No custom commands found" />`
4. **ant-only**：代码中保留但已被 `false && antOnlyCommands.length > 0` 条件禁用，属于死代码。

### 底部信息栏

- 文档链接：`<Link url="https://code.claude.com/docs/en/overview" />`
- 取消提示：根据 `exitState.pending` 显示 "Press {key} again to exit" 或 "{dismissShortcut} to cancel"

### React Compiler 缓存

文件已被 React Compiler 编译，使用 `_c(44)` 进行 44 槽位的细粒度 memo。`tabs` 数组、`close` 回调、各个子 JSX 节点均按依赖变化进行缓存，避免高频重渲染。

## 关键代码路径与文件引用

- **本文件**：`src/components/HelpV2/HelpV2.tsx`
- **调用方**：`src/commands/help/help.tsx` — `/help` 命令的 JSX 入口，将 `commands` 和 `onDone` 传入 `HelpV2`
- **子组件**：
  - `src/components/HelpV2/General.tsx`
  - `src/components/HelpV2/Commands.tsx`
- **依赖文件**：
  - `src/hooks/useExitOnCtrlCDWithKeybindings.ts` — 双击退出逻辑
  - `src/keybindings/useShortcutDisplay.ts` — 快捷键显示
  - `src/commands.ts` — `Command` 类型、`builtInCommandNames`、`INTERNAL_ONLY_COMMANDS`
  - `src/context/modalContext.ts` — `useIsInsideModal`
  - `src/hooks/useTerminalSize.ts` — 终端尺寸
  - `src/ink.ts` — `Box`、`Link`、`Text`
  - `src/keybindings/useKeybinding.ts` — `useKeybinding`
  - `src/components/design-system/Pane.tsx` — 弹窗容器
  - `src/components/design-system/Tabs.tsx` — 标签页组件

## 依赖与外部交互

| 依赖 | 交互方式 | 说明 |
|------|----------|------|
| `HelpV2` → `commands.ts` | 导入类型与工具函数 | 获取命令分类依据 `builtInCommandNames` |
| `HelpV2` → `Tabs.tsx` | JSX 嵌套 + 隐式焦点协议 | `Commands` 内部通过 `useTabHeaderFocus` 与 `Tabs` 协同 |
| `HelpV2` → `modalContext.ts` | `useIsInsideModal()` | 决定是否在模态槽位中，影响高度与边框 |
| `HelpV2` → `useKeybinding.ts` | Hook 调用 | 注册 `help:dismiss` 关闭快捷键 |
| `HelpV2` → `useExitOnCtrlCDWithKeybindings.ts` | Hook 调用 | 注册双击退出行为 |
| `help/help.tsx` → `HelpV2.tsx` | JSX 调用 | `/help` 命令的渲染入口 |

## 风险、边界与改进建议

1. **死代码残留**：`antOnlyCommands` 及其对应的 Tab 被 `false &&` 永久禁用，但相关 JSX 缓存逻辑和变量声明仍保留在组件中，增加了维护负担和编译产物体积。建议彻底移除。

2. **`MACRO.VERSION` 构建依赖**：标题中的版本号依赖构建时宏注入，若在非标准构建环境（如某些测试场景）中运行，可能导致 `MACRO.VERSION` 未定义或报错。

3. **高度计算过于粗放**：`maxHeight = Math.floor(rows / 2)` 在大终端（如 60+ 行）上会占用 30 行，可能过度遮挡 REPL 历史记录。建议引入上限 clamp（例如最多 20 行）或根据内容实际高度动态调整。

4. **命令分类的单向依赖**：`builtInCommandNames()` 是唯一的分类标准，这意味着：
   - 若某个内置命令被动态技能同名覆盖，它会被划入 "custom-commands"；
   - 若插件命令与内置命令同名，同样会被误分类。
   建议增加 `source` 或 `loadedFrom` 字段作为辅助分类依据。

5. **无加载态与错误态**：`commands` 由调用方直接传入，HelpV2 假设数据已就绪。若命令加载失败或为空，仅能通过 `emptyMessage` 做简单提示，没有重试或错误详情展示。

6. **缺少测试覆盖**：目录下无测试文件，命令分类逻辑、Tab 渲染条件、尺寸计算、关闭回调等关键行为均缺乏自动化测试。

7. **改进建议**：
   - 清理 `antOnlyCommands` 死代码。
   - 将 `maxHeight` 计算抽离为可测试的纯函数，并增加上限约束。
   - 为命令分类逻辑编写单元测试，覆盖内置/自定义/隐藏/别名等边界场景。
   - 考虑在帮助页底部增加 `/docs` 或 `/keybindings` 等快速入口的交互链接。
