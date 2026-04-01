# UI.tsx 研究文档

## 场景与职责

UI.tsx 是 LSPTool 的用户界面层，负责渲染 LSP 工具的使用消息、结果消息和错误消息。它使用 React 和 Ink（终端 UI 库）提供丰富的命令行交互体验。

**主要职责：**
- 渲染工具使用时的实时反馈（renderToolUseMessage）
- 渲染工具结果的可折叠/展开视图（renderToolResultMessage）
- 渲染工具错误消息（renderToolUseErrorMessage）
- 提供操作特定的标签和描述
- 支持 verbose 和非 verbose 两种显示模式

**使用场景：**
- 用户调用 LSP 工具时显示操作类型和目标
- 显示搜索结果统计（如 "Found 15 references across 3 files"）
- 提供 Ctrl+O 展开查看详细结果的交互
- 错误发生时显示友好的错误提示

---

## 功能点目的

### 1. 操作标签映射（OPERATION_LABELS）

为每种 LSP 操作定义单数/复数标签，用于结果统计显示：

```typescript
const OPERATION_LABELS: Record<Input['operation'], {
  singular: string
  plural: string
  special?: string  // 特殊显示文本
}> = {
  goToDefinition: { singular: 'definition', plural: 'definitions' },
  findReferences: { singular: 'reference', plural: 'references' },
  hover: { singular: 'hover info', plural: 'hover info', special: 'available' },
  // ... 其他操作
}
```

### 2. 结果摘要组件（LSPResultSummary）

核心 UI 组件，提供：
- **统计信息**：显示结果数量和涉及文件数
- **可折叠视图**：非 verbose 模式下显示摘要，支持 Ctrl+O 展开
- **详细视图**：verbose 模式下显示完整结果内容

### 3. 工具使用消息（renderToolUseMessage）

显示工具调用时的上下文信息：
- 对于位置相关操作（goToDefinition, findReferences, hover, goToImplementation）：
  - 尝试提取并显示目标符号名称
  - 显示文件路径
- 对于其他操作：显示操作类型和文件路径

### 4. 错误消息渲染

- 非 verbose 模式下显示简化错误信息
- verbose 模式下显示详细错误

---

## 具体技术实现

### 关键组件：LSPResultSummary

```typescript
function LSPResultSummary({
  operation,
  resultCount,
  fileCount,
  content,
  verbose
}) {
  // 1. 根据操作获取标签配置
  const labelConfig = OPERATION_LABELS[operation]
  const countLabel = resultCount === 1 ? labelConfig.singular : labelConfig.plural
  
  // 2. 构建主文本
  const primaryText = operation === 'hover' && resultCount > 0 && labelConfig.special
    ? <Text>Hover info {labelConfig.special}</Text>
    : <Text>Found <Text bold={true}>{resultCount} </Text>{countLabel}</Text>
  
  // 3. 构建次要文本（多文件时显示）
  const secondaryText = fileCount > 1 
    ? <Text> across <Text bold={true}>{fileCount} </Text>files</Text>
    : null
  
  // 4. 根据 verbose 模式渲染不同视图
  if (verbose) {
    // 详细视图：显示完整内容
    return <Box flexDirection="column">
      <Box flexDirection="row"><Text>{indent}{primaryText}{secondaryText}</Text></Box>
      <Box marginLeft={5}><Text>{content}</Text></Box>
    </Box>
  } else {
    // 简洁视图：显示摘要 + Ctrl+O 提示
    return <MessageResponse height={1}>
      <Text>{primaryText}{secondaryText} {resultCount > 0 && <CtrlOToExpand />}</Text>
    </MessageResponse>
  }
}
```

### 符号提取显示

```typescript
export function renderToolUseMessage(input: Partial<Input>, { verbose }) {
  // 位置相关操作：尝试提取符号名称
  if ((input.operation === 'goToDefinition' || 
       input.operation === 'findReferences' || 
       input.operation === 'hover' || 
       input.operation === 'goToImplementation') && 
      input.filePath && 
      input.line !== undefined && 
      input.character !== undefined) {
    
    // 转换 1-based 到 0-based 进行文件读取
    const symbol = getSymbolAtPosition(
      input.filePath, 
      input.line - 1, 
      input.character - 1
    )
    
    const displayPath = verbose ? input.filePath : getDisplayPath(input.filePath)
    
    if (symbol) {
      return `operation: "${input.operation}", symbol: "${symbol}", in: "${displayPath}"`
    } else {
      return `operation: "${input.operation}", file: "${displayPath}", position: ${input.line}:${input.character}`
    }
  }
  
  // 其他操作：简化显示
  // ...
}
```

### 错误消息处理

```typescript
export function renderToolUseErrorMessage(result, { verbose }) {
  if (!verbose && typeof result === 'string' && extractTag(result, 'tool_use_error')) {
    // 非 verbose 模式：简化错误显示
    return <MessageResponse>
      <Text color="error">LSP operation failed</Text>
    </MessageResponse>
  }
  // 使用默认错误消息组件
  return <FallbackToolUseErrorMessage result={result} verbose={verbose} />
}
```

### 结果消息渲染

