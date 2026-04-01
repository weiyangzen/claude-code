# UI.tsx 研究文档

## 场景与职责

UI.tsx 是 MCP 工具的**用户界面渲染模块**，负责将 MCP 工具的输入、进度和结果以可视化的方式呈现给用户。它使用 React 和 Ink（终端 UI 库）构建交互式命令行界面。

### 核心职责

1. **工具使用消息渲染** (`renderToolUseMessage`)：显示工具调用时的参数摘要
2. **进度消息渲染** (`renderToolUseProgressMessage`)：显示工具执行进度（进度条或文本状态）
3. **结果消息渲染** (`renderToolResultMessage`)：显示工具执行结果，支持多种输出格式
4. **JSON 内容智能解析**：提供 `tryFlattenJson` 和 `tryUnwrapTextPayload` 优化 JSON 输出显示
5. **Slack 消息特殊处理**：`trySlackSendCompact` 为 Slack 发送操作提供简洁显示

### 使用场景

- 当模型调用 MCP 工具时，显示工具名称和参数
- 当 MCP 工具执行时间较长时，显示进度条或处理状态
- 当 MCP 工具返回结果时，智能格式化显示（支持图片、文本、JSON）
- 当结果过大时，显示警告提示

---

## 功能点目的

### 1. 工具使用消息渲染 (`renderToolUseMessage`)

```typescript
export function renderToolUseMessage(
  input: z.infer<ReturnType<typeof inputSchema>>,
  { verbose }: { verbose: boolean }
): React.ReactNode
```

**功能**：
- 将工具输入参数格式化为 `key: value` 字符串列表
- 非详细模式下截断过长的值（超过 80 字符）
- 空输入时返回空字符串

**关键常数**：
```typescript
const MAX_INPUT_VALUE_CHARS = 80  // 非详细模式最大输入值长度
```

### 2. 进度消息渲染 (`renderToolUseProgressMessage`)

```typescript
export function renderToolUseProgressMessage(
  progressMessagesForMessage: ProgressMessage<MCPProgress>[]
): React.ReactNode
```

**功能**：
- 显示工具执行状态（"Running…"、"Processing…"）
- 支持进度条显示（当提供 progress/total 时）
- 显示进度百分比和可选的进度消息

**UI 组件**：
- 使用 `ProgressBar` 组件显示进度条
- 使用 `MessageResponse` 包装消息
- 使用 Ink 的 `Box` 和 `Text` 进行布局

### 3. 结果消息渲染 (`renderToolResultMessage`)

```typescript
export function renderToolResultMessage(
  output: string | MCPToolResult,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  { verbose, input }: { verbose: boolean; input?: unknown }
): React.ReactNode
```

**功能**：
- 特殊处理 Slack 发送结果（简洁显示）
- 大结果警告（超过 10,000 tokens）
- 支持多种内容类型：
  - 图片（显示 `[Image]` 占位符）
  - 文本（使用 `MCPTextOutput` 或 `OutputLine`）
  - 数组内容（逐个渲染）

**关键常数**：
```typescript
const MCP_OUTPUT_WARNING_THRESHOLD_TOKENS = 10_000  // 大输出警告阈值
```

### 4. MCP 文本输出组件 (`MCPTextOutput`)

**三种渲染策略**（按优先级）：

1. **文本解包** (`tryUnwrapTextPayload`)：
   - 检测 JSON 中是否包含主导文本字段（多行或长文本）
   - 解包并单独显示主体内容，其他字段作为元数据
   - 适用于 Slack 消息等场景

2. **扁平 JSON** (`tryFlattenJson`)：
   - 将小型扁平对象渲染为对齐的键值对列表
   - 最多 12 个键，最多 5,000 字符

3. **默认输出** (`OutputLine`)：
   - 使用标准输出行组件，支持语法高亮和截断

### 5. Slack 发送结果特殊处理 (`trySlackSendCompact`)

```typescript
export function trySlackSendCompact(
  output: string | MCPToolResult,
  input: unknown
): { channel: string; url: string } | null
```

