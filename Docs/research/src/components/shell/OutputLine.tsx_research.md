# OutputLine.tsx 研究文档

## 场景与职责

`OutputLine.tsx` 是终端 shell 输出渲染的核心组件，负责将原始字符串（stdout/stderr/工具输出）格式化为可在终端 UI 中呈现的 Ink 节点。其职责包括：

- JSON 美化打印（带精度保护）
- URL 自动超链接化（OSC 8）
- 基于终端宽度的智能截断（3 行折叠 + `ctrl+o` 展开提示）
- 下划线 ANSI 转义序列泄漏修复
- 错误/警告颜色区分

该组件被 `BashTool`、`PowerShellTool`、`MCPTool`、`ListMcpResourcesTool`、`ReadMcpResourceTool` 等多个工具的 UI 层广泛复用。

## 功能点目的

### 1. JSON 格式化与精度保护
`tryFormatJson(line)` 尝试对单行 JSON 进行 `parse + stringify(null, 2)` 美化。为避免大整数超过 `Number.MAX_SAFE_INTEGER` 导致精度丢失，它会将原始行与 round-trip 后的结果做归一化比较（去除空白、将 `\/` 替换为 `/`）。若不一致，则放弃格式化，返回原行。

`tryJsonFormatContent(content)` 对多行内容逐行应用上述逻辑，但设有 `MAX_JSON_FORMAT_LENGTH = 10_000` 的防护上限，避免对大体积内容做无意义的解析尝试。

### 2. URL 超链接化
`linkifyUrlsInText(content)` 使用正则 `URL_IN_JSON = /https?:\/\/[^\s"'<>\\]+/g` 匹配文本中的 URL，并通过 `createHyperlink(url)` 生成 OSC 8 终端超链接。该功能在 MCPTool 等场景下通过 `linkifyUrls` prop 显式开启。

### 3. 截断与展开控制
组件通过 `useTerminalSize()` 获取终端列数，结合 `useExpandShellOutput()` 判断当前是否处于“最新 bash 输出自动展开”上下文，以及 `InVirtualListContext` 判断是否在虚拟列表中，最终决定是否调用 `renderTruncatedContent()` 进行截断。

截断规则：
- `verbose || expandShellOutput` → 显示完整内容
- 否则 → 调用 `renderTruncatedContent(formatted, columns, inVirtualList)`，最多显示 3 行，剩余行数提示 `… +N lines (ctrl+o to expand)`；若处于虚拟列表则隐藏 `ctrl+o` 提示。

### 4. 下划线 ANSI 泄漏修复
`stripUnderlineAnsi(content)` 专门剥离 ANSI 下划线转义序列（`\u001b\[...4...m`）。注释说明下划线序列容易“泄漏”到后续行，而直接 `stripAnsi` 又会丢失所有颜色格式引发用户抱怨，因此采取针对性剥离策略。

## 具体技术实现

### 关键流程

```
接收 props: content, verbose, isError, isWarning, linkifyUrls
    │
    ▼
formatted = tryJsonFormatContent(content)
    │
    ├─ linkifyUrls === true → formatted = linkifyUrlsInText(formatted)
    │
    ▼
shouldShowFull = verbose || useExpandShellOutput()
    │
    ├─ shouldShowFull → display = stripUnderlineAnsi(formatted)
    └─ 否则           → display = stripUnderlineAnsi(
                           renderTruncatedContent(formatted, columns, inVirtualList))
    │
    ▼
color = isError ? "error" : isWarning ? "warning" : undefined
    │
    ▼
返回 <MessageResponse><Text color={color}><Ansi>{display}</Ansi></Text></MessageResponse>
```

### 数据结构

- `MAX_JSON_FORMAT_LENGTH = 10_000`：JSON 格式化的长度安全阀。
- `URL_IN_JSON = /https?:\/\/[^\s"'<>\\]+/g`：保守的 URL 匹配正则，避免匹配 JSON 结构中的引号、尖括号等。

### 依赖工具函数

| 函数 | 来源 | 作用 |
|------|------|------|
| `useTerminalSize()` | `src/hooks/useTerminalSize.ts` | 读取 `TerminalSizeContext` 获取终端尺寸 |
| `renderTruncatedContent()` | `src/utils/terminal.ts` | ANSI 感知的 3 行截断与提示 |
| `jsonParse` / `jsonStringify` | `src/utils/slowOperations.ts` | 带慢操作监控的 JSON 方法 |
| `createHyperlink()` | `src/utils/hyperlink.ts` | OSC 8 超链接生成，含终端能力回退 |
| `stripUnderlineAnsi()` | 本文件导出 | 下划线 ANSI 剥离 |

