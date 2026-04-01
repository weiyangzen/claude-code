# tokenBudget.ts 研究文档

## 场景与职责

`tokenBudget.ts` 是 Claude Code CLI 的 Token 预算解析模块，负责从用户输入中解析和识别 Token 预算指令。这是实现 "预算感知" 查询功能的基础，允许用户通过自然语言或简写语法指定本次查询的 Token 预算。

主要使用场景：
1. **预算指令解析**：从用户输入中提取如 "+500k"、"use 2M tokens" 等预算指令
2. **高亮定位**：确定预算指令在文本中的位置，用于 UI 高亮显示
3. **进度反馈**：生成预算使用进度的反馈消息

## 功能点目的

### 1. 多格式预算指令支持
支持三种输入格式：
- **简写前缀**：`+500k`（消息开头）
- **简写后缀**：`... +500k.`（消息结尾）
- **详细语法**：`use 2M tokens` 或 `spend 1.5b tokens`

### 2. 单位自动转换
- `k` = 1,000
- `m` = 1,000,000
- `b` = 1,000,000,000

### 3. 位置识别
精确定位预算指令在文本中的位置，支持：
- 输入框中高亮显示
- 验证是否重复计数

## 具体技术实现

### 正则表达式设计

```typescript
// 简写前缀：^\s*+(\d+(?:\.\d+)?)\s*(k|m|b)\b
const SHORTHAND_START_RE = /^\s*\+(\d+(?:\.\d+)?)\s*(k|m|b)\b/i

// 简写后缀：\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*[.!?]?\s*$
const SHORTHAND_END_RE = /\s\+(\d+(?:\.\d+)?)\s*(k|m|b)\s*[.!?]?\s*$/i

// 详细语法：\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b
const VERBOSE_RE = /\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b/i
const VERBOSE_RE_G = new RegExp(VERBOSE_RE.source, 'gi')
```

**关键设计决策**：

1. **避免 Lookbehind**
   ```typescript
   // 不使用：/(?<=\s)\+(\d+)/  （负向后行断言）
   // 原因： defeat YARR JIT in JSC，解释器 O(n) 扫描
   
   // 使用：/\s+(\d+)/ 并手动调整索引
   // 调用方需要 offset match.index by 1
   ```

2. **大小写不敏感**
   - 所有正则使用 `i` 标志
   - 单位后缀 `k/m/b` 大小写均可

### 核心函数

#### `parseTokenBudget`

```typescript
export function parseTokenBudget(text: string): number | null
```

**解析优先级**：
1. 简写前缀（`+500k at start`）
2. 简写后缀（`message +500k.`）
3. 详细语法（`use 500k tokens`）

**返回值**：
- 成功：预算数值（如 `500000`）
- 失败：`null`

#### `findTokenBudgetPositions`

```typescript
export function findTokenBudgetPositions(
  text: string,
): Array<{ start: number; end: number }>
```

**特殊处理 - 避免重复计数**：
```typescript
// 当输入只有 "+500k" 时，前缀和后缀正则都会匹配
// 需要检测并去重
const alreadyCovered = positions.some(
  p => endStart >= p.start && endStart < p.end,
)
```

**索引调整**：
```typescript
// 后缀正则包含前导 \s，需要 +1 排除
const endStart = endMatch.index! + 1
```

#### `getBudgetContinuationMessage`

```typescript
export function getBudgetContinuationMessage(
  pct: number,        // 已使用百分比
  turnTokens: number,  // 当前回合 Token 数
  budget: number,      // 预算上限
): string
```

**输出示例**：
```
"Stopped at 75% of token target (750,000 / 1,000,000). Keep working — do not summarize."
```

