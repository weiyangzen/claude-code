# sgr.ts 研究报告

## 场景与职责

`sgr.ts` 实现了 **SGR (Select Graphic Rendition)** 参数解析器。SGR 是 ANSI 转义序列中最常用的类型，用于设置文本样式（颜色、粗体、下划线等），格式为 `CSI <参数> m`。

该文件的核心职责：
1. **解析 SGR 参数** —— 支持分号 `;` 和冒号 `:` 分隔的参数格式
2. **应用样式变更** —— 将 SGR 参数应用到 `TextStyle` 对象
3. **支持扩展颜色** —— 256 色索引和 24-bit RGB 真彩色
4. **支持下划线样式** —— 单线、双线、波浪线、点线、虚线

## 功能点目的

### 1. 命名颜色

```typescript
const NAMED_COLORS: NamedColor[] = [
  'black', 'red', 'green', 'yellow', 'blue', 'magenta', 'cyan', 'white',
  'brightBlack', 'brightRed', 'brightGreen', 'brightYellow',
  'brightBlue', 'brightMagenta', 'brightCyan', 'brightWhite',
]
```

**颜色码映射**：
- 30-37：标准前景色（黑、红、绿、黄、蓝、洋红、青、白）
- 40-47：标准背景色
- 90-97：明亮前景色
- 100-107：明亮背景色
- 39：默认前景色
- 49：默认背景色

### 2. 下划线样式

```typescript
const UNDERLINE_STYLES: UnderlineStyle[] = [
  'none', 'single', 'double', 'curly', 'dotted', 'dashed'
]
```

**SGR 4 子参数**：
- `4` 或 `4:1`：单下划线
- `4:2`：双下划线
- `4:3`：波浪下划线
- `4:4`：点下划线
- `4:5`：虚线下划线

### 3. 参数解析

```typescript
type Param = { 
  value: number | null      // 主参数值
  subparams: number[]       // 冒号分隔的子参数
  colon: boolean            // 是否使用冒号格式
}

function parseParams(str: string): Param[]
```

**支持格式**：
- 分号分隔：`31;1;4`（红色、粗体、下划线）
- 冒号分隔：`38:2:255:0:0`（RGB 红色）
- 混合：`38:2::255:0:0;1`（Kitty 兼容格式 + 粗体）
- 空参数：视为 0（重置）

### 4. 扩展颜色解析

```typescript
function parseExtendedColor(
  params: Param[],
  idx: number,
): { r: number; g: number; b: number } | { index: number } | null
```

**支持格式**：

| 格式 | 说明 | 示例 |
|------|------|------|
| `38;5;<n>` | 256 色前景 | `\x1b[38;5;196m`（亮红）|
| `38:5:<n>` | 256 色前景（冒号）| `\x1b[38:5:196m` |
| `38;2;<r>;<g>;<b>` | RGB 前景 | `\x1b[38;2;255;0;0m`（红）|
| `38:2::<r>:<g>:<b>` | RGB 前景（Kitty）| `\x1b[38:2::255:0:0m` |
| `48;5;<n>` | 256 色背景 | `\x1b[48;5;196m` |
| `48;2;<r>;<g>;<b>` | RGB 背景 | `\x1b[48;2;255;0;0m` |
| `58;...` | 下划线颜色 | `\x1b[58;2;255;0;0m` |

### 5. 样式应用

```typescript
export function applySGR(paramStr: string, style: TextStyle): TextStyle
```

**处理逻辑**：
1. 解析参数字符串为 `Param[]` 数组
2. 遍历参数，根据参数码应用样式变更
3. 返回新的 `TextStyle` 对象（不可变更新）

**SGR 参数码**：

| 码 | 效果 |
|----|------|
| 0 | 重置所有样式 |
| 1 | 粗体 |
| 2 | 暗淡 |
| 3 | 斜体 |
| 4 | 下划线（可带子参数）|
| 5, 6 | 闪烁 |
| 7 | 反显（前景背景交换）|
| 8 | 隐藏 |
| 9 | 删除线 |
| 21 | 双下划线 |
| 22 | 关闭粗体/暗淡 |
| 23 | 关闭斜体 |
| 24 | 关闭下划线 |
| 25 | 关闭闪烁 |
| 27 | 关闭反显 |
| 28 | 关闭隐藏 |
| 29 | 关闭删除线 |
| 53 | 上划线 |
| 55 | 关闭上划线 |
| 30-37 | 前景色（标准）|
| 38 | 前景色（扩展）|
| 39 | 默认前景色 |
| 40-47 | 背景色（标准）|
| 48 | 背景色（扩展）|
| 49 | 默认背景色 |
| 58 | 下划线颜色（扩展）|
| 59 | 默认下划线颜色 |
| 90-97 | 前景色（明亮）|
| 100-107 | 背景色（明亮）|

## 具体技术实现

### 参数解析状态机

```typescript
function parseParams(str: string): Param[] {
  const result: Param[] = []
  let current: Param = { value: null, subparams: [], colon: false }
  let num = ''
  let inSub = false

  for (let i = 0; i <= str.length; i++) {
    const c = str[i]
    if (c === ';' || c === undefined) {
      // 分号或结束：完成当前参数
      const n = num === '' ? null : parseInt(num, 10)
      if (inSub) current.subparams.push(n)
      else current.value = n
      result.push(current)
      current = { value: null, subparams: [], colon: false }
      num = ''
      inSub = false
    } else if (c === ':') {
      // 冒号：进入子参数模式
      const n = num === '' ? null : parseInt(num, 10)
      if (!inSub) {
        current.value = n
        current.colon = true
        inSub = true
      } else {
        current.subparams.push(n)
      }
      num = ''
    } else if (c >= '0' && c <= '9') {
      num += c
    }
  }
  return result
}
```

