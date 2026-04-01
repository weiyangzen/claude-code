# textHighlighting.ts 研究文档

## 场景与职责

`textHighlighting.ts` 是 Claude Code CLI 的文本高亮处理模块，负责将高亮标记（highlight markers）应用到文本内容，生成带有 ANSI 颜色代码的分段文本。这是实现 UI 中高亮显示（如搜索匹配、关键词标记等）的核心组件。

主要使用场景：
1. **搜索高亮**：在搜索结果中高亮匹配的关键词
2. **关键词标记**：如 "ultrathink" 关键词的彩虹色高亮
3. **输入框高亮**：PromptInput 组件中的文本高亮显示
4. **差异高亮**：diff 视图中的添加/删除标记

## 功能点目的

### 1. 高亮分段 (`segmentTextByHighlights`)
- **输入**：原始文本 + 高亮标记数组
- **输出**：分段数组，每段包含文本和可选的高亮信息
- **处理**：解决重叠高亮、按优先级排序、生成连续分段

### 2. ANSI 感知的高亮应用 (`HighlightSegmenter`)
- **问题**：文本中可能已包含 ANSI 颜色代码，高亮不能破坏现有格式
- **解决方案**：使用 `@alcalzone/ansi-tokenize` 进行 token 级处理，在高亮边界处正确插入/重置 ANSI 代码

### 3. 重叠高亮处理
- **策略**：按优先级排序，高优先级高亮优先应用
- **冲突解决**：重叠区域只保留一个高亮（优先级高的）

## 具体技术实现

### 核心类型定义

```typescript
export type TextHighlight = {
  start: number        // 高亮起始位置（可见字符索引）
  end: number          // 高亮结束位置（不包含）
  color: keyof Theme | undefined  // 主题颜色键
  dimColor?: boolean   // 是否使用暗淡变体
  inverse?: boolean    // 是否反色
  shimmerColor?: keyof Theme  // 闪烁效果颜色
  priority: number     // 优先级（高优先级的覆盖低优先级）
}

export type TextSegment = {
  text: string         // 分段文本（包含 ANSI 代码）
  start: number        // 在原始文本中的起始位置
  highlight?: TextHighlight  // 关联的高亮信息
}
```

### 核心函数：`segmentTextByHighlights`

```typescript
export function segmentTextByHighlights(
  text: string,
  highlights: TextHighlight[],
): TextSegment[]
```

**算法流程**：

1. **空高亮优化**
   ```typescript
   if (highlights.length === 0) {
     return [{ text, start: 0 }]
   }
   ```

2. **排序高亮**（按起始位置，相同位置按优先级降序）
   ```typescript
   const sortedHighlights = [...highlights].sort((a, b) => {
     if (a.start !== b.start) return a.start - b.start
     return b.priority - a.priority
   })
   ```

3. **解决重叠**（贪心算法）
   ```typescript
   for (const highlight of sortedHighlights) {
     if (highlight.start === highlight.end) continue  // 跳过空高亮
     
     const overlaps = usedRanges.some(range => /* 重叠检测 */)
     if (!overlaps) {
       resolvedHighlights.push(highlight)
       usedRanges.push({ start: highlight.start, end: highlight.end })
     }
   }
   ```

4. **生成分段**（委托给 `HighlightSegmenter`）

### 核心类：`HighlightSegmenter`

```typescript
class HighlightSegmenter {
  private readonly tokens: Token[]    // ANSI token 数组
  private visiblePos = 0              // 可见字符位置（排除 ANSI 代码）
  private stringPos = 0               // 原始字符串位置（包含 ANSI 代码）
  private tokenIdx = 0                // 当前 token 索引
  private charIdx = 0                 // 当前 token 内的字符偏移
  private codes: AnsiCode[] = []      // 当前活动的 ANSI 代码
}
```

**双位置系统**：
- `visiblePos`：用户看到的位置（用于高亮边界）
- `stringPos`：原始字符串中的字节位置（用于切片）

**分段算法** (`segmentTo`)：

1. **消费前置 ANSI 代码**
   ```typescript
   while (token.type === 'ansi') {
     this.codes.push(token)
     this.stringPos += token.code.length
     this.tokenIdx++
   }
   ```

2. **推进到目标位置**
   ```typescript
   while (this.visiblePos < targetVisiblePos && this.tokenIdx < this.tokens.length) {
     if (token.type === 'ansi') {
       // 累积 ANSI 代码
     } else {
       // 消费文本字符，更新 visiblePos 和 stringPos
     }
   }
   ```

3. **生成带 ANSI 包装的分段**
   ```typescript
   const prefixCodes = reduceCodes(codesStart)
   const suffixCodes = reduceCodes(this.codes)
   
   return {
     text: prefix + rawText + suffix,
     start: visibleStart,
   }
   ```

### ANSI 代码优化 (`reduceCodes`)

```typescript
function reduceCodes(codes: AnsiCode[]): AnsiCode[] {
  return reduceAnsiCodes(codes).filter(c => c.code !== c.endCode)
}
```

