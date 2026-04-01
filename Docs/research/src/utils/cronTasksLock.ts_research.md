# cronTasksLock.ts 深度研究文档

## 场景与职责

`cronTasksLock.ts` 是 Claude Code 定时任务系统的 **分布式锁管理模块**，负责协调同一项目目录下多个 Claude 会话之间的任务调度权。它确保同一时间只有一个会话负责触发定时任务，防止重复执行。

### 核心职责

1. **锁获取**：尝试原子性地获取调度器锁
2. **锁释放**：在会话结束或主动停止时释放锁
3. **僵尸锁检测**：检测并清理已死亡进程持有的锁
4. **锁恢复**：在检测到僵尸锁时自动接管调度权

### 使用场景

- **REPL 模式**：启动调度器时获取锁，关闭时释放
- **Daemon 模式**：SDK/headless 模式同样需要锁协调
- **多终端场景**：同一项目的多个 Claude 实例通过锁协调
- **崩溃恢复**：进程崩溃后，其他会话自动接管锁

---

## 功能点目的

### 1. 锁获取 (`tryAcquireSchedulerLock`)

**目的**：尝试获取调度器锁，成为负责任务调度的会话。

**获取流程**：
1. 尝试使用 `O_EXCL` 标志（`wx`）原子创建锁文件
2. 如果创建成功 → 获取锁成功
3. 如果文件已存在 → 读取现有锁内容
4. 如果锁是我们的（相同 sessionId）→ 更新 PID，获取成功
5. 如果锁是其他存活进程的 → 获取失败，进入被动模式
6. 如果锁是僵尸进程（PID 不存在）→ 删除锁，重试创建

**锁文件格式**：
```json
{
  "sessionId": "<会话ID>",
  "pid": 12345,
  "acquiredAt": 1699999999999
}
```

### 2. 锁释放 (`releaseSchedulerLock`)

**目的**：在会话结束时释放持有的锁。

**释放流程**：
1. 取消注册的清理函数
2. 读取当前锁内容
3. 如果锁是我们的 → 删除锁文件
4. 如果锁不是我们的 → 静默返回（可能已被其他会话接管）

### 3. 僵尸锁检测与恢复

**目的**：处理持有锁的进程崩溃的情况。

**检测机制**：
- 使用 `isProcessRunning(pid)` 检查 PID 是否存在
- 如果 PID 不存在 → 锁是僵尸锁

**恢复流程**：
1. 检测到僵尸锁
2. 记录日志：`recovering stale scheduler lock from PID ${pid}`
3. 尝试删除锁文件
4. 重新尝试获取锁
5. 如果其他会话同时尝试恢复，只有一个会成功（O_EXCL 保证）

### 4. 清理注册

**目的**：确保进程异常退出时也能释放锁。

**实现**：
- 获取锁后注册清理函数到 `cleanupRegistry`
- 清理函数调用 `releaseSchedulerLock`
- 正常退出和异常退出（SIGINT 等）都会触发清理

---

## 具体技术实现

### 核心数据结构

```typescript
type SchedulerLock = {
  sessionId: string   // 锁持有者标识（REPL 用 sessionId，Daemon 用 UUID）
  pid: number         // 进程 ID（存活检测）
  acquiredAt: number  // 获取时间戳
}

type SchedulerLockOptions = {
  dir?: string           // 项目目录（SDK 模式必需）
  lockIdentity?: string  // 锁标识（SDK 模式使用进程 UUID）
}
```

### 锁文件路径

```typescript
const LOCK_FILE_REL = join('.claude', 'scheduled_tasks.lock')
// 实际路径: <projectRoot>/.claude/scheduled_tasks.lock
```

### 锁获取算法

```
tryAcquireSchedulerLock(opts)
    ↓
确定 sessionId = opts.lockIdentity ?? getSessionId()
创建锁对象 = { sessionId, pid: process.pid, acquiredAt: Date.now() }
    ↓
tryCreateExclusive(lock, dir)
    ↓
[成功] → 注册清理 → 返回 true
[失败] → readLock(dir)
    ↓
[锁是我们的] → 更新 PID → 返回 true
[锁是其他存活进程] → 记录日志 → 返回 false
[锁是僵尸] → unlink → tryCreateExclusive → 返回结果
```

### 目录创建处理

```typescript
// 如果 .claude/ 目录不存在，创建它后重试
if (code === 'ENOENT') {
  await mkdir(dirname(path), { recursive: true })
  // 重试创建锁文件
}
```

### 重复日志抑制

