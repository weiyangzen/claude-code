# src/vim/operators.ts 研究文档

## 场景与职责

本文件是 Claude Code 的 Vim 模式实现中的**操作符（Operator）执行模块**。它负责执行 Vim 的各种操作命令，包括删除（delete）、修改（change）、复制（yank）等，以及相关的辅助操作（粘贴、替换、缩进等）。

**核心职责：**
- 执行操作符+运动组合（如 `dw`, `ciw`, `y$` 等）
- 执行行级操作（`dd`, `cc`, `yy`）
- 执行字符级操作（`x`, `r`, `~`）
- 执行粘贴操作（`p`, `P`）
- 执行缩进操作（`>>`, `<<`）
- 执行行操作（`J`, `o`, `O`）
- 管理寄存器（register）状态

**在 Vim 子系统中的位置：**
```
useVimInput.ts (Hook)
  → transitions.ts (状态机)
    → operators.ts (操作执行) ← 本文档
      → motions.ts (运动解析)
      → textObjects.ts (文本对象)
        → Cursor.ts (光标操作)
```

## 功能点目的

### 1. 操作符上下文 `OperatorContext`

**目的：** 定义操作符执行所需的上下文接口

**包含的能力：**
- `cursor`: 当前光标位置
- `text`/`setText`: 获取/设置文本内容
- `setOffset`: 设置光标偏移量
- `enterInsert`: 进入插入模式
- `getRegister`/`setRegister`: 寄存器读写
- `getLastFind`/`setLastFind`: 上次查找记录（用于 `;` 和 `,` 重复）
- `recordChange`: 记录变更（用于 `.` 重复）

### 2. 操作符+运动执行

#### `executeOperatorMotion`
执行操作符+简单运动组合（如 `dw`, `y$`）

**特殊处理：**
- `cw`/`cW` 的特殊逻辑：修改到单词结尾，而非下一个单词开头
- 行级运动的边界处理（删除到文件末尾时包含前导换行符）
- 包含性运动的范围扩展（`e`, `E`, `$` 包含目标字符）
- Image 引用保护：自动扩展范围以覆盖完整的 `[Image #N]` 芯片

#### `executeOperatorFind`
执行操作符+查找运动（如 `dfx`, `ct;`）

**流程：**
1. 使用 `Cursor.findCharacter` 查找目标字符位置
2. 计算操作范围（包含性）
3. 应用操作符
4. 记录查找状态用于后续 `;` 和 `,` 重复

#### `executeOperatorTextObj`
执行操作符+文本对象（如 `ciw`, `da"`）

**依赖：** 调用 `textObjects.ts` 中的 `findTextObject` 查找范围

#### `executeOperatorG` / `executeOperatorGg`
执行操作符+`G` 或 `gg`（如 `dG`, `ygg`）

**特殊逻辑：**
- `count=1` 表示无计数：G 到最后一行，gg 到第一行
- 有计数时跳转到指定行

### 3. 行级操作 `executeLineOp`

**支持的命令：** `dd`, `cc`, `yy`

**实现细节：**
- 通过计算换行符数量确定当前逻辑行
- 处理文件末尾删除的特殊情况（包含前导换行符避免残留空行）
- `cc` 的特殊处理：删除后保留一个空行并进入插入模式

### 4. 字符级操作

#### `executeX` - 删除字符（`x` 命令）
- 按 grapheme（字素簇）计数，正确处理 Unicode
- 删除内容存入寄存器

#### `executeReplace` - 替换字符（`r` 命令）
- 支持计数（如 `5rx` 替换5个字符为 x）
- 按 grapheme 处理 Unicode 字符

#### `executeToggleCase` - 切换大小写（`~` 命令）
- 支持计数
- 按 grapheme 处理

### 5. 行操作

#### `executeJoin` - 合并行（`J` 命令）
- 合并当前行与后续行
- 自动去除后续行的前导空白
- 在行间添加空格（如果需要）

#### `executeOpenLine` - 打开新行（`o`, `O` 命令）
- `o`: 在当前行下方打开
- `O`: 在当前行上方打开
- 自动进入插入模式

