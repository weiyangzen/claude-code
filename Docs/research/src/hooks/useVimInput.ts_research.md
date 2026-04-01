# useVimInput.ts 研究文档

## 场景与职责

`useVimInput` 是一个 React Hook，为 Claude Code 的文本输入组件提供完整的 Vim 模式编辑支持。它将基础的文本输入功能（来自 `useTextInput`）扩展为支持 Vim 的 NORMAL/INSERT 双模式编辑系统。

核心职责：
1. **模式管理**：维护 Vim 的 INSERT 和 NORMAL 模式状态
2. **命令解析**：解析 Vim 命令序列（如 `dd`、`ci"`、`3w` 等）
3. **操作执行**：执行删除、修改、复制、粘贴等 Vim 操作
4. **光标控制**：精确控制光标位置（包括 Vim 特有的行为，如退出 INSERT 时左移一格）
5. **点重复**：支持 `.` 命令重复上一次的修改操作

该 Hook 被 `VimTextInput.tsx` 组件使用，后者是用户与 Claude Code 交互的主要输入界面。

## 功能点目的

### 1. 双模式系统

```typescript
type VimMode = 'INSERT' | 'NORMAL'

const [mode, setMode] = useState<VimMode>('INSERT')
```

- **INSERT 模式**：正常输入文本，与常规编辑器行为一致
- **NORMAL 模式**：命令模式，支持 Vim 的各种导航和编辑命令
- 默认进入 INSERT 模式（与常规用户体验一致）

### 2. 状态机架构

使用 `vimStateRef` 维护命令状态机：

```typescript
type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

`CommandState` 支持多种状态：
- `idle`：等待命令输入
- `count`：正在输入数字前缀（如 `3d` 中的 `3`）
- `operator`：已输入操作符（如 `d`、`c`、`y`），等待动作
- `operatorCount`：操作符后有数字前缀
- `operatorFind`：操作符 + 查找（如 `dfx`）
- `operatorTextObj`：操作符 + 文本对象（如 `ci"`）
- `find`：查找命令（`f`、`F`、`t`、`T`）
- `g`：`g` 前缀命令（如 `gg`、`gj`）
- `replace`：替换命令（`r`）
- `indent`：缩进命令（`>>`、`<<`）

### 3. 持久化状态

```typescript
const persistentRef = React.useRef<PersistentState>(
  createInitialPersistentState(),
)
```

持久化状态包括：
- `register`：复制/删除的文本寄存器
- `registerIsLinewise`：寄存器内容是否为整行模式
- `lastFind`：上一次的查找操作（用于 `;` 和 `,` 重复）
- `lastChange`：上一次的修改操作（用于 `.` 重复）

### 4. 点重复系统

```typescript
function replayLastChange(): void {
  const change = persistentRef.current.lastChange
  if (!change) return
  // 根据 change.type 执行对应的操作
}
```

支持的重复操作类型：
- `insert`：插入的文本
- `x`：删除字符
- `replace`：替换字符
- `toggleCase`：切换大小写
- `indent`：缩进/反缩进
- `join`：合并行
- `openLine`：开新行
- `operator`：操作符 + 动作
- `operatorFind`：操作符 + 查找
- `operatorTextObj`：操作符 + 文本对象

## 具体技术实现

### 关键流程

#### 1. 模式切换

**INSERT → NORMAL** (`switchToNormalMode`)：
```typescript
const switchToNormalMode = useCallback((): void => {
  // 1. 记录插入的文本用于点重复
  if (current.mode === 'INSERT' && current.insertedText) {
    persistentRef.current.lastChange = {
      type: 'insert',
      text: current.insertedText,
    }
  }

  // 2. Vim 行为：退出 INSERT 时左移一格（除非在行首或 offset 0）
  const offset = textInput.offset
  if (offset > 0 && props.value[offset - 1] !== '\n') {
    textInput.setOffset(offset - 1)
  }

  // 3. 切换到 NORMAL 模式
  vimStateRef.current = { mode: 'NORMAL', command: { type: 'idle' } }
  setMode('NORMAL')
  onModeChange?.('NORMAL')
}, [onModeChange, textInput, props.value])
```

**NORMAL → INSERT** (`switchToInsertMode`)：
- 可选地接受一个偏移量参数，将光标移动到指定位置
- 重置 `insertedText` 为空字符串，开始记录新的插入

#### 2. 输入处理流程

```typescript
function handleVimInput(rawInput: string, key: Key): void {
  // 1. 应用 inputFilter（如果有）
  const filtered = inputFilter ? inputFilter(rawInput, key) : rawInput
  const input = state.mode === 'INSERT' ? filtered : rawInput

  // 2. 处理特殊键（Ctrl、Escape、Enter）
  if (key.ctrl) { /* ... */ }
  if (key.escape && state.mode === 'INSERT') { switchToNormalMode(); return }
  if (key.return) { /* ... */ }

  // 3. INSERT 模式：记录插入文本并传递给基础输入处理
  if (state.mode === 'INSERT') {
    // 跟踪插入文本用于点重复
    if (key.backspace || key.delete) {
      // 处理退格/删除
    } else {
      vimStateRef.current = {
        mode: 'INSERT',
        insertedText: state.insertedText + input,
      }
    }
    textInput.onInput(input, key)
    return
  }

  // 4. NORMAL 模式：命令解析和执行
  // ... 状态机转换逻辑
}
```

#### 3. 命令状态机转换

核心转换逻辑委托给 `transition` 函数（来自 `vim/transitions.ts`）：

