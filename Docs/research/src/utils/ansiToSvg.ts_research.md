# ansiToSvg.ts 研究文档

## 场景与职责

`ansiToSvg.ts` 是 Claude Code CLI 的终端输出可视化工具，负责将 ANSI 转义序列格式的终端文本转换为 SVG 矢量图形。该模块主要用于：

1. **终端截图生成**：将带颜色的终端输出转换为可分享的 SVG 图片
2. **ANSI 颜色解析**：支持标准 16 色、256 色和 24 位真彩色
3. **代码高亮展示**：保留终端输出的颜色、粗体等样式信息

## 功能点目的

### 1. ANSI 颜色解析 (`parseAnsi`)
- 解析 ANSI 转义序列（`\x1b[` 开头的控制序列）
- 支持基本颜色（30-37, 90-97）
- 支持 256 色模式（`38;5;n`）
- 支持 24 位真彩色（`38;2;r;g;b`）
- 支持粗体样式（code 1）和重置（code 0）

### 2. 256 色调色板 (`get256Color`)
- 标准颜色（0-15）：基本 ANSI 颜色
- 216 色立方（16-231）：6×6×6 RGB 立方
- 灰度色（232-255）：24 级灰度

### 3. SVG 生成 (`ansiToSvg`)
- 生成可缩放矢量图形
- 使用 `<tspan>` 元素实现不同颜色的文本段
- 支持自定义字体、字号、行高、内边距
- 支持圆角背景和透明背景

## 具体技术实现

### 关键数据结构

```typescript
// ANSI RGB 颜色
export type AnsiColor = {
  r: number
  g: number
  b: number
}

// 文本片段（带样式）
export type TextSpan = {
  text: string
  color: AnsiColor
  bold: boolean
}

// 解析后的行
export type ParsedLine = TextSpan[]

// SVG 生成选项
export type AnsiToSvgOptions = {
  fontFamily?: string
  fontSize?: number
  lineHeight?: number
  paddingX?: number
  paddingY?: number
  backgroundColor?: string
  borderRadius?: number
}
```

### 关键流程

#### ANSI 解析流程
1. 按 `\n` 分割文本为多行
2. 对每行遍历字符，检测 `\x1b[` 转义序列
3. 解析 SGR（Select Graphic Rendition）参数
4. 根据当前状态（颜色、粗体）分组文本
5. 返回按行组织的 TextSpan 数组

#### SVG 生成流程
1. 调用 `parseAnsi` 解析文本
2. 去除末尾空行
3. 计算最大行宽和所需 SVG 尺寸
4. 生成 SVG XML：
   - `<rect>` 背景矩形（带圆角）
   - `<text>` 每行一个，内部用 `<tspan>` 区分颜色
   - 使用 `xml:space="preserve"` 保留空白

### 颜色解析实现

```typescript
// 扩展颜色解析（256色和真彩色）
if (code === 38) {
  if (codes[k + 1] === 5 && codes[k + 2] !== undefined) {
    // 256-color mode: 38;5;n
    const colorIndex = codes[k + 2]!
    currentColor = get256Color(colorIndex)
    k += 2
  } else if (
    codes[k + 1] === 2 &&
    codes[k + 2] !== undefined &&
    codes[k + 3] !== undefined &&
    codes[k + 4] !== undefined
  ) {
    // 24-bit true color: 38;2;r;g;b
    currentColor = {
      r: codes[k + 2]!,
      g: codes[k + 3]!,
      b: codes[k + 4]!,
    }
    k += 4
  }
}
```

## 关键代码路径与文件引用

### 本文件导出
- `parseAnsi(text: string): ParsedLine[]` - ANSI 解析主函数
- `ansiToSvg(ansiText: string, options?: AnsiToSvgOptions): string` - SVG 生成主函数
- `DEFAULT_FG`, `DEFAULT_BG` - 默认前景/背景色
- `AnsiColor`, `TextSpan`, `ParsedLine`, `AnsiToSvgOptions` - 类型定义

### 依赖文件
- `src/utils/xml.ts` - XML 转义工具 (`escapeXml`)

### 调用方
- `src/utils/ansiToPng.ts` - 复用 `parseAnsi` 和颜色类型生成 PNG

## 依赖与外部交互

### 内部依赖
| 文件 | 用途 |
|------|------|
| `src/utils/xml.ts` | XML/HTML 特殊字符转义 |

### 外部依赖
- 无外部 npm 依赖

## 风险、边界与改进建议

### 已知限制
1. **仅支持前景色**：背景色代码（40-47, 100-107, 48;5;n, 48;2;r;g;b）未实现
2. **粗体仅标记**：通过 CSS class 标记，不加载粗体字体
3. **字体估算**：字符宽度按 `fontSize * 0.6` 估算，可能不准确
4. **Unicode 处理**：依赖浏览器/渲染器的文本布局

### 安全风险
1. **XSS 防护**：使用 `escapeXml` 转义文本内容，防止恶意脚本注入
2. **输入长度**：未对输入长度做限制，超长输入可能导致内存问题

### 改进建议
1. 添加背景色支持以完整呈现终端输出
2. 使用更精确的字符宽度计算（考虑东亚字符）
3. 添加最大尺寸限制防止 DoS
4. 考虑使用 `Intl.Segmenter` 处理复杂 Unicode

### 性能考虑
- 解析是同步的，大文本可能阻塞事件循环
- 建议使用流式处理或 Web Worker 处理超大输出
