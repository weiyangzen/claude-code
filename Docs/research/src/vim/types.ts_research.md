# src/vim/types.ts 研究文档

## 场景与职责

本文件是 Claude Code 的 Vim 模式实现中的**类型定义模块**。它定义了完整的 Vim 状态机类型系统，是理解整个 Vim 子系统的基础。文件头部的注释明确指出："The types ARE the documentation - reading them tells you how the system works."

**核心职责：**
- 定义 Vim 模式的核心类型（Operator、FindType、TextObjScope）
- 定义完整的状态机类型（VimState、CommandState）
- 定义持久化状态类型（PersistentState、RecordedChange）
- 定义键分组常量（OPERATORS、SIMPLE_MOTIONS、FIND_KEYS 等）
- 提供状态工厂函数（createInitialVimState、createInitialPersistentState）

**在 Vim 子系统中的位置：**
```
types.ts (类型定义) ← 本文档
  ↑
  被所有其他 Vim 模块导入
```

## 功能点目的

### 1. 核心类型定义

#### `Operator` - 操作符类型
```typescript
export type Operator = 'delete' | 'change' | 'yank'
```
- 定义支持的操作符：删除、修改、复制
- 这是 Vim 的 d/c/y 命令的抽象

#### `FindType` - 查找类型
```typescript
export type FindType = 'f' | 'F' | 't' | 'T'
```
- `f` - 向前查找字符（到字符）
- `F` - 向后查找字符（到字符）
- `t` - 向前查找字符（到字符前）
- `T` - 向后查找字符（到字符后）

#### `TextObjScope` - 文本对象范围
```typescript
export type TextObjScope = 'inner' | 'around'
```
- `inner` - 内部（如 `i"` 不包含引号）
- `around` - 周围（如 `a"` 包含引号）

### 2. 状态机类型

#### `VimState` - Vim 完整状态
```typescript
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

**设计特点：**
- 使用 discriminated union 区分模式
- `INSERT` 模式跟踪插入的文本（用于 `.` 重复）
- `NORMAL` 模式包含命令状态机

#### `CommandState` - 命令状态机
```typescript
export type CommandState =
  | { type: 'idle' }
  | { type: 'count'; digits: string }
  | { type: 'operator'; op: Operator; count: number }
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }
  | { type: 'find'; find: FindType; count: number }
  | { type: 'g'; count: number }
  | { type: 'operatorG'; op: Operator; count: number }
  | { type: 'replace'; count: number }
  | { type: 'indent'; dir: '>' | '<'; count: number }
```

**状态说明：**

| 状态 | 字段 | 说明 |
|------|------|------|
| `idle` | - | 等待命令输入 |
| `count` | `digits: string` | 已输入的计数数字 |
| `operator` | `op: Operator`, `count: number` | 等待操作对象 |
| `operatorCount` | 同上 + `digits: string` | 操作符后的计数 |
| `operatorFind` | 同上 + `find: FindType` | 操作符+查找等待字符 |
| `operatorTextObj` | 同上 + `scope: TextObjScope` | 操作符+文本对象等待类型 |
| `find` | `find: FindType`, `count: number` | 查找等待字符 |
| `g` | `count: number` | g 命令等待后续 |
| `operatorG` | `op: Operator`, `count: number` | 操作符+g 等待后续 |
| `replace` | `count: number` | 替换等待字符 |
| `indent` | `dir: '>' \| '<'`, `count: number` | 缩进等待确认 |

### 3. 持久化状态

#### `PersistentState` - 跨命令持久状态
```typescript
export type PersistentState = {
  lastChange: RecordedChange | null    // 上次变更（用于 . 重复）
  lastFind: { type: FindType; char: string } | null  // 上次查找（用于 ; 和 ,）
  register: string                      // 寄存器内容
  registerIsLinewise: boolean          // 寄存器是否为行级
}
```

**设计特点：**
- 独立于 `VimState`，跨命令持久保存
- 实现 Vim 的寄存器、重复、查找记忆功能

#### `RecordedChange` - 记录的变更
```typescript
export type RecordedChange =
  | { type: 'insert'; text: string }
  | { type: 'operator'; op: Operator; motion: string; count: number }
  | { type: 'operatorTextObj'; op: Operator; objType: string; scope: TextObjScope; count: number }
  | { type: 'operatorFind'; op: Operator; find: FindType; char: string; count: number }
  | { type: 'replace'; char: string; count: number }
  | { type: 'x'; count: number }
  | { type: 'toggleCase'; count: number }
  | { type: 'indent'; dir: '>' | '<'; count: number }
  | { type: 'openLine'; direction: 'above' | 'below' }
  | { type: 'join'; count: number }
