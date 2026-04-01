# 研究文档：src/utils/todo/types.ts

## 1. 场景与职责

### 1.1 文件定位
`src/utils/todo/types.ts` 是 Claude Code 项目中负责**待办事项（Todo）数据类型定义**的核心类型声明文件。它定义了基于 Zod 的运行时验证 Schema 以及对应的 TypeScript 类型，用于整个应用的待办事项数据流转。

### 1.2 使用场景

该文件主要在以下场景中被使用：

1. **会话任务管理**：主会话和子代理（subagent）使用 TodoWrite 工具创建、更新和跟踪任务列表
2. **远程代理任务**：RemoteAgentTask 在云端执行时维护 todoList 状态，支持远程会话的任务追踪
3. **会话恢复**：从 transcript 中提取并恢复历史 todo 列表状态
4. **附件提醒系统**：当模型长时间未更新 todo 列表时，通过附件（attachment）机制提醒模型

### 1.3 架构角色

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用架构中的位置                          │
├─────────────────────────────────────────────────────────────────┤
│  src/utils/todo/types.ts (本文件)                               │
│     ├── 定义: TodoItemSchema, TodoListSchema, TodoStatusSchema   │
│     ├── 导出: TodoItem, TodoList 类型                            │
│     └── 依赖: lazySchema (延迟初始化), zod (验证)                │
├─────────────────────────────────────────────────────────────────┤
│  调用方                                                          │
│     ├── TodoWriteTool (src/tools/TodoWriteTool/)                │
│     │   └── 工具实现，处理 todo 列表的 CRUD                      │
│     ├── AppStateStore (src/state/AppStateStore.ts)              │
│     │   └── 全局状态存储 todos: { [agentId: string]: TodoList }  │
│     ├── attachments.ts (src/utils/attachments.ts)               │
│     │   └── 生成 todo_reminder 附件                            │
│     ├── sessionRestore.ts (src/utils/sessionRestore.ts)         │
│     │   └── 从 transcript 恢复 todo 状态                        │
│     ├── RemoteAgentTask (src/tasks/RemoteAgentTask/)            │
│     │   └── 远程代理任务状态中的 todoList 字段                   │
│     └── remoteSession.ts (src/utils/background/remote/)         │
│         └── BackgroundRemoteSession 类型定义                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 | 实现方式 |
|--------|------|----------|
| **TodoStatusSchema** | 定义任务状态枚举 | `z.enum(['pending', 'in_progress', 'completed'])` |
| **TodoItemSchema** | 定义单个待办项结构 | 包含 content, status, activeForm 字段 |
| **TodoListSchema** | 定义待办列表结构 | `z.array(TodoItemSchema())` |
| **类型推断** | 从 Schema 导出 TypeScript 类型 | `z.infer<ReturnType<typeof ...Schema>>` |
| **延迟初始化** | 避免模块加载时的循环依赖 | 使用 `lazySchema` 包装 |

### 2.2 字段设计意图

```typescript
// TodoItem 字段说明
{
  content: string     // 任务描述（祈使语气，如 "Run tests"）
  status: TodoStatus  // 状态：pending/in_progress/completed
  activeForm: string  // 执行中描述（现在进行时，如 "Running tests"）
}
```

- **content**: 用于展示任务内容，采用祈使语气描述需要做什么
- **activeForm**: 用于 UI 展示（如 spinner），采用现在进行时描述正在做什么
- **status**: 三态状态机，支持基本的任务生命周期管理

### 2.3 与 Task V2 的关系

该文件定义的 Todo 类型是**传统版本（V1）**的待办系统。项目同时存在新的 Task 系统（V2，定义在 `src/utils/tasks.ts`）：

- **Todo V1**: 基于内存（AppState），通过 TodoWrite 工具管理，适用于简单场景
- **Task V2**: 基于文件系统持久化，支持任务依赖、认领、阻塞等复杂工作流

两者通过 `isTodoV2Enabled()` 函数进行功能切换：
```typescript
// src/utils/tasks.ts
export function isTodoV2Enabled(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_TASKS)) {
    return true
  }
  return !getIsNonInteractiveSession()  // 交互模式默认启用 V2
}
```

