# keyboard-event.ts 深度研究文档

## 场景与职责

`KeyboardEvent` 是 Ink 中用于 DOM 风格键盘事件分发的事件类，与 `InputEvent` 形成互补：

- **InputEvent**：通过 EventEmitter 广播，供 `useInput` hook 使用
- **KeyboardEvent**：通过 Dispatcher 分发，支持捕获/冒泡，供 `onKeyDown` 等属性使用

这种双轨制设计允许：
1. 全局输入处理（`useInput`）
2. 组件级键盘事件处理（`onKeyDown`）

## 功能点目的

### 1. DOM 风格键盘事件
模拟浏览器 `KeyboardEvent` API：
- `key`: 按键的字符值（如 'a'、'Enter'、'ArrowDown'）
- `ctrl`, `shift`, `meta`, `superKey`, `fn`: 修饰键状态

### 2. 可打印字符检测
遵循浏览器约定：可打印字符的 `key.length === 1`

### 3. 两阶段事件分发
通过继承 `TerminalEvent`，支持：
- 捕获阶段处理器（`onKeyDownCapture`）
- 冒泡阶段处理器（`onKeyDown`）

## 具体技术实现

### 类定义

```typescript
export class KeyboardEvent extends TerminalEvent {
  readonly key: string
  readonly ctrl: boolean
  readonly shift: boolean
  readonly meta: boolean
  readonly superKey: boolean
  readonly fn: boolean

  constructor(parsedKey: ParsedKey) {
    super('keydown', { bubbles: true, cancelable: true })
    this.key = keyFromParsed(parsedKey)
    this.ctrl = parsedKey.ctrl
    this.shift = parsedKey.shift
    this.meta = parsedKey.meta || parsedKey.option
    this.superKey = parsedKey.super
    this.fn = parsedKey.fn
  }
}
```

### keyFromParsed 函数

核心逻辑（lines 32-51）：

```typescript
function keyFromParsed(parsed: ParsedKey): string {
  const seq = parsed.sequence ?? ''
  const name = parsed.name ?? ''

  // 1. Ctrl 组合：返回字母本身
  if (parsed.ctrl) return name

  // 2. 单可打印字符：返回字符本身
  if (seq.length === 1) {
    const code = seq.charCodeAt(0)
    if (code >= 0x20 && code !== 0x7f) return seq
  }

  // 3. 特殊键：返回名称
  return name || seq
}
```

### 与浏览器 KeyboardEvent 的对比

| 属性 | Ink KeyboardEvent | DOM KeyboardEvent | 说明 |
|-----|-------------------|-------------------|------|
| key | 字符或键名 | 字符或键名 | 行为一致 |
| ctrlKey | ctrl | ctrlKey | 命名不同 |
| shiftKey | shift | shiftKey | 命名不同 |
| altKey | meta | altKey | Ink 使用 meta（历史原因） |
| metaKey | superKey | metaKey | Ink 的 super 对应 Cmd/Win |
| code | 不支持 | code | Ink 不提供物理键码 |

### 设计决策

1. **使用 `meta` 而非 `alt`**:
   - 历史原因：Ink 早期版本将 Alt/Option 映射为 meta
   - 与 `InputEvent.key.meta` 保持一致

2. **使用 `superKey` 而非 `meta`**:
   - 区分 Alt (meta) 和 Cmd/Win (super)
   - 只在支持 Kitty 键盘协议的终端可用

3. **只支持 keydown**:
   - 终端通常只发送按键按下事件
   - 不支持 keyup/keypress

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/keyboard-event.ts` - KeyboardEvent 类定义

### 使用位置

1. **ink.tsx**（line 19）:
   ```typescript
   import { KeyboardEvent } from './events/keyboard-event.js'
   ```
   在 `dispatchKeyboardEvent` 方法中创建和分发事件。

2. **Box.tsx**（lines 8, 35-36）:
   ```typescript
   import type { KeyboardEvent } from '../events/keyboard-event.js'
   onKeyDown?: (event: KeyboardEvent) => void
   onKeyDownCapture?: (event: KeyboardEvent) => void
   ```

3. **event-handlers.ts**（lines 3, 7, 22-23, 48）:
   ```typescript
   import type { KeyboardEvent } from './keyboard-event.js'
   type KeyboardEventHandler = (event: KeyboardEvent) => void
   onKeyDown?: KeyboardEventHandler
   onKeyDownCapture?: KeyboardEventHandler
   ```

4. **keybindings/useKeybinding.ts**:
   使用 KeyboardEvent 进行快捷键绑定。

### 调用链
```
App.processKeysInBatch (App.tsx:510)
  → props.dispatchKeyboardEvent(item) (ink.tsx)
    → new KeyboardEvent(parsedKey)
    → dispatcher.dispatchDiscrete(target, event) (focus.ts:234)
      → dispatch (dispatcher.ts:185)
        → collectListeners (dispatcher.ts:46)
        → processDispatchQueue (dispatcher.ts:87)
          → handler(event)  // onKeyDown/onKeyDownCapture
