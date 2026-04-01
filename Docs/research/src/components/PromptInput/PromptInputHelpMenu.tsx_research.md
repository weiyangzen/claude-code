# PromptInputHelpMenu.tsx 深度研究文档

> **研究对象**: `src/components/PromptInput/PromptInputHelpMenu.tsx`  
> **研究范围**: 源码、调用方、快捷键系统、平台工具及相关依赖  
> **执行器**: kimi (k2p5)  
> **研究日期**: 2026-04-01

---

## 1. 场景与职责

### 1.1 核心定位

`PromptInputHelpMenu.tsx` 是 Claude Code CLI 中**输入区域的快捷键帮助菜单组件**。当用户按下 `?` 键时，该组件在 prompt footer 位置渲染一个三栏式的快捷键速查表，帮助用户快速了解可用的键盘操作和输入模式前缀。

### 1.2 应用场景

| 场景 | 说明 |
|------|------|
| **快捷键帮助** | 用户输入 `?` 或在特定状态下触发帮助菜单 |
| **模式前缀提示** | 显示 `!`、 `/`、 `@`、`&` 等输入模式前缀的用途 |
| **功能开关提示** | 根据功能开关（fast mode、terminal panel、keybindings 定制）动态显示/隐藏对应条目 |
| **平台适配** | Windows 平台不显示 `ctrl+z to suspend` |

### 1.3 职责边界

- **纯展示组件**：只负责渲染帮助文本，不处理任何输入或状态切换
- **动态功能门控**：根据 GrowthBook feature flag、平台类型、用户配置决定显示哪些条目
- **可配置快捷键感知**：通过 `useShortcutDisplay` 读取用户自定义的快捷键绑定

---

## 2. 功能点目的

### 2.1 输入模式前缀速查

帮助新用户快速理解特殊输入前缀：

| 前缀 | 说明 |
|------|------|
| `!` | bash 模式（直接执行 shell 命令） |
| `/` | commands 模式（斜杠命令） |
| `@` | file paths 模式（文件路径引用） |
| `&` | background 模式（后台任务） |
| `/btw` | side question 模式（边问边做） |

### 2.2 可配置快捷键展示

所有涉及快捷键的提示都通过 `useShortcutDisplay` 读取用户自定义绑定，而非硬编码：

```typescript
const transcriptShortcut = formatShortcut(
  useShortcutDisplay("app:toggleTranscript", "Global", "ctrl+o")
);
```

### 2.3 功能开关动态门控

- **Terminal Panel**：仅在 `TERMINAL_PANEL` feature 和 GrowthBook flag `tengu_terminal_panel` 开启时显示 `meta+j for terminal`
- **Fast Mode**：仅在 `isFastModeEnabled() && isFastModeAvailable()` 时显示 `alt+o to toggle fast mode`
- **Keybindings 定制**：仅在 `isKeybindingCustomizationEnabled()` 时显示 `/keybindings to customize`

### 2.4 换行输入提示适配

根据终端类型和键位绑定安装状态，显示不同的换行操作提示：

```typescript
const newlineInstructions = getNewlineInstructions();
// Apple Terminal → "shift + ⏎ for newline"
// iTerm2/VSCode (已安装绑定) → "shift + ⏎ for newline"
// 其他 → "backslash (\) + return (⏎) for newline" 或 "\⏎ for newline"
```

---

## 3. 具体技术实现

### 3.1 组件 Props

```typescript
type Props = {
  dimColor?: boolean;      // 是否以暗淡颜色渲染
  fixedWidth?: boolean;    // 是否固定三栏宽度
  gap?: number;            // 栏间距
  paddingX?: number;       // 水平内边距
};
```

### 3.2 快捷键格式化

将 `ctrl+o` 格式化为更友好的 `ctrl + o`：

```typescript
function formatShortcut(shortcut: string): string {
  return shortcut.replace(/\+/g, ' + ');
}
```

### 3.3 三栏布局结构

帮助菜单采用固定三栏布局：

- **左栏**（宽度 24）：模式前缀提示
  - `! for bash mode`
  - `/ for commands`
  - `@ for file paths`
  - `& for background`
  - `/btw for side question`

- **中栏**（宽度 35）：界面切换与导航
  - `double tap esc to clear input`
  - `shift + tab to auto-accept edits`
  - `ctrl + o for verbose output`
  - `ctrl + t to toggle tasks`
  - 换行提示（动态）
  - terminal 提示（条件）

- **右栏**（自动宽度）：编辑与高级功能
  - `ctrl + _ to undo`
  - `ctrl + z to suspend`（非 Windows）
  - `ctrl + v to paste images`
  - `alt + p to switch model`
  - `alt + o to toggle fast mode`（条件）
  - `ctrl + s to stash prompt`
  - `ctrl + g to edit in $EDITOR`
  - `/keybindings to customize`（条件）

### 3.4 功能门控实现细节

**Terminal Panel 条目**：
```typescript
const terminalShortcutElement = feature("TERMINAL_PANEL") 
  ? getFeatureValue_CACHED_MAY_BE_STALE("tengu_terminal_panel", false)
    ? <Box><Text dimColor>{terminalShortcut} for terminal</Text></Box>
    : null
  : null;
```