```typescript
// 避免在轮询时重复输出 "held by X" 日志
let lastBlockedBy: string | undefined

if (lastBlockedBy !== existing.sessionId) {
  lastBlockedBy = existing.sessionId
  logForDebugging(`[ScheduledTasks] scheduler lock held by...`)
}
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `tryAcquireSchedulerLock` | 111-173 | 尝试获取调度器锁 |
| `releaseSchedulerLock` | 178-195 | 释放调度器锁 |

### 导出类型

| 类型 | 行号 | 用途 |
|------|------|------|
| `SchedulerLockOptions` | 40-43 | 锁选项（SDK 模式使用） |

### 依赖文件

```
cronTasksLock.ts
├── 被调用方（上游）
│   └── src/utils/cronScheduler.ts           # 调度器核心
├── 被依赖模块（下游）
│   ├── src/bootstrap/state.js               # getProjectRoot, getSessionId
│   ├── src/utils/cleanupRegistry.js         # registerCleanup
│   ├── src/utils/debug.js                   # logForDebugging
│   ├── src/utils/errors.js                  # getErrnoCode
│   ├── src/utils/genericProcessUtils.js     # isProcessRunning
│   ├── src/utils/json.js                    # safeParseJSON
│   ├── src/utils/lazySchema.js              # lazySchema
│   └── src/utils/slowOperations.js          # jsonStringify
└── 外部依赖
    ├── zod/v4                               # 锁内容验证
    ├── fs/promises                          # 文件操作
    └── path                                 # 路径处理
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `zod/v4` | 锁内容结构验证 |
| `fs/promises` | 异步文件操作 |
| `path` | 路径拼接 |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.js` | `getProjectRoot`, `getSessionId` | 获取项目目录和会话 ID |
| `src/utils/cleanupRegistry.js` | `registerCleanup` | 注册退出清理 |
| `src/utils/debug.js` | `logForDebugging` | 调试日志 |
| `src/utils/errors.js` | `getErrnoCode` | 错误码提取 |
| `src/utils/genericProcessUtils.js` | `isProcessRunning` | 进程存活检测 |
| `src/utils/json.js` | `safeParseJSON` | 安全 JSON 解析 |
| `src/utils/lazySchema.js` | `lazySchema` | 延迟加载 Zod 模式 |
| `src/utils/slowOperations.js` | `jsonStringify` | JSON 序列化 |

### 进程存活检测

```typescript
// 来自 genericProcessUtils.js
isProcessRunning(pid: number): boolean
// 实现：尝试 process.kill(pid, 0)，成功则进程存在
```

---

## 风险、边界与改进建议

### 已知风险

1. **PID 复用**
   - 僵尸锁的 PID 可能被新进程复用
   - 可能误判僵尸锁为存活锁，或反之
   - 概率较低但存在

2. **网络文件系统**
   - NFS 等网络文件系统可能不支持 O_EXCL 原子性
   - 可能导致锁竞争失败

3. **容器环境**
   - 容器内 PID 命名空间隔离，主机 PID 不可见
   - `isProcessRunning` 可能无法正确检测

4. **时钟回拨**
   - `acquiredAt` 用于调试，但不参与逻辑
   - 时钟回拨不影响锁功能

5. **锁文件损坏**
   - 锁文件内容损坏时视为僵尸锁
   - 可能导致不必要的锁争夺

### 边界情况

| 场景 | 处理 |
|------|------|
| 锁文件被手动删除 | 下一次获取会成功创建新锁 |
| 锁文件权限不足 | 抛出异常，上层处理 |
| 目录创建失败 | 抛出异常 |
| 同时恢复僵尸锁 | O_EXCL 保证只有一个成功 |
| 自身 PID 检测 | 总是认为自己存活 |
| 清理函数重复注册 | 先取消旧注册，再注册新 |

### 改进建议

1. **锁增强**
   - 使用更可靠的分布式锁（如基于文件的 advisory lock）
   - 添加锁续期机制，防止长时间持有不释放
   - 实现锁超时，自动释放长时间未更新的锁

2. **容器支持**
   - 检测容器环境，使用替代锁机制
   - 支持基于主机 PID 的检测（如果可行）

3. **可观测性**
   - 添加锁获取/释放事件日志
   - 记录锁持有时间指标
   - 实现 `/debug scheduler-lock` 命令查看锁状态

4. **容错性**
   - 添加锁文件损坏的自动修复
   - 实现锁获取超时，避免无限等待
   - 添加锁获取失败重试（带退避）

5. **测试覆盖**
   - 添加多进程竞争测试
   - 测试僵尸锁恢复场景
   - 测试容器环境下的行为

### 相关 Issue/PR 参考

- 本模块是 `#19931`（定时任务功能）的并发控制基础设施
- 设计模式参考 `computerUseLock.ts` 的实现
