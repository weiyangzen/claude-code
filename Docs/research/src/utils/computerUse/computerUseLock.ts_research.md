# computerUseLock.ts 研究文档

## 场景与职责

本文件实现 **Computer Use 功能的进程级文件锁机制**，确保同一时刻只有一个 Claude Code 会话可以控制计算机。这是多会话安全的关键组件，防止并发控制冲突。

核心职责：
1. **原子锁获取**：使用 O_EXCL 标志实现测试并设置的原子操作
2. **会话识别**：通过 sessionId 和 PID 标识锁持有者
3. **陈旧锁恢复**：检测并回收已死亡进程的锁
4. **清理注册**：注册进程退出时的自动清理

## 功能点目的

### 1. 锁状态检查 (`checkComputerUseLock`)
- 用于 `request_access` / `list_granted_applications` 工具
- 遵循 `defersLockAcquire` 契约：检查但不获取锁
- 执行陈旧 PID 恢复，防止死会话阻塞权限请求

### 2. 锁获取 (`tryAcquireComputerUseLock`)
- 三种返回状态：
  - `acquired` + `fresh: true`：首次获取，触发进入通知
  - `acquired` + `fresh: false`：重入，同一会话已持有
  - `blocked` + `by`：被其他活动会话阻塞
- 使用 O_EXCL (`flag: 'wx'`) 实现原子创建

### 3. 本地锁状态检查 (`isLockHeldLocally`)
- 零系统调用检查
- 用于 `cleanup.ts` 门控，避免非 CU 回合触碰磁盘
- 基于 `unregisterCleanup` 变量状态

### 4. 锁释放 (`releaseComputerUseLock`)
- 仅当当前会话持有锁时才释放
- 返回 `true` 表示实际删除了文件（用于触发退出通知）
- 幂等操作：重复调用返回 `false`

## 具体技术实现

### 核心数据结构

```typescript
type ComputerUseLock = {
  readonly sessionId: string
  readonly pid: number
  readonly acquiredAt: number
}

type AcquireResult =
  | { readonly kind: 'acquired'; readonly fresh: boolean }
  | { readonly kind: 'blocked'; readonly by: string }

type CheckResult =
  | { readonly kind: 'free' }
  | { readonly kind: 'held_by_self' }
  | { readonly kind: 'blocked'; readonly by: string }
```

### 锁文件位置

```typescript
const LOCK_FILENAME = 'computer-use.lock'

function getLockPath(): string {
  return join(getClaudeConfigHomeDir(), LOCK_FILENAME)
}
// 例如: ~/.config/claude/computer-use.lock
```

### 关键流程

#### `tryAcquireComputerUseLock()` - 锁获取

```
输入: 无（使用全局 sessionId 和 process.pid）
输出: AcquireResult

流程:
1. 准备锁数据: { sessionId, pid, acquiredAt }
2. 确保配置目录存在
3. 尝试独占创建 (O_EXCL)
   - 成功: 注册清理处理器，返回 fresh
4. 读取现有锁
5. 检查所有权:
   - 同一会话: 返回 reentrant
   - 其他活动进程: 返回 blocked
6. 陈旧锁恢复:
   - 记录调试日志
   - 删除锁文件
   - 重试独占创建
   - 如果仍然失败，返回 blocked（可能是并发竞争）
```

#### `checkComputerUseLock()` - 锁检查

```
输入: 无
输出: CheckResult

流程:
1. 读取现有锁
2. 无锁: 返回 free
3. 同一会话: 返回 held_by_self
4. 检查 PID 存活:
   - 存活: 返回 blocked
   - 死亡: 删除锁，返回 free
```

#### PID 存活检测

```typescript
function isProcessRunning(pid: number): boolean {
  try {
    process.kill(pid, 0)
    return true
  } catch {
    return false
  }
}
```
- 使用 signal 0 探测（不实际发送信号）
- 存在 PID 复用小概率风险

### 清理注册

```typescript
function registerLockCleanup(): void {
  unregisterCleanup?.()
  unregisterCleanup = registerCleanup(async () => {
    await releaseComputerUseLock()
  })
}
```
- 使用 `cleanupRegistry.ts` 注册进程退出清理
- 确保即使回合结束清理未执行，锁也能释放

## 关键代码路径与文件引用

### 本文件导出
- `checkComputerUseLock()` - 检查锁状态
- `tryAcquireComputerUseLock()` - 尝试获取锁
- `releaseComputerUseLock()` - 释放锁
- `isLockHeldLocally()` - 本地锁状态检查

### 调用方
- `src/utils/computerUse/wrapper.tsx:182` - `checkCuLock` 回调
- `src/utils/computerUse/wrapper.tsx:208` - `acquireCuLock` 回调
- `src/utils/computerUse/cleanup.ts:66` - `isLockHeldLocally` 和 `releaseComputerUseLock`

### 依赖文件
- `src/bootstrap/state.ts` - `getSessionId`
- `src/utils/cleanupRegistry.ts` - `registerCleanup`
- `src/utils/debug.ts` - `logForDebugging`
- `src/utils/envUtils.ts` - `getClaudeConfigHomeDir`
- `src/utils/slowOperations.ts` - `jsonParse`, `jsonStringify`
- `src/utils/errors.ts` - `getErrnoCode`

## 依赖与外部交互

### 外部包依赖
- Node.js `fs/promises` - 文件操作
- Node.js `path` - 路径处理

### 文件系统交互
- 创建/读取/删除锁文件（JSON 格式）
- 使用 O_EXCL 标志确保原子性
- 文件权限继承自用户 umask

### 并发安全
- O_EXCL 保证最多一个进程看到创建成功
- 并发恢复竞争时，只有一个能成功创建
- 其他进程读取到获胜者的锁数据

## 风险、边界与改进建议

### 已知风险

1. **PID 复用风险**：
   - 如果持有进程退出且新进程分配到相同 PID，检测会返回 true
   - 注释说明：这在实践中极不可能发生
   - 缓解：使用 sessionId + PID 双重验证

2. **网络文件系统**：
   - O_EXCL 在某些 NFS 配置下可能不保证原子性
   - 可能影响多机器共享配置目录的场景

3. **权限问题**：
   - 如果锁文件由不同用户创建，可能无法删除
   - 可能导致锁永久阻塞

### 边界情况

1. **损坏的锁文件**：
   - `readLock()` 在解析失败时返回 undefined
   - 视为陈旧锁，触发恢复流程

2. **并发竞争**：
   - 两个会话同时恢复同一陈旧锁
   - 只有一个能成功创建，另一个读取到获胜者

3. **快速重启**：
   - 同一会话 ID 快速重启
   - `held_by_self` 检测正确处理

### 改进建议

1. **锁文件增强**：
   - 添加更多元数据（主机名、启动时间）
   - 帮助检测跨机器的锁冲突

2. **心跳机制**：
   - 定期更新锁文件的 `acquiredAt`
   - 允许检测长时间无响应的持有者

3. **强制释放**：
   - 添加命令行选项强制释放锁
   - 用于手动恢复卡住的状态

4. **监控和告警**：
   - 记录锁获取/释放的指标
   - 监控锁持有时间，检测异常长的会话

5. **优雅降级**：
   - 考虑在锁机制失败时提供只读模式
   - 允许用户选择强制获取锁

6. **测试覆盖**：
   - 添加并发测试模拟多会话竞争
   - 测试 PID 复用边界情况
   - 测试文件系统权限失败场景
