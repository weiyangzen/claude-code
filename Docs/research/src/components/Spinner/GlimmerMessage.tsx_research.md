# GlimmerMessage.tsx 研究文档

## 场景与职责

`GlimmerMessage.tsx` 是 Spinner 组件体系中负责**消息文本动态渲染**的核心组件。它根据当前 Spinner 模式（mode）和动画状态，为消息文本提供多种视觉效果：微光扫过效果、停滞变红警告、工具使用闪烁等。

### 使用场景

1. **标准微光效果 (Glimmer)**: 在 requesting/responding 模式下，微光高亮在消息文本上扫过
2. **停滞警告 (Stalled)**: 当响应停滞超过 3 秒，消息逐渐变红提示用户
3. **工具闪烁 (Tool Flash)**: 在 tool-use 模式下，整行消息平滑闪烁
4. **队友状态显示**: 显示正在运行的 teammate 代理的名称和进度

---

## 功能点目的

### 1. 多模式渲染策略
根据 `mode` 和 `stalledIntensity` 决定渲染路径：

| 条件 | 渲染效果 |
|-----|---------|
| `stalledIntensity > 0` | 红色渐变警告 |
| `mode === 'tool-use'` | 整行颜色闪烁 |
| 其他模式 | 微光扫过高亮 |

### 2. Unicode 字素分割
使用 `Intl.Segmenter` 正确处理多字节字符（如 emoji、CJK 文字），确保微光效果按"视觉字符"而非字节定位。

### 3. 宽度感知布局
通过 `stringWidth` 计算字符显示宽度，处理东亚双宽字符、零宽字符等特殊情况。

---

## 具体技术实现

### 关键流程

```
Props 输入 (message, mode, glimmerIndex, flashOpacity, stalledIntensity)
    ↓
字素分割 → 计算消息宽度
    ↓
判断渲染路径:
    ├─ stalledIntensity > 0 → 红色插值渲染
    ├─ mode === 'tool-use' → FlashingChar 风格闪烁
    └─ 其他 → 微光分段渲染
    ↓
输出 <Text> 组件树
```

### 数据结构

```typescript
// Props 定义
type Props = {
  message: string;              // 要渲染的消息文本
  mode: SpinnerMode;            // 当前 spinner 模式
  messageColor: keyof Theme;    // 基础颜色键
  glimmerIndex: number;         // 微光中心位置索引
  flashOpacity: number;         // 闪烁强度 (0-1)
  shimmerColor: keyof Theme;    // 微光/闪烁颜色键
  stalledIntensity?: number;    // 停滞强度 (0-1, 可选)
}

// 字素段信息
interface SegmentInfo {
  segment: string;  // 字素字符串
  width: number;    // 显示宽度
}

// 错误红色常量
const ERROR_RED = { r: 171, g: 43, b: 63 }
```

### 核心算法

**1. 微光分段渲染**
```typescript
const shimmerStart = glimmerIndex - 1;
const shimmerEnd = glimmerIndex + 1;

// 将消息分为三部分
let before = "";  // 微光前
let shim = "";    // 微光中（使用 shimmerColor）
let after = "";   // 微光后

for (const { segment, width } of segments) {
  if (colPos + width <= clampedStart) {
    before += segment;
  } else if (colPos > shimmerEnd) {
    after += segment;
  } else {
    shim += segment;
  }
  colPos += width;
}
```

**2. 停滞红色插值**
```typescript
const baseColorStr = theme[messageColor];
const baseRGB = baseColorStr ? parseRGB(baseColorStr) : null;
if (baseRGB) {
  const interpolated = interpolateColor(baseRGB, ERROR_RED, stalledIntensity);
  return <Text color={toRGBColor(interpolated)}>{message}</Text>;
}
// ANSI 回退
const color = stalledIntensity > 0.5 ? "error" : messageColor;
```

**3. Tool-use 模式闪烁**
```typescript
if (baseRGB_0 && shimmerRGB) {
  const interpolated_0 = interpolateColor(baseRGB_0, shimmerRGB, flashOpacity);
  return <Text color={toRGBColor(interpolated_0)}>{message}</Text>;
}
// 二进制回退
const color_1 = flashOpacity > 0.5 ? shimmerColor : messageColor;
```

### 性能优化

**1. 字素分割缓存**
```typescript
// 使用 React Compiler 缓存
if ($[10] !== message) {
  segs = [];
  for (const { segment } of getGraphemeSegmenter().segment(message)) {
    segs.push({ segment, width: stringWidth(segment) });
  }
  $[10] = message;
  $[11] = segs;
}
```

**2. 消息宽度缓存**
```typescript
if ($[12] !== message) {
  t3 = stringWidth(message);
  $[12] = message;
  $[13] = t3;
}
```