```

**设计特点：**
- 使用 discriminated union 区分不同类型的变更
- 包含重放所需的全部信息
- 支持 `.` 命令重复几乎所有操作

### 4. 键分组常量

#### `OPERATORS` - 操作符映射
```typescript
export const OPERATORS = {
  d: 'delete',
  c: 'change',
  y: 'yank',
} as const satisfies Record<string, Operator>
```
- 使用 `as const` 确保类型安全
- 使用 `satisfies` 验证类型兼容性
- 提供 `isOperatorKey` 类型守卫函数

#### `SIMPLE_MOTIONS` - 简单运动集合
```typescript
export const SIMPLE_MOTIONS = new Set([
  'h', 'l', 'j', 'k',           // 基本移动
  'w', 'b', 'e', 'W', 'B', 'E', // 单词移动
  '0', '^', '$',                // 行定位
])
```

#### `FIND_KEYS` - 查找键集合
```typescript
export const FIND_KEYS = new Set(['f', 'F', 't', 'T'])
```

#### `TEXT_OBJ_SCOPES` - 文本对象范围映射
```typescript
export const TEXT_OBJ_SCOPES = {
  i: 'inner',
  a: 'around',
} as const satisfies Record<string, TextObjScope>
```

#### `TEXT_OBJ_TYPES` - 文本对象类型集合
```typescript
export const TEXT_OBJ_TYPES = new Set([
  'w', 'W',                     // 单词
  '"', "'", '`',                // 引号
  '(', ')', 'b',                // 圆括号
  '[', ']',                     // 方括号
  '{', '}', 'B',                // 花括号
  '<', '>',                     // 尖括号
])
```

#### `MAX_VIM_COUNT` - 计数上限
```typescript
export const MAX_VIM_COUNT = 10000
```

### 5. 状态工厂函数

#### `createInitialVimState`
```typescript
export function createInitialVimState(): VimState {
  return { mode: 'INSERT', insertedText: '' }
}
```
- 默认从 INSERT 模式开始
- 这与传统 Vim 不同（Vim 默认 NORMAL），但符合 Claude Code 的交互设计

#### `createInitialPersistentState`
```typescript
export function createInitialPersistentState(): PersistentState {
  return {
    lastChange: null,
    lastFind: null,
    register: '',
    registerIsLinewise: false,
  }
}
```

## 具体技术实现

### 类型守卫函数

#### `isOperatorKey`
```typescript
export function isOperatorKey(key: string): key is keyof typeof OPERATORS {
  return key in OPERATORS
}
```
- 使用 `key is keyof typeof OPERATORS` 返回类型谓词
- 允许 TypeScript 在条件分支中收窄类型

#### `isTextObjScopeKey`
```typescript
export function isTextObjScopeKey(key: string): key is keyof typeof TEXT_OBJ_SCOPES {
  return key in TEXT_OBJ_SCOPES
}
```

### 关键代码路径

**文件位置：** `/home/sansha/Github/claude-code-instructkr/src/vim/types.ts`

**类型定义位置：**

| 类型 | 行号 | 说明 |
|------|------|------|
| `Operator` | 33 | 操作符类型 |
| `FindType` | 35 | 查找类型 |
| `TextObjScope` | 37 | 文本对象范围 |
| `VimState` | 49-52 | Vim 完整状态 |
| `CommandState` | 59-76 | 命令状态机 |
| `PersistentState` | 81-86 | 持久化状态 |
| `RecordedChange` | 92-120 | 记录的变更 |
| `OPERATORS` | 125-129 | 操作符常量 |
| `SIMPLE_MOTIONS` | 135-149 | 简单运动常量 |
| `TEXT_OBJ_TYPES` | 164-180 | 文本对象类型常量 |

## 依赖与外部交互

### 导入依赖

本文件**无外部导入**，是纯类型定义文件。

### 被调用方

| 模块 | 路径 | 导入内容 |
|------|------|----------|
| `transitions.ts` | `./transitions.js` | `CommandState`, `FindType`, `Operator`, `SIMPLE_MOTIONS`, ... |
| `operators.ts` | `./operators.js` | `FindType`, `Operator`, `RecordedChange`, `TextObjScope` |
| `textObjects.ts` | `./textObjects.js` | `TextObjScope` |
| `useVimInput.ts` | `../hooks/useVimInput.ts` | `createInitialPersistentState`, `createInitialVimState`, `PersistentState`, `RecordedChange`, `VimState` |

### 类型使用示例

**在 `transitions.ts` 中：**
```typescript
import {
  type CommandState,
  FIND_KEYS,
  type FindType,
  isOperatorKey,
  MAX_VIM_COUNT,
  OPERATORS,
  type Operator,
  SIMPLE_MOTIONS,
  TEXT_OBJ_SCOPES,
  TEXT_OBJ_TYPES,
  type TextObjScope,
} from './types.js'
```

**在 `useVimInput.ts` 中：**
```typescript
import {
  createInitialPersistentState,
  createInitialVimState,
  type PersistentState,
  type RecordedChange,
  type VimState,
} from '../vim/types.js'
```

## 风险、边界与改进建议

### 设计决策分析

1. **默认 INSERT 模式**
   ```typescript
   export function createInitialVimState(): VimState {
     return { mode: 'INSERT', insertedText: '' }
   }
   ```
   - 与传统 Vim 不同（Vim 默认 NORMAL）
   - 符合 Claude Code 的交互设计（用户可以直接输入）
   - 但可能让 Vim 用户感到困惑

2. **计数上限 10000**
   ```typescript
   export const MAX_VIM_COUNT = 10000
   ```
   - 防止过大计数导致的性能问题
   - 足够大以满足正常使用

3. **使用 `as const satisfies`**
   ```typescript
   export const OPERATORS = {
     d: 'delete',
     c: 'change',
     y: 'yank',
   } as const satisfies Record<string, Operator>
   ```
   - `as const` 使属性变为只读字面量类型
   - `satisfies` 验证类型而不加宽
   - 这是 TypeScript 4.9+ 的最佳实践

### 潜在风险

1. **状态类型的演进**
   - 添加新状态需要修改所有使用 `CommandState` 的 switch 语句
   - TypeScript 的 exhaustive check 会帮助发现遗漏
   - 但仍需仔细测试

2. **`RecordedChange` 的完整性**
   - 新操作类型需要添加到 `RecordedChange`
   - 忘记添加会导致 `.` 重复不支持该操作

3. **常量集合的同步**
   - `SIMPLE_MOTIONS` 和 `TEXT_OBJ_TYPES` 需要与实现同步
   - 添加新运动或文本对象时需要更新

### 改进建议

1. **添加更多类型守卫**
   ```typescript
   export function isSimpleMotion(key: string): key is typeof SIMPLE_MOTIONS extends Set<infer T> ? T : never {
     return SIMPLE_MOTIONS.has(key as any)
   }
   ```

2. **提取文本对象类型定义**
   ```typescript
   export const TEXT_OBJ_PAIRS = {
     parentheses: { open: '(', close: ')', aliases: ['(', ')', 'b'] },
     brackets: { open: '[', close: ']', aliases: ['[', ']'] },
     braces: { open: '{', close: '}', aliases: ['{', '}', 'B'] },
     angles: { open: '<', close: '>', aliases: ['<', '>'] },
     doubleQuote: { open: '"', close: '"', aliases: ['"'] },
     singleQuote: { open: "'", close: "'", aliases: ["'"] },
     backtick: { open: '`', close: '`', aliases: ['`'] },
   } as const
   ```

3. **添加操作符元数据**
   ```typescript
   export const OPERATOR_METADATA = {
     delete: { char: 'd', entersInsert: false },
     change: { char: 'c', entersInsert: true },
     yank: { char: 'y', entersInsert: false },
   } as const
   ```

4. **支持更多 Vim 特性**
   - 添加 `Register` 类型支持多寄存器（`"ay`, `"ap`）
   - 添加 `Mark` 类型支持标记（`ma`, `` `a ``）
   - 添加 `VisualMode` 支持可视模式

### 代码质量观察

1. **类型安全**
   - 全面使用 TypeScript 的严格类型
   - discriminated union 确保状态处理完整性
   - 类型守卫函数提供运行时类型检查

2. **文档质量**
   - 文件头部的状态图注释非常清晰
   - 类型名称自解释
   - 注释说明设计决策

3. **可维护性**
   - 常量集中定义，便于修改
   - 工厂函数封装初始化逻辑
   - 类型定义与实现分离

4. **与 Vim 的兼容性**
   - 类型设计基于 Vim 的实际行为
   - 注释中标注了与 Vim 的差异（如默认 INSERT 模式）
