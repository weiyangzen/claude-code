# TodoWriteTool.ts 研究文档

## 场景与职责

`TodoWriteTool` 是 Claude Code CLI 中用于管理结构化任务列表（Todo List）的核心工具。它允许 AI 助手在复杂的多步骤任务中创建、更新和跟踪任务进度，向用户展示工作进展和整体任务状态。

该工具主要服务于以下场景：
1. **复杂多步骤任务管理** - 当任务需要 3 个或以上不同步骤时
2. **用户可见的进度跟踪** - 帮助用户理解任务进展和整体进度
3. **会话状态持久化** - 在会话恢复时从 transcript 中重建任务列表

**重要说明**：该工具是 Todo V1 实现。当 `isTodoV2Enabled()` 返回 true 时（即交互式会话中），此工具会被禁用，由基于文件的 Task V2 系统（TaskCreateTool/TaskUpdateTool/TaskListTool 等）取代。

## 功能点目的

### 1. 任务列表管理
- 允许 AI 创建、更新和维护结构化的任务列表
- 支持三种任务状态：`pending`（待处理）、`in_progress`（进行中）、`completed`（已完成）
- 每个任务包含内容描述（content）、状态（status）和活动形式（activeForm）

### 2. 验证代理提醒（Verification Agent Nudge）
- 当主线程代理完成 3+ 个任务且没有验证步骤时，自动提醒调用验证代理
- 通过 GrowthBook 功能标志 `VERIFICATION_AGENT` 和 `tengu_hive_evidence` 控制
- 防止 AI 在没有验证的情况下自行标记任务为部分完成（PARTIAL）

### 3. 会话状态恢复
- 在会话恢复时，从 transcript 中提取最后一次 TodoWrite 调用恢复任务状态
- 存储在 `AppState.todos` 中，以 `agentId` 或 `sessionId` 为键

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式（Input Schema）
{
  todos: TodoList  // 更新的任务列表
}

// 输出模式（Output Schema）
{
  oldTodos: TodoList,      // 更新前的任务列表
  newTodos: TodoList,      // 更新后的任务列表
  verificationNudgeNeeded: boolean  // 是否需要验证提醒
}

// 任务项结构（来自 types.ts）
{
  content: string,      // 任务内容（祈使句形式，如 "Run tests"）
  status: 'pending' | 'in_progress' | 'completed',
  activeForm: string    // 进行时形式（如 "Running tests"）
}
```

### 核心流程

1. **工具调用流程** (`call` 方法)：
   ```
   1. 获取 AppState 和 todoKey（agentId ?? sessionId）
   2. 读取 oldTodos（当前存储的任务列表）
   3. 检查是否所有任务都已完成 (allDone)
   4. 如果全部完成，newTodos 设为空数组（清理已完成列表）
   5. 检查是否需要验证提醒（verificationNudgeNeeded）
      - VERIFICATION_AGENT 功能标志启用
      - tengu_hive_evidence 为 true
      - 不是子代理（!context.agentId）
      - 所有任务完成且任务数 >= 3
      - 没有任务内容包含 "verif" 字样
   6. 更新 AppState.todos
   7. 返回结果
   ```

2. **验证提醒逻辑**：
   ```typescript
   if (
     feature('VERIFICATION_AGENT') &&
     getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
     !context.agentId &&  // 仅主线程
     allDone &&
     todos.length >= 3 &&
     !todos.some(t => /verif/i.test(t.content))  // 无验证步骤
   ) {
     verificationNudgeNeeded = true
   }
   ```

3. **结果渲染** (`mapToolResultToToolResultBlockParam`)：
   - 基础消息："Todos have been modified successfully..."
   - 验证提醒（如需要）：提示调用 verification 类型的子代理

### 工具配置

```typescript
{
  name: 'TodoWrite',
  searchHint: 'manage the session task checklist',
  maxResultSizeChars: 100_000,
  strict: true,  // 启用严格模式
  shouldDefer: true,  // 延迟加载，需通过 ToolSearch 查找
  isEnabled: () => !isTodoV2Enabled(),  // V2 启用时禁用
  userFacingName: () => ''  // 空字符串表示不在 UI 中显示特定名称
}
```

## 关键代码路径与文件引用

### 本文件关键路径

| 行号 | 代码 | 说明 |
|------|------|------|
| 13-17 | `inputSchema` | 使用 lazySchema 延迟加载 Zod schema |
| 20-27 | `outputSchema` | 定义输出结构，包含 verificationNudgeNeeded |
| 31-115 | `buildTool` | 工具定义和实现 |
| 65-103 | `call` | 核心调用逻辑 |
| 76-86 | verification nudge | 验证代理提醒逻辑 |
| 88-94 | `setAppState` | 更新全局状态 |
| 104-114 | `mapToolResultToToolResultBlockParam` | 结果渲染 |

### 依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/utils/todo/types.ts` | TodoItemSchema, TodoListSchema 定义 |
| `src/utils/tasks.ts` | isTodoV2Enabled() 检查 |
| `src/Tool.ts` | buildTool, ToolDef 类型 |
| `src/utils/lazySchema.ts` | lazySchema 延迟加载工具 |
| `src/bootstrap/state.ts` | getSessionId() 获取会话 ID |
| `src/services/analytics/growthbook.ts` | getFeatureValue_CACHED_MAY_BE_STALE 功能标志 |
| `src/tools/AgentTool/constants.ts` | VERIFICATION_AGENT_TYPE |

