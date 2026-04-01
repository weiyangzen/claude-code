# UserAgentNotificationMessage.tsx 深度研究文档

## 1. 场景与职责

### 1.1 定位
`UserAgentNotificationMessage.tsx` 是 Claude Code CLI 中专门用于渲染**用户代理通知消息**的组件。它负责展示来自用户代理（User Agent）的状态通知，特别是子代理（subagent）执行结果的摘要信息。

### 1.2 核心职责
- **代理状态可视化**：将子代理的执行状态（完成/失败/终止）以图标+文本形式展示
- **消息内容提取**：从 XML 格式的消息文本中提取 `summary` 和 `status` 标签内容
- **状态颜色映射**：根据状态值映射到对应的颜色主题（success/error/warning）
- **简洁通知展示**：提供紧凑的单行通知格式，适合嵌入消息流

### 1.3 使用场景
- 子代理完成任务后显示结果摘要
- 后台代理执行失败时显示错误提示
- 代理被用户终止时显示状态通知
- 转录模式（transcript mode）中的代理活动记录

---

## 2. 功能点目的

### 2.1 核心功能

| 功能 | 描述 |
|------|------|
| 标签内容提取 | 使用 `extractTag` 从 XML 文本中提取 `summary` 和 `status` |
| 状态颜色映射 | `completed`→success, `failed`→error, `killed`→warning |
| 条件渲染 | 无 summary 时返回 null，避免空消息 |
| 图标状态指示 | 使用 `BLACK_CIRCLE` 配合状态颜色作为视觉指示器 |

### 2.2 UI 设计

```
● Task completed successfully
↑
└─ 状态颜色图标（success=绿色, error=红色, warning=黄色）

[summary 文本内容]
```

### 2.3 消息格式

输入消息预期格式：
```xml
<summary>Task completed successfully</summary>
<status>completed</status>
```

或纯文本（无标签时返回 null）：
```
普通文本消息（不会被渲染）
```

---

## 3. 具体技术实现

### 3.1 Props 定义

```typescript
type Props = {
  addMargin: boolean;      // 控制顶部边距
  param: TextBlockParam;   // Anthropic SDK 的文本块参数
};

// TextBlockParam 来自 @anthropic-ai/sdk
type TextBlockParam = {
  type: 'text';
  text: string;
};
```

### 3.2 状态颜色映射函数

```typescript
function getStatusColor(status: string | null): TextProps['color'] {
  switch (status) {
    case 'completed':
      return 'success';
    case 'failed':
      return 'error';
    case 'killed':
      return 'warning';
    default:
      return 'text';  // 默认文本颜色
  }
}
```

### 3.3 主组件实现

```typescript
export function UserAgentNotificationMessage({
  addMargin,
  param: { text },
}: Props): React.ReactNode {
  // 1. 提取 summary 标签内容
  const summary = extractTag(text, 'summary');
  if (!summary) {
    return null;  // 无摘要时不渲染
  }

  // 2. 提取 status 标签内容并映射颜色
  const status = extractTag(text, 'status');
  const color = getStatusColor(status);

  // 3. 渲染通知
  return (
    <Box marginTop={addMargin ? 1 : 0}>
      <Text>
        <Text color={color}>{BLACK_CIRCLE}</Text> {summary}
      </Text>
    </Box>
  );
}
```

### 3.4 extractTag 工具函数

`extractTag` 定义在 `src/utils/messages.ts`，实现 XML 标签内容提取：

```typescript
export function extractTag(html: string, tagName: string): string | null {
  if (!html.trim() || !tagName.trim()) {
    return null;
  }

  const escapedTag = escapeRegExp(tagName);

  // 处理自闭合标签、带属性标签、嵌套标签、多行内容
  const pattern = new RegExp(
    `<${escapedTag}(?:\\s+[^>]*)?>` +  // 开始标签（可选属性）
      '([\\s\\S]*?)' +                  // 内容（非贪婪匹配）
      `<\\/${escapedTag}>`,            // 结束标签
    'gi',
  );

  // 处理嵌套标签的深度计算逻辑...
  // 返回匹配到的内容或 null
}
```