**3. 分段字符串缓存**
微光范围计算后的 `before`、`shim`、`after` 字符串也被缓存，避免每帧重新拼接。

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../../ink/stringWidth.js` | `stringWidth` | 计算字符显示宽度 |
| `../../ink.js` | `Text`, `useTheme` | Ink UI 组件 |
| `../../utils/intl.js` | `getGraphemeSegmenter` | Unicode 字素分割 |
| `../../utils/theme.js` | `getTheme`, `Theme` | 主题颜色解析 |
| `./types.js` | `SpinnerMode` | 模式类型定义 |
| `./utils.js` | `interpolateColor`, `parseRGB`, `toRGBColor` | 颜色工具 |

### 被调用方

| 文件路径 | 使用方式 |
|---------|---------|
| `SpinnerAnimationRow.tsx` | 主要调用方，传递动画状态参数 |
| `TeammateSpinnerLine.tsx` | 队友状态行渲染 |

### 源码位置

```
src/components/Spinner/
├── GlimmerMessage.tsx       # 本文件
├── SpinnerAnimationRow.tsx  # 主要调用方
├── utils.ts                 # 颜色工具
├── types.ts                 # 类型定义（缺失）
└── index.ts                 # 模块导出
```

---

## 依赖与外部交互

### 核心依赖详解

**1. Intl.Segmenter (字素分割)**
```typescript
import { getGraphemeSegmenter } from '../../utils/intl.js';

// 使用方式
for (const { segment } of getGraphemeSegmenter().segment(message)) {
  // segment 是一个"视觉字符"，可能包含多个 Unicode 码点
}
```
- 支持 emoji、组合字符、CJK 文字的正确分割
- 懒加载单例模式，避免重复创建

**2. stringWidth (宽度计算)**
```typescript
import { stringWidth } from '../../ink/stringWidth.js';

// 区分全角(2)和半角(1)字符
// 处理零宽字符、控制字符
```

**3. 颜色工具链**
```typescript
import { interpolateColor, parseRGB, toRGBColor } from './utils.js';

// parseRGB: "rgb(255,128,0)" → {r:255, g:128, b:0}
// interpolateColor: 两个 RGB 之间线性插值
// toRGBColor: {r,g,b} → "rgb(r,g,b)"
```

### 数据流

```
SpinnerAnimationRow
    ↓ 传递
GlimmerMessage Props
    ├─ message: string
    ├─ mode: SpinnerMode
    ├─ glimmerIndex: number (来自 useShimmerAnimation)
    ├─ flashOpacity: number (来自 SpinnerAnimationRow 计算)
    ├─ stalledIntensity: number (来自 useStalledAnimation)
    ↓
渲染输出
```

---

## 风险、边界与改进建议

### 已知风险

1. **类型文件缺失**
   - `src/components/Spinner/types.ts` 不存在
   - `SpinnerMode` 类型通过 `./types.js` 导入，但实际文件缺失
   - 依赖 TypeScript 的模块解析容错或编译后文件

2. **微光索引越界**
   ```typescript
   const shimmerStart = glimmerIndex - 1;
   const shimmerEnd = glimmerIndex + 1;
   if (shimmerStart >= messageWidth || shimmerEnd < 0) {
     // 越界处理：返回普通文本
   }
   ```
   - 当 `glimmerIndex = -100` (停滞时) 会触发越界处理
   - 逻辑正确但依赖魔法数字

3. **性能瓶颈**
   - 每帧遍历所有字素段（虽然缓存了分割结果）
   - 长消息（>100 字符）可能影响性能

### 边界情况

| 场景 | 行为 |
|-----|------|
| 空消息 | 返回 `null` |
| `glimmerIndex` 越界 | 返回普通文本（无微光） |
| 颜色解析失败 | 使用主题回退颜色 |
| 单字符消息 | 整个字符高亮 |
| Emoji 字符 | 正确识别为单个字素 |

### 改进建议

1. **类型安全**
   - 创建 `src/components/Spinner/types.ts`:
   ```typescript
   export type SpinnerMode = 'requesting' | 'thinking' | 'tool-use' | 'tool-input' | 'responding';
   export type RGBColor = { r: number; g: number; b: number };
   ```

2. **性能优化**
   - 对超长消息截断处理
   - 使用虚拟列表只渲染可见部分
   - 考虑 Web Worker 处理字素分割

3. **可维护性**
   - 将渲染策略提取为策略模式
   ```typescript
   const renderStrategies = {
     stalled: StalledRenderer,
     toolUse: ToolUseRenderer,
     glimmer: GlimmerRenderer,
   };
   ```

4. **测试覆盖**
   - Unicode 字符渲染测试（emoji、CJK、RTL）
   - 所有主题的颜色对比度测试
   - 性能基准测试（长消息渲染时间）

5. **无障碍增强**
   - 为停滞状态添加屏幕阅读器提示
   - 提供非颜色指示器（如图标变化）

---

## 附录：编译后代码结构

原始源码约 80 行，编译后约 328 行，主要增加：
- React Compiler 缓存逻辑（`$` 数组操作）
- 条件分支的显式处理
- Source map 注释

关键编译模式：
```typescript
// 缓存检查模式
if ($[0] !== flashOpacity || $[1] !== message || ...) {
  // 重新计算
  $[0] = flashOpacity;
  // ...
} else {
  // 复用缓存
}
```
