# UserToolRejectMessage.tsx 深度研究文档

## 场景与职责

`UserToolRejectMessage` 是 Claude Code 终端 UI 中用于渲染**工具使用被拒绝**状态的专用组件。与 `UserToolErrorMessage` 不同，此组件专门处理工具使用请求被用户拒绝的场景，支持工具特定的自定义拒绝 UI。

### 核心职责
1. **工具特定拒绝渲染**：调用工具的 `renderToolUseRejectedMessage` 方法显示自定义拒绝 UI
2. **输入验证**：验证工具输入数据符合工具的 `inputSchema`
3. **回退处理**：当工具未定义拒绝渲染方法或验证失败时，显示回退 UI
4. **终端尺寸适配**：响应终端尺寸变化，适配不同宽度的显示

## 功能点目的

### 1. 工具特定拒绝 UI
- 检查工具是否定义了 `renderToolUseRejectedMessage` 方法
- 如果定义，使用工具特定的渲染逻辑（如显示被拒绝的文件编辑 diff）
- 支持 `style` 参数（如 `'condensed'` 紧凑模式）

### 2. 输入数据验证
- 使用工具的 `inputSchema.safeParse()` 验证输入数据
- 验证失败时回退到 `FallbackToolUseRejectedMessage`
- 确保传递给渲染方法的输入数据类型安全

### 3. 主题和尺寸适配
- 使用 `useTerminalSize` 获取终端列数
- 使用 `useTheme` 获取当前主题
- 将终端尺寸和主题传递给工具的渲染方法

## 具体技术实现

### 组件接口
```typescript
type Props = {
  input: { [key: string]: unknown };  // 工具输入数据
  progressMessagesForMessage: ProgressMessage[];
  style?: 'condensed';  // 可选的紧凑样式
  tool?: Tool;  // 可能为 undefined
  tools: Tools;
  lookups: ReturnType<typeof buildMessageLookups>;
  verbose: boolean;
  isTranscriptMode?: boolean;
};

export function UserToolRejectMessage(props: Props): React.ReactNode
```

### 渲染逻辑
```tsx
export function UserToolRejectMessage({
  input,
  progressMessagesForMessage,
  style,
  tool,
  tools,
  verbose,
  isTranscriptMode,
}: Props): React.ReactNode {
  const { columns } = useTerminalSize();
  const [theme] = useTheme();

  // 1. 检查工具是否支持自定义拒绝 UI
  if (!tool || !tool.renderToolUseRejectedMessage) {
    return <FallbackToolUseRejectedMessage />;
  }

  // 2. 验证输入数据
  const parsedInput = tool.inputSchema.safeParse(input);
  if (!parsedInput.success) {
    return <FallbackToolUseRejectedMessage />;
  }

  // 3. 调用工具特定的拒绝渲染
  return tool.renderToolUseRejectedMessage(parsedInput.data, {
    columns,
    messages: [],
    tools,
    verbose,
    progressMessagesForMessage: filterToolProgressMessages(progressMessagesForMessage),
    style,
    theme,
    isTranscriptMode,
  }) ?? <FallbackToolUseRejectedMessage />;
}
```

### React Compiler 优化
- 使用 `_c(13)` 创建 13 个记忆化槽位
- 使用 `Symbol.for("react.early_return_sentinel")` 实现提前返回优化
- 缓存依赖包括：
  - `columns`（终端列数）
  - `input`（工具输入）
  - `isTranscriptMode`
  - `progressMessagesForMessage`
  - `style`
  - `theme`
  - `tool`
  - `tools`
  - `verbose`

## 关键代码路径与文件引用

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/hooks/useTerminalSize.js` | 获取终端尺寸 |
| `src/ink.js` | `useTheme` |
| `src/Tool.js` | `Tool`, `Tools`, `filterToolProgressMessages` |
| `src/types/message.js` | `ProgressMessage` 类型 |
| `src/utils/messages.js` | `buildMessageLookups` 类型 |
| `src/components/FallbackToolUseRejectedMessage.js` | 回退拒绝 UI |

### 调用方
- `UserToolResultMessage.tsx`：当检测到 `REJECT_MESSAGE` 前缀或 `INTERRUPT_MESSAGE_FOR_TOOL_USE` 时渲染此组件

### 相关常量
```typescript
// src/utils/messages.ts
export const REJECT_MESSAGE =
  "The user doesn't want to proceed with this tool use. The tool use was rejected...";
export const INTERRUPT_MESSAGE_FOR_TOOL_USE = 
  '[Request interrupted by user for tool use]';
```

### 工具接口
```typescript
// src/Tool.ts
interface Tool {
  renderToolUseRejectedMessage?(
    input: z.infer<Input>,
    options: {
      columns: number;
      messages: Message[];
      style?: 'condensed';
      theme: ThemeName;
      tools: Tools;
      verbose: boolean;
      progressMessagesForMessage: ProgressMessage<P>[];
      isTranscriptMode?: boolean;
    },
  ): React.ReactNode;
}
```

## 依赖与外部交互

### 上游数据流
1. **用户拒绝**：用户在权限提示界面选择拒绝
2. **消息构造**：系统生成带 `REJECT_MESSAGE` 前缀的拒绝消息
3. **条件检测**：`UserToolResultMessage` 检测到拒绝条件
4. **拒绝渲染**：调用 `UserToolRejectMessage` 进行工具特定的拒绝渲染

### 工具特定拒绝示例
某些工具（如 `FileEditTool`）可能定义自定义拒绝 UI：
```tsx
// 示例：FileEditTool 可能显示被拒绝的编辑 diff
renderToolUseRejectedMessage(input, options) {
  return (
    <Box flexDirection="column">
      <Text color="error">Edit rejected</Text>
      <DiffView oldContent={input.old_string} newContent={input.new_string} />
    </Box>
  );
}
```

## 风险、边界与改进建议

### 已知风险
1. **输入验证失败**：如果工具输入不符合 schema，无法显示自定义拒绝 UI
2. **工具未定义方法**：大多数工具未定义 `renderToolUseRejectedMessage`，导致大量使用回退 UI
3. **终端尺寸变化**：频繁的终端尺寸变化可能导致不必要的重渲染

### 边界情况
| 场景 | 当前行为 | 建议 |
|-----|---------|------|
| tool = undefined | 显示 FallbackToolUseRejectedMessage | 符合预期 |
| inputSchema.safeParse 失败 | 显示 FallbackToolUseRejectedMessage | 添加日志记录 |
| renderToolUseRejectedMessage 返回 null | 显示 FallbackToolUseRejectedMessage | 符合预期 |
| columns = 0 | 可能布局异常 | 添加最小宽度检查 |

### 改进建议
1. **输入验证日志**：记录输入验证失败的原因，便于调试
   ```tsx
   if (!parsedInput.success) {
     logDebug(`Input validation failed for ${tool.name}:`, parsedInput.error);
     return <FallbackToolUseRejectedMessage />;
   }
   ```

2. **默认拒绝 UI 增强**：为常用工具类型提供默认的拒绝 UI 模板

3. **拒绝原因收集**：在拒绝时提供原因选项，传递给工具的渲染方法
   ```typescript
   renderToolUseRejectedMessage(input, options, rejectionReason)
   ```

4. **性能优化**：考虑使用 `useMemo` 缓存 `parsedInput` 结果

5. **响应式优化**：使用防抖处理终端尺寸变化

### 测试要点
- 验证工具特定拒绝 UI 的正确渲染
- 验证输入验证失败时的回退行为
- 验证终端尺寸变化时的响应
- 验证空工具场景的处理
- 验证紧凑模式（`condensed`）的正确应用
