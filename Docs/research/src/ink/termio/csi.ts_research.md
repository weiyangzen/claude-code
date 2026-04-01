# csi.ts 研究报告

## 场景与职责

`csi.ts` 实现了 **CSI (Control Sequence Introducer)** 控制序列的完整支持。CSI 是 ANSI 转义序列中最常用的类型，格式为 `ESC [ <参数> <终止字节>`，用于控制光标移动、屏幕擦除、滚动、文本样式（SGR）等。

该文件的核心职责：
1. **定义 CSI 字节范围常量** —— 参数字节、中间字节、终止字节的合法范围
2. **提供 CSI 字节分类函数** —— 判断字节类型（参数/中间/终止）
3. **定义 CSI 命令常量** —— 光标移动、擦除、滚动、模式设置等命令的终止字节
4. **生成 CSI 序列** —— 提供 `csi()` 函数和各类便捷函数生成转义序列
5. **定义光标样式和擦除区域常量**

## 功能点目的

### 1. CSI 字节范围定义

```typescript
export const CSI_RANGE = {
  PARAM_START: 0x30,      // '0'
  PARAM_END: 0x3f,        // '?'
  INTERMEDIATE_START: 0x20,  // ' '
  INTERMEDIATE_END: 0x2f,    // '/'
  FINAL_START: 0x40,      // '@'
  FINAL_END: 0x7e,        // '~'
} as const
```

CSI 序列结构：`ESC [ <参数字节>... <中间字节>... <终止字节>`

### 2. CSI 命令常量

| 类别 | 常量 | 字节 | 说明 |
|------|------|------|------|
| **光标移动** | CUU | 0x41 ('A') | 光标上移 |
| | CUD | 0x42 ('B') | 光标下移 |
| | CUF | 0x43 ('C') | 光标右移 |
| | CUB | 0x44 ('D') | 光标左移 |
| | CNL | 0x45 ('E') | 光标下移一行 |
| | CPL | 0x46 ('F') | 光标上移一行 |
| | CHA | 0x47 ('G') | 光标水平绝对定位 |
| | CUP | 0x48 ('H') | 光标定位（行列）|
| | VPA | 0x64 ('d') | 垂直位置绝对 |
| | HVP | 0x66 ('f') | 水平垂直位置 |
| **擦除** | ED | 0x4a ('J') | 擦除显示区域 |
| | EL | 0x4b ('K') | 擦除行 |
| | ECH | 0x58 ('X') | 擦除字符 |
| **插入/删除** | IL | 0x4c ('L') | 插入行 |
| | DL | 0x4d ('M') | 删除行 |
| | ICH | 0x40 ('@') | 插入字符 |
| | DCH | 0x50 ('P') | 删除字符 |
| **滚动** | SU | 0x53 ('S') | 向上滚动 |
| | SD | 0x54 ('T') | 向下滚动 |
| **模式** | SM | 0x68 ('h') | 设置模式 |
| | RM | 0x6c ('l') | 重置模式 |
| **样式** | SGR | 0x6d ('m') | 选择图形表现 |
| **其他** | DSR | 0x6e ('n') | 设备状态报告 |
| | DECSCUSR | 0x71 ('q') | 设置光标样式 |
| | DECSTBM | 0x72 ('r') | 设置滚动边界 |
| | SCOSC | 0x73 ('s') | 保存光标位置 |
| | SCORC | 0x75 ('u') | 恢复光标位置 |
| | CBT | 0x5a ('Z') | 光标后退制表 |

### 3. 序列生成函数

```typescript
// 通用 CSI 序列生成
export function csi(...args: (string | number)[]): string
// 示例: csi(31, 'm') => '\x1b[31m' (红色文本)
// 示例: csi(2, 10, 'H') => '\x1b[2;10H' (定位到第2行第10列)

// 便捷函数
export function cursorUp(n = 1): string
export function cursorDown(n = 1): string
export function cursorForward(n = 1): string
export function cursorBack(n = 1): string
export function cursorTo(col: number): string
export function cursorPosition(row: number, col: number): string
export function cursorMove(x: number, y: number): string
export function eraseToEndOfLine(): string
export function eraseLine(): string
export function eraseScreen(): string
export function eraseLines(n: number): string
export function scrollUp(n = 1): string
export function scrollDown(n = 1): string
export function setScrollRegion(top: number, bottom: number): string
```

### 4. 光标样式定义

```typescript
export type CursorStyle = 'block' | 'underline' | 'bar'
export const CURSOR_STYLES: Array<{ style: CursorStyle; blinking: boolean }>
// 索引: 0=默认(块闪烁), 1=块闪烁, 2=块稳定, 3=下划线闪烁, 4=下划线稳定, 5=竖线闪烁, 6=竖线稳定
```

### 5. 高级功能常量

```typescript
// 粘贴标记（DEC mode 2004）
export const PASTE_START = csi('200~')   // CSI 200 ~
export const PASTE_END = csi('201~')     // CSI 201 ~

// 焦点事件（DEC mode 1004）
export const FOCUS_IN = csi('I')         // CSI I
export const FOCUS_OUT = csi('O')        // CSI O

// Kitty 键盘协议
export const ENABLE_KITTY_KEYBOARD = csi('>1u')    // CSI > 1 u
export const DISABLE_KITTY_KEYBOARD = csi('<u')    // CSI < u

// xterm modifyOtherKeys
export const ENABLE_MODIFY_OTHER_KEYS = csi('>4;2m')   // CSI > 4 ; 2 m
export const DISABLE_MODIFY_OTHER_KEYS = csi('>4m')    // CSI > 4 m
```

## 具体技术实现

### CSI 序列构造算法

