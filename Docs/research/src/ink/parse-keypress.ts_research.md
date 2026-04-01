# Research: src/ink/parse-keypress.ts

## 场景与职责

`parse-keypress.ts` 是 Ink 的**终端输入解析器**。它从 `stdin` 读取原始字节流（`Buffer` 或 `string`），将其拆分为有意义的输入事件：普通按键、组合键、鼠标事件、粘贴内容，以及终端对查询的响应序列（如 DA1、DECRPM、OSC 回复）。该模块是 Ink 事件系统的最底层，直接面对终端模拟器的各种转义序列协议，兼容性是核心挑战。

## 功能点目的

1. **按键事件解析**：将 ESC 序列、CSI 序列、OSC 序列等转换为结构化的 `ParsedKey`。
2. **鼠标事件解析**：支持 SGR 鼠标协议和 X10 鼠标协议，提取点击/拖拽/滚轮坐标。
3. **终端响应解析**：识别终端对 DECRQM、DA1/DA2、Kitty keyboard、光标位置、OSC、XTVERSION 等查询的回复，与按键事件区分。
4. **粘贴模式**：支持 bracketed paste（`\e[200~` ... `\e[201~`），将粘贴内容作为单个事件发出。
5. **多协议兼容**：同时支持传统 xterm 序列、Kitty keyboard protocol（CSI u）、xterm `modifyOtherKeys`、应用键盘模式（application keypad）等。

## 具体技术实现

### 核心数据结构

```ts
export type ParsedKey = {
  kind: 'key'
  fn: boolean
  name: string | undefined
  ctrl: boolean
  meta: boolean
  shift: boolean
  option: boolean
  super: boolean
  sequence: string | undefined
  raw: string | undefined
  code?: string
  isPasted: boolean
}

export type ParsedMouse = {
  kind: 'mouse'
  button: number
  action: 'press' | 'release'
  col: number
  row: number
  sequence: string
}

export type ParsedResponse = {
  kind: 'response'
  sequence: string
  response: TerminalResponse
}

export type ParsedInput = ParsedKey | ParsedMouse | ParsedResponse
```

### 状态机 `KeyParseState`

```ts
export type KeyParseState = {
  mode: 'NORMAL' | 'IN_PASTE'
  incomplete: string
  pasteBuffer: string
  _tokenizer?: Tokenizer
}

export const INITIAL_STATE: KeyParseState = {
  mode: 'NORMAL',
  incomplete: '',
  pasteBuffer: '',
}
```

- `mode`：是否在 bracketed paste 区域内。
- `incomplete`：tokenizer 中未完成的序列尾部。
- `pasteBuffer`：累积的粘贴内容。
- `_tokenizer`：复用的 `Tokenizer` 实例，避免每帧重建。

### 主入口 `parseMultipleKeypresses`

```ts
export function parseMultipleKeypresses(
  prevState: KeyParseState,
  input: Buffer | string | null = ''
): [ParsedInput[], KeyParseState]
```

流程：
1. `input === null` 表示 flush（如定时器触发或流结束）。
2. `inputToString(input)`：将 `Buffer` 转为字符串，处理高位字节（`>127` 且单字节时映射为 `\x1b + char`）。
3. 获取或创建 `Tokenizer`（来自 `./termio/tokenize.js`，启用 `x10Mouse: true`）。
4. `tokenizer.feed(inputString)` 或 `tokenizer.flush()` 得到 `Token[]`。
5. 遍历 tokens：
   - `sequence` 类型：
     - `PASTE_START`（`\e[200~`）→ 进入粘贴模式；
     - `PASTE_END`（`\e[201~`）→ 发出 `createPasteKey(pasteBuffer)`，退出粘贴模式；
     - 粘贴模式中的序列 → 追加到 `pasteBuffer`；
     - 否则先尝试 `parseTerminalResponse`，再尝试 `parseMouseEvent`，最后回退到 `parseKeypress`。
   - `text` 类型：
     - 粘贴模式中 → 追加到 `pasteBuffer`；
     - 否则检测是否为"孤儿"鼠标序列尾部（因事件循环阻塞导致 ESC 被提前 flush），重新合成后解析；
     - 最后回退到 `parseKeypress`。
6. flush 时若仍在粘贴模式，发出剩余内容。
7. 返回解析结果和新状态。

### `parseTerminalResponse` 终端响应解析

支持的响应类型：

| 类型 | 正则 | 说明 |
|------|------|------|
| DECRPM | `^\x1b\[\?(\d+);(\d+)\$y$` | 模式状态查询回复 |
| DA1 | `^\x1b\[\?([\d;]*)c$` | 主设备属性 |
| DA2 | `^\x1b\[>([\d;]*)c$` | 次设备属性 |
| Kitty Keyboard | `^\x1b\[\?(\d+)u$` | Kitty 键盘协议标志 |
| Cursor Position | `^\x1b\[\?(\d+);(\d+)R$` | 光标位置报告（带 `?` 区分于 F3 键） |
| OSC | `^\x1b\](\d+);(.*?)\x07$` 或 `\x1b\\` | 通用 OSC 回复 |
| XTVERSION | `^\x1bP>\|(.*?)(?:\x07|\x1b\\)$` | 终端名称/版本 |

### `parseKeypress` 按键解析

按优先级依次匹配：

1. **CSI u（Kitty keyboard protocol）**：`^\x1b\[(\d+)(?:;(\d+))?u`
   - codepoint → `keycodeToName`；modifier 默认 1（无修饰符）。
