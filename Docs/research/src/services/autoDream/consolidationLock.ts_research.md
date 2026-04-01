# consolidationLock.ts 深度研究文档

## 1. 场景与职责

### 1.1 模块定位
`consolidationLock.ts` 是 auto-dream 模块的**分布式锁管理器**，负责协调多个 Claude Code 进程之间的记忆整合任务互斥执行。

### 1.2 业务场景
- **多进程冲突避免**：用户可能在同一项目的多个终端同时运行 Claude Code
- **崩溃恢复**：进程崩溃后，锁应该能被其他进程安全地重新获取
- **时间戳管理**：锁文件的 mtime 同时充当 `lastConsolidatedAt` 时间戳

### 1.3 核心职责
1. **互斥锁管理**：确保同一时间只有一个进程执行记忆整合
2. **时间戳持久化**：通过文件 mtime 记录上次整合时间
3. **死锁检测**：通过 PID 检查识别并回收僵尸锁
4. **会话扫描**：提供基于 mtime 的高效会话文件筛选

---

## 2. 功能点目的

### 2.1 锁文件设计
```
文件路径: <autoMemPath>/.consolidate-lock
文件内容: 持有者 PID（字符串）
文件 mtime: 上次整合时间（lastConsolidatedAt）
```

**设计优势**：
- **原子性**：文件创建/写入是原子操作
- **持久化**：mtime 自动持久化，无需额外写入
- **可恢复**：通过 PID 检查可以识别持有者是否存活

### 2.2 过期策略
- **HOLDER_STALE_MS = 60 * 60 * 1000** (1小时)
- 即使 PID 存活，超过 1 小时也视为过期（防止 PID 重用导致的误判）

### 2.3 锁的生命周期
```
1. tryAcquireConsolidationLock() → 获取锁，返回 priorMtime
2. 执行记忆整合任务
3. 成功: 锁文件保留，mtime = 任务开始时间
   失败: rollbackConsolidationLock(priorMtime) → 恢复 mtime 或删除文件
```

---

## 3. 具体技术实现

### 3.1 核心数据结构
```typescript
const LOCK_FILE = '.consolidate-lock'
const HOLDER_STALE_MS = 60 * 60 * 1000  // 1小时

function lockPath(): string {
  return join(getAutoMemPath(), LOCK_FILE)
}
```

### 3.2 关键函数实现

#### 3.2.1 `readLastConsolidatedAt()` - 读取上次整合时间
```typescript
export async function readLastConsolidatedAt(): Promise<number> {
  try {
    const s = await stat(lockPath())
    return s.mtimeMs  // 返回毫秒时间戳
  } catch {
    return 0  // 文件不存在，返回 0（表示从未整合）
  }
}
```

**特点**：
- 单次 `stat` 系统调用，性能开销极小
- 文件不存在时返回 0，使时间门检查通过

#### 3.2.2 `tryAcquireConsolidationLock()` - 尝试获取锁
```typescript
export async function tryAcquireConsolidationLock(): Promise<number | null> {
  const path = lockPath()
  
  // 1. 读取现有锁信息
  let mtimeMs: number | undefined
  let holderPid: number | undefined
  try {
    const [s, raw] = await Promise.all([stat(path), readFile(path, 'utf8')])
    mtimeMs = s.mtimeMs
    const parsed = parseInt(raw.trim(), 10)
    holderPid = Number.isFinite(parsed) ? parsed : undefined
  } catch {
    // ENOENT — 无现有锁
  }
  
  // 2. 检查锁是否被持有
  if (mtimeMs !== undefined && Date.now() - mtimeMs < HOLDER_STALE_MS) {
    if (holderPid !== undefined && isProcessRunning(holderPid)) {
      // PID 存活，锁被持有
      logForDebugging(`[autoDream] lock held by live PID ${holderPid}`)
      return null
    }
    // PID 死亡或无法解析 — 可以回收
  }
  
  // 3. 创建/覆盖锁文件
  await mkdir(getAutoMemPath(), { recursive: true })
  await writeFile(path, String(process.pid))
  
  // 4. 验证写入（防止竞争条件）
  let verify: string
  try {
    verify = await readFile(path, 'utf8')
  } catch {
    return null
  }
  if (parseInt(verify.trim(), 10) !== process.pid) {
    return null  // 竞争失败
  }
  
  return mtimeMs ?? 0  // 返回获取锁之前的时间戳（用于回滚）
}
```

