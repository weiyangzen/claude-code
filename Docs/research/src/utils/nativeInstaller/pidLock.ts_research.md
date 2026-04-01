# `src/utils/nativeInstaller/pidLock.ts` 技术调研文档

## 场景与职责

`pidLock.ts` 是 Claude Code Native Installer 的**版本锁定核心模块**，负责为当前正在运行的 Claude Code 进程提供基于 PID（Process ID）的互斥锁机制。该模块主要解决以下场景问题：

1. **多进程版本目录冲突**：在 Native Installer 架构下，Claude Code 的不同版本被安装到独立的目录中。当多个进程（或同一进程的不同实例）尝试对同一版本目录进行更新、清理或执行操作时，必须避免并发冲突。
2. **替代传统的 mtime 锁定**：旧有的锁定机制基于文件修改时间（mtime），在进程异常崩溃后，锁文件可能持续保留长达 30 天，导致新版本无法被及时清理或更新。PID 锁定可以在进程结束后**立即**检测到锁已失效。
3. **诊断与运维支持**：通过 `Doctor.tsx` 等诊断界面，向用户展示当前系统中所有活跃的版本锁状态，帮助排查安装器卡顿、版本无法升级等问题。

该模块的核心职责包括：
- 提供功能开关（Feature Gate + 环境变量覆盖）控制是否启用 PID 锁定。
- 实现锁的获取、验证、持有（进程生命周期内）与释放。
- 检测锁是否失效（stale），包括 PID 存活检查、命令行校验、以及兜底的时间戳超时机制。
- 清理遗留锁文件，兼容旧版 `proper-lockfile` 生成的目录锁。

---

## 功能点目的

模块导出以下关键函数与类型，各自目的如下：

| 导出项 | 类型 | 目的 |
|--------|------|------|
| `isPidBasedLockingEnabled` | 函数 | 决定当前环境是否启用 PID 锁定。优先读取环境变量 `ENABLE_PID_BASED_VERSION_LOCKING`，若未设置则回退到 GrowthBook Feature Gate `tengu_pid_based_version_locking`，用于灰度发布与外部用户隔离。 |
| `tryAcquireLock` | 函数 | 尝试对指定版本路径获取锁。若锁已被其他活跃进程持有，则返回 `null`；否则原子写入锁文件并返回释放函数。包含竞态条件二次校验。 |
| `acquireProcessLifetimeLock` | 函数 | 获取锁并在当前进程存活期间一直持有。通过在 `process.on('exit' / 'SIGINT' / 'SIGTERM')` 上注册清理回调，确保进程正常退出时自动释放锁。 |
| `withLock` | 函数 | 以回调函数模式持有锁：获取锁后执行 `callback`，无论成功或失败都在 `finally` 中释放锁，简化调用方逻辑。 |
| `isLockActive` | 函数 | 判断指定锁文件是否仍代表一个活跃的锁。是锁获取前的核心判断逻辑，也是清理 stale 锁的依据。 |
| `readLockContent` | 函数 | 读取并反序列化锁文件内容，同时进行字段校验（`pid` 必须为数字、`version` 和 `execPath` 必须存在）。 |
| `getAllLockInfo` | 函数 | 扫描锁目录下所有 `.lock` 文件，返回供诊断使用的结构化信息数组（包含 PID 是否仍在运行、获取时间等）。 |
| `cleanupStaleLocks` | 函数 | 批量清理失效锁。既能清理 PID 锁（文件形式），也能清理旧版 `proper-lockfile` 遗留的目录锁（目录形式）。 |
| `VersionLockContent` | 类型 | 定义锁文件 JSON 的数据结构：`pid`、`version`、`execPath`、`acquiredAt`。 |
| `LockInfo` | 类型 | 定义诊断信息的数据结构，在 `VersionLockContent` 基础上增加了 `isProcessRunning` 布尔值和 `lockFilePath`。 |

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1. 锁文件格式与存储协议

锁文件采用**纯文本 JSON** 格式，字段定义如下：

```typescript
type VersionLockContent = {
  pid: number          // 持有锁的进程 ID
  version: string      // 版本名称（通常是 versionPath 的 basename）
  execPath: string     // 进程的可执行文件路径（process.execPath）
  acquiredAt: number   // 获取锁时的 Unix 时间戳（毫秒）
}
```

文件命名由调用方决定（通常以 `.lock` 结尾），统一存放在某个锁目录（`locksDir`）下。

### 2. 原子写入机制

`writeLockFile` 实现了**先写临时文件再重命名**的原子写入策略，防止锁文件内容在写入过程中被其他进程读取到不完整数据：