```typescript
const ctx: TransitionContext = {
  ...createOperatorContext(cursor, false),
  onUndo: props.onUndo,
  onDotRepeat: replayLastChange,
}

const result = transition(state.command, vimInput, ctx)

if (result.execute) {
  result.execute()
}

// 更新命令状态
if (vimStateRef.current.mode === 'NORMAL') {
  if (result.next) {
    vimStateRef.current = { mode: 'NORMAL', command: result.next }
  } else if (result.execute) {
    vimStateRef.current = { mode: 'NORMAL', command: { type: 'idle' } }
  }
}
```

### 数据结构

#### OperatorContext
```typescript
type OperatorContext = {
  cursor: Cursor                    // 当前光标位置
  text: string                      // 当前文本内容
  setText: (newText: string) => void  // 设置文本
  setOffset: (offset: number) => void // 设置光标偏移
  enterInsert: (offset: number) => void // 进入 INSERT 模式
  getRegister: () => string         // 获取寄存器内容
  setRegister: (content: string, linewise: boolean) => void // 设置寄存器
  getLastFind: () => { type: FindType; char: string } | null // 获取上次查找
  setLastFind: (type: FindType, char: string) => void // 设置上次查找
  recordChange: (change: RecordedChange) => void // 记录修改
}
```

#### RecordedChange（点重复记录）
```typescript
type RecordedChange =
  | { type: 'insert'; text: string }
  | { type: 'x'; count: number }
  | { type: 'replace'; char: string; count: number }
  | { type: 'toggleCase'; count: number }
  | { type: 'indent'; dir: '>' | '<'; count: number }
  | { type: 'join'; count: number }
  | { type: 'openLine'; direction: 'above' | 'below' }
  | { type: 'operator'; op: Operator; motion: string; count: number }
  | { type: 'operatorFind'; op: Operator; find: FindType; char: string; count: number }
  | { type: 'operatorTextObj'; op: Operator; scope: TextObjScope; objType: string; count: number }
```

### 特殊键映射

```typescript
// 方向键映射为 Vim 动作
if (key.leftArrow) vimInput = 'h'
else if (key.rightArrow) vimInput = 'l'
else if (key.upArrow) vimInput = 'k'
else if (key.downArrow) vimInput = 'j'
else if (expectsMotion && key.backspace) vimInput = 'h'
else if (expectsMotion && state.command.type !== 'count' && key.delete) vimInput = 'x'
```

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useVimInput.ts` - Hook 实现

### 依赖文件
| 文件 | 用途 |
|------|------|
| `../ink.js` | Ink 渲染库类型定义 |
| `../types/textInputTypes.js` | VimInputState, VimMode 类型 |
| `../utils/Cursor.js` | 光标操作工具类 |
| `../utils/intl.js` | 国际化工具（lastGrapheme） |
| `../vim/operators.js` | Vim 操作执行函数 |
| `../vim/transitions.js` | 状态机转换逻辑 |
| `../vim/types.js` | Vim 状态类型定义 |
| `./useTextInput.js` | 基础文本输入 Hook |

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/components/VimTextInput.tsx` - Vim 文本输入组件

## 依赖与外部交互

### Vim 子系统依赖
```
vim/
├── operators.ts    # 操作执行（delete, yank, change, paste 等）
├── transitions.ts  # 状态机转换表
├── motions.ts      # 动作解析
├── textObjects.ts  # 文本对象解析
└── types.ts        # 类型定义
```

### 核心外部函数

| 函数 | 来源 | 用途 |
|------|------|------|
| `useTextInput` | `./useTextInput.js` | 基础文本输入功能 |
| `Cursor.fromText` | `../utils/Cursor.js` | 创建光标实例 |
| `transition` | `../vim/transitions.js` | 状态机转换 |
| `executeX` 等 | `../vim/operators.js` | 执行 Vim 操作 |
| `lastGrapheme` | `../utils/intl.js` | 获取最后一个字素 |

## 风险、边界与改进建议

### 潜在风险

1. **状态同步问题**：
   - `vimStateRef` 和 `mode` state 需要保持同步
   - 如果直接修改 ref 而不调用 `setMode`，会导致 UI 和内部状态不一致

2. **光标位置边界**：
   - 退出 INSERT 时的左移逻辑需要检查边界（`offset > 0` 和不是换行符）
   - 如果文本在渲染期间变化，光标位置可能失效

3. **输入过滤器与 Vim 命令冲突**：
   - `inputFilter` 只在 INSERT 模式应用过滤后的输入
   - NORMAL 模式使用原始输入，因为命令查找期望单字符

4. **点重复限制**：
   - 当前实现只记录特定类型的修改
   - 复杂的宏或组合命令可能无法正确重复

### 边界情况

| 场景 | 处理 |
|------|------|
| 在文件开头按 Escape | 不左移光标（`offset > 0` 检查）|
| 在换行符前按 Escape | 不左移光标（`value[offset - 1] !== '\n'` 检查）|
| 退格删除插入的文本 | 从 `insertedText` 中移除最后一个字素 |
| 快速连续按键 | 通过 `inputFilter` 和状态机正确处理 |
| 空文本 | `Cursor.fromText` 处理空文本情况 |

### 改进建议

1. **添加视觉模式**：
   - 当前只支持 NORMAL 和 INSERT
   - 添加 VISUAL 模式以支持选区操作

2. **宏录制支持**：
   - 扩展 `lastChange` 系统支持多命令宏
   - 添加 `q` 寄存器录制功能

3. **更完善的 Undo/Redo**：
   - 当前依赖 `props.onUndo` 回调
   - 可考虑集成更完善的编辑历史管理

4. **性能优化**：
   - 大型文本文件中的光标操作可能有性能问题
   - 可考虑虚拟化或增量更新

5. **配置扩展**：
   - 支持用户自定义键映射
   - 支持 `vimrc` 风格的配置

6. **测试覆盖**：
   - 添加单元测试覆盖各种 Vim 命令组合
   - 测试边界情况（空文本、单字符、多字节字符）
