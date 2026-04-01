# PromptInputQueuedCommands.tsx 深度研究文档

> **研究对象**: `src/components/PromptInput/PromptInputQueuedCommands.tsx`  
> **研究范围**: 源码、调用方、命令队列系统、消息格式化、QueuedMessageContext 及相关依赖  
> **执行器**: kimi (k2p5)  
> **研究日期**: 2026-04-01

---

## 1. 场景与职责

### 1.1 核心定位

`PromptInputQueuedCommands.tsx` 是 Claude Code CLI 中**命令队列预览组件**，负责在输入框上方渲染当前排队等待执行的命令。它让用户能够提前看到即将被处理的消息（如后台任务通知、待执行的 bash 命令、channel 消息等），并在查看 teammate transcript 时自动隐藏以避免干扰。

### 1.2 应用场景

| 场景 | 说明 |
|------|------|
| **任务通知预览** | 后台任务完成时，在输入框上方显示任务结果摘要 |
| **待执行命令预览** | 用户通过 UP 键从队列中拉取命令前，先显示队列内容 |
| **Channel 消息预览** | KAIROS 模式下，来自外部 channel 的消息排队显示 |
| **Bash 命令预览** | 队列中的 bash 命令以 `<bash-input>` 标签包裹后预览 |
| **查看 teammate 时隐藏** | 当用户正在查看某个 teammate 的 transcript 时，不显示 leader 的队列内容 |

### 1.3 职责边界

- **队列内容消费**：通过 `useCommandQueue()` 订阅统一命令队列
- **可见性过滤**：只显示 `isQueuedCommandVisible()` 判定为可见的命令
- **通知数量限制**：任务通知最多显示 3 条，超出时折叠为 "+N more tasks completed"
- **消息格式转换**：将 `QueuedCommand` 转换为 `Message` 组件可渲染的 `NormalizedMessage`
- **Brief 布局适配**：在 KAIROS_BRIEF 模式下使用紧凑布局（无水平内边距）

---

## 2. 功能点目的

### 2.1 队列内容可视化

将原本在后台静默处理的命令队列暴露给用户，提供系统状态的透明度。用户可以看到：
- 哪些后台任务已完成
- 哪些命令正在等待执行
- 是否有来自其他 channel 的消息

### 2.2 任务通知数量控制

防止大量后台任务通知淹没输入区域：

```typescript
const MAX_VISIBLE_NOTIFICATIONS = 3;
```

当任务通知超过 3 条时，显示前 2 条 + 1 条汇总消息。

### 2.3 Idle 通知静默过滤

某些系统通知（如 `idle_notification`）不应显示给用户：

```typescript
function isIdleNotification(value: string): boolean {
  try {
    const parsed = jsonParse(value);
    return parsed?.type === 'idle_notification';
  } catch {
    return false;
  }
}
```

### 2.4 Bash 命令标签包裹

队列中的 bash 命令在预览时需要包裹在 `<bash-input>` XML 标签中，以便 `Message` 组件正确渲染：

```typescript
if (cmd.mode === 'bash' && typeof content === 'string') {
  content = `<bash-input>${content}</bash-input>`;
}
```

---

## 3. 具体技术实现

### 3.1 队列处理流程

```typescript
function processQueuedCommands(queuedCommands: QueuedCommand[]): QueuedCommand[] {
  // 1. 过滤 idle 通知
  const filteredCommands = queuedCommands.filter(
    cmd => typeof cmd.value !== 'string' || !isIdleNotification(cmd.value)
  );

  // 2. 分离任务通知和其他命令
  const taskNotifications = filteredCommands.filter(cmd => cmd.mode === 'task-notification');
  const otherCommands = filteredCommands.filter(cmd => cmd.mode !== 'task-notification');

  // 3. 如果通知数量在限制内，直接返回
  if (taskNotifications.length <= MAX_VISIBLE_NOTIFICATIONS) {
    return [...otherCommands, ...taskNotifications];
  }

  // 4. 截断并生成汇总消息
  const visibleNotifications = taskNotifications.slice(0, MAX_VISIBLE_NOTIFICATIONS - 1);
  const overflowCount = taskNotifications.length - (MAX_VISIBLE_NOTIFICATIONS - 1);
  
  const overflowCommand: QueuedCommand = {
    value: createOverflowNotificationMessage(overflowCount),
    mode: 'task-notification'
  };
  
  return [...otherCommands, ...visibleNotifications, overflowCommand];
}
```

