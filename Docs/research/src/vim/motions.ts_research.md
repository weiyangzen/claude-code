# src/vim/motions.ts 研究文档

## 场景与职责

本文件是 Claude Code 的 Vim 模式实现中的**运动（Motion）解析模块**。它负责将 Vim 运动命令（如 `h`, `j`, `w`, `b` 等）解析为具体的光标位置变化。

**核心职责：**
- 提供纯函数式的运动解析，不修改任何状态
- 支持重复计数（count）的运动执行
- 区分包含性（inclusive）和排他性（exclusive）运动
- 区分字符级（characterwise）和行级（linewise）运动

**在 Vim 子系统中的位置：**
```
useVimInput.ts (Hook) 
  → transitions.ts (状态机)
    → operators.ts (操作执行)
      → motions.ts (运动解析) ← 本文档
        → Cursor.ts (光标操作)
```

## 功能点目的

### 1. 运动解析主函数 `resolveMotion`

**目的：** 将 Vim 运动命令解析为目标光标位置

**设计决策：**
- 纯函数设计：输入 `key`, `cursor`, `count`，输出新的 `Cursor` 对象
- 循环执行支持计数前缀（如 `5w` 表示向前移动5个单词）
- 边界保护：如果运动无效果（`next.equals(result)`），提前终止循环

### 2. 单步运动应用 `applySingleMotion`

**目的：** 执行单个运动命令

**支持的运动类型：**

| 命令 | 功能 | 对应 Cursor 方法 |
|------|------|------------------|
| `h` | 左移一个字符 | `cursor.left()` |
| `l` | 右移一个字符 | `cursor.right()` |
| `j` | 下移一个逻辑行 | `cursor.downLogicalLine()` |
| `k` | 上移一个逻辑行 | `cursor.upLogicalLine()` |
| `gj` | 下移一个显示行 | `cursor.down()` |
| `gk` | 上移一个显示行 | `cursor.up()` |
| `w` | 下一个单词开头 | `cursor.nextVimWord()` |
| `b` | 上一个单词开头 | `cursor.prevVimWord()` |
| `e` | 单词结尾 | `cursor.endOfVimWord()` |
| `W` | 下一个 WORD 开头 | `cursor.nextWORD()` |
| `B` | 上一个 WORD 开头 | `cursor.prevWORD()` |
| `E` | WORD 结尾 | `cursor.endOfWORD()` |
| `0` | 行首 | `cursor.startOfLogicalLine()` |
| `^` | 行首第一个非空白字符 | `cursor.firstNonBlankInLogicalLine()` |
| `$` | 行尾 | `cursor.endOfLogicalLine()` |
| `G` | 最后一行开头 | `cursor.startOfLastLine()` |

**注意：** `gg` 运动不在此处理，因为它需要计数支持（`5gg` 表示跳转到第5行），由 `transitions.ts` 中的 `fromG` 函数专门处理。

### 3. 运动类型判断

**`isInclusiveMotion(key: string)`**
- 包含性运动在操作时会包含目标位置的字符
- 返回 `true` 的命令：`e`, `E`, `$`

**`isLinewiseMotion(key: string)`**
- 行级运动在操作时会操作整行
- 返回 `true` 的命令：`j`, `k`, `G`
- 注意：`gj`/`gk` 是字符级排他运动（per `:help gj`）

## 具体技术实现

### 关键流程

```typescript
// 运动解析流程
resolveMotion(key, cursor, count)
  └── 循环 count 次
       └── applySingleMotion(key, currentCursor)
            └── 根据 key 调用对应的 Cursor 方法
       └── 如果新位置与当前位置相同，跳出循环
  └── 返回最终 cursor
```

### 数据结构

**函数签名：**
```typescript
export function resolveMotion(
  key: string,      // 运动命令字符
  cursor: Cursor,   // 当前光标对象
  count: number,    // 重复计数
): Cursor          // 返回新的光标对象
```

### 关键代码路径

**文件位置：** `/home/sansha/Github/claude-code-instructkr/src/vim/motions.ts`

