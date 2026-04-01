# CompactSummary.tsx 研究文档

## 场景与职责

`CompactSummary` 是一个用于渲染对话紧凑摘要的 React 组件。它在对话历史被压缩（compact）后显示摘要信息，帮助用户了解哪些消息被压缩以及压缩的上下文。

### 核心场景

1. **对话压缩展示**：当对话历史被自动或手动压缩后，显示压缩摘要信息
2. **转录模式支持**：在转录（transcript）模式下显示完整的压缩内容
3. **压缩元数据展示**：显示被压缩的消息数量、压缩方向、用户上下文等信息

### 使用场景

该组件在以下场景中被使用：
- 自动压缩（auto-compact）后的摘要展示
- 手动压缩命令后的结果展示
- 历史消息中的压缩边界标记

## 功能点目的

### 1. 压缩摘要展示
- 显示 "Compact summary" 或 "Summarized conversation" 标题
- 根据是否有元数据展示不同级别的详细信息

### 2. 元数据展示（有 summarizeMetadata 时）
- 被压缩的消息数量（`messagesSummarized`）
- 压缩方向（`direction`: "up_to" | "from_this_point"）
- 用户提供的压缩上下文（`userContext`）

### 3. 转录模式支持
- 在非转录模式下显示快捷键提示（Ctrl+O 展开历史）
- 在转录模式下显示完整的压缩文本内容

### 4. 视觉区分
- 使用黑色圆点（BLACK_CIRCLE）作为视觉标记
- 通过 `MessageResponse` 组件统一消息响应样式

## 具体技术实现

### 组件 Props 定义

```typescript
type Props = {
  message: NormalizedUserMessage;  // 规范化后的用户消息
  screen: Screen;                   // 当前屏幕类型
}

// Screen 类型来自 REPL.tsx
type Screen = 'main' | 'transcript' | 'diff' | 'logs' | 'settings' | 'help' | 'skills' | 'agents'
```

### 关键流程

1. **模式检测**：
   ```typescript
   const isTranscriptMode = screen === "transcript"
   const textContent = getUserMessageText(message) || ""
   const metadata = message.summarizeMetadata
   ```

2. **渲染决策树**：
   ```
   has metadata?
   ├── Yes → 渲染 "Summarized conversation" 详细视图
   │           ├── 黑色圆点标记
   │           ├── 标题
   │           ├── 元数据信息（消息数、方向、上下文）
   │           ├── 快捷键提示（非转录模式）
   │           └── 完整内容（转录模式）
   └── No → 渲染 "Compact summary" 简洁视图
               ├── 黑色圆点标记
               ├── 标题
               ├── 快捷键提示（非转录模式）
               └── 完整内容（转录模式）
   ```

### 数据结构

**NormalizedUserMessage**（来自 `src/types/message.js`）：
```typescript
type NormalizedUserMessage = {
  type: 'user'
  message: {
    role: 'user'
    content: Array<{ type: 'text', text: string }>
  }
  summarizeMetadata?: {
    messagesSummarized: number
    userContext?: string
    direction?: 'up_to' | 'from_this_point'
  }
  // ... 其他字段
}
```

**SummarizeMetadata**：
```typescript
type SummarizeMetadata = {
  messagesSummarized: number      // 被压缩的消息数量
  userContext?: string            // 用户提供的压缩上下文
  direction?: 'up_to' | 'from_this_point'  // 压缩方向
}
```

### UI 结构

**有元数据时的渲染结构**：
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Box minWidth={2}>
      <Text color="text">{BLACK_CIRCLE}</Text>
    </Box>
    <Box flexDirection="column">
      <Text bold={true}>Summarized conversation</Text>
      {/* 非转录模式：显示元数据和快捷键 */}
      {!isTranscriptMode && (
        <MessageResponse>
          <Box flexDirection="column">
            <Text dimColor={true}>
              Summarized {metadata.messagesSummarized} messages {directionText}
            </Text>
            {metadata.userContext && (
              <Text dimColor={true}>Context: "{metadata.userContext}"</Text>
            )}
            <Text dimColor={true}>
              <ConfigurableShortcutHint 
                action="app:toggleTranscript" 
                context="Global" 
                fallback="ctrl+o" 
                description="expand history" 
                parens={true} 
              />
            </Text>
          </Box>
        </MessageResponse>
      )}
      {/* 转录模式：显示完整内容 */}
      {isTranscriptMode && (
        <MessageResponse>
          <Text>{textContent}</Text>
        </MessageResponse>
      )}
    </Box>
  </Box>