### 调用方

| 文件路径 | 用途 |
|----------|------|
| `src/utils/sessionRestore.ts` | 会话恢复时从 transcript 提取 todos |
| `src/constants/tools.ts` | 工具注册 |
| `src/tools.ts` | 工具集合 |

## 依赖与外部交互

### 依赖模块

1. **Zod Schema 验证** (`zod/v4`)
   - 使用 `z.strictObject` 严格验证输入
   - `TodoListSchema` 验证任务列表结构

2. **功能标志系统** (GrowthBook)
   - `feature('VERIFICATION_AGENT')` - 验证代理功能开关
   - `getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence')` - 证据收集功能

3. **应用状态管理** (AppState)
   - `context.getAppState()` - 读取当前状态
   - `context.setAppState()` - 更新 todos
   - 存储结构：`todos: { [agentId: string]: TodoList }`

4. **会话管理**
   - `getSessionId()` - 获取当前会话 ID 作为默认键
   - `context.agentId` - 子代理 ID（如存在）

### 外部交互

- **无直接文件系统交互** - 所有状态通过 AppState 管理
- **无网络请求** - 纯内存状态操作
- **Transcript 恢复** - 通过 `extractTodosFromTranscript` 在会话恢复时重建状态

## 风险、边界与改进建议

### 已知风险

1. **V1/V2 切换风险**
   - 工具在 `isTodoV2Enabled()` 返回 true 时被禁用
   - 但代码中仍保留，可能导致混淆
   - 建议：明确标记为 deprecated，规划移除路径

2. **验证提醒误报**
   - 正则 `/verif/i` 可能匹配到非验证相关的任务（如 "verify" 出现在普通任务描述中）
   - 建议：使用更精确的关键词或结构化标记

3. **状态清理逻辑**
   - 当所有任务完成时，列表被清空（`allDone ? [] : todos`）
   - 这可能导致历史任务记录丢失
   - 建议：考虑保留历史记录或提供归档机制

4. **并发问题**
   - 多个子代理同时更新同一 agentId 的 todos 可能产生竞态条件
   - 当前实现依赖 React 的 setState 合并，非原子操作

### 边界情况

1. **空任务列表**
   - `todos: []` 会清空该 agentId 的任务列表
   - 这是有效的清理操作

2. **子代理使用**
   - 子代理使用自身的 `agentId` 作为键
   - 主线程使用 `sessionId`
   - 两者任务列表相互隔离

3. **会话恢复**
   - 仅在 `!isTodoV2Enabled()` 时从 transcript 恢复
   - 交互式会话使用文件存储的 V2 任务系统

### 改进建议

1. **代码清理**
   - 既然 V2 已取代 V1，考虑完全移除此工具
   - 或将其标记为 deprecated 并添加迁移警告

2. **验证提醒增强**
   - 使用更结构化的方式标记验证步骤
   - 考虑添加任务类型字段（type: 'verification' | 'implementation' | 'review'）

3. **历史记录保留**
   - 考虑保留已完成的任务历史
   - 提供查询已完成任务的能力

4. **类型安全**
   - 当前 `userFacingName` 返回空字符串
   - 考虑返回更有意义的名称或 null

5. **文档完善**
   - prompt.ts 中的使用说明非常详细
   - 但代码注释可以补充更多实现细节