```typescript
const tempPath = `${lockFilePath}.tmp.${process.pid}.${Date.now()}`
writeFileSync_DEPRECATED(tempPath, jsonStringify(content, null, 2), { encoding: 'utf8', flush: true })
fs.renameSync(tempPath, lockFilePath)
```

- 临时文件名包含当前 PID 和时间戳，降低冲突概率。
- `flush: true` 确保数据在重命名前已落盘。
- 若写入或重命名失败，会尽力删除临时文件，避免残留。

### 3. 进程存活检测（PID Check）

`isProcessRunning(pid)` 使用 Node.js 的 `process.kill(pid, 0)` 进行**非侵入式进程探测**：

- `signal 0` 在 POSIX 系统中不会真正向目标进程发送信号，仅检查调用方是否有向该进程发送信号的权限，从而推断进程是否存在。
- 对 `pid <= 1` 做短路返回 `false`：PID 0 通常指当前进程组，PID 1 是 init/systemd，均不应被视为合法的锁持有者。
- 若抛出异常（如 `ESRCH`），则判定进程已退出。

### 4. 命令行校验（PID 复用防护）

操作系统在进程退出后会复用 PID，单纯依赖 `process.kill` 存在误判风险。`isClaudeProcess(pid, expectedExecPath)` 通过 `getProcessCommand(pid)` 读取目标进程的命令行进行二级校验：

- 若当前 PID 等于 `process.pid`，直接判定为合法（自锁）。
- 若无法读取命令行（权限不足或平台不支持），选择**保守信任** PID 检查结果，返回 `true`。
- 若能读取命令行，则检查命令行字符串（小写后）是否包含 `'claude'` 或 `expectedExecPath`，不匹配则视为 stale。

### 5. 锁活跃性综合判定（`isLockActive`）

`isLockActive` 是模块最核心的状态机判断逻辑，采用三层防御：

1. **主检查**：`isProcessRunning(pid)` —— 进程不存在则直接判定 stale。
2. **二级检查**：`isClaudeProcess(pid, execPath)` —— 进程存在但命令行不像 Claude，则判定 stale 并记录调试日志。
3. **兜底检查**：若锁文件的 mtime 超过 `FALLBACK_STALE_MS`（2 小时），再次调用 `isProcessRunning` 做最终确认。该层用于应对 `process.kill` 在某些边缘场景下（如僵尸进程、权限变更）可能给出的不明确结果。

### 6. 竞态条件防护（Race Condition Guard）

`tryAcquireLock` 在原子写入锁文件后，会**立即重新读取**锁文件内容进行验证：

```typescript
const verifyContent = readLockContent(lockFilePath)
if (verifyContent?.pid !== process.pid) {
  return null  // 另一个进程赢得了竞态
}
```

这解决了两个进程几乎同时检测到锁为 stale 并同时写入的问题：后写入的进程虽然覆盖了文件，但先写入的进程在验证阶段会发现文件内容已被篡改，从而主动放弃锁。

### 7. 进程生命周期锁的清理策略

`acquireProcessLifetimeLock` 在成功获取锁后，向 Node.js 进程注册了三个事件监听器：

- `process.on('exit', cleanup)`
- `process.on('SIGINT', cleanup)`
- `process.on('SIGTERM', cleanup)`

`cleanup` 内部调用 `release()`，而 `release()` 会再次读取锁文件，确认 PID 仍为自己时才执行 `unlinkSync`。这防止了在信号处理期间锁被其他进程接管而导致误删。

### 8. 遗留锁兼容清理

`cleanupStaleLocks` 在扫描锁目录时，通过 `fs.lstatSync` 区分两种锁形态：

- **目录类型**（`stats.isDirectory()`）：认定为旧版 `proper-lockfile` 生成的目录锁，在启用 PID 锁定的前提下**无条件删除**。
- **文件类型**：读取后走 `isLockActive` 判定，失效则删除。

---

## 关键代码路径与文件引用

### 本模块

- **`src/utils/nativeInstaller/pidLock.ts`**：本文档的研究对象，包含 PID 锁的全部实现逻辑。

### 调用方

- **`src/utils/nativeInstaller/installer.ts`**：
  - 调用 `isPidBasedLockingEnabled()` 决定使用何种锁定策略。
  - 调用 `acquireProcessLifetimeLock()` 在进程启动时锁定当前正在使用的版本目录。
  - 调用 `withLock()` 在执行需要互斥的版本操作（如安装、更新）时短期持有锁。
  - 调用 `isLockActive()` 和 `readLockContent()` 在操作前检查锁状态。
  - 调用 `cleanupStaleLocks()` 在维护流程中清理历史遗留锁。

