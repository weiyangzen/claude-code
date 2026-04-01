# stringUtils.ts 研究文档

## 场景与职责

`stringUtils.ts` 是 Claude Code 的字符串工具函数集合，提供各种字符串处理功能，包括正则转义、大小写转换、复数处理、字符计数、全角字符规范化等。此外，还提供安全字符串累积的 `EndTruncatingAccumulator` 类，用于处理可能产生大量输出的场景。

## 功能点目的

### 基础字符串操作
1. **正则转义** (`escapeRegExp`): 转义正则特殊字符
2. **首字母大写** (`capitalize`): 将字符串首字母转为大写
3. **复数处理** (`plural`): 根据数量选择单复数形式
4. **首行提取** (`firstLineOf`): 高效获取字符串第一行
5. **字符计数** (`countCharInString`): 统计字符出现次数

### 全角字符规范化
1. **全角数字** (`normalizeFullWidthDigits`): 将全角数字转为半角
2. **全角空格** (`normalizeFullWidthSpace`): 将全角空格转为半角

### 安全字符串累积
1. **安全连接** (`safeJoinLines`): 安全地连接字符串数组，支持截断
2. **EndTruncatingAccumulator**: 类实现，从末尾截断的安全累积器

### 行截断
1. **按行截断** (`truncateToLines`): 截断到指定行数

## 具体技术实现

### EndTruncatingAccumulator

核心类，用于安全处理大输出：

```typescript
export class EndTruncatingAccumulator {
  private content: string = ''
  private isTruncated = false
  private totalBytesReceived = 0

  constructor(private readonly maxSize: number = MAX_STRING_LENGTH) {}

  append(data: string | Buffer): void {
    const str = typeof data === 'string' ? data : data.toString()
    this.totalBytesReceived += str.length

    if (this.isTruncated && this.content.length >= this.maxSize) {
      return  // 已截断且已满
    }

    if (this.content.length + str.length > this.maxSize) {
      const remainingSpace = this.maxSize - this.content.length
      if (remainingSpace > 0) {
        this.content += str.slice(0, remainingSpace)
      }
      this.isTruncated = true
    } else {
      this.content += str
    }
  }

  toString(): string {
    if (!this.isTruncated) return this.content
    const truncatedBytes = this.totalBytesReceived - this.maxSize
    const truncatedKB = Math.round(truncatedBytes / 1024)
    return this.content + `\n... [output truncated - ${truncatedKB}KB removed]`
  }
}
```

### 关键常量

```typescript
const MAX_STRING_LENGTH = 2 ** 25  // 33,554,432 字符 (~32MB)
```

### 全角字符处理

```typescript
export function normalizeFullWidthDigits(input: string): string {
  return input.replace(/[０-９]/g, ch =>
    String.fromCharCode(ch.charCodeAt(0) - 0xfee0),
  )
}

export function normalizeFullWidthSpace(input: string): string {
  return input.replace(/\u3000/g, ' ')
}
```

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `escapeRegExp` | 函数 | 正则转义 |
| `capitalize` | 函数 | 首字母大写 |
| `plural` | 函数 | 复数处理 |
| `firstLineOf` | 函数 | 首行提取 |
| `countCharInString` | 函数 | 字符计数 |
| `normalizeFullWidthDigits` | 函数 | 全角数字规范化 |
| `normalizeFullWidthSpace` | 函数 | 全角空格规范化 |
| `safeJoinLines` | 函数 | 安全连接 |
| `EndTruncatingAccumulator` | 类 | 安全累积器 |
| `truncateToLines` | 函数 | 按行截断 |

### 依赖模块
无外部依赖。

### 调用方

通过 Grep 发现大量调用方，主要包括：

| 文件 | 使用功能 |
|------|---------|
| `src/components/**/*.tsx` | `capitalize`, `plural` |
| `src/tools/**/*.ts` | `escapeRegExp`, `capitalize` |
| `src/utils/streamlinedTransform.ts` | `capitalize` |
| `src/utils/doctorContextWarnings.ts` | `plural` |
| `src/utils/messages.ts` | `escapeRegExp` |
| `src/utils/ripgrep.ts` | `escapeRegExp` |
| `src/utils/task/TaskOutput.ts` | `EndTruncatingAccumulator` |
| `src/utils/shell/prefix.ts` | `firstLineOf` |

## 依赖与外部交互

### 与 Shell 系统的集成
- `EndTruncatingAccumulator` 被 Shell 工具使用处理命令输出
- 溢出内容会被写入磁盘（由 ShellCommand 处理）

### 与 UI 组件的集成
- `capitalize` 和 `plural` 广泛用于组件显示文本
- 与 `figures` 等终端显示库配合使用

### 与搜索系统的集成
- `escapeRegExp` 用于 Grep 和 Ripgrep 工具的模式转义
- 确保用户输入不会破坏正则表达式

## 风险、边界与改进建议

### 潜在风险

1. **内存限制**: `MAX_STRING_LENGTH` 为 32MB，在某些场景可能仍过大
2. **编码问题**: `Buffer.toString()` 默认使用 UTF-8，其他编码可能有问题
3. **精度丢失**: `normalizeFullWidthDigits` 只处理数字，不处理其他全角字符

### 边界情况

1. **空字符串**: 所有函数正确处理空字符串
2. **超大输入**: `EndTruncatingAccumulator` 截断超大输入
3. **负数长度**: `plural` 函数对负数使用复数形式（可能不符合某些语言习惯）

### 改进建议

1. **更多全角字符支持**: 扩展全角字符规范化
```typescript
export function normalizeFullWidthChars(input: string): string {
  // 处理全角字母、标点等
  return input.replace(/[\uFF01-\uFF5E]/g, ch =>
    String.fromCharCode(ch.charCodeAt(0) - 0xFEE0)
  )
}
```

2. **国际化复数**: 支持更复杂的复数规则
```typescript
// 使用 Intl.PluralRules
export function pluralI18n(n: number, locale: string, forms: string[]): string {
  const rule = new Intl.PluralRules(locale).select(n)
  return forms[['zero', 'one', 'two', 'few', 'many', 'other'].indexOf(rule)]
}
```

3. **流式处理**: 为超大字符串添加流式处理支持
```typescript
export async function* streamLines(lines: string[]): AsyncGenerator<string> {
  for (const line of lines) {
    yield line
  }
}
```

4. **性能优化**: 对于高频调用的函数，考虑使用更快的算法
```typescript
// 使用 Boyer-Moore 或类似算法优化 countCharInString
export function countCharFast(str: string, char: string): number {
  // 实现更快的算法
}
```

5. **类型安全**: 添加更严格的类型约束
```typescript
// 使用模板字面量类型
export function plural<N extends number>(
  n: N,
  singular: string,
  plural?: string
): string
```
