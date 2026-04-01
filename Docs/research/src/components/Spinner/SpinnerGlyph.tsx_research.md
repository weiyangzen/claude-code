# SpinnerGlyph.tsx 研究文档

## 场景与职责

`SpinnerGlyph.tsx` 是 Spinner 组件体系中负责**旋转动画图形**的专用组件。它通过循环显示一系列 Unicode 符号（✢ ✳ ✶ ✻ ✽）来创建经典的"旋转"加载指示器效果。

### 视觉设计

- **标准模式**: 循环显示装饰性 Unicode 符号序列
- **Reduced Motion 模式**: 显示静态圆点，以 2 秒周期淡入淡出
- **停滞警告**: 颜色逐渐变红，提示响应停滞

---

## 功能点目的

### 1. 帧动画旋转
通过 `frame` 参数索引到预定义的符号数组，创建旋转效果：
```typescript
const spinnerChar = SPINNER_FRAMES[frame % SPINNER_FRAMES.length];
```

### 2. 平台适配字符集
根据终端类型和操作系统选择最优字符：
- **Ghostty 终端**: 使用 `*` 替代 `✽`（避免偏移渲染问题）
- **macOS**: 使用完整装饰字符集
- **其他平台**: 使用简化字符集（部分字符用 `*` 替代）

### 3. 无障碍支持
- **Reduced Motion**: 静态圆点 + 淡入淡出效果
- **停滞检测**: 颜色变化提供非动画状态指示

---

## 具体技术实现

### 关键流程

```
Props 输入 (frame, messageColor, stalledIntensity, reducedMotion, time)
    ↓
判断 reducedMotion?
    ├─ 是 → 显示静态圆点，计算淡入淡出
    ↓
判断 stalledIntensity > 0?
    ├─ 是 → 颜色插值到 ERROR_RED
    ↓
标准旋转动画
    ↓
渲染 <Box><Text>{spinnerChar}</Text></Box>
```

### 数据结构

```typescript
// Props 定义
type Props = {
  frame: number;                    // 当前动画帧索引
  messageColor: keyof Theme;        // 基础颜色键
  stalledIntensity?: number;        // 停滞强度 (0-1)
  reducedMotion?: boolean;          // 减少动画偏好
  time?: number;                    // 动画时间戳
}

// 字符集（平台适配）
const DEFAULT_CHARACTERS = getDefaultCharacters();
// macOS: ['·', '✢', '✳', '✶', '✻', '✽']
// Other: ['·', '✢', '*', '✶', '✻', '✽']
// Ghostty: ['·', '✢', '✳', '✶', '✻', '*']

// 动画帧序列（正序+倒序，形成循环）
const SPINNER_FRAMES = [
  ...DEFAULT_CHARACTERS,
  ...[...DEFAULT_CHARACTERS].reverse()
];
// 结果: ['·', '✢', '✳', '✶', '✻', '✽', '✽', '✻', '✶', '✳', '✢', '·']

// Reduced Motion 模式
const REDUCED_MOTION_DOT = '●';
const REDUCED_MOTION_CYCLE_MS = 2000; // 2秒周期

// 错误红色
const ERROR_RED = { r: 171, g: 43, b: 63 };
```

### 核心算法

**1. 平台适配字符集**
```typescript
export function getDefaultCharacters(): string[] {
  if (process.env.TERM === 'xterm-ghostty') {
    return ['·', '✢', '✳', '✶', '✻', '*']; // * 替代 ✽
  }
  return process.platform === 'darwin'
    ? ['·', '✢', '✳', '✶', '✻', '✽']
    : ['·', '✢', '*', '✶', '✻', '✽'];
}
```

**2. 帧索引计算**
```typescript
const spinnerChar = SPINNER_FRAMES[frame % SPINNER_FRAMES.length];
```
- `frame` 来自父组件的 `Math.floor(time / 120)`
- 120ms 每帧，约 8.3fps

**3. Reduced Motion 淡入淡出**
```typescript
const isDim = Math.floor(time / (REDUCED_MOTION_CYCLE_MS / 2)) % 2 === 1;
// 0-1000ms: 正常亮度
// 1000-2000ms: dim 状态
```