### 3.5 React Compiler 优化

组件使用 12 个缓存槽位：
- `$[0-1]`: summary 提取结果缓存（依赖 `text`）
- `$[2-3]`: color 计算结果缓存（依赖 `text`）
- `$[4-5]`: 图标文本缓存（依赖 `color`）
- `$[6-8]`: 组合文本缓存（依赖 `summary` 和图标）
- `$[9-11]`: 根容器缓存（依赖 `addMargin` 和组合文本）

---

## 4. 关键代码路径与文件引用

### 4.1 组件调用链

```
UserMessage (Message.tsx)
  └── case "text":
        └── UserTextMessage
              └── 内部解析逻辑
                    └── 检测到代理通知特征
                          └── UserAgentNotificationMessage
```

### 4.2 依赖文件映射

| 依赖路径 | 用途 |
|---------|------|
| `@anthropic-ai/sdk/resources/index.mjs` | TextBlockParam 类型 |
| `../../constants/figures.js` | BLACK_CIRCLE 常量 |
| `../../ink.js` | Box, Text, TextProps 组件和类型 |
| `../../utils/messages.js` | extractTag 工具函数 |

### 4.3 相关常量

```typescript
// src/constants/figures.ts
export const BLACK_CIRCLE = env.platform === 'darwin' ? '⏺' : '●';
```

### 4.4 颜色映射

```typescript
// Ink 主题系统中的颜色定义
type TextProps['color'] = 
  | 'success'  // 绿色 - 完成状态
  | 'error'    // 红色 - 失败状态
  | 'warning'  // 黄色 - 警告/终止状态
  | 'text';    // 默认文本颜色
```

---

## 5. 依赖与外部交互

### 5.1 与消息系统的集成

**输入来源**：
- 子代理通过 `AgentTool` 返回的结果
- `TaskOutputTool` 输出的任务结果
- 系统内部生成的代理状态通知

**典型消息生成流程**：
```typescript
// 在 AgentTool 或相关工具中
const notificationText = `
<summary>${result.summary}</summary>
<status>${result.status}</status>
`;

return createUserMessage({
  content: [{ type: 'text', text: notificationText }],
});
```

### 5.2 与 UserTextMessage 的关系

`UserAgentNotificationMessage` 通常被 `UserTextMessage` 内部调用：

```typescript
// UserTextMessage.tsx 中的伪代码
export function UserTextMessage({ param, ... }) {
  // 1. 首先尝试作为代理通知渲染
  const notification = tryRenderAgentNotification(param.text);
  if (notification) return notification;
  
  // 2. 然后尝试其他特殊格式
  const bashInput = tryRenderBashInput(param.text);
  if (bashInput) return bashInput;
  
  // 3. 最后作为普通文本渲染
  return <Text>...</Text>;
}
```

### 5.3 extractTag 的健壮性

`extractTag` 函数处理了多种边界情况：

1. **嵌套标签**：通过深度计数正确处理嵌套的同类型标签
2. **多行内容**：使用 `[\s\S]*?` 匹配包括换行符在内的任意字符
3. **标签属性**：使用 `(?:\s+[^>]*)?` 跳过开始标签的属性
4. **大小写不敏感**：使用 `gi` 标志进行不区分大小写的匹配

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险点 | 描述 | 严重程度 |
|-------|------|---------|
| 硬编码标签名 | `summary` 和 `status` 是硬编码的，变更需要同步修改 | 低 |
| 状态值硬编码 | `completed/failed/killed` 状态值无集中定义 | 低 |
| 无验证解析 | 依赖 `extractTag` 的健壮性，无 Schema 验证 | 中 |
| 单标签限制 | 只提取第一个匹配的 `summary` 标签 | 低 |
| 颜色回退 | 未知状态使用 `text` 颜色，可能不够醒目 | 低 |