### 6. 粘贴操作 `executePaste`

**关键特性：**
- 自动检测行级粘贴（寄存器内容以换行符结尾）
- 行级粘贴：在指定行后/前插入完整行
- 字符级粘贴：在光标位置后/前插入文本
- 支持计数（重复粘贴多次）

### 7. 缩进操作 `executeIndent`

**支持的命令：** `>>`, `<<`

**缩进规则：**
- 缩进使用两个空格
- 取消缩进时优先移除两个空格
- 如果没有两个空格，尝试移除一个制表符
- 否则尽可能移除前导空白字符

## 具体技术实现

### 关键流程

#### 操作符+运动执行流程
```
executeOperatorMotion(op, motion, count, ctx)
  ├── resolveMotion(motion, ctx.cursor, count)  // 解析目标位置
  ├── getOperatorRange(cursor, target, motion, op, count)  // 计算范围
  │   ├── 特殊处理: cw/cW (change word)
  │   ├── 特殊处理: 行级运动
  │   ├── 特殊处理: 包含性运动
  │   └── Image 引用保护: snapOutOfImageRef
  └── applyOperator(op, from, to, ctx, linewise)  // 应用操作
       ├── 设置寄存器
       └── 根据操作类型执行:
            ├── yank: 设置光标位置
            ├── delete: 删除文本并调整光标
            └── change: 删除文本并进入插入模式
```

#### 行级操作流程
```
executeLineOp(op, count, ctx)
  ├── 计算当前逻辑行（通过统计换行符）
  ├── 计算影响范围（lineStart 到 lineEnd）
  ├── 设置寄存器（确保以换行符结尾）
  └── 根据操作类型:
       ├── yank: 光标移到行首
       ├── delete: 删除行并调整光标位置
       └── change: 保留空行并进入插入模式
```

### 数据结构

#### OperatorContext 接口（行26-37）
```typescript
export type OperatorContext = {
  cursor: Cursor
  text: string
  setText: (text: string) => void
  setOffset: (offset: number) => void
  enterInsert: (offset: number) => void
  getRegister: () => string
  setRegister: (content: string, linewise: boolean) => void
  getLastFind: () => { type: FindType; char: string } | null
  setLastFind: (type: FindType, char: string) => void
  recordChange: (change: RecordedChange) => void
}
```

#### 操作符范围（行429-475）
```typescript
function getOperatorRange(
  cursor: Cursor,
  target: Cursor,
  motion: string,
  op: Operator,
  count: number,
): { from: number; to: number; linewise: boolean }
```

### 关键代码路径

**文件位置：** `/home/sansha/Github/claude-code-instructkr/src/vim/operators.ts`

**核心函数位置：**

| 函数 | 行号 | 说明 |
|------|------|------|
| `executeOperatorMotion` | 42-54 | 操作符+运动执行入口 |
| `executeOperatorFind` | 59-75 | 操作符+查找执行 |
| `executeOperatorTextObj` | 80-97 | 操作符+文本对象执行 |
| `executeLineOp` | 102-166 | 行级操作（dd/cc/yy） |
| `executeX` | 171-194 | 删除字符 |
| `executeReplace` | 199-217 | 替换字符 |
| `executeToggleCase` | 222-253 | 切换大小写 |
| `executeJoin` | 258-289 | 合并行 |
| `executePaste` | 294-343 | 粘贴 |
| `executeIndent` | 348-392 | 缩进 |
| `executeOpenLine` | 397-416 | 打开新行 |
| `applyOperator` | 493-522 | 应用操作符的核心逻辑 |
| `getOperatorRange` | 429-475 | 计算操作范围 |

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `Cursor` | `../utils/Cursor.js` | 光标操作 |
| `firstGrapheme`, `lastGrapheme` | `../utils/intl.js` | Unicode grapheme 处理 |
| `countCharInString` | `../utils/stringUtils.js` | 统计字符数量 |
| `isInclusiveMotion`, `isLinewiseMotion`, `resolveMotion` | `./motions.js` | 运动解析 |
| `findTextObject` | `./textObjects.js` | 文本对象查找 |
| `FindType`, `Operator`, `RecordedChange`, `TextObjScope` | `./types.js` | 类型定义 |

