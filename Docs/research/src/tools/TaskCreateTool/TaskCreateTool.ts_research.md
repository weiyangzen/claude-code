# TaskCreateTool.ts 研究文档

## 场景与职责

TaskCreateTool 是 Claude Code 任务管理系统的核心工具之一，负责在任务列表中创建新任务。它是 TodoV2 任务系统的一部分，用于替代旧版的 TodoWrite 工具，提供更结构化的任务管理能力。

主要使用场景：
- **复杂多步骤任务**：当任务需要 3 个或更多不同步骤时
- **计划模式（Plan Mode）**：在使用计划模式时创建任务列表跟踪工作
- **用户明确要求**：用户直接要求使用待办列表时
- **多任务输入**：用户提供需要完成的多个事项列表时
- **任务跟踪**：帮助用户理解任务进度和整体进展

## 功能点目的

### 1. 任务创建
- 创建带有主题（subject）、描述（description）和可选活动形式（activeForm）的任务
- 所有新任务默认状态为 `pending`（待处理）
- 支持附加任意元数据（metadata）到任务

### 2. 钩子集成（Hooks Integration）
- 在任务创建时触发 `TaskCreated` 钩子
- 钩子可以阻止任务创建（通过返回阻塞错误）
- 如果钩子阻塞，自动删除已创建的任务并抛出错误

### 3. UI 自动展开
- 创建任务后自动展开任务列表面板（`expandedView: 'tasks'`）
- 帮助用户立即看到新创建的任务

### 4. Agent Swarms 支持
- 支持在团队环境中创建任务
- 获取当前 Agent 名称和团队名称传递给钩子系统

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式（Input Schema）
{
  subject: string;        // 任务标题
  description: string;    // 任务描述
  activeForm?: string;    // 进行时的显示文本（如 "Running tests"）
  metadata?: Record<string, unknown>; // 任意元数据
}

