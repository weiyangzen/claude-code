# pluginAutoupdate.ts 研究文档

## 场景与职责

`src/utils/plugins/pluginAutoupdate.ts` 负责 Claude Code 启动时的**后台插件自动更新**。其核心职责包括：

1. **后台自动刷新市场（marketplace）**：在启动时，对开启了 `autoUpdate` 的市场执行 `git pull` 或重新下载，获取最新的 marketplace 元数据。
2. **后台自动更新已安装插件**：基于刷新后的 marketplace，检查当前项目相关的已安装插件是否有新版本，若有则下载/缓存到新的版本目录。
3. **非原地更新（non-inplace）**：所有更新只写磁盘，**不修改当前运行会话的内存状态**。用户必须执行 `/reload-plugins` 或重启进程才能使新版本生效。
4. **通知 UI**：通过回调机制向 REPL 的 Notification 系统报告有哪些插件已自动更新，提示用户重启。

该模块是插件生态“静默自愈”能力的关键一环，尤其对官方 Anthropic marketplace 默认开启自动更新。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `autoUpdateMarketplacesAndPluginsInBackground()` | 启动入口。检查全局开关 `shouldSkipPluginAutoupdate()`，获取 autoUpdate 开启的市场列表，依次刷新市场，然后更新对应插件。 |
| `getAutoUpdateEnabledMarketplaces()` | 计算哪些市场应当被自动更新。优先读取用户在 settings 中显式声明的 `autoUpdate` 值；若未声明，则官方市场默认 `true`，第三方市场默认 `false`。 |
| `updatePluginsForMarketplaces(marketplaceNames)` | 对指定市场集合中的已安装插件执行批量更新。被 `ManageMarketplaces.tsx` 的用户手动“Update marketplace”路径复用，解决旧版本只刷新市场不更新 `installed_plugins.json` 导致的孤儿缓存问题（#29512）。 |
| `updatePlugin(pluginId, installations)` | 对单个插件的某个 scope 安装执行 `updatePluginOp`，捕获成功/失败/已最新三种结果。 |
| `onPluginsAutoUpdated(callback)` / `getAutoUpdatedPluginNames()` | 为 REPL 提供注册/查询自动更新通知的能力，处理“更新完成在 callback 注册之前”的竞态条件。 |

---

## 具体技术实现

### 关键流程

#### 1. 启动时后台自动更新流程

```
main.tsx / startDeferredPrefetches
  └─> backgroundHousekeeping.ts::startBackgroundHousekeeping()
        └─> autoUpdateMarketplacesAndPluginsInBackground()
              ├─> shouldSkipPluginAutoupdate() ? 提前返回
              ├─> getAutoUpdateEnabledMarketplaces()
              │     ├─> loadKnownMarketplacesConfig() 读取 known_marketplaces.json
              │     ├─> getDeclaredMarketplaces() 读取 settings 中的显式声明
              │     └─> 合并决策：settings 显式值 > isMarketplaceAutoUpdate() 默认值
              ├─> Promise.allSettled(refreshMarketplace(name, ..., {disableCredentialHelper: true}))
              ├─> updatePlugins(autoUpdateEnabledMarketplaces)
              │     └─> updatePluginsForMarketplaces(set)
              │           ├─> loadInstalledPluginsFromDisk() 读取 V2 installed_plugins.json
              │           ├─> 过滤：插件 ID 后缀匹配市场名
              │           ├─> 过滤：isInstallationRelevantToCurrentProject()
              │           └─> Promise.allSettled(updatePluginOp(pluginId, scope))
              └─> 若有更新插件
                    ├─> callback 已注册 => 立即调用 pluginUpdateCallback(updatedPlugins)
                    └─> callback 未注册 => pendingNotification = updatedPlugins（下次注册时立即触发）
```

#### 2. 非原地更新机制

- `updatePluginOp`（定义在 `pluginOperations.ts`）会下载/计算新版本，将插件缓存到新的版本目录（`cache/{marketplace}/{plugin}/{version}/`），然后调用 `updateInstallationPathOnDisk()` 只修改磁盘上的 `installed_plugins.json`。
- **内存中的 `inMemoryInstalledPlugins` 不被修改**。当前会话继续使用旧版本，直到用户执行 `/reload-plugins` 重新加载插件。
- `hasPendingUpdates()` / `getPendingUpdatesDetails()`（`installedPluginsManager.ts`）通过对比磁盘与内存状态判断是否有待生效的更新。

### 数据结构

- `PluginAutoUpdateCallback = (updatedPlugins: string[]) => void`：通知回调类型。
- 模块级变量：
  - `pluginUpdateCallback: PluginAutoUpdateCallback | null`：当前注册的回调。
  - `pendingNotification: string[] | null`：用于处理“更新发生在 REPL mount 之前”的竞态缓存。

### 协议/命令

- 市场刷新使用 `refreshMarketplace()`，底层根据市场 source 类型可能执行 `git pull`、HTTP 下载或 npm 安装。
- 自动更新时显式传入 `disableCredentialHelper: true`，避免在后台静默更新时弹出凭证助手交互窗口。
- 插件更新调用 `updatePluginOp()`，内部可能触发 `git clone`、`cachePlugin`、版本计算、ZIP 缓存转换等操作。

