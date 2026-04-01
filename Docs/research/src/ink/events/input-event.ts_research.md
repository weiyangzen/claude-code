# input-event.ts 深度研究文档

## 场景与职责

`InputEvent` 是 Ink 中处理键盘输入的核心事件类，负责将底层的终端输入序列解析为结构化的键盘事件。这是连接原始终端输入与 Ink 应用逻辑的桥梁。

主要挑战：
1. **终端输入协议复杂**：需要支持多种键盘协议（传统转义序列、CSI u、modifyOtherKeys）
2. **跨终端兼容性**：不同终端模拟器发送的序列可能不同
3. **修饰键处理**：Ctrl、Alt、Shift 等修饰键的组合需要正确识别
4. **功能键映射**：方向键、F1-F12、Home/End 等需要统一映射

## 功能点目的

### 1. 键盘输入解析
将 `ParsedKey`（来自 `parse-keypress.ts` 的原始解析结果）转换为应用友好的 `Key` 对象和 `input` 字符串。

### 2. 修饰键标准化
统一处理不同协议中的修饰键表示：
- `ctrl`: Ctrl 键
- `shift`: Shift 键
- `meta`: Alt/Option 键
- `super`: Cmd (macOS) / Win 键
- `fn`: Fn 功能键

### 3. 输入字符提取
从复杂的转义序列中提取实际输入字符，处理各种边界情况：
- CSI u 序列（Kitty 键盘协议）
- modifyOtherKeys 序列
- 应用小键盘模式序列
- 控制字符转换

## 具体技术实现

### Key 类型定义

```typescript
export type Key = {
  upArrow: boolean
  downArrow: boolean
  leftArrow: boolean
  rightArrow: boolean
  pageDown: boolean
  pageUp: boolean
  wheelUp: boolean
  wheelDown: boolean
  home: boolean
  end: boolean
  return: boolean
  escape: boolean
  ctrl: boolean
  shift: boolean
  fn: boolean
  tab: boolean
  backspace: boolean
  delete: boolean
  meta: boolean
  super: boolean
}
```

### parseKey 函数

核心解析逻辑（lines 27-189），处理流程：

1. **基础键识别**：
   ```typescript
   const key: Key = {
     upArrow: keypress.name === 'up',
     downArrow: keypress.name === 'down',
     // ...
   }
   ```

2. **输入字符提取**：
   ```typescript
   let input = keypress.ctrl ? keypress.name : keypress.sequence
   ```

3. **边界情况处理**：
   - `undefined` 输入处理（lines 60-63）
   - Ctrl+Space 转换（lines 66-71）
   - 未识别功能键过滤（lines 78-80）
   - SGR 鼠标片段过滤（lines 82-92）

4. **协议特定处理**：

   **CSI u (Kitty 键盘协议)**（lines 105-130）：
   ```typescript
   if (/^\[\d/.test(input) && input.endsWith('u')) {
     // 处理 Kitty 协议序列
     // 如: [98;3u -> Alt+b
     if (!keypress.name) {
       input = ''  // 未映射的功能键，吞掉
     } else {
       input = keypress.name === 'space' ? ' ' : 
               keypress.name === 'escape' ? '' : 
               keypress.name
     }
   }
   ```

   **modifyOtherKeys**（lines 132-152）：
   ```typescript
   if (input.startsWith('[27;') && input.endsWith('~')) {
     // 处理 xterm modifyOtherKeys
     // 如: [27;3;98~ -> Alt+b
   }
   ```

   **应用小键盘模式**（lines 154-165）：
   ```typescript
   if (input.startsWith('O') && input.length === 2 && keypress.name?.length === 1) {
     // 处理小键盘数字
     // 如: Op -> 0, Oy -> 9
     input = keypress.name
   }
   ```

5. **非字母数字键过滤**（lines 167-176）：
   ```typescript
   if (!processedAsSpecialSequence && 
       keypress.name && 
       nonAlphanumericKeys.includes(keypress.name)) {
     input = ''
   }
   ```

6. **大写字母 Shift 标记**（lines 178-187）：
   ```typescript
   if (input.length === 1 && input[0] >= 'A' && input[0] <= 'Z') {
     key.shift = true
   }
   ```

### InputEvent 类

