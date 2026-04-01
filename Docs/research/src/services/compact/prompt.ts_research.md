# prompt.ts 深度研究文档

## 场景与职责

`prompt.ts` 是 Claude Code 压缩(compact)功能的提示词(prompt)管理模块，负责生成和管理用于对话摘要生成的各种提示词模板。该模块定义了模型如何总结对话内容的指令，包括完整压缩、部分压缩以及用户摘要消息的格式化。

**核心场景：**
1. **完整对话压缩 (Full Compact)** - 总结整个对话历史
2. **部分对话压缩 (Partial Compact)** - 总结对话的特定部分（从某点开始或到某点结束）
3. **用户摘要消息生成** - 将模型生成的摘要格式化为用户可见的消息

**关键设计决策：**
- 使用 XML 标签（`<analysis>` 和 `<summary>`）结构化模型输出，便于后续解析
- 提供 "无工具" 前置提示，防止模型在压缩过程中调用工具（浪费回合）
- 区分 `from` 和 `up_to` 两种部分压缩方向，适应不同的缓存策略需求

## 功能点目的

### 1. 无工具前置提示 (NO_TOOLS_PREAMBLE)

**目的**：防止模型在压缩过程中尝试调用工具

**背景**：
- 缓存共享的分支路径继承父进程的完整工具集（为了缓存键匹配）
- 在 Sonnet 4.6+ 自适应思考模型上，即使有较弱的尾部指令，模型有时会尝试工具调用
- `maxTurns: 1` 意味着如果被拒绝的工具调用会导致没有文本输出，退流到流式回退

**实现**：
```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`
```

### 2. 详细分析指令 (Detailed Analysis Instructions)

**目的**：指导模型进行结构化的对话分析

**两个变体**：
- `DETAILED_ANALYSIS_INSTRUCTION_BASE`：分析整个对话（用于完整压缩）
- `DETAILED_ANALYSIS_INSTRUCTION_PARTIAL`：仅分析最近的消息（用于部分压缩）

**分析要求**：
1. 按时间顺序分析每条消息
2. 识别用户明确请求和意图
3. 记录技术概念、代码模式
4. 提取具体细节（文件名、代码片段、函数签名）
5. 记录错误及修复方法
6. 关注用户反馈

### 3. 完整压缩提示 (BASE_COMPACT_PROMPT)

**目的**：生成整个对话的详细摘要

**摘要结构（9个部分）**：
1. **Primary Request and Intent** - 用户的主要请求和意图
2. **Key Technical Concepts** - 关键技术概念
3. **Files and Code Sections** - 文件和代码段（含完整代码片段）
4. **Errors and fixes** - 错误及修复
5. **Problem Solving** - 问题解决过程
6. **All user messages** - 所有非工具结果的用户消息
7. **Pending Tasks** - 待处理任务
8. **Current Work** - 当前正在进行的工作
9. **Optional Next Step** - 可选的下一步（基于最近对话）

### 4. 部分压缩提示 (PARTIAL_COMPACT_PROMPT)

**目的**：总结对话的最近部分（保留早期上下文）

**与完整压缩的区别**：
- 聚焦于 "RECENT" 消息
- 早期消息保持完整，不需要总结
- 适用于保留早期上下文的同时压缩近期对话

### 5. 前缀保留压缩提示 (PARTIAL_COMPACT_UP_TO_PROMPT)

**目的**：总结某点之前的对话，保留之后的消息

**使用场景**：
- 模型只看到被总结的前缀（缓存命中）
- 摘要将位于保留的近期消息之前
- 适用于 "up_to" 方向的部分压缩

**差异点**：
- 第8节为 "Work Completed"（已完成的工作）
- 第9节为 "Context for Continuing Work"（继续工作所需的上下文）

### 6. 提示获取函数

#### `getCompactPrompt(customInstructions?: string): string`
- 返回完整压缩提示
- 支持自定义指令追加

#### `getPartialCompactPrompt(customInstructions?: string, direction?: PartialCompactDirection): string`
- 返回部分压缩提示
- `direction`: `'from'` 或 `'up_to'`

### 7. 摘要格式化 (formatCompactSummary)

**目的**：处理模型输出，提取可用的摘要内容

**处理步骤**：
1. 移除 `<analysis>` 区段（草稿垫，无信息价值）
2. 提取 `<summary>` 内容并替换为 "Summary:" 标题
3. 清理多余空行

**示例转换**：
```
输入：
<analysis>
思考过程...
</analysis>

<summary>
1. Primary Request: ...
</summary>