- 使用 `@alcalzone/ansi-tokenize` 的 `reduceAnsiCodes` 消除冗余代码
- 过滤掉 "end codes"（如超链接关闭），只保留 "start codes"

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/components/TextInput.tsx` | 文本输入框高亮 |
| `src/components/BaseTextInput.tsx` | 基础文本输入高亮 |
| `src/components/VimTextInput.tsx` | Vim 模式输入高亮 |
| `src/components/PromptInput/PromptInput.tsx` | 提示输入框高亮 |
| `src/components/PromptInput/ShimmeredInput.tsx` | 闪烁效果输入高亮 |
| `src/types/textInputTypes.ts` | 类型定义引用 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `@alcalzone/ansi-tokenize` | ANSI token 解析和处理 |
| `./theme.js` | `Theme` 类型定义 |

## 依赖与外部交互

### 与 @alcalzone/ansi-tokenize 的协作

该库提供精确的 ANSI 转义序列解析：

```typescript
import {
  type AnsiCode,
  ansiCodesToString,
  reduceAnsiCodes,
  type Token,
  tokenize,
  undoAnsiCodes,
} from '@alcalzone/ansi-tokenize'
```

**关键功能**：
- `tokenize(text)`：将文本分割为 ANSI 代码和文本 token
- `reduceAnsiCodes(codes)`：优化 ANSI 代码序列（如合并重复设置）
- `ansiCodesToString(codes)`：将代码数组转换为转义序列字符串
- `undoAnsiCodes(codes)`：生成对应的重置代码

### 与 Theme 的集成

高亮颜色通过 `keyof Theme` 类型与主题系统关联：
- 实际颜色解析在渲染层完成（如使用 chalk）
- 本模块只负责确定哪些字符需要高亮，不负责具体颜色值

### 位置系统说明

**为什么需要双位置系统？**

考虑文本：`"\x1b[31mred\x1b[0m text"`

| 字符 | 原始位置 | 可见位置 |
|-----|---------|---------|
| `\x1b[31m` | 0-4 | -（宽度 0）|
| `r` | 5 | 0 |
| `e` | 6 | 1 |
| `d` | 7 | 2 |
| `\x1b[0m` | 8-11 | -（宽度 0）|
| ` ` | 12 | 3 |
| `t` | 13 | 4 |

高亮标记使用可见位置（如 `{start: 0, end: 3}` 表示 "red"），但切片需要原始位置。

## 风险、边界与改进建议

### 潜在风险

1. **性能问题**
   - `tokenize` 需要完整解析整个文本
   - 对于超长文本（如 MB 级），可能产生大量 token
   - 当前实现会遍历所有高亮之间的文本

2. **高亮冲突处理过于简单**
   - 贪心算法可能不是最优解
   - 示例：高亮 A(0-10, priority=1) 和 B(5-15, priority=2) → B 完全覆盖 A 的 5-10 区域
   - 理想情况下应该分割为：0-5(A), 5-10(B), 10-15(B)

3. **零宽度字符处理**
   - 当前 `visiblePos` 推进逻辑假设每个文本字符宽度为 1
   - 未正确处理零宽度字符（如组合标记）

### 边界条件

| 场景 | 行为 |
|-----|------|
| 空高亮数组 | 返回单一段落包含完整文本 |
| `start === end` | 跳过该高亮（视为空） |
| 高亮超出文本范围 | `segmentTo(Infinity)` 会消费到文本末尾 |
| 所有高亮相互重叠 | 只保留优先级最高的不重叠子集 |
| 文本含未闭合 ANSI 代码 | 保留到分段末尾，不自动重置 |
| 相邻高亮 | 产生独立分段，可能优化为合并 |

### 改进建议

1. **更智能的重叠处理**
   ```typescript
   // 建议：分割重叠区域而非完全丢弃低优先级
   // 输入：A(0-10, p=1), B(5-15, p=2)
   // 当前输出：[0-10(A被丢弃)], [5-15(B)]
   // 建议输出：[0-5(A)], [5-10(B)], [10-15(B)]
   ```

2. **性能优化**
   ```typescript
   // 建议：对于无 ANSI 的纯文本使用快速路径
   if (!text.includes('\x1b')) {
     // 直接基于字符串切片，跳过 tokenize
   }
   ```

3. **支持更多高亮属性**
   ```typescript
   interface TextHighlight {
     // 现有属性...
     bold?: boolean
     italic?: boolean
     underline?: boolean
     strikethrough?: boolean
   }
   ```

4. **流式处理**
   ```typescript
   // 建议：支持流式文本，避免一次性加载大文本
   class StreamingHighlightSegmenter {
     push(chunk: string): TextSegment[]
     end(): TextSegment[]
   }
   ```

5. **更好的零宽度字符支持**
   ```typescript
   // 建议：集成 stringWidth 计算可见宽度
   import { stringWidth } from '../ink/stringWidth.js'
   // 在 visiblePos 推进时使用 stringWidth(char)
   ```

6. **缓存机制**
   ```typescript
   // 建议：缓存 tokenize 结果，避免重复解析相同文本
   const tokenCache = new LRUCache<string, Token[]>({ max: 100 })
   ```

### 测试建议

应覆盖以下场景：
- 纯文本（无 ANSI）的高亮
- 含 ANSI 代码的文本高亮
- 重叠高亮的各种组合
- 空高亮、空文本
- 高亮边界恰好是 ANSI 代码位置
- 超长文本性能
- 特殊 Unicode 字符（emoji、CJK、组合字符）
