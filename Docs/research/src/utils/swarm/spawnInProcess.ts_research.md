# spawnInProcess.ts 深度研究文档

## 场景与职责

`spawnInProcess.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**进程内 teammate 创建与管理模块**，负责创建和注册在相同 Node.js 进程中运行的 teammate 任务。

### 核心场景

与基于进程的 teammate（tmux/iTerm2）不同，**进程内 teammate** 使用 AsyncLocalStorage 进行上下文隔离，在同一个 Node.js 进程中运行。这种方式：

1. **更低资源消耗**：不需要创建新的终端 pane 或进程
2. **更快启动速度**：避免了进程创建和 CLI 初始化的开销
3. **更紧密集成**：可以直接访问父进程的内存和状态

### 职责边界

- 创建 teammate 的上下文和身份
- 注册 `InProcessTeammateTaskState` 到 `AppState`
- 提供生命周期管理（spawn/kill）
- **不**执行实际的 agent 循环（由 `InProcessTeammateTask` 组件处理）

---

## 功能点目的

### 1. 进程内 Teammate 创建

**函数**：`spawnInProcessTeammate()`

**目的**：创建并注册一个进程内 teammate。

**关键步骤**：
1. 生成确定性 agent ID（格式：`name@team`）
2. 创建独立的 `AbortController`（teammate 不应随领导查询中断而中止）
3. 获取父会话 ID 用于 transcript 关联
4. 创建 `TeammateIdentity`（存储为纯数据）
5. 创建 `TeammateContext`（用于 AsyncLocalStorage）
6. 注册到 Perfetto trace（用于层级可视化）
7. 创建 `InProcessTeammateTaskState`
8. 注册清理处理程序
9. 注册任务到 `AppState`

### 2. 进程内 Teammate 终止

**函数**：`killInProcessTeammate()`

**目的**：通过中止控制器终止进程内 teammate。

**关键步骤**：
1. 从 `AppState` 获取任务状态
2. 中止 `AbortController`
3. 调用清理处理程序
4. 调用空闲回调（解除 `engine.waitForIdle` 等待）
5. 更新任务状态为 `'killed'`
6. 从 `teamContext.teammates` 移除
7. 从团队文件移除成员
8. 触发 SDK 任务终止事件
9. 延迟后驱逐终端任务

---

## 具体技术实现

### 数据结构

#### SpawnContext

```typescript
{
  setAppState: SetAppStateFn;  // 用于注册任务
  toolUseId?: string;           // 关联的工具使用 ID
}
```

#### InProcessSpawnConfig

```typescript
{
  name: string;                 // 显示名称（如 "researcher"）
  teamName: string;             // 所属团队
  prompt: string;               // 初始提示/任务
  color?: string;               // UI 颜色
  planModeRequired: boolean;    // 是否需要在实现前进入计划模式
  model?: string;               // 可选的模型覆盖
}
```

#### InProcessSpawnOutput

```typescript
{
  success: boolean;             // 是否成功
  agentId: string;              // 完整 agent ID（格式："name@team"）
  taskId?: string;              // AppState 中的任务 ID
  abortController?: AbortController;  // 生命周期管理
  teammateContext?: ReturnType<typeof createTeammateContext>;  // AsyncLocalStorage 上下文
  error?: string;               // 错误信息
}
```

### 关键流程

#### 创建流程

```
spawnInProcessTeammate(config, context)
├── formatAgentId(name, teamName) → agentId
├── generateTaskId('in_process_teammate') → taskId
├── createAbortController() → abortController
├── getSessionId() → parentSessionId
├── 创建 TeammateIdentity
├── createTeammateContext({...}) → teammateContext
├── registerPerfettoAgent(agentId, name, parentSessionId) [可选]
├── 创建 InProcessTeammateTaskState
│   ├── createTaskStateBase(taskId, type, description, toolUseId)
│   ├── 设置 identity, prompt, model, abortController
│   ├── 设置 spinnerVerb, pastTenseVerb（随机选择）
│   └── 设置 permissionMode, isIdle, shutdownRequested
├── registerCleanup(() => abortController.abort()) → unregisterCleanup
└── registerTask(taskState, setAppState)
```

#### 终止流程

```
killInProcessTeammate(taskId, setAppState)
├── setAppState(prev => {
│   ├── 查找任务
│   ├── 检查状态是否为 'running'
│   ├── 捕获 identity 信息
│   ├── abortController.abort()
│   ├── unregisterCleanup()
│   ├── 调用 onIdleCallbacks
│   ├── 从 teamContext.teammates 移除
│   └── 更新任务状态为 'killed'
│})
├── removeMemberByAgentId(teamName, agentId) [如果 teamName 和 agentId 存在]
├── evictTaskOutput(taskId)
├── emitTaskTerminatedSdk(taskId, 'stopped', {...})
├── setTimeout(evictTerminalTask, STOPPED_DISPLAY_MS)
└── unregisterPerfettoAgent(agentId)
```

### 状态转换

```
创建时: status = 'running', isIdle = false, shutdownRequested = false

