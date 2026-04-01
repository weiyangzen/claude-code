# theme.ts 研究文档

## 场景与职责

`theme.ts` 是 Claude Code CLI 的主题系统核心模块，定义了完整的颜色主题体系。该模块负责：
1. 定义主题颜色类型和常量
2. 提供 6 种预设主题（dark/light/dark-daltonized/light-daltonized/dark-ansi/light-ansi）
3. 主题切换和颜色转换工具

主题系统支持：
- **真彩色（24-bit RGB）**：现代终端的标准显示模式
- **ANSI 16 色**：兼容性模式，用于不支持真彩色的终端
- **色盲友好（Daltonized）**：针对红绿色盲优化的配色方案

## 功能点目的

### 1. 统一的颜色定义
通过 `Theme` 类型定义 80+ 个语义化颜色变量，涵盖：
- 品牌色（claude, permission, planMode）
- 语义色（success, error, warning）
- Diff 色（diffAdded, diffRemoved）
- Agent 专用色（*_FOR_SUBAGENTS_ONLY）
- TUI V2 颜色（userMessageBackground, selectionBg 等）
- 彩虹色（用于 ultrathink 关键词高亮）

### 2. 多主题支持
提供 6 种预设主题，满足不同用户需求：
- `dark`/`light`：标准明暗主题（RGB）
- `dark-ansi`/`light-ansi`：ANSI 16 色兼容模式
- `dark-daltonized`/`light-daltonized`：色盲友好主题

### 3. Apple Terminal 兼容
特殊处理 Apple Terminal 的 256 色限制：
```typescript
const chalkForChart = env.terminal === 'Apple_Terminal'
  ? new Chalk({ level: 2 })  // 256 colors
  : chalk                    // 24-bit true color
```

## 具体技术实现

### Theme 类型定义

```typescript
export type Theme = {
  // 品牌色
  autoAccept: string
  bashBorder: string
  claude: string
  claudeShimmer: string
  // ... 80+ 个颜色变量
  
  // 彩虹色（ultrathink 高亮）
  rainbow_red: string
  rainbow_orange: string
  // ...
  rainbow_red_shimmer: string
  // ...
}
```

### 主题常量

```typescript
export const THEME_NAMES = [
  'dark',
  'light', 
  'light-daltonized',
  'dark-daltonized',
  'light-ansi',
  'dark-ansi',
] as const

export type ThemeName = (typeof THEME_NAMES)[number]
export type ThemeSetting = 'auto' | ThemeName
```

### 主题定义示例

**Dark 主题**（真彩色）：
```typescript
const darkTheme: Theme = {
  autoAccept: 'rgb(175,135,255)',      // Electric violet
  bashBorder: 'rgb(253,93,177)',       // Bright pink
  claude: 'rgb(215,119,87)',           // Claude orange
  claudeShimmer: 'rgb(235,159,127)',   // Lighter orange
  // ...
  userMessageBackground: 'rgb(55, 55, 55)',
  selectionBg: 'rgb(38, 79, 120)',     // VS Code dark style
}
```

**Dark ANSI 主题**：
```typescript
const darkAnsiTheme: Theme = {
  autoAccept: 'ansi:magentaBright',
  bashBorder: 'ansi:magentaBright',
  claude: 'ansi:redBright',
  // ...
}
```

**Daltonized 主题设计原则**：
- 避免红绿对比（deuteranopia 最常见）
- 使用蓝/黄对比替代
- Success 色使用蓝色而非绿色
- 调整橙色/红色以更好区分

### 主题获取函数

```typescript
export function getTheme(themeName: ThemeName): Theme {
  switch (themeName) {
    case 'light': return lightTheme
    case 'light-ansi': return lightAnsiTheme
    case 'dark-ansi': return darkAnsiTheme
    case 'light-daltonized': return lightDaltonizedTheme
    case 'dark-daltonized': return darkDaltonizedTheme
    default: return darkTheme
  }
}
```

### 颜色转换工具

```typescript
export function themeColorToAnsi(themeColor: string): string
```

**功能**：将 `rgb(r,g,b)` 格式转换为 ANSI 转义序列

**实现**：
1. 正则解析 RGB 值
2. 使用 chalk.rgb 生成带颜色代码的标记字符串
3. 提取转义序列前缀

**Apple Terminal 特殊处理**：
- 检测到 `Apple_Terminal` 时使用 `level: 2`（256 色）
- chalk 自动将 24-bit RGB 降级为 256 色

## 关键代码路径与文件引用

### 调用方（被谁使用）

该模块被 50+ 个文件引用，主要包括：