---

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 用途 |
|------|------|
| `autoUpdateMarketplacesAndPluginsInBackground()` | 启动后台自动更新（主入口） |
| `updatePluginsForMarketplaces(Set<string>)` | 为指定市场集合更新已安装插件（被 UI 手动更新复用） |
| `onPluginsAutoUpdated(callback)` | 注册自动更新完成通知回调 |
| `getAutoUpdatedPluginNames()` | 获取当前有待更新的插件名称列表 |
| `PluginAutoUpdateCallback` | 回调类型定义 |

### 直接依赖文件

- `src/services/plugins/pluginOperations.ts`：`updatePluginOp`
- `src/utils/plugins/installedPluginsManager.ts`：`getPendingUpdatesDetails`, `hasPendingUpdates`, `isInstallationRelevantToCurrentProject`, `loadInstalledPluginsFromDisk`
- `src/utils/plugins/marketplaceManager.ts`：`getDeclaredMarketplaces`, `loadKnownMarketplacesConfig`, `refreshMarketplace`
- `src/utils/plugins/pluginIdentifier.ts`：`parsePluginIdentifier`
- `src/utils/plugins/schemas.ts`：`isMarketplaceAutoUpdate`, `PluginScope`
- `src/utils/config.ts`：`shouldSkipPluginAutoupdate`
- `src/utils/debug.ts`：`logForDebugging`
- `src/utils/errors.ts`：`errorMessage`
- `src/utils/log.ts`：`logError`

### 调用方文件

- `src/utils/backgroundHousekeeping.ts`：在 `startBackgroundHousekeeping()` 中调用 `autoUpdateMarketplacesAndPluginsInBackground()`，作为启动后台任务之一。
- `src/hooks/notifs/usePluginAutoupdateNotification.tsx`：注册 `onPluginsAutoUpdated`，在收到更新后向用户展示“Run /reload-plugins to apply”通知。
- `src/commands/plugin/ManageMarketplaces.tsx`：在用户手动执行“Update marketplace”后，调用 `updatePluginsForMarketplaces()` 确保 `installed_plugins.json` 同步到新版本，避免孤儿缓存被 GC 误删（#29512）。

---

## 依赖与外部交互

### 外部系统/配置

- **`known_marketplaces.json`**（`~/.claude/plugins/known_marketplaces.json`）：存储市场的 source、lastUpdated、autoUpdate 等状态。
- **`installed_plugins.json`**（`~/.claude/plugins/installed_plugins.json`）：V2 格式，记录每个插件在每个 scope 下的安装路径和版本。自动更新只修改磁盘上的该文件。
- **Git / HTTP / NPM**：`refreshMarketplace` 和 `updatePluginOp` 底层可能执行网络请求或 git 操作。
- **Settings 层**：`getDeclaredMarketplaces()` 读取用户 settings（`extraKnownMarketplaces`）以决定 autoUpdate 显式覆盖值。

### 与 UI 的交互

- 通过 `onPluginsAutoUpdated` 回调与 `usePluginAutoupdateNotification` 解耦。更新完成后，UI 展示带插件名称列表的通知，并提示 `/reload-plugins`。

---

## 风险、边界与改进建议

### 风险与边界

1. **竞态条件：REPL 未挂载时更新已完成**
   - 本模块通过 `pendingNotification` 变量缓存结果，待 `onPluginsAutoUpdated` 注册后立即补发。但若进程在注册前退出，则通知丢失（用户下次启动才会再次检测到有 pending updates）。

2. **后台更新失败静默吞掉**
   - `refreshMarketplace` 和 `updatePlugin` 的异常均被 `try/catch` 捕获并降级为 debug log，用户不会收到“某个市场刷新失败”或“某个插件更新失败”的主动通知，除非去查看 debug 日志。

3. **第三方市场默认不自动更新**
   - 官方市场默认 `autoUpdate: true`，第三方市场默认 `false`。若用户希望第三方市场也自动更新，必须手动在 UI 中开启，存在 discoverability 问题。

4. **非原地更新的用户体验**
   - 更新后必须 `/reload-plugins` 才能生效。如果用户长时间不执行，磁盘与内存状态持续分叉，可能导致“我明明更新了为什么还是旧版本”的困惑。

5. **项目相关性过滤**
   - `isInstallationRelevantToCurrentProject` 只把 `user`/`managed` 和匹配当前 `projectPath` 的 `project`/`local` scope 视为相关。如果用户在 A 项目启动 Claude，B 项目的 project-scope 插件不会被自动更新，这是符合设计的，但跨项目使用时可能产生版本不一致。

### 改进建议

1. **持久化 pendingNotification**
   - 将 pending 的自动更新通知写入一个小型状态文件（如 `~/.claude/plugins/.pending-autoupdate`），进程重启后仍可恢复通知，避免完全依赖内存。

2. **失败聚合报告**
   - 在 `autoUpdateMarketplacesAndPluginsInBackground` 返回时，将刷新/更新失败的市场/插件列表通过通知系统一次性报告给用户（低优先级 warning 通知），而不是完全静默。

3. **自动更新策略可配置化**
   - 考虑在 settings 中增加全局 `pluginAutoUpdate` 策略（如 `all`, `official-only`, `none`），减少用户为每个第三方市场单独开关的操作成本。

4. **与 `/reload-plugins` 的联动**
   - 可在检测到 pending updates 时，在 REPL 的顶部或状态栏增加一个常驻提示（而不仅是 10 秒超时的通知），提高用户执行 `/reload-plugins` 的概率。
