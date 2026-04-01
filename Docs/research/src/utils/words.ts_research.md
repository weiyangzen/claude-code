# words.ts 研究文档

## 场景与职责

`words.ts` 是一个随机单词 slug 生成器，用于生成人类可读的标识符（如计划 ID）。设计灵感来自 `random-word-slugs` 项目，但使用了 Claude 风格的词汇列表。

**核心使用场景：**
- 生成计划模式（plan mode）的唯一标识符
- 生成工作区（worktree）名称
- 生成其他需要人类友好标识符的场景

## 功能点目的

### 1. 完整 Slug 生成 (`generateWordSlug`)
- **格式**：`adjective-verb-noun`
- **示例**：`gleaming-brewing-phoenix`, `cosmic-pondering-lighthouse`
- **目的**：生成描述性强、易于记忆的标识符

### 2. 短 Slug 生成 (`generateShortWordSlug`)
- **格式**：`adjective-noun`
- **示例**：`graceful-unicorn`, `cosmic-lighthouse`
- **目的**：在需要较短标识符时使用

### 3. 加密安全随机
- **实现**：使用 Node.js `crypto.randomBytes` 而非 `Math.random`
- **目的**：确保生成的标识符不可预测（安全敏感场景）

## 具体技术实现

### 词汇表结构

```typescript
// 形容词列表（约 230 个）
const ADJECTIVES = [
  // Classic pleasant adjectives
  'abundant', 'ancient', 'bright', 'calm', ...
  // Whimsical / magical
  'breezy', 'bubbly', 'cosmic', 'crystalline', ...
  // Programming concepts
  'abstract', 'adaptive', 'agile', 'async', ...
] as const

// 名词列表（约 420 个）
const NOUNS = [
  // Nature & cosmic
  'aurora', 'avalanche', 'blossom', 'breeze', ...
  // Cute creatures
  'alpaca', 'axolotl', 'badger', 'bear', ...
  // Fun objects & concepts
  'acorn', 'anchor', 'balloon', 'beacon', ...
  // Computer scientists
  'abelson', 'adleman', 'aho', 'allen', ...
] as const

// 动词列表（约 110 个）
const VERBS = [
  'baking', 'beaming', 'booping', 'bouncing', 
  'brewing', 'bubbling', 'chasing', ...
] as const
```

### 随机数生成

```typescript
function randomInt(max: number): number {
  // 使用 4 字节（32 位）随机数
  const bytes = randomBytes(4)
  const value = bytes.readUInt32BE(0)
  return value % max
}
```

**注意**：使用取模运算会引入轻微偏差（modulo bias），但对于非加密安全场景可接受。

### 生成流程

```
generateWordSlug()
├── pickRandom(ADJECTIVES) → 形容词
├── pickRandom(VERBS)      → 动词  
├── pickRandom(NOUNS)      → 名词
└── 返回 `${adjective}-${verb}-${noun}`

generateShortWordSlug()
├── pickRandom(ADJECTIVES) → 形容词
├── pickRandom(NOUNS)      → 名词
└── 返回 `${adjective}-${noun}`
```

## 关键代码路径与文件引用

### 导出函数
- `src/utils/words.ts:785` - `generateWordSlug()`
- `src/utils/words.ts:796` - `generateShortWordSlug()`

### 内部函数
- `src/utils/words.ts:767` - `randomInt(max)` - 加密安全随机整数
- `src/utils/words.ts:777` - `pickRandom<T>(array)` - 从数组随机选择

### 词汇表定义
- `src/utils/words.ts:9` - `ADJECTIVES` 常量数组
- `src/utils/words.ts:234` - `NOUNS` 常量数组
- `src/utils/words.ts:651` - `VERBS` 常量数组

### 依赖
| 依赖 | 用途 |
|------|------|
| `crypto` (Node.js) | `randomBytes` 函数 |

## 依赖与外部交互

### 外部依赖
```typescript
import { randomBytes } from 'crypto'
```

### 内部依赖
无内部依赖。

## 风险、边界与改进建议

### 已知风险

1. **Modulo Bias（取模偏差）**
   ```typescript
   // 当前实现
   return value % max
   ```
   当 `max` 不是 2^32 的约数时，较小的值会略微更频繁出现。
   
   **影响**：对于当前词汇表大小，偏差极小（< 0.001%），可忽略。

2. **词汇表大小限制**
   - 完整 slug 组合数：230 × 110 × 420 ≈ 1060 万
   - 短 slug 组合数：230 × 420 ≈ 9.7 万
   - 在高频生成场景下可能出现碰撞

3. **内存占用**
   - 三个大数组常驻内存
   - 文件大小约 11KB，词汇表占主要部分

### 边界情况

1. **空数组**
   - 如果传入空数组给 `pickRandom`，会返回 `undefined`（`array[0]`）
   - 当前实现中不会遇到（常量数组非空）

2. **大数组**
   - `randomInt` 使用 32 位随机数，支持最大 2^32-1 的数组长度
   - 远超当前需求

### 改进建议

1. **消除 Modulo Bias**
   ```typescript
   function randomInt(max: number): number {
     const maxValid = Math.floor(0x100000000 / max) * max
     let value
     do {
       value = randomBytes(4).readUInt32BE(0)
     } while (value >= maxValid)
     return value % max
   }
   ```

2. **可配置词汇表**
   - 允许用户传入自定义词汇表
   - 支持主题化（如技术主题、自然主题）

3. **碰撞检测**
   - 在需要全局唯一标识符的场景，添加已使用 slug 的跟踪

4. **性能优化**
   - 考虑使用预计算的随机数池，减少 `randomBytes` 调用

5. **扩展格式**
   - 支持更多格式变体（如添加数字后缀、自定义分隔符）
