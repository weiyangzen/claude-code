# parser.ts 研究报告

## 场景与职责

`parser.ts` 是 termio 模块的核心，实现了 **ANSI 转义序列的语义解析器**。它将原始终端输入（包含文本和转义序列）解析为结构化的语义动作（`Action`），并维护文本样式状态。

该文件的核心职责：
1. **流式解析** —— 支持增量输入处理，维护解析状态
2. **语义输出** —— 生成结构化 `Action` 对象，而非原始字符串
3. **样式跟踪** —— 维护当前文本样式状态，应用 SGR 参数
4. **字素处理** —— 正确处理 Unicode 字素（包括宽字符、emoji）
5. **序列分类** —— 识别 CSI、OSC、ESC、SS3 序列并分发到相应解析器

## 功能点目的

### 1. Parser 类

```typescript
export class Parser {
  private tokenizer: Tokenizer = createTokenizer()
  style: TextStyle = defaultStyle()  // 当前文本样式
  inLink = false                     // 是否在超链接内
  linkUrl: string | undefined        // 当前超链接 URL

  reset(): void                      // 重置解析器状态
  feed(input: string): Action[]      // 输入数据，返回动作列表
}
```

### 2. 字素处理

```typescript
// Emoji 检测（Unicode 范围）
function isEmoji(codePoint: number): boolean

// 东亚宽字符检测（CJK、全角符号）
function isEastAsianWide(codePoint: number): boolean

// 字素宽度计算（1 或 2 列）
function graphemeWidth(grapheme: string): 1 | 2

// 字素分割生成器
function* segmentGraphemes(str: string): Generator<Grapheme>
```

**字素宽度规则**：
- 多码点字素（如带修饰符的 emoji）：2 列
- Emoji（U+2600-U+26FF, U+2700-U+27BF, U+1F300-U+1F9FF 等）：2 列
- 东亚宽字符（CJK、全角符号）：2 列
- 其他：1 列

### 3. CSI 序列解析

```typescript
function parseCSI(rawSequence: string): Action | null
```

**解析步骤**：
1. 提取终止字节（最后一个字符）
2. 检测私有模式标记（`?`, `>`, `=`）
3. 提取中间字节（非数字/分隔符字符）
4. 解析参数（分号或冒号分隔的数字）
5. 根据终止字节分发到具体处理

**支持的 CSI 命令**：

| 命令 | 动作类型 | 说明 |
|------|----------|------|
| SGR (m) | `{ type: 'sgr', params: string }` | 选择图形表现（样式变更）|
| CUU (A) | `cursor.move(up)` | 光标上移 |
| CUD (B) | `cursor.move(down)` | 光标下移 |
| CUF (C) | `cursor.move(forward)` | 光标右移 |
| CUB (D) | `cursor.move(back)` | 光标左移 |
| CNL (E) | `cursor.nextLine` | 光标下移一行 |
| CPL (F) | `cursor.prevLine` | 光标上移一行 |
| CHA (G) | `cursor.column` | 光标水平定位 |
| CUP/HVP (H/f) | `cursor.position` | 光标定位（行列）|
| VPA (d) | `cursor.row` | 光标垂直定位 |
| ED (J) | `erase.display` | 擦除显示区域 |
| EL (K) | `erase.line` | 擦除行 |
| ECH (X) | `erase.chars` | 擦除字符 |
| SU (S) | `scroll.up` | 向上滚动 |
| SD (T) | `scroll.down` | 向下滚动 |
| DECSTBM (r) | `scroll.setRegion` | 设置滚动区域 |
| SCOSC (s) | `cursor.save` | 保存光标位置 |
| SCORC (u) | `cursor.restore` | 恢复光标位置 |
| DECSCUSR (q + 空格) | `cursor.style` | 设置光标样式 |
| SM/RM + ? (h/l) | `mode.*` | DEC 私有模式设置/重置 |

### 4. DEC 私有模式处理

```typescript
// 解析 CSI ? <mode> h/l 序列
if (privateMode === '?' && (finalByte === CSI.SM || finalByte === CSI.RM)) {
  const enabled = finalByte === CSI.SM
  
  if (p0 === DEC.CURSOR_VISIBLE) {
    return { type: 'cursor', action: enabled ? { type: 'show' } : { type: 'hide' } }
  }
  if (p0 === DEC.ALT_SCREEN_CLEAR || p0 === DEC.ALT_SCREEN) {
    return { type: 'mode', action: { type: 'alternateScreen', enabled } }
  }
  if (p0 === DEC.BRACKETED_PASTE) {
    return { type: 'mode', action: { type: 'bracketedPaste', enabled } }
  }
  // ... 鼠标跟踪、焦点事件等
}
```

### 5. 序列类型识别

```typescript
function identifySequence(seq: string): 'csi' | 'osc' | 'esc' | 'ss3' | 'unknown' {
  if (seq.length < 2) return 'unknown'
  if (seq.charCodeAt(0) !== C0.ESC) return 'unknown'
  
  const second = seq.charCodeAt(1)
  if (second === 0x5b) return 'csi'  // [
  if (second === 0x5d) return 'osc'  // ]
  if (second === 0x4f) return 'ss3'  // O
  return 'esc'
}
```

### 6. 文本处理