### 3.2 汇总消息格式

```typescript
function createOverflowNotificationMessage(count: number): string {
  return `<${TASK_NOTIFICATION_TAG}>
<${SUMMARY_TAG}>+${count} more tasks completed</${SUMMARY_TAG}>
<${STATUS_TAG}>completed</${STATUS_TAG}>
</${TASK_NOTIFICATION_TAG}>`;
}
```

### 3.3 消息创建与 Memoization

```typescript
const messages = useMemo(() => {
  if (queuedCommands.length === 0) return null;
  
  const visibleCommands = queuedCommands.filter(isQueuedCommandVisible);
  if (visibleCommands.length === 0) return null;
  
  const processedCommands = processQueuedCommands(visibleCommands);
  
  return normalizeMessages(
    processedCommands.map(cmd => {
      let content = cmd.value;
      if (cmd.mode === 'bash' && typeof content === 'string') {
        content = `<bash-input>${content}</bash-input>`;
      }
      return createUserMessage({ content });
    })
  );
}, [queuedCommands]);
```

**Memoization 的必要性**：`createUserMessage()` 每次调用都会生成新的 UUID。如果没有 `useMemo`，每次重渲染都会导致 `Message` 组件的 `areMessagePropsEqual` 比较失败（因为比较 `uuid`），从而引起闪烁。

### 3.4 Brief 布局判定

```typescript
const useBriefLayout = feature('KAIROS') || feature('KAIROS_BRIEF')
  ? useAppState(s => s.isBriefOnly)
  : false;
```

在 brief 模式下，`QueuedMessageProvider` 会将 `paddingX` 设为 0，避免与 `HighlightedThinkingText` / `BriefTool` 的现有缩进产生双重缩进。

### 3.5 渲染输出

```typescript
return (
  <Box marginTop={1} flexDirection="column">
    {messages.map((message, i) => (
      <QueuedMessageProvider 
        key={i} 
        isFirst={i === 0} 
        useBriefLayout={useBriefLayout}
      >
        <Message 
          message={message} 
          lookups={EMPTY_LOOKUPS}
          addMargin={false}
          tools={[]}
          commands={[]}
          verbose={false}
          inProgressToolUseIDs={EMPTY_SET}
          progressMessagesForMessage={[]}
          shouldAnimate={false}
          shouldShowDot={false}
          isTranscriptMode={false}
          isStatic={true}
        />
      </QueuedMessageProvider>
    ))}
  </Box>
);
```

---

## 4. 关键代码路径与文件引用

### 4.1 组件入口

| 路径 | 说明 |
|------|------|
| `src/components/PromptInput/PromptInputQueuedCommands.tsx` | 主组件文件 |
| `src/components/PromptInput/PromptInput.tsx` | 直接调用方 |
| `src/screens/REPL.tsx` | 间接调用方（渲染 PromptInput） |

### 4.2 队列系统依赖

| 路径 | 说明 |
|------|------|
| `src/hooks/useCommandQueue.ts` | React hook，通过 `useSyncExternalStore` 订阅队列 |
| `src/utils/messageQueueManager.ts` | 统一命令队列的核心实现，提供 `subscribeToCommandQueue`、`getCommandQueueSnapshot`、`isQueuedCommandVisible`、`isQueuedCommandEditable` 等 |
| `src/types/textInputTypes.ts` | `QueuedCommand`、`PromptInputMode` 类型定义 |

### 4.3 消息系统依赖

| 路径 | 说明 |
|------|------|
| `src/utils/messages.ts` | `createUserMessage`、`EMPTY_LOOKUPS`、`normalizeMessages` |
| `src/components/Message.tsx` | 消息渲染组件 |
| `src/context/QueuedMessageContext.tsx` | 为队列中的消息提供上下文（`isQueued`、`isFirst`、`paddingWidth`） |
| `src/constants/xml.ts` | `STATUS_TAG`、`SUMMARY_TAG`、`TASK_NOTIFICATION_TAG`、`BASH_INPUT_TAG` |
| `src/utils/slowOperations.ts` | `jsonParse` |