</Box>
```

**无元数据时的渲染结构**：
```tsx
<Box flexDirection="column" marginTop={1}>
  <Box flexDirection="row">
    <Box minWidth={2}>
      <Text color="text">{BLACK_CIRCLE}</Text>
    </Box>
    <Box flexDirection="column">
      <Text bold={true}>
        Compact summary
        {/* 非转录模式显示快捷键提示 */}
        {!isTranscriptMode && (
          <Text dimColor={true}>
            <ConfigurableShortcutHint ... />
          </Text>
        )}
      </Text>
    </Box>
  </Box>
  {/* 转录模式显示完整内容 */}
  {isTranscriptMode && (
    <MessageResponse>
      <Text>{textContent}</Text>
    </MessageResponse>
  )}
</Box>
```

### 方向文本映射

```typescript
metadata.direction === "up_to" 
  ? "up to this point" 
  : "from this point"
```

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/CompactSummary.tsx`

### 依赖文件

| 路径 | 用途 |
|------|------|
| `src/constants/figures.js` | `BLACK_CIRCLE` 视觉标记 |
| `src/ink.js` | `Box`, `Text` UI 组件 |
| `src/screens/REPL.js` | `Screen` 类型定义 |
| `src/types/message.js` | `NormalizedUserMessage` 类型 |
| `src/utils/messages.js` | `getUserMessageText` 消息文本提取 |
| `src/components/ConfigurableShortcutHint.js` | 快捷键提示组件 |
| `src/components/MessageResponse.js` | 消息响应样式组件 |

### 调用方文件

| 路径 | 调用场景 |
|------|----------|
| `src/components/Message.tsx` | 消息列表中的紧凑摘要渲染 |
| `src/components/MessageSelector.tsx` | 消息选择器中的摘要展示 |

### 相关服务

| 路径 | 用途 |
|------|------|
| `src/services/compact/compact.ts` | 压缩服务实现 |
| `src/services/compact/prompt.ts` | 压缩提示模板 |
| `src/services/compact/sessionMemoryCompact.ts` | 会话内存压缩 |

## 依赖与外部交互

### 运行时依赖

1. **React Compiler**：使用 `_c` 函数进行自动记忆化
2. **Ink**：终端 UI 渲染框架

### 消息类型系统

与 `messages.js` 工具函数交互：
- `getUserMessageText(message)`：从规范化消息中提取文本内容
- 处理多内容块的消息格式

### 快捷键系统

通过 `ConfigurableShortcutHint` 组件集成：
- 动作：`app:toggleTranscript`
- 上下文：`Global`
- 默认快捷键：`ctrl+o`

### 压缩服务

与压缩系统的数据流：
```
compact.ts → 创建压缩消息 → 设置 summarizeMetadata
    ↓
Message.tsx → 检测 isCompactSummary → 渲染 CompactSummary
    ↓
CompactSummary → 展示元数据和内容
```

## 风险、边界与改进建议

### 潜在风险

1. **元数据不一致**：
   - `messagesSummarized` 可能与实际压缩的消息数量不匹配
   - 需要确保压缩服务正确设置元数据

2. **长上下文截断**：
   - `userContext` 可能很长，当前实现没有长度限制
   - 可能导致终端显示问题

3. **文本内容提取失败**：
   - `getUserMessageText` 可能在某些边缘情况下返回空字符串
   - 需要确保有合理的降级显示

### 边界情况

1. **空元数据**：
   - 当 `summarizeMetadata` 为 undefined 时，显示简化版本
   - 这是正常情况，不是错误

2. **零消息压缩**：
   - 如果 `messagesSummarized` 为 0，显示 "Summarized 0 messages"
   - 可能需要特殊处理这种情况

3. **长用户上下文**：
   - `userContext` 可能包含多行文本
   - 当前实现会直接显示，可能影响布局

4. **转录模式切换**：
   - 组件根据 `screen` prop 改变渲染
   - 需要确保状态一致性

### 改进建议

1. **元数据验证**：
   ```typescript
   // 建议：添加元数据验证
   const validatedMetadata = metadata && metadata.messagesSummarized > 0 
     ? metadata 
     : undefined
   ```

2. **长文本截断**：
   ```typescript
   // 建议：对用户上下文进行长度限制
   const displayContext = userContext && userContext.length > 100
     ? userContext.slice(0, 100) + '...'
     : userContext
   ```

3. **空状态处理**：
   ```typescript
   // 建议：对空内容显示占位符
   const displayText = textContent || '(empty summary)'
   ```

4. **国际化支持**：
   - 当前文本都是硬编码的英文
   - 考虑添加 i18n 支持

5. **代码优化**：
   - React Compiler 生成的代码可读性较差
   - 考虑提取重复的渲染逻辑为子组件

6. **测试覆盖**：
   - 添加单元测试验证不同屏幕模式下的渲染
   - 测试元数据的各种边界情况

7. **可访问性**：
   ```typescript
   // 建议：添加语义化标记
   <Box aria-label={`Summarized ${count} messages`}>
   ```
