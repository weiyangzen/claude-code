# 研究文档：src/tasks/DreamTask/DreamTask.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文。  
> 生成时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 1. 场景与职责

`DreamTask` 是 **auto-dream（后台记忆整合子代理）的 UI 入口与状态包装层**。它的核心职责是：

- **可见性**：把原本“不可见”的 forked 子代理（auto-dream）暴露到现有的任务注册表（task registry）中，使其出现在底部状态栏（footer pill）和 `Shift+↓` 后台任务对话框（BackgroundTasksDialog）里。
- **生命周期管理**：提供注册、推进（turn-by-turn）、完成、失败、强制终止（kill）五个标准生命周期动作。
- **轻量状态聚合**：在子代理运行期间，实时收集并截断展示数据（assistant 文本回复、工具调用计数、被触碰的文件路径），供 React UI 渲染。

> 文件头注释明确指出：*"The dream agent itself is unchanged — this is pure UI surfacing via the existing task registry."*  
> 这意味着 `DreamTask.ts` 本身不实现任何 LLM 逻辑或记忆整合算法，它只负责**状态包装与 UI 桥接**。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| `registerDreamTask` | 在 auto-dream 触发时，向 `AppState.tasks` 注册一个类型为 `dream` 的任务，生成唯一 ID，绑定 `AbortController`。 |
| `addDreamTurn` | 每收到 forked agent 的一条 assistant 消息，将其文本内容+工具调用计数追加到 `turns` 数组，同时收集 `Edit`/`Write` 工具中触碰的文件路径。 |
| `completeDreamTask` | forked agent 正常结束后，将状态置为 `completed`，记录 `endTime`，并立即标记 `notified=true`（UI-only，无需模型通知）。 |
| `failDreamTask` | forked agent 异常或失败时，将状态置为 `failed`，同样立即标记 `notified=true`。 |
| `DreamTask.kill` | 用户通过 `Shift+↓` 对话框按 `x` 停止时，调用 `abortController.abort()` 中断子代理，并回滚 consolidation lock 的 mtime，以便下次触发重试。 |
| `isDreamTask` | 类型守卫（type guard），在消费端安全窄化 `TaskState` 到 `DreamTaskState`。 |

### 2.1 UI 展示目的

- **Footer pill**：`BackgroundTask.tsx` 对 `dream` 类型任务渲染为 `· {phase} · {detail}`（如 `· updating · 3 files` 或 `· starting · 5 sessions`）。
- **Detail dialog**：`DreamDetailDialog.tsx` 展示运行时长、review 的 session 数量、已触碰文件数、最近 6 条可见 turns（更早的折叠为计数）。
- **BackgroundTasksDialog**：`dream` 任务与其他后台任务（bash、local_agent、remote_agent 等）并列，支持 `↑/↓` 选择、`Enter` 查看详情、`x` 停止。

---

## 3. 具体技术实现

### 3.1 关键数据结构

```ts
// src/tasks/DreamTask/DreamTask.ts
export type DreamTurn = {
  text: string           // assistant 文本块拼接结果
  toolUseCount: number   // 该 turn 中 tool_use 块的数量（折叠展示）
}

export type DreamPhase = 'starting' | 'updating'

export type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: DreamPhase
  sessionsReviewing: number   // 本次要 review 的 session 总数
  filesTouched: string[]      // 从 Edit/Write 工具参数里 pattern-match 出的路径（不完整）
  turns: DreamTurn[]          // 最近 turns，内存中截断保留
  abortController?: AbortController
  priorMtime: number          // kill 时用于回滚 lock mtime
}
```

- `MAX_TURNS = 30`：内存中只保留最近 30 条 turn，防止长时间运行导致状态膨胀。
- `turns` **不包含 prompt**：注释明确说明 *"Prompt is NOT included"*，只收集 assistant 的回复内容。
- `filesTouched` **显式声明不完整**：注释指出它只捕获通过 `Edit`/`Write` 的 `tool_use` 块 pattern-match 到的路径，**遗漏所有通过 bash 间接写入的文件**。

### 3.2 关键流程

#### 3.2.1 注册流程 `registerDreamTask`

