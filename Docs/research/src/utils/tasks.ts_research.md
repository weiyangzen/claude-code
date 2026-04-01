# tasks.ts 研究文档

## 场景与职责

`tasks.ts` 是 Claude Code 的任务管理系统核心模块，实现了基于文件系统的任务存储、并发控制、状态管理和团队协作功能。该模块支持多进程并发访问（通过文件锁），适用于单个 Claude 实例和多 Agent 集群（swarm）场景。

## 功能点目的

### 任务生命周期管理
- **创建** (`createTask`): 生成唯一 ID，写入文件系统
- **读取** (`getTask`): 读取任务文件，支持状态迁移
- **更新** (`updateTask`): 更新任务字段，带文件锁保护
- **删除** (`deleteTask`): 删除任务，更新高水位标记，清理依赖关系
- **列表** (`listTasks`): 获取所有任务

### 并发控制
- **文件锁**: 使用 `proper-lockfile` 实现进程间互斥
- **高水位标记**: 防止 ID 重用（`.highwatermark` 文件）
- **重试策略**: 30 次重试，5-100ms 退避

### 任务依赖关系
- **阻塞关系** (`blockTask`): 建立任务间的阻塞关系
- **自动清理**: 删除任务时自动清理相关依赖

### 任务认领
- **原子认领** (`claimTask`): 带忙检查的原子认领
- **阻塞检测**: 检查未完成的阻塞任务
- **忙状态检查**: 防止 Agent 同时处理多个任务

### 团队集成
- **Agent 状态** (`getAgentStatuses`): 基于任务所有权计算 Agent 状态
- **任务释放** (`unassignTeammateTasks`): 队友退出时释放任务

## 具体技术实现

### 任务数据结构

```typescript
export const TaskSchema = lazySchema(() =>
  z.object({
    id: z.string(),
    subject: z.string(),
    description: z.string(),
    activeForm: z.string().optional(),  // 进行时形式（如 "Running tests"）
    owner: z.string().optional(),       // Agent ID
    status: TaskStatusSchema(),         // 'pending' | 'in_progress' | 'completed'
    blocks: z.array(z.string()),        // 此任务阻塞的任务 ID
    blockedBy: z.array(z.string()),     // 阻塞此任务的任务 ID
    metadata: z.record(z.string(), z.unknown()).optional(),
  }),
)
```

### 文件锁配置

```typescript
const LOCK_OPTIONS = {
  retries: {
    retries: 30,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

### 任务 ID 生成

```typescript
async function findHighestTaskId(taskListId: string): Promise<number> {
  const [fromFiles, fromMark] = await Promise.all([
    findHighestTaskIdFromFiles(taskListId),
    readHighWaterMark(taskListId),
  ])
  return Math.max(fromFiles, fromMark)
}