**Fast Mode 条目**：
```typescript
const fastModeElement = isFastModeEnabled() && isFastModeAvailable()
  ? <Box><Text dimColor>{fastModeShortcut} to toggle fast mode</Text></Box>
  : null;
```

**Keybindings 定制条目**：
```typescript
const keybindingsElement = isKeybindingCustomizationEnabled()
  ? <Box><Text dimColor>/keybindings to customize</Text></Box>
  : null;
```

### 3.5 React Compiler 编译特征

源码经过 React Compiler 编译，使用 `_c(99)` 大容量 memo cache 数组。每个帮助条目都被独立缓存，只有依赖的快捷键字符串或 `dimColor` 变化时才会重新创建 JSX 节点。

---

## 4. 关键代码路径与文件引用

### 4.1 组件入口

| 路径 | 说明 |
|------|------|
| `src/components/PromptInput/PromptInputHelpMenu.tsx` | 主组件文件（编译后输出） |
| `src/components/PromptInput/PromptInputFooter.tsx` | 直接调用方，在 `helpOpen` 为 true 时渲染 |
| `src/components/PromptInput/PromptInput.tsx` | 管理 `helpOpen` 状态，处理 `?` 键切换 |
| `src/components/HelpV2/General.tsx` | HelpV2 中也引用了类似的帮助内容结构 |
| `src/commands/help/help.tsx` | 帮助命令页面 |

### 4.2 核心依赖

| 路径 | 说明 |
|------|------|
| `src/keybindings/useShortcutDisplay.ts` | 读取用户自定义快捷键显示文本 |
| `src/keybindings/loadUserBindings.ts` | 提供 `isKeybindingCustomizationEnabled()` |
| `src/utils/platform.ts` | `getPlatform()` 用于平台判断 |
| `src/utils/fastMode.ts` | `isFastModeAvailable()` / `isFastModeEnabled()` |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE()` |
| `src/components/PromptInput/utils.ts` | `getNewlineInstructions()` |
| `src/ink.js` | `Box`、`Text` 组件 |

---

## 5. 依赖与外部交互

### 5.1 快捷键系统交互

```
PromptInputHelpMenu.tsx
  → useShortcutDisplay(action, context, fallback)
    → useOptionalKeybindingContext()
      → KeybindingContext.tsx
```

`useShortcutDisplay` 在 fallback 被使用时还会通过 `useEffect` 向 analytics 发送 `tengu_keybinding_fallback_used` 事件。

### 5.2 Feature Flag 交互

- `feature("TERMINAL_PANEL")`：Bun bundle 编译时 feature flag
- `getFeatureValue_CACHED_MAY_BE_STALE("tengu_terminal_panel", false)`：GrowthBook 运行时 flag
- `feature("KAIROS")` / `feature("KAIROS_BRIEF")`：其他编译时 feature（虽然本组件未直接使用，但同目录其他组件使用）

### 5.3 平台适配

```typescript
const suspendElement = getPlatform() !== "windows" 
  ? <Box><Text dimColor>ctrl + z to suspend</Text></Box>
  : null;
```

Windows 平台没有 Unix 风格的 `SIGTSTP` 挂起机制，因此隐藏该提示。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 |
|------|------|
| **编译后代码极度冗长** | `_c(99)` 和 99 个缓存槽位使源码长达 358 行，几乎不可手工维护 |
| **硬编码的栏目宽度** | 左栏 24、中栏 35 的固定宽度在超窄终端下可能溢出 |
| **帮助文本分散** | 帮助条目字符串散落在组件各处，国际化或统一修改困难 |
| **cycleMode 文案死代码** | `false ? "to cycle modes" : "to auto-accept edits"` 中 `"to cycle modes"` 分支永远不会执行，但保留了字符串字面量 |
| **无测试覆盖** | 未找到针对该组件的单元测试 |

### 6.2 边界情况

- **快捷键上下文缺失**：如果 `KeybindingContext` 未加载，`useShortcutDisplay` 会回退到默认值，并通过 analytics 记录一次
- **GrowthBook 缓存可能过期**：`getFeatureValue_CACHED_MAY_BE_STALE` 提示该值可能不是最新的，但帮助菜单对实时性要求不高
- **平台识别失败**：`getPlatform()` 返回未知值时，suspend 提示仍会显示（只要不是 `"windows"`）

### 6.3 改进建议

1. **提取帮助数据结构**：将三栏帮助条目提取为纯数据数组（如 `HELP_MENU_SECTIONS`），渲染逻辑与内容解耦
2. **响应式栏目宽度**：根据 `columns` 动态计算栏目宽度，避免在窄终端下截断
3. **移除死代码**：清理 `false ? "to cycle modes" : "to auto-accept edits"` 中的不可达分支
4. **增加快照测试**：由于输出是静态文本，适合用快照测试验证不同 feature flag 组合下的渲染结果
5. **考虑 i18n 支持**：如果未来需要多语言，应将帮助文本集中管理并支持翻译键
