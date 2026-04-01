# General.tsx 研究文档

## 场景与职责

`General.tsx` 是 Claude Code 帮助系统（HelpV2）中 **"general" 标签页的内容组件**。它负责展示产品的核心定位文案以及一组常用快捷键提示，是用户打开 `/help` 后看到的第一个标签页。该组件为纯展示型，不承担任何交互逻辑或状态管理。

## 功能点目的

1. **产品简介展示**：向用户说明 Claude Code 的核心能力——理解代码库、在获得许可时进行编辑、直接在终端执行命令。
2. **快捷键汇总**：通过嵌入 `PromptInputHelpMenu` 组件，将 REPL 输入框可用的全局快捷键以结构化方式呈现给用户。
3. **视觉层级构建**：使用 `Box` 嵌套和 `gap`/`paddingY` 等布局属性，在终端内建立清晰的信息层级。

## 具体技术实现

### 组件结构

```tsx
<Box flexDirection="column" paddingY={1} gap={1}>
  <Box>
    <Text>Claude understands your codebase...</Text>
  </Box>
  <Box flexDirection="column">
    <Box>
      <Text bold>Shortcuts</Text>
    </Box>
    <PromptInputHelpMenu gap={2} fixedWidth={true} />
  </Box>
</Box>
```

### 布局细节

- 外层 `Box` 使用 `flexDirection="column"`、`paddingY={1}`、`gap={1}`，在垂直方向上为各区块提供均匀呼吸感。
- 内层 "Shortcuts" 区块同样使用 `flexDirection="column"`，标题加粗（`bold`），下方紧跟 `PromptInputHelpMenu`。
- `PromptInputHelpMenu` 接收 `gap={2}` 和 `fixedWidth={true}`，使快捷键列表在终端中以固定列宽对齐，保持整齐。

### React Compiler 缓存

文件已被 React Compiler 编译。由于组件无 props 和 hooks，整个 JSX 树被静态缓存到 memo 槽位中（`$[0]`、`$[1]`），实际运行时几乎为零开销。

## 关键代码路径与文件引用

- **本文件**：`src/components/HelpV2/General.tsx`
- **调用方**：`src/components/HelpV2/HelpV2.tsx`（在 `<Tab key="general" title="general">` 中渲染）
- **依赖文件**：
  - `src/ink.ts` — `Box`、`Text`
  - `src/components/PromptInput/PromptInputHelpMenu.tsx` — `PromptInputHelpMenu`

## 依赖与外部交互

| 依赖 | 交互方式 | 说明 |
|------|----------|------|
| `General.tsx` → `PromptInputHelpMenu.tsx` | JSX 嵌套 | 传入 `gap={2}`、`fixedWidth={true}` 控制子组件布局 |
| `General.tsx` → `ink.ts` | 导入基础组件 | 使用 `Box`、`Text` 进行终端 UI 布局 |

`PromptInputHelpMenu` 内部进一步依赖：
- `src/keybindings/useShortcutDisplay.ts` — 解析并显示用户当前配置的快捷键
- `src/utils/platform.ts` — 平台判断（如 Windows 下不显示 `ctrl + z`）
- `src/utils/fastMode.ts` — 判断是否显示 fast mode 快捷键
- `src/keybindings/loadUserBindings.ts` — 判断是否显示 `/keybindings` 自定义提示

## 风险、边界与改进建议

1. **文案完全硬编码**：产品介绍文本写死在组件中，无外部配置或国际化（i18n）机制。若未来需要多语言支持，必须重构为配置化或翻译键值形式。

2. **与 PromptInputHelpMenu 强耦合**：General 标签页的所有实质信息都来自 `PromptInputHelpMenu`，这意味着如果 REPL 输入框的快捷键发生变化，帮助页面会自动同步，这是优势；但如果希望帮助页有独立于输入框的快捷键说明，则当前设计缺乏灵活性。

3. **无滚动或截断处理**：当终端高度极小时，整个 General 标签页内容可能超出可视区域。虽然外层 `Tabs` 在部分模式下会包裹 `ScrollBox`，但 `General.tsx` 本身没有针对内容过长的自适应处理（如折叠或分页）。

4. **无测试覆盖**：组件虽简单，但缺少快照测试或渲染测试，无法防止未来布局属性被意外修改。

5. **改进建议**：
   - 将产品介绍文本抽离到常量文件或配置中，便于非研发人员修改。
   - 增加对终端最小高度的检测，在高度不足时给出简化版提示或启用滚动。
