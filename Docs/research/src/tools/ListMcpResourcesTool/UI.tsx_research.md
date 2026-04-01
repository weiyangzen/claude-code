# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 是 `ListMcpResourcesTool` 的**用户界面渲染模块**，负责在终端中展示工具调用过程和结果。它使用 React + Ink 框架实现命令行界面（CLI）的交互式渲染。

### 核心职责

1. **工具调用提示渲染**：显示当前正在执行的操作（`renderToolUseMessage`）
2. **结果展示渲染**：格式化并显示资源列表结果（`renderToolResultMessage`）
3. **空状态处理**：当没有资源时显示友好的提示信息
4. **终端适配**：根据终端宽度自动截断和格式化输出

### 使用场景

- REPL 模式下用户执行 `listMcpResources` 命令时的视觉反馈
- 模型调用工具时的实时状态展示
- 结果查看时的交互式输出（支持展开/折叠）

---

## 功能点目的

### 1. renderToolUseMessage - 工具调用提示

**输入**: 部分工具参数（`{ server?: string }`）
**输出**: 字符串或 React 节点

| 输入 | 输出 |
|------|------|
| `{ server: "myserver" }` | `List MCP resources from server "myserver"` |
| `{}` | `List all MCP resources` |

**目的**：在工具开始执行时向用户展示正在进行的操作。

### 2. renderToolResultMessage - 结果展示

**输入**: 
- `output`: 资源列表数据
- `_progressMessagesForMessage`: 进度消息（当前未使用）
- `options.verbose`: 是否详细模式

**输出**: React 节点

| 场景 | 渲染效果 |
|------|----------|
| 无资源 | `<MessageResponse>` 包裹的 `(No resources found)` 灰色文字 |
| 有资源 | `<OutputLine>` 格式化的 JSON 输出 |

---

## 具体技术实现

### 关键流程

#### 1. 工具调用提示渲染

```typescript
export function renderToolUseMessage(
  input: Partial<{ server?: string }>,
): React.ReactNode {
  return input.server
    ? `List MCP resources from server "${input.server}"`
    : `List all MCP resources`
}
```

**特点**：
- 简单字符串返回，无需 React 组件包装
- 支持部分输入（工具参数流式传输时可能不完整）

#### 2. 结果消息渲染流程

```typescript
export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  { verbose }: { verbose: boolean },
): React.ReactNode {
  // 1. 空状态处理
  if (!output || output.length === 0) {
    return (
      <MessageResponse height={1}>
        <Text dimColor>(No resources found)</Text>
      </MessageResponse>
    )
  }

  // 2. JSON 格式化（带缩进）
  const formattedOutput = jsonStringify(output, null, 2)
  
  // 3. 使用 OutputLine 渲染（支持截断/展开）
  return <OutputLine content={formattedOutput} verbose={verbose} />
}
```

### 数据结构

#### 输入数据类型

```typescript
// 来自 ListMcpResourcesTool.ts
type Output = Array<{
  uri: string
  name: string
  mimeType?: string
  description?: string
  server: string
}>
```

#### 示例输出格式

```json
[
  {
    "uri": "file:///docs/api.md",
    "name": "API Documentation",
    "mimeType": "text/markdown",
    "description": "API reference documentation",
    "server": "docs-server"
  },
  {
    "uri": "https://api.example.com/data",
    "name": "Example Data",
    "server": "api-server"
  }
]
```

### 依赖组件详解

#### MessageResponse

**路径**: `src/components/MessageResponse.tsx`

**功能**: 统一的消息响应容器，提供：
- 左侧缩进前缀（`  ⎿ `）
- 防止嵌套渲染的 Context 机制
- 可选的固定高度

**关键代码**:
```typescript
export function MessageResponse({ children, height }: Props): React.ReactNode {
  const isMessageResponse = useContext(MessageResponseContext)
  if (isMessageResponse) {
    return children  // 避免嵌套
  }
  return (
    <MessageResponseProvider>
      <Box flexDirection="row" height={height} overflowY="hidden">
        <NoSelect fromLeftEdge={true} flexShrink={0}>
          <Text dimColor>{'  '}⎿ &nbsp;</Text>
        </NoSelect>
        <Box flexShrink={1} flexGrow={1}>{children}</Box>
      </Box>
    </MessageResponseProvider>
  )
}
```

#### OutputLine

**路径**: `src/components/shell/OutputLine.tsx`

**功能**: 智能输出渲染器，提供：
- JSON 格式化（自动检测并美化）
- URL 超链接化
- 终端宽度适配截断
- 详细模式切换

**关键特性**:
```typescript
export function OutputLine({
  content,
  verbose,
  isError,
  isWarning,
  linkifyUrls,
}) {
  const { columns } = useTerminalSize()
  const expandShellOutput = useExpandShellOutput()
  const shouldShowFull = verbose || expandShellOutput

  // 1. 尝试 JSON 格式化
  let formatted = tryJsonFormatContent(content)
  
  // 2. URL 超链接化
  if (linkifyUrls) {
    formatted = linkifyUrlsInText(formatted)
  }

  // 3. 根据模式决定显示方式
  const displayContent = shouldShowFull
    ? stripUnderlineAnsi(formatted)
    : stripUnderlineAnsi(renderTruncatedContent(formatted, columns, inVirtualList))

  return (
    <MessageResponse>
      <Text color={color}><Ansi>{displayContent}</Ansi></Text>
    </MessageResponse>
  )
}
```