**4. 停滞颜色插值**
```typescript
if (stalledIntensity > 0) {
  const baseColorStr = theme[messageColor];
  const baseRGB = baseColorStr ? parseRGB(baseColorStr) : null;
  if (baseRGB) {
    const interpolated = interpolateColor(baseRGB, ERROR_RED, stalledIntensity);
    return <Text color={toRGBColor(interpolated)}>{spinnerChar}</Text>;
  }
  // ANSI 回退
  const color = stalledIntensity > 0.5 ? "error" : messageColor;
}
```

### React Compiler 优化

编译后代码使用 9 槽位缓存：
```typescript
const $ = _c(9);

// Reduced Motion 分支: 槽位 0-2
if ($[0] !== isDim || $[1] !== messageColor) {
  t4 = <Box ...><Text color={messageColor} dimColor={isDim}>{REDUCED_MOTION_DOT}</Text></Box>;
  $[0] = isDim;
  $[1] = messageColor;
  $[2] = t4;
}

// Stalled 分支: 槽位 3-5
if ($[3] !== color || $[4] !== spinnerChar) {
  // ...
  $[5] = t4;
}

// 正常分支: 槽位 6-8
if ($[6] !== messageColor || $[7] !== spinnerChar) {
  // ...
  $[8] = t4;
}
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../../ink.js` | `Box`, `Text`, `useTheme` | Ink UI 组件和主题钩子 |
| `../../utils/theme.js` | `getTheme`, `Theme` | 主题颜色解析 |
| `./utils.js` | `getDefaultCharacters`, `interpolateColor`, `parseRGB`, `toRGBColor` | 字符集和颜色工具 |

### 被调用方

| 文件路径 | 使用方式 |
|---------|---------|
| `SpinnerAnimationRow.tsx` | 主要调用方，传递 frame 和 time |
| 其他 Spinner 组件 | 可能直接使用 |

### 源码位置

```
src/components/Spinner/
├── SpinnerGlyph.tsx         # 本文件
├── SpinnerAnimationRow.tsx  # 主要调用方
├── utils.ts                 # 字符集和颜色工具
└── index.ts                 # 模块导出
```

---

## 依赖与外部交互

### 依赖关系图

```
SpinnerGlyph.tsx
    ├─ import { Box, Text, useTheme } from '../../ink.js'
    │       ├─ Box: 布局容器 (flexWrap, height, width)
    │       ├─ Text: 文本渲染 (color, dimColor)
    │       └─ useTheme: 获取当前主题
    │
    ├─ import { getTheme } from '../../utils/theme.js'
    │       └─ 主题颜色解析
    │
    └─ import { getDefaultCharacters, ... } from './utils.js'
            ├─ getDefaultCharacters: 平台适配字符集
            ├─ interpolateColor: RGB 插值
            ├─ parseRGB: 解析 rgb() 字符串
            └─ toRGBColor: 转回 rgb() 字符串
```

### 与 utils.ts 的协作

```typescript
// utils.ts
export function getDefaultCharacters(): string[] {
  if (process.env.TERM === 'xterm-ghostty') {
    return ['·', '✢', '✳', '✶', '✻', '*'];
  }
  return process.platform === 'darwin'
    ? ['·', '✢', '✳', '✶', '✻', '✽']
    : ['·', '✢', '*', '✶', '✻', '✽'];
}
```

**设计决策**:
- Ghostty 终端: `✽` 渲染偏移，用 `*` 替代
- 非 macOS: `✳` 可能显示不正确，用 `*` 替代

---

## 风险、边界与改进建议

### 已知风险

1. **平台检测可靠性**
   - `process.env.TERM` 和 `process.platform` 可能不准确
   - 某些终端模拟器可能不报告标准 TERM 值
   - 远程 SSH 会话可能传递错误的平台信息

2. **字符渲染一致性**
   - Unicode 装饰字符在不同字体中宽度不一致
   - 某些终端可能不支持这些字符，显示为方框或空格
   - 双宽字符可能导致布局错位

3. **类型文件缺失**
   - 同其他 Spinner 组件，`types.ts` 缺失

### 边界情况

| 场景 | 行为 |
|-----|------|
| `frame = 0` | 显示第一个字符 '·' |
| `frame` 很大 | 取模循环，正常显示 |
| `reducedMotion = true` | 显示 '●'，淡入淡出 |
| `stalledIntensity = 1` | 完全红色 |
| `stalledIntensity = 0.5` | 中间颜色 |
| 颜色解析失败 | 使用 "error" 或 messageColor |

### 改进建议

