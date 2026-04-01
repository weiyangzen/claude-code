# UserToolErrorMessage.tsx 深度研究文档

## 场景与职责

`UserToolErrorMessage` 是 Claude Code 终端 UI 中用于渲染**工具执行错误**的核心组件。它负责处理多种错误类型的条件渲染，包括用户中断、计划拒绝、工具拒绝、分类器拒绝以及通用工具错误。

### 核心职责
1. **多类型错误分发**：根据错误内容前缀和特征路由到不同的子组件
2. **用户中断处理**：检测并渲染用户主动中断的场景
3. **计划拒绝处理**：提取并渲染被拒绝的计划内容
4. **分类器拒绝处理**：支持自动模式分类器的拒绝提示
5. **通用错误回退**：使用工具特定的错误渲染或回退组件

## 功能点目的

### 1. 错误类型路由
组件通过检查 `param.content` 的内容特征，将错误路由到不同的渲染路径：

| 检查条件 | 路由目标 | 说明 |
|---------|---------|------|
| 包含 `INTERRUPT_MESSAGE_FOR_TOOL_USE` | `InterruptedByUser` | 用户中断 |
| 以 `PLAN_REJECTION_PREFIX` 开头 | `RejectedPlanMessage` | 计划拒绝 |
| 以 `REJECT_MESSAGE_WITH_REASON_PREFIX` 开头 | `RejectedToolUseMessage` | 工具拒绝 |
| 分类器拒绝（`isClassifierDenial`） | 内置分类器拒绝 UI | 自动模式拒绝 |
| 默认 | `renderToolUseErrorMessage` 或 `FallbackToolUseErrorMessage` | 通用错误 |

### 2. 分类器拒绝支持
当 `TRANSCRIPT_CLASSIFIER` 功能启用时：
- 检测分类器拒绝消息
- 显示简洁的拒绝提示："Denied by auto mode classifier ∙ /feedback if incorrect"
- 使用 `BULLET_OPERATOR`（∙）作为分隔符

### 3. 工具特定错误渲染
- 优先调用工具的 `renderToolUseErrorMessage` 方法
- 如果工具未定义该方法，使用 `FallbackToolUseErrorMessage`
- 支持 `verbose` 和 `isTranscriptMode` 模式

## 具体技术实现

### 组件接口
```typescript
type Props = {
  progressMessagesForMessage: ProgressMessage[];
  tool?: Tool;  // 可能为 undefined（恢复旧对话时）
  tools: Tools;
  param: ToolResultBlockParam;
  verbose: boolean;
  isTranscriptMode?: boolean;
};

export function UserToolErrorMessage(props: Props): React.ReactNode
```

### 错误路由逻辑
```tsx
// 1. 用户中断检查
if (typeof param.content === "string" && 
    param.content.includes(INTERRUPT_MESSAGE_FOR_TOOL_USE)) {
  return <MessageResponse height={1}><InterruptedByUser /></MessageResponse>;
}

// 2. 计划拒绝检查
if (typeof param.content === "string" && 
    param.content.startsWith(PLAN_REJECTION_PREFIX)) {
  const planContent = param.content.substring(PLAN_REJECTION_PREFIX.length);
  return <RejectedPlanMessage plan={planContent} />;
}

// 3. 工具拒绝检查
if (typeof param.content === "string" && 
    param.content.startsWith(REJECT_MESSAGE_WITH_REASON_PREFIX)) {
  return <RejectedToolUseMessage />;
}

// 4. 分类器拒绝检查（功能开关控制）
if (feature("TRANSCRIPT_CLASSIFIER") && 
    typeof param.content === "string" && 
    isClassifierDenial(param.content)) {
  return <MessageResponse height={1}>
    <Text dimColor>
      Denied by auto mode classifier {BULLET_OPERATOR} /feedback if incorrect
    </Text>
  </MessageResponse>;
}

// 5. 通用错误处理
return tool?.renderToolUseErrorMessage?.(param.content, {
  progressMessagesForMessage: filterToolProgressMessages(progressMessagesForMessage),
  tools,
  verbose,
  isTranscriptMode
}) ?? <FallbackToolUseErrorMessage result={param.content} verbose={verbose} />;
```