| 类别 | 代表文件 |
|-----|---------|
| 组件 | `Spinner.tsx`, `ThemePicker.tsx`, `ThemedBox.tsx`, `ThemedText.tsx` |
| 工具 | `textHighlighting.ts`, `markdown.ts`, `treeify.ts` |
| Agent 系统 | `agentColorManager.ts`, `AgentTool/UI.tsx` |
| 权限系统 | `PermissionDialog.tsx`, `PermissionRequestTitle.tsx` |
| 设置 | `supportedSettings.ts`, `config.ts` |
| 状态显示 | `Status.tsx`, `Stats.tsx` |
| 消息渲染 | `AssistantToolUseMessage.tsx`, `AttachmentMessage.tsx` |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `chalk` | ANSI 颜色生成 |
| `./env.js` | 检测终端类型（Apple Terminal） |

## 依赖与外部交互

### 与 Chalk 的集成

Chalk 是本模块唯一的运行时依赖，用于：
1. 生成 ANSI 转义序列
2. 自动降级（24-bit → 256色 → 16色）

```typescript
// 示例：将 rgb(255,0,0) 转换为 \x1b[38;2;255;0;0m
const colored = chalk.rgb(255, 0,0)('X')
const ansiCode = colored.slice(0, colored.indexOf('X'))
// 结果：\x1b[38;2;255;0;0m
```

### 与系统主题的集成

通过 `systemTheme.ts` 检测系统明暗模式：
- macOS: `defaults read -g AppleInterfaceStyle`
- Windows: 注册表查询
- Linux: `XDG_CURRENT_DESKTOP` + `gsettings`

### 色盲友好设计

Daltonized 主题针对 deuteranopia（绿色盲，最常见）优化：

| 标准主题 | Daltonized 主题 | 原因 |
|---------|----------------|------|
| `success: green` | `success: blue` | 绿盲难以区分红绿 |
| `diffAdded: green` | `diffAdded: blue` | 使用蓝/黄对比 |
| `warning: amber` | `warning: orange` | 调整色调避免与绿色混淆 |

## 风险、边界与改进建议

### 潜在风险

1. **颜色一致性**
   - 80+ 个颜色变量，维护一致性困难
   - shimmer 变体需要手动计算（当前是人工指定的 RGB）

2. **终端兼容性**
   - ANSI 主题依赖终端的 16 色定义
   - 用户自定义的终端配色可能破坏预期效果

3. **Apple Terminal 检测**
   - 依赖 `env.terminal` 的检测准确性
   - 通过 SSH 或使用 tmux 时可能检测失败

4. **色盲主题局限性**
   - 仅针对 deuteranopia 优化
   - protanopia（红色盲）和 tritanopia（蓝色盲）支持不足

### 边界条件

| 场景 | 行为 |
|-----|------|
| 未知主题名 | 回退到 `darkTheme` |
| `themeColorToAnsi` 解析失败 | 回退到 `\x1b[35m`（洋红） |
| Apple Terminal 使用 RGB 格式 | 自动降级为 256 色 |
| 非 RGB/ANSI 格式 | 原样返回（调用方需处理） |

### 改进建议

1. **自动生成 shimmer 变体**
   ```typescript
   function generateShimmerColor(baseColor: string): string {
     // 解析 RGB，增加亮度 20-30%
     const [r, g, b] = parseRgb(baseColor)
     return `rgb(${Math.min(255, r + 40)}, ${Math.min(255, g + 40)}, ${Math.min(255, b + 40)})`
   }
   ```

2. **动态主题系统**
   ```typescript
   // 支持用户自定义主题
   export function registerCustomTheme(name: string, theme: Partial<Theme>): void
   ```

3. **更完善的色盲支持**
   ```typescript
   export const THEME_NAMES = [
     // ...现有主题
     'light-protanopia',
     'dark-protanopia', 
     'light-tritanopia',
     'dark-tritanopia',
   ]
   ```

4. **CSS 变量导出**
   ```typescript
   // 为 Web 组件导出 CSS 变量
   export function themeToCssVariables(theme: Theme): Record<string, string>
   // 结果：{ '--claude-color': '#d77757', ... }
   ```

5. **对比度检查**
   ```typescript
   // 确保所有颜色组合满足 WCAG 对比度要求
   function validateContrast(theme: Theme): ContrastReport
   ```

6. **主题预览**
   ```typescript
   // 生成主题预览图（ASCII art 或 Sixel）
   export function generateThemePreview(theme: Theme): string
   ```

### 测试建议

应覆盖以下场景：
- 所有 6 个主题的完整性（检查无 undefined 颜色）
- `getTheme` 的边界情况
- `themeColorToAnsi` 的 RGB 解析
- Apple Terminal 降级行为
- 色盲主题的视觉可区分性（可能需要视觉回归测试）