- **`src/screens/Doctor.tsx`**：
  - 调用 `isPidBasedLockingEnabled()` 判断是否在诊断界面展示 PID 锁相关面板。
  - 调用 `getAllLockInfo()` 获取所有锁的列表，渲染到 UI 中供用户查看。
  - 调用 `cleanupStaleLocks()` 提供一键清理失效锁的按钮功能。
  - 导入 `LockInfo` 类型用于 TypeScript 类型约束与 UI 渲染。

### 被调用的底层依赖模块

- **`src/services/analytics/growthbook.js`**：提供 `getFeatureValue_CACHED_MAY_BE_STALE`，用于灰度开关。
- **`src/utils/debug.js`**：提供 `logForDebugging`，输出带模块前缀的调试日志。
- **`src/utils/envUtils.js`**：提供 `isEnvTruthy` 和 `isEnvDefinedFalsy`，解析环境变量布尔值。
- **`src/utils/errors.js`**：提供 `isENOENT` 和 `toError`，统一错误类型判断与转换。
- **`src/utils/fsOperations.js`**：提供 `getFsImplementation()`，返回经过抽象或 shim 的同步文件系统操作对象。
- **`src/utils/genericProcessUtils.js`**：提供 `getProcessCommand(pid)`，跨平台读取进程命令行。
- **`src/utils/log.js`**：提供 `logError`，输出结构化错误日志。
- **`src/utils/slowOperations.js`**：提供 `jsonParse`、`jsonStringify`、`writeFileSync_DEPRECATED`，封装可能涉及性能开销或未来需要迁移的同步 I/O 操作。

---

## 依赖与外部交互

### Node.js 内置模块

- **`path`**（`basename`, `join`）：用于构造锁文件路径和提取版本名称。
- **`process`**（全局对象）：
  - `process.pid`：获取当前进程 ID。
  - `process.execPath`：获取当前可执行文件路径。
  - `process.env.ENABLE_PID_BASED_VERSION_LOCKING`：读取功能开关环境变量。
  - `process.kill(pid, 0)`：非侵入式检测目标进程是否存在。
  - `process.on('exit' / 'SIGINT' / 'SIGTERM')`：注册进程退出时的锁释放回调。

### 外部系统/平台交互

- **操作系统进程表**：通过 `process.kill` 和 `getProcessCommand` 与操作系统进程管理子系统交互。
  - 在 Linux/macOS 上，这通常涉及 `/proc/<pid>/cmdline` 或 `ps` 系统调用。
  - 在 Windows 上，可能涉及 WMI 或 `wmic` 查询（具体取决于 `genericProcessUtils.js` 的实现）。
- **GrowthBook 服务**：通过 `growthbook.js` 获取远程功能开关配置，模块内部做了缓存（`CACHED_MAY_BE_STALE`），意味着开关状态在进程生命周期内通常是静态的，不会实时响应服务端变更。

### 数据流总结

```
installer.ts / Doctor.tsx
        │
        ▼
pidLock.ts (本模块)
        │
        ├──► growthbook.js (Feature Gate)
        ├──► genericProcessUtils.js (OS 进程命令行查询)
        ├──► fsOperations.js (抽象 FS)
        └──► debug.js / log.js (日志输出)
```

---

## 风险、边界与改进建议

### 1. PID 复用风险（已部分缓解但非绝对安全）

**风险**：操作系统在进程退出后会回收并重新分配 PID。如果原锁持有进程退出，新分配的同名 PID 恰好运行了非 Claude 程序，`isProcessRunning` 会返回 `true`，导致锁被误判为活跃。

**现有缓解**：`isClaudeProcess` 通过命令行进行二次校验。

**边界**：
- 若攻击者（或巧合）在新 PID 上启动了一个命令行包含 `"claude"` 的进程，校验仍会被绕过。
- 某些平台（如受限容器、Windows）可能无法读取其他进程的命令行，此时系统会**保守信任** PID 检查，风险敞口增大。

**改进建议**：
- 引入**更严格的进程身份校验**，例如对比进程启动时间戳（start time）与锁文件中的 `acquiredAt`，确保 PID 复用后的新进程启动时间晚于锁获取时间（在 Linux 上可通过 `/proc/<pid>/stat` 读取）。
- 在锁文件中增加一个**随机 nonce**，获取锁后通过进程间通信（如 Unix Domain Socket 或临时文件心跳）验证持有进程是否知道该 nonce。

### 2. 竞态条件与原子性边界

