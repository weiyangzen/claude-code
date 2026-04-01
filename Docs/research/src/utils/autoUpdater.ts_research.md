# src/utils/autoUpdater.ts 深入研究

## 场景与职责

`autoUpdater.ts` 是 Claude Code CLI 的自动更新核心引擎，负责：
- **版本合规检查**：在启动时断言当前版本不低于服务端配置的最小版本（`assertMinVersion`），否则强制退出并提示用户更新。
- **版本上限控制**：读取 GrowthBook 动态配置 `tengu_max_version_config`，为不同用户类型（ant / external）提供最大允许版本，作为服务端 kill switch。
- **全局包安装与更新**：通过 `npm` 或 `bun` 执行全局安装，支持稳定版（stable）与最新版（latest）双通道。
- **原生安装辅助**：为原生安装器提供版本查询（`getLatestVersion`、`getLatestVersionFromGcs`）、跳过逻辑（`shouldSkipVersion`）及权限检查（`checkGlobalInstallPermissions`）。
- **Shell 配置清理**：更新前自动移除旧版 `claude` alias，防止本地安装与全局安装冲突。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `assertMinVersion` | 防止过旧客户端继续运行，确保功能与安全策略同步 |
| `getMaxVersion` / `getMaxVersionMessage` | 服务端 incidents 期间暂停自动更新，并展示说明横幅 |
| `shouldSkipVersion` | 用户设置 `minimumVersion` 后，避免降级到更低版本 |
| `acquireLock` / `releaseLock` | 文件锁（`~/.claude/.update.lock`）防止并发更新导致文件损坏 |
| `checkGlobalInstallPermissions` | 检查 npm/bun 全局前缀是否可写，提前发现权限不足 |
| `getLatestVersion` / `getNpmDistTags` | 从 npm registry 查询可用版本，供 doctor / UI 展示 |
| `getLatestVersionFromGcs` / `getGcsDistTags` | 为无 npm 环境（包管理器安装）提供 GCS 版本指针 |
| `getVersionHistory` | ant 用户回滚时列出带原生二进制包的版本历史 |
| `installGlobalPackage` | 实际执行 `npm install -g` 或 `bun install -g`，并记录安装方式 |
| `removeClaudeAliasesFromShellConfigs` | 迁移期清理 shell alias，减少多安装方式冲突 |

## 具体技术实现

### 锁机制（TOCTOU 防护）
- 锁文件路径：`join(getClaudeConfigHomeDir(), '.update.lock')`
- 超时：5 分钟（`LOCK_TIMEOUT_MS = 5 * 60 * 1000`）
- 实现细节：
  1. `stat` 检查锁文件；若存在且未超时，返回 `false`。
  2. 若发现 stale lock，**立即二次 `stat` 复核**（防止进程 A 刚创建新锁就被进程 B 删除）。
  3. 使用 `writeFile(flag: 'wx')` 原子创建锁文件；若目录不存在则 `mkdir` 后重试。
  4. 锁内容写入当前 `process.pid`，释放时校验 PID 一致性再 `unlink`。

### 版本查询双源策略
- **npm 源**：`npm view <PACKAGE_URL>@<tag> version --prefer-online`，超时 5s，始终从 `homedir()` 运行以规避恶意项目级 `.npmrc` 重定向攻击。
- **GCS 源**：`axios.get(${GCS_BUCKET_URL}/${channel})`，用于原生安装或 npm 不可用场景。

### WSL / Windows npm 防护
- `env.isNpmFromWindowsPath()` 检测 WSL 中使用 `/mnt/c/` 下的 Windows npm，直接拒绝更新并给出修复指引。

### SemVer 与 SHA 策略
- 比较最小/最大版本时使用 `semver.lt/gte`，**忽略 build metadata**（+SHA）。
- 更新检测使用**精确字符串比较**，确保 SHA 变更也能触发更新。

## 关键代码路径与文件引用

