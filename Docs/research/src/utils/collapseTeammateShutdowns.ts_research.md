# collapseTeammateShutdowns.ts 研究文档

> 文件路径：`src/utils/collapseTeammateShutdowns.ts`  
> 行数：55 行  
> 研究日期：2026-04-01

---

## 场景与职责

在 Claude Code 的多代理（teammate）模式下，当一个在途（in-process）子代理完成任务并关闭时，系统会生成一条 `task_status` 类型的附件消息（attachment），其状态为 `completed`。如果用户同时启动了多个子代理，这些关闭通知会连续出现在消息流中，导致消息列表被大量重复的 " teammate completed" 提示占据。

`collapseTeammateShutdowns.ts` 的职责非常聚焦：**将连续的 teammate 关闭附件消息折叠成一条聚合附件**，仅显示数量（如 "3 teammates completed"），从而保持消息列表的整洁。

---

## 功能点目的

| 导出项 | 签名 | 目的 |
|--------|------|------|
| `collapseTeammateShutdowns` | `(messages: RenderableMessage[]) => RenderableMessage[]` | 遍历消息数组，把连续出现的 in-process teammate `completed` 状态附件合并为单条 `teammate_shutdown_batch` 附件 |

**折叠规则：**
- 仅匹配 `type === 'attachment' && attachment.type === 'task_status' && attachment.taskType === 'in_process_teammate' && attachment.status === 'completed'` 的消息。
- 连续出现的此类消息会被计数。
- 若计数为 1，保留原消息不变；若大于 1，生成一条新的 `teammate_shutdown_batch` 附件，携带 `count` 字段。
- 非匹配消息原样透传。

---

## 具体技术实现

### 3.1 类型守卫

```ts
function isTeammateShutdownAttachment(
  msg: RenderableMessage,
): msg is AttachmentMessage {
  return (
    msg.type === 'attachment' &&
    msg.attachment.type === 'task_status' &&
    msg.attachment.taskType === 'in_process_teammate' &&
    msg.attachment.status === 'completed'
  )
}
```

该类型守卫精确筛选出需要折叠的目标消息。注意它**不**折叠其他类型的 task_status（如 `started` 或 `failed`），也不折叠非 teammate 的任务状态。

### 3.2 折叠算法

采用双指针（单索引递增）遍历：

```ts
while (i < messages.length) {
  const msg = messages[i]!
  if (isTeammateShutdownAttachment(msg)) {
    let count = 0
    while (i < messages.length && isTeammateShutdownAttachment(messages[i]!)) {
      count++
      i++
    }
    if (count === 1) {
      result.push(msg)
    } else {
      result.push({
        type: 'attachment',
        uuid: msg.uuid,
        timestamp: msg.timestamp,
        attachment: {
          type: 'teammate_shutdown_batch',
          count,
        },
      })
    }
  } else {
    result.push(msg)
    i++
  }
}
```

**关键设计点：**
- 聚合后的新附件复用**第一条**原消息的 `uuid` 和 `timestamp`，保证消息列表的键稳定（React key 不会抖动）和时间轴连续性。
- 仅对**连续**的关闭通知生效；如果中间插入了其他消息，不会跨段聚合。

### 3.3 输出消息结构

聚合后生成的消息格式：
```ts
{
  type: 'attachment',
  uuid: string,        // 取自首条原消息
  timestamp: number,   // 取自首条原消息
  attachment: {
    type: 'teammate_shutdown_batch',
    count: number,
  }
}
```

---

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/types/message.js`（或等效类型定义） | `AttachmentMessage`、`RenderableMessage` 类型 |

### 调用方
| 文件 | 调用点 |
|------|--------|
| `src/components/Messages.tsx` | 消息列表渲染前调用 `collapseTeammateShutdowns` 进行预处理 |
| `src/components/messages/AttachmentMessage.tsx` | 渲染 `teammate_shutdown_batch` 类型的附件 |

---

## 依赖与外部交互

- **纯函数**：无外部状态、无 I/O、无副作用。
- **输入/输出契约**：接收并返回 `RenderableMessage[]`，对非目标消息完全透传，不影响消息流的其他部分。
- **下游渲染**：`AttachmentMessage.tsx` 需要识别 `attachment.type === 'teammate_shutdown_batch'` 并渲染为类似 "+N teammates completed" 的 UI。

---

## 风险、边界与改进建议

### 风险与边界

1. **连续性依赖**
   - 如果消息流中两个 teammate 关闭通知之间隔了一条系统消息或用户消息，它们不会被聚合。这在子代理异步完成时可能发生，导致聚合效果打折。

2. **UUID 复用**
   - 聚合消息只保留第一条的 `uuid`，若下游有逻辑依赖每条 teammate 关闭通知的独立 `uuid`（如点击跳转、高亮），可能会丢失后续消息的标识。

3. **无状态去重**
   - 该函数每次对完整消息数组重新扫描，时间复杂度 O(n)。由于消息数组通常不大（几十到几百条），性能可接受，但对于极长会话理论上存在线性累积。

### 改进建议

1. **扩展聚合范围**
   如果业务上允许，可考虑在扫描时跳过 `shouldSkipMessage` 类型的消息（如 thinking、系统状态），实现“跨空白聚合”。但这需要与 `collapseReadSearch.ts` 的跳过逻辑对齐，避免顺序错乱。

2. **保留子代理 ID 列表**
   除了 `count`，还可以在聚合附件中附加 `agentIds: string[]`，让 UI 支持点击展开查看具体是哪些 teammate 完成了。

3. **单元测试**
   建议补充以下边界测试：
   - 空数组、单条、多条连续、多条不连续
   - 混合其他 attachment 类型时不误折叠
   - 输出消息的 `uuid`/`timestamp` 与首条输入一致