### 扩展颜色解析逻辑

```typescript
function parseExtendedColor(params: Param[], idx: number): ColorSpec | null {
  const p = params[idx]
  
  // 冒号格式: 38:5:<n> 或 38:2::<r>:<g>:<b>
  if (p.colon && p.subparams.length >= 1) {
    if (p.subparams[0] === 5 && p.subparams.length >= 2) {
      return { index: p.subparams[1]! }  // 256 色
    }
    if (p.subparams[0] === 2 && p.subparams.length >= 4) {
      const off = p.subparams.length >= 5 ? 1 : 0  // 兼容省略颜色空间 ID
      return {
        r: p.subparams[1 + off]!,
        g: p.subparams[2 + off]!,
        b: p.subparams[3 + off]!,
      }
    }
  }
  
  // 分号格式: 38;5;<n> 或 38;2;<r>;<g>;<b>
  const next = params[idx + 1]
  if (next?.value === 5 && params[idx + 2]?.value !== null) {
    return { index: params[idx + 2]!.value! }
  }
  if (next?.value === 2) {
    const r = params[idx + 2]?.value
    const g = params[idx + 3]?.value
    const b = params[idx + 4]?.value
    if (r !== null && g !== null && b !== null) {
      return { r, g, b }
    }
  }
  
  return null
}
```

### 样式应用循环

```typescript
export function applySGR(paramStr: string, style: TextStyle): TextStyle {
  const params = parseParams(paramStr)
  let s = { ...style }
  let i = 0

  while (i < params.length) {
    const p = params[i]!
    const code = p.value ?? 0

    if (code === 0) { s = defaultStyle(); i++; continue }
    if (code === 1) { s.bold = true; i++; continue }
    // ... 其他简单参数
    
    if (code === 38) {
      const c = parseExtendedColor(params, i)
      if (c) {
        s.fg = 'index' in c 
          ? { type: 'indexed', index: c.index }
          : { type: 'rgb', ...c }
        i += p.colon ? 1 : 'index' in c ? 3 : 5  // 跳过已消费的参数
        continue
      }
    }
    // ... 48, 58 类似处理
    
    i++
  }
  return s
}
```

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `NamedColor`, `TextStyle`, `UnderlineStyle` | `./types.js` | 类型定义 |
| `defaultStyle` | `./types.js` | 默认样式 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `parser.ts` | `applySGR` | 应用 SGR 参数到当前样式 |

### 导出内容

```typescript
export function applySGR(paramStr: string, style: TextStyle): TextStyle
```

## 依赖与外部交互

### 内部依赖

- `types.ts`：`TextStyle`、`NamedColor`、`UnderlineStyle` 类型和 `defaultStyle()`

### 外部使用场景

1. **Parser 样式更新**：`parser.ts` 在解析到 SGR 序列时调用 `applySGR`：
   ```typescript
   if (action.type === 'sgr') {
     this.style = applySGR(action.params, this.style)
     return []
   }
   ```

2. **样式继承**：SGR 是累积的，新参数基于当前样式修改

3. **重置机制**：参数 0 重置为 `defaultStyle()`

## 风险、边界与改进建议

### 边界情况

1. **空参数字符串**：`parseParams('')` 返回 `[{ value: 0, subparams: [], colon: false }]`，等效于重置
2. **无效颜色参数**：`parseExtendedColor` 返回 `null`，样式不变
3. **参数消费**：扩展颜色格式需要跳过已消费的参数（2-5 个）
4. **颜色值范围**：不验证 RGB 值是否在 0-255 范围内

### 风险

1. **参数解析性能**：每个 SGR 序列都创建新数组和对象，高频更新可能有 GC 压力
2. **冒号格式兼容性**：冒号格式（Kitty 风格）不是所有终端都支持
3. **下划线颜色**：SGR 58/59 较新，部分终端不支持
4. **上划线**：SGR 53/55 较少见，支持有限

### 改进建议

1. **添加参数验证**：
   ```typescript
   if (r < 0 || r > 255 || g < 0 || g > 255 || b < 0 || b > 255) {
     // 记录警告或截断到有效范围
   }
   ```

2. **支持更多 SGR 参数**：
   - SGR 10-19：字体选择
   - SGR 73/74/75：上标/下标/比例字符
   - SGR 76：字符间距

3. **性能优化**：
   - 使用对象池重用 `Param` 对象
   - 缓存常用 SGR 参数的解析结果
   - 使用位掩码表示简单样式（粗体、斜体等）

4. **添加 SGR 序列生成**：
   ```typescript
   export function sgrFromStyle(style: TextStyle): string {
     // 从 TextStyle 生成 SGR 参数字符串
   }
   ```

5. **文档改进**：
   - 添加终端兼容性矩阵
   - 添加颜色格式示例
   - 说明 Kitty 格式与传统格式的区别

### 测试建议

- 测试各种参数格式（分号、冒号、混合）
- 测试扩展颜色（256 色、RGB）
- 测试参数边界值（空字符串、无效数字）
- 测试样式累积和重置
- 测试下划线子参数
- 验证与常见终端的兼容性（xterm、iTerm2、GNOME Terminal）