正常完成:
  'running' → 'completed' (由 InProcessTeammateTask 设置)

被杀死:
  'running' → 'killed' (由 killInProcessTeammate 设置)
  
空闲状态:
  isIdle = true (当 teammate 等待新任务时)
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `SpawnContext` | Type | 创建上下文类型 |
| `InProcessSpawnConfig` | Type | 创建配置类型 |
| `InProcessSpawnOutput` | Type | 创建输出类型 |
| `spawnInProcessTeammate()` | Function | 创建进程内 teammate |
| `killInProcessTeammate()` | Function | 终止进程内 teammate |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/tools/shared/spawnMultiAgent.ts` | `spawnInProcessTeammate`, `InProcessSpawnConfig` | 处理 in-process 创建模式 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | `killInProcessTeammate` | 终止 teammate 任务 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/state/AppState.ts` | `AppState` 类型 |
| `src/Task.ts` | `createTaskStateBase`, `generateTaskId` |
| `src/tasks/InProcessTeammateTask/types.ts` | `InProcessTeammateTaskState`, `TeammateIdentity` |
| `src/utils/agentId.ts` | `formatAgentId` |
| `src/utils/abortController.ts` | `createAbortController` |
| `src/utils/teammateContext.ts` | `createTeammateContext` |
| `src/utils/task/framework.ts` | `registerTask`, `evictTerminalTask`, `STOPPED_DISPLAY_MS` |
| `src/utils/task/diskOutput.ts` | `evictTaskOutput` |
| `src/utils/sdkEventQueue.ts` | `emitTaskTerminatedSdk` |
| `src/utils/telemetry/perfettoTracing.ts` | `registerAgent`, `unregisterAgent` |
| `src/utils/swarm/teamHelpers.ts` | `removeMemberByAgentId` |
| `src/utils/cleanupRegistry.ts` | `registerCleanup` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/bootstrap/state.ts` | `getSessionId` |
| `src/constants/spinnerVerbs.ts` | `getSpinnerVerbs` |
| `src/constants/turnCompletionVerbs.ts` | `TURN_COMPLETION_VERBS` |

---

## 依赖与外部交互

### 模块依赖图

```
spawnInProcess.ts
├── AppState.ts              # 状态类型
├── Task.ts                  # 任务基础创建
├── InProcessTeammateTask/types.ts  # 任务状态类型
├── agentId.ts               # Agent ID 格式化
├── abortController.ts       # 中止控制器
├── teammateContext.ts       # AsyncLocalStorage 上下文
├── task/framework.ts        # 任务注册
├── task/diskOutput.ts       # 任务输出清理
├── sdkEventQueue.ts         # SDK 事件
├── perfettoTracing.ts       # 性能追踪
├── teamHelpers.ts           # 团队成员移除
├── cleanupRegistry.ts       # 清理注册
├── debug.ts                 # 调试日志
├── bootstrap/state.ts       # 会话状态
└── constants/               # 动词常量
```

### 与 InProcessTeammateTask 的交互

```typescript
// InProcessTeammateTask.tsx
import { killInProcessTeammate } from '../../utils/swarm/spawnInProcess.js';

// 当用户点击终止按钮时
const handleKill = () => {
  killInProcessTeammate(taskId, setAppState);
};

// 实际的 agent 执行循环在 InProcessTeammateTask 中
// 使用 runWithTeammateContext() 执行
```

### 与 spawnMultiAgent.ts 的交互

```typescript
// spawnMultiAgent.ts
import { spawnInProcessTeammate, InProcessSpawnConfig } from '../../utils/swarm/spawnInProcess.js';
import { startInProcessTeammate } from '../../utils/swarm/inProcessRunner.js';

async function handleSpawnInProcess(input: SpawnInput, context: ToolUseContext) {
  const config: InProcessSpawnConfig = {
    name: sanitizedName,
    teamName,
    prompt,
    color: teammateColor,
    planModeRequired: plan_mode_required ?? false,
    model,
  };
  
  const result = await spawnInProcessTeammate(config, context);
  
  if (result.success && result.taskId && result.teammateContext) {
    // 启动 agent 执行循环
    startInProcessTeammate({
      identity: {...},
      taskId: result.taskId,
      prompt,
      teammateContext: result.teammateContext,
      abortController: result.abortController,
      // ...
    });
  }
}
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 独立 AbortController 的风险

