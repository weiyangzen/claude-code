# keyword.ts 研究文档

## 场景与职责

`keyword.ts` 是 Claude Code CLI 中 `/ultraplan` 和 `/ultrareview` 功能的关键词检测模块。负责在用户输入中检测触发词（"ultraplan" 和 "ultrareview"），并提供智能过滤以避免误触发。

**核心场景：**
1. **关键词触发**：用户在普通提示词中包含 "ultraplan"，系统自动路由到 `/ultraplan` 命令
2. **视觉反馈**：在 PromptInput 中高亮显示关键词（彩虹色动画）
3. **输入重写**：将 "ultraplan" 替换为 "plan" 以保持转发到 CCR 时的语法正确性

**文件位置**：`src/utils/ultraplan/keyword.ts` (127 行)

---

## 功能点目的

### 1. 智能关键词检测

不同于简单的字符串匹配，本模块实现了复杂的关键词检测逻辑，能够：
- 识别真正的用户意图（避免代码、路径中的误匹配）
- 处理各种边界情况（引号内、文件路径、命令行参数等）
- 支持大小写不敏感匹配

### 2. 触发词高亮

为 PromptInput 组件提供位置信息，实现彩虹色字符级高亮效果。

### 3. 输入重写

在将提示词转发到远程 CCR 之前，将 "ultraplan" 替换为 "plan"，使提示词保持语法正确：
- `"please ultraplan this"` → `"please plan this"`

---

## 具体技术实现

### 数据结构

```typescript
type TriggerPosition = { 
  word: string    // 匹配到的词（保留原始大小写）
  start: number   // 在文本中的起始位置
  end: number     // 在文本中的结束位置
}
```

### 配对定界符映射

```typescript
const OPEN_TO_CLOSE: Record<string, string> = {
  '`': '`',   // 反引号（代码）
  '"': '"',   // 双引号
  '<': '>',   // 尖括号（标签）
  '{': '}',   // 花括号
  '[': ']',   // 方括号
  '(': ')',   // 圆括号
  "'": "'",   // 单引号
}
```

### 核心算法：findKeywordTriggerPositions

**步骤 1：引号范围检测**

遍历文本，识别所有被定界符包围的范围：

```typescript
for (let i = 0; i < text.length; i++) {
  const ch = text[i]!
  if (openQuote) {
    // 检查闭合条件
    if (ch !== OPEN_TO_CLOSE[openQuote]) continue
    if (openQuote === "'" && isWord(text[i + 1])) continue  // 撇号处理
    quotedRanges.push({ start: openAt, end: i + 1 })
    openQuote = null
  } else if (
    (ch === '<' && i + 1 < text.length && /[a-zA-Z/]/.test(text[i + 1]!)) ||
    (ch === "'" && !isWord(text[i - 1])) ||
    (ch !== '<' && ch !== "'" && ch in OPEN_TO_CLOSE)
  ) {
    openQuote = ch
    openAt = i
  }
}
```

**特殊处理：**
- `<` 仅在后跟字母或 `/` 时视为标签开始（避免 `n < 5` 被误判）
- `'` 仅在前面不是单词字符时视为引号（避免 "let's" 中的撇号）
- `[[` 嵌套处理（`[Pasted text #N]` 占位符）

**步骤 2：关键词匹配与过滤**

```typescript
const wordRe = new RegExp(`\\b${keyword}\\b`, 'gi')
const matches = text.matchAll(wordRe)
for (const match of matches) {
  if (match.index === undefined) continue
  const start = match.index
  const end = start + match[0].length
  
  // 过滤 1：在引号范围内
  if (quotedRanges.some(r => start >= r.start && start < r.end)) continue
  
  // 过滤 2：路径/标识符上下文
  const before = text[start - 1]
  const after = text[end]
  if (before === '/' || before === '\\' || before === '-') continue
  if (after === '/' || after === '\\' || after === '-' || after === '?') continue
  if (after === '.' && isWord(text[end + 1])) continue
  
  positions.push({ word: match[0], start, end })
}
```

**过滤规则详解：**