1. 调用 `generateTaskId('dream')` 生成 `d` 前缀的 9 位随机 ID（1 前缀 + 8 随机字符）。
2. 调用 `createTaskStateBase(id, 'dream', 'dreaming')` 创建基础字段（`status='pending'`、`outputFile` 指向磁盘临时目录等）。
3. 覆盖/补充字段：`status='running'`、`phase='starting'`、`filesTouched=[]`、`turns=[]`、注入外部传入的 `abortController`/`priorMtime`/`sessionsReviewing`。
4. 调用 `registerTask(task, setAppState)` 写入全局 `AppState.tasks`。

#### 3.2.2 Turn 追加流程 `addDreamTurn`

1. 通过 `updateTaskState<DreamTaskState>` 以函数式更新方式修改任务状态。
2. 使用 `Set` 去重，过滤出本次新增的 `touchedPaths`。
3. **空更新短路**：如果 `turn.text === ''`、`toolUseCount === 0` 且没有新文件被触碰，则直接返回原对象引用，避免触发无意义的 React re-render。
4. 若存在新触碰文件，将 `phase` 从 `starting` 翻转为 `updating`。
5. `turns` 数组维护：`task.turns.slice(-(MAX_TURNS - 1)).concat(turn)`，保证长度不超过 30。

#### 3.2.3 完成/失败流程

- `completeDreamTask` / `failDreamTask`：统一设置 `status`、`endTime`、`notified=true`，并清理 `abortController` 引用。
- 特别说明：`notified=true` 被立即设置，因为 dream 任务**没有面向模型的通知路径**（UI-only），且任务框架的 `evictTerminalTask` 要求 `terminal + notified` 才会从 `AppState` 中移除。

#### 3.2.4 Kill 流程 `DreamTask.kill`

```ts
async kill(taskId, setAppState) {
  let priorMtime: number | undefined
  updateTaskState<DreamTaskState>(taskId, setAppState, task => {
    if (task.status !== 'running') return task   // 已终止则 noop
    task.abortController?.abort()
    priorMtime = task.priorMtime
    return { ...task, status: 'killed', endTime: Date.now(), notified: true, abortController: undefined }
  })
  if (priorMtime !== undefined) {
    await rollbackConsolidationLock(priorMtime)
  }
}
```

- 先检查状态：只有 `running` 的任务才会真正执行 abort 和状态变更；已处于 terminal 状态的任务直接短路。
- abort 后回滚 `priorMtime`：与 `autoDream.ts` 中 fork 失败时的 catch 分支走同一条路径，确保用户 kill 后下一次 session 仍然满足时间门条件可以重试。

### 3.3 协议与约定

- **Task 接口契约**：`DreamTask` 实现了 `Task` 接口（`name`、`type`、`kill`），并在 `src/tasks.ts` 的 `getAllTasks()` 中被注册到全局任务表，使 `getTaskByType('dream')` 可以按类型多态调度 kill。
- **BackgroundTask 契约**：`src/tasks/types.ts` 将 `DreamTaskState` 纳入 `BackgroundTaskState` 联合类型；`isBackgroundTask()` 仅检查 `status === 'running' || 'pending'`，因此 dream 任务天然符合后台任务显示条件。
- **DiskOutput 契约**：`createTaskStateBase` 会为每个任务分配 `outputFile`（通过 `getTaskOutputPath(id)` 指向项目临时目录），但 `DreamTask` 目前**不写入任何内容到该文件**——它完全依赖内存中的 `turns` 做 UI 展示。

---

## 4. 关键代码路径与文件引用

### 4.1 目标文件

- `src/tasks/DreamTask/DreamTask.ts` — 本文研究对象。

