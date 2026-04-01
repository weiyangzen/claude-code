# UI.tsx 研究文档

> 文件路径：`src/tools/TaskStopTool/UI.tsx`  
> 研究时间：2026-04-01  
> 执行器：kimi / k2p5

---

## 一、场景与职责

`UI.tsx` 为 `TaskStopTool` 提供 **终端 UI 渲染函数**，负责在交互式（REPL/TUI）模式下向用户展示工具调用结果。该文件遵循 Claude Code 工具 UI 的通用模式：

- `renderToolUseMessage`：工具**开始执行时**的即时渲染。
- `renderToolResultMessage`：工具**执行完毕后**的结果渲染。

由于 TaskStop 是一个轻量管理操作（只需一个 task_id），其工具使用阶段没有需要展示的复杂输入摘要，因此 `renderToolUseMessage` 被设计为空渲染；结果阶段则展示被停止任务的命令/描述，并附加 "· stopped" 状态后缀。

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| `renderToolUseMessage` | 返回空字符串 `''`，表示工具开始执行时不在消息流中额外渲染任何内容（TaskStop 的输入本身不值得即时展示）。 |
| `truncateCommand` | 对命令文本做截断：限制最多 2 行、160 个显示宽度字符，避免长 Bash 命令或 Agent 描述撑爆消息行。 |
| `renderToolResultMessage` | 在 `MessageResponse` 组件中渲染被停止任务的命令 + "· stopped" 后缀；`verbose` 模式下展示完整命令，非 verbose 模式下展示截断版本。 |
| Ant 环境隐藏 | 存在 `"external" === 'ant'` 分支，意图在 ant 构建中隐藏结果渲染（当前为死代码，见下文）。 |

---

## 三、具体技术实现

### 3.1 关键常量

```ts
const MAX_COMMAND_DISPLAY_LINES = 2
const MAX_COMMAND_DISPLAY_CHARS = 160
```

- **行数限制**：若命令包含换行符且超过 2 行，只保留前 2 行。
- **宽度限制**：使用 `stringWidth()`（终端列宽感知）测量后，若超过 160 列，则调用 `truncateToWidthNoEllipsis` 截断。

### 3.2 截断逻辑 `truncateCommand`

```ts
function truncateCommand(command: string): string {
  const lines = command.split('\n')
  let truncated = command
  if (lines.length > MAX_COMMAND_DISPLAY_LINES) {
    truncated = lines.slice(0, MAX_COMMAND_DISPLAY_LINES).join('\n')
  }
  if (stringWidth(truncated) > MAX_COMMAND_DISPLAY_CHARS) {
    truncated = truncateToWidthNoEllipsis(truncated, MAX_COMMAND_DISPLAY_CHARS)
  }
  return truncated.trim()
}
```

注意：这里先按行截断，再按宽度截断；宽度截断**不追加省略号**（由调用方在 suffix 中统一处理）。

### 3.3 结果渲染 `renderToolResultMessage`

```tsx
export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: unknown[],
  { verbose }: { verbose: boolean }
): React.ReactNode {
  if ("external" === 'ant') {   // ← 死代码
    return null
  }

  const rawCommand = output.command ?? ''
  const command = verbose ? rawCommand : truncateCommand(rawCommand)
  const suffix = command !== rawCommand ? '… · stopped' : ' · stopped'

  return (
    <MessageResponse>
      <Text>
        {command}{suffix}
      </Text>
    </MessageResponse>
  )
}
```

- **verbose 模式**：直接展示 `output.command` 原值。
- **非 verbose 模式**：展示截断后的命令；若发生截断，suffix 为 `… · stopped`，否则为 ` · stopped`。
- **组件结构**：`MessageResponse` 提供左侧 `⎿` 缩进装饰，`Text` 来自 `src/ink.js`（经过 ThemeProvider 包装的主题感知文本组件）。

---

## 四、关键代码路径与文件引用

