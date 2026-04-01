# 研究文档：src/utils/nativeInstaller/installer.ts

## 场景与职责

`installer.ts` 是 Claude Code **原生安装器（Native Installer）的核心实现文件**，长达 1700+ 行。它负责管理原生二进制文件的完整生命周期：

- **目录结构管理**：基于 XDG 规范组织 `versions`（持久化二进制）、`staging`（临时下载缓存）、`locks`（并发锁文件）和 `~/.local/bin/claude`（用户可执行入口）。
- **版本安装与激活**：下载、校验、原子移动到最终路径，并创建/更新符号链接（Windows 下为复制）。
- **多进程安全**：支持 PID-based 和 mtime-based 两种锁机制，防止并发安装冲突。
- **版本清理与回退保护**：清理旧版本但保留当前运行版本和最近 2 个版本；锁定当前运行版本防止被误删。
- **安装健康检查**：验证 `claude` 命令是否存在、是否为有效二进制、是否已在 PATH 中。
- **迁移辅助**：从 npm 安装迁移到原生安装时，清理旧的 npm 包和 shell alias。

它是 `index.ts` 的唯一实现源，也是整个原生安装体验的底层引擎。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getPlatform()` / `getBinaryName()` | 确定当前平台标识（如 `macos-arm64`、`linux-x64-musl`）和二进制文件名 |
| `getBaseDirectories()` | 基于 XDG 返回所有原生安装器使用的目录路径 |
| `getVersionPaths()` | 创建必要目录并返回某版本的 `stagingPath` 和 `installPath` |
| `tryWithVersionLock()` | 对单个版本文件尝试获取锁，支持重试和两种锁后端 |
| `performVersionUpdate()` | 核心更新操作：下载（如需）→ 安装 → 更新 symlink |
| `updateLatest()` | 判断是否需要更新，处理 maxVersion/minimumVersion 逻辑，调用 `performVersionUpdate` |
| `installLatest()` / `installLatestImpl()` | 带 **in-process singleflight** 的公共入口，防止重复并发下载 |
| `checkInstall()` | 检查原生安装是否完整，返回需要用户操作的 `SetupMessage[]` |
| `lockCurrentVersion()` | 为当前运行版本获取进程生命周期锁，防止被清理 |
| `cleanupOldVersions()` | 启动时异步清理：旧 staging、旧 Windows exe、stale locks、过期版本二进制 |
| `cleanupNpmInstallations()` | 卸载全局 npm 包并清理本地安装目录 |
| `cleanupShellAliases()` | 从 `.bashrc`/`.zshrc`/`.config/fish/config.fish` 中移除 installer 创建的 alias |
| `removeInstalledSymlink()` | 安全移除 `~/.local/bin/claude` symlink（会跳过 npm-managed 的情况） |
| `updateSymlink()` | 创建/更新 symlink；Windows 下使用复制+重命名策略代替 symlink |

## 具体技术实现

### 1. 平台检测 (`getPlatform`)

```ts
const os = env.platform  // 'macos' | 'linux' | 'windows' | 'wsl'
const arch = process.arch === 'x64' ? 'x64' : process.arch === 'arm64' ? 'arm64' : null
if (os === 'linux' && envDynamic.isMuslEnvironment()) return `linux-${arch}-musl`
return `${os}-${arch}`
```

- 支持 musl 检测，生成 `linux-x64-musl` 等平台标签。
- 不支持的架构会抛出 Error 并记录 debug 日志。

### 2. 目录布局（XDG 规范）

| 用途 | 路径 |
|------|------|
| 版本持久化 | `~/.local/share/claude/versions/{version}` |
| 临时下载 | `~/.cache/claude/staging/{version}` |
| 锁文件 | `~/.local/state/claude/locks/{version}.lock` |
| 用户入口 | `~/.local/bin/claude`（或 Windows 等价路径） |

### 3. 并发控制：双模式锁 (`tryWithVersionLock`)

#### PID-based 锁（默认启用，由 GrowthBook 控制）
- 调用 `pidLock.ts` 的 `withLock()`。
- 锁文件内容为 JSON：`{ pid, version, execPath, acquiredAt }`。
- 通过 `process.kill(pid, 0)` 检测进程是否存活，可立即识别崩溃进程（对比 mtime 的 7~30 天超时）。
- 支持最多 3 次重试，指数退避（100ms → 500ms 或 1s → 5s）。

#### Mtime-based 锁（fallback）
- 使用 `proper-lockfile`（通过 `src/utils/lockfile.ts` 懒加载）。
- `stale` 超时设为 `LOCK_STALE_MS = 7 * 24 * 60 * 60 * 1000`（7 天）。
- `lockCurrentVersion()` 中甚至使用 30 天 stale，以在笔记本休眠后仍视为有效。

### 4. 原子安装 (`atomicMoveToInstallPath`)

为避免跨文件系统移动失败（`EXDEV`），不使用 `rename` 直接从 staging 移到 versions，而是：

1. `copyFile(stagedBinaryPath, tempInstallPath)` 到 installPath 同级目录。
2. `chmod(tempInstallPath, 0o755)`。
3. `rename(tempInstallPath, installPath)` 原子替换。

临时文件命名：`${installPath}.tmp.${process.pid}.${Date.now()}`

### 5. 安装流程分支 (`installVersion`)

根据 `download.ts` 返回的 `downloadType`（`'npm' | 'binary'`）：

- **npm**：从 `staging/node_modules/@anthropic-ai/${nativePackage}/cli` 提取二进制。
- **binary**：直接从 `staging/${binaryName}` 提取。

两者最终都调用 `atomicMoveToInstallPath`。

### 6. 核心更新决策 (`updateLatest`)

```ts
let version = await getLatestVersion(channelOrVersion)