**竞争处理**：
- 两个进程同时尝试获取锁时，都写入自己的 PID
- 最后写入的进程获胜
- 失败者通过重新读取验证发现自己失败，返回 `null`

#### 3.2.3 `rollbackConsolidationLock()` - 回滚锁状态
```typescript
export async function rollbackConsolidationLock(
  priorMtime: number,
): Promise<void> {
  const path = lockPath()
  try {
    if (priorMtime === 0) {
      // 之前没有锁文件，删除恢复初始状态
      await unlink(path)
      return
    }
    // 恢复之前的 mtime，清空内容（表示当前进程不再持有）
    await writeFile(path, '')
    const t = priorMtime / 1000  // utimes 需要秒级时间戳
    await utimes(path, t, t)
  } catch (e: unknown) {
    logForDebugging(`[autoDream] rollback failed: ${(e as Error).message}`)
  }
}
```

**使用场景**：
- Forked agent 执行失败时，恢复时间戳以便下次可以重新触发
- 用户手动终止任务时，通过 `DreamTask.kill()` 调用

#### 3.2.4 `listSessionsTouchedSince()` - 扫描变更会话
```typescript
export async function listSessionsTouchedSince(
  sinceMs: number,
): Promise<string[]> {
  const dir = getProjectDir(getOriginalCwd())
  const candidates = await listCandidates(dir, true)  // true = 需要 stat
  return candidates.filter(c => c.mtime > sinceMs).map(c => c.sessionId)
}
```

**性能优化**：
- 使用 `listCandidates` 批量获取文件 stat 信息
- 基于 mtime 快速过滤，避免读取文件内容
- 只返回 session ID，延迟加载详细信息

#### 3.2.5 `recordConsolidation()` - 手动记录整合
```typescript
export async function recordConsolidation(): Promise<void> {
  try {
    await mkdir(getAutoMemPath(), { recursive: true })
    await writeFile(lockPath(), String(process.pid))
  } catch (e: unknown) {
    logForDebugging(`[autoDream] recordConsolidation write failed: ${(e as Error).message}`)
  }
}
```

**用途**：
- 用户手动执行 `/dream` 命令时调用
- 乐观更新，不等待技能完成
- 防止 auto-dream 立即重复触发

### 3.3 进程存活检查
```typescript
import { isProcessRunning } from '../../utils/genericProcessUtils.js'

export function isProcessRunning(pid: number): boolean {
  if (pid <= 1) return false  // 0=当前进程组, 1=init
  try {
    process.kill(pid, 0)  // 信号 0 用于检测进程是否存在
    return true
  } catch {
    return false  // ESRCH (不存在) 或 EPERM (无权限)
  }
}
```

**注意**：
- `EPERM`（权限不足）时返回 `false`，这是保守策略（宁可误判为死亡，也不阻塞）
- 需要 root 权限查看的其他用户进程会被误判为死亡

---

## 4. 关键代码路径与文件引用

### 4.1 调用方
| 文件 | 函数 | 用途 |
|------|------|------|
| `autoDream.ts:133` | `readLastConsolidatedAt()` | 时间门检查 |
| `autoDream.ts:156` | `listSessionsTouchedSince()` | 会话门检查 |
| `autoDream.ts:182` | `tryAcquireConsolidationLock()` | 获取执行锁 |
| `autoDream.ts:270` | `rollbackConsolidationLock()` | 失败回滚 |
| `DreamTask.ts:154` | `rollbackConsolidationLock()` | 用户终止任务 |

### 4.2 调用链
```
tryAcquireConsolidationLock (consolidationLock.ts:46)
  ├── stat(lockPath) → 获取现有 mtime
  ├── readFile(lockPath) → 获取持有者 PID
  ├── isProcessRunning(pid) → genericProcessUtils.ts
  ├── mkdir(getAutoMemPath()) → 确保目录存在
  ├── writeFile(path, process.pid) → 写入当前 PID
  └── readFile(path) → 验证写入成功
```

### 4.3 文件关系
```
<autoMemPath>/
  ├── .consolidate-lock     # 锁文件（本模块管理）
  ├── MEMORY.md             # 记忆索引入口
  └── <topic-files>.md      # 主题记忆文件
```

---

## 5. 依赖与外部交互