2. **xterm modifyOtherKeys**：`^\x1b\[27;(\d+);(\d+)~`
   - 参数顺序与 CSI u 相反：先 modifier，后 keycode。
3. **SGR 鼠标滚轮**：`^\x1b\[<(\d+);(\d+);(\d+)([Mm])$`
   - 滚轮事件（bit 6 置位）作为 `ParsedKey`（`wheelup`/`wheeldown`）返回；点击/拖拽由 `parseMouseEvent` 处理为 `ParsedMouse`。
4. **X10 鼠标**：`^\x1b\[M[\x20-\x7f]{3}`（6 字节序列）
   - 同样只处理滚轮，其他忽略。
5. **单字节控制字符**：`\r` → `return`；`\n` → `enter`；`\t` → `tab`；`\b`/`\x7f` → `backspace`；`\x1b` → `escape`；` ` → `space`；`\x01`-`\x1a` → `ctrl+a`..`ctrl+z`。
6. **可打印字符**：数字 → `number`；小写字母 → 字母本身；大写字母 → 小写 + `shift=true`。
7. **Meta 键**：`^\x1b([a-zA-Z0-9])$` → `meta=true`。
8. **功能键（FN_KEY_RE）**：`^\x1b+(O|N|\[|\[\[)(?:(\d+)(?:;(\d+))?([~^$])|(?:1;)?(\d+)?([a-zA-Z]))`
   - 匹配方向键、F1-F12、Home/End/Insert/Delete/PageUp/PageDown 等。
   - 通过 `keyName` 映射表和 `decodeModifier` 解析修饰键。

### `keycodeToName` 映射

- ASCII 控制字符：`tab` (9)、`return` (13)、`escape` (27)、`space` (32)、`backspace` (127)。
- Kitty numpad：使用 Unicode PUA 码点（57399-57415）映射到 `0-9`、`.`、`/`、`*`、`-`、`+`、`return`、`=`。
- 可打印 ASCII（32-126）：直接转为小写字母。

### `decodeModifier`

XTerm 风格修饰符编码：`modifier = 1 + (shift ? 1 : 0) + (alt ? 2 : 0) + (ctrl ? 4 : 0) + (super ? 8 : 0)`。

### `parseMouseEvent`

```ts
function parseMouseEvent(s: string): ParsedMouse | null
```

- 仅解析 SGR 鼠标序列（`CSI < btn ; col ; row M/m`）。
- 若 button 的 bit 6（`0x40`）置位，表示滚轮，返回 `null`（由上层作为 `wheelup`/`wheeldown` 处理）。
- 非滚轮的点击/拖拽返回 `ParsedMouse`，包含 1-indexed 的 `col`/`row`。

### `nonAlphanumericKeys`

用于 `input-event.ts` 的输入过滤：若按键名在此列表中，则清空 `input` 字符串，防止功能键（如方向键）泄漏到文本输入框。

## 关键代码路径与文件引用

- **调用方**：
  - `src/ink/components/App.tsx:11` — `App` 组件在 `stdin` data 事件和定时器中调用 `parseMultipleKeypresses`。
- **消费方**：
  - `src/ink/events/input-event.ts` — 将 `ParsedKey` 转换为 `Key` + `input` 字符串。
  - `src/ink/events/keyboard-event.ts` — 将 `ParsedKey` 包装为 `KeyboardEvent`。
  - `src/ink/terminal-querier.ts` — 消费 `TerminalResponse` 类型。
  - `src/ink/ink.tsx:28` — `Ink` 类导入 `ParsedKey` 类型。
- **依赖模块**：
  - `src/ink/termio/tokenize.js` — 提供 `createTokenizer`。
  - `src/ink/termio/csi.js` — 提供 `PASTE_START` / `PASTE_END` 常量。

## 依赖与外部交互

- 无直接 I/O，纯解析函数。
- 输入来自 `stdin` 的原始字节，由 `App.tsx` 传入。

## 风险、边界与改进建议

- **风险**：终端输入协议极其碎片化，同一物理键在不同终端/SSH/tmux 下可能产生不同序列。正则匹配顺序错误会导致误解析（如 `MODIFY_OTHER_KEYS_RE` 必须在 `FN_KEY_RE` 之前，否则部分匹配会留下垃圾尾部）。
- **边界**：
  - `FN_KEY_RE` 是复杂正则，对未识别序列（如 F13+）可能部分匹配，导致序列尾部泄漏为文本。`input-event.ts` 有额外的 `keypress.code && !keypress.name` 防护。
  - 孤儿鼠标序列的检测（`/^\[<\d+;\d+;\d+[Mm]$/`）是防御性补丁，依赖具体序列格式，未来若终端行为变化可能失效。
  - `inputToString` 对单字节高位 Buffer 的 `-128` 处理是历史兼容逻辑，对现代 UTF-8 输入可能不正确。
- **改进建议**：
  1. **协议分层**：将 tokenizer、序列分类器、按键解析器进一步解耦，便于单元测试和新增协议（如 iTerm2 扩展键）。
  2. **模糊匹配兜底**：对未识别的 ESC 序列，统一吞掉而不是部分解析，减少垃圾字符泄漏。
  3. 增加对 `CSI u` 功能键（如 F13-F35、媒体键）的映射，避免这些键被当作文本输入。
  4. `parseMultipleKeypresses` 的返回值使用元组 `[ParsedInput[], KeyParseState]`，可考虑改为对象以提升可读性。
  5. 对 `Tokenizer` 的 `x10Mouse: true` 配置，可探索动态开关：在非 alt-screen 模式下不需要鼠标跟踪，关闭可减少 tokenizer 状态机开销。