**核心函数（行13-25）：**
```typescript
export function resolveMotion(
  key: string,
  cursor: Cursor,
  count: number,
): Cursor {
  let result = cursor
  for (let i = 0; i < count; i++) {
    const next = applySingleMotion(key, result)
    if (next.equals(result)) break  // 边界保护
    result = next
  }
  return result
}
```

**单步运动映射（行30-67）：**
```typescript
function applySingleMotion(key: string, cursor: Cursor): Cursor {
  switch (key) {
    case 'h': return cursor.left()
    case 'l': return cursor.right()
    // ... 其他运动
    default: return cursor  // 未知命令保持原位
  }
}
```

## 依赖与外部交互

### 导入依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| `Cursor` | `../utils/Cursor.js` | 光标操作的核心类 |

### 被调用方

| 模块 | 路径 | 调用方式 |
|------|------|----------|
| `operators.ts` | `./operators.js` | `import { resolveMotion, isInclusiveMotion, isLinewiseMotion }` |
| `transitions.ts` | `./transitions.js` | `import { resolveMotion }` |

### Cursor 类依赖的方法

`motions.ts` 依赖 `Cursor` 类提供的以下方法：

**基本移动：**
- `left()`, `right()` - 字符级左右移动
- `up()`, `down()` - 显示行上下移动
- `upLogicalLine()`, `downLogicalLine()` - 逻辑行上下移动

**单词移动（Vim 语义）：**
- `nextVimWord()`, `prevVimWord()`, `endOfVimWord()` - 小写 word 移动
- `nextWORD()`, `prevWORD()`, `endOfWORD()` - 大写 WORD 移动

**行定位：**
- `startOfLogicalLine()`, `endOfLogicalLine()` - 逻辑行首尾
- `firstNonBlankInLogicalLine()` - 行首非空白
- `startOfLastLine()` - 最后一行开头

**工具方法：**
- `equals(other: Cursor)` - 位置比较

## 风险、边界与改进建议

### 已知边界情况

1. **运动无效果时的提前终止**
   - 当运动无法继续（如已在行首按 `h`），循环会提前终止
   - 这符合 Vim 的行为，但可能让用户困惑为什么 `100h` 没有移动100次

2. **`gg` 运动不在此处理**
   - `gg` 需要特殊处理因为它可以接受计数参数（`5gg`）
   - 由 `transitions.ts` 中的 `fromG` 函数处理
   - 但 `isLinewiseMotion` 中却包含了 `'gg'`，存在不一致

3. **未知命令处理**
   - `default: return cursor` 对未知命令保持原位
   - 这是安全的，但可能静默忽略用户输入错误

### 潜在风险

1. **无限循环风险**
   - 当前实现依赖 `next.equals(result)` 来检测无效果运动
   - 如果 Cursor 方法有 bug 导致循环返回相同位置，可能造成无限循环
   - 建议：添加最大迭代次数保护

2. **与 Cursor 类的紧耦合**
   - 直接依赖 `Cursor` 类的具体方法
   - 如果 `Cursor` 类接口变更，需要同步修改

### 改进建议

1. **添加运动验证**
   ```typescript
   // 建议添加
   const VALID_MOTIONS = new Set(['h', 'l', 'j', 'k', ...])
   export function isValidMotion(key: string): boolean {
     return VALID_MOTIONS.has(key)
   }
   ```

2. **统一 `gg` 处理**
   - 考虑将 `gg` 的基本运动逻辑移到 `motions.ts`，让 `transitions.ts` 只处理计数解析

3. **添加调试支持**
   - 在开发模式下记录运动解析过程，便于调试复杂运动链

4. **文档化 Vim 语义差异**
   - 明确标注哪些行为与原生 Vim 有差异（如 `gj`/`gk` 的处理）

### 测试建议

需要覆盖的边界情况：
- 计数为 0 或负数（虽然正常流程不应出现）
- 光标已在边界时的运动（行首 `h`，行尾 `l` 等）
- 大计数参数（如 `99999w`）
- 包含性/排他性运动的正确分类