**功能**：
- 检测 Slack 发送消息的结果格式
- 提取消息链接和频道信息
- 返回简洁的显示格式："Sent a message to #channel"

**匹配模式**：
```typescript
const SLACK_ARCHIVES_RE = /^https:\/\/[a-z0-9-]+\.slack\.com\/archives\/([A-Z0-9]+)\/p\d+$/
```

---

## 具体技术实现

### 关键流程

#### 1. JSON 解析流程 (`parseJsonEntries`)

```typescript
function parseJsonEntries(
  content: string,
  { maxChars, maxKeys }: { maxChars: number; maxKeys: number }
): [string, unknown][] | null
```

**步骤**：
1. 检查内容长度和开头字符（必须是 `{`）
2. 使用 `jsonParse` 解析 JSON
3. 验证结果为对象且键数量在范围内
4. 返回键值对数组

#### 2. 扁平 JSON 检测 (`tryFlattenJson`)

**条件**：
- 内容长度 ≤ 5,000 字符 (`MAX_FLAT_JSON_CHARS`)
- 键数量 ≤ 12 (`MAX_FLAT_JSON_KEYS`)
- 所有值必须是标量或小型嵌套对象（≤120 字符）

**输出格式**：
```
key1: value1
key2: value2
...
```

#### 3. 文本解包检测 (`tryUnwrapTextPayload`)

**条件**：
- 内容长度 ≤ 200,000 字符 (`MAX_JSON_PARSE_CHARS`)
- 键数量 ≤ 4
- 恰好有一个主导字符串字段（长度 > 200 或包含换行且长度 > 50）
- 其他字段为小型标量（字符串 ≤150 字符）

**输出结构**：
```typescript
{
  body: string,      // 主导文本内容
  extras: [string, string][]  // 其他字段作为元数据
}
```

### 数据结构

#### MCPProgress 类型（来自 types/tools.js）

```typescript
type MCPProgress = {
  type: 'mcp_progress'
  status: 'started' | 'progress' | 'completed' | 'failed'
  serverName: string
  toolName: string
  progress?: number
  total?: number
  progressMessage?: string
  elapsedTimeMs?: number
}
```

#### MCPToolResult 类型

```typescript
type MCPToolResult = string | ContentBlockParam[] | undefined

// ContentBlockParam 可以是：
// - { type: 'text', text: string }
// - { type: 'image', source: { type: 'base64', data: string, media_type: string } }
```

---

## 关键代码路径与文件引用

### 直接依赖

```
UI.tsx
├── react/compiler-runtime            # React 编译器运行时
├── bun:bundle (feature)              # 功能标志
├── figures                           # 终端符号（如警告图标）
├── react                             # React 核心
├── zod/v4 (type)                     # 类型引用
├── ../../components/design-system/ProgressBar.js   # 进度条组件
├── ../../components/MessageResponse.js             # 消息响应容器
├── ../../components/shell/OutputLine.js            # 输出行组件
├── ../../ink/stringWidth.js          # 字符串宽度计算
├── ../../ink.js (Ansi, Box, Text)    # Ink UI 组件
├── ../../Tool.js (type)              # ToolProgressData 类型
├── ../../types/message.js (type)     # ProgressMessage 类型
├── ../../types/tools.js (type)       # MCPProgress 类型
├── ../../utils/format.js             # formatNumber
├── ../../utils/hyperlink.js          # createHyperlink
├── ../../utils/mcpValidation.js      # getContentSizeEstimate, MCPToolResult
├── ../../utils/slowOperations.js     # jsonParse, jsonStringify
└── ./MCPTool.js (type)               # inputSchema 类型
```

### 被引用位置

```
src/tools/MCPTool/MCPTool.ts
├── import { renderToolResultMessage, renderToolUseMessage, renderToolUseProgressMessage } from './UI.js'
```

---

## 依赖与外部交互

### 核心依赖模块