// 创建时：id = String(highestId + 1)
```

### 任务认领流程

```typescript
export async function claimTask(
  taskListId: string,
  taskId: string,
  claimantAgentId: string,
  options: ClaimTaskOptions = {},
): Promise<ClaimTaskResult> {
  // 1. 检查任务存在性（无锁）
  // 2. 获取任务级或列表级锁
  // 3. 重新读取任务状态（防止 TOCTOU）
  // 4. 检查：已被认领、已完成、被阻塞
  // 5. 如需忙检查，验证 Agent 无其他未完成任务
  // 6. 更新任务 owner
  // 7. 释放锁
}
```

### 任务列表 ID 解析

```typescript
export function getTaskListId(): string {
  // 优先级：
  // 1. CLAUDE_CODE_TASK_LIST_ID 环境变量
  // 2. In-process teammate 上下文
  // 3. CLAUDE_CODE_TEAM_NAME 环境变量
  // 4. Leader team name
  // 5. Session ID
  if (process.env.CLAUDE_CODE_TASK_LIST_ID) {
    return process.env.CLAUDE_CODE_TASK_LIST_ID
  }
  const teammateCtx = getTeammateContext()
  if (teammateCtx) {
    return teammateCtx.teamName
  }
  return getTeamName() || leaderTeamName || getSessionId()
}
```

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `createTask` | 函数 | 创建任务 |
| `getTask` | 函数 | 读取任务 |
| `updateTask` | 函数 | 更新任务 |
| `deleteTask` | 函数 | 删除任务 |
| `listTasks` | 函数 | 列出任务 |
| `blockTask` | 函数 | 建立阻塞关系 |
| `claimTask` | 函数 | 认领任务 |
| `resetTaskList` | 函数 | 重置任务列表 |
| `getAgentStatuses` | 函数 | 获取 Agent 状态 |
| `unassignTeammateTasks` | 函数 | 释放队友任务 |
| `getTaskListId` | 函数 | 获取任务列表 ID |
| `isTodoV2Enabled` | 函数 | 检查任务功能启用 |
| `onTasksUpdated` | 函数 | 订阅任务更新 |
| `notifyTasksUpdated` | 函数 | 通知任务更新 |
| `Task`, `TaskStatus`, `ClaimTaskResult`, etc. | 类型 | 类型定义 |

### 依赖模块

| 模块 | 用途 |
|------|------|
| `fs/promises` | 文件操作 |
| `zod/v4` | 模式验证 |
| `../bootstrap/state.js` | `getIsNonInteractiveSession`, `getSessionId` |
| `./array.js` | `uniq` |
| `./debug.js` | `logForDebugging` |
| `./envUtils.js` | `getClaudeConfigHomeDir`, `isEnvTruthy` |
| `./errors.js` | `errorMessage`, `getErrnoCode` |
| `./lazySchema.js` | `lazySchema` |
| `./lockfile.js` | 文件锁 |
| `./log.js` | `logError` |
| `./signal.js` | `createSignal` |
| `./slowOperations.js` | `jsonParse`, `jsonStringify` |
| `./teammate.js` | `getTeamName` |
| `./teammateContext.js` | `getTeammateContext` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/cli/print.ts` | `unassignTeammateTasks` |
| `src/utils/task/framework.ts` | `getRunningTasks` |
| 各种 Task 工具 | 任务 CRUD 操作 |

## 依赖与外部交互

### 与队友系统的集成
- 使用 `getTeammateContext()` 识别 in-process teammate
- 使用 `getTeamName()` 获取团队名称
- 支持 leader team name 设置（`setLeaderTeamName`）

### 与文件锁的集成
- 通过 `lockfile.ts` 延迟加载 `proper-lockfile`
- 支持任务级锁和列表级锁
- 异步锁 API 避免阻塞事件循环

### 与信号系统的集成
- 使用 `createSignal` 实现进程内通知
- `notifyTasksUpdated` 触发 UI 刷新
- 监听器错误被捕获，不影响任务操作

## 风险、边界与改进建议

### 潜在风险

1. **文件系统竞争**: 虽然使用文件锁，极端并发下仍可能有问题
2. **磁盘空间**: 大量任务文件可能占用磁盘空间
3. **状态迁移**: 旧状态名（`open`, `resolved`）的迁移代码需要维护
4. **Zod 验证失败**: 模式变更可能导致旧任务无法读取

### 边界情况

1. **任务 ID 溢出**: 理论上任务 ID 可以无限增长
2. **循环依赖**: 任务阻塞关系可能形成循环（当前未检测）
3. **孤儿任务**: 删除被阻塞任务后，阻塞关系变为空引用
4. **锁超时**: 30 次重试后仍可能失败

### 改进建议

1. **循环依赖检测**: 添加阻塞关系循环检测
```typescript
function wouldCreateCycle(fromId: string, toId: string): boolean {
  // DFS 检测从 toId 是否能到达 fromId
}
```

2. **批量操作**: 支持批量创建/更新任务
```typescript
export async function createTasks(
  taskListId: string,
  tasksData: Array<Omit<Task, 'id'>>
): Promise<string[]>
```

3. **任务归档**: 添加归档功能，将旧任务移出活跃列表
```typescript
export async function archiveTask(taskListId: string, taskId: string): Promise<void>
```

4. **事件历史**: 记录任务状态变更历史
```typescript
interface TaskEvent {
  timestamp: number
  type: 'created' | 'updated' | 'claimed' | 'completed'
  actor: string
  changes: Partial<Task>
}
```

5. **查询功能**: 支持更复杂的任务查询
```typescript
export async function queryTasks(
  taskListId: string,
  filter: { status?: TaskStatus[]; owner?: string; blocked?: boolean }
): Promise<Task[]>
```

6. **压缩存储**: 对大量任务使用压缩存储
```typescript
// 使用 JSONL 或 SQLite 替代单个文件
```

7. **分布式锁**: 对于多机部署，考虑使用分布式锁
```typescript
// 使用 Redis 或类似服务
```
