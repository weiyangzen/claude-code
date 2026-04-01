# AutoUpdater.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`AutoUpdater` 是 Claude Code CLI 的**自动更新核心组件**，负责检测、下载和安装新版本。它支持 npm 全局安装和本地安装两种更新路径，确保用户始终使用最新版本。

### 1.2 使用场景
- **启动时检查**：会话启动时自动检查更新
- **定期轮询**：每 30 分钟检查一次新版本
- **版本升级**：自动下载并安装新版本
- **安装迁移**：从 npm 全局安装迁移到本地安装

### 1.3 更新流程概览
```
启动/定时触发
    │
    ▼
检查更新 ──→ 发现新版本 ──→ 执行更新
    │                           │
    └── 已是最新 ←──────────────┘
```

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| 版本检查 | 通过 npm registry 获取最新版本 |
| 版本比较 | 使用 semver 比较当前版本和最新版本 |
| 锁机制 | 防止多进程并发更新 |
| 安装类型检测 | 自动检测 npm-global/npm-local/development |
| 更新执行 | 根据安装类型选择合适的更新方法 |
| 状态反馈 | 向用户显示更新进度和结果 |
| 分析追踪 | 记录更新成功/失败事件 |

### 2.2 Props 接口

```typescript
type Props = {
  isUpdating: boolean;                                    // 是否正在更新
  onChangeIsUpdating: (isUpdating: boolean) => void;      // 更新状态变更回调
  onAutoUpdaterResult: (result: AutoUpdaterResult) => void; // 结果回调
  autoUpdaterResult: AutoUpdaterResult | null;            // 上次更新结果
  showSuccessMessage: boolean;                            // 是否显示成功消息
  verbose: boolean;                                       // 详细输出模式
}

type AutoUpdaterResult = {
  version: string | null;
  status: InstallStatus;
  notifications?: string[];
}

type InstallStatus = 'success' | 'no_permissions' | 'install_failed' | 'in_progress'
```

---

## 3. 具体技术实现

### 3.1 更新检查流程

```typescript
const checkForUpdates = React.useCallback(async () => {
  // 1. 防止并发更新
  if (isUpdatingRef.current) return
  
  // 2. 跳过测试/开发环境
  if ("production" === 'test' || "production" === 'development') return
  
  const currentVersion = MACRO.VERSION
  const channel = getInitialSettings()?.autoUpdatesChannel ?? 'latest'
  
  // 3. 获取最新版本
  let latestVersion = await getLatestVersion(channel)
  
  // 4. 检查服务器端版本上限（kill switch）
  const maxVersion = await getMaxVersion()
  if (maxVersion && latestVersion && gt(latestVersion, maxVersion)) {
    if (gte(currentVersion, maxVersion)) {
      // 当前版本已高于上限，跳过更新
      return
    }
    latestVersion = maxVersion
  }
  
  // 5. 检查是否需要更新
  if (!isDisabled && currentVersion && latestVersion && 
      !gte(currentVersion, latestVersion) && 
      !shouldSkipVersion(latestVersion)) {
    
    // 6. 执行更新
    onChangeIsUpdating(true)
    // ... 更新逻辑
  }
}, [onAutoUpdaterResult])
```

### 3.2 安装类型检测与处理

```typescript
const installationType = await getCurrentInstallationType()

switch (installationType) {
  case 'npm-local':
    // 本地安装：使用 npm install 更新 ~/.claude/local
    updateMethod = 'local'
    installStatus = await installOrUpdateClaudePackage(channel)
    break
    
  case 'npm-global':
    // 全局安装：使用 npm install -g
    updateMethod = 'global'
    installStatus = await installGlobalPackage()
    break
    
  case 'native':
    // 原生安装：不应该到达这里（由 NativeAutoUpdater 处理）
    logForDebugging('AutoUpdater: Unexpected native installation')
    return
    
  case 'development':
    // 开发模式：跳过更新
    logForDebugging('AutoUpdater: Cannot auto-update development build')
    return
    
  default:
    // 未知类型：回退到配置检测
    const isMigrated = config.installMethod === 'local'
    updateMethod = isMigrated ? 'local' : 'global'
    installStatus = isMigrated 
      ? await installOrUpdateClaudePackage(channel)
      : await installGlobalPackage()
}
```