### 5.1 导入依赖
```typescript
import { mkdir, readFile, stat, unlink, utimes, writeFile } from 'fs/promises'
import { join } from 'path'
import { getOriginalCwd } from '../../bootstrap/state.js'
import { getAutoMemPath } from '../../memdir/paths.js'
import { logForDebugging } from '../../utils/debug.js'
import { isProcessRunning } from '../../utils/genericProcessUtils.js'
import { listCandidates } from '../../utils/listSessionsImpl.js'
import { getProjectDir } from '../../utils/sessionStorage.js'
```

### 5.2 关键依赖说明
| 依赖 | 用途 |
|------|------|
| `fs/promises` | 文件系统操作 |
| `getAutoMemPath()` | 确定锁文件存储位置 |
| `isProcessRunning()` | 检查锁持有者是否存活 |
| `listCandidates()` | 高效列出会话文件 |
| `getProjectDir()` | 获取当前项目会话目录 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 PID 重用攻击
- **风险**：Linux PID 重用可能导致误判锁持有者为存活
- **缓解**：1小时过期时间限制风险窗口
- **评估**：实际风险极低，需要精确的时间窗口匹配

#### 6.1.2 时钟回拨
- **风险**：系统时间回拨可能导致 `HOLDER_STALE_MS` 计算错误
- **影响**：锁可能被认为未过期，实际已过期很久
- **缓解**：使用单调时钟（`process.hrtime()`）代替 `Date.now()`

#### 6.1.3 文件系统权限
- **风险**：无写入权限时锁操作失败
- **处理**：错误被捕获并记录，不会阻塞主流程
- **潜在问题**：静默失败可能导致重复触发

### 6.2 边界条件

| 场景 | 行为 |
|------|------|
| 首次运行（无锁文件） | `readLastConsolidatedAt()` 返回 0，立即触发 |
| 锁文件存在但为空 | `parseInt` 返回 NaN，视为死亡 PID，可回收 |
| 锁文件存在但 PID 为 0/1 | `isProcessRunning` 返回 false，可回收 |
| 进程无权限检查 PID | `isProcessRunning` 返回 false（保守策略）|
| 回滚时文件已被删除 | 捕获错误，记录日志，继续执行 |
| 回滚 mtime = 0 | 删除锁文件，恢复初始状态 |

### 6.3 改进建议

#### 6.3.1 使用单调时钟
```typescript
// 建议：使用 hrtime 避免时钟回拨问题
import { hrtime } from 'process'

const startTime = hrtime.bigint()
// ...
const elapsedMs = Number(hrtime.bigint() - startTime) / 1_000_000
```

#### 6.3.2 增加锁心跳机制
```typescript
// 建议：长时间任务中定期更新锁文件
async function refreshLock(): Promise<void> {
  await writeFile(lockPath(), String(process.pid))
  // 更新 mtime 为当前时间，延长过期时间
}

// 在 autoDream 执行期间定期调用
const heartbeat = setInterval(refreshLock, 5 * 60 * 1000)  // 每5分钟
```

#### 6.3.3 增加锁信息丰富度
```typescript
// 建议：锁文件存储更多信息，便于调试
interface LockFileContent {
  pid: number
  startedAt: string  // ISO timestamp
  version: string    // Claude Code 版本
}

await writeFile(path, JSON.stringify({
  pid: process.pid,
  startedAt: new Date().toISOString(),
  version: MACRO.VERSION,
}))
```

#### 6.3.4 增加锁状态查询接口
```typescript
// 建议：提供查询当前锁状态的接口
export async function getLockStatus(): Promise<{
  held: boolean
  holderPid?: number
  heldSince?: number
  isStale: boolean
}> {
  // 实现锁状态查询
}
```

#### 6.3.5 改进竞争检测
```typescript
// 建议：使用文件锁（flock）替代 PID 验证
import { open } from 'fs/promises'

async function acquireLockWithFlock(): Promise<boolean> {
  const fd = await open(lockPath(), 'w')
  try {
    // 非阻塞尝试获取排他锁
    await fd.write('')  // 写入 PID
    return true
  } catch {
    await fd.close()
    return false
  }
}
```

### 6.4 测试建议
建议增加以下测试场景：
1. 两个进程同时尝试获取锁，只有一个成功
2. 锁持有者进程死亡后，其他进程可以获取锁
3. 锁持有者存活时，其他进程获取锁失败
4. 回滚操作正确恢复 mtime
5. 回滚操作正确处理 priorMtime = 0 的情况
6. 文件系统权限不足时的优雅降级