```typescript
export function renderToolResultMessage(output, _progressMessages, { verbose }) {
  // 有统计信息时使用可折叠视图
  if (output.resultCount !== undefined && output.fileCount !== undefined) {
    return <LSPResultSummary 
      operation={output.operation}
      resultCount={output.resultCount}
      fileCount={output.fileCount}
      content={output.result}
      verbose={verbose}
    />
  }
  
  // 无统计信息（错误情况）直接显示结果
  return <MessageResponse>
    <Text>{output.result}</Text>
  </MessageResponse>
}
```

---

## 关键代码路径与文件引用

### 依赖关系

```
UI.tsx
  ├── @anthropic-ai/sdk/resources/index.mjs  # ToolResultBlockParam 类型
  ├── react                                  # React 核心
  ├── ../../components/CtrlOToExpand.js      # Ctrl+O 展开提示
  ├── ../../components/FallbackToolUseErrorMessage.js  # 默认错误组件
  ├── ../../components/MessageResponse.js    # 消息响应容器
  ├── ../../ink.js                           # Ink UI 组件（Box, Text）
  ├── ../../utils/file.js                    # getDisplayPath
  ├── ../../utils/messages.js                # extractTag
  ├── ./LSPTool.js                           # Input, Output 类型
  └── ./symbolContext.js                     # getSymbolAtPosition
```

### 类型定义

```typescript
import type { ToolResultBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'
import type { Input, Output } from './LSPTool.js'
```

### 导出函数

| 函数 | 用途 | 调用时机 |
|------|------|----------|
| `userFacingName()` | 返回显示名称 "LSP" | 工具列表显示 |
| `renderToolUseMessage()` | 渲染工具使用消息 | 工具被调用时 |
| `renderToolUseErrorMessage()` | 渲染错误消息 | 工具执行出错时 |
| `renderToolResultMessage()` | 渲染结果消息 | 工具执行完成时 |

---

## 依赖与外部交互

### React Compiler 优化

代码使用了 React Compiler（通过 `_c` 函数），自动进行记忆化优化：

```typescript
const $ = _c(24)  // 创建 24 个缓存槽位

// 条件性计算和缓存
if ($[0] !== operation) {
  $[0] = operation
  $[1] = t1
} else {
  t1 = $[1]  // 复用缓存值
}
```

### Ink UI 组件

使用 Ink 提供的终端 UI 组件：
- **Box**: 布局容器，支持 flex 布局
- **Text**: 文本渲染，支持颜色、粗体等样式

### 符号提取集成

通过 `getSymbolAtPosition` 从文件读取符号名称：

```typescript
import { getSymbolAtPosition } from './symbolContext.js'

// 在 renderToolUseMessage 中使用
const symbol = getSymbolAtPosition(filePath, line - 1, character - 1)
```

注意：这是同步调用，因为 React 渲染函数必须是同步的。

### 文件路径显示

```typescript
import { getDisplayPath } from '../../utils/file.js'

// verbose 模式显示完整路径，否则显示简化路径
const displayPath = verbose ? input.filePath : getDisplayPath(input.filePath)
```

---

## 风险、边界与改进建议

### 已知风险

1. **同步文件读取阻塞渲染**
   - 风险：`getSymbolAtPosition` 是同步调用，可能阻塞 UI
   - 缓解：symbolContext.ts 中只读取前 64KB，且有 try-catch 保护

2. **React Compiler 依赖**
   - 风险：编译器生成的代码难以手动维护
   - 缓解：保持简单组件结构，避免复杂逻辑

3. **verbose 模式内容溢出**
   - 风险：大结果在 verbose 模式下可能导致终端滚动
   - 现状：依赖外层 MessageResponse 的滚动处理

### 边界情况

| 场景 | 行为 |
|------|------|
| 无操作类型 | renderToolUseMessage 返回 null |
| 无文件路径 | 显示 operation 但不显示文件信息 |
| 符号提取失败 | 回退到显示 position: line:char |
| 无结果 | 显示 "Found 0 xxx" |
| 单文件结果 | 不显示 "across X files" |
| 无统计信息 | 直接显示 result 文本 |

### 改进建议

1. **异步符号提取**
   - 当前：同步读取文件可能阻塞
   - 建议：考虑使用异步 API 或缓存常用文件内容

2. **结果截断显示**
   - 当前：verbose 模式显示完整内容
   - 建议：添加智能截断，显示前 N 行 + "...and X more"

3. **语法高亮**
   - 当前：纯文本显示
   - 建议：集成代码高亮，提升可读性

4. **交互增强**
   - 当前：Ctrl+O 展开/折叠
   - 建议：添加直接跳转到位置的快捷键

5. **加载状态**
   - 当前：无明确加载指示
   - 建议：添加 spinner 或进度指示器

6. **错误详情展开**
   - 当前：错误信息较简略
   - 建议：允许展开查看完整错误堆栈

### 性能考虑

- React Compiler 自动优化重渲染
- 缓存标签配置和文本计算结果
- 避免在渲染中执行昂贵操作（文件读取已限制在 64KB）

### 可访问性

- 使用颜色区分错误状态（`color="error"`）
- 粗体强调关键数字
- 清晰的视觉层次（缩进、间距）
