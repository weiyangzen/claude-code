# src/vim/transitions.ts 研究文档

## 场景与职责

本文件是 Claude Code 的 Vim 模式实现中的**状态机转换模块**。它实现了完整的 Vim 命令状态机，处理从各种状态（idle、operator、count 等）的输入转换，是 Vim 模式的核心调度器。

**核心职责：**
- 实现 Vim 命令状态机的所有状态转换
- 处理 NORMAL 模式下的所有按键输入
- 协调操作符、运动、文本对象、查找等子系统
- 支持计数前缀（如 `5dw`）
- 支持 `.` 重复和 `;`/`，` 查找重复

**在 Vim 子系统中的位置：**
```
useVimInput.ts (Hook)
  → transitions.ts (状态机转换) ← 本文档
    → operators.ts (操作执行)
    → motions.ts (运动解析)
    → types.ts (状态类型定义)
```

## 功能点目的

### 1. 状态机架构

**支持的命令状态（`CommandState`）：**

| 状态 | 说明 | 触发条件 |
|------|------|----------|
| `idle` | 空闲状态，等待命令 | 初始状态或命令完成后 |
| `count` | 计数前缀状态 | 输入数字（1-9） |
| `operator` | 等待运动/文本对象 | 输入操作符（d/c/y） |
| `operatorCount` | 操作符后的计数 | 操作符后输入数字 |
| `operatorFind` | 操作符+查找等待字符 | 操作符后输入 f/F/t/T |
| `operatorTextObj` | 操作符+文本对象等待类型 | 操作符后输入 i/a |
| `find` | 查找等待字符 | 输入 f/F/t/T |
| `g` | g 命令等待后续 | 输入 g |
| `operatorG` | 操作符+g 等待后续 | 操作符后输入 g |
| `replace` | 替换等待字符 | 输入 r |
| `indent` | 缩进等待确认 | 输入 > 或 < |

### 2. 主转换函数 `transition`

**目的：** 根据当前状态和输入决定下一步操作

**返回值：**
```typescript
type TransitionResult = {
  next?: CommandState    // 下一个状态（如果有）
  execute?: () => void   // 要执行的函数（如果有）
}
```

**设计特点：**
- 纯函数设计（除了 `execute` 回调）
- 使用 TypeScript 的 exhaustive switch 确保所有状态都被处理
- 状态转换表是"可扫描的真理源"

### 3. 共享输入处理

#### `handleNormalInput`
处理在 `idle` 和 `count` 状态下都有效的输入

**支持的输入：**
- 操作符键（d/c/y）→ 进入 `operator` 状态
- 简单运动（h/j/k/l/w/b/e/...）→ 立即执行
- 查找键（f/F/t/T）→ 进入 `find` 状态
- 特殊键（g/r/>/<）→ 进入对应状态
- 即时执行命令（~, x, J, p, P, D, C, Y, G, ., ;, ,, u, i, I, a, A, o, O）

#### `handleOperatorInput`
处理在 `operator` 和 `operatorCount` 状态下等待的操作对象

**支持的输入：**
- 文本对象范围（i/a）→ 进入 `operatorTextObj` 状态
- 查找键（f/F/t/T）→ 进入 `operatorFind` 状态
- 简单运动 → 立即执行操作符+运动
- `G` → 执行操作符+G
- `g` → 进入 `operatorG` 状态

### 4. 状态转换函数

每个状态都有专门的转换函数：

| 函数 | 处理状态 | 关键逻辑 |
|------|----------|----------|
| `fromIdle` | `idle` | 数字 → count，0 → 行首，其他 → handleNormalInput |
| `fromCount` | `count` | 数字追加，其他 → handleNormalInput 后返回 idle |
| `fromOperator` | `operator` | 重复键 → 行操作，数字 → operatorCount，其他 → handleOperatorInput |
| `fromOperatorCount` | `operatorCount` | 数字追加，其他 → handleOperatorInput 后返回 idle |
| `fromOperatorFind` | `operatorFind` | 执行操作符+查找 |
| `fromOperatorTextObj` | `operatorTextObj` | 有效类型 → 执行，无效 → 返回 idle |
| `fromFind` | `find` | 执行查找并记录状态 |
| `fromG` | `g` | gj/gk → 显示行移动，gg → 行跳转 |
| `fromOperatorG` | `operatorG` | gj/gk/gg → 执行对应操作 |
| `fromReplace` | `replace` | 执行替换（空输入取消） |
| `fromIndent` | `indent` | 重复键 → 执行缩进 |

