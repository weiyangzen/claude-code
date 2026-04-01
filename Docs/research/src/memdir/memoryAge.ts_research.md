# memoryAge.ts 研究文档

## 场景与职责

`memoryAge.ts` 是一个**轻量级的时间计算工具模块**，专门用于处理记忆文件的新鲜度（freshness）计算和展示。它解决了模型在理解原始时间戳时的困难，将技术性的时间戳转换为人类可读的相对时间描述。

### 核心职责
1. **记忆年龄计算**：计算记忆文件自修改以来的天数
2. **人类可读格式化**：将时间差转换为 "today"/"yesterday"/"N days ago"
3. **新鲜度警告生成**：为超过1天的记忆生成陈旧性警告
4. **系统提醒包装**：将警告包装在 `<system-reminder>` 标签中

### 使用场景
- 文件读取工具（FileReadTool）展示记忆文件时附加新鲜度信息
- 消息包装器（messages.ts）在注入相关记忆时附加时间上下文
- 任何需要向模型解释记忆文件新旧程度的场景

---

## 功能点目的

### 1. `memoryAgeDays()` - 天数计算
**目的**：计算自文件修改以来经过的完整天数

**算法**：
```typescript
Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
```
- 86,400,000 = 24 * 60 * 60 * 1000（一天的毫秒数）
- 向下取整（floor）
- 负值（未来时间）钳制到0

**返回值**：
- `0` = 今天
- `1` = 昨天
- `2+` = 更久以前

### 2. `memoryAge()` - 人类可读格式化
**目的**：将天数转换为自然语言描述

**映射规则**：
| 天数 | 输出 |
|------|------|
| 0 | "today" |
| 1 | "yesterday" |
| ≥2 | "{d} days ago" |

**设计理由**：模型对原始 ISO 时间戳的日期算术能力较弱，相对时间描述更能触发陈旧性推理。

### 3. `memoryFreshnessText()` - 纯文本陈旧警告
**目的**：为超过1天的记忆生成警告文本

**行为**：
- ≤1天：返回空字符串（无警告）
- >1天：返回包含天数和陈旧性说明的警告

**警告内容**：
```
This memory is {d} days old. Memories are point-in-time observations, 
not live state — claims about code behavior or file:line citations 
may be outdated. Verify against current code before asserting as fact.
```

**使用场景**：调用方已提供自己的包装（如 `wrapMessagesInSystemReminder`）

### 4. `memoryFreshnessNote()` - 系统提醒包装
**目的**：将陈旧警告包装在 `<system-reminder>` 标签中

**行为**：
- ≤1天：返回空字符串
- >1天：返回 `<system-reminder>{警告文本}</system-reminder>\n`

**使用场景**：调用方不添加自己的包装（如 FileReadTool 输出）

---

## 具体技术实现

### 核心算法

```typescript
// 天数计算（毫秒 → 天）
const MS_PER_DAY = 86_400_000
export function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / MS_PER_DAY))
}

// 人类可读格式化
export function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}

// 纯文本警告
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}

// 系统提醒包装
export function memoryFreshnessNote(mtimeMs: number): string {
  const text = memoryFreshnessText(mtimeMs)
  if (!text) return ''
  return `<system-reminder>${text}</system-reminder>\n`
}
```

### 设计决策

1. **1天阈值**：
   - 今天和昨天的记忆被视为"新鲜"
   - 2天及以上的记忆才触发警告
   - 平衡了提醒频率和实用性

2. **空字符串返回**：
   - 新鲜记忆返回空字符串，便于调用方直接拼接
   - 避免调用方需要额外条件判断

3. **向下取整**：
   - 使用 `Math.floor` 而非 `Math.round`
   - 36小时的记忆显示为"1 day ago"而非"2 days ago"
   - 更保守的时间估计

---

## 关键代码路径与文件引用

### 被调用方
| 文件 | 用途 |
|------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 读取记忆文件时附加 `memoryFreshnessNote` |
| `src/utils/messages.ts` | `wrapMessagesInSystemReminder()` 使用 `memoryFreshnessText` |

### 无内部依赖
该模块是纯工具函数，不依赖其他模块。

### 无外部依赖
仅使用 JavaScript 标准 API（`Date.now()`, `Math`）。

---

## 依赖与外部交互

### 纯函数设计
- 所有函数都是纯函数（给定相同输入，总是返回相同输出）
- 无副作用
- 不依赖外部状态

### 时间源
- 使用 `Date.now()` 获取当前时间
- 在测试环境中可以通过 `jest.useFakeTimers()` 控制

---

## 风险、边界与改进建议

### 已知风险

1. **时钟偏差**
   - 系统时钟可能不准确
   - 未来时间的修改时间戳会被钳制到0
   - 不会崩溃，但可能产生误导性输出

2. **时区问题**
   - `mtimeMs` 通常是本地文件系统时间戳
   - `Date.now()` 是 UTC 时间
   - 跨时区使用时可能有1天的误差

3. **精度限制**
   - 只精确到天，不区分小时/分钟
   - 1.9天的记忆显示为"1 day ago"

### 边界情况

| 输入 | 输出 | 说明 |
|------|------|------|
| `Date.now()` | "today" | 当前时间 |
| `Date.now() - 86400000` | "yesterday" | 24小时前 |
| `Date.now() + 1000000` | "today" | 未来时间（钳制到0） |
| `0` | "N days ago" | 1970年的文件 |
| 负数 | "today" | 异常输入处理 |

### 改进建议

1. **更细粒度的时间描述**
   ```typescript
   // 当前：2 days ago
   // 改进：2 days ago (49 hours)
   ```

2. **可配置的阈值**
   ```typescript
   export function memoryFreshnessText(mtimeMs: number, warningThresholdDays = 1): string
   ```

3. **相对时间格式化选项**
   ```typescript
   export type AgeFormat = 'short' | 'long' | 'verbose'
   export function memoryAge(mtimeMs: number, format: AgeFormat = 'short'): string
   ```

4. **时区感知**
   ```typescript
   // 考虑使用 Intl.RelativeTimeFormat
   const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' })
   return rtf.format(-days, 'day')
   ```

5. **测试辅助函数**
   ```typescript
   // 导出常量便于测试
   export const MS_PER_DAY = 86_400_000
   ```

### 测试建议

```typescript
// 应该测试的边界情况
describe('memoryAgeDays', () => {
  it('returns 0 for today', () => { /* ... */ })
  it('returns 1 for yesterday', () => { /* ... */ })
  it('returns 0 for future timestamps', () => { /* ... */ })
  it('handles daylight saving time boundaries', () => { /* ... */ })
})
```