```
main.tsx
  └── assertMinVersion()              [启动阻塞检查]

src/components/AutoUpdater.tsx
  └── getLatestVersion(), shouldSkipVersion(), installGlobalPackage()

src/components/PackageManagerAutoUpdater.tsx
  └── getLatestVersionFromGcs(), getMaxVersion(), shouldSkipVersion()

src/components/NativeAutoUpdater.tsx
  └── getMaxVersion(), getMaxVersionMessage()

src/utils/nativeInstaller/installer.ts
  └── getMaxVersion(), shouldSkipVersion()  [原生安装器版本决策]

src/utils/doctorDiagnostic.ts
  └── checkGlobalInstallPermissions()       [doctor 权限诊断]

src/screens/Doctor.tsx
  └── getNpmDistTags(), getGcsDistTags()    [doctor 版本信息展示]
```

### 依赖模块
- `src/services/analytics/growthbook.js` — 动态配置（最小版本、最大版本、实验开关）
- `src/utils/config.js` — `ReleaseChannel`、`saveGlobalConfig`
- `src/utils/env.js` / `envUtils.js` — Bun/npm 环境检测
- `src/utils/execFileNoThrow.js` — 安全执行 npm/bun 子进程
- `src/utils/fsOperations.js` — 抽象文件系统（支持测试 mock）
- `src/utils/gracefulShutdown.js` — 版本不合规时同步退出
- `src/utils/semver.js` — SemVer 比较
- `src/utils/settings/settings.js` — 读取 `minimumVersion`
- `src/utils/shellConfig.js` — 读取/过滤/写入 shell 配置行

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| npm registry | `execFileNoThrowWithCwd('npm', ...)` | 版本查询与全局安装 |
| GCS | `axios.get` | `storage.googleapis.com/claude-code-dist-...` |
| GrowthBook (Statsig) | `getDynamicConfig_BLOCKS_ON_INIT` | `tengu_version_config`、`tengu_max_version_config` |
| 文件系统 | `fs/promises` + `getFsImplementation()` | 锁文件、shell 配置修改 |
| 分析系统 | `logEvent` | 锁竞争、Windows npm 异常等事件上报 |

## 风险、边界与改进建议

### 风险
1. **npm registry 单点故障**：若 npm 网络不可达且未配置 GCS fallback，更新/版本检查会失败。
2. **锁超时过短**：5 分钟在慢速网络或大型 CI 环境中可能误杀合法并发进程。
3. **WSL Windows npm 检测硬编码**：依赖路径前缀 `/mnt/c/`，若用户自定义 drvfs 挂载点可能漏检。
4. **全局安装权限检查不完整**：仅检查 prefix 目录可写，未覆盖 npm 的 `prefix` 配置与 `unsafe-perm` 组合场景。

### 边界
- `assertMinVersion` 在 `NODE_ENV === 'test'` 时直接返回，避免测试被网络配置阻塞。
- `getVersionHistory` 仅限 `USER_TYPE === 'ant'`；外部用户返回空数组。
- `installGlobalPackage` 在 Bun 环境下使用 `bun install -g`，但 `getInstallationPrefix` 中 Bun 使用 `bun pm bin -g` 获取前缀，与 npm 逻辑不完全对称。

### 改进建议
1. **统一版本源优先级**：将 GCS 作为 npm 失败后的显式 fallback，而非仅由调用方分别使用。
2. **锁机制持久化**：考虑使用 `proper-lockfile` 或基于 PID 的跨平台库，减少自制锁的 TOCTOU 风险。
3. **Bun 全局安装测试覆盖**：目前 Bun 分支在 `installGlobalPackage` 与 `getInstallationPrefix` 中的行为差异较大，建议增加集成测试。
4. **最小版本阻塞体验**：当前直接 `gracefulShutdownSync(1)`，可考虑提供 `--skip-version-check` 逃生舱（仅限内部测试）。