// 输出模式（Output Schema）
{
  task: {
    id: string;      // 任务唯一标识
    subject: string; // 任务标题
  }
}
```

### 核心流程

1. **工具调用入口** (`call` 方法)
   ```typescript
   async call({ subject, description, activeForm, metadata }, context) {
     // 1. 创建任务文件
     const taskId = await createTask(getTaskListId(), { ... })
     
     // 2. 执行 TaskCreated 钩子
     const generator = executeTaskCreatedHooks(...)
     for await (const result of generator) {
       if (result.blockingError) {
         blockingErrors.push(...)
       }
     }
     
     // 3. 处理钩子阻塞
     if (blockingErrors.length > 0) {
       await deleteTask(getTaskListId(), taskId)
       throw new Error(blockingErrors.join('\n'))
     }
     
     // 4. 展开任务面板
     context.setAppState(prev => ({ ...prev, expandedView: 'tasks' }))
     
     // 5. 返回结果
     return { data: { task: { id: taskId, subject } } }
   }
   ```

2. **任务创建流程** (`createTask` in `src/utils/tasks.ts`)
   - 使用文件锁（proper-lockfile）防止并发冲突
   - 读取当前最高任务 ID
   - 生成新 ID（最高 ID + 1）
   - 写入 JSON 文件到 `~/.claude/tasks/{taskListId}/{id}.json`
   - 触发 `notifyTasksUpdated()` 通知 UI 更新

3. **钩子执行流程** (`executeTaskCreatedHooks` in `src/utils/hooks.ts`)
   - 构建 `TaskCreatedHookInput` 对象
   - 调用 `executeHooks` 执行匹配的钩子
   - 支持多种钩子类型：command、prompt、agent、http、callback、function

### 关键配置项

| 属性 | 值 | 说明 |
|------|-----|------|
| `name` | `'TaskCreate'` | 工具标识名 |
| `searchHint` | `'create a task in the task list'` | 工具搜索提示 |
| `maxResultSizeChars` | `100_000` | 最大结果大小 |
| `shouldDefer` | `true` | 延迟加载工具 |
| `isEnabled` | `isTodoV2Enabled()` | 基于 TodoV2 开关 |
| `isConcurrencySafe` | `true` | 支持并发执行 |

## 关键代码路径与文件引用

### 主要文件
- **当前文件**: `src/tools/TaskCreateTool/TaskCreateTool.ts`
- **常量定义**: `src/tools/TaskCreateTool/constants.ts`
- **提示文本**: `src/tools/TaskCreateTool/prompt.ts`

### 依赖文件
- **工具基础类**: `src/Tool.ts` - `buildTool`, `ToolDef` 类型
- **任务操作**: `src/utils/tasks.ts` - `createTask`, `deleteTask`, `getTaskListId`, `isTodoV2Enabled`
- **钩子系统**: `src/utils/hooks.ts` - `executeTaskCreatedHooks`, `getTaskCreatedHookMessage`
- **Agent 信息**: `src/utils/teammate.ts` - `getAgentName`, `getTeamName`
- **延迟 Schema**: `src/utils/lazySchema.ts` - `lazySchema`

### 调用关系图
```
TaskCreateTool.call
├── createTask (src/utils/tasks.ts)
│   ├── lockfile.lock (并发控制)
│   ├── writeFile (写入任务文件)
│   └── notifyTasksUpdated (UI 通知)
├── executeTaskCreatedHooks (src/utils/hooks.ts)
│   └── executeHooks (钩子执行引擎)
├── deleteTask (钩子阻塞时回滚)
└── setAppState (展开任务面板)
```

## 依赖与外部交互

### 外部依赖
1. **Zod** (`zod/v4`): 用于输入/输出 schema 验证
2. **proper-lockfile**: 文件锁实现，防止并发任务创建冲突

### 内部模块依赖
| 模块 | 用途 |
|------|------|
| `src/Tool.ts` | 工具定义基础设施 |
| `src/utils/tasks.ts` | 任务 CRUD 操作和文件存储 |
| `src/utils/hooks.ts` | 钩子执行框架 |
| `src/utils/teammate.ts` | Agent/团队信息获取 |
| `src/utils/lazySchema.ts` | 延迟初始化 Zod schema |

### 文件系统交互
- **任务存储目录**: `~/.claude/tasks/{taskListId}/`
- **任务文件格式**: `{id}.json`
- **锁文件**: `.lock`（用于并发控制）
- **高水位标记**: `.highwatermark`（防止 ID 重用）

### 环境变量依赖
- `CLAUDE_CODE_ENABLE_TASKS`: 强制启用任务系统
- `CLAUDE_CODE_TASK_LIST_ID`: 显式指定任务列表 ID
- `CLAUDE_CODE_TEAM_NAME`: 团队名称（tmux 队友模式）

## 风险、边界与改进建议

### 潜在风险

1. **并发冲突**
   - 风险：多个 Agent 同时创建任务可能导致 ID 冲突
   - 缓解：使用 proper-lockfile 进行文件级锁定
   - 注意：锁超时设置（30 次重试，5-100ms 退避）

2. **钩子阻塞导致的残留任务**
   - 风险：钩子阻塞后删除任务可能失败
   - 当前处理：先创建任务，钩子阻塞后删除
   - 边界：如果删除失败，可能留下孤儿任务

3. **文件系统权限**
   - 风险：无法写入 `~/.claude/tasks/` 目录
   - 影响：任务创建失败，抛出异常

4. **任务列表 ID 解析复杂性**
   - 优先级：环境变量 > 队友上下文 > 团队名称 > leaderTeamName > sessionId
   - 风险：ID 解析错误可能导致任务存储在错误位置

### 边界条件

1. **空任务列表**: 首次创建时自动创建目录
2. **最大任务 ID**: 使用高水位标记防止 ID 重用
3. **元数据大小**: 无明确限制，但受限于文件系统
4. **钩子超时**: 默认 10 分钟（`TOOL_HOOK_EXECUTION_TIMEOUT_MS`）

### 改进建议

1. **事务性任务创建**
   - 建议：使用临时文件 + 原子重命名，避免创建后删除
   - 实现：先写入 `.tmp-{id}.json`，确认钩子通过后再重命名

2. **批量任务创建**
   - 建议：支持一次创建多个任务，减少文件锁竞争
   - 场景：计划模式初始化时创建多个任务

3. **任务 ID 生成优化**
   - 建议：考虑使用 UUID 或雪花算法，避免文件锁
   - 权衡：可读性 vs 并发性能

4. **钩子失败处理增强**
   - 建议：支持部分成功（某些钩子失败不阻塞）
   - 配置：添加 `failPolicy` 参数控制失败行为

5. **元数据验证**
   - 建议：对 metadata 添加可选的 schema 验证
   - 用途：确保元数据符合预期格式

6. **任务模板支持**
   - 建议：支持从模板创建任务（预定义描述、依赖关系）
   - 场景：常见工作流的标准化

### 测试建议

1. **并发测试**: 模拟多个 Agent 同时创建任务
2. **钩子阻塞测试**: 验证钩子阻塞后的回滚逻辑
3. **文件权限测试**: 模拟只读文件系统
4. **边界测试**: 最大任务数、最大描述长度等
