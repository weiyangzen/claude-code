# cronScheduler.ts 深度研究文档

## 场景与职责

`cronScheduler.ts` 是 Claude Code 定时任务系统的 **核心调度引擎**，负责管理 `.claude/scheduled_tasks.json` 中定义的定时任务的整个生命周期。它是 REPL 模式和 SDK/Daemon 模式共享的调度基础设施。

### 核心职责

1. **任务加载与监控**：从 JSON 文件加载任务，并使用 chokidar 监控文件变更
2. **执行调度**：基于 cron 表达式计算下次执行时间，定时触发任务
3. **锁管理**：通过文件锁确保同一项目的多个 Claude 会话不会重复触发任务
4. **错过任务处理**：检测并处理 Claude 未运行时错过的任务
5. **会话任务支持**：支持仅在内存中的临时任务（`durable: false`）
6. **抖动应用**：应用 jitter 策略分散任务执行时间

### 使用场景

- **REPL 模式**：`useScheduledTasks.ts` 钩子挂载调度器，任务触发时入队到命令队列
- **Daemon 模式**：`print.ts` 以 headless 模式运行调度器，任务触发时执行回调
- **多会话协调**：同一项目的多个 Claude 实例通过锁机制协调任务所有权
- **错过任务恢复**：启动时检测错过的任务并提示用户处理

---

## 功能点目的

### 1. 调度器创建 (`createCronScheduler`)

**目的**：创建可启动/停止的调度器实例，支持多种运行模式。

**配置选项**：

| 选项 | 类型 | 用途 |
|------|------|------|
| `onFire` | `(prompt: string) => void` | 任务触发回调（简单模式） |
| `onFireTask` | `(task: CronTask) => void` | 任务触发回调（完整信息） |
| `onMissed` | `(tasks: CronTask[]) => void` | 错过任务回调 |
| `isLoading` | `() => boolean` | 检查是否处于加载状态（阻塞触发） |
| `assistantMode` | `boolean` | 助手模式（绕过 isLoading 检查） |
| `dir` | `string` | 任务文件目录（SDK 模式必需） |
| `lockIdentity` | `string` | 锁标识（SDK 模式使用进程 UUID） |
| `getJitterConfig` | `() => CronJitterConfig` | 抖动配置获取（REPL 模式） |
| `isKilled` | `() => boolean` | 终止开关检查 |
| `filter` | `(t: CronTask) => boolean` | 任务过滤（Daemon 用于筛选 permanent 任务） |

### 2. 任务检查循环 (`check`)

**目的**：每秒检查一次任务，触发到期的任务。

**处理流程**：
1. 检查终止开关和加载状态
2. 获取当前抖动配置
3. 遍历文件任务（仅当持有锁时）
4. 遍历会话任务（仅 REPL 模式）
5. 对每个任务：
   - 计算下次执行时间（首次从 `createdAt`/`lastFiredAt`，后续从内存）
   - 检查是否到期
   - 触发任务回调
   - 更新执行状态（周期性任务重新调度，一次性任务删除）

### 3. 锁管理 (`tryAcquireSchedulerLock` / `releaseSchedulerLock`)

**目的**：确保同一项目的多个 Claude 实例不会同时触发相同的磁盘任务。

**锁机制**：
- 锁文件：`.claude/scheduled_tasks.lock`
- 锁内容：`{ sessionId, pid, acquiredAt }`
- 获取方式：O_EXCL 原子创建（`wx` 标志）
- 存活检测：检查持有进程的 PID 是否仍在运行
- 自动恢复：检测到僵尸锁时自动清理并重新获取

### 4. 错过任务处理 (`load` 中的初始加载)

**目的**：检测 Claude 未运行时错过的任务，并提示用户处理。

**处理逻辑**：
1. 仅在初始加载时检测（文件变更时不检测）
2. 筛选出已过期的一次性任务
3. 调用 `onMissed` 回调或默认的 `onFire` 回调
4. 自动从文件中删除错过的任务
5. 防止重复提示（使用 `missedAsked` Set 跟踪）