**设计意图**：
- 明确告知用户预算状态
- 指示模型继续工作而非总结（避免浪费已用 Token）

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/screens/REPL.tsx` | REPL 主界面解析用户输入 |
| `src/query/tokenBudget.ts` | 查询预算处理 |
| `src/query.ts` | 查询执行 |
| `src/components/PromptInput/PromptInput.tsx` | 输入框预算高亮 |

### 依赖模块

本模块无外部依赖，纯正则解析逻辑。

## 依赖与外部交互

### 与查询系统的集成

预算解析结果用于：
1. 设置查询的 Token 上限
2. 触发预算检查逻辑
3. 生成进度反馈消息

```typescript
// query.ts 伪代码
const budget = parseTokenBudget(userInput)
if (budget) {
  queryConfig.maxTokens = budget
}
```

### 与 UI 的集成

位置信息用于输入框高亮：
```typescript
// PromptInput.tsx 伪代码
const positions = findTokenBudgetPositions(inputText)
positions.forEach(pos => {
  highlightRange(pos.start, pos.end, 'budget-color')
})
```

## 风险、边界与改进建议

### 潜在风险

1. **正则性能**
   - 虽然避免了 lookbehind，但多个正则顺序执行仍有开销
   - 超长输入（如粘贴大段文本）可能影响性能

2. **解析歧义**
   - `+500k` 可能被视为数学表达式而非预算指令
   - 当前设计假设：以 `+` 开头且后跟数字+单位的都是预算指令

3. **单位大小写**
   - 当前 `k/m/b` 大小写不敏感
   - 但 `B` 可能被误解为 Bytes 而非 Billion

4. **浮点精度**
   - `parseFloat` + 大数乘法可能产生精度问题
   - 示例：`1.5 * 1000000000 = 1500000000.0000002`

### 边界条件

| 场景 | 行为 |
|-----|------|
| `+0k` | 解析为 0（可能无意义但技术上有效）|
| `+1.5.5k` | `parseFloat` 解析为 1.5，剩余 `.5k` 被忽略 |
| `+999999999999999k` | 大数可能超出安全整数范围 |
| 多个详细语法 | `VERBOSE_RE_G` 会找到所有匹配 |
| 简写前缀 + 详细语法 | 两者都会被找到（可能重复计数）|
| 纯空格输入 | 所有正则都不匹配，返回 null/空数组 |

### 改进建议

1. **更严格的数字验证**
   ```typescript
   function parseBudgetMatch(value: string, suffix: string): number | null {
     const num = parseFloat(value)
     if (!Number.isFinite(num) || num <= 0) {
       return null
     }
     const result = num * MULTIPLIERS[suffix.toLowerCase()]!
     if (result > Number.MAX_SAFE_INTEGER) {
       return null
     }
     return Math.round(result)
   }
   ```

2. **支持更多单位**
   ```typescript
   const MULTIPLIERS: Record<string, number> = {
     k: 1_000,
     m: 1_000_000,
     b: 1_000_000_000,
     t: 1_000_000_000_000,  // trillion
   }
   ```

3. **上下文感知解析**
   ```typescript
   // 避免在代码块中解析
   export function parseTokenBudget(text: string, options?: {
     ignoreCodeBlocks?: boolean
   }): number | null
   ```

4. **用户确认提示**
   ```typescript
   // 对于超大预算，要求用户确认
   if (budget > 10_000_000) {
     showConfirmation(`Are you sure you want to use ${formatNumber(budget)} tokens?`)
   }
   ```

5. **预算建议**
   ```typescript
   // 根据上下文大小建议预算
   export function suggestTokenBudget(contextTokens: number): number {
     return Math.min(2_000_000, contextTokens * 2)
   }
   ```

6. **格式化输出**
   ```typescript
   // 将数字格式化为简写形式
   export function formatTokenBudget(tokens: number): string {
     if (tokens >= 1_000_000_000) return `${(tokens / 1e9).toFixed(1)}B`
     if (tokens >= 1_000_000) return `${(tokens / 1e6).toFixed(1)}M`
     if (tokens >= 1_000) return `${(tokens / 1e3).toFixed(1)}K`
     return String(tokens)
   }
   ```

### 测试建议

应覆盖以下场景：
- 各种有效格式（`+500k`, `+1.5m`, `use 2B tokens` 等）
- 大小写混合（`+500K`, `USE 2M TOKENS`）
- 边界位置（开头、结尾、中间）
- 无效输入（`+abc`, `use tokens`, `+`）
- 超大数字（`+999999999999k`）
- 多个预算指令的组合
- 浮点精度（`+1.1k` → 1100）
