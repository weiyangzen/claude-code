# useTextInput.ts 深度研究文档

## 场景与职责

`useTextInput` 是一个功能丰富的 React Hook，用于处理文本输入。它提供了完整的键盘编辑功能，包括 Emacs 风格快捷键、Kill-Yank 系统、多行输入支持等。

### 核心职责

1. **文本输入处理**: 处理所有键盘输入事件
2. **光标管理**: 管理光标位置和移动
3. **快捷键支持**: 实现 Emacs 风格的编辑快捷键
4. **Kill-Yank 系统**: 支持文本剪切和粘贴
5. **多行输入**: 支持多行文本输入
6. **历史导航**: 支持上下键触发历史导航

### 使用场景

- **命令输入**: REPL 主输入框
- **搜索输入**: 各种搜索对话框
- **表单输入**: 配置和设置输入
- **任何需要文本输入的组件**

---

## 功能点目的

### 1. 基础编辑功能

- 字符输入和删除
- 光标移动（字符、单词、行级别）
- 文本选择（通过 Kill-Yank 间接支持）

### 2. Emacs 风格快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+A` | 行首 |
| `Ctrl+E` | 行尾 |
| `Ctrl+B` | 左移 |
| `Ctrl+F` | 右移 |
| `Ctrl+D` | 删除字符（空输入时退出） |
| `Ctrl+H` | 退格 |
| `Ctrl+K` | 剪切到行尾 |
| `Ctrl+U` | 剪切到行首 |
| `Ctrl+W` | 剪切单词 |
| `Ctrl+Y` | 粘贴（Yank） |
| `Ctrl+N` | 下移/历史下 |
| `Ctrl+P` | 上移/历史上 |

### 3. Meta（Alt）快捷键

| 快捷键 | 功能 |
|--------|------|
| `Meta+B` | 上一个单词 |
| `Meta+F` | 下一个单词 |
| `Meta+D` | 删除下一个单词 |
| `Meta+Y` | 循环粘贴（Yank-Pop） |

### 4. 特殊功能

- **双按退出**: `Ctrl+C` 和 `Ctrl+D` 需要双按确认退出
- **Escape 双按**: 双按 Escape 清空输入并保存到历史
- **反斜杠换行**: 支持 `\` + Enter 的多行输入
- **Meta/Shift+Enter**: 插入换行
- **DEL 字符处理**: 过滤 SSH/tmux 环境中的 DEL 字符

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface UseTextInputProps {
  value: string                           // 当前值
  onChange: (value: string) => void       // 变化回调
  onSubmit?: (value: string) => void      // 提交回调
  onExit?: () => void                     // 退出回调
  onExitMessage?: (show: boolean, key?: string) => void
  onHistoryUp?: () => void                // 历史上回调
  onHistoryDown?: () => void              // 历史下回调
  onHistoryReset?: () => void             // 历史重置回调
  onClearInput?: () => void               // 清空回调
  focus?: boolean                         // 是否聚焦
  mask?: string                           // 掩码字符（密码输入）
  multiline?: boolean                     // 是否多行
  cursorChar: string                      // 光标字符
  columns: number                         // 列数
  onImagePaste?: (...) => void           // 图片粘贴回调
  disableCursorMovementForUpDownKeys?: boolean  // 禁用上下键光标移动
  disableEscapeDoublePress?: boolean      // 禁用 Escape 双按
  maxVisibleLines?: number                // 最大可见行数
  externalOffset: number                  // 外部光标偏移
  onOffsetChange: (offset: number) => void  // 偏移变化回调
  inputFilter?: (input: string, key: Key) => string  // 输入过滤器
  inlineGhostText?: InlineGhostText       // 内联幽灵文本
  dim?: (text: string) => string          // 暗淡文本函数
}

// 返回类型
interface TextInputState {
  onInput: (input: string, key: Key) => void
  renderedValue: string
  offset: number
  setOffset: (offset: number) => void
  cursorLine: number
  cursorColumn: number
  viewportCharOffset: number
  viewportCharEnd: number
}
```

### 核心流程

#### 1. 输入处理流程
```
onInput 调用
  ↓
应用 inputFilter（如果提供）
  ↓
检查过滤后的输入
  ↓
处理 DEL 字符（SSH/tmux 环境）
  ↓
重置 Kill/Yank 状态（非 kill/yank 键）
  ↓
根据 key 路由到对应处理器:
  - Escape → handleEscape
  - Ctrl+Key → handleCtrl
  - Meta+Key → handleMeta
  - 方向键 → 光标移动
  - Home/End → 行首/行尾
  - PageUp/PageDown → 行首/行尾（非全屏）
  - Enter → handleEnter
  - 其他 → 默认字符输入
  ↓
更新光标状态
  ↓
检查 SSH 合并的 Enter
```