```typescript
export function csi(...args: (string | number)[]): string {
  if (args.length === 0) return CSI_PREFIX           // ESC [
  if (args.length === 1) return `${CSI_PREFIX}${args[0]}`  // ESC [ <single>
  const params = args.slice(0, -1)                   // 除最后一个都是参数
  const final = args[args.length - 1]                // 最后一个是终止字节
  return `${CSI_PREFIX}${params.join(SEP)}${final}`  // ESC [ p1;p2;...;pN final
}
```

### 字节分类实现

```typescript
export function isCSIParam(byte: number): boolean {
  return byte >= CSI_RANGE.PARAM_START && byte <= CSI_RANGE.PARAM_END
  // 0x30-0x3F: '0'-'9', ':', ';', '<', '=', '>', '?'
}

export function isCSIIntermediate(byte: number): boolean {
  return byte >= CSI_RANGE.INTERMEDIATE_START && byte <= CSI_RANGE.INTERMEDIATE_END
  // 0x20-0x2F: ' ', '!', '"', '#', '$', '%', '&', "'", '(', ')', '*', '+', ',', '-', '.', '/'
}

export function isCSIFinal(byte: number): boolean {
  return byte >= CSI_RANGE.FINAL_START && byte <= CSI_RANGE.FINAL_END
  // 0x40-0x7E: '@'-'~'
}
```

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `ESC`, `ESC_TYPE`, `SEP` | `./ansi.js` | 构建 CSI 前缀 `ESC [` 和参数分隔 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `parser.ts` | `CSI`, `CURSOR_STYLES`, `ERASE_DISPLAY`, `ERASE_LINE_REGION` | 解析 CSI 序列，应用光标样式 |
| `tokenize.ts` | `isCSIFinal`, `isCSIIntermediate`, `isCSIParam` | 分词器 CSI 状态机 |
| `dec.ts` | `csi` | 构建 DEC 私有模式序列 |
| `terminal.ts` | `cursorMove`, `cursorTo`, `eraseLines` | 终端输出控制 |
| `screen.ts` | `BEL`, `ESC`, `SEP` | 样式池和屏幕渲染 |

### 导出内容

```typescript
// 常量
export const CSI_PREFIX
export const CSI_RANGE
export const CSI
export const ERASE_DISPLAY
export const ERASE_LINE_REGION
export const CURSOR_STYLES
export type CursorStyle

// 便捷序列常量
export const CURSOR_LEFT, CURSOR_HOME, CURSOR_SAVE, CURSOR_RESTORE
export const ERASE_LINE, ERASE_SCREEN, ERASE_SCROLLBACK
export const RESET_SCROLL_REGION
export const PASTE_START, PASTE_END, FOCUS_IN, FOCUS_OUT
export const ENABLE_KITTY_KEYBOARD, DISABLE_KITTY_KEYBOARD
export const ENABLE_MODIFY_OTHER_KEYS, DISABLE_MODIFY_OTHER_KEYS

// 函数
export function isCSIParam, isCSIIntermediate, isCSIFinal
export function csi, cursorUp, cursorDown, cursorForward, cursorBack
export function cursorTo, cursorPosition, cursorMove
export function eraseToEndOfLine, eraseToStartOfLine, eraseLine, eraseLines
export function eraseToEndOfScreen, eraseToStartOfScreen, eraseScreen
export function scrollUp, scrollDown, setScrollRegion
```

## 依赖与外部交互

### 内部依赖

- `ansi.ts`：提供 `ESC`, `ESC_TYPE`, `SEP` 基础常量

### 外部使用场景

1. **终端渲染**：`terminal.ts` 使用 `cursorMove`, `cursorTo`, `eraseLines` 控制光标
2. **输入解析**：`parser.ts` 使用 `CSI` 常量识别 CSI 命令类型
3. **分词**：`tokenize.ts` 使用字节分类函数识别 CSI 序列边界
4. **DEC 模式**：`dec.ts` 使用 `csi()` 函数构建 DEC 私有模式序列

## 风险、边界与改进建议

### 边界情况

1. **参数默认值**：`cursorUp(0)` 返回空字符串（优化），但 `cursorUp()` 默认参数为 1
2. **参数分隔符**：使用分号 `;`，但 Kitty 等终端也支持冒号 `:` 用于子参数
3. **私有模式标记**：`?` 前缀用于 DEC 私有模式，但本文件不处理（由 `dec.ts` 处理）

### 风险

1. **无参数验证**：`csi()` 不验证参数范围，错误参数可能产生无效序列
2. **字符串拼接**：`cursorMove` 使用字符串拼接，大量调用可能有性能影响
3. **硬编码默认值**：`CURSOR_STYLES` 索引 0 被定义为"默认"，但不同终端默认样式不同

### 改进建议

1. **添加参数验证**：
   ```typescript
   export function csi(...args: (string | number)[]): string {
     // 验证终止字节在 FINAL_START-FINAL_END 范围内
   }
   ```

2. **支持冒号分隔符**：为 Kitty 子参数格式添加支持
   ```typescript
   // 38:2::255:255:255 格式（省略颜色空间 ID）
   ```

3. **添加更多便捷函数**：
   - `cursorNextLine(n)` / `cursorPrevLine(n)`
   - `insertChars(n)` / `deleteChars(n)`
   - `insertLines(n)` / `deleteLines(n)`

4. **性能优化**：
   - 对频繁使用的序列（如 `ERASE_LINE`）使用预计算常量
   - 考虑使用模板字符串缓存

5. **文档改进**：
   - 为每个 CSI 命令添加标准参考（ECMA-48 章节）
   - 添加终端兼容性说明

### 测试建议

- 测试 `csi()` 各种参数组合
- 测试字节分类函数边界值
- 验证生成的序列符合 ECMA-48 标准
- 测试光标移动函数的坐标计算
