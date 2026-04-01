# types.ts 研究报告

## 场景与职责

`types.ts` 是 termio 模块的类型定义中心，声明了所有语义类型和数据结构。这些类型代表 ANSI 转义序列的语义含义，而非其字符串表示，设计灵感来自 ghostty 的基于动作的设计。

该文件的核心职责：
1. **定义颜色类型** —— 命名色、索引色、RGB 真彩色
2. **定义文本样式** —— 粗体、斜体、下划线、颜色等属性
3. **定义光标动作** —— 移动、定位、保存/恢复、样式变更
4. **定义擦除/滚动/模式动作** —— 屏幕操作和终端模式
5. **定义链接/标题/标签状态动作** —— 超链接和元数据
6. **定义解析输出** —— 文本段、字素、动作联合类型
7. **提供工具函数** —— 默认样式、相等性比较

## 功能点目的

### 1. 颜色类型

```typescript
/** 16 色调色板的命名颜色 */
export type NamedColor =
  | 'black' | 'red' | 'green' | 'yellow'
  | 'blue' | 'magenta' | 'cyan' | 'white'
  | 'brightBlack' | 'brightRed' | 'brightGreen' | 'brightYellow'
  | 'brightBlue' | 'brightMagenta' | 'brightCyan' | 'brightWhite'

/** 颜色规范 - 可以是命名色、索引色（256）或 RGB */
export type Color =
  | { type: 'named'; name: NamedColor }
  | { type: 'indexed'; index: number }  // 0-255
  | { type: 'rgb'; r: number; g: number; b: number }
  | { type: 'default' }
```

### 2. 文本样式

```typescript
/** 下划线样式变体 */
export type UnderlineStyle =
  | 'none' | 'single' | 'double' | 'curly' | 'dotted' | 'dashed'

/** 文本样式属性 - 表示当前样式状态 */
export type TextStyle = {
  bold: boolean
  dim: boolean
  italic: boolean
  underline: UnderlineStyle
  blink: boolean
  inverse: boolean
  hidden: boolean
  strikethrough: boolean
  overline: boolean
  fg: Color
  bg: Color
  underlineColor: Color
}

/** 创建默认（重置）文本样式 */
export function defaultStyle(): TextStyle {
  return {
    bold: false, dim: false, italic: false,
    underline: 'none', blink: false, inverse: false,
    hidden: false, strikethrough: false, overline: false,
    fg: { type: 'default' },
    bg: { type: 'default' },
    underlineColor: { type: 'default' },
  }
}
```

### 3. 光标动作

```typescript
export type CursorDirection = 'up' | 'down' | 'forward' | 'back'

export type CursorAction =
  | { type: 'move'; direction: CursorDirection; count: number }
  | { type: 'position'; row: number; col: number }
  | { type: 'column'; col: number }
  | { type: 'row'; row: number }
  | { type: 'save' }
  | { type: 'restore' }
  | { type: 'show' }
  | { type: 'hide' }
  | { type: 'style'; style: 'block' | 'underline' | 'bar'; blinking: boolean }
  | { type: 'nextLine'; count: number }
  | { type: 'prevLine'; count: number }
```

### 4. 擦除动作

```typescript
export type EraseAction =
  | { type: 'display'; region: 'toEnd' | 'toStart' | 'all' | 'scrollback' }
  | { type: 'line'; region: 'toEnd' | 'toStart' | 'all' }
  | { type: 'chars'; count: number }
```

### 5. 滚动动作

```typescript
export type ScrollAction =
  | { type: 'up'; count: number }
  | { type: 'down'; count: number }
  | { type: 'setRegion'; top: number; bottom: number }
```

### 6. 模式动作

```typescript
export type ModeAction =
  | { type: 'alternateScreen'; enabled: boolean }
  | { type: 'bracketedPaste'; enabled: boolean }
  | { type: 'mouseTracking'; mode: 'off' | 'normal' | 'button' | 'any' }
  | { type: 'focusEvents'; enabled: boolean }
```

### 7. 链接动作（OSC 8）

```typescript
export type LinkAction =
  | { type: 'start'; url: string; params?: Record<string, string> }
  | { type: 'end' }
```

### 8. 标题动作（OSC 0/1/2）

```typescript
export type TitleAction =
  | { type: 'windowTitle'; title: string }
  | { type: 'iconName'; name: string }
  | { type: 'both'; title: string }
```

### 9. 标签状态动作（OSC 21337）

```typescript
/**
 * 每个标签的 chrome 元数据。每个字段三态：
 * - 属性不存在 → 序列中未提及，无变化
 * - null → 显式清除（裸键或 key= 空值）
 * - value → 设置为此值
 */
export type TabStatusAction = {
  indicator?: Color | null      // 指示器颜色
  status?: string | null        // 状态文本
  statusColor?: Color | null    // 状态颜色
}
```

### 10. 解析输出

```typescript
/** 样式化文本段 */
export type TextSegment = {
  type: 'text'
  text: string
  style: TextStyle
}

/** 字素（视觉字符单元）带宽度信息 */
export type Grapheme = {
  value: string
  width: 1 | 2  // 列宽
}

/** 所有可能的解析动作 */
export type Action =
  | { type: 'text'; graphemes: Grapheme[]; style: TextStyle }
  | { type: 'cursor'; action: CursorAction }
  | { type: 'erase'; action: EraseAction }
  | { type: 'scroll'; action: ScrollAction }
  | { type: 'mode'; action: ModeAction }
  | { type: 'link'; action: LinkAction }
  | { type: 'title'; action: TitleAction }
  | { type: 'tabStatus'; action: TabStatusAction }
  | { type: 'sgr'; params: string }  // 选择图形表现（样式变更）
  | { type: 'bell' }
  | { type: 'reset' }  // 完全终端复位（ESC c）
  | { type: 'unknown'; sequence: string }  // 未识别序列
```