### 3.3 版本上限控制（Kill Switch）

```typescript
// 服务器端可配置最大版本，用于紧急暂停更新
export async function getMaxVersion(): Promise<string | undefined> {
  const config = await getMaxVersionConfig()
  if (process.env.USER_TYPE === 'ant') {
    return config.ant || undefined  // 内部用户版本
  }
  return config.external || undefined  // 外部用户版本
}
```

### 3.4 锁机制防止并发

```typescript
// 使用文件锁防止多进程同时更新
export function getLockFilePath(): string {
  return join(getClaudeConfigHomeDir(), '.update.lock')
}

async function acquireLock(): Promise<boolean> {
  // 1. 检查现有锁是否过期（5分钟超时）
  // 2. 使用 O_EXCL 原子创建锁文件
  // 3. 写入当前进程 PID
}

async function releaseLock(): Promise<void> {
  // 验证 PID 匹配后删除锁文件
}
```

### 3.5 分析事件

| 事件 | 触发条件 | 数据 |
|------|----------|------|
| `tengu_auto_updater_success` | 更新成功 | fromVersion, toVersion, durationMs, wasMigrated, installationType |
| `tengu_auto_updater_fail` | 更新失败 | fromVersion, attemptedVersion, status, durationMs, wasMigrated, installationType |
| `tengu_auto_updater_lock_contention` | 锁竞争 | pid, currentVersion |

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置
```
src/components/AutoUpdater.tsx
```

### 4.2 依赖图

```
AutoUpdater.tsx
├── react, usehooks-ts (React 和 hooks)
├── src/services/analytics/index.js
│   └── logEvent
├── ../hooks/useUpdateNotification.js
│   └── useUpdateNotification hook
├── ../ink.js (Box, Text)
├── ../utils/autoUpdater.js
│   ├── getLatestVersion
│   ├── getMaxVersion
│   ├── installGlobalPackage
│   └── shouldSkipVersion
├── ../utils/config.js
│   ├── getGlobalConfig
│   └── isAutoUpdaterDisabled
├── ../utils/debug.js
│   └── logForDebugging
├── ../utils/doctorDiagnostic.js
│   └── getCurrentInstallationType
├── ../utils/localInstaller.js
│   ├── installOrUpdateClaudePackage
│   └── localInstallationExists
├── ../utils/nativeInstaller/index.js
│   └── removeInstalledSymlink
├── ../utils/semver.js
│   ├── gt (大于)
│   └── gte (大于等于)
└── ../utils/settings/settings.js
    └── getInitialSettings
```

### 4.3 相关工具函数