输出：
Summary:
1. Primary Request: ...
```

### 8. 用户摘要消息生成 (getCompactUserSummaryMessage)

**目的**：生成压缩后显示给用户的摘要消息

**功能**：
- 格式化摘要内容
- 可选添加完整转录路径提示
- 可选添加 "近期消息已保留" 提示
- 支持抑制后续问题（自动压缩时使用）
- 主动/自主模式检测和提示

**主动模式特殊处理**：
```typescript
if (feature('PROACTIVE') || feature('KAIROS')) {
  if (proactiveModule?.isProactiveActive()) {
    continuation += `
You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already working autonomously before compaction. Continue your work loop: pick up where you left off based on the summary above. Do not greet the user or ask what to work on.`
  }
}
```

## 具体技术实现

### 数据结构

```typescript
// 部分压缩方向类型
export type PartialCompactDirection = 'from' | 'up_to'
// 'from': 总结某点之后的消息，保留之前的
// 'up_to': 总结某点之前的消息，保留之后的
```

### 提示组装流程

```
NO_TOOLS_PREAMBLE
  + BASE_COMPACT_PROMPT / PARTIAL_COMPACT_PROMPT / PARTIAL_COMPACT_UP_TO_PROMPT
  + [可选] Additional Instructions (customInstructions)
  + NO_TOOLS_TRAILER
```

### 正则表达式处理

```typescript
// 移除 analysis 区段
formattedSummary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')

// 提取 summary 内容
const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
```

### 条件特性导入

```typescript
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? (require('../../proactive/index.js') as typeof import('../../proactive/index.js'))
    : null
```

## 关键代码路径与文件引用

### 调用方 (Callers)

| 文件路径 | 调用函数 | 用途 |
|---------|---------|------|
| `src/services/compact/compact.ts:440` | `getCompactPrompt` | 完整压缩 |
| `src/services/compact/compact.ts:614` | `getCompactUserSummaryMessage` | 生成用户摘要消息 |
| `src/services/compact/compact.ts:840` | `getPartialCompactPrompt` | 部分压缩 |
| `src/services/compact/sessionMemoryCompact.ts:464` | `getCompactUserSummaryMessage` | Session memory 压缩摘要 |

### 常量定义

| 常量 | 用途 |
|-----|------|
| `NO_TOOLS_PREAMBLE` | 强制文本输出的前置警告 |
| `NO_TOOLS_TRAILER` | 末尾提醒 |
| `DETAILED_ANALYSIS_INSTRUCTION_BASE` | 完整分析指令 |
| `DETAILED_ANALYSIS_INSTRUCTION_PARTIAL` | 部分分析指令 |
| `BASE_COMPACT_PROMPT` | 完整压缩模板 |
| `PARTIAL_COMPACT_PROMPT` | 部分压缩模板（from方向） |
| `PARTIAL_COMPACT_UP_TO_PROMPT` | 部分压缩模板（up_to方向） |

## 依赖与外部交互

### 导入依赖

```typescript
import { feature } from 'bun:bundle'
import type { PartialCompactDirection } from '../../types/message.js'
```

### 特性标志依赖

| 特性标志 | 用途 |
|---------|------|
| `PROACTIVE` | 控制主动模式检测 |
| `KAIROS` | 控制主动模式检测（替代标志） |

### 动态导入

```typescript
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? (require('../../proactive/index.js') as typeof import('../../proactive/index.js'))
    : null
```

## 风险、边界与改进建议

### 已知风险

1. **模型不遵守无工具指令**
   - 尽管有强力的前置和后置提示，Sonnet 4.6+ 仍可能尝试工具调用
   - 缓解：`maxTurns: 1` 确保工具调用被拒绝后不会继续

2. **XML 解析失败**
   - 如果模型输出格式不正确，`formatCompactSummary` 可能无法提取摘要
   - 缓解：返回原始文本，调用方处理

3. **自定义指令注入风险**
   - `customInstructions` 直接拼接到提示中
   - 如果包含恶意内容可能影响模型行为
   - 缓解：调用方负责验证用户输入

### 边界情况

1. **空自定义指令**
   ```typescript
   if (customInstructions && customInstructions.trim() !== '')
   ```
   正确处理空字符串和纯空白字符

2. **Summary 标签不存在**
   如果模型输出不包含 `<summary>` 标签，返回处理后的全文

3. **Analysis 区段位置**
   假设 `<analysis>` 在 `<summary>` 之前，如果顺序颠倒可能处理不正确

### 改进建议

1. **添加结构化输出模式 (Structured Output)**
   - 使用 Anthropic API 的结构化输出功能替代 XML 标签解析
   - 提高摘要提取的可靠性

2. **提示版本控制**
   - 添加提示版本号，便于 A/B 测试和回滚
   - 记录不同版本的性能指标

3. **多语言支持**
   - 当前提示为英文，考虑根据用户设置本地化
   - 关键术语保持一致性

4. **自定义指令长度限制**
   - 添加 `customInstructions` 长度检查
   - 过长的指令可能稀释核心提示效果

5. **摘要质量验证**
   - 添加启发式检查验证摘要完整性
   - 例如检查是否包含关键部分（用户请求、文件列表等）

6. **错误恢复机制**
   - 如果 `formatCompactSummary` 检测到格式问题，尝试替代解析策略
   - 记录解析失败案例用于改进提示

### 代码示例：改进的格式验证

```typescript
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary

  // 移除 analysis 区段
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )

  // 提取并格式化 summary 区段
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    const content = summaryMatch[1] || ''
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${content.trim()}`,
    )
  } else {
    // 建议：添加警告日志
    logForDebugging('Compact summary missing <summary> tags', { level: 'warn' })
  }

  // 清理多余空行
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')

  return formattedSummary.trim()
}
```