| 规则 | 示例（被过滤） | 示例（通过） |
|------|---------------|-------------|
| 引号内 | `` `ultraplan` `` | `please ultraplan this` |
| 路径前缀 | `src/ultraplan/foo.ts` | - |
| 路径后缀 | `ultraplan.tsx` | - |
| 命令行参数 | `--ultraplan-mode` | - |
| 问号结尾 | `what is ultraplan?` | `use ultraplan.` |
| 文件扩展名 | `ultraplan.ts` | `ultraplan.` |

**步骤 3：Slash 命令排除**

```typescript
if (text.startsWith('/')) return []
```

确保 `/rename ultraplan foo` 不会触发关键词检测（实际执行的是 `/rename`）。

### 公共 API

```typescript
// 查找 ultraplan 触发位置
export function findUltraplanTriggerPositions(text: string): TriggerPosition[]

// 查找 ultrareview 触发位置
export function findUltrareviewTriggerPositions(text: string): TriggerPosition[]

// 检查是否包含 ultraplan 关键词
export function hasUltraplanKeyword(text: string): boolean

// 检查是否包含 ultrareview 关键词
export function hasUltrareviewKeyword(text: string): boolean

// 替换第一个可触发的 "ultraplan" 为 "plan"
export function replaceUltraplanKeyword(text: string): string
```

### 替换逻辑

```typescript
export function replaceUltraplanKeyword(text: string): string {
  const [trigger] = findUltraplanTriggerPositions(text)
  if (!trigger) return text
  
  const before = text.slice(0, trigger.start)
  const after = text.slice(trigger.end)
  
  // 如果替换后只剩空白，返回空字符串
  if (!(before + after).trim()) return ''
  
  // 保留用户的大小写（"Ultraplan" → "Plan"）
  return before + trigger.word.slice('ultra'.length) + after
}
```

---

## 关键代码路径与文件引用

### 调用链

```
用户输入
  └── PromptInput.tsx
      └── findUltraplanTriggerPositions() (用于高亮)
      └── findUltrareviewTriggerPositions() (用于高亮)
      
用户提交
  └── processUserInput.ts
      └── hasUltraplanKeyword() (检测)
      └── replaceUltraplanKeyword() (重写)
      └── processSlashCommand('/ultraplan ...')
```

### 依赖文件

| 文件 | 用途 |
|------|------|
| 无外部依赖 | 纯文本处理模块 |

### 被调用方

| 文件 | 用途 |
|------|------|
| `src/components/PromptInput/PromptInput.tsx` | 彩虹高亮、通知提示 |
| `src/utils/processUserInput/processUserInput.ts` | 关键词检测和路由 |

---

## 依赖与外部交互

### 与 PromptInput 的集成

**彩虹高亮实现：**
```typescript
// PromptInput.tsx
const ultraplanTriggers = useMemo(() => 
  feature('ULTRAPLAN') && !ultraplanSessionUrl && !ultraplanLaunching 
    ? findUltraplanTriggerPositions(displayedValue) 
    : [], 
  [displayedValue, ultraplanSessionUrl, ultraplanLaunching]
)

// 在 combinedHighlights 中
if (feature('ULTRAPLAN')) {
  for (const trigger of ultraplanTriggers) {
    for (let i = trigger.start; i < trigger.end; i++) {
      highlights.push({
        start: i,
        end: i + 1,
        color: getRainbowColor(i - trigger.start),
        shimmerColor: getRainbowColor(i - trigger.start, true),
        priority: 10
      })
    }
  }
}
```

**通知提示：**
```typescript
useEffect(() => {
  if (feature('ULTRAPLAN') && ultraplanTriggers.length) {
    addNotification({
      key: 'ultraplan-active',
      text: 'This prompt will launch an ultraplan session in Claude Code on the web',
      priority: 'immediate',
      timeoutMs: 5000
    })
  } else {
    removeNotification('ultraplan-active')
  }
}, [addNotification, removeNotification, ultraplanTriggers.length])
```

### 与 processUserInput 的集成