1. **字符集配置化**
   ```typescript
   const SPINNER_CHARSETS = {
     default: ['·', '✢', '✳', '✶', '✻', '✽'],
     conservative: ['|', '/', '-', '\\'],  // ASCII 回退
     minimal: ['·', '•', '◦'],
   };
   ```

2. **运行时字符检测**
   ```typescript
   function detectCharacterSupport(): string[] {
     // 尝试渲染测试字符，检测实际显示宽度
     // 返回最适合当前终端的字符集
   }
   ```

3. **动画参数可配置**
   ```typescript
   interface SpinnerGlyphConfig {
     frameDurationMs: number;      // 默认 120
     reducedMotionCycleMs: number; // 默认 2000
     characters?: string[];        // 自定义字符集
   }
   ```

4. **类型文件恢复**
   ```typescript
   // types.ts
   export type SpinnerGlyphProps = {
     frame: number;
     messageColor: keyof Theme;
     stalledIntensity?: number;
     reducedMotion?: boolean;
     time?: number;
   };
   ```

5. **测试覆盖**
   - 各平台字符集渲染测试
   - 颜色插值精度测试
   - Reduced Motion 模式行为测试

---

## 附录：源码与编译后对比

### 原始源码（约 50 行）

```typescript
import * as React from 'react'
import { Box, Text, useTheme } from '../../ink.js'
import { getTheme, type Theme } from '../../utils/theme.js'
import {
  getDefaultCharacters,
  interpolateColor,
  parseRGB,
  toRGBColor,
} from './utils.js'

const DEFAULT_CHARACTERS = getDefaultCharacters()
const SPINNER_FRAMES = [...DEFAULT_CHARACTERS, ...[...DEFAULT_CHARACTERS].reverse()]
const REDUCED_MOTION_DOT = '●'
const REDUCED_MOTION_CYCLE_MS = 2000
const ERROR_RED = { r: 171, g: 43, b: 63 }

type Props = {
  frame: number
  messageColor: keyof Theme
  stalledIntensity?: number
  reducedMotion?: boolean
  time?: number
}

export function SpinnerGlyph({
  frame,
  messageColor,
  stalledIntensity = 0,
  reducedMotion = false,
  time = 0,
}: Props): React.ReactNode {
  const [themeName] = useTheme()
  const theme = getTheme(themeName)

  if (reducedMotion) {
    const isDim = Math.floor(time / (REDUCED_MOTION_CYCLE_MS / 2)) % 2 === 1
    return (
      <Box flexWrap="wrap" height={1} width={2}>
        <Text color={messageColor} dimColor={isDim}>{REDUCED_MOTION_DOT}</Text>
      </Box>
    )
  }

  const spinnerChar = SPINNER_FRAMES[frame % SPINNER_FRAMES.length]

  if (stalledIntensity > 0) {
    const baseColorStr = theme[messageColor]
    const baseRGB = baseColorStr ? parseRGB(baseColorStr) : null
    if (baseRGB) {
      const interpolated = interpolateColor(baseRGB, ERROR_RED, stalledIntensity)
      return (
        <Box flexWrap="wrap" height={1} width={2}>
          <Text color={toRGBColor(interpolated)}>{spinnerChar}</Text>
        </Box>
      )
    }
    const color = stalledIntensity > 0.5 ? "error" : messageColor
    return (
      <Box flexWrap="wrap" height={1} width={2}>
        <Text color={color}>{spinnerChar}</Text>
      </Box>
    )
  }

  return (
    <Box flexWrap="wrap" height={1} width={2}>
      <Text color={messageColor}>{spinnerChar}</Text>
    </Box>
  )
}
```

### 编译后特点

- 增加 React Compiler 缓存机制（9 槽位）
- 条件分支显式化为 `if/else` 链
- 添加 source map 注释

### 渲染示例

```typescript
// 标准模式，第 5 帧
<SpinnerGlyph frame={5} messageColor="claude" />
// 渲染: <Box><Text color="claude">✻</Text></Box>

// Reduced Motion 模式，时间 1500ms
<SpinnerGlyph frame={0} messageColor="claude" reducedMotion time={1500} />
// 渲染: <Box><Text color="claude" dimColor>●</Text></Box>

// 停滞状态，50% 强度
<SpinnerGlyph frame={3} messageColor="claude" stalledIntensity={0.5} />
// 渲染: 插值后的中间颜色
```