**截断逻辑**（`src/utils/terminal.ts`）:
```typescript
export function renderTruncatedContent(
  content: string,
  terminalWidth: number,
  suppressExpandHint = false,
): string {
  // 最多显示 3 行
  const MAX_LINES_TO_SHOW = 3
  // ... 截断逻辑
  return [
    aboveTheFold,
    estimatedRemaining > 0
      ? chalk.dim(`… +${estimatedRemaining} lines${ctrlOToExpand()}`)
      : '',
  ].filter(Boolean).join('\n')
}
```

#### Text 组件

**路径**: `src/ink.ts` 导出

**功能**: Ink 框架的文本组件，支持：
- `dimColor`: 暗淡颜色（用于次要信息）
- `color`: 指定颜色（error, warning 等）
- ANSI 代码处理

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `src/components/MessageResponse.tsx` | `MessageResponse` | 消息响应容器 |
| `src/components/shell/OutputLine.tsx` | `OutputLine` | 智能输出行渲染 |
| `src/ink.ts` | `Text` | 文本组件 |
| `src/Tool.ts` | `ToolProgressData` | 工具进度数据类型 |
| `src/types/message.ts` | `ProgressMessage` | 进度消息类型 |
| `src/utils/slowOperations.ts` | `jsonStringify` | 性能监控 JSON 序列化 |
| `./ListMcpResourcesTool.ts` | `Output` | 输出数据类型 |

### 依赖关系图

```
UI.tsx
├── MessageResponse.tsx
│   ├── ink.ts (Box, NoSelect, Text)
│   └── design-system/Ratchet.js
├── OutputLine.tsx
│   ├── useTerminalSize() (hooks)
│   ├── Ansi, Text (ink.ts)
│   ├── createHyperlink() (utils)
│   ├── jsonParse, jsonStringify (utils/slowOperations)
│   ├── renderTruncatedContent() (utils/terminal)
│   └── MessageResponse.tsx
└── jsonStringify (utils/slowOperations)
```

### Ink 框架集成

Ink 是 React 的命令行渲染器，类似于 React DOM 但输出到终端：

```typescript
// src/ink.ts 关键导出
export { default as Box } from './components/design-system/ThemedBox.js'
export { default as Text } from './components/design-system/ThemedText.js'
export { Ansi } from './ink/Ansi.js'
```

---

## 依赖与外部交互

### React 版本

使用 React Compiler 优化（从 `_c` 函数可以看出）：

```typescript
import { c as _c } from "react/compiler-runtime"
```

### 主题系统

通过 `src/ink.ts` 的 `ThemeProvider` 自动注入：

```typescript
function withTheme(node: ReactNode): ReactNode {
  return createElement(ThemeProvider, null, node)
}

export async function render(node: ReactNode, options?: RenderOptions): Promise<Instance> {
  return inkRender(withTheme(node), options)
}
```

### 终端交互

| 功能 | 实现方式 |
|------|---------|
| 终端尺寸检测 | `useTerminalSize()` hook |
| 输出展开 | `useExpandShellOutput()` Context |
| 虚拟列表优化 | `InVirtualListContext` |

---

## 风险、边界与改进建议

### 已知风险

1. **JSON 格式化性能**
   - 风险：大量资源（>1000 条）时 JSON 格式化可能卡顿
   - 缓解：`jsonStringify` 有性能监控，超过阈值会记录慢操作日志
   - 代码：`using _ = slowLogging\`JSON.stringify(${value})\``

2. **终端宽度变化**
   - 风险：终端调整大小时，截断计算可能过时
   - 缓解：`useTerminalSize()` 会响应尺寸变化重新渲染

3. **内存泄漏**
   - 风险：大型 JSON 字符串在内存中保留
   - 缓解：React Compiler 自动优化，但大数据量仍需注意

### 边界情况

| 场景 | 行为 |
|------|------|
| `output` 为 `null` | 显示 `(No resources found)` |
| `output` 为空数组 `[]` | 显示 `(No resources found)` |
| `output` 包含循环引用 | `jsonStringify` 可能抛出错误 |
| 终端宽度 < 20 | `renderTruncatedContent` 使用最小宽度 10 |
| 单条资源 | 正常渲染，无截断提示 |
| 资源包含特殊字符 | `Ansi` 组件正确处理 ANSI 转义序列 |

### 改进建议

1. **添加资源计数显示**
   ```typescript
   // 建议：在结果前显示资源数量
   return (
     <>
       <Text dimColor>Found {output.length} resources:</Text>
       <OutputLine content={formattedOutput} verbose={verbose} />
     </>
   )
   ```

2. **支持表格视图**
   - 当前使用 JSON 格式，对于大量资源可读性较差
   - 可考虑添加表格视图选项（类似 `ls -la`）

3. **添加服务器分组**
   ```typescript
   // 建议：按服务器分组显示
   const grouped = groupBy(output, 'server')
   // 渲染为：
   // server-a:
   //   - resource1
   //   - resource2
   // server-b:
   //   - resource3
   ```

4. **优化空状态提示**
   - 当前仅显示 `(No resources found)`
   - 可添加帮助信息：
     - 提示如何配置 MCP 服务器
     - 提示使用 `ReadMcpResourceTool` 的前提条件

5. **支持资源过滤预览**
   - 在 `renderToolUseMessage` 中显示更多参数信息
   - 如支持通配符过滤，可显示过滤条件

6. **添加加载状态**
   - 当前无进度指示，大量资源时用户可能以为卡死
   - 可考虑使用 `renderToolUseProgressMessage` 添加进度条

### 测试建议

1. **单元测试**
   - 测试空数组渲染
   - 测试单条/多条资源渲染
   - 测试 JSON 格式化正确性

2. **集成测试**
   - 测试终端尺寸变化时的响应
   - 测试展开/折叠功能
   - 测试颜色主题切换

3. **性能测试**
   - 测试 1000+ 资源时的渲染性能
   - 测试超大资源描述（>10KB）时的表现