```typescript
export class InputEvent extends Event {
  readonly keypress: ParsedKey  // 原始解析结果
  readonly key: Key             // 结构化键信息
  readonly input: string        // 输入字符

  constructor(keypress: ParsedKey) {
    super()
    const [key, input] = parseKey(keypress)
    this.keypress = keypress
    this.key = key
    this.input = input
  }
}
```

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/input-event.ts` - InputEvent 类和 parseKey 函数

### 使用位置

1. **App.tsx**（lines 9, 506-507）:
   ```typescript
   import { InputEvent } from '../events/input-event.js'
   // ...
   const event = new InputEvent(item)
   app.internal_eventEmitter.emit('input', event)
   ```

2. **use-input.ts**（lines 3, 6, 69-79）:
   ```typescript
   import type { InputEvent, Key } from '../events/input-event.js'
   type Handler = (input: string, key: Key, event: InputEvent) => void
   // ...
   const handleData = useEventCallback((event: InputEvent) => {
     const { input, key } = event
     inputHandler(input, key, event)
   })
   ```

3. **parse-keypress.ts**（line 1）:
   ```typescript
   import { nonAlphanumericKeys, type ParsedKey } from '../parse-keypress.js'
   ```

### 调用链
```
App.processKeysInBatch (App.tsx:444)
  → parseMultipleKeypresses (parse-keypress.ts:213)
    → parseKeypress (parse-keypress.ts:611)
      → createNavKey / 返回 ParsedKey
  → new InputEvent(item) (App.tsx:506)
    → parseKey (input-event.ts:27)
      → 返回 [Key, string]
  → internal_eventEmitter.emit('input', event)
    → useInput.handleData (use-input.ts:69)
      → inputHandler(input, key, event)
```

## 依赖与外部交互

### 依赖

- `parse-keypress.ts` - `ParsedKey` 类型和 `nonAlphanumericKeys` 数组
- `event.ts` - `Event` 基类

### 被依赖

- `App.tsx` - 创建 InputEvent 并分发
- `use-input.ts` - 处理输入事件
- `event-handlers.ts` - 类型引用（通过 PasteEvent，未在批次中）
- `ink.ts` - 导出 Key 和 InputEvent 类型

### 与 parse-keypress.ts 的关系

```
parse-keypress.ts                    input-event.ts
├─ parseMultipleKeypresses()    →    InputEvent.constructor()
├─ parseKeypress()              →    parseKey()
├─ ParsedKey (type)             →    Key (type)
└─ nonAlphanumericKeys          →    用于过滤输入
```

`parse-keypress.ts` 负责：
- 从原始字节流中识别转义序列边界
- 解析出基本的键信息（name、sequence、modifiers）

`input-event.ts` 负责：
- 进一步规范化键信息
- 提取可打印的输入字符
- 处理协议特定的转换

## 风险、边界与改进建议

### 风险点

1. **协议兼容性问题**:
   - 不同终端对 CSI u 和 modifyOtherKeys 的支持程度不同
   - 某些边缘序列可能被错误解析

2. **输入丢失**:
   - 代码中有多个 `input = ''` 的防御性处理
   - 虽然防止了垃圾输入，但也可能误杀合法输入

3. **TODO 遗留**:
   ```typescript
   // TODO(vadimdemedes): consider removing this in the next major version.
   meta: keypress.meta || keypress.name === 'escape' || keypress.option,
   ```
   兼容性代码可能需要清理

4. **正则性能**:
   - 使用多个正则表达式匹配输入
   - 极端情况下可能影响性能

### 边界情况

1. **空输入**:
   ```typescript
   // lines 60-63
   if (input === undefined) {
     input = ''
   }
   ```

2. **CSI u 未映射键**:
   ```typescript
   // lines 112-116
   if (!keypress.name) {
     input = ''  // 吞掉未映射的 Kitty 功能键
   }
   ```

3. **SGR 鼠标片段**:
   ```typescript
   // lines 90-92
   if (!keypress.name && /^\[<\d+;\d+;\d+[Mm]/.test(input)) {
     input = ''  // 防御性处理鼠标序列片段
   }
   ```

4. **大写字母**:
   - 通过检查输入字符自动设置 `shift = true`
   - 依赖输入字符，而非修饰键标记

### 改进建议

1. **添加输入诊断**:
   ```typescript
   // 开发模式下记录未识别的序列
   if (process.env.NODE_ENV === 'development' && input === '') {
     console.log('Unrecognized input sequence:', keypress)
   }
   ```

2. **支持更多键盘协议**:
   ```typescript
   // 添加对 iTerm2 扩展键协议的支持
   // 添加对 Windows 终端协议的支持
   ```

3. **输入历史**:
   ```typescript
   // 在 InputEvent 中添加序列历史，便于调试
   readonly sequenceHistory: string[]
   ```

4. **可插拔解析器**:
   ```typescript
   // 允许注册自定义解析器
   InputEvent.registerParser(myCustomParser)
   ```

5. **类型安全增强**:
   ```typescript
   // 使用更精确的类型替代 string
   type InputChar = string  // 单个字符或空字符串
   type KeyName = 'up' | 'down' | 'left' | 'right' | ...
   ```

6. **清理兼容性代码**:
   ```typescript
   // 评估并移除 TODO 标记的兼容性代码
   // 添加版本检测，只在需要时启用兼容性处理
   ```

### 测试建议

1. 测试各种终端模拟器的输入序列
2. 测试修饰键组合（Ctrl+字母、Alt+字母、Shift+字母）
3. 测试功能键（F1-F12、方向键、Home/End）
4. 测试小键盘输入
5. 测试 Kitty 键盘协议的渐进增强
6. 测试 modifyOtherKeys 序列
7. 测试边界情况（空输入、未识别序列）