### 5. 特殊命令处理

#### `executeRepeatFind` - `;` 和 `,` 命令

**功能：** 重复上一次的 f/F/t/T 查找

**逻辑：**
- `;` - 同方向重复
- `,` - 反方向重复
- 使用 `flipMap` 翻转查找方向

## 具体技术实现

### 关键流程

#### 完整命令处理流程（以 `dw` 为例）
```
1. 输入 'd'
   state: idle → fromIdle('d')
   → handleNormalInput 识别为操作符
   → 返回 { next: { type: 'operator', op: 'delete', count: 1 } }

2. 输入 'w'
   state: operator → fromOperator({op:'delete',count:1}, 'w')
   → 不是重复键，不是数字
   → handleOperatorInput 识别为简单运动
   → 返回 { execute: () => executeOperatorMotion('delete', 'w', 1, ctx) }
   → 执行后返回 idle 状态
```

#### 计数命令流程（以 `5d2w` 为例）
```
1. 输入 '5' → count 状态（digits: '5'）
2. 输入 'd' → operator 状态（op: 'delete', count: 5）
3. 输入 '2' → operatorCount 状态（op: 'delete', count: 5, digits: '2'）
4. 输入 'w' 
   → 解析 motionCount = 2
   → effectiveCount = 5 * 2 = 10
   → 执行 delete + 10w
```

### 数据结构

#### 转换上下文（行43-46）
```typescript
export type TransitionContext = OperatorContext & {
  onUndo?: () => void      // u 命令回调
  onDotRepeat?: () => void // . 命令回调
}
```

#### 转换结果（行51-54）
```typescript
export type TransitionResult = {
  next?: CommandState      // 下一个状态
  execute?: () => void     // 执行函数
}
```

### 关键代码路径

**文件位置：** `/home/sansha/Github/claude-code-instructkr/src/vim/transitions.ts`

**核心函数位置：**

| 函数 | 行号 | 说明 |
|------|------|------|
| `transition` | 59-88 | 主入口，状态分发 |
| `handleNormalInput` | 98-200 | 共享的普通输入处理 |
| `handleOperatorInput` | 206-242 | 共享的操作符输入处理 |
| `fromIdle` | 248-263 | 空闲状态转换 |
| `fromCount` | 265-281 | 计数状态转换 |
| `fromOperator` | 283-308 | 操作符状态转换 |
| `fromG` | 385-418 | g 命令状态转换 |
| `executeRepeatFind` | 465-490 | ; 和 , 命令实现 |

**关键代码片段：**

**主转换分发（行64-88）：**
```typescript
export function transition(
  state: CommandState,
  input: string,
  ctx: TransitionContext,
): TransitionResult {
  switch (state.type) {
    case 'idle': return fromIdle(input, ctx)
    case 'count': return fromCount(state, input, ctx)
    case 'operator': return fromOperator(state, input, ctx)
    // ... 其他状态
  }
}
```

**计数上限保护（行271-273）：**
```typescript
if (/[0-9]/.test(input)) {
  const newDigits = state.digits + input
  const count = Math.min(parseInt(newDigits, 10), MAX_VIM_COUNT)
  return { next: { type: 'count', digits: String(count) } }
}
```

**操作符重复检测（行289-291）：**
```typescript
// dd, cc, yy = line operation
if (input === state.op[0]) {
  return { execute: () => executeLineOp(state.op, state.count, ctx) }
}
```

**查找方向翻转（行477-483）：**
```typescript
const flipMap: Record<FindType, FindType> = {
  f: 'F',
  F: 'f',
  t: 'T',
  T: 't',
}
findType = flipMap[findType]
```

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `resolveMotion` | `./motions.js` | 运动解析 |
| `executeIndent`, `executeJoin`, `executeLineOp`, ... | `./operators.js` | 操作执行 |
| `CommandState`, `FindType`, `Operator`, `SIMPLE_MOTIONS`, ... | `./types.js` | 类型和常量 |

### 被调用方

| 模块 | 路径 | 调用方式 |
|------|------|----------|
| `useVimInput.ts` | `../hooks/useVimInput.ts` | `import { transition, type TransitionContext }` |

### 与 useVimInput.ts 的交互

**调用流程：**
```typescript
// useVimInput.ts
const result = transition(state.command, vimInput, ctx)

if (result.execute) {
  result.execute()
}

if (vimStateRef.current.mode === 'NORMAL') {
  if (result.next) {
    vimStateRef.current = { mode: 'NORMAL', command: result.next }
  } else if (result.execute) {
    vimStateRef.current = { mode: 'NORMAL', command: { type: 'idle' } }
  }
}
```