| 引用关系 | 文件路径 | 说明 |
|---------|----------|------|
| **类型依赖** | `src/tools/TaskStopTool/TaskStopTool.ts` | 导入 `Output` 类型（工具输出结构）。 |
| **组件依赖** | `src/components/MessageResponse.tsx` | `MessageResponse`：标准消息响应容器，带左侧缩进与嵌套去重逻辑。 |
| **工具函数** | `src/ink/stringWidth.ts` | `stringWidth`：精确计算终端显示宽度（支持 emoji、CJK、ANSI）。 |
| **组件依赖** | `src/ink.ts` | `Text`：主题感知的 Ink 文本组件。 |
| **工具函数** | `src/utils/format.ts` | `truncateToWidthNoEllipsis`：按终端宽度截断字符串，不追加省略号，按 grapheme 边界分割。 |
| **被调用方** | `src/tools/TaskStopTool/TaskStopTool.ts` | `renderToolUseMessage` 与 `renderToolResultMessage` 被注册到 `buildTool` 的对应字段上。 |

---

## 五、依赖与外部交互

### 5.1 渲染时依赖

- **Ink / React**：`UI.tsx` 是一个标准的 React 组件文件，使用 JSX 语法，最终由 `src/ink.ts` 的 `render()` 或 REPL 消息列表渲染器消费。
- **ThemeProvider**：`src/ink.ts` 在 `render()` 时自动包裹 `ThemeProvider`，因此 `Text` 会自动响应当前主题（明暗、颜色配置）。

### 5.2 与 `TaskStopTool.ts` 的协作

```ts
// TaskStopTool.ts
import { renderToolResultMessage, renderToolUseMessage } from './UI.js'

buildTool({
  // ...
  renderToolUseMessage,
  renderToolResultMessage,
  // ...
})
```

`buildTool` 会将这两个函数挂载到 `Tool` 对象上；当 `toolExecution.ts` 执行完工具调用后，REPL 的消息渲染层会根据消息类型调用对应的渲染函数。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

1. **死代码分支**：
   ```tsx
   if ("external" === 'ant') {
     return null
   }
   ```
   这是一个**编译期/构建期常量判断被硬编码为字符串比较**的残留代码。`"external"` 永远不等于 `'ant'`，因此该分支永远不会进入。其原始意图可能是通过某种构建替换（如 bundler define）在 ant 构建中隐藏 TaskStop 的 UI，但当前实现已失效，且会误导阅读者。

2. **`_progressMessagesForMessage` 未使用**：函数签名中包含 `_progressMessagesForMessage: unknown[]`，但函数体完全未使用。这是 `renderToolResultMessage` 接口的统一签名要求，部分工具（如 BashTool）会利用进度消息做动画渲染；TaskStop 作为瞬时操作没有进度消息，因此用下划线前缀忽略。类型为 `unknown[]` 而非 `ProgressMessage[]`，说明该文件在类型层面也没有做更严格的约束。

3. **无 source map 剥离**：文件末尾包含一个巨大的 inline source map（`//# sourceMappingURL=data:application/json;charset=utf-8;base64,...`）。这在生产 bundle 中会增加文件体积，建议构建流程配置为外置或移除 source map。

### 6.2 边界行为

- **空 command**：当 `output.command` 为 `undefined` 时，会回退到空字符串 `''`，渲染结果仅为 ` · stopped`。这通常发生在旧 transcript 恢复时（`command` 字段可选）。
- **超长单行命令**：若命令无换行但宽度超过 160，`truncateToWidthNoEllipsis` 会将其截断到 160 列，suffix 显示为 `… · stopped`。
- **多行命令**：若命令有 10 行，先截断为前 2 行；若这 2 行仍超过 160 列，再执行宽度截断。不会在中途换行处断开 grapheme cluster。

### 6.3 改进建议

1. **清理死代码**：移除 `"external" === 'ant'` 分支，或将其替换为真正基于 `process.env.USER_TYPE === 'ant'` 的条件渲染（如果产品确实需要区分）。当前代码属于明显的技术债务。
2. **提取通用截断逻辑**：`truncateCommand` 的模式（先按行、再按宽度）与 BashTool、PowerShellTool 等可能类似。可考虑在 `src/utils/format.ts` 或新建 `src/utils/truncate.ts` 中提供 `truncateMultilineCommand(command, {maxLines, maxWidth})` 通用函数，减少复制粘贴。
3. **source map 外置**：将 inline base64 source map 改为外置 `.map` 文件或构建时删除，降低 bundle 体积。
4. **补充 UI 快照测试**：当前仓库未找到 TaskStopTool 的测试。建议补充一个最小化的 Ink 渲染测试，验证：
   - `renderToolUseMessage` 返回空
   - `renderToolResultMessage` 在 verbose 与非 verbose 模式下输出文本正确
   - 截断逻辑对多行、超长命令的处理符合预期
