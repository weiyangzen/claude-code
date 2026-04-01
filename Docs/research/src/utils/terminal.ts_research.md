# terminal.ts 研究文档

## 场景与职责

`terminal.ts` 是 Claude Code CLI 的终端内容渲染工具模块，专注于解决终端输出中的文本截断和格式化问题。它是 TUI（文本用户界面）渲染管道的关键组件，负责将可能很长的工具输出、命令结果等内容适配到有限的终端显示空间中。

主要使用场景：
1. **工具输出截断显示**：当 BashTool、PowerShellTool 等工具产生大量输出时，只显示前 3 行并提供展开提示
2. **终端宽度适配**：根据当前终端宽度自动换行，处理 ANSI 转义序列（颜色代码）
3. **性能优化**：避免对超大输出（如 64MB 二进制转储）进行完整的 O(n) 换行处理

## 功能点目的

### 1. 智能内容截断 (`renderTruncatedContent`)
- **问题**：工具输出可能非常长（如 `cat` 大文件、`find` 大量结果），完整显示会淹没对话历史
- **解决方案**：只显示前 3 行，剩余内容通过 "...+N lines (ctrl+o to expand)" 提示，用户可按 Ctrl+O 展开
- **特殊处理**：如果只有 1 行被隐藏，直接显示它而不是显示 "+1 line" 提示

### 2. ANSI 感知的文本换行 (`wrapText`)
- **问题**：标准字符串切片会破坏 ANSI 颜色代码，导致颜色泄漏或显示异常
- **解决方案**：使用 `sliceAnsi` 进行 ANSI 感知的切片，保留颜色格式

### 3. 截断预检测 (`isOutputLineTruncated`)
- **目的**：快速判断内容是否会被截断，用于 UI 决策（如是否显示展开按钮）
- **优化**：只计算换行符数量，不处理终端宽度换行，作为近似判断

## 具体技术实现

### 核心常量

```typescript
const MAX_LINES_TO_SHOW = 3           // 默认显示的最大行数
const PADDING_TO_PREVENT_OVERFLOW = 10 // 防止溢出的边距（考虑 MessageResponse 前缀 "  ⎿ " = 5 字符）
```

### 关键函数实现

#### `renderTruncatedContent`

```typescript
export function renderTruncatedContent(
  content: string,
  terminalWidth: number,
  suppressExpandHint = false,
): string
```

**算法流程**：
1. 计算有效换行宽度：`terminalWidth - PADDING_TO_PREVENT_OVERFLOW`
2. **预截断优化**：只处理前 `MAX_LINES_TO_SHOW * wrapWidth * 4` 个字符，避免对大输出进行完整处理
3. 调用 `wrapText` 进行 ANSI 感知的换行
4. 计算剩余行数，生成截断提示
5. 使用 chalk.dim 渲染提示文本

**性能优化细节**：
```typescript
// 只处理足够显示可见行的内容，避免对超大输出进行 O(n) 换行
const maxChars = MAX_LINES_TO_SHOW * wrapWidth * 4
const preTruncated = trimmedContent.length > maxChars
const contentForWrapping = preTruncated
  ? trimmedContent.slice(0, maxChars)
  : trimmedContent
```

#### `wrapText`

```typescript
function wrapText(
  text: string,
  wrapWidth: number,
): { aboveTheFold: string; remainingLines: number }
```

**算法流程**：
1. 按 `\n` 分割输入文本
2. 对每行计算可见宽度（使用 `stringWidth` 处理 Unicode 和全角字符）
3. 如果行宽超过 `wrapWidth`，使用 `sliceAnsi` 分段切片
4. 返回前 `MAX_LINES_TO_SHOW` 行和剩余行数

#### `isOutputLineTruncated`

```typescript
export function isOutputLineTruncated(content: string): boolean
```

