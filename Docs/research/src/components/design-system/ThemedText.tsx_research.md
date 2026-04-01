# ThemedText.tsx 研究文档

## 场景与职责

`ThemedText.tsx` 是 Claude Code TUI 中所有文本渲染的**主题感知入口层**。它包装了底层 Ink `Text` 组件，使业务代码能够使用语义化主题键（如 `"text"`、`"error"`、`"inactive"`）作为 `color` 或 `backgroundColor`，而无需关心当前主题的具体 RGB/ANSI 值。

与 `ThemedBox` 类似，该组件通过 `src/ink.ts` 以 `Text` 之名重新导出，成为全代码库默认使用的文本组件。此外，它还提供了一个特殊的 React Context —— `TextHoverColorContext` —— 用于在跨 Box 边界时向子树中的未着色文本注入悬停色，解决 Ink 样式级联无法跨 Box 传播的问题。

## 功能点目的

1. **主题键转原始色值**：将 `color` 和 `backgroundColor` prop 中的 `keyof Theme` 解析为底层 `Text` 可接受的 `Color`。
2. **`dimColor` 的语义化实现**：当 `dimColor={true}` 时，不依赖 ANSI 的 `dim` SGR（会与 `bold` 互斥），而是直接使用 `theme.inactive` 颜色，实现“变暗且兼容 bold”的效果。
3. **悬停色跨边界级联**：通过 `TextHoverColorContext`，父组件可以在鼠标悬停时向深层未显式设置 `color` 的 `ThemedText` 注入颜色（如 `"text"`），且优先级为：`显式 color > hoverColor > dimColor`。
4. **保持 Ink Text 完整能力**：保留 `bold`、`italic`、`underline`、`strikethrough`、`inverse`、`wrap` 等所有文本样式 prop。
5. **React Compiler 手动 memo 优化**：使用 `_c(10)` 缓存槽减少重渲染。

## 具体技术实现

### 类型定义

```tsx
export const TextHoverColorContext = React.createContext<keyof Theme | undefined>(undefined);

export type Props = {
  readonly color?: keyof Theme | Color;
  readonly backgroundColor?: keyof Theme;
  readonly dimColor?: boolean;
  readonly bold?: boolean;
  readonly italic?: boolean;
  readonly underline?: boolean;
  readonly strikethrough?: boolean;
  readonly inverse?: boolean;
  readonly wrap?: Styles['textWrap'];
  readonly children?: ReactNode;
};
```

- `color` 接受 `keyof Theme | Color`，允许主题键或原始色值。
- `backgroundColor` 仅接受 `keyof Theme`，这是设计上的限制：业务代码中的文本背景色通常只需要主题语义键（如 `"userMessageBackground"`）。
- `TextHoverColorContext` 的值类型也是 `keyof Theme | undefined`，注释明确说明其优先级高于 `dimColor`、低于显式 `color`，且能跨 Box 边界传播。

### 颜色解析函数 `resolveColor`

与 `ThemedBox.tsx` 中的实现几乎完全一致：

```tsx
function resolveColor(
  color: keyof Theme | Color | undefined,
  theme: Theme
): Color | undefined {
  if (!color) return undefined;
  if (
    color.startsWith('rgb(') ||
    color.startsWith('#') ||
    color.startsWith('ansi256(') ||
    color.startsWith('ansi:')
  ) {
    return color as Color;
  }
  return theme[color as keyof Theme] as Color;
}
```

### 颜色优先级逻辑

```tsx
const resolvedColor = !color && hoverColor
  ? resolveColor(hoverColor, theme)
  : dimColor
    ? theme.inactive as Color
    : resolveColor(color, theme);

const resolvedBackgroundColor = backgroundColor
  ? theme[backgroundColor] as Color
  : undefined;
```

优先级链：
1. 若显式传了 `color`，始终使用 `resolveColor(color, theme)`（`dimColor` 和 `hoverColor` 均被忽略）。
2. 若未传 `color` 但存在 `hoverColor`（来自 Context），使用 `hoverColor`。
3. 若前两者都不存在且 `dimColor={true}`，使用 `theme.inactive`。
4. 若都没有，返回 `undefined`（继承父级或默认终端前景色）。

### 渲染流程

1. 解构 props，为布尔值和 `wrap` 提供默认值（`dimColor=false`, `bold=false`, `italic=false`, `underline=false`, `strikethrough=false`, `inverse=false`, `wrap="wrap"`）。
2. 调用 `useTheme()` 获取主题名，`getTheme(themeName)` 获取主题对象。
3. 通过 `useContext(TextHoverColorContext)` 读取悬停色。
4. 按优先级计算 `resolvedColor` 和 `resolvedBackgroundColor`。
5. 将解析后的颜色与所有样式 prop 传给底层 `<Text>`。

### React Compiler Memo 结构