## 风险、边界与改进建议

### 已知边界情况

1. **计数上限（行272, 322）**
   ```typescript
   const count = Math.min(parseInt(newDigits, 10), MAX_VIM_COUNT)
   ```
   - `MAX_VIM_COUNT = 10000`
   - 超过上限的计数会被截断

2. **0 的特殊处理（行249-257）**
   ```typescript
   if (/[1-9]/.test(input)) {
     return { next: { type: 'count', digits: input } }
   }
   if (input === '0') {
     return { execute: () => ctx.setOffset(ctx.cursor.startOfLogicalLine().offset) }
   }
   ```
   - `0` 不是计数前缀，而是行首运动
   - 这与 Vim 的行为一致

3. **Backspace/Delete 的特殊映射（行257-271 in useVimInput.ts）**
   ```typescript
   const expectsMotion =
     state.command.type === 'idle' ||
     state.command.type === 'count' ||
     state.command.type === 'operator' ||
     state.command.type === 'operatorCount'
   
   if (expectsMotion && key.backspace) vimInput = 'h'
   else if (expectsMotion && state.command.type !== 'count' && key.delete) vimInput = 'x'
   ```
   - Backspace 映射为 `h`（左移）
   - Delete 映射为 `x`（删除字符）
   - 但在 `count` 状态下 Delete 不映射（避免 `5x` 误触发）

4. **r+Backspace 取消（行446）**
   ```typescript
   if (input === '') return { next: { type: 'idle' } }
   ```
   - 替换状态下输入空字符串（Backspace/Delete）取消替换

### 潜在风险

1. **状态转换的完整性**
   - TypeScript 的 exhaustive switch 在编译时检查
   - 但如果运行时出现未知状态类型，会静默返回 `undefined`
   - 建议添加 `default` 分支抛出错误

2. **输入过滤的时机（行180-181 in useVimInput.ts）**
   ```typescript
   const filtered = inputFilter ? inputFilter(rawInput, key) : rawInput
   const input = state.mode === 'INSERT' ? filtered : rawInput
   ```
   - NORMAL 模式下使用原始输入，忽略 filter
   - 这是有意的设计（注释说明命令查找期望单字符）
   - 但可能导致 filter 状态不一致

3. **箭头键映射（行265-268 in useVimInput.ts）**
   ```typescript
   if (key.leftArrow) vimInput = 'h'
   else if (key.rightArrow) vimInput = 'l'
   else if (key.upArrow) vimInput = 'k'
   else if (key.downArrow) vimInput = 'j'
   ```
   - 箭头键映射为 hjkl
   - 但在非 idle 状态下可能不符合用户预期

### 改进建议

1. **添加默认状态处理**
   ```typescript
   default: 
     const _exhaustiveCheck: never = state
     throw new Error(`Unknown state type: ${_exhaustiveCheck}`)
   ```

2. **提取常量**
   - 箭头键映射、Backspace/Delete 映射等可以提取为配置常量
   - 便于用户自定义

3. **添加调试模式**
   - 在开发模式下记录状态转换过程
   - 便于调试复杂的命令序列

4. **支持更多 Vim 命令**
   - `Ctrl-a`/`Ctrl-x` - 数字增减
   - `>`/`<` 配合运动（当前只支持 `>>`/`<<`）
   - `gq` - 文本格式化

5. **优化 gg 处理**
   - 当前 `gg` 逻辑分散在 `fromG` 和 `executeOperatorGg`
   - 可以考虑统一处理

### 测试建议

需要覆盖的边界情况：
- 无效的状态转换（如从 `find` 状态输入非字符）
- 大计数参数的处理
- 状态超时或取消（Esc 键）
- 快速输入序列的正确处理
- 组合键（Ctrl、Alt）的处理

### 代码质量观察

1. **类型安全**
   - 使用 TypeScript 的 discriminated union 表示状态
   - exhaustive switch 确保所有状态都被处理

2. **代码组织**
   - 每个状态有独立的转换函数
   - 共享逻辑提取为 `handleNormalInput` 和 `handleOperatorInput`

3. **注释质量**
   - 文件头有详细的状态图注释
   - 关键逻辑有内联注释

4. **与 Vim 的兼容性**
   - 大部分行为与 Vim 一致
   - 注释中标注了与 Vim 的差异（如 `gj`/`gk` 的处理）