## 关键代码路径与文件引用

### 本文件内部

| 行号 | 内容 |
|------|------|
| 12-31 | `tryFormatJson`：JSON 精度保护格式化 |
| 32-39 | `tryJsonFormatContent`：多行 JSON 格式化入口 |
| 43-46 | `linkifyUrlsInText`：URL 超链接化 |
| 47-104 | `OutputLine` 组件主体 |
| 113-117 | `stripUnderlineAnsi`：下划线 ANSI 剥离 |

### 上游调用方

| 路径 | 用途 |
|------|------|
| `src/tools/BashTool/BashToolResultMessage.tsx:6` | 渲染 stdout/stderr |
| `src/tools/PowerShellTool/UI.tsx:6` | 渲染 stdout/stderr |
| `src/tools/MCPTool/UI.tsx:8` | 渲染 MCP 文本输出（含 `linkifyUrls`） |
| `src/tools/ListMcpResourcesTool/UI.tsx:3` | 渲染列表结果 |
| `src/tools/ReadMcpResourceTool/UI.tsx:4` | 渲染读取结果 |
| `src/components/FallbackToolUseErrorMessage.tsx:4` | 复用 `stripUnderlineAnsi` |

### 下游依赖

| 路径 | 用途 |
|------|------|
| `src/utils/terminal.ts` | `renderTruncatedContent`、`isOutputLineTruncated` |
| `src/utils/slowOperations.ts` | `jsonParse`、`jsonStringify` |
| `src/utils/hyperlink.ts` | `createHyperlink` |
| `src/components/MessageResponse.tsx` | 统一消息前缀包装 |
| `src/components/messageActions.tsx` | `InVirtualListContext` |
| `src/components/shell/ExpandShellOutputContext.tsx` | `useExpandShellOutput` |

## 依赖与外部交互

- **Ink 组件**：`Ansi`、`Text`（来自 `src/ink.ts`）用于终端渲染。
- **React Compiler**：产物中使用了 `_c(11)` 等 memo cache，说明该组件已被编译器优化。
- **虚拟列表**：通过 `InVirtualListContext` 感知自身是否处于 `Messages.tsx` 的虚拟滚动列表中，以决定是否显示 `ctrl+o` 展开提示（虚拟列表无终端 scrollback，提示无意义）。

## 风险、边界与改进建议

1. **JSON 格式化精度检测的局限性**  
   `tryFormatJson` 的归一化比较仅处理 `\/` 和空白差异，对于其他语义等价但文本不同的 JSON（如 `1.0` vs `1`、对象 key 顺序差异）可能误判为“精度丢失”而放弃格式化。当前策略是保守且安全的。

2. **URL 正则的误匹配风险**  
   `URL_IN_JSON` 排除了引号、空白、尖括号，但在非 JSON 场景（如 Markdown、HTML）中仍可能截断 URL（例如遇到 `)` 或 `]` 时不会停止）。目前该正则主要服务于 JSON 输出场景，命名也体现了这一点。

3. **截断性能与大内容**  
   `renderTruncatedContent` 内部已做性能优化：对超大内容先按 `maxChars = MAX_LINES_TO_SHOW * wrapWidth * 4` 做预截断，避免对 64MB 二进制输出做全量换行处理。但 `tryJsonFormatContent` 的 `split('\n').map(...)` 仍会对 10KB 以下内容逐行解析，若内容频繁更新（如进度条）可能累积开销。

4. **`stripUnderlineAnsi` 的维护成本**  
   该函数基于正则硬编码下划线 ANSI 模式，若未来遇到其他泄漏的 ANSI 属性（如斜体、闪烁），可能需要继续追加白名单/黑名单。更根本的修复应定位到 Ink 的 ANSI 解析/重置逻辑，但当前方案作为 workaround 足够实用。

5. **测试覆盖**  
   仓库中未检索到针对 `OutputLine` 的单元测试。建议补充以下测试：
   - JSON 精度保护场景（大整数、含 `\/` 的 URL）
   - `verbose` / `useExpandShellOutput` / `inVirtualList` 三种截断控制组合
   - `linkifyUrls` 对含 URL 文本的转换
   - `stripUnderlineAnsi` 对常见 ANSI 序列的保留/剥离行为