// 1. maxVersion 服务端限流
if (!forceReinstall && maxVersion && gt(version, maxVersion)) {
    if (gte(MACRO.VERSION, maxVersion)) return { success: true, latestVersion: version }
    version = maxVersion
}

// 2. minimumVersion 用户设置（防止降级到 stable 时版本过低）
if (!forceReinstall && shouldSkipVersion(version)) return { success: true, latestVersion: version }

// 3. 早期退出：当前版本已安装且 executable 有效
if (!forceReinstall && version === MACRO.VERSION && versionIsAvailable() && isPossibleClaudeBinary(executablePath))
    return early success

// 4. 执行安装
if (ENABLE_LOCKLESS_UPDATES) {
    wasNewInstall = await performVersionUpdate(version, forceReinstall)
} else {
    // 获取版本锁后执行
}
```

### 7. Singleflight 防重下载 (`installLatest`)

```ts
let inFlightInstall: Promise<InstallLatestResult> | null = null

export function installLatest(channelOrVersion: string, forceReinstall = false): Promise<InstallLatestResult> {
  if (forceReinstall) return installLatestImpl(...)
  if (inFlightInstall) return inFlightInstall
  const promise = installLatestImpl(...)
  inFlightInstall = promise
  void promise.then(clear, clear)
  return promise
}
```

这段代码直接修复了 issue #22413：由于 `NativeAutoUpdater` 组件在 PromptInput 的 suggestion overlay 切换时会反复 remount，每次 remount 都会发起新的 271MB 二进制下载，导致内存暴涨。singleflight 保证同一进程内只会有一个下载在飞。

### 8. Windows 特殊处理 (`updateSymlink`)

Windows 不支持可靠的 symlink 到可执行文件（权限/文件锁定问题），因此：
- **不创建 symlink，而是复制 `.exe`**。
- 更新时采用**重命名策略**：将旧文件重命名为 `claude.exe.old.${timestamp}`，复制新文件，成功后尝试删除旧文件（如果仍在运行则忽略，由后续 `cleanupOldVersions` 清理）。
- 回滚机制：如果复制失败，尝试将旧文件重命名回 `claude.exe`。

### 9. 版本清理 (`cleanupOldVersions`)

启动时异步执行，包含多个子任务：

1. **Windows 旧 exe 清理**：删除 `claude.exe.old.\d+` 文件（可能因运行中而失败，忽略错误）。
2. **Staging 清理**：删除 mtime 超过 1 小时的孤儿 staging 目录。
3. **Stale lock 清理**：若 PID-based 锁定启用，清理已失效的锁文件。
4. **版本二进制清理**：
   - 遍历 `~/.local/share/claude/versions/`。
   - 保护当前运行版本、当前 symlink 指向的版本、以及任何持有 active lock 的版本。
   - 对未受保护的版本按 mtime 降序排列，保留最新的 `VERSION_RETENTION_COUNT = 2` 个，其余逐个获取版本锁后删除。

### 10. 安装健康检查 (`checkInstall`)

返回 `SetupMessage[]`，检查项包括：
- `~/.local/bin` 目录是否存在。
- `claude` 可执行文件是否存在且有效（Windows 直接检查文件；Unix 检查 symlink 指向的目标）。
- `~/.local/bin` 是否在 `PATH` 中（支持 Windows 大小写不敏感比较）。
- 若不在 PATH，返回带平台特定指导的 `SetupMessage`（Windows 环境变量 / Unix shell config）。

### 11. npm 清理 (`cleanupNpmInstallations`)

尝试 `npm uninstall -g` 以下包：
- `@anthropic-ai/claude-code`
- `MACRO.PACKAGE_URL`（如果不同）

若遇到 `ENOTEMPTY` 错误，回退到 `manualRemoveNpmPackage()`：
- 获取 `npm config get prefix`。
- 仅删除 bin 入口（symlink 或 Windows 的 `.cmd`/`.ps1`/无扩展名脚本），**不删除 `node_modules`**，并返回 warning 提示用户手动清理。

同时删除 `~/.claude/local` 本地安装目录。

## 关键代码路径与文件引用

| 路径 | 角色 |
|------|------|
| `src/utils/nativeInstaller/installer.ts` | 本文件，核心实现 |
| `src/utils/nativeInstaller/download.ts` | 被 `performVersionUpdate` 调用，负责下载 |
| `src/utils/nativeInstaller/pidLock.ts` | 被 `tryWithVersionLock`、`lockCurrentVersion`、`cleanupOldVersions` 调用 |
| `src/utils/xdg.ts` | 提供 `getXDGDataHome`、`getXDGCacheHome`、`getXDGStateHome`、`getUserBinDir` |
| `src/utils/lockfile.ts` | 懒加载 `proper-lockfile`，用于 mtime-based 锁 fallback |
| `src/utils/shellConfig.ts` | 提供 `getShellConfigPaths`、`filterClaudeAliases`、`readFileLines`、`writeFileLines` |
| `src/utils/autoUpdater.ts` | 提供 `getMaxVersion`、`shouldSkipVersion` |
| `src/utils/fsOperations.ts` | 提供 `getFsImplementation` |
| `src/utils/doctorDiagnostic.ts` | `checkInstall` 调用 `getCurrentInstallationType` |
| `src/services/analytics/index.ts` | 大量 `logEvent` 埋点 |

## 依赖与外部交互

### 外部命令

- `npm`：卸载旧 npm 安装时使用。
- 无直接网络请求：所有下载逻辑委托给 `download.ts`。

### 环境变量

| 变量 | 作用 |
|------|------|
| `DISABLE_INSTALLATION_CHECKS` | 为 true 时 `checkInstall` 直接返回空数组 |
| `ENABLE_LOCKLESS_UPDATES` | 为 true 时跳过版本锁，依赖原子操作 |
| `ENABLE_PID_BASED_VERSION_LOCKING` | 显式控制 PID-based 锁开关 |

### NPM 包

- `proper-lockfile`（通过 `lockfile.ts` 懒加载）
- Node.js 内置 `fs/promises`、`path`、`os`

## 风险、边界与改进建议

### 风险

1. **文件过大（1700+ 行）**：`installer.ts` 承担了安装、清理、锁、symlink、npm 卸载、健康检查等过多职责，已经是一个"上帝文件"。修改任何子功能都需要理解整个文件，回归测试成本高。
2. **Windows 复制的原子性陷阱**：虽然使用了重命名策略，但在复制新文件的过程中如果进程崩溃，用户可能既没有旧文件也没有新文件。虽然回滚逻辑存在，但回滚本身也可能失败。
3. **锁机制的双轨复杂性**：同时维护 PID-based 和 mtime-based 两套锁，增加了理解和测试负担。 GrowthBook 的渐进式 rollout 意味着部分用户跑 A 套逻辑，部分跑 B 套，问题排查困难。
4. **版本清理的竞态条件**：`cleanupOldVersions` 在遍历 versions 目录时，可能与其他进程的 `performVersionUpdate` 并发。虽然逐个删除前会获取版本锁，但 `readdir` 和 `stat` 之间存在 TOCTOU 窗口（注释中已承认）。
5. **npm 卸载的残留**：`manualRemoveNpmPackage` 故意不删除 `node_modules`，这会导致磁盘空间残留，用户可能不理解为什么还需要手动清理。

### 边界

- **保留版本数硬编码**：`VERSION_RETENTION_COUNT = 2`，无法通过配置调整。
- **staging 清理阈值硬编码**：1 小时，无法配置。
- **Windows 不支持 symlink**：这是设计选择，但意味着 Windows 用户每次更新都要复制完整二进制，磁盘 I/O 更高。
- **musl 检测依赖 `envDynamic.isMuslEnvironment()`**：如果该函数实现有偏差，可能导致向 glibc 系统分发 musl 二进制或反之。

### 改进建议

1. **按功能拆分文件**：将 `installer.ts` 拆分为 `install-core.ts`、`cleanup.ts`、`symlink-manager.ts`、`health-check.ts`、`npm-migration.ts` 等子模块，由 `index.ts` 统一导出。这是最高优先级的可维护性改进。
2. **统一锁抽象层**：将 PID-based 和 mtime-based 锁封装到一个统一的 `VersionLock` 类/模块中，对外隐藏实现细节，减少 `installer.ts` 中的条件分支。
3. **Windows 更新的更安全原子性**：考虑使用 Windows 的 transactional NTFS 或至少引入一个"last known good"标记文件，确保在更新中断时可以自动回滚到上一个可用版本。
4. **可配置的保留策略**：将 `VERSION_RETENTION_COUNT` 和 staging 清理时间暴露为环境变量或配置项，方便高级用户和 CI 环境调整。
5. **减少 `installer.ts` 中的 telemetry 噪音**：文件中几乎每个分支都有 `logEvent`，虽然对监控有帮助，但也增加了代码噪音。可以考虑用一个小型装饰器/包装函数统一埋点。
6. **引入集成测试**：当前没有搜索到针对 nativeInstaller 的测试文件。该模块涉及大量文件系统操作和并发控制，非常适合用临时目录和 mock fs 做集成测试。