**风险**：注释说明 "teammates should not be aborted when the leader's query is interrupted"，但独立的 `AbortController` 意味着领导无法直接控制 teammate。

**当前实现**：
```typescript
// Create independent AbortController for this teammate
// Teammates should not be aborted when the leader's query is interrupted
const abortController = createAbortController();
```

**改进建议**：
```typescript
// 创建关联但独立的控制器
const abortController = createAbortController();
const parentSignal = context.parentAbortController?.signal;

// 父中止时通知子，但不强制中止
parentSignal?.addEventListener('abort', () => {
  logForDebugging(`[spawnInProcessTeammate] Parent aborted, child continues`);
  // 可选：设置标志让 teammate 知道领导已中断
});
```

#### 2. 任务状态同步问题

**风险**：`killInProcessTeammate` 中的 `setAppState` 回调可能与其他更新竞争。

**改进建议**：
```typescript
// 使用函数式更新确保原子性
setAppState((prev: AppState) => {
  const task = prev.tasks[taskId];
  // 双重检查锁定模式
  if (!task || task.type !== 'in_process_teammate' || task.status !== 'running') {
    return prev;  // 已被其他调用更新
  }
  // ... 更新
});
```

#### 3. 清理处理程序泄漏

**风险**：如果 `killInProcessTeammate` 被多次调用，可能导致重复清理。

**当前保护**：
```typescript
// 在状态更新中清除引用
tasks: {
  ...prev.tasks,
  [taskId]: {
    ...teammateTask,
    unregisterCleanup: undefined,  // 清除引用
  }
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| 任务不存在 | `killInProcessTeammate` 返回 `false` |
| 任务类型不匹配 | 返回 `false`，不执行操作 |
| 任务非运行状态 | 返回 `false`，不执行操作 |
| 中止控制器已触发 | 重复调用无害（`abort()` 可安全多次调用） |
| teamName/agentId 缺失 | 跳过 `removeMemberByAgentId`，其他操作继续 |

### 性能考虑

1. **内存使用**：每个进程内 teammate 都在同一进程中，共享内存空间
2. **CPU 隔离**：没有真正的 CPU 隔离，teammate 可能阻塞主线程
3. **启动速度**：比 tmux/iTerm2 模式快，因为没有进程创建开销

### 改进建议

#### 1. 添加创建超时

```typescript
export async function spawnInProcessTeammate(
  config: InProcessSpawnConfig,
  context: SpawnContext,
  timeoutMs: number = 30000
): Promise<InProcessSpawnOutput> {
  return Promise.race([
    doSpawn(config, context),
    new Promise<InProcessSpawnOutput>((_, reject) =>
      setTimeout(() => reject(new Error('Spawn timeout')), timeoutMs)
    )
  ]);
}
```

#### 2. 资源限制

```typescript
// 添加资源使用监控
export type InProcessTeammateMetrics = {
  cpuTime: number;
  memoryUsage: number;
  toolCallCount: number;
  startTime: number;
};

// 在 taskState 中添加
const taskState: InProcessTeammateTaskState = {
  // ...
  metrics: {
    cpuTime: 0,
    memoryUsage: 0,
    toolCallCount: 0,
    startTime: Date.now(),
  }
};
```

#### 3. 优雅关闭支持

```typescript
// 支持优雅关闭（等待当前工具完成）
export async function gracefulShutdownTeammate(
  taskId: string,
  setAppState: SetAppStateFn,
  timeoutMs: number = 30000
): Promise<boolean> {
  // 1. 设置 shutdownRequested 标志
  // 2. 等待 teammate 完成当前工作
  // 3. 超时后强制中止
}
```

#### 4. 批量创建支持

```typescript
// 支持同时创建多个 teammate
export async function spawnInProcessTeammates(
  configs: InProcessSpawnConfig[],
  context: SpawnContext
): Promise<InProcessSpawnOutput[]> {
  // 并行创建，但限制并发数
  const CONCURRENCY_LIMIT = 5;
  return pMap(configs, config => spawnInProcessTeammate(config, context), {
    concurrency: CONCURRENCY_LIMIT
  });
}
```

### 测试建议

1. **单元测试**：
   - 创建成功/失败场景
   - 终止成功/失败场景
   - 重复终止处理

2. **集成测试**：
   - 与 `InProcessTeammateTask` 的集成
   - 与 `AppState` 的同步

3. **压力测试**：
   - 同时创建大量进程内 teammate
   - 测量内存使用和性能

4. **边界测试**：
   - 空配置
   - 超长名称
   - 特殊字符
