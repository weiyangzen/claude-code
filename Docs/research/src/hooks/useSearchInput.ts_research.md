# useSearchInput.ts 深度研究文档

## 场景与职责

`useSearchInput` 是一个专门用于搜索/查找输入的 React Hook，提供类似 Vim/less 的搜索体验。它支持丰富的键盘快捷键、Emacs 风格的编辑命令，以及 Kill-Yank（剪切-粘贴）功能。

### 核心职责

1. **搜索输入管理**：处理搜索查询字符串的输入和编辑
2. **光标控制**：支持精确的光标定位和移动
3. **键盘快捷键**：实现 Emacs/Vim 风格的编辑快捷键
4. **Kill-Yank 系统**：支持文本剪切和粘贴（类似 Emacs 的 kill ring）
5. **导航集成**：支持与外部导航（如历史记录）的集成

### 使用场景

- **全局搜索对话框** (`GlobalSearchDialog`): 文件/符号搜索
- **历史搜索** (`HistorySearchDialog`): 命令历史搜索
- **帮助系统** (`HelpV2`): 帮助内容搜索
- **日志选择器** (`LogSelector`): 日志过滤
- **任何需要搜索输入的组件**

---

## 功能点目的

### 1. 搜索查询编辑

提供完整的文本编辑功能：
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
| `Ctrl+D` | 删除字符（或空输入时退出） |
| `Ctrl+H` | 退格 |
| `Ctrl+K` | 剪切到行尾 |
| `Ctrl+U` | 剪切到行首 |
| `Ctrl+W` | 剪切单词 |
| `Ctrl+Y` | 粘贴（Yank） |

### 3. Meta（Alt）快捷键

| 快捷键 | 功能 |
|--------|------|
| `Meta+B` | 上一个单词 |
| `Meta+F` | 下一个单词 |
| `Meta+D` | 删除下一个单词 |
| `Meta+Y` | 循环粘贴（Yank-Pop） |

### 4. 导航快捷键

| 快捷键 | 功能 |
|--------|------|
| `Enter` / `↓` | 确认搜索并退出 |
| `↑` | 向上退出（到历史记录） |
| `Esc` | 取消（或清空后退出） |
| `Backspace` | 退格（空输入时可退出） |

### 5. Kill-Yank 系统

全局共享的 kill ring：
- 连续剪切会累积到同一个 kill ring 条目
- 支持 `prepend` 和 `append` 两种累积方向
- `Meta+Y` 循环历史剪切内容

---

## 具体技术实现

### 关键数据结构

```typescript
interface UseSearchInputOptions {
  isActive: boolean                    // 是否激活输入
  onExit: () => void                   // 确认退出回调
  onCancel?: () => void                // 取消回调（优先于 onExit）
  onExitUp?: () => void                // 向上退出回调
  columns?: number                     // 列数（默认使用终端宽度）
  passthroughCtrlKeys?: string[]        // 透传的 Ctrl 键
  initialQuery?: string                // 初始查询字符串
  backspaceExitsOnEmpty?: boolean      // 空输入时退格是否退出
}

interface UseSearchInputReturn {
  query: string                        // 当前查询字符串
  setQuery: (q: string) => void        // 设置查询（重置光标）
  cursorOffset: number                 // 光标偏移量
  handleKeyDown: (e: KeyboardEvent) => void  // 键盘处理器
}
```

### 核心流程

#### 1. 键盘事件处理流程
```
handleKeyDown 调用
  ↓
检查 isActive → 未激活直接返回
  ↓
创建 Cursor 对象（基于当前 query、columns、cursorOffset）
  ↓
检查透传键 → 匹配则直接返回
  ↓
重置 Kill/Yank 状态（非 kill/yank 键）
  ↓
按键分类处理:
  - Return/Down → onExit()
  - Up → onExitUp()
  - Escape → onCancel() 或清空
  - Backspace → 退格处理
  - Delete → 删除字符
  - 方向键（带修饰符）→ 单词跳转
  - 方向键 → 字符移动
  - Home/End → 行首/行尾
  - Ctrl+Key → Emacs 命令
  - Meta+Key → 扩展命令
  - Tab → 忽略
  - 普通字符 → 插入文本
```

#### 2. Kill 操作处理
```typescript
// 判断是否为 kill 键
function isKillKey(e: KeyboardEvent): boolean {
  if (e.ctrl && (e.key === 'k' || e.key === 'u' || e.key === 'w')) {
    return true
  }
  if (e.meta && e.key === 'backspace') {
    return true
  }
  return false
}

// 处理时重置累积状态
if (!isKillKey(e)) {
  resetKillAccumulation()
}
```