**实现特点**：
- 使用 `indexOf` 循环查找换行符，避免创建临时数组
- 考虑 `trimEnd()` 行为：尾部换行符是终止符而非新行
- 快速路径：找到超过 `MAX_LINES_TO_SHOW` 个换行符即返回 true

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/components/shell/OutputLine.tsx` | 渲染 shell 命令输出 |
| `src/tools/BashTool/BashTool.tsx` | Bash 工具输出显示 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | PowerShell 工具输出显示 |
| `src/tools/MCPTool/MCPTool.ts` | MCP 工具输出显示 |
| `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` | MCP 资源读取结果显示 |
| `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts` | MCP 资源列表显示 |
| `src/ink/render-node-to-output.ts` | Ink 渲染管道 |
| `src/ink/components/App.tsx` | 应用主组件 |
| `src/ink/useTerminalNotification.ts` | 终端通知 |
| `src/interactiveHelpers.tsx` | 交互式辅助函数 |
| `src/ink/ink.tsx` | Ink 实例管理 |
| `src/screens/REPL.tsx` | REPL 主界面 |
| `src/components/ScrollKeybindingHandler.tsx` | 滚动快捷键处理 |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 输入框底部状态 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `chalk` | 终端颜色/样式 |
| `../components/CtrlOToExpand.js` | "(ctrl+o to expand)" 提示组件 |
| `../ink/stringWidth.js` | Unicode 字符串宽度计算 |
| `./sliceAnsi.js` | ANSI 感知的字符串切片 |

## 依赖与外部交互

### 与 sliceAnsi 的协作

`sliceAnsi` 是本模块的核心依赖，提供 ANSI 转义序列感知的字符串切片能力：
- 正确处理颜色代码、超链接（OSC 8）等 ANSI 序列
- 使用 `@alcalzone/ansi-tokenize` 进行精确的 token 解析
- 保持切片后的样式一致性

### 与 stringWidth 的协作

`stringWidth` 提供准确的字符显示宽度计算：
- 正确处理 CJK 字符（宽度 2）
- 正确处理 emoji（宽度 2）
- 正确处理组合字符（宽度 0）
- 优先使用 Bun.stringWidth，回退到 JavaScript 实现

### 与 CtrlOToExpand 的协作

提供统一的展开提示：
- React 组件版本：`CtrlOToExpand()` 用于 Ink 渲染
- 字符串版本：`ctrlOToExpand()` 用于纯文本场景

## 风险、边界与改进建议

### 潜在风险

1. **性能陷阱**
   - 虽然实现了预截断优化，但对于包含大量 ANSI 序列的文本，`sliceAnsi` 仍需完整 tokenize
   - 极端情况：单行超长（如 1MB 无换行）且含大量 ANSI 序列

2. **宽度计算不准确**
   - `stringWidth` 的近似可能导致某些特殊字符宽度计算错误
   - 某些终端对同一字符的宽度渲染可能不同

3. **换行宽度估算**
   - `PADDING_TO_PREVENT_OVERFLOW = 10` 是经验值，可能在某些布局中不准确
   - 未考虑缩进层级变化

### 边界条件

| 场景 | 行为 |
|-----|------|
| `terminalWidth < 10` | 强制使用最小宽度 10，防止负数或过小宽度 |
| 内容为空字符串 | 返回空字符串 |
| 内容只有空白字符 | 返回空字符串（经过 `trimEnd()`） |
| 恰好 4 行内容 | 显示全部 4 行（特殊处理：剩余 1 行时直接显示） |
| 恰好 5+ 行内容 | 显示前 3 行，提示 "+N lines" |
| 单行超长（需换行） | 按 `wrapWidth` 切片，可能产生多于 3 个视觉行 |

### 改进建议

1. **动态边距计算**
   ```typescript
   // 建议：根据实际布局层级计算边距
   function calculatePadding(indentLevel: number): number {
     return 5 + indentLevel * 2  // 基础 5 + 每层缩进 2
   }
   ```

2. **可配置的最大行数**
   - 当前硬编码 `MAX_LINES_TO_SHOW = 3`
   - 建议通过参数或配置暴露给用户

3. **更好的超长单行处理**
   ```typescript
   // 建议：检测单行视觉高度超过阈值时主动截断
   if (visibleWidth > wrapWidth * MAX_LINES_TO_SHOW) {
     // 主动截断并添加 "..." 提示
   }
   ```

4. **集成展开功能**
   - 当前仅显示提示，实际展开逻辑在调用方
   - 可考虑提供 `TruncatedContent` 组件封装展开/收起状态

5. **RTL 语言支持**
   - 当前未考虑从右到左语言的布局
   - 可能需要检测文本方向并调整截断逻辑

### 测试建议

应覆盖以下场景：
- ANSI 颜色代码不被破坏
- 超链接（OSC 8）不被破坏
- CJK 字符宽度正确计算
- Emoji 宽度正确计算
- 空内容、纯空白内容
- 恰好 3/4/5 行的边界情况
- 单行超长（超过终端宽度 3 倍）
- `suppressExpandHint = true` 时无提示
