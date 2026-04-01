# src/utils/backgroundHousekeeping.ts 深入研究

## 场景与职责

`backgroundHousekeeping.ts` 是 Claude Code 启动后的**后台任务调度器**。它在 REPL 会话启动后延迟触发一系列“非常慢”的维护操作，并在长会话中按固定周期（24 小时）执行 recurring cleanup。

核心职责：
- 启动低优先级后台初始化（MagicDocs、Skill Improvement、AutoDream、插件自动更新、Deep Link 协议注册）。
- 延迟执行一次性清理（旧消息文件、旧版本二进制）。
- 为 ant 用户安排周期性清理（npm 缓存、旧版本节流清理）。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `startBackgroundHousekeeping()` | 集中调度所有后台任务，避免在 `main.tsx` 中散落 `setTimeout` |
| `initMagicDocs()` | 后台初始化 MagicDocs 服务 |
| `initSkillImprovement()` | 后台初始化技能改进模块 |
| `initAutoDream()` | 启动 AutoDream 特性 |
| `autoUpdateMarketplacesAndPluginsInBackground()` | 静默更新 marketplace 与插件（磁盘非原地） |
| `ensureDeepLinkProtocolRegistered()` | 在交互式会话中注册 deep link 协议（Lodestone 特性） |
| `runVerySlowOps()` | 延迟 10 分钟后执行一次性清理，若用户最近 1 分钟有交互则再次延后 |
| `cleanupNpmCacheForAnthropicPackages()` | ant 专属：清理 anthropic 包的 npm 缓存 |
| `cleanupOldVersionsThrottled()` | ant 专属：按标记文件/锁节流清理旧版本 |

## 具体技术实现

### 启动时立即执行的任务
```ts
void initMagicDocs()
void initSkillImprovement()
if (feature('EXTRACT_MEMORIES')) extractMemoriesModule!.initExtractMemories()
initAutoDream()
void autoUpdateMarketplacesAndPluginsInBackground()
if (feature('LODESTONE') && getIsInteractive()) {
  void registerProtocolModule!.ensureDeepLinkProtocolRegistered()
}
```
- 使用 `void` 显式忽略 Promise，防止未处理 Promise rejection 警告（实际错误由各模块内部捕获）。
- `feature('...')` 来自 `bun:bundle`，在编译期做死代码消除。

### 延迟慢任务（`runVerySlowOps`）
- 延迟：10 分钟（`DELAY_VERY_SLOW_OPERATIONS_THAT_HAPPEN_EVERY_SESSION = 10 * 60 * 1000`）
- 交互感知保护：若 `getLastInteractionTime() > Date.now() - 60_000`，则重新 `setTimeout` 延后 10 分钟。
- 执行顺序：
  1. `cleanupOldMessageFilesInBackground()`
  2. 再次检查交互感知
  3. `cleanupOldVersions()`

### 周期性清理（ant only）
- 周期：24 小时（`RECURRING_CLEANUP_INTERVAL_MS = 24 * 60 * 60 * 1000`）
- 使用 `setInterval(...).unref()`，避免阻止进程自然退出。
- 清理函数内部自带 marker file + lockfile 节流，保证即使多进程也不会频繁执行。

## 关键代码路径与文件引用

```
src/main.tsx:2818
  └── import('./utils/backgroundHousekeeping.js').then(m => m.startBackgroundHousekeeping())

src/screens/REPL.tsx:74
  └── startBackgroundHousekeeping()

src/utils/cleanup.ts
  └── cleanupOldMessageFilesInBackground(), cleanupNpmCacheForAnthropicPackages(), cleanupOldVersionsThrottled()

src/utils/nativeInstaller/index.ts
  └── cleanupOldVersions()

src/utils/plugins/pluginAutoupdate.ts
  └── autoUpdateMarketplacesAndPluginsInBackground()

src/services/autoDream/autoDream.js
  └── initAutoDream()

src/utils/hooks/skillImprovement.js
  └── initSkillImprovement()

src/utils/deepLink/registerProtocol.js
  └── ensureDeepLinkProtocolRegistered()
```

### 依赖模块
- `bun:bundle` 的 `feature()` — 编译期特性开关
- `src/bootstrap/state.js` — `getIsInteractive`, `getLastInteractionTime`
- `src/utils/cleanup.js` — 各类清理实现
- `src/utils/nativeInstaller/index.js` — 旧版本二进制清理
- `src/utils/plugins/pluginAutoupdate.js` — 插件自动更新

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| 文件系统 | `cleanup.ts` / `nativeInstaller` | 删除过期消息、错误日志、旧版本二进制 |
| npm 缓存 | `cleanupNpmCacheForAnthropicPackages()` | 调用 npm cache 命令清理 |
| 用户交互时间戳 | `getLastInteractionTime()` | 避免在用户活跃时执行慢操作 |
| Bun 编译特性 | `feature('EXTRACT_MEMORIES')`, `feature('LODESTONE')` | 编译期条件加载，减少包体积 |

## 风险、边界与改进建议

### 风险
1. **`void` 忽略 Promise**：若 `initMagicDocs` 等模块抛出未捕获异常，可能成为未处理 rejection，虽然 `void` 在语法上合法，但运行时仍可能触发 `unhandledRejection`。
2. **`runVerySlowOps` 无限推迟**：若用户持续交互（如长时间 REPL 会话），10 分钟延迟可能永远达不到，导致一次性清理始终不执行。
3. **ant-only 周期任务硬编码**：`process.env.USER_TYPE === 'ant'` 在运行时判断，若未来想为 external 用户开启类似机制，需要修改此处。
4. **`require()` 与 ESM 混用**：`feature('EXTRACT_MEMORIES')` 分支使用 `require`，在纯 ESM 环境下可能受限（但 Bun 支持）。

### 边界
- 所有后台任务均标记 `.unref()`，不会阻止 CLI 在空闲时退出。
- `cleanupOldVersions()` 在 `runVerySlowOps` 中执行，而 `cleanupOldVersionsThrottled()` 在 24h 周期中执行，两者职责有重叠但触发条件不同。

### 改进建议
1. **Promise 错误兜底**：将 `void` 改为 `.catch(logError)` 或统一 `Promise.allSettled` 包裹，确保异常可观测。
2. **慢任务执行上限**：为 `runVerySlowOps` 增加最大推迟次数（如最多推迟 6 次/1 小时），保证清理最终一定会执行。
3. **配置化周期清理**：将 `USER_TYPE === 'ant'` 改为基于 GrowthBook 配置或设置项，便于灰度开放给 external 用户。
4. **任务健康检查**：增加一个后台任务执行日志（debug 级别），记录每次任务开始/结束时间，便于排查“为什么没执行清理”。