### 4.2 直接调用方（上游）

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/services/autoDream/autoDream.ts` | `registerDreamTask`、`addDreamTurn`（通过 `makeDreamProgressWatcher`）、`completeDreamTask`、`failDreamTask`、`isDreamTask` | auto-dream 的核心调度器，负责在时间/会话/锁三门条件满足后 fork 子代理。 |
| `src/tasks.ts` | `import { DreamTask }` 并加入 `getAllTasks()` 数组 | 任务类型注册表。 |
| `src/components/tasks/BackgroundTasksDialog.tsx` | `import { DreamTask }` 并调用 `DreamTask.kill(taskId, setAppState)` | 后台任务对话框中的停止操作。 |

### 4.3 直接依赖（下游/被调用）

| 文件 | 使用内容 | 说明 |
|------|----------|------|
| `src/services/autoDream/consolidationLock.ts` | `rollbackConsolidationLock` | kill 时回滚锁文件的 mtime。 |
| `src/Task.ts` | `SetAppState`、`Task`、`TaskStateBase`、`generateTaskId`、`createTaskStateBase` | 任务系统的基础类型与工具函数。 |
| `src/utils/task/framework.ts` | `registerTask`、`updateTaskState` | 任务状态写入框架。 |

### 4.4 UI 消费端

| 文件 | 消费内容 | 说明 |
|------|----------|------|
| `src/components/tasks/DreamDetailDialog.tsx` | `import type { DreamTaskState }` | 详情弹窗，展示 turns、filesTouched、elapsedTime。 |
| `src/components/tasks/BackgroundTask.tsx` | `case "dream"` | Footer pill 与列表项的渲染逻辑。 |
| `src/components/tasks/BackgroundTasksDialog.tsx` | `import { DreamTask, type DreamTaskState }` | 列表选择、kill 调度、详情路由。 |
| `src/tasks/types.ts` | `import type { DreamTaskState }` | 联合类型声明。 |

### 4.5 配置与开关

| 文件 | 作用 |
|------|------|
| `src/services/autoDream/config.ts` | `isAutoDreamEnabled()`：用户设置 `autoDreamEnabled` 优先，否则回退 GrowthBook `tengu_onyx_plover.enabled`。 |
| `src/services/autoDream/consolidationLock.ts` | 锁文件 `.consolidate-lock` 的读写、PID 竞争检测、mtime 回滚。 |
| `src/services/autoDream/autoDream.ts` | 三门逻辑（时间门 `minHours`、会话门 `minSessions`、锁门）及 `runForkedAgent` 调用。 |

---

## 5. 依赖与外部交互

### 5.1 运行时数据流

```
┌─────────────────────────────────────────────────────────────────┐
│  stopHooks / REPL loop                                          │
│  └── executeAutoDream(context, appendSystemMessage)             │
│       └── autoDream.ts: runner                                  │
│            ├── 三门检查 (time / sessions / lock)                │
│            ├── registerDreamTask(setAppState, {sessionsReviewing, priorMtime, abortController})
│            ├── runForkedAgent({ onMessage: makeDreamProgressWatcher(taskId, setAppState) })
│            │      └── 每收到 assistant message                  │
│            │           └── addDreamTurn(taskId, turn, touchedPaths, setAppState)
│            ├── completeDreamTask(taskId, setAppState)           │
│            └── catch: failDreamTask(taskId, setAppState)        │
│                                                                  │
│  User UI (Shift+↓)                                               │
│  └── BackgroundTasksDialog ──► DreamDetailDialog                │
│       └── killDreamTask() ──► DreamTask.kill()                  │
│            └── abortController.abort() + rollbackConsolidationLock()
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 与任务框架的集成

- `registerTask` 在写入 `AppState` 时会自动发出 `enqueueSdkEvent({ type: 'system', subtype: 'task_started', ... })`。
- `updateTaskState` 提供引用相等优化：如果 updater 返回原对象，则跳过 `setAppState` 的 spread，避免无意义重渲染。
- 任务框架的 `pollTasks` / `generateTaskAttachments` 会轮询所有 running 任务的 `outputFile` 增量，但 `DreamTask` **不写入该文件**，因此该轮询对其永远是空内容，不影响状态。

### 5.3 与锁机制的耦合

- `autoDream.ts` 在 fork 前调用 `tryAcquireConsolidationLock()` 获取 `priorMtime`。
- `DreamTask` 把 `priorMtime` 存入状态，以便 `kill` 时精确回滚。
- 如果 fork 过程中抛异常（非 abort），`autoDream.ts` 也会调用 `rollbackConsolidationLock(priorMtime)`。
- 这种设计保证了：**无论是用户主动 kill 还是 fork 失败，都不会让锁文件的 mtime 停留在“已占用”状态，从而阻塞后续触发**。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