### 被调用方

| 模块 | 路径 | 调用方式 |
|------|------|----------|
| `transitions.ts` | `./transitions.js` | 导入所有 execute* 函数 |
| `useVimInput.ts` | `../hooks/useVimInput.ts` | 导入部分函数用于 `.` 重复 |

### 与 Cursor 类的交互

**使用的方法：**
- `cursor.equals(other)` - 位置比较
- `cursor.measuredText.nextOffset(offset)` - 获取下一个 grapheme 偏移
- `cursor.measuredText.prevOffset(offset)` - 获取上一个 grapheme 偏移
- `cursor.snapOutOfImageRef(offset, direction)` - Image 引用保护
- `cursor.findCharacter(char, type, count)` - 字符查找
- `cursor.startOfLastLine()`, `cursor.goToLine(n)` - 行定位

## 风险、边界与改进建议

### 已知边界情况

1. **cw/cW 的特殊处理（行441-450）**
   ```typescript
   if (op === 'change' && (motion === 'w' || motion === 'W')) {
     // 特殊处理：修改到单词结尾，而非下一个单词开头
   }
   ```
   - 这是 Vim 的标准行为，但实现较复杂
   - 需要向前移动 (count-1) 个单词，然后找到该单词的结尾

2. **文件末尾删除（行134-141, 457-464）**
   - 删除到文件末尾时，如果存在前导换行符，则包含它
   - 避免删除最后一行后留下尾随换行符

3. **Image 引用保护（行471-472）**
   ```typescript
   from = cursor.snapOutOfImageRef(from, 'start')
   to = cursor.snapOutOfImageRef(to, 'end')
   ```
   - 确保 `dw`/`cw`/`yw` 不会留下部分 `[Image #N]` 占位符

4. **单文件删除的特殊处理（行152-154）**
   ```typescript
   if (lines.length === 1) {
     ctx.setText('')
     ctx.enterInsert(0)
   }
   ```
   - 单文件时 `cc` 直接清空并进入插入模式

### 潜在风险

1. **寄存器状态管理**
   - `registerIsLinewise` 标志由调用方（通过 `setRegister`）设置
   - 如果调用方忘记设置，粘贴行为可能不正确

2. **光标位置计算**
   - 多处使用 `lastGrapheme(newText).length` 计算光标位置
   - 如果文本为空，需要特殊处理（`|| 1` 保护）

3. **行号计算（行111）**
   ```typescript
   const currentLine = countCharInString(text.slice(0, ctx.cursor.offset), '\n')
   ```
   - 通过统计换行符计算逻辑行号
   - 假设文本使用 `\n` 作为换行符（在 Windows 上可能有兼容性问题）

### 改进建议

1. **统一换行符处理**
   - 考虑使用 `\r?\n` 正则处理不同平台的换行符

2. **提取公共逻辑**
   - `getLineStartOffset` 函数在多处重复计算，可以缓存
   - 行级操作的寄存器设置逻辑可以提取

3. **增强类型安全**
   - `executeOperatorG` 和 `executeOperatorGg` 的 `count` 参数语义特殊（1 表示无计数）
   - 考虑使用 `count?: number` 或专门的无计数标记

4. **添加更多边界测试**
   - 空文本的操作
   - Unicode grapheme 跨越边界的操作
   - 大计数参数的操作

5. **优化性能**
   - `executeJoin` 中多次调用 `lines[currentLine + i]`，可以预先计算
   - `executePaste` 中的重复内容生成可以优化

### 代码质量观察

1. **一致的错误处理**
   - 大部分函数在无效输入时静默返回（如 `if (target.equals(ctx.cursor)) return`）
   - 这是 Vim 的行为，但可能让调试困难

2. **文档注释**
   - 大部分函数有清晰的 JSDoc 注释
   - 复杂的边界情况有内联注释说明

3. **代码组织**
   - 内部辅助函数（`getOperatorRange`, `applyOperator` 等）放在文件底部
   - 使用 `// ============================================================================` 分隔不同区域