### React Compiler 优化
- 使用 `_c(14)` 创建 14 个记忆化槽位
- 每个条件分支的结果都被独立缓存
- 通用错误路径缓存依赖：
  - `isTranscriptMode`
  - `param.content`
  - `progressMessagesForMessage`
  - `tool`
  - `tools`
  - `verbose`

## 关键代码路径与文件引用

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `@anthropic-ai/sdk/resources/index.mjs` | `ToolResultBlockParam` 类型 |
| `src/constants/figures.js` | `BULLET_OPERATOR` 常量 |
| `src/ink.js` | `Text` 组件 |
| `src/Tool.js` | `Tool`, `Tools`, `filterToolProgressMessages` |
| `src/types/message.js` | `ProgressMessage` 类型 |
| `src/utils/messages.js` | 错误消息常量 |
| `src/components/FallbackToolUseErrorMessage.js` | 回退错误 UI |
| `src/components/InterruptedByUser.js` | 中断提示 |
| `src/components/MessageResponse.js` | 消息容器 |
| `./RejectedPlanMessage.js` | 计划拒绝组件 |
| `./RejectedToolUseMessage.js` | 工具拒绝组件 |

### 关键常量
```typescript
// src/utils/messages.ts
export const INTERRUPT_MESSAGE_FOR_TOOL_USE = 
  '[Request interrupted by user for tool use]';
export const PLAN_REJECTION_PREFIX = 
  'The agent proposed a plan that was rejected...';
export const REJECT_MESSAGE_WITH_REASON_PREFIX = 
  "The user doesn't want to proceed with this tool use...";

export function isClassifierDenial(content: string): boolean {
  return content.startsWith(AUTO_MODE_REJECTION_PREFIX);
}
```

### 功能开关
```typescript
// bun:bundle feature system
feature("TRANSCRIPT_CLASSIFIER")  // 控制分类器拒绝 UI
```

## 依赖与外部交互

### 上游数据流
1. **工具执行**：工具执行失败，生成错误结果
2. **消息归一化**：`normalizeMessages` 处理消息结构
3. **条件检测**：`UserToolResultMessage` 检测到 `is_error` 标志
4. **错误渲染**：调用 `UserToolErrorMessage` 进行具体渲染

### 下游组件交互
```
UserToolResultMessage
  └── UserToolErrorMessage
       ├── InterruptedByUser (用户中断)
       ├── RejectedPlanMessage (计划拒绝)
       ├── RejectedToolUseMessage (工具拒绝)
       ├── [内置分类器 UI] (分类器拒绝)
       └── FallbackToolUseErrorMessage (通用错误回退)
```

## 风险、边界与改进建议

### 已知风险
1. **前缀冲突**：如果错误消息内容恰好包含某个前缀，可能被错误路由
2. **空工具处理**：`tool` 可能为 `undefined`（旧对话恢复场景），依赖可选链操作符
3. **功能开关耦合**：分类器拒绝检测与功能开关强耦合

### 边界情况
| 场景 | 当前行为 | 建议 |
|-----|---------|------|
| 多个前缀匹配 | 按代码顺序第一个匹配生效 | 明确优先级文档 |
| content 非字符串 | 跳过前缀检查，进入通用错误 | 添加类型守卫日志 |
| tool.renderToolUseErrorMessage 返回 null | 回退到 FallbackToolUseErrorMessage | 符合预期 |

### 改进建议
1. **优先级文档化**：明确错误类型的匹配优先级
   ```typescript
   // 建议添加注释说明优先级
   // Priority: Interrupt > Plan Rejection > Tool Rejection > Classifier Denial > Generic
   ```

2. **错误代码化**：使用结构化错误代码替代字符串前缀匹配
   ```typescript
   if (param.errorCode === ErrorCode.INTERRUPT) { ... }
   ```

3. **分类器信息增强**：显示分类器拒绝的具体原因
   ```tsx
   <Text dimColor>
     Denied by auto mode: {classifierReason} {BULLET_OPERATOR} /feedback if incorrect
   </Text>
   ```

4. **错误聚合**：对于批量工具错误，考虑聚合显示

### 测试要点
- 验证每种错误类型的正确路由
- 验证功能开关对分类器拒绝的影响
- 验证空工具场景的回退行为
- 验证前缀边界情况（如内容恰好以前缀开头但非预期场景）
- 验证 React Compiler 缓存行为