```typescript
private processText(text: string): Action[] {
  // 处理文本中的 BEL 字符
  // 分割字素
  // 生成 { type: 'text', graphemes, style } 动作
}
```

## 具体技术实现

### 参数解析

```typescript
function parseCSIParams(paramStr: string): number[] {
  if (paramStr === '') return []
  return paramStr.split(/[;:]/).map(s => (s === '' ? 0 : parseInt(s, 10)))
}
```

**参数规则**：
- 空参数视为 0
- 支持分号 `;` 和冒号 `:` 分隔
- 默认值由具体命令决定（如 `CSI A` 默认上移 1 行）

### 样式应用

```typescript
private processSequence(seq: string): Action[] {
  // ...
  case 'csi': {
    const action = parseCSI(seq)
    if (action.type === 'sgr') {
      this.style = applySGR(action.params, this.style)
      return []  // SGR 不产生动作，只更新状态
    }
    return [action]
  }
}
```

### 超链接状态跟踪

```typescript
case 'osc': {
  const action = parseOSC(content)
  if (action?.type === 'link') {
    if (action.action.type === 'start') {
      this.inLink = true
      this.linkUrl = action.action.url
    } else {
      this.inLink = false
      this.linkUrl = undefined
    }
  }
  return action ? [action] : []
}
```

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `getGraphemeSegmenter` | `../../utils/intl.js` | Unicode 字素分割 |
| `C0` | `./ansi.js` | 控制字符常量 |
| `CSI`, `CURSOR_STYLES`, `ERASE_DISPLAY`, `ERASE_LINE_REGION` | `./csi.js` | CSI 命令常量 |
| `DEC` | `./dec.js` | DEC 模式号常量 |
| `parseEsc` | `./esc.js` | ESC 序列解析 |
| `parseOSC` | `./osc.js` | OSC 序列解析 |
| `applySGR` | `./sgr.js` | SGR 参数应用 |
| `createTokenizer`, `Token`, `Tokenizer` | `./tokenize.js` | 序列分词 |
| `Action`, `Grapheme`, `TextStyle`, `defaultStyle` | `./types.js` | 类型定义 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `termio.ts` | `Parser` | 模块公开 API |
| `Ansi.tsx` | `Parser` | ANSI 字符串解析为 React 组件 |

### 导出内容

```typescript
export class Parser {
  style: TextStyle
  inLink: boolean
  linkUrl: string | undefined
  
  reset(): void
  feed(input: string): Action[]
}
```

## 依赖与外部交互

### 内部依赖

- `ansi.ts`：基础控制字符
- `csi.ts`：CSI 命令定义
- `dec.ts`：DEC 模式号
- `esc.ts`：ESC 序列解析
- `osc.ts`：OSC 序列解析
- `sgr.ts`：SGR 参数应用
- `tokenize.ts`：序列边界检测
- `types.ts`：类型定义
- `intl.ts`：Unicode 字素分割

### 外部使用场景

1. **Ansi 组件**：`Ansi.tsx` 使用 `Parser` 将 ANSI 字符串解析为 Ink 组件
   ```typescript
   const parser = new Parser()
   const actions = parser.feed(input)
   // 将 actions 转换为 React 元素
   ```

2. **样式跟踪**：Parser 维护当前样式状态，跨调用保持

3. **流式处理**：支持增量输入，适合处理大型输出

## 风险、边界与改进建议

### 边界情况

1. **SS3 序列**：识别为 `'ss3'` 类型但仅返回 `unknown`，未完整实现应用键盘模式解析
2. **不完整序列**：依赖 tokenizer 缓冲，parser 不处理不完整序列
3. **SGR 参数**：委托给 `applySGR`，parser 只传递参数字符串
4. **字素宽度**：emoji 和东亚字符的检测范围可能不完整

### 风险

1. **正则表达式性能**：`parseCSIParams` 使用正则分割，大量参数时可能有性能问题
2. **内存使用**：`feed()` 返回所有动作，大量输入可能累积大量动作对象
3. **样式状态泄漏**：`reset()` 必须正确调用，否则样式状态可能错误传递
4. **超链接状态**：`inLink` 和 `linkUrl` 状态需要正确维护，否则超链接嵌套可能出错

### 改进建议

1. **添加 SS3 支持**：
   ```typescript
   case 'ss3': {
     const byte = seq.charCodeAt(2)
     // 解析光标键、功能键等
     return parseSS3(byte)
   }
   ```

2. **性能优化**：
   - 使用生成器替代数组返回，支持流式处理
   - 缓存常用字素宽度结果
   - 优化 CSI 参数解析（避免正则）

3. **错误恢复**：
   - 添加无效序列的容错处理
   - 记录解析错误用于调试

4. **功能扩展**：
   - 支持更多 CSI 命令（插入/删除字符、行）
   - 支持 DCS 序列（设备控制字符串）
   - 支持 APC 序列（应用程序命令）

5. **文档改进**：
   - 添加支持的序列完整列表
   - 添加与 xterm、iTerm2 的兼容性说明

### 测试建议

- 测试各种 CSI 序列解析
- 测试 SGR 样式累积和重置
- 测试宽字符和 emoji 字素宽度
- 测试超链接开始/结束状态
- 测试增量输入处理
- 测试无效序列的容错
- 测试与 `Ansi.tsx` 的集成