### 11. 相等性比较

```typescript
/** 检查两个样式是否相等 */
export function stylesEqual(a: TextStyle, b: TextStyle): boolean

/** 检查两个颜色是否相等 */
export function colorsEqual(a: Color, b: Color): boolean
```

## 具体技术实现

### 相等性比较实现

```typescript
export function stylesEqual(a: TextStyle, b: TextStyle): boolean {
  return (
    a.bold === b.bold &&
    a.dim === b.dim &&
    a.italic === b.italic &&
    a.underline === b.underline &&
    a.blink === b.blink &&
    a.inverse === b.inverse &&
    a.hidden === b.hidden &&
    a.strikethrough === b.strikethrough &&
    a.overline === b.overline &&
    colorsEqual(a.fg, b.fg) &&
    colorsEqual(a.bg, b.bg) &&
    colorsEqual(a.underlineColor, b.underlineColor)
  )
}

export function colorsEqual(a: Color, b: Color): boolean {
  if (a.type !== b.type) return false
  switch (a.type) {
    case 'named':
      return a.name === (b as typeof a).name
    case 'indexed':
      return a.index === (b as typeof a).index
    case 'rgb':
      return (
        a.r === (b as typeof a).r &&
        a.g === (b as typeof a).g &&
        a.b === (b as typeof a).b
      )
    case 'default':
      return true
  }
}
```

### 类型设计原则

1. **语义化**：类型表示"做什么"而非"怎么做"
   - `CursorAction` 表示光标移动意图，而非转义序列字符串

2. **可扩展性**：使用联合类型和 discriminated union
   - `Action` 类型可以轻松添加新动作类型

3. **精确性**：使用字面量类型和严格联合
   - `CursorDirection` 限制为 4 个有效方向
   - `UnderlineStyle` 限制为 6 个有效样式

4. **可空性**：使用 `undefined` 表示可选，`null` 表示显式清除
   - `TabStatusAction` 的三态设计

## 关键代码路径与文件引用

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `ansi.ts` | 无 | 基础常量，不依赖 types |
| `csi.ts` | 无 | 序列生成，不依赖 types |
| `dec.ts` | 无 | 序列生成，不依赖 types |
| `esc.ts` | `Action` | 返回解析结果 |
| `osc.ts` | `Action`, `Color`, `TabStatusAction` | 返回解析结果 |
| `parser.ts` | `Action`, `Grapheme`, `TextStyle`, `defaultStyle` | 解析输出 |
| `sgr.ts` | `NamedColor`, `TextStyle`, `UnderlineStyle`, `defaultStyle` | 样式应用 |
| `tokenize.ts` | 无 | 分词，不依赖语义类型 |
| `termio.ts` | 大量类型 | 模块公开 API |

### 导出内容

```typescript
// 颜色类型
export type NamedColor
export type Color

// 样式类型
export type UnderlineStyle
export type TextStyle

// 动作类型
export type CursorDirection
export type CursorAction
export type EraseAction
export type ScrollAction
export type ModeAction
export type LinkAction
export type TitleAction
export type TabStatusAction

// 输出类型
export type TextSegment
export type Grapheme
export type Action

// 工具函数
export function defaultStyle(): TextStyle
export function stylesEqual(a: TextStyle, b: TextStyle): boolean
export function colorsEqual(a: Color, b: Color): boolean
```

## 依赖与外部交互

### 无内部依赖

`types.ts` 是 termio 模块的类型基础，**不依赖任何其他模块**。

### 外部使用场景

1. **模块公开 API**：`termio.ts` 重新导出主要类型
2. **解析器输出**：`parser.ts` 返回 `Action[]`
3. **样式处理**：`sgr.ts` 使用 `TextStyle` 和 `applySGR`
4. **组件渲染**：`Ansi.tsx` 将 `Action` 转换为 React 组件

## 风险、边界与改进建议

### 边界情况

1. **颜色值范围**：`Color` 类型不验证 RGB 值范围（0-255）或索引范围（0-255）
2. **坐标值**：`CursorAction` 中的行列坐标不验证正数
3. **空字符串**：`Grapheme.value` 可以是空字符串（spacer 单元）

### 风险

1. **类型安全**：运行时无法保证类型约束（如 RGB 范围）
2. **性能**：`stylesEqual` 和 `colorsEqual` 进行深度比较，高频调用可能有开销
3. **扩展性**：添加新动作类型需要更新所有使用 `Action` 的 switch 语句

### 改进建议

1. **添加 branded types**：
   ```typescript
   type RGBValue = number & { __brand: 'RGBValue' }
   function rgbValue(n: number): RGBValue {
     if (n < 0 || n > 255) throw new Error('RGB value out of range')
     return n as RGBValue
   }
   ```

2. **添加验证函数**：
   ```typescript
   export function isValidColor(c: Color): boolean
   export function isValidCursorAction(a: CursorAction): boolean
   ```

3. **性能优化**：
   - 使用对象引用相等性优化 `stylesEqual`
   - 添加 `styleId` 概念（类似 `screen.ts` 中的 `StylePool`）

4. **添加更多动作类型**：
   - `charset`：字符集切换
   - `softReset`：软复位（DECSTR）
   - `windowManipulation`：窗口操作（CSI t）

5. **文档改进**：
   - 添加每个动作类型的示例
   - 添加与 ANSI 标准的映射
   - 添加终端兼容性说明

### 测试建议

- 测试 `defaultStyle()` 返回值
- 测试 `stylesEqual` 和 `colorsEqual` 各种情况
- 测试类型收窄（discriminated union）
- 验证所有导出类型都有使用
- 测试与 `parser.ts` 输出的一致性
