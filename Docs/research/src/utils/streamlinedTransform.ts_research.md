# streamlinedTransform.ts 研究文档

## 场景与职责

`streamlinedTransform.ts` 实现了 Claude Code 的 "streamlined" 输出模式转换器。该模式是一种"抗蒸馏"的输出格式，旨在为 SDK 和自动化场景提供更简洁、更易处理的输出流。

## 功能点目的

### Streamlined 输出模式
- **目标**: 提供一种简洁的输出格式，适合自动化处理和日志记录
- **策略**:
  - 保持文本消息完整
  - 累积工具调用并生成摘要（当文本出现时重置计数）
  - 省略思考内容
  - 从初始化消息中剥离工具列表和模型信息

### 工具调用摘要
将多个工具调用归类为：
- 搜索操作（Grep、Glob、WebSearch、LSP）
- 读取操作（FileRead、ListMcpResources）
- 写入操作（FileWrite、FileEdit、NotebookEdit）
- 命令操作（Shell 工具、Tmux、TaskStop）
- 其他工具

## 具体技术实现

### 工具分类

```typescript
const SEARCH_TOOLS = [
  GREP_TOOL_NAME,      // 'Grep'
  GLOB_TOOL_NAME,      // 'Glob'
  WEB_SEARCH_TOOL_NAME,// 'WebSearch'
  LSP_TOOL_NAME,       // 'LSP'
]
const READ_TOOLS = [FILE_READ_TOOL_NAME, LIST_MCP_RESOURCES_TOOL_NAME]
const WRITE_TOOLS = [
  FILE_WRITE_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
]
const COMMAND_TOOLS = [...SHELL_TOOL_NAMES, 'Tmux', TASK_STOP_TOOL_NAME]
```

### 转换器工厂函数

```typescript
export function createStreamlinedTransformer(): (
  message: StdoutMessage,
) => StdoutMessage | null {
  let cumulativeCounts = createEmptyToolCounts()

  return function transformToStreamlined(
    message: StdoutMessage,
  ): StdoutMessage | null {
    switch (message.type) {
      case 'assistant': {
        const text = extractTextContent(content, '\n').trim()
        accumulateToolUses(message, cumulativeCounts)

        if (text.length > 0) {
          // 文本消息：输出文本并重置计数
          cumulativeCounts = createEmptyToolCounts()
          return { type: 'streamlined_text', text, ... }
        }

        // 纯工具消息：输出累积摘要
        const toolSummary = getToolSummaryText(cumulativeCounts)
        if (!toolSummary) return null
        return { type: 'streamlined_tool_use_summary', tool_summary: toolSummary, ... }
      }
      // ... 其他消息类型处理
    }
  }
}
```

### 摘要生成

```typescript
function getToolSummaryText(counts: ToolCounts): string | undefined {
  const parts: string[] = []
  if (counts.searches > 0) {
    parts.push(`searched ${counts.searches} ${counts.searches === 1 ? 'pattern' : 'patterns'}`)
  }
  if (counts.reads > 0) {
    parts.push(`read ${counts.reads} ${counts.reads === 1 ? 'file' : 'files'}`)
  }
  if (counts.writes > 0) {
    parts.push(`wrote ${counts.writes} ${counts.writes === 1 ? 'file' : 'files'}`)
  }
  if (counts.commands > 0) {
    parts.push(`ran ${counts.commands} ${counts.commands === 1 ? 'command' : 'commands'}`)
  }
  if (counts.other > 0) {
    parts.push(`${counts.other} other ${counts.other === 1 ? 'tool' : 'tools'}`)
  }
  return capitalize(parts.join(', '))
}
```

### 消息类型映射

| 输入类型 | 输出类型 | 处理 |
|---------|---------|------|
| `assistant` (有文本) | `streamlined_text` | 提取文本内容 |
| `assistant` (纯工具) | `streamlined_tool_use_summary` | 生成工具摘要 |
| `result` | `result` | 保持不变 |
| `system`, `user`, `stream_event`, etc. | `null` | 过滤掉 |