**getLatestVersion** (`src/utils/autoUpdater.ts`):
```typescript
export async function getLatestVersion(channel: ReleaseChannel): Promise<string | null> {
  const npmTag = channel === 'stable' ? 'stable' : 'latest'
  const result = await execFileNoThrowWithCwd(
    'npm',
    ['view', `${MACRO.PACKAGE_URL}@${npmTag}`, 'version`, '--prefer-online'],
    { abortSignal: AbortSignal.timeout(5000), cwd: homedir() }
  )
  return result.code === 0 ? result.stdout.trim() : null
}
```

**installGlobalPackage** (`src/utils/autoUpdater.ts`):
```typescript
export async function installGlobalPackage(specificVersion?: string | null): Promise<InstallStatus> {
  if (!(await acquireLock())) {
    return 'in_progress'
  }
  
  try {
    // 检查 Windows WSL 中的 Windows npm
    if (!env.isRunningWithBun() && env.isNpmFromWindowsPath()) {
      return 'install_failed'
    }
    
    // 检查全局安装权限
    const { hasPermissions } = await checkGlobalInstallPermissions()
    if (!hasPermissions) {
      return 'no_permissions'
    }
    
    // 执行安装
    const packageManager = env.isRunningWithBun() ? 'bun' : 'npm'
    const installResult = await execFileNoThrowWithCwd(
      packageManager,
      ['install', '-g', packageSpec],
      { cwd: homedir() }
    )
    
    return installResult.code === 0 ? 'success' : 'install_failed'
  } finally {
    await releaseLock()
  }
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 类别 | 依赖 | 用途 |
|------|------|------|
| React | react, usehooks-ts | 组件和定时器 hook |
| 分析 | logEvent | 更新事件追踪 |
| UI | ink (Box, Text) | 终端 UI 渲染 |
| 更新逻辑 | autoUpdater.js | 版本检查和安装 |
| 配置 | config.js | 全局配置读取 |
| 诊断 | doctorDiagnostic.js | 安装类型检测 |
| 本地安装 | localInstaller.js | 本地 npm 安装 |
| 原生安装 | nativeInstaller | 符号链接管理 |
| 版本比较 | semver.js | semver 比较 |
| 设置 | settings.js | 更新通道配置 |

### 5.2 数据流

```
AutoUpdater 组件
    │
    ├── useInterval(30min) ──→ checkForUpdates()
    │
    ├── getLatestVersion() ──→ npm registry
    │
    ├── getCurrentInstallationType() ──→ 文件系统检测
    │
    ├── installGlobalPackage() / installOrUpdateClaudePackage()
    │       │
    │       ├── acquireLock() ──→ ~/.claude/.update.lock
    │       ├── npm install ──→ npm registry
    │       └── releaseLock()
    │
    ├── logEvent() ──→ 分析服务
    │
    └── onAutoUpdaterResult() ──→ 父组件
```

### 5.3 配置项

```typescript
// ~/.claude.json
{
  "autoUpdates": true,              // 是否启用自动更新
  "autoUpdatesProtectedForNative": false,  // 原生安装保护标志
  "installMethod": "global" | "local" | "native" | "unknown"
}

// ~/.claude/settings.json
{
  "autoUpdatesChannel": "latest" | "stable",
  "minimumVersion": "x.x.x"  // 最低要求版本（用于降级保护）
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 严重程度 |
|------|------|----------|
| 更新中断 | 更新过程中进程退出可能导致损坏 | 中 |
| 权限不足 | 全局安装需要写权限 | 中 |
| 网络失败 | npm registry 不可达 | 中 |
| 版本冲突 | 多版本安装并存 | 中 |
| 锁过期 | 5分钟锁超时可能不够 | 低 |
| WSL 问题 | Windows npm 在 WSL 中不兼容 | 中 |

### 6.2 边界情况

1. **并发更新**：多进程同时尝试更新时的锁竞争
2. **部分更新**：更新过程中断后的恢复
3. **回滚需求**：新版本有 bug 时的降级机制
4. **开发环境**：NODE_ENV=development 时跳过更新
5. **离线环境**：无网络连接时的优雅降级

### 6.3 改进建议

1. **可靠性增强**：
   ```typescript
   // 添加更新前备份
   async function backupCurrentInstallation(): Promise<void> {
     const backupDir = join(getClaudeConfigHomeDir(), 'backups', Date.now().toString())
     await copyDir(getInstallDir(), backupDir)
   }
   
   // 添加更新失败回滚
   async function rollbackToBackup(): Promise<void> {
     // 恢复备份
   }
   ```

2. **用户体验**：
   - 显示更新进度百分比
   - 提供 "稍后提醒" 选项
   - 支持手动触发更新检查
   - 添加更新日志预览

3. **安全性**：
   - 验证下载包的签名/校验和
   - 添加更新来源白名单
   - 企业环境下支持私有 registry

4. **可观察性**：
   ```typescript
   // 添加更详细的分析
   logEvent('tengu_auto_updater_version_check', {
     currentVersion,
     latestVersion,
     channel,
     timeToCheckMs
   })
   ```

5. **配置管理**：
   - 支持按项目禁用自动更新
   - 添加更新窗口配置（仅特定时间段更新）
   - 支持 Beta/Canary 通道

6. **架构优化**：
   - 将更新逻辑提取到独立服务
   - 支持后台静默更新
   - 添加更新队列（多个更新时顺序执行）

### 6.4 相关组件

- `AutoUpdaterWrapper.tsx`: 根据安装类型选择更新器
- `NativeAutoUpdater.tsx`: 原生安装更新器
- `PackageManagerAutoUpdater.tsx`: 包管理器安装更新器
