# UserLocalCommandOutputMessage.tsx 研究文档

## 场景与职责

`UserLocalCommandOutputMessage` 是一个 React 组件，用于渲染本地命令的输出消息。该组件处理本地命令的标准输出(stdout)和标准错误(stderr)，支持 Markdown 渲染，并对 Cloud Launch 内容提供特殊格式化。

**核心职责：**
- 从消息内容中提取本地命令的 stdout 和 stderr
- 使用 Markdown 组件渲染输出内容
- 对 Cloud Launch 内容（以菱形符号开头）提供特殊渲染
- 在无内容时显示占位消息

## 功能点目的

1. **输出提取**：解析 `<local-command-stdout>` 和 `<local-command-stderr>` 标签
2. **内容格式化**：使用缩进和 Markdown 渲染输出
3. **Cloud Launch 支持**：识别并特殊格式化 Cloud Launch 输出（◇/◆ 符号开头）
4. **空内容处理**：无输出时显示 `(no content)` 占位符

## 具体技术实现

### 关键流程

```
输入: content (string)
  ↓
提取 stdout = extractTag(content, "local-command-stdout")
提取 stderr = extractTag(content, "local-command-stderr")
  ↓
如果 stdout 和 stderr 都为空：
  返回 <MessageResponse><Text dimColor>{NO_CONTENT_MESSAGE}</Text></MessageResponse>
  ↓
创建 lines 数组
如果 stdout 非空：
  lines.push(<IndentedContent key="stdout">{stdout.trim()}</IndentedContent>)
如果 stderr 非空：
  lines.push(<IndentedContent key="stderr">{stderr.trim()}</IndentedContent>)
  ↓
返回 lines 数组
```

### 数据结构

**Props 接口：**
```typescript
{
  content: string  // 包含 local-command 标签的 XML 内容
}
```

**内部组件 IndentedContent：**
- 检查内容是否以 `DIAMOND_OPEN` (◇) 或 `DIAMOND_FILLED` (◆) 开头
- 如果是：使用 `CloudLaunchContent` 特殊渲染
- 否则：使用标准缩进 + Markdown 渲染

### 关键代码路径

**文件位置：** `src/components/messages/UserLocalCommandOutputMessage.tsx`

**主组件逻辑：**
```typescript
export function UserLocalCommandOutputMessage({ content }: Props): React.ReactNode {
  const stdout = extractTag(content, 'local-command-stdout')
  const stderr = extractTag(content, 'local-command-stderr')

  if (!stdout && !stderr) {
    return (
      <MessageResponse>
        <Text dimColor>{NO_CONTENT_MESSAGE}</Text>
      </MessageResponse>
    )
  }

  const lines: React.ReactNode[] = []
  if (stdout?.trim()) {
    lines.push(<IndentedContent key="stdout">{stdout.trim()}</IndentedContent>)
  }
  if (stderr?.trim()) {
    lines.push(<IndentedContent key="stderr">{stderr.trim()}</IndentedContent>)
  }

  return lines
}
```

**IndentedContent 组件：**
```typescript
function IndentedContent({ children }: { children: string }) {
  if (children.startsWith(`${DIAMOND_OPEN} `) || children.startsWith(`${DIAMOND_FILLED} `)) {
    return <CloudLaunchContent>{children}</CloudLaunchContent>
  }

  return (
    <Box flexDirection="row">
      <Text dimColor>{'  ⏏  '}</Text>
      <Box flexDirection="column" flexGrow={1}>
        <Markdown>{children}</Markdown>
      </Box>
    </Box>
  )
}
```

**CloudLaunchContent 组件：**
```typescript
function CloudLaunchContent({ children }: { children: string }) {
  const diamond = children[0]
  
  // 解析 header 和 rest
  const nl = children.indexOf('\n')
  const header = nl === -1 ? children.slice(2) : children.slice(2, nl)
  const rest = nl === -1 ? '' : children.slice(nl + 1).trim()
  
  // 解析 label 和 suffix（以 " · " 分隔）
  const sep = header.indexOf(' · ')
  const label = sep === -1 ? header : header.slice(0, sep)
  const suffix = sep === -1 ? '' : header.slice(sep)

  return (
    <Box flexDirection="column">
      <Text>
        <Text color="background">{diamond} </Text>
        <Text bold>{label}</Text>
        {suffix && <Text dimColor>{suffix}</Text>}
      </Text>
      {rest && (
        <Box flexDirection="row">
          <Text dimColor>{'  ⏏  '}</Text>
          <Text dimColor>{rest}</Text>
        </Box>
      )}
    </Box>
  )
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| React | 'react' | UI 框架 |
| DIAMOND_FILLED, DIAMOND_OPEN | '../../constants/figures.js' | 菱形符号（◆ ◇） |
| NO_CONTENT_MESSAGE | '../../constants/messages.js' | 空内容提示文本 |
| Box, Text | '../../ink.js' | 终端 UI 组件 |
| extractTag | '../../utils/messages.js' | XML 标签提取工具 |
| Markdown | '../Markdown.js' | Markdown 渲染组件 |
| MessageResponse | '../MessageResponse.js' | 消息响应包装组件 |

### 相关常量

**figures.js：**
```typescript
export const DIAMOND_OPEN = '\u25c7'   // ◇ - running
export const DIAMOND_FILLED = '\u25c6' // ◆ - completed/failed
```

**messages.js：**
```typescript
export const NO_CONTENT_MESSAGE = '(no content)'
```

**xml.js：**
```typescript
export const LOCAL_COMMAND_STDOUT_TAG = 'local-command-stdout'
export const LOCAL_COMMAND_STDERR_TAG = 'local-command-stderr'
```

### Markdown 组件

**Markdown.tsx：**
- 使用 `marked` 库解析 Markdown
- 支持代码高亮（通过 cliHighlight）
- 支持表格渲染
- 有 token 缓存机制优化性能

### MessageResponse 组件

**MessageResponse.tsx：**
- 提供统一的响应消息样式
- 使用 `⎿` 符号作为响应前缀
- 支持嵌套检测避免重复前缀

## 风险、边界与改进建议

### 潜在风险

1. **Markdown 解析性能**：长输出可能导致解析耗时
2. **Cloud Launch 解析脆弱性**：依赖固定格式（菱形 + 空格 + 内容）
3. **stderr/stdout 顺序**：当前分开渲染，可能丢失原始顺序信息

### 边界情况

1. **stdout 和 stderr 都为空**：显示 `(no content)`
2. **只有 stderr**：正常渲染 stderr 内容
3. **只有 stdout**：正常渲染 stdout 内容
4. **内容以菱形开头但格式不符**：可能解析错误
5. **超长输出**：Markdown 组件有截断机制

### 改进建议

1. **保留输出顺序**：考虑交错显示 stdout 和 stderr 保持原始顺序
2. **流式输出**：支持增量渲染，提高大输出体验
3. **折叠长输出**：添加展开/折叠功能
4. **搜索功能**：在输出中搜索关键词
5. **复制功能**：一键复制命令输出
6. **错误高亮**：stderr 使用不同颜色或样式区分
7. **Cloud Launch 格式验证**：添加更严格的格式验证和错误回退

### 测试建议

1. 各种 stdout/stderr 组合测试
2. Cloud Launch 格式变体测试
3. Markdown 内容渲染测试
4. 超长输出性能测试
5. 特殊字符和编码测试
6. 空内容边界测试