1. **`filesTouched` 不完整**  
   代码注释已明确警告：只捕获 `Edit`/`Write` 的 `tool_use` 参数中的 `file_path`，**遗漏所有 bash 中介写入**。这会导致 inline completion message（`"Improved [files...]"`）和详情页展示的文件列表不完整，可能给用户造成“只改了这些文件”的误导。

2. **无测试覆盖**  
   全局搜索未找到任何针对 `DreamTask.ts` 或 `autoDream.ts` 的 `.test.ts` / `.spec.ts` 文件。`DreamTask` 的生命周期函数（尤其是 `addDreamTurn` 的空更新短路、`kill` 的 mtime 回滚分支）缺乏单元测试保护。

3. **`MAX_TURNS` 硬编码且无配置出口**  
   30 条 turn 的截断对极长运行可能丢失早期上下文；虽然 UI 只展示最近 6 条，但内存中保留 30 条的阈值是写死的。

4. **Phase 粒度过粗**  
   只有 `starting` / `updating` 两阶段。注释提到 dream prompt 实际有 4 阶段结构（orient / gather / consolidate / prune），但未做解析。如果未来需要在 UI 展示更细粒度进度，需要扩展 parser。

5. **磁盘 outputFile 空置**  
   `TaskStateBase` 强制要求 `outputFile`，但 `DreamTask` 从不写入。任务框架的轮询逻辑虽然能处理空读，但这引入了一个隐式约定：所有消费 `outputFile` 的通用代码都必须兼容空文件或 ENOENT。

6. **Kill 的竞态条件**  
   `DreamTask.kill` 中 `task.abortController?.abort()` 发生在 `updateTaskState` 的同步闭包里，而 `rollbackConsolidationLock` 是异步的。如果 `autoDream.ts` 的 catch 块与 `kill` 几乎同时触发，存在理论上的 double-rollback 风险（虽然 `autoDream.ts` 通过 `abortController.signal.aborted` 检查做了部分防护）。

### 6.2 边界行为

- **空 turn 过滤**：`addDreamTurn` 中如果 `text === ''`、`toolUseCount === 0` 且没有新文件，则直接返回原对象，React 不会 re-render。
- **重复注册**：`registerTask` 内部会检测 `isReplacement`，如果是恢复（resume）则合并旧状态（`retain`、`messages`、`diskLoaded` 等）。`DreamTask` 目前不涉及 resume 路径，但框架已兼容。
- **Eviction 条件**：任务进入 `completed`/`failed`/`killed` 且 `notified=true` 后，任务框架的 `generateTaskAttachments` 会将其加入 `evictedTaskIds`，随后从 `AppState.tasks` 中移除。`DreamTask` 在 complete/fail/kill 时都立即设置 `notified=true`，因此会尽快被清理。

### 6.3 改进建议

1. **补充单元测试**  
   建议为 `DreamTask.ts` 增加独立测试，覆盖：
   - `registerDreamTask` 生成正确状态结构；
   - `addDreamTurn` 的去重、空更新短路、`MAX_TURNS` 截断、`phase` 翻转；
   - `kill` 对非 `running` 任务的 noop 行为、对 `running` 任务的 abort + rollback 调用。

2. **扩展 `filesTouched` 收集面**  
   可在 `onMessage` 的 watcher 中增加对 `BashTool` 的 pattern-match（例如解析 `cat >`、`echo >`、`<(` 等写操作），或改为在 forked agent 结束后扫描 `memoryRoot` 的 mtime 变化，以获得更准确的文件改动列表。

3. **考虑利用 `outputFile`**  
   如果未来 `turns` 数据量显著增长，可将 turns 序列化写入 `outputFile`，让 `DreamTask` 与任务框架的轮询机制对齐，减少内存占用。

4. **细化 Phase（可选）**  
   若产品需要更细进度展示，可在 `makeDreamProgressWatcher` 或 prompt 层面加入阶段标记（如 XML tag），并扩展 `DreamPhase` 类型与 UI 渲染。

5. **统一 rollback 入口**  
   当前 `autoDream.ts` catch 块和 `DreamTask.kill` 各自调用 `rollbackConsolidationLock`，可考虑将 rollback 逻辑收敛到 `DreamTask` 的失败处理中，减少分散的重复代码。

---

*文档结束*
