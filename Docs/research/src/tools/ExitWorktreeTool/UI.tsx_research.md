# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 为 `ExitWorktreeTool` 提供基于 React + Ink 的终端 UI 渲染能力。它负责在工具被调用时显示"正在退出 worktree"的进度提示，以及在工具执行完成后展示最终状态（保留或删除 worktree、分支信息、返回的原始目录）。

该文件是工具与终端用户之间的视觉交互层，遵循项目统一的 `renderToolUseMessage` / `renderToolResultMessage` 接口契约。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `renderToolUseMessage()` | 在工具执行期间向用户展示简洁的进度文案，告知当前正在退出 worktree |
| `renderToolResultMessage(output, ...)` | 在工具完成后展示结构化结果：操作类型（保留/删除）、分支名、返回的原始工作目录 |

## 具体技术实现

### 1. 导入依赖

```tsx
import * as React from 'react'
import { Box, Text } from '../../ink.js'
import type { ToolProgressData } from '../../Tool.js'
import type { ProgressMessage } from '../../types/message.js'
import type { ThemeName } from '../../utils/theme.js'
import type { Output } from './ExitWorktreeTool.js'
```

- 使用 `Box` 和 `Text` 组件构建垂直布局的终端输出
- 引入 `Output` 类型确保渲染函数与工具输出 schema 严格对齐
- `ProgressMessage` 和 `ThemeName` 为接口要求的形式参数，但当前实现未实际使用进度消息和主题

### 2. `renderToolUseMessage`

```tsx
export function renderToolUseMessage(): React.ReactNode {
  return 'Exiting worktree…'
}
```

- 返回纯字符串，Ink 会自动将其包装为 `<Text>`
- 使用省略号（…）表示动作正在进行中
- 与 `ExitWorktreeTool.ts` 中的 `userFacingName()`（返回 `'Exiting worktree'`）保持一致

### 3. `renderToolResultMessage`

```tsx
export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  _options: { theme: ThemeName },
): React.ReactNode {
  const actionLabel = output.action === 'keep' ? 'Kept worktree' : 'Removed worktree'
  return (
    <Box flexDirection="column">
      <Text>
        {actionLabel}
        {output.worktreeBranch ? (
          <>
            {' '}
            (branch <Text bold>{output.worktreeBranch}</Text>)
          </>
        ) : null}
      </Text>
      <Text dimColor>Returned to {output.originalCwd}</Text>
    </Box>
  )
}
```

- **布局**：使用 `flexDirection="column"` 将两行文本垂直排列
- **第一行**：根据 `output.action` 显示 `"Kept worktree"` 或 `"Removed worktree"`；若存在 `worktreeBranch`，则以粗体高亮分支名
- **第二行**：使用 `dimColor`（暗淡颜色）显示 `"Returned to {output.originalCwd}"`，降低视觉优先级
- **未使用的参数**：`_progressMessagesForMessage` 和 `_options` 以下划线前缀命名，表示当前实现不依赖进度消息和主题配置

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ExitWorktreeTool/UI.tsx` | 本文件，UI 渲染实现 |
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | 导入 `renderToolUseMessage` 和 `renderToolResultMessage`，并在 `buildTool` 中注册 |
| `src/ink.js` | 提供 `Box`、`Text` 等 Ink 组件（项目对 `ink` 的封装层） |
| `src/Tool.ts` | 定义 `ToolProgressData`、`renderToolResultMessage` 接口契约 |
| `src/types/message.ts` | 定义 `ProgressMessage` 类型 |
| `src/utils/theme.ts` | 定义 `ThemeName` 类型 |

## 依赖与外部交互

### 与 ExitWorktreeTool.ts 的绑定

在 `ExitWorktreeTool.ts` 中：

```ts
import { renderToolResultMessage, renderToolUseMessage } from './UI.js'

export const ExitWorktreeTool = buildTool({
  // ...
  renderToolUseMessage,
  renderToolResultMessage,
  // ...
})
```

- `buildTool` 将这两个渲染函数注册到工具对象上
- REPL / 交互式会话在收到 `tool_use` 和 `tool_result` 时会调用对应函数渲染终端界面
- 非交互式模式（如 SDK）不会调用这些函数

### 与 Ink 渲染管线的关系

- `renderToolUseMessage` 在工具调用开始、参数尚未完全流式到达时即可渲染（因此接口签名中 `input` 为 `Partial`）
- `renderToolResultMessage` 在工具执行完毕、拿到 `Output` 后渲染
- 两者均返回 `React.ReactNode`，由上层 `MessageRenderer` 或 `Transcript` 组件挂载到 Ink 树中

## 风险、边界与改进建议

### 风险与边界

1. **信息密度较低**
   - 当前 UI 仅展示操作类型、分支名和返回目录。对于 `remove` 操作，没有直观提示用户有多少文件/commit 被丢弃（虽然这些数字已存在于 `Output` 的 `discardedFiles` / `discardedCommits` 字段中）。

2. **tmux 信息未在 UI 中呈现**
   - `Output` 包含 `tmuxSessionName`，但 `renderToolResultMessage` 未使用它。`keep` 模式下用户无法在 UI 中直接看到 tmux 会话名（只能在 `message` 文本中看到），降低了可发现性。

3. **无错误状态 UI**
   - 该文件未提供 `renderToolUseErrorMessage` 或 `renderToolUseRejectedMessage`。当 `validateInput` 失败（如 errorCode 2/3）时，终端会回退到通用的错误/拒绝消息模板，用户体验一致但缺少 worktree 特有的上下文提示。

4. **Source Map 内联**
   - 文件末尾包含一行内联的 `//# sourceMappingURL=data:application/json;base64,...`。这是构建产物特征，对运行时无影响，但会增加文件体积。

### 改进建议

1. **在 UI 中展示丢弃统计**
   - 当 `output.action === 'remove'` 且 `discardedFiles > 0 || discardedCommits > 0` 时，增加一行红色或黄色警告文本，例如：
     ```tsx
     {output.discardedFiles || output.discardedCommits ? (
       <Text color="yellow">
         Discarded {output.discardedCommits ?? 0} commits and {output.discardedFiles ?? 0} uncommitted files
       </Text>
     ) : null}
     ```
     这样可以让用户更直观地意识到操作的破坏性后果。

2. **展示 tmux 会话状态**
   - 在 `keep` 模式下，若 `tmuxSessionName` 存在，可增加一行提示：
     ```tsx
     {output.tmuxSessionName ? (
       <Text dimColor>tmux session: {output.tmuxSessionName}</Text>
     ) : null}
     ```

3. **增加错误/拒绝状态的专属渲染（可选）**
   - 考虑实现 `renderToolUseErrorMessage`，在 `validateInput` 返回 errorCode 2（存在未提交变更）时，渲染一个更友好的提示框，引导用户确认或改用 `keep`。

4. **移除内联 source map**
   - 如果该文件是由构建流程生成的，建议在 CI/build 步骤中分离 source map 到独立文件，或配置构建器不内联 source map，以减小产物体积。