### 5. 任务生命周期管理

**周期性任务**：
- 触发后从当前时间重新计算下次执行
- 更新 `lastFiredAt` 并写回文件
- 支持自动过期（`recurringMaxAgeMs`）

**一次性任务**：
- 触发后立即删除
- 会话任务：同步从内存删除
- 文件任务：异步删除 + chokidar 重新加载

** aged-out 任务**：
- 超过 `recurringMaxAgeMs` 的周期性任务
- 触发一次后删除（给予用户最后一次执行机会）

---

## 具体技术实现

### 核心数据结构

```typescript
type CronSchedulerOptions = {
  onFire: (prompt: string) => void
  isLoading: () => boolean
  assistantMode?: boolean
  onFireTask?: (task: CronTask) => void
  onMissed?: (tasks: CronTask[]) => void
  dir?: string
  lockIdentity?: string
  getJitterConfig?: () => CronJitterConfig
  isKilled?: () => boolean
  filter?: (t: CronTask) => boolean
}

type CronScheduler = {
  start: () => void
  stop: () => void
  getNextFireTime: () => number | null
}
```

### 内部状态

```typescript
let tasks: CronTask[] = []                    // 文件任务列表
const nextFireAt = new Map<string, number>()  // 任务 ID → 下次执行时间
const missedAsked = new Set<string>()         // 已提示的错过任务
const inFlight = new Set<string>()            // 正在执行的任务（防止重复触发）
let isOwner = false                           // 是否持有调度器锁
```

### 关键流程：任务处理

```
check()
    ↓
[检查终止开关] → 终止则返回
    ↓
[检查加载状态] → 加载中且非助手模式则返回
    ↓
获取抖动配置
    ↓
遍历文件任务（仅当 isOwner）
    ↓
process(task, isSession)
    ↓
[首次见到任务] → 计算下次执行时间 → 存入 nextFireAt
    ↓
[已见过任务] → 从 nextFireAt 读取
    ↓
检查是否到期（now >= next）
    ↓
[未到期] → 返回
[到期] → 触发回调
    ↓
[周期性任务] → 重新计算下次时间 → 更新 lastFiredAt
[一次性任务] → 删除任务
[aged-out 任务] → 触发后删除
```

### 关键流程：锁获取

```
enable()
    ↓
tryAcquireSchedulerLock()
    ↓
尝试 O_EXCL 创建锁文件
    ↓
[成功] → 注册清理函数 → isOwner = true
[失败] → 读取现有锁
    ↓
[锁是我们的] → 更新 PID → isOwner = true
[锁是别人的且存活] → isOwner = false → 启动探测定时器
[锁是僵尸] → 删除锁 → 重试创建
```

### 错过任务通知格式