**关键词路由逻辑：**
```typescript
// processUserInput.ts
if (
  feature('ULTRAPLAN') &&
  mode === 'prompt' &&
  !context.options.isNonInteractiveSession &&
  inputString !== null &&
  !effectiveSkipSlash &&
  !inputString.startsWith('/') &&
  !context.getAppState().ultraplanSessionUrl &&
  !context.getAppState().ultraplanLaunching &&
  hasUltraplanKeyword(preExpansionInput ?? inputString)
) {
  logEvent('tengu_ultraplan_keyword', {})
  const rewritten = replaceUltraplanKeyword(inputString).trim()
  const { processSlashCommand } = await import('./processSlashCommand.js')
  const slashResult = await processSlashCommand(
    `/ultraplan ${rewritten}`,
    precedingInputBlocks,
    imageContentBlocks,
    [],
    context,
    setToolJSX,
    uuid,
    isAlreadyProcessing,
    canUseTool,
  )
  return addImageMetadataMessage(slashResult, imageMetadataTexts)
}
```

**关键设计决策：**
1. 在 `preExpansionInput` 上检测（粘贴内容展开前），避免粘贴内容中的关键词误触发
2. 仅在 `prompt` 模式下触发（非 bash/slash 命令模式）
3. 检查 `ultraplanSessionUrl` 和 `ultraplanLaunching` 防止重复启动
4. 重写后的提示词转发到 `/ultraplan` 命令处理

---

## 风险、边界与改进建议

### 已知风险

1. **正则表达式性能**
   - 使用 `\b` 单词边界和 `gi` 全局忽略大小写标志
   - 长文本输入可能导致多次正则匹配
   - 当前实现遍历文本两次（一次引号检测，一次匹配过滤）

2. **Unicode 支持**
   - `isWord` 函数使用 `[\p{L}\p{N}_]` Unicode 属性转义
   - 需要确保运行环境支持 ES2018+ Unicode 属性类

3. **误过滤风险**
   - `ultraplan?` 被过滤（用户询问功能时不会触发）
   - `ultraplan.tsx` 被过滤（文件路径）
   - 但 `ultraplan-mode` 也被过滤（可能期望触发）

### 边界情况

1. **多重触发词**
   - 输入 `"ultraplan and ultraplan again"` 会找到两个位置
   - 但 `replaceUltraplanKeyword` 只替换第一个

2. **嵌套引号**
   - `"ultraplan in quotes" and ultraplan outside`
   - 只有外部的会触发

3. **大小写混合**
   - `"Ultraplan"`, `"ULTRAPLAN"`, `"ultraPlan"` 都能匹配
   - 替换时保留用户的大小写（`"Ultraplan"` → `"Plan"`）

4. **粘贴内容处理**
   - 检测在 `preExpansionInput` 上进行
   - 重写后的文本包含粘贴内容
   - 避免用户粘贴包含 "ultraplan" 的代码片段时误触发

### 改进建议

1. **性能优化**
   - 考虑使用单次遍历算法同时检测引号和关键词
   - 对于超长输入（>10KB）可以添加快速路径（简单字符串搜索）

2. **可配置性**
   - 添加用户设置禁用关键词触发
   - 允许自定义关键词（如 "plan" 的变体）

3. **更智能的过滤**
   - 考虑使用机器学习模型判断用户意图
   - 添加 "确认" 步骤对于模糊的匹配

4. **扩展支持**
   - 支持更多编程语言的字符串字面量（如 Python 的三引号）
   - 支持模板字符串中的插值检测

5. **测试覆盖**
   - 添加单元测试覆盖各种边界情况
   - 测试不同自然语言的输入
   - 测试性能边界（超长输入）

### 相关实现对比

与 `thinking.ts` 中的 `findThinkingTriggerPositions` 保持一致的 API 设计：

```typescript
// thinking.ts
export function findThinkingTriggerPositions(text: string): TriggerPosition[]

// keyword.ts  
export function findUltraplanTriggerPositions(text: string): TriggerPosition[]
export function findUltrareviewTriggerPositions(text: string): TriggerPosition[]
```

这种一致性使得 `PromptInput.tsx` 可以统一处理不同类型的触发词高亮。