```

## 依赖与外部交互

### 依赖

- `terminal-event.ts` - `TerminalEvent` 基类
- `parse-keypress.ts` - `ParsedKey` 类型

### 被依赖

- `ink.tsx` - 创建和分发键盘事件
- `event-handlers.ts` - 类型定义
- `Box.tsx` - 组件 Props 类型
- `keybindings/useKeybinding.ts` - 快捷键处理

### 与 InputEvent 的关系

```
原始输入
  ├─→ parse-keypress.ts
  │     ├─→ InputEvent ──→ EventEmitter ──→ useInput
  │     └─→ KeyboardEvent ──→ Dispatcher ──→ onKeyDown
  │
  └─→ 两者都基于 ParsedKey
```

**InputEvent** 适合：
- 全局输入监听
- 命令行风格的应用
- 需要原始输入字符的场景

**KeyboardEvent** 适合：
- 组件级事件处理
- 需要捕获/冒泡的场景
- 表单、按钮等交互组件

## 风险、边界与改进建议

### 风险点

1. **与 InputEvent 的重复**:
   - 两者都从 `ParsedKey` 构建
   - 维护成本增加
   - 可能产生不一致的行为

2. **key 值不一致**:
   - `InputEvent.input` 和 `KeyboardEvent.key` 可能不同
   - 例如 CSI u 序列的处理

3. **缺少 keyup**:
   - 无法检测按键释放
   - 影响某些交互模式（如按住拖动）

4. **修饰键命名混淆**:
   - `meta` 在 Ink 中是 Alt，在 DOM 中是 Cmd/Win
   - 可能导致跨平台代码困惑

### 边界情况

1. **空序列**:
   ```typescript
   const seq = parsed.sequence ?? ''
   const name = parsed.name ?? ''
   // 两者都为空时返回空字符串
   return name || seq  // ''
   ```

2. **控制字符**:
   - `code !== 0x7f` 排除 DEL 字符
   - 但 0x7f 在某些终端是 Backspace

3. **多字节字符**:
   - `seq.length === 1` 检查可能不适用于所有 Unicode
   - 但终端输入通常是单字节或已解析的序列

### 改进建议

1. **统一事件模型**:
   ```typescript
   // 考虑合并 InputEvent 和 KeyboardEvent
   // 或者明确分离职责
   ```

2. **添加 keyup 支持**:
   ```typescript
   // 如果终端支持按键释放报告
   // 某些协议（如 Kitty）支持
   export class KeyboardEvent extends TerminalEvent {
     readonly type: 'keydown' | 'keyup'
   }
   ```

3. **添加 repeat 属性**:
   ```typescript
   // 检测自动重复按键（长按）
   readonly repeat: boolean
   ```

4. **添加 location 属性**:
   ```typescript
   // 区分小键盘和主键盘的数字
   readonly location: 'standard' | 'numpad' | ...
   ```

5. **标准化修饰键命名**:
   ```typescript
   // 考虑添加 altKey 别名
   get altKey(): boolean { return this.meta }
   ```

6. **添加组合键解析**:
   ```typescript
   // 辅助方法判断组合键
   matches(shortcut: string): boolean {
     // 'Ctrl+Shift+A'
   }
   ```

### 测试建议

1. 测试各种键的 `key` 值（字母、数字、符号、功能键）
2. 测试修饰键组合
3. 测试捕获和冒泡阶段
4. 测试 `stopPropagation` 和 `preventDefault`
5. 测试与 `useInput` 的互操作性
6. 测试不同终端的兼容性