编译后使用 `_c(10)` 缓存槽：
- 输入 prop 解构后直接使用局部变量。
- 最终 JSX 的缓存依赖为：`bold | children | inverse | italic | resolvedBackgroundColor | resolvedColor | strikethrough | underline | wrap`。
- 只要这 9 个值不变，跳过 `<Text>` 的重新创建。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/design-system/ThemedText.tsx` | 本文件，主题感知 Text 实现。 |
| `src/ink/components/Text.tsx` | 底层 Ink Text，接收原始 `Color` 与文本样式 prop。 |
| `src/ink/styles.ts` | 定义 `Color`、`Styles`、`TextStyles` 类型。 |
| `src/utils/theme.ts` | `Theme` 类型与 `getTheme()` 实现。 |
| `src/components/design-system/ThemeProvider.tsx` | 提供 `useTheme()` hook。 |
| `src/ink.ts` | 重新导出 `ThemedText` 作为 `Text`，并导出 `TextProps` 类型。 |

### 典型调用方

- `src/components/VirtualMessageList.tsx`：在 `VirtualItem` 中通过 `TextHoverColorContext.Provider` 向消息内容注入悬停色；`hovered && !expanded ? "text" : undefined`。
- `src/components/messages/SystemTextMessage.tsx`：系统消息中大量使用 `<Text>`（即 ThemedText）设置 `dimColor`、`color="warning"`、`bold` 等。
- `src/components/TaskListV2.tsx`：任务列表中直接使用 `ThemedText` 导入来渲染任务名称与状态。
- `src/components/teams/TeamsDialog.tsx`：对话框中使用 `ThemedText` 渲染 teammate 名称与颜色标签。
- `src/components/design-system/Dialog.tsx`、`ListItem.tsx`、`Tabs.tsx` 等 design-system 组件：内部均使用 `<Text>` 或 `<ThemedText>` 渲染标签与提示文字。

## 依赖与外部交互

### 上游依赖
- **`useTheme()` / `getTheme()`**：同 ThemedBox，获取当前主题色表。
- **底层 `<Text>`**（`ink/components/Text.tsx`）：最终渲染为 `<ink-text>` 自定义元素，由 Ink reconciler 处理文本测量与样式应用。
- **`TextHoverColorContext`**：由调用方（如 `VirtualMessageList`）通过 `Provider` 注入，ThemedText 作为 Consumer 读取。

### 下游消费
- 通过 `src/ink.ts` 以 `Text` 名义被全代码库消费；任何 `import { Text } from '../ink.js'` 的模块都在间接使用 `ThemedText`。
- 直接 import `ThemedText` 的模块通常需要同时消费 `TextHoverColorContext`（如 `VirtualMessageList.tsx`）。

### 数据流
```
业务组件 (TextProps with theme keys)
        ↓
ThemedText (resolveColor + TextHoverColorContext → raw Color)
        ↓
Ink Text (ink-text reconciler → textStyles + wrap styles)
        ↓
Ink renderer (measure-text + applyTextStyles → ANSI 输出)
```

## 风险、边界与改进建议

### 风险与边界

1. **`resolveColor` 与 ThemedBox 重复**：两处维护完全相同的颜色解析逻辑。若未来新增颜色格式或修改前缀判断规则，需要同时修改两个文件，容易遗漏。
2. **`backgroundColor` 类型不对称**：`color` 接受 `keyof Theme | Color`，但 `backgroundColor` 只接受 `keyof Theme`。这意味着调用方无法直接给文本设置原始背景色（如 `#ff0000`）。虽然业务上极少需要，但在构建通用 design-system 组件时可能造成意外限制。
3. **HoverColor 的 Context 穿透与性能**：`TextHoverColorContext` 的值变化会导致整个子树中所有 `ThemedText` 重新评估 `resolvedColor`。在消息列表快速滚动或大量文本节点场景下，频繁的 hover 状态切换可能触发大量 Text 重渲染。不过由于 React Compiler 的 memo，实际影响被限制在 Context 子树内未 memo 的节点。
4. **`dimColor` 与 `color` 的互斥不够显式**：类型系统允许同时传入 `color` 和 `dimColor`，但运行时 `color` 优先级更高，`dimColor` 被静默忽略。这对新开发者来说不够直观。
5. **无 children 时的行为由底层 Text 决定**：ThemedText 本身不处理 `children === undefined/null` 的情况，全部委托给底层 `Text.tsx`。底层 Text 在 `children == null` 时返回 `null`，行为一致。

### 改进建议

1. **提取公共 `resolveColor` 工具函数**：将 `resolveColor` 提取到 `src/components/design-system/resolveColor.ts`（或 `utils/resolveColor.ts`），供 `ThemedBox`、`ThemedText`、`color.ts` 共用，消除重复代码。
2. **放宽 `backgroundColor` 类型为 `keyof Theme | Color`**：与 `color` 保持一致，提升设计系统的通用性。需要确认底层 Ink Text 的 `backgroundColor` 类型是否已支持 `Color`（`Text.tsx` 中 `backgroundColor?: Color`，因此类型上是可行的）。
3. **为 `dimColor` + `color` 同时存在时增加 dev warning**：在开发环境下若两者同时传入且 `color` 非空，可打印警告提示 `dimColor` 将被忽略，减少调试困惑。
4. **考虑将 `TextHoverColorContext` 的默认值改为 `null` 而非 `undefined`**：目前 Context 默认值为 `undefined`，与“未设置 hoverColor”和“Context 未提供”语义重叠。改为 `null` 可以更清晰地区分“显式清空 hoverColor”与“未在 Provider 中”。
5. **文档化颜色优先级**：在组件 JSDoc 或 design-system 文档中明确写出 `color > hoverColor > dimColor > default` 的优先级链，帮助业务开发者正确选择 prop。