#### 3. Cursor 操作示例
```typescript
// Ctrl+K: 剪切到行尾
case 'k': {
  const { cursor: newCursor, killed } = cursor.deleteToLineEnd()
  pushToKillRing(killed, 'append')
  setQueryState(newCursor.text)
  setCursorOffset(newCursor.offset)
  return
}

// Meta+Y: 循环粘贴
case 'y': {
  const popResult = yankPop()
  if (popResult) {
    const { text, start, length } = popResult
    const before = query.slice(0, start)
    const after = query.slice(start + length)
    const newText = before + text + after
    const newOffset = start + text.length
    updateYankLength(text.length)
    setQueryState(newText)
    setCursorOffset(newOffset)
  }
  return
}
```

### 关键代码路径

#### 特殊键过滤（行 63-82）
```typescript
const UNHANDLED_SPECIAL_KEYS = new Set([
  'pageup', 'pagedown', 'insert',
  'wheelup', 'wheeldown', 'mouse',
  'f1', 'f2', 'f3', 'f4', 'f5', 'f6',
  'f7', 'f8', 'f9', 'f10', 'f11', 'f12',
])

// 在默认分支中过滤
if (e.key.length >= 1 && !UNHANDLED_SPECIAL_KEYS.has(e.key)) {
  // 处理普通字符输入
}
```

#### 退格退出逻辑（行 161-166）
```typescript
if (query.length === 0) {
  // Backspace past the / — cancel (clear + snap back), not commit.
  // less: same. vim: deletes the / and exits command mode.
  if (backspaceExitsOnEmpty) (onCancel ?? onExit)()
  return
}
```

#### 向后兼容桥接（行 352-361）
```typescript
// 现有消费者尚未迁移到 onKeyDown 模式
// 通过 useInput 订阅并适配 InputEvent → KeyboardEvent
useInput(
  (_input, _key, event) => {
    handleKeyDown(new KeyboardEvent(event.keypress))
  },
  { isActive },
)
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `../ink/events/keyboard-event.js` | `KeyboardEvent` 类型 |
| `../ink.js` | `useInput` Hook |
| `../utils/Cursor.js` | `Cursor` 类和 Kill-Yank 函数 |
| `./useTerminalSize.js` | 终端尺寸获取 |

### Cursor 类功能

`Cursor` 类（来自 `../utils/Cursor.js`）提供：
- `fromText(text, columns, offset)`: 创建 Cursor 实例
- `left()/right()`: 字符级移动
- `prevWord()/nextWord()`: 单词级移动
- `backspace()/del()`: 删除操作
- `deleteToLineStart()/deleteToLineEnd()`: 行级删除
- `deleteWordBefore()/deleteWordAfter()`: 单词删除
- `insert(text)`: 文本插入
- `startOfLine()/endOfLine()`: 行首/行尾

### Kill-Yank 全局函数

```typescript
// 来自 ../utils/Cursor.js
pushToKillRing(text, direction: 'prepend' | 'append')
getLastKill(): string
resetKillAccumulation()
recordYank(start, length)
yankPop(): { text, start, length } | null
updateYankLength(length)
resetYankState()
```

---

## 风险、边界与改进建议

### 已知风险

1. **向后兼容桥接**: `useInput` 桥接模式将在所有调用方迁移后移除，需要跟踪迁移进度
2. **全局 Kill Ring**: 所有输入字段共享同一个 kill ring，可能导致意外的粘贴内容
3. **多字节字符**: 依赖 `Cursor` 类的 grapheme 处理，需要确保 Unicode 正确性

### 边界情况

1. **空查询处理**: `backspaceExitsOnEmpty` 控制空查询时的退格行为
2. **列数变化**: 终端尺寸变化时 `Cursor` 需要重新计算换行
3. **批量输入**: `e.key.length >= 1` 支持批量输入（如粘贴）
4. **修饰键冲突**: `passthroughCtrlKeys` 允许某些 Ctrl 快捷键透传给父组件

### 改进建议

1. **迁移完成清理**: 完成 `onKeyDown` 迁移后移除 `useInput` 桥接代码
2. **Kill Ring 隔离**: 考虑为不同类型的输入提供独立的 kill ring
3. **搜索历史**: 添加搜索历史功能（类似 shell 的 `Ctrl+R` 历史）
4. **正则支持**: 支持正则表达式搜索模式切换
5. **实时搜索**: 支持输入时实时搜索（debounced）
6. **搜索高亮**: 在结果中高亮匹配文本

### 测试关注点

1. 各种键盘快捷键的正确性
2. Kill-Yank 累积和循环行为
3. 多字节字符（CJK、Emoji）处理
4. 终端尺寸变化时的光标定位
5. 边界条件（空输入、超长输入）
6. 与父组件的键盘事件协调