### 4.4 状态依赖

| 路径 | 说明 |
|------|------|
| `src/state/AppState.ts` | `useAppState`，读取 `viewingAgentTaskId`、`isBriefOnly` |

---

## 5. 依赖与外部交互

### 5.1 统一命令队列架构

```
用户输入 / 系统通知 / 桥接消息
  ↓
messageQueueManager.ts (模块级队列)
  ↓
useSyncExternalStore
  ↓
useCommandQueue.ts
  ↓
PromptInputQueuedCommands.tsx
```

队列支持三种优先级：`now` > `next` > `later`。本组件只读取队列状态，不参与入队/出队。

### 5.2 可见性判定逻辑

```typescript
export function isQueuedCommandVisible(cmd: QueuedCommand): boolean {
  if (
    (feature('KAIROS') || feature('KAIROS_CHANNELS')) &&
    cmd.origin?.kind === 'channel'
  )
    return true;
  return isQueuedCommandEditable(cmd);
}
```

可见命令包括：
- 所有可编辑命令（非 `task-notification` 模式且 `isMeta !== true`）
- KAIROS/KAIROS_CHANNELS 功能开启时的 channel 来源命令

### 5.3 与 Message 组件的交互

`Message` 组件接收大量 props，本组件中大部分传空值/默认值：
- `isStatic={true}`：表示这是静态消息，不参与流式动画
- `shouldAnimate={false}`：禁用打字机动画
- `tools={[]}`、`commands={[]}`：队列预览不显示工具调用
- `lookups={EMPTY_LOOKUPS}`：无查找表

`QueuedMessageContext` 则向 `Message` 组件的后代传递 `isQueued=true`，使消息渲染器知道这是队列中的消息，可能应用不同的样式（如更暗淡的颜色）。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 |
|------|------|
| **`key={i}` 使用索引** | `messages.map((message, i) => <QueuedMessageProvider key={i} ...>)` 在消息顺序变化时可能导致 React 重渲染问题；不过队列消息通常是追加的，风险较低 |
| **硬编码 `MAX_VISIBLE_NOTIFICATIONS = 3`** | 没有根据终端高度动态调整，在极矮终端上仍可能占用过多空间 |
| **`jsonParse` 异常处理** | `isIdleNotification` 中 `jsonParse` 失败时返回 `false`，但如果命令值是超大字符串，解析本身可能有性能开销 |
| **无测试覆盖** | 未找到针对该组件的单元测试 |

### 6.2 边界情况

- **队列为空**：`queuedCommands.length === 0` 时返回 `null`
- **无可见命令**：`visibleCommands.length === 0` 时返回 `null`
- **查看 teammate 时**：`viewingAgent` 为 `true` 时返回 `null`，避免 leader 队列与 teammate transcript 混淆
- **任务通知恰好 3 条**：无需折叠，全部显示
- **任务通知 4 条**：显示 2 条 + 1 条 "+2 more tasks completed"
- **Bash 命令值为 `ContentBlockParam[]`**：不会包裹 `<bash-input>`，因为类型检查 `typeof content === 'string'` 会失败

### 6.3 改进建议

1. **使用稳定 key**：如果 `QueuedCommand` 有 `uuid` 字段，优先使用 `uuid` 作为 `key` 而非数组索引
2. **动态限制通知数量**：根据终端 `rows` 动态计算 `MAX_VISIBLE_NOTIFICATIONS`，在矮终端上显示更少条目
3. **优化 idle 通知检测**：`isIdleNotification` 可以先检查字符串是否以 `{"type":"idle_notification"` 开头，避免不必要的 `jsonParse`
4. **提取消息转换逻辑**：将 `processQueuedCommands` + `map` + `normalizeMessages` 提取为独立的纯函数，便于单元测试
5. **增加测试覆盖**：测试队列过滤、通知折叠、bash 标签包裹、viewingAgent 隐藏等核心逻辑