| 模块 | 用途 |
|------|------|
| `ProgressBar` | 显示进度条，支持自定义宽度和颜色 |
| `MessageResponse` | 统一的消息响应容器，带前缀样式 |
| `OutputLine` | 标准输出行，支持语法高亮和截断 |
| `stringWidth` | 计算字符串显示宽度（处理 Unicode） |
| `createHyperlink` | 创建终端可点击链接 |
| `getContentSizeEstimate` | 估算内容 token 数量 |
| `jsonParse/jsonStringify` | 安全的 JSON 序列化/反序列化 |

### 功能标志

```typescript
feature('MCP_RICH_OUTPUT')  // 控制是否启用富文本输出
```

### 常数配置

| 常数 | 值 | 说明 |
|------|-----|------|
| `MCP_OUTPUT_WARNING_THRESHOLD_TOKENS` | 10,000 | 大输出警告阈值 |
| `MAX_INPUT_VALUE_CHARS` | 80 | 输入值截断长度 |
| `MAX_FLAT_JSON_KEYS` | 12 | 扁平 JSON 最大键数 |
| `MAX_FLAT_JSON_CHARS` | 5,000 | 扁平 JSON 最大字符数 |
| `MAX_JSON_PARSE_CHARS` | 200,000 | JSON 解析最大字符数 |
| `UNWRAP_MIN_STRING_LEN` | 200 | 文本解包最小长度 |

---

## 风险、边界与改进建议

### 当前风险

1. **JSON 解析性能**：
   - 大内容（200KB）解析可能阻塞主线程
   - 风险：`MAX_JSON_PARSE_CHARS` 限制可能不足以防止性能问题
   - 缓解：使用 `jsonParse`（来自 slowOperations）可能是异步的

2. **正则表达式安全性**：
   - `SLACK_ARCHIVES_RE` 用于匹配 Slack URL
   - 风险：特定输入可能导致 ReDoS
   - 缓解：正则相对简单，风险较低

3. **内存使用**：
   - 大型 MCP 结果可能占用大量内存
   - 风险：图片内容以 base64 存储，内存开销大
   - 缓解：有 `MAX_JSON_PARSE_CHARS` 限制

### 边界情况

1. **空内容处理**：
   - `renderToolResultMessage` 对 `null`/`undefined` 输出显示 "(No content)"

2. **图片内容**：
   - 仅显示 `[Image]` 占位符，不显示实际图片
   - 图片数据通过 `MCPToolResult` 数组传递

3. **进度消息缺失**：
   - 当 `progressMessagesForMessage` 为空或最后一个消息无数据时，显示 "Running…"

4. **total 为 0 或未定义**：
   - 显示文本进度 "Processing… {progress}" 而非进度条

### 改进建议

1. **性能优化**：
   - 考虑对超大 JSON 使用流式解析
   - 添加 Web Worker 支持处理大内容解析

2. **功能增强**：
   - 支持更多 MCP 内容类型（如音频、视频）
   - 添加图片预览（在支持的终端中）
   - 支持结果折叠/展开动画

3. **错误处理**：
   - 添加 JSON 解析失败的降级显示
   - 改进无效内容的错误提示

4. **可访问性**：
   - 为进度条添加屏幕阅读器支持
   - 提供更多文本替代方案

5. **配置扩展**：
   - 允许用户自定义截断阈值
   - 支持主题自定义（颜色、符号）

---

## 附录：渲染示例

### 示例 1：工具使用消息

```
channel: "#general", message: "Hello world"
```

### 示例 2：进度消息（带进度条）

```
Uploading file...
[████████████░░░░░░░░] 60%
```

### 示例 3：扁平 JSON 显示

```
name:     "John Doe"
age:      30
email:    john@example.com
```

### 示例 4：解包文本显示

```
messages: 3 users 
-----------------
Hello everyone!
This is a multi-line message.
```

### 示例 5：Slack 发送简洁显示

```
Sent a message to #general
```

### 示例 6：大结果警告

```
⚠ Large MCP response (~15,000 tokens), this can fill up context quickly
[truncated content...]
```
