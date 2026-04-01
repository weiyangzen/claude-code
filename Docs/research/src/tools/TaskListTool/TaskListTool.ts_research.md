# TaskListTool.ts 研究文档

## 场景与职责

TaskListTool 是 Claude Code 任务管理系统（Todo V2）的核心工具之一，负责**列出任务列表中的所有任务**。它是任务管理工具套件（TaskCreateTool、TaskGetTool、TaskUpdateTool、TaskListTool）的读取端入口，为 AI Agent 提供查看项目任务全景的能力。

主要使用场景：
1. **任务发现**：Agent 需要查看当前有哪些待处理任务
2. **进度检查**：查看项目整体完成状态
3. **依赖分析**：识别被阻塞的任务及其依赖关系
4. ** teammate 工作流**：多 Agent 协作时，teammate 使用此工具发现可认领的任务

## 功能点目的

### 1. 任务列表查询
- 从文件系统读取当前任务列表中的所有任务
- 过滤掉内部任务（`_internal` metadata 标记）
- 返回任务的完整状态信息

### 2. 依赖关系处理
- 自动过滤已解决任务的阻塞关系（`blockedBy` 中已完成的任务会被移除）
- 帮助 Agent 识别真正可执行的任务

### 3. 结果格式化
- 将任务列表格式化为人类可读的文本输出
- 包含任务 ID、状态、主题、负责人和阻塞信息

### 4. 延迟加载支持
- 作为延迟加载工具（`shouldDefer: true`），通过 ToolSearch 机制按需加载
- 减少初始系统提示词大小

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式 - 空对象（无需参数）
const inputSchema = z.strictObject({})  // 不接受任何参数

// 输出模式
const outputSchema = z.object({
  tasks: z.array(
    z.object({
      id: z.string(),
      subject: z.string(),
      status: TaskStatusSchema(),  // 'pending' | 'in_progress' | 'completed'
      owner: z.string().optional(),
      blockedBy: z.array(z.string()),
    }),
  ),
})
```

### 核心流程

```
call() 执行流程:
1. 获取任务列表 ID（getTaskListId()）
2. 列出所有任务（listTasks(taskListId)）
3. 过滤内部任务（!t.metadata?._internal）
4. 构建已解决任务 ID 集合
5. 映射任务数据，过滤 blockedBy 中已解决的任务
6. 返回格式化结果
```

### 关键代码路径

**任务列表获取与处理**（行 65-90）：
```typescript
async call() {
  const taskListId = getTaskListId()
  
  // 获取所有非内部任务
  const allTasks = (await listTasks(taskListId)).filter(
    t => !t.metadata?._internal,
  )

  // 构建已解决任务集合用于过滤
  const resolvedTaskIds = new Set(
    allTasks.filter(t => t.status === 'completed').map(t => t.id),
  )

  // 映射并过滤阻塞关系
  const tasks = allTasks.map(task => ({
    id: task.id,
    subject: task.subject,
    status: task.status,
    owner: task.owner,
    blockedBy: task.blockedBy.filter(id => !resolvedTaskIds.has(id)),
  }))

  return { data: { tasks } }
}
```

**结果格式化**（行 91-115）：
```typescript
mapToolResultToToolResultBlockParam(content, toolUseID) {
  const { tasks } = content as Output
  if (tasks.length === 0) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: 'No tasks found',
    }
  }

  const lines = tasks.map(task => {
    const owner = task.owner ? ` (${task.owner})` : ''
    const blocked =
      task.blockedBy.length > 0
        ? ` [blocked by ${task.blockedBy.map(id => `#${id}`).join(', ')}]`
        : ''
    return `#${task.id} [${task.status}] ${task.subject}${owner}${blocked}`
  })

  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: lines.join('\n'),
  }
}
```

### 工具定义属性

```typescript
{
  name: TASK_LIST_TOOL_NAME,  // 'TaskList'
  searchHint: 'list all tasks',
  maxResultSizeChars: 100_000,
  shouldDefer: true,          // 延迟加载
  isEnabled: isTodoV2Enabled, // 基于环境/模式启用
  isConcurrencySafe: true,    // 并发安全（只读）
  isReadOnly: true,           // 纯读取操作
  userFacingName: 'TaskList',
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `z` | `zod/v4` | Schema 验证 |
| `buildTool` | `../../Tool.js` | 工具构建工厂 |
| `lazySchema` | `../../utils/lazySchema.js` | 延迟 Schema 构造 |
| `getTaskListId` | `../../utils/tasks.js` | 获取任务列表 ID |
| `isTodoV2Enabled` | `../../utils/tasks.js` | 功能开关检查 |
| `listTasks` | `../../utils/tasks.js` | 读取任务列表 |
| `TaskStatusSchema` | `../../utils/tasks.js` | 状态枚举 Schema |
| `TASK_LIST_TOOL_NAME` | `./constants.ts` | 工具名称常量 |
| `DESCRIPTION`, `getPrompt` | `./prompt.ts` | 提示词内容 |

### 调用方

1. **tools.ts**（行 85, 219）：注册到全局工具列表
2. **inProcessRunner.ts**（行 54, 991）：Teammate 工具白名单
3. **classifierDecision.ts**（行 14, 71）：YOLO 自动模式白名单

### 任务存储系统

任务数据存储在文件系统中：
- 存储路径：`~/.claude/tasks/{taskListId}/{taskId}.json`
- 通过 `listTasks()` 读取目录下所有 `.json` 文件
- 每个任务是一个独立的 JSON 文件

## 风险、边界与改进建议

### 风险点

1. **文件系统依赖**
   - 任务列表依赖文件系统 I/O，在高并发场景下可能遇到锁竞争
   - `listTasks()` 使用 `proper-lockfile` 进行并发控制，但大量任务时性能可能下降

2. **任务数量上限**
   - `maxResultSizeChars: 100_000` 限制输出大小
   - 如果任务数量极多（数千个），可能触发截断

3. **无分页机制**
   - 当前实现一次性返回所有任务
   - 大量任务时输出可能过长

### 边界情况

1. **空任务列表**：返回 `"No tasks found"`
2. **任务列表 ID 解析**：通过 `getTaskListId()` 按优先级解析（环境变量 > teammate 上下文 > team 名称 > session ID）
3. **内部任务过滤**：`_internal` metadata 标记的任务对用户不可见
4. **阻塞关系过滤**：自动隐藏已解决任务的阻塞关系

### 改进建议

1. **添加分页支持**
   ```typescript
   // 建议添加可选参数
   const inputSchema = z.strictObject({
     limit: z.number().optional(),
     offset: z.number().optional(),
     status: z.enum(['pending', 'in_progress', 'completed']).optional(),
   })
   ```

2. **添加过滤和排序**
   - 按状态过滤（只显示 pending）
   - 按负责人过滤
   - 按 ID 或优先级排序

3. **性能优化**
   - 考虑缓存任务列表（短时间 TTL）
   - 添加任务数量上限保护

4. **增强输出**
   - 添加任务统计摘要（总任务数、完成数、待处理数）
   - 支持 JSON 格式输出（便于程序化使用）

### 相关文件引用

- 实现：`src/tools/TaskListTool/TaskListTool.ts`
- 常量：`src/tools/TaskListTool/constants.ts`
- 提示词：`src/tools/TaskListTool/prompt.ts`
- 任务工具函数：`src/utils/tasks.ts`
- 工具注册：`src/tools.ts`
- 权限分类器：`src/utils/permissions/classifierDecision.ts`
- Teammate 运行器：`src/utils/swarm/inProcessRunner.ts`
