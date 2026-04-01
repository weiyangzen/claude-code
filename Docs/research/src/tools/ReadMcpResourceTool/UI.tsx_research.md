# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 是 `ReadMcpResourceTool` 的**终端用户界面渲染层**，负责在 Claude Code 的 REPL（交互式命令行）环境中呈现：

1. **工具使用提示（Tool Use Message）**：当模型决定调用 `ReadMcpResourceTool` 时，向用户显示一条简洁的文本说明，告知正在从哪个服务器读取哪个资源。
2. **工具结果展示（Tool Result Message）**：当 MCP 服务器返回资源内容后，将结构化输出以可读形式渲染到终端。

该文件与 `ReadMcpResourceTool.ts` 解耦，遵循 Claude Code 中“工具逻辑 + UI 渲染”分离的惯例。所有导出函数均为纯函数，接收输入/输出数据并返回 `React.ReactNode`，由上层消息组件（如 `UserToolResultMessage`、`AssistantToolUseMessage`）负责挂载到 ink 渲染树。

## 功能点目的

### `renderToolUseMessage`
- **目的**：在工具开始执行前给用户一个即时反馈。
- **行为**：若 `input.uri` 和 `input.server` 已提供，返回形如 `Read resource "<uri>" from server "<server>"` 的字符串；否则返回 `null`（不渲染任何内容）。

### `userFacingName`
- **目的**：提供一个面向用户的短名称，用于紧凑视图、权限提示、分类器等场景。
- **返回值**：固定字符串 `readMcpResource`。

### `renderToolResultMessage`
- **目的**：将 `ReadMcpResourceTool` 的输出（`Output` 类型）渲染为终端可读的 React 节点。
- **行为**：
  - 空结果或无内容时：显示 `(No content)` 占位文本。
  - 有内容时：将输出格式化为缩进 JSON（`jsonStringify(output, null, 2)`），并通过 `OutputLine` 组件渲染，支持自动截断、JSON 美化、URL 超链接化以及 `verbose` 模式展开。

## 具体技术实现

### 组件与导入

```tsx
import * as React from 'react'
import type { z } from 'zod/v4'
import { MessageResponse } from '../../components/MessageResponse.js'
import { OutputLine } from '../../components/shell/OutputLine.js'
import { Box, Text } from '../../ink.js'
import type { ToolProgressData } from '../../Tool.js'
import type { ProgressMessage } from '../../types/message.js'
import { jsonStringify } from '../../utils/slowOperations.js'
import type { inputSchema, Output } from './ReadMcpResourceTool.js'
```

### 关键渲染逻辑

#### 1. `renderToolUseMessage`
```tsx
export function renderToolUseMessage(
  input: Partial<z.infer<ReturnType<typeof inputSchema>>>,
): React.ReactNode {
  if (!input.uri || !input.server) {
    return null
  }
  return `Read resource "${input.uri}" from server "${input.server}"`
}
```
- 参数类型使用 `Partial<z.infer<...>>`，因为在流式解析阶段，工具参数可能尚未完整接收。
- 返回字符串而非 React 元素，这是 Claude Code 中 `renderToolUseMessage` 的惯用写法——上层组件可直接将其作为纯文本处理，也可嵌入更复杂的布局。

#### 2. `renderToolResultMessage`
```tsx
export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  { verbose }: { verbose: boolean },
): React.ReactNode {
  if (!output || !output.contents || output.contents.length === 0) {
    return (
      <Box justifyContent="space-between" overflowX="hidden" width="100%">
        <MessageResponse height={1}>
          <Text dimColor>(No content)</Text>
        </MessageResponse>
      </Box>
    )
  }

  // eslint-disable-next-line no-restricted-syntax -- human-facing UI, not tool_result
  const formattedOutput = jsonStringify(output, null, 2)
  return <OutputLine content={formattedOutput} verbose={verbose} />
}
```

- **空状态处理**：使用 `<MessageResponse height={1}>` 包裹 `<Text dimColor>(No content)</Text>`，保持与 Shell 工具、文件读取工具一致的视觉风格（左侧带 `⎿` 前缀）。
- **JSON 格式化**：显式使用 `jsonStringify(output, null, 2)` 将数组/对象结构展开为缩进 JSON，提升终端可读性。
  - 注释 `eslint-disable-next-line no-restricted-syntax` 说明此处是“面向人类的 UI”，不适用通常禁止在工具结果中直接 `JSON.stringify` 的 lint 规则（该规则主要防止模型侧收到未处理的原始 JSON）。
- **OutputLine**：
  - 接收 `content`（字符串）和 `verbose`（布尔）。
  - 内部实现（`src/components/shell/OutputLine.tsx`）会：
    1. 尝试对每行进行 JSON 格式化（`tryJsonFormatContent`）。
    2. 若 `linkifyUrls` 启用，将文本中的 URL 替换为终端超链接（`createHyperlink`）。
    3. 根据 `verbose` 与 `useExpandShellOutput()` 上下文决定是否显示完整内容，或调用 `renderTruncatedContent` 截断到前 3 行并附加 `… +N lines (ctrl+o to expand)` 提示。

### 进度消息

