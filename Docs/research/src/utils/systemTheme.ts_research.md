# systemTheme.ts 研究文档

## 场景与职责

`systemTheme.ts` 负责终端暗/亮模式检测，用于支持 `'auto'` 主题设置。与基于操作系统外观设置的检测不同，该模块通过查询终端的实际背景颜色（通过 OSC 11）来确定主题，确保即使在浅色模式的 OS 上使用深色终端也能正确识别。

## 功能点目的

### 终端主题检测
- **目标**: 检测终端当前使用的是深色还是浅色主题
- **方法**: 解析 OSC 11 查询返回的背景色
- **回退**: 使用 `COLORFGBG` 环境变量进行同步初始猜测

### 缓存机制
- **目的**: 避免重复的异步 OSC 往返
- **实现**: 模块级缓存，由 watcher 更新
- **优势**: 调用者可以同步解析 `'auto'` 设置

## 具体技术实现

### 核心类型

```typescript
export type SystemTheme = 'dark' | 'light'
```

### 缓存管理

```typescript
let cachedSystemTheme: SystemTheme | undefined

export function getSystemThemeName(): SystemTheme {
  if (cachedSystemTheme === undefined) {
    cachedSystemTheme = detectFromColorFgBg() ?? 'dark'
  }
  return cachedSystemTheme
}

export function setCachedSystemTheme(theme: SystemTheme): void {
  cachedSystemTheme = theme
}
```

### OSC 颜色解析

支持两种格式：
1. **XParseColor 格式**: `rgb:RRRR/GGGG/BBBB`（1-4 位十六进制）
2. **十六进制格式**: `#RRGGBB` 或 `#RRRRGGGGBBBB`

```typescript
export function themeFromOscColor(data: string): SystemTheme | undefined {
  const rgb = parseOscRgb(data)
  if (!rgb) return undefined
  
  // ITU-R BT.709 相对亮度计算
  const luminance = 0.2126 * rgb.r + 0.7152 * rgb.g + 0.0722 * rgb.b
  return luminance > 0.5 ? 'light' : 'dark'
}
```

### 亮度计算

使用 ITU-R BT.709 标准的相对亮度公式：
- `luminance = 0.2126 * R + 0.7152 * G + 0.0722 * B`
- 阈值：> 0.5 为浅色，≤ 0.5 为深色

### COLORFGBG 回退

```typescript
function detectFromColorFgBg(): SystemTheme | undefined {
  const colorfgbg = process.env['COLORFGBG']
  if (!colorfgbg) return undefined
  
  const parts = colorfgbg.split(';')
  const bg = parts[parts.length - 1]
  const bgNum = Number(bg)
  
  // rxvt 约定：0-6 和 8 是深色，7 和 9-15 是浅色
  return bgNum <= 6 || bgNum === 8 ? 'dark' : 'light'
}
```

## 关键代码路径与文件引用

### 本文件导出
- `getSystemThemeName()`: 获取当前主题（缓存）
- `setCachedSystemTheme(theme)`: 更新缓存
- `resolveThemeSetting(setting)`: 将主题设置解析为具体主题名
- `themeFromOscColor(data)`: 从 OSC 颜色数据解析主题

### 依赖模块

| 模块 | 用途 |
|------|------|
| `./theme.js` | `ThemeName`, `ThemeSetting` 类型 |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/QueryEngine.ts` | 主题解析 |
| `src/components/FastIcon.tsx` | 图标主题 |
| `src/components/LogoV2/LogoV2.tsx` | Logo 主题 |
| `src/components/design-system/ThemeProvider.tsx` | 主题提供 |
| `src/components/Stats.tsx` | 统计组件主题 |

### 调用代码片段

```typescript
// 解析主题设置
const themeName = resolveThemeSetting(settings.theme)  // 'auto' -> 'dark' | 'light'

// 获取当前系统主题
const currentTheme = getSystemThemeName()
```

## 依赖与外部交互

### 与 Theme Watcher 的集成
- `systemThemeWatcher.ts` 查询 OSC 11 并调用 `setCachedSystemTheme`
- 非 React 调用点通过缓存保持同步
- 缓存初始值为 `undefined`，首次调用时回退到 `COLORFGBG` 或 `'dark'`

### 与主题系统的集成
- 接收 `ThemeSetting` 类型（`'auto'` 或具体主题名）
- 返回 `ThemeName` 类型（`'dark'` 或 `'light'`）
- 与 `theme.ts` 中的主题定义配合使用

### 与终端的集成
- 依赖终端支持 OSC 11 查询
- 支持 xterm、iTerm2、Terminal.app、Ghostty、kitty、Alacritty 等
- 某些终端可能返回 `rgba:` 格式（带 alpha），代码正确处理

## 风险、边界与改进建议

### 潜在风险

1. **终端不支持**: 某些终端可能不支持 OSC 11，只能依赖 `COLORFGBG`
2. **颜色解析失败**: 非标准格式的颜色响应可能解析失败
3. **亮度阈值**: 0.5 的阈值可能对某些颜色产生误判

### 边界情况

1. **透明背景**: 终端使用透明背景时，OSC 11 返回的颜色可能不代表实际观感
2. **动态主题**: 终端动态切换主题时，缓存可能过期
3. **SSH 会话**: 通过 SSH 连接时，环境变量可能不准确

### 改进建议

1. **更多回退机制**: 添加更多检测方法
```typescript
function detectFromTermProgram(): SystemTheme | undefined {
  const term = process.env.TERM_PROGRAM
  if (term === 'Apple_Terminal') {
    // 使用 macOS 特定 API
  }
  return undefined
}
```

2. **置信度评分**: 返回检测置信度
```typescript
export type ThemeDetectionResult = {
  theme: SystemTheme
  confidence: 'high' | 'medium' | 'low'
  source: 'osc11' | 'colorfgbg' | 'default'
}
```

3. **定期刷新**: 添加定期刷新机制
```typescript
setInterval(() => {
  cachedSystemTheme = undefined  // 强制下次重新检测
}, 60000)
```

4. **用户覆盖**: 允许用户手动指定系统主题
```typescript
export function setManualSystemTheme(theme: SystemTheme | null): void {
  manualOverride = theme
}
```

5. **更多颜色空间**: 支持更多颜色格式
```typescript
// HSL、Lab 等颜色空间的解析
function parseHslColor(data: string): Rgb | undefined
function parseLabColor(data: string): Rgb | undefined
```

6. **对比度计算**: 不仅检测亮/暗，还计算对比度
```typescript
export function getContrastRatio(fg: Rgb, bg: Rgb): number
```
