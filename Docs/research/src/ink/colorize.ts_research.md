# colorize.ts 研究文档

## 场景与职责

`colorize.ts` 是 Ink 终端 UI 框架的颜色处理核心模块，负责将结构化颜色值转换为 ANSI 转义序列。它桥接了 Ink 的声明式样式系统与底层终端颜色输出。

### 核心职责

1. **颜色值解析**：解析多种颜色格式（ansi:、hex、ansi256、rgb）
2. **ANSI 序列生成**：使用 chalk 库生成 SGR（Select Graphic Rendition）序列
3. **文本样式应用**：应用粗体、斜体、下划线等文本修饰
4. **终端能力适配**：根据终端能力调整颜色级别（truecolor/256色）

### 使用场景

- Text 组件的颜色渲染
- 边框、背景色的终端输出
- 语法高亮和主题系统
- 任何需要颜色输出的渲染路径

## 功能点目的

### 1. 颜色级别适配

**VS Code / xterm.js 增强**：
- 问题：code-server/Coder 容器常未设置 `COLORTERM=truecolor`
- chalk 的 `supports-color` 不认识 `TERM_PROGRAM=vscode`
- 结果：truecolor 降级为 256 色，颜色失真（如橙色变鲑鱼色）
- 解决：检测到 `TERM_PROGRAM=vscode` 且 level=2 时，提升到 level=3

**tmux 降级**：
- 问题：tmux 客户端发射器默认不传递 truecolor 到外部终端
- 结果：显式背景色的单元格显示为默认背景（黑色）
- 解决：在 tmux 中运行时降级到 256 色
- 逃逸舱口：`CLAUDE_CODE_TMUX_TRUECOLOR` 环境变量跳过降级

### 2. 颜色格式支持

| 格式 | 示例 | 说明 |
|------|------|------|
| ansi: | `ansi:red`, `ansi:greenBright` | 标准 16 色 |
| hex | `#ff5733` | 真彩色 |
| ansi256 | `ansi256(196)` | 256 色调色板 |
| rgb | `rgb(255,87,51)` | RGB 真彩色 |

### 3. 文本样式应用

应用顺序（从内到外）：
1. dim
2. bold
3. italic
4. underline
5. strikethrough
6. inverse
7. color（前景色）
8. backgroundColor（背景色）

## 具体技术实现

### 颜色级别调整

```typescript
// VS Code / xterm.js 增强
function boostChalkLevelForXtermJs(): boolean {
  if (process.env.TERM_PROGRAM === 'vscode' && chalk.level === 2) {
    chalk.level = 3
    return true
  }
  return false
}

// tmux 降级
function clampChalkLevelForTmux(): boolean {
  if (process.env.CLAUDE_CODE_TMUX_TRUECOLOR) return false
  if (process.env.TMUX && chalk.level > 2) {
    chalk.level = 2
    return true
  }
  return false
}

// 模块加载时执行一次
export const CHALK_BOOSTED_FOR_XTERMJS = boostChalkLevelForXtermJs()
export const CHALK_CLAMPED_FOR_TMUX = clampChalkLevelForTmux()
```

### 颜色解析与转换

```typescript
const RGB_REGEX = /^rgb\(\s?(\d+),\s?(\d+),\s?(\d+)\s?\)$/
const ANSI_REGEX = /^ansi256\(\s?(\d+)\s?\)$/

export const colorize = (
  str: string,
  color: string | undefined,
  type: ColorType,  // 'foreground' | 'background'
): string => {
  if (!color) return str

  // ansi: 前缀 - 标准 16 色
  if (color.startsWith('ansi:')) {
    const value = color.substring('ansi:'.length)
    switch (value) {
      case 'black':
        return type === 'foreground' ? chalk.black(str) : chalk.bgBlack(str)
      // ... 其他 15 种颜色
    }
  }

  // hex 颜色
  if (color.startsWith('#')) {
    return type === 'foreground'
      ? chalk.hex(color)(str)
      : chalk.bgHex(color)(str)
  }

  // ansi256
  if (color.startsWith('ansi256')) {
    const matches = ANSI_REGEX.exec(color)
    if (!matches) return str
    const value = Number(matches[1])
    return type === 'foreground'
      ? chalk.ansi256(value)(str)
      : chalk.bgAnsi256(value)(str)
  }

  // rgb
  if (color.startsWith('rgb')) {
    const matches = RGB_REGEX.exec(color)
    if (!matches) return str
    const [r, g, b] = matches.slice(1).map(Number)
    return type === 'foreground'
      ? chalk.rgb(r, g, b)(str)
      : chalk.bgRgb(r, g, b)(str)
  }

  return str
}
```

### 文本样式应用