#### 2. Kill-Yank 处理
```typescript
// 判断是否为 kill 键
function isKillKey(key: Key, input: string): boolean {
  if (key.ctrl && (input === 'k' || input === 'u' || input === 'w')) {
    return true
  }
  if (key.meta && (key.backspace || key.delete)) {
    return true
  }
  return false
}

// 处理时重置累积状态
if (!isKillKey(key, filteredInput)) {
  resetKillAccumulation()
}
```

#### 3. 双按退出处理
```typescript
const handleCtrlC = useDoublePress(
  show => onExitMessage?.(show, 'Ctrl-C'),  // 第一次按下
  () => onExit?.(),                          // 第二次按下
  () => {                                     // 取消（输入非空时）
    if (originalValue) {
      onChange('')
      setOffset(0)
      onHistoryReset?.()
    }
  },
)
```

### 关键代码路径

#### DEL 字符处理（行 442-465）
```typescript
// Fix Issue #1853: Filter DEL characters that interfere with backspace in SSH/tmux
if (!key.backspace && !key.delete && input.includes('\x7f')) {
  const delCount = (input.match(/\x7f/g) || []).length
  let currentCursor = cursor
  for (let i = 0; i < delCount; i++) {
    currentCursor = currentCursor.deleteTokenBefore() ?? currentCursor.backspace()
  }
  if (!cursor.equals(currentCursor)) {
    if (cursor.text !== currentCursor.text) {
      onChange(currentCursor.text)
    }
    setOffset(currentCursor.offset)
  }
  resetKillAccumulation()
  resetYankState()
  return
}
```

#### SSH 合并 Enter 处理（行 490-499）
```typescript
// SSH-coalesced Enter: on slow links, "o" + Enter can arrive as one chunk "o\r"
if (
  filteredInput.length > 1 &&
  filteredInput.endsWith('\r') &&
  !filteredInput.slice(0, -1).includes('\r') &&
  filteredInput[filteredInput.length - 2] !== '\\'
) {
  onSubmit?.(nextCursor.text)
}
```

#### 幽灵文本渲染（行 506-509）
```typescript
const ghostTextForRender =
  inlineGhostText && dim && inlineGhostText.insertPosition === offset
    ? { text: inlineGhostText.text, dim }
    : undefined
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `strip-ansi` | 去除 ANSI 转义序列 |
| `../commands/terminalSetup/terminalSetup.js` | `markBackslashReturnUsed` |
| `../history.js` | `addToHistory` |
| `../ink.js` | `Key` 类型 |
| `../types/textInputTypes.js` | `InlineGhostText`, `TextInputState` |
| `../utils/Cursor.js` | `Cursor` 类和 Kill-Yank 函数 |
| `../utils/env.js` | `env` |
| `../utils/fullscreen.js` | `isFullscreenEnvEnabled` |
| `../utils/imageResizer.js` | `ImageDimensions` |
| `../utils/modifiers.js` | `isModifierPressed`, `prewarmModifiers` |
| `./useDoublePress.js` | `useDoublePress` |

### 外部交互

1. **Cursor 类**: 
   - 所有文本操作通过 `Cursor` 类完成
   - 提供不可变的文本操作方法

2. **Kill-Yank 系统**: 
   - 全局共享的 kill ring
   - 连续剪切累积到同一项

3. **历史系统**: 
   - `addToHistory`: 保存输入到历史
   - `onHistoryUp/Down`: 触发历史导航

---

## 风险、边界与改进建议

### 已知风险

1. **全局 Kill Ring**: 所有输入字段共享同一个 kill ring
2. **Apple Terminal 特殊处理**: 需要预热修饰键检测
3. **SSH/tmux 兼容**: 多种环境特殊处理增加复杂性

### 边界情况

1. **多字节字符**: Emoji、CJK 字符的正确处理
2. **终端宽度变化**: 列数变化时的光标定位
3. **批量输入**: 粘贴时的批量字符处理
4. **掩码输入**: 密码输入时的显示处理

### 改进建议

1. **Kill Ring 隔离**: 为不同类型的输入提供独立的 kill ring
2. **输入法支持**: 更好地支持 CJK 输入法
3. **语音输入**: 支持语音转文本输入
4. **自动完成**: 集成更智能的自动完成功能
5. **语法高亮**: 输入时实时语法高亮
6. **撤销/重做**: 支持输入历史的撤销/重做

### 测试关注点

1. 各种键盘快捷键的正确性
2. Kill-Yank 累积和循环行为
3. 多字节字符处理
4. 双按退出逻辑
5. DEL 字符过滤
6. SSH 合并 Enter 检测
7. 幽灵文本显示