## 关键代码路径与文件引用

### 本文件导出
- `createStreamlinedTransformer()`: 创建转换器函数
- `shouldIncludeInStreamlined(message)`: 预过滤辅助函数

### 依赖模块

| 模块 | 用途 |
|------|------|
| `src/entrypoints/agentSdkTypes.js` | `SDKAssistantMessage` 类型 |
| `src/entrypoints/sdk/controlTypes.js` | `StdoutMessage` 类型 |
| `src/tools/*/constants.js`, `prompt.js` | 工具名称常量 |
| `src/utils/messages.js` | `extractTextContent` |
| `src/utils/shell/shellToolUtils.js` | `SHELL_TOOL_NAMES` |
| `src/utils/stringUtils.js` | `capitalize` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/cli/print.ts` | 在 `runHeadless` 中，当 `feature('STREAMLINED_OUTPUT')` 和 `CLAUDE_CODE_STREAMLINED_OUTPUT` 启用时创建转换器 |

### 调用代码片段

```typescript
// src/cli/print.ts
const transformToStreamlined =
  feature('STREAMLINED_OUTPUT') &&
  isEnvTruthy(process.env.CLAUDE_CODE_STREAMLINED_OUTPUT) &&
  options.outputFormat === 'stream-json'
    ? createStreamlinedTransformer()
    : null

// 在消息循环中使用
for await (const message of runHeadlessStreaming(...)) {
  if (transformToStreamlined) {
    const transformed = transformToStreamlined(message)
    if (transformed) {
      await structuredIO.write(transformed)
    }
  }
  // ...
}
```

## 依赖与外部交互

### 与工具系统的集成
- 硬编码了多个工具名称常量
- 使用 `SHELL_TOOL_NAMES` 数组获取所有 Shell 相关工具
- 通过工具名称前缀匹配进行分类（`startsWith`）

### 与消息系统的集成
- 使用 `extractTextContent` 从消息块中提取纯文本
- 处理 `tool_use` 类型的内容块进行工具计数

### 构建标志控制
通过 `feature('STREAMLINED_OUTPUT')` 进行死代码消除，外部构建可能不包含此功能。

## 风险、边界与改进建议

### 潜在风险

1. **工具名称硬编码**: 新增工具需要更新此文件，容易遗漏
2. **前缀匹配歧义**: `startsWith` 匹配可能导致误判（如工具名包含关系）
3. **状态累积**: `cumulativeCounts` 在长时间运行中可能累积大量计数

### 边界情况

1. **空工具调用**: 纯工具消息但计数为零时返回 `null`
2. **混合内容**: 同时包含文本和工具调用的消息，文本优先
3. **长文本**: 未对文本长度进行截断，可能输出大量内容

### 改进建议

1. **工具分类配置化**: 从工具定义中读取分类，而非硬编码
```typescript
// 建议：工具定义中添加分类
interface ToolDefinition {
  name: string
  category: 'search' | 'read' | 'write' | 'command' | 'other'
}
```

2. **摘要阈值**: 添加最小阈值，避免频繁的小摘要
```typescript
const MIN_TOOLS_FOR_SUMMARY = 3
if (totalTools < MIN_TOOLS_FOR_SUMMARY) return null
```

3. **时间窗口**: 添加时间窗口，定期刷新摘要
```typescript
setInterval(() => {
  if (hasPendingSummary()) flushSummary()
}, 5000)
```

4. **更多元数据**: 在摘要中包含工具执行时间、结果状态等
```typescript
interface ToolCounts {
  // ...
  totalDuration: number
  successCount: number
  errorCount: number
}
```

5. **可配置过滤**: 允许用户配置保留/过滤哪些消息类型
```typescript
interface StreamlinedOptions {
  includeSystem?: boolean
  includeToolProgress?: boolean
  maxTextLength?: number
}
```

6. **测试覆盖**: 添加单元测试验证各种消息组合的转换结果