### 6.2 边界情况

1. **空 summary**：
   ```typescript
   if (!summary) {
     return null;
   }
   ```
   当 `summary` 标签为空或不存在时，组件不渲染任何内容。

2. **未知状态**：
   ```typescript
   default:
     return 'text';
   ```
   未识别的状态值使用默认文本颜色。

3. **多行 summary**：
   `extractTag` 支持多行内容，但渲染时不会特殊处理换行。

4. **特殊字符**：
   XML 实体（如 `&lt;`）需要调用方预先解码。

### 6.3 改进建议

1. **类型安全增强**：
   ```typescript
   // 定义状态枚举
   const AgentStatus = {
     COMPLETED: 'completed',
     FAILED: 'failed',
     KILLED: 'killed',
   } as const;
   
   type AgentStatus = typeof AgentStatus[keyof typeof AgentStatus];
   
   function getStatusColor(status: AgentStatus | null): TextProps['color'] {
     const colorMap: Record<AgentStatus, TextProps['color']> = {
       [AgentStatus.COMPLETED]: 'success',
       [AgentStatus.FAILED]: 'error',
       [AgentStatus.KILLED]: 'warning',
     };
     return status ? colorMap[status] ?? 'text' : 'text';
   }
   ```

2. **Schema 验证**：
   ```typescript
   import { z } from 'zod';
   
   const AgentNotificationSchema = z.object({
     summary: z.string().min(1).max(500),
     status: z.enum(['completed', 'failed', 'killed']).optional(),
   });
   
   type AgentNotification = z.infer<typeof AgentNotificationSchema>;
   ```

3. **支持更多状态**：
   ```typescript
   function getStatusColor(status: string | null): TextProps['color'] {
     switch (status) {
       case 'completed': return 'success';
       case 'failed': return 'error';
       case 'killed': return 'warning';
       case 'running': return 'info';      // 新增
       case 'pending': return 'dim';       // 新增
       default: return 'text';
     }
   }
   ```

4. **图标差异化**：
   ```typescript
   // 不同状态使用不同图标
   function getStatusIcon(status: string | null): string {
     switch (status) {
       case 'completed': return figures.tick;      // ✓
       case 'failed': return figures.cross;        // ✗
       case 'killed': return figures.warning;      // ⚠
       default: return BLACK_CIRCLE;
     }
   }
   ```

5. **时间戳支持**：
   ```typescript
   // 添加可选的时间戳显示
   const timestamp = extractTag(text, 'timestamp');
   {timestamp && (
     <Text dimColor> ({formatRelativeTime(new Date(timestamp))})</Text>
   )}
   ```

6. **国际化支持**：
   ```typescript
   const statusLabels: Record<string, string> = {
     completed: t('agent.status.completed'),
     failed: t('agent.status.failed'),
     killed: t('agent.status.killed'),
   };
   ```

### 6.4 测试建议

- **单元测试**：
  - 各状态颜色映射测试
  - `extractTag` 边界情况测试（空标签、嵌套标签、多行内容）
  - 无 summary 时返回 null 的测试

- **集成测试**：
  - 与 AgentTool 的端到端测试
  - 消息渲染流程测试

- **视觉测试**：
  - 不同状态下的颜色对比度验证
  - 长 summary 文本的截断处理

### 6.5 相关代码参考

**类似组件对比**：

| 组件 | 用途 | 标签 |
|------|------|------|
| UserAgentNotificationMessage | 代理状态通知 | `<summary>`, `<status>` |
| UserBashInputMessage | Bash 输入显示 | `<bash-input>` |
| UserMemoryInputMessage | 内存输入显示 | `<memory>` |

这些组件共享相同的 `extractTag` 解析模式，可以考虑抽象为统一的解析钩子。
