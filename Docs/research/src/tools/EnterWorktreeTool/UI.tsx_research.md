# UI.tsx 研究文档

## 场景与职责

UI.tsx 是 EnterWorktreeTool 的用户界面渲染模块，负责在终端中显示工具使用状态和结果。它使用 Ink（React for CLI）框架提供富文本、带样式的终端输出。

**核心场景：**
1. 用户调用 EnterWorktreeTool 时显示 "Creating worktree…" 进度提示
2. 工具执行完成后显示切换到的 worktree 分支和路径信息

**职责边界：**
- 仅负责视觉呈现，不包含业务逻辑
- 提供两种渲染模式：工具使用中和工具完成后
- 遵循项目统一的 UI 主题系统

## 功能点目的

### 1. renderToolUseMessage - 工具使用中提示
- **目的**：在工具执行期间向用户显示进度状态
- **显示内容**：简单的文本 "Creating worktree…"
- **设计意图**：提供即时反馈，让用户知道系统正在响应请求

### 2. renderToolResultMessage - 工具结果展示
- **目的**：在工具成功执行后显示详细结果
- **显示内容**：
  - 切换到的分支名称（加粗显示）
  - Worktree 的完整路径（暗淡颜色）
- **布局**：垂直排列的 Box 容器，包含两个 Text 组件

## 具体技术实现

### 技术栈
- **Ink**: React for CLI 框架，提供组件化终端 UI
- **React**: JSX 组件定义

### 组件实现

```typescript
// 工具使用中消息
export function renderToolUseMessage(): React.ReactNode {
  return 'Creating worktree…'
}
```

```typescript
// 工具结果消息
export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  _options: { theme: ThemeName }
): React.ReactNode {
  return (
    <Box flexDirection="column">
      <Text>
        Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
      </Text>
      <Text dimColor>{output.worktreePath}</Text>
    </Box>
  )
}
```

### 样式设计

| 元素 | 样式 | 用途 |
|-----|------|------|
| 分支名称 | `bold` | 突出显示关键信息 |
| 路径 | `dimColor` | 次要信息，降低视觉干扰 |
| 容器 | `flexDirection="column"` | 垂直布局，信息层次分明 |

### 类型定义

```typescript
// 从 EnterWorktreeTool.ts 导入的输出类型
import type { Output } from './EnterWorktreeTool.js'

// Output 结构
{
  worktreePath: string
  worktreeBranch?: string
  message: string
}
```

## 关键代码路径与文件引用

### 当前文件
- `/src/tools/EnterWorktreeTool/UI.tsx` - UI 渲染实现

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `/src/ink.js` | Ink 框架导入（Box, Text 组件） |
| `/src/Tool.js` | ToolProgressData 类型定义 |
| `/src/types/message.ts` | ProgressMessage 类型定义 |
| `/src/utils/theme.ts` | ThemeName 类型定义 |
| `./EnterWorktreeTool.ts` | Output 类型定义 |

### 被引用位置

UI.tsx 的导出函数被以下位置使用：
- `/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`
  - `renderToolUseMessage` → `buildTool.renderToolUseMessage`
  - `renderToolResultMessage` → `buildTool.renderToolResultMessage`

## 依赖与外部交互

### 渲染流程

```
工具调用
  ↓
EnterWorktreeTool.call()
  ↓
返回 ToolResult<Output>
  ↓
框架调用 renderToolResultMessage(output, progressMessages, options)
  ↓
Ink 渲染到终端
```

### 主题集成

- 接受 `ThemeName` 参数，支持亮色/暗色主题适配
- 当前实现未直接使用主题变量，但保留扩展能力

### 进度消息处理

- `_progressMessagesForMessage` 参数保留但未使用
- 设计意图：未来可支持显示详细的进度更新（如文件复制进度）

## 风险、边界与改进建议

### 已知限制

1. **无进度详情**
   - 当前仅显示静态 "Creating worktree…" 文本
   - 不显示实际进度（如 git 命令执行阶段、文件复制进度等）

2. **错误状态处理**
   - UI.tsx 仅处理成功状态
   - 错误处理由框架的 `renderToolUseErrorMessage` 默认实现处理

3. **分支名缺失处理**
   - `output.worktreeBranch` 是可选字段
   - 当前实现直接渲染，如果为 undefined 会显示空值
   - 建议添加条件渲染

### 边界情况

1. **长路径显示**
   - 终端宽度有限时，长路径可能被截断
   - 建议考虑路径截断或换行处理

2. **特殊字符**
   - 分支名或路径包含特殊字符时，Ink 的 Text 组件能正确处理
   - 无需额外的 HTML 转义

### 改进建议

1. **增强进度显示**
   ```typescript
   // 建议：根据进度消息显示不同阶段
   export function renderToolUseProgressMessage(
     progressMessages: ProgressMessage<ToolProgressData>[]
   ): React.ReactNode {
     const latestStage = progressMessages.at(-1)?.data?.stage
     return `Creating worktree${latestStage ? `: ${latestStage}` : '…'}`
   }
   ```

2. **条件渲染分支名**
   ```typescript
   // 建议：处理 worktreeBranch 可能缺失的情况
   {output.worktreeBranch && (
     <Text>
       Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
     </Text>
   )}
   ```

3. **路径截断**
   ```typescript
   // 建议：长路径智能截断
   import { truncatePath } from '../../utils/path.js'
   
   <Text dimColor>{truncatePath(output.worktreePath, 60)}</Text>
   ```

4. **添加图标指示**
   ```typescript
   // 建议：使用图标增强视觉识别
   <Text>✓ Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text></Text>
   ```

5. **主题适配**
   ```typescript
   // 建议：使用主题颜色
   <Text color={_options.theme === 'dark' ? 'gray' : 'darkgray'}>
     {output.worktreePath}
   </Text>
   ```

### 测试建议

1. **快照测试**
   - 对 renderToolResultMessage 输出进行快照测试
   - 覆盖不同分支名和路径组合

2. **边界测试**
   - 测试超长路径渲染
   - 测试空分支名处理
   - 测试特殊字符路径

3. **主题测试**
   - 验证在亮色/暗色主题下的可读性