---

## 3. 具体技术实现

### 3.1 延迟 Schema 初始化模式

```typescript
import { z } from 'zod/v4'
import { lazySchema } from '../lazySchema.js'

const TodoStatusSchema = lazySchema(() =>
  z.enum(['pending', 'in_progress', 'completed']),
)
```

**设计原因**：
- 避免模块加载时的循环依赖问题
- 延迟 Zod Schema 构造到首次访问时
- 缓存机制确保多次调用返回同一实例

**lazySchema 实现**（`src/utils/lazySchema.ts`）：
```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

### 3.2 类型导出模式

```typescript
export const TodoItemSchema = lazySchema(() =>
  z.object({
    content: z.string().min(1, 'Content cannot be empty'),
    status: TodoStatusSchema(),
    activeForm: z.string().min(1, 'Active form cannot be empty'),
  }),
)
export type TodoItem = z.infer<ReturnType<typeof TodoItemSchema>>
```

注意类型推断使用 `ReturnType<typeof TodoItemSchema>` 因为 `TodoItemSchema` 是一个函数（lazySchema 返回的工厂函数）。

### 3.3 数据验证规则

| 字段 | 规则 | 错误信息 |
|------|------|----------|
| content | min(1) | "Content cannot be empty" |
| status | enum | 必须是 pending/in_progress/completed |
| activeForm | min(1) | "Active form cannot be empty" |

### 3.4 关键流程：Todo 状态恢复

```typescript
// src/utils/sessionRestore.ts
function extractTodosFromTranscript(messages: Message[]): TodoList {
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i]
    if (msg?.type !== 'assistant') continue
    const toolUse = msg.message.content.find(
      block => block.type === 'tool_use' && block.name === TODO_WRITE_TOOL_NAME,
    )
    if (!toolUse || toolUse.type !== 'tool_use') continue
    const input = toolUse.input
    if (input === null || typeof input !== 'object') return []
    const parsed = TodoListSchema().safeParse(
      (input as Record<string, unknown>).todos,
    )
    return parsed.success ? parsed.data : []
  }
  return []
}
```

流程说明：
1. 从消息列表末尾向前遍历（找最新的 TodoWrite）
2. 查找 assistant 消息中的 TodoWrite tool_use 块
3. 提取 input.todos 字段
4. 使用 `TodoListSchema().safeParse()` 进行验证
5. 验证成功返回解析后的数据，失败返回空数组

### 3.5 关键流程：Todo 提醒附件生成

```typescript
// src/utils/attachments.ts
async function getTodoReminderAttachments(
  messages: Message[] | undefined,
  toolUseContext: ToolUseContext,
): Promise<Attachment[]> {
  // 检查 TodoWrite 工具是否可用
  if (!toolUseContext.options.tools.some(t =>
    toolMatchesName(t, TODO_WRITE_TOOL_NAME)
  )) {
    return []
  }

  // 计算距离上次写入和上次提醒的轮数
  const { turnsSinceLastTodoWrite, turnsSinceLastReminder } =
    getTodoReminderTurnCounts(messages)

  // 触发条件：10轮未写入 && 10轮未提醒
  if (
    turnsSinceLastTodoWrite >= TODO_REMINDER_CONFIG.TURNS_SINCE_WRITE &&
    turnsSinceLastReminder >= TODO_REMINDER_CONFIG.TURNS_BETWEEN_REMINDERS
  ) {
    const todoKey = toolUseContext.agentId ?? getSessionId()
    const appState = toolUseContext.getAppState()
    const todos = appState.todos[todoKey] ?? []
    return [{
      type: 'todo_reminder',
      content: todos,
      itemCount: todos.length,
    }]
  }
  return []
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
src/utils/todo/types.ts
    │
    ├── 导入 ─────────────────────────────┐
    │   ├── zod/v4                        │
    │   └── ../lazySchema.js              │
    │                                      │
    ├── 被导入 ───────────────────────────┤
        ├── src/tools/TodoWriteTool/TodoWriteTool.ts
        │   ├── 使用: TodoListSchema (input/output schema)
        │   └── 功能: 工具实现
        │
        ├── src/state/AppStateStore.ts
        │   ├── 使用: TodoList (类型)
        │   └── 功能: AppState.todos 字段类型定义
        │
        ├── src/utils/attachments.ts
        │   ├── 使用: TodoList (类型), TodoListSchema (验证)
        │   └── 功能: todo_reminder 附件生成
        │
        ├── src/utils/sessionRestore.ts
        │   ├── 使用: TodoList, TodoListSchema
        │   └── 功能: 从 transcript 恢复 todo 状态
        │
        ├── src/tasks/RemoteAgentTask/RemoteAgentTask.tsx
        │   ├── 使用: TodoList (类型)
        │   └── 功能: RemoteAgentTaskState.todoList 字段
        │
        └── src/utils/background/remote/remoteSession.ts
            ├── 使用: TodoList (类型)
            └── 功能: BackgroundRemoteSession.todoList 字段
```

### 4.2 核心代码路径

| 路径 | 功能 | 关键代码 |
|------|------|----------|
| `TodoWriteTool.call()` | 更新 todo 列表 | `appState.todos[todoKey] = newTodos` |
| `getTodoReminderAttachments()` | 生成提醒附件 | 检查 turnsSinceLastTodoWrite |
| `extractTodosFromTranscript()` | 恢复 todo 状态 | 遍历 messages 找 TodoWrite |
| `RemoteAgentTaskState` | 远程任务状态 | `todoList: TodoList` 字段 |

### 4.3 配置常量

```typescript
// src/utils/attachments.ts
export const TODO_REMINDER_CONFIG = {
  TURNS_SINCE_WRITE: 10,      // 10轮未写入触发提醒
  TURNS_BETWEEN_REMINDERS: 10, // 提醒间隔10轮
} as const
```

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `zod/v4` | npm | Schema 定义和运行时验证 |
| `lazySchema` | `src/utils/lazySchema.ts` | 延迟初始化模式 |

### 5.2 运行时依赖（调用方）

| 调用方 | 依赖类型 | 说明 |
|--------|----------|------|
| TodoWriteTool | Schema + 类型 | 工具输入输出验证 |
| AppStateStore | 类型 | 全局状态类型定义 |
| attachments.ts | Schema + 类型 | 附件生成和验证 |
| sessionRestore.ts | Schema + 类型 | 会话恢复验证 |
| RemoteAgentTask | 类型 | 任务状态定义 |
| remoteSession.ts | 类型 | 背景会话类型定义 |

### 5.3 数据流

```
用户/模型 ──TodoWrite 工具调用──► TodoWriteTool
                                      │
                                      ▼
                              AppState.todos[agentId]
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              附件系统(提醒)      会话恢复系统       远程任务系统
         getTodoReminderAttachments  extractTodosFromTranscript  RemoteAgentTaskState
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险 1：与 Task V2 的重复
- **问题**: Todo 系统（V1）和 Task 系统（V2）功能重叠，维护成本高
- **现状**: `isTodoV2Enabled()` 控制切换，但两者代码并存
- **影响**: 新功能需要同时考虑两套系统

#### 风险 2：无持久化（V1）
- **问题**: Todo V1 仅存储在 AppState 内存中
- **现状**: 依赖 `sessionRestore.ts` 从 transcript 恢复
- **影响**: 程序崩溃时可能丢失未写入 transcript 的 todo 状态

#### 风险 3：Schema 版本兼容性
- **问题**: Zod v4 是相对较新的版本
- **现状**: 项目使用 `zod/v4` 子路径导入
- **影响**: 升级 Zod 大版本时需要验证 Schema 兼容性

### 6.2 边界情况

| 场景 | 行为 | 代码位置 |
|------|------|----------|
| 空 todo 列表 | 返回空数组 `[]` | TodoWriteTool.call() line 70 |
| 所有任务完成 | 清空列表（`allDone ? [] : todos`） | TodoWriteTool.call() line 70 |
| Schema 验证失败 | 返回空数组 | sessionRestore.ts line 90 |
| 子代理 | 使用 agentId 作为 key | TodoWriteTool.call() line 67 |
| 主会话 | 使用 sessionId 作为 key | TodoWriteTool.call() line 67 |

### 6.3 改进建议

#### 建议 1：统一任务系统
- 逐步迁移所有使用 Todo V1 的代码到 Task V2
- 移除 `isTodoV2Enabled()` 开关，统一使用文件持久化的 Task 系统
- 好处：单一真相源，减少维护负担

#### 建议 2：增强类型安全
```typescript
// 当前：使用 string 表示状态
status: z.enum(['pending', 'in_progress', 'completed'])

// 建议：使用 const 断言 + 联合类型
export const TODO_STATUSES = ['pending', 'in_progress', 'completed'] as const
export type TodoStatus = typeof TODO_STATUSES[number]
```

#### 建议 3：添加元数据支持
```typescript
// 当前：固定字段
export const TodoItemSchema = lazySchema(() =>
  z.object({
    content: z.string().min(1),
    status: TodoStatusSchema(),
    activeForm: z.string().min(1),
  }),
)

// 建议：添加可选元数据字段支持扩展
export const TodoItemSchema = lazySchema(() =>
  z.object({
    content: z.string().min(1),
    status: TodoStatusSchema(),
    activeForm: z.string().min(1),
    metadata: z.record(z.unknown()).optional(), // 扩展字段
    createdAt: z.number().optional(),
    updatedAt: z.number().optional(),
  }),
)
```

#### 建议 4：Schema 版本控制
```typescript
// 建议添加版本字段以便未来迁移
export const TodoListSchemaV1 = lazySchema(() =>
  z.object({
    version: z.literal(1).default(1),
    items: z.array(TodoItemSchema()),
  }),
)
```

### 6.4 测试建议

当前未发现针对该文件的直接单元测试。建议添加：

1. **Schema 验证测试**: 测试各种边界输入的验证行为
2. **类型兼容性测试**: 确保导出的 TypeScript 类型与运行时 Schema 一致
3. **迁移测试**: 测试从 transcript 恢复时的容错能力

---

## 7. 附录

### 7.1 完整代码

```typescript
import { z } from 'zod/v4'
import { lazySchema } from '../lazySchema.js'

const TodoStatusSchema = lazySchema(() =>
  z.enum(['pending', 'in_progress', 'completed']),
)

export const TodoItemSchema = lazySchema(() =>
  z.object({
    content: z.string().min(1, 'Content cannot be empty'),
    status: TodoStatusSchema(),
    activeForm: z.string().min(1, 'Active form cannot be empty'),
  }),
)
export type TodoItem = z.infer<ReturnType<typeof TodoItemSchema>>

export const TodoListSchema = lazySchema(() => z.array(TodoItemSchema()))
export type TodoList = z.infer<ReturnType<typeof TodoListSchema>>
```

### 7.2 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| `src/utils/todo/types.ts` | 本文件，类型定义 |
| `src/utils/lazySchema.ts` | 延迟初始化工具 |
| `src/tools/TodoWriteTool/TodoWriteTool.ts` | TodoWrite 工具实现 |
| `src/tools/TodoWriteTool/constants.ts` | 工具常量 |
| `src/tools/TodoWriteTool/prompt.ts` | 工具提示词 |
| `src/state/AppStateStore.ts` | 全局状态存储 |
| `src/utils/attachments.ts` | 附件生成系统 |
| `src/utils/sessionRestore.ts` | 会话恢复逻辑 |
| `src/utils/tasks.ts` | Task V2 系统 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 远程代理任务 |
| `src/utils/background/remote/remoteSession.ts` | 背景远程会话 |

---

*文档生成时间: 2026-04-01*  
*研究范围: 代码、类型定义、调用方实现*  
*排除范围: README、文档、checklist、todo 文件*