`_progressMessagesForMessage` 参数在当前实现中**未被使用**。这是为了符合 `Tool.renderToolResultMessage` 的通用签名而保留的占位符。`ReadMcpResourceTool` 本身不支持流式进度上报（与 `BashTool`、`WebSearchTool` 等不同），因此该数组始终为空。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ReadMcpResourceTool/UI.tsx` | 本文件，UI 渲染实现 |
| `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` | 工具主逻辑，提供 `inputSchema` 与 `Output` 类型 |
| `src/components/MessageResponse.tsx` | 消息响应容器，提供左侧 `⎿` 前缀与嵌套去重逻辑 |
| `src/components/shell/OutputLine.tsx` | Shell 输出行组件，负责 JSON 美化、URL 链接化、截断/展开 |
| `src/ink.ts` | Claude Code 对 `ink` 的封装导出，提供 `Box`、`Text`、`Ansi` 等终端 React 组件 |
| `src/Tool.ts` | `ToolProgressData` 类型定义 |
| `src/utils/slowOperations.ts` | `jsonStringify`（带慢操作检测的 JSON 序列化） |
| `src/utils/terminal.ts` | `renderTruncatedContent`、`isOutputLineTruncated`（截断算法） |

## 依赖与外部交互

### React / ink 生态

- **React**：使用函数组件与 `React.ReactNode` 返回类型。未使用 Hooks（纯渲染函数）。
- **ink**：通过 `src/ink.ts` 间接引入 `Box` 和 `Text`。`Box` 用于布局（`justifyContent="space-between"`、`overflowX="hidden"`、`width="100%"`），`Text` 用于带样式文本（`dimColor`）。

### 上层消息系统

`renderToolUseMessage` 与 `renderToolResultMessage` 不会直接被 ink `render()` 调用，而是由以下上层组件在构建消息树时调用：

- `src/components/messages/AssistantToolUseMessage.tsx`：渲染模型发起的 `tool_use` 块时调用 `renderToolUseMessage`。
- `src/components/messages/UserToolResultMessage/UserToolSuccessMessage.tsx`：渲染工具成功结果时调用 `renderToolResultMessage`。

这些上层组件负责传递 `verbose`、`theme`、`tools` 等上下文参数，并将返回的 React 节点嵌入到完整的消息流中。

### 类型系统

- `ProgressMessage<ToolProgressData>` 来自 `src/types/message.js`（构建时生成/解析的模块），代表工具执行过程中产生的进度通知。由于 `ReadMcpResourceTool` 不产生进度消息，该类型在此仅用于签名兼容。

## 风险、边界与改进建议

### 风险与边界

1. **大 JSON 输出的渲染性能**：
   - `OutputLine` 内部对整段内容做 `tryJsonFormatContent`，若 `ReadMcpResourceTool` 返回的 `contents` 数组包含极长的 `text`（如数万行日志），`JSON.stringify` 与终端截断算法可能产生显著 CPU 开销。
   - `OutputLine` 的 `MAX_JSON_FORMAT_LENGTH = 10_000` 会在内容超过 10KB 时跳过 JSON 格式化，但 `ReadMcpResourceTool` 的 `maxResultSizeChars = 100_000`，意味着大内容仍会被完整传入 `OutputLine`。

2. **二进制路径的文本展示缺失**：
   - 当资源为二进制时，`ReadMcpResourceTool.ts` 会在 `text` 字段注入 `Binary content (... ) saved to <path>` 的说明。`UI.tsx` 仅做通用 JSON 展示，不会为二进制资源提供特殊的高亮或图标提示，用户体验与纯文本资源无异。

3. **无进度/加载态**：
   - 读取远程 MCP 资源可能因网络或服务器处理而耗时数秒，但 `ReadMcpResourceTool` 未实现 `renderToolUseProgressMessage`，用户在这段时间内看不到任何“读取中”的动态反馈，只有调用提示和最终结果之间的空白。

4. **Source Map 内嵌**：
   - 文件末尾包含一行极长的 `//# sourceMappingURL=data:application/json;base64,...`，这是编译产物特征。若该文件为源码（`.tsx`），内嵌 source map 会增加文件体积并在某些工具链中造成噪音；若为生成文件，则修改源码后需重新生成。

### 改进建议

1. **增加二进制资源专属 UI 提示**：
   - 在 `renderToolResultMessage` 中检测 `blobSavedTo` 字段，若存在可渲染一个带文件路径和 MIME 类型标签的专用组件（类似 `FilePathLink` 或带图标的文本块），让用户一眼识别出这是二进制资源。

2. **引入轻量进度渲染**：
   - 实现 `renderToolUseProgressMessage`（即使只是显示 `Reading resource from <server>...` 的 spinner），提升长耗时操作的可感知性。

3. **内容长度分级处理**：
   - 在 `renderToolResultMessage` 层对 `formattedOutput.length` 做前置判断，超过某个阈值时直接提示“内容过长，请使用 Read 工具查看”或截断提示，避免将超大字符串完全交给 `OutputLine`。

4. **统一 Source Map 策略**：
   - 若 `.tsx` 文件为手写源码，建议移除内嵌 source map，改为外部 `.map` 文件或构建流程自动处理，保持源码整洁。

5. **支持资源元数据折叠**：
   - 当 `contents` 数组包含多条资源时，当前会一次性展开所有 JSON。可考虑借鉴 `GroupedToolUseContent` 的折叠模式，对多资源结果进行分组/折叠，减少终端信息噪音。