**风险**：`tryAcquireLock` 中的竞态防护依赖“写入后重读”，但读取和验证之间仍然存在极短的时间窗口。在极高并发场景下，两个进程可能都通过验证（例如第三个进程在验证之后才写入）。

**边界**：当前实现没有使用操作系统级别的文件锁（如 `flock`、`lockf`、Windows `LockFile`），完全依赖用户态的文件重命名和重读校验。

**改进建议**：
- 在 `writeLockFile` 中结合 `fs.openSync` 与 `O_EXCL` 标志创建临时文件，确保创建操作本身具有原子性（当前已用时间戳和 PID 降低冲突，但 `O_EXCL` 更严谨）。
- 考虑引入操作系统文件锁作为底层互斥原语，将当前 JSON 锁文件作为元数据层，目录锁清理逻辑可保留。

### 3. 进程异常退出导致锁泄漏

**风险**：`acquireProcessLifetimeLock` 仅在 `exit`、`SIGINT`、`SIGTERM` 上注册了清理回调。若进程因 `SIGKILL`、`uncaughtException`、`unhandledRejection` 或硬件断电而终止，清理回调不会执行，锁文件将残留。

**边界**：虽然 `isLockActive` 会在下次锁获取时检测到 PID 已不存在并清理，但这要求有新的进程尝试获取同一版本的锁。如果该版本长期无人使用，锁文件将成为磁盘垃圾。

**改进建议**：
- 增加 `process.on('uncaughtException', cleanup)` 与 `process.on('unhandledRejection', cleanup)` 监听器（需注意 Node.js 在 `uncaughtException` 后的默认行为是退出，因此清理仍有效）。
- `cleanupStaleLocks` 可被调用方（如 installer.ts）在每次启动时全局执行一次，作为兜底清理策略。

### 4. 同步 I/O 与异步函数的混合

**风险**：`tryAcquireLock` 和 `withLock` 被声明为 `async` 函数，但内部大量使用了同步文件系统 API（`readFileSync`、`writeFileSync_DEPRECATED`、`renameSync`、`unlinkSync`、`statSync`、`lstatSync`）。这会导致事件循环阻塞，尤其在锁目录位于网络文件系统（NFS）或高延迟存储上时，可能显著影响应用响应性。

**改进建议**：
- 评估将核心 I/O 操作迁移为异步版本（`fs.promises`）。
- 若必须保留同步 I/O（例如为了简化锁获取的原子性逻辑），应在函数签名中移除 `async` 关键字，避免调用方误以为该操作是非阻塞的。

### 5. 平台差异与权限问题

**风险**：`process.kill(pid, 0)` 在 Windows 上的行为与 POSIX 系统存在差异：Windows 不严格支持 signal 0 的语义，Node.js 内部可能通过其他机制模拟，结果可能不如 POSIX 可靠。

**风险**：`getProcessCommand` 在 macOS 上可能需要 root 权限才能读取其他用户的进程命令行（取决于 SIP 和权限配置），在读取失败时系统选择信任 PID，降低了安全性。

**改进建议**：
- 在 Windows 环境下，考虑使用 `tasklist` 或 Node.js 的 `process._getActiveHandles` 等替代方案补充进程存在性检查。
- 对 `getProcessCommand` 的失败增加更细粒度的日志记录，区分“权限拒绝”与“进程不存在”，避免在诊断时产生误导。

### 6. `FALLBACK_STALE_MS` 的语义矛盾

**风险**：`isLockActive` 中的兜底逻辑检查锁文件的 `mtime`，但锁文件内容中已有 `acquiredAt` 字段。如果外部工具（如 `touch`）修改了锁文件的 mtime，可能导致 2 小时超时被重置，使一个实际上已 stale 的锁被延长寿命。

**改进建议**：
- 优先使用锁文件 JSON 内容中的 `acquiredAt` 作为时间基准，而非文件系统的 `mtime`，确保超时判定不受外部文件操作影响。

### 7. 日志与可观测性

**风险**：当前模块主要使用 `logForDebugging` 输出调试信息，在生产环境中这些日志可能被关闭，导致排查锁竞争问题时缺乏审计线索。

**改进建议**：
- 对锁的获取、释放、竞争失败、stale 检测等关键事件增加结构化日志（structured logging），包含 `version`、`pid`、`execPath`、`acquiredAt` 等字段，便于后续通过日志平台进行聚合分析。
- 在 `getAllLockInfo` 中补充锁的“stale 原因”字段（如 `pid_not_running`、`command_mismatch`、`timeout`），提升诊断界面的信息密度。