```typescript
export function applyTextStyles(text: string, styles: TextStyles): string {
  let result = text

  // 从内到外应用样式（chalk 包装顺序）
  if (styles.inverse) result = chalk.inverse(result)
  if (styles.strikethrough) result = chalk.strikethrough(result)
  if (styles.underline) result = chalk.underline(result)
  if (styles.italic) result = chalk.italic(result)
  if (styles.bold) result = chalk.bold(result)
  if (styles.dim) result = chalk.dim(result)

  // 颜色最后应用（最外层）
  if (styles.color) {
    result = colorize(result, styles.color, 'foreground')
  }
  if (styles.backgroundColor) {
    result = colorize(result, styles.backgroundColor, 'background')
  }

  return result
}
```

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/colorize.ts`
- **导出函数**：
  - `colorize` - 颜色应用函数
  - `applyTextStyles` - 完整文本样式应用
  - `applyColor` - 仅前景色应用
- **导出常量**：
  - `CHALK_BOOSTED_FOR_XTERMJS` - 是否提升了 chalk 级别
  - `CHALK_CLAMPED_FOR_TMUX` - 是否限制了 chalk 级别

### 依赖关系

**被导入**：
- `chalk` - ANSI 颜色库
- `./styles.js` - Color、TextStyles 类型

**导入使用**：
```typescript
import chalk from 'chalk'
import type { Color, TextStyles } from './styles.js'
```

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `boostChalkLevelForXtermJs` | VS Code 颜色级别提升 | 20-26 |
| `clampChalkLevelForTmux` | tmux 颜色级别降级 | 47-57 |
| `colorize` | 颜色值转 ANSI 序列 | 69-169 |
| `applyTextStyles` | 应用完整文本样式 | 176-220 |
| `applyColor` | 仅应用前景色 | 226-231 |

### 相关文件

- `src/ink/styles.ts` - Color、TextStyles 类型定义
- `src/ink/components/Text.tsx` - 使用 applyTextStyles
- `src/ink/render-border.ts` - 边框颜色渲染
- `src/ink/output.ts` - 输出颜色处理

## 依赖与外部交互

### 外部库

**chalk**：
- 提供跨平台的 ANSI 颜色支持
- 自动检测终端颜色能力（level 0-3）
- 支持 16色、256色、真彩色（hex/rgb）

### 环境变量

| 变量 | 用途 |
|------|------|
| `TERM_PROGRAM` | 检测 VS Code 终端 |
| `TMUX` | 检测 tmux 环境 |
| `CLAUDE_CODE_TMUX_TRUECOLOR` | 跳过 tmux 降级 |
| `COLORTERM` | chalk 用于检测 truecolor |
| `FORCE_COLOR` | chalk 用于强制颜色级别 |
| `NO_COLOR` | chalk 用于禁用颜色 |

### 调用时机

```
渲染管线
    ↓
组件渲染（Text、Box 等）
    ↓
样式解析
    ↓
applyTextStyles / colorize
    ↓
chalk 生成 ANSI 序列
    ↓
输出到终端
```

## 风险、边界与改进建议

### 已知风险

1. **chalk 级别全局修改**：
   - chalk 是单例，级别修改影响整个进程
   - 模块加载时执行，顺序很重要（boost 必须在 clamp 之前）
   - 其他代码可能意外修改级别

2. **正则表达式性能**：
   - `RGB_REGEX` 和 `ANSI_REGEX` 每次调用都编译
   - 高频调用场景可能有性能影响

3. **颜色格式验证**：
   - 无效格式静默返回原字符串
   - 可能隐藏配置错误

### 边界情况

1. **空/undefined 颜色**：直接返回原字符串
2. **无效颜色格式**：返回原字符串，无错误抛出
3. **颜色值越界**：chalk 内部处理，可能产生无效序列
4. **样式顺序**：bold/dim 互斥由调用方保证

### 改进建议

1. **性能优化**：
   - 预编译正则表达式
   - 添加颜色值缓存（LRU）
   
   ```typescript
   // 示例：颜色缓存
   const colorCache = new Map<string, (s: string) => string>()
   ```

2. **错误处理**：
   - 添加颜色格式验证
   - 开发模式下警告无效颜色
   
   ```typescript
   if (process.env.NODE_ENV === 'development') {
     console.warn(`Invalid color format: ${color}`)
   }
   ```

3. **功能扩展**：
   - 支持 HSL 颜色格式
   - 添加颜色混合/透明度支持
   - 主题系统集成

4. **测试覆盖**：
   - 各种颜色格式的单元测试
   - 终端能力模拟测试
   - chalk 级别调整验证

5. **代码改进**：
   - 将 16 色 switch 提取为配置表
   - 添加颜色格式文档注释