```
The following one-shot scheduled task was missed while Claude was not running.
It has already been removed from .claude/scheduled_tasks.json.

Do NOT execute this prompt yet. First use the AskUserQuestion tool to ask
whether to run it now. Only execute if the user confirms.

[cron schedule, created timestamp]
```
[任务提示内容]
```
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `createCronScheduler` | 142-531 | 创建调度器实例 |
| `isRecurringTaskAged` | 53-60 | 检查周期性任务是否过期 |
| `buildMissedTaskNotification` | 542-565 | 构建错过任务通知文本 |

### 导出类型

| 类型 | 行号 | 用途 |
|------|------|------|
| `CronSchedulerOptions` | 62-128 | 调度器配置选项 |
| `CronScheduler` | 130-140 | 调度器接口 |

### 依赖文件

```
cronScheduler.ts
├── 被调用方（上游）
│   ├── src/hooks/useScheduledTasks.ts       # REPL 模式钩子
│   └── src/cli/print.ts                      # Daemon 模式
├── 被依赖模块（下游）
│   ├── src/bootstrap/state.js               # 会话任务状态
│   ├── src/services/analytics/index.js      # 事件日志
│   ├── src/utils/cron.js                    # cron 计算
│   ├── src/utils/cronTasks.js               # 任务 CRUD
│   ├── src/utils/cronTasksLock.js           # 锁管理
│   └── src/utils/debug.js                   # 调试日志
└── 外部依赖
    └── chokidar (动态导入)                   # 文件监控
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `chokidar` | 文件系统监控（动态导入） |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.js` | `getScheduledTasksEnabled`, `getSessionCronTasks`, `removeSessionCronTasks`, `setScheduledTasksEnabled` | 会话任务状态管理 |
| `src/services/analytics/index.js` | `logEvent` | 任务触发事件日志 |
| `src/utils/cron.js` | `cronToHuman` | 错过任务通知格式化 |
| `src/utils/cronTasks.js` | `CronJitterConfig`, `CronTask`, `DEFAULT_CRON_JITTER_CONFIG`, `findMissedTasks`, `getCronFilePath`, `hasCronTasksSync`, `jitteredNextCronRunMs`, `markCronTasksFired`, `oneShotJitteredNextCronRunMs`, `readCronTasks`, `removeCronTasks` | 任务管理 |
| `src/utils/cronTasksLock.js` | `releaseSchedulerLock`, `tryAcquireSchedulerLock` | 锁管理 |
| `src/utils/debug.js` | `logForDebugging` | 调试日志 |

### 常量定义

| 常量 | 行号 | 值 | 说明 |
|------|------|-----|------|
| `CHECK_INTERVAL_MS` | 40 | 1000 | 检查间隔（1秒） |
| `FILE_STABILITY_MS` | 41 | 300 | 文件稳定时间（防抖） |
| `LOCK_PROBE_INTERVAL_MS` | 44 | 5000 | 锁探测间隔（5秒） |

---

## 风险、边界与改进建议

### 已知风险

1. **锁竞争**
   - 多个会话同时启动时可能产生锁竞争
   - 当前实现通过 O_EXCL 原子操作缓解，但仍可能失败

2. **chokidar 可靠性**
   - 文件系统事件可能丢失或重复
   - `awaitWriteFinish` 配置缓解但无法完全消除

3. **时间精度**
   - 1 秒检查间隔意味着任务可能延迟最多 1 秒触发
   - 对于需要秒级精度的任务不适用

4. **内存泄漏**
   - `nextFireAt` Map 可能累积已删除任务的条目
   - `missedAsked` Set 在长时间运行会话中可能增长

5. **跨平台兼容性**
   - 锁机制依赖 PID 检测，在容器环境中可能不可靠
   - chokidar 在某些文件系统（如网络文件系统）上行为不一致

### 边界情况

| 场景 | 处理 |
|------|------|
| 任务文件被删除 | 清空任务列表和调度 |
| 任务文件损坏 | 依赖 `readCronTasks` 的错误处理 |
| 系统时间跳跃 | 可能错过或重复触发任务 |
| 任务回调抛出异常 | 不捕获，可能导致调度器停止 |
| 锁文件损坏 | 视为僵尸锁，尝试恢复 |
| 同一任务 ID 重复 | 后加载的覆盖先加载的 |

### 改进建议

1. **可靠性增强**
   - 添加任务执行确认机制，确保任务真正被执行
   - 实现任务执行历史记录，便于调试和审计
   - 添加任务执行超时保护

2. **性能优化**
   - 使用最小堆优化下次执行时间的查找
   - 减少不必要的文件系统操作
   - 添加任务数量上限保护

3. **可观测性**
   - 添加调度器状态指标（任务数、下次执行时间等）
   - 实现 `/debug scheduled-tasks` 命令查看内部状态
   - 记录任务调度决策日志

4. **功能扩展**
   - 支持暂停/恢复单个任务
   - 添加任务执行历史查看
   - 支持任务执行结果回调

5. **容错性**
   - 添加任务回调的 try-catch 保护
   - 实现任务执行失败重试机制
   - 添加优雅关闭时的任务状态保存

### 相关 Issue/PR 参考

- `#19931`: 定时任务功能基础实现
- `#20425`: 助手模式空闲行为变更
- `#20467`: Brief 模式工具结果处理
