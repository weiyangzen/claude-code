# pluginInstallationHelpers.ts 研究文档

## 场景与职责

`src/utils/plugins/pluginInstallationHelpers.ts` 是 Claude Code 插件安装流程的**核心共享 helpers 集合**。它封装了从 marketplace 条目到本地磁盘缓存、再到 `installed_plugins.json` 注册的完整安装链路，供 CLI、交互式 UI、LSP 推荐、Hint 推荐等多种入口复用。

主要职责包括：

1. **统一缓存与注册**：将远程或本地插件 source 缓存到版本化目录，并写入 `installed_plugins.json`。
2. **依赖闭包解析与安装**：在安装根插件的同时，递归解析并安装其依赖插件。
3. **策略拦截**：在安装前检查组织策略（policy）是否禁止该插件或其依赖。
4. **路径安全校验**：防止本地 source 的相对路径逃逸出 marketplace 目录（路径遍历攻击防护）。
5. **UI/CLI 包装**：提供 `installPluginFromMarketplace`，为交互式 UI 添加 analytics、错误格式化、成功消息等包装。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `validatePathWithinBase(basePath, relativePath)` | 校验解析后的绝对路径是否仍在 `basePath` 内，防止 `../../../etc/passwd` 等路径遍历攻击。 |
| `cacheAndRegisterPlugin(pluginId, entry, scope, projectPath?, localSourcePath?)` | 核心缓存+注册函数。调用 `cachePlugin` 下载/复制插件，计算版本，移动到版本化路径，处理 ZIP 缓存，最后写入 `installed_plugins.json`。 |
| `registerPluginInstallation(info, scope, projectPath?)` | 对无需缓存的本地插件（如已存在于磁盘）直接注册到 `installed_plugins.json`。 |
| `installResolvedPlugin({pluginId, entry, scope, marketplaceInstallLocation})` | **安装核心**。处理本地 source guard、依赖闭包解析、策略检查、settings 批量写入、逐个缓存注册。返回结构化的 `InstallCoreResult`。 |
| `formatResolutionError(r)` | 将依赖解析失败结果转换为人类可读的错误消息（cycle、cross-marketplace、not-found）。 |
| `installPluginFromMarketplace({pluginId, entry, marketplaceName, scope, trigger})` | 交互式 UI 包装器。在 `installResolvedPlugin` 基础上增加 try/catch、analytics 事件、UI 风格消息。 |
| `parsePluginId(pluginId)` | 将 `name@marketplace` 解析为 `{name, marketplace}`，格式非法时返回 `null`。 |
| `getCurrentTimestamp()` | 返回 ISO 8601 当前时间，用于 `installedAt` / `lastUpdated`。 |

---

## 具体技术实现

### 关键流程

#### 1. `cacheAndRegisterPlugin` 完整缓存流程

```
cacheAndRegisterPlugin(pluginId, entry, scope, projectPath, localSourcePath)
  ├─> 确定 source：若 entry.source 是字符串且 localSourcePath 存在，则使用 localSourcePath
  ├─> await cachePlugin(source, {manifest: entry})
  │     └─> 返回 {path, manifest, gitCommitSha?}
  ├─> 确定 pathForGitSha：localSourcePath ?? cacheResult.path
  ├─> await getGitCommitSha(pathForGitSha)   // 若 cachePlugin 未返回 SHA
  ├─> await calculatePluginVersion(pluginId, entry.source, cacheResult.manifest, pathForGitSha, entry.version, cacheResult.gitCommitSha)
  ├─> versionedPath = getVersionedCachePath(pluginId, version)
  ├─> if cacheResult.path !== versionedPath
  │     ├─> mkdir(dirname(versionedPath))
  │     ├─> rm(versionedPath, {recursive: true, force: true})
  │     ├─> 处理 versionedPath 是 cacheResult.path 子目录的边界情况
  │     │     （如 exa-mcp-server@exa-mcp-server）→ 先 rename 到同级 temp，再 rename 到目标
  │     └─> rename(cacheResult.path, versionedPath)
  │     └─> finalPath = versionedPath
  ├─> if isPluginZipCacheEnabled()
  │     └─> await convertDirectoryToZipInPlace(finalPath, zipPath)
  │         └─> finalPath = zipPath
  └─> addInstalledPlugin(pluginId, {version, installedAt, lastUpdated, installPath: finalPath, gitCommitSha}, scope, projectPath)
```

#### 2. `installResolvedPlugin` 安装核心流程

```
installResolvedPlugin({pluginId, entry, scope, marketplaceInstallLocation})
  ├─> scopeToSettingSource(scope)
  ├─> 策略拦截：isPluginBlockedByPolicy(pluginId) ? 返回 blocked-by-policy
  ├─> 本地 source guard：isLocalPluginSource(entry.source) && !marketplaceInstallLocation ? 返回 local-source-no-location
  ├─> 初始化 depInfo Map（缓存 marketplace lookup）
  ├─> if marketplaceInstallLocation → depInfo.set(pluginId, {entry, marketplaceInstallLocation})
  ├─> 获取根市场的 allowCrossMarketplaceDependenciesOn 列表
  ├─> await resolveDependencyClosure(pluginId, resolverFn, alreadyEnabled, allowedCrossMarketplaces)
  │     └─> 返回 {ok: true, closure: string[]} 或 {ok: false, reason, ...}
  ├─> if !resolution.ok → 返回 resolution-failed
  ├─> 依赖策略拦截：遍历 closure，若依赖被 policy block → 返回 dependency-blocked-by-policy
  ├─> 批量写入 settings：updateSettingsForSource(settingSource, {enabledPlugins: {...existing, ...closureEnabled}})
  │     └─> 若失败 → 返回 settings-write-failed
  ├─> Materialize 阶段：遍历 closure
  │     ├─> 若根插件未预填充 marketplaceInstallLocation，则异步补查
  │     ├─> 对本地 source 调用 validatePathWithinBase(marketplaceInstallLocation, source)
  │     └─> await cacheAndRegisterPlugin(id, info.entry, scope, projectPath, localSourcePath)
  ├─> clearAllCaches()
  └─> 返回 {ok: true, closure, depNote}
```

#### 3. `installPluginFromMarketplace` UI 包装流程

```
installPluginFromMarketplace({pluginId, entry, marketplaceName, scope, trigger})
  ├─> await getPluginById(pluginId)  // 为本地 source 插件补查 marketplaceInstallLocation
  ├─> await installResolvedPlugin(...)
  ├─> 若 result.ok === false → 按 reason 映射为用户友好错误消息
  ├─> 若 result.ok === true →
  │     ├─> logEvent('tengu_plugin_installed', {...})
  │     │     含 telemetry redaction：非官方市场 plugin_id 写为 'third-party'
  │     └─> 返回 success message："✓ Installed {name}{depNote}. Run /reload-plugins to activate."
  └─> catch → 返回通用失败消息并 logError
```

### 数据结构

```typescript
export type PluginInstallationInfo = {
  pluginId: string
  installPath: string
  version?: string
}

export type InstallCoreResult =
  | { ok: true; closure: string[]; depNote: string }
  | { ok: false; reason: 'local-source-no-location'; pluginName: string }
  | { ok: false; reason: 'settings-write-failed'; message: string }
  | { ok: false; reason: 'resolution-failed'; resolution: ResolutionResult & { ok: false } }
  | { ok: false; reason: 'blocked-by-policy'; pluginName: string }
  | { ok: false; reason: 'dependency-blocked-by-policy'; pluginName: string; blockedDependency: string }

export type InstallPluginResult =
  | { success: true; message: string }
  | { success: false; error: string }

export type InstallPluginParams = {
  pluginId: string
  entry: PluginMarketplaceEntry
  marketplaceName: string
  scope?: 'user' | 'project' | 'local'
  trigger?: 'hint' | 'user'
}
```

---

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 用途 |
|------|------|
| `cacheAndRegisterPlugin(...)` | 下载/缓存并注册到 installed_plugins.json |
| `registerPluginInstallation(...)` | 不缓存直接注册 |
| `installResolvedPlugin(...)` | 安装核心逻辑（CLI + UI 共享） |
| `installPluginFromMarketplace(...)` | UI 专用包装器 |
| `formatResolutionError(r)` | 依赖解析失败消息格式化 |
| `validatePathWithinBase(...)` | 路径遍历防护 |
| `parsePluginId(pluginId)` | 插件 ID 解析（严格版） |
| `getCurrentTimestamp()` | ISO 时间戳 |
| `PluginInstallationInfo` / `InstallCoreResult` / `InstallPluginResult` / `InstallPluginParams` | 类型定义 |

### 直接依赖文件

- `src/services/analytics/index.ts`：`logEvent`, analytics 类型
- `src/utils/plugins/pluginLoader.ts`：`cachePlugin`, `getVersionedCachePath`, `getVersionedZipCachePath`
- `src/utils/plugins/installedPluginsManager.ts`：`addInstalledPlugin`, `getGitCommitSha`
- `src/utils/plugins/marketplaceManager.ts`：`getMarketplaceCacheOnly`, `getPluginById`
- `src/utils/plugins/dependencyResolver.ts`：`resolveDependencyClosure`, `getEnabledPluginIdsForScope`, `formatDependencyCountSuffix`
- `src/utils/plugins/pluginIdentifier.ts`：`parsePluginIdentifier`, `isOfficialMarketplaceName`, `scopeToSettingSource`
- `src/utils/plugins/pluginPolicy.ts`：`isPluginBlockedByPolicy`
- `src/utils/plugins/pluginVersioning.ts`：`calculatePluginVersion`
- `src/utils/plugins/zipCache.ts`：`convertDirectoryToZipInPlace`, `isPluginZipCacheEnabled`
- `src/utils/plugins/cacheUtils.ts`：`clearAllCaches`
- `src/utils/plugins/managedPlugins.ts`：`getManagedPluginNames`
- `src/utils/settings/settings.ts`：`getSettingsForSource`, `updateSettingsForSource`
- `src/utils/telemetry/pluginTelemetry.ts`：`buildPluginTelemetryFields`
- `src/utils/cwd.ts`：`getCwd`
- `src/utils/fsOperations.ts`：`getFsImplementation`
- `src/utils/log.ts`：`logError`
- `src/utils/errors.ts`：`toError`

### 调用方文件

- `src/services/plugins/pluginOperations.ts`：`installResolvedPlugin`, `formatResolutionError`（用于 `installPluginOp` 和 `updatePluginOp`）
- `src/services/plugins/pluginCliCommands.ts`：间接通过 `pluginOperations.ts`
- `src/commands/plugin/BrowseMarketplace.tsx`：`installPluginFromMarketplace`
- `src/commands/plugin/DiscoverPlugins.tsx`：`installPluginFromMarketplace`
- `src/hooks/useClaudeCodeHintRecommendation.tsx`：`installPluginFromMarketplace`（hint 触发安装）
- `src/hooks/useLspPluginRecommendation.tsx`：`cacheAndRegisterPlugin`（LSP 推荐时的本地缓存）
- `src/utils/plugins/pluginStartupCheck.ts`：`installResolvedPlugin`
- `src/utils/plugins/pluginLoader.ts`：`validatePathWithinBase`

---

## 依赖与外部交互

### 外部系统/文件

- **`installed_plugins.json`**：通过 `addInstalledPlugin` 写入安装记录（V2 格式）。
- **Settings 文件（`settings.json` / `.claude/settings.json` 等）**：`updateSettingsForSource` 批量写入 `enabledPlugins`，将根插件及其依赖闭包一次性标记为启用。
- **Marketplace 缓存/克隆**：`getPluginById` 和 `getMarketplaceCacheOnly` 读取本地 marketplace 数据；`cachePlugin` 可能触发网络下载或 git 克隆。
- **ZIP 缓存（可选）**：当 `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` 启用时，`convertDirectoryToZipInPlace` 将目录转换为 ZIP 并删除原目录。

### 与 Analytics 的交互

- `installPluginFromMarketplace` 在成功安装后发送 `tengu_plugin_installed` 事件：
  - 包含 `_PROTO_plugin_name`、`_PROTO_marketplace_name`（PII-tagged）。
  - `plugin_id` 字段根据 `isOfficialMarketplaceName` 决定是否保留原始值或替换为 `'third-party'`。
  - `trigger` 区分 `hint`（UI 建议触发）和 `user`（用户主动从 marketplace 安装）。

---

## 风险、边界与改进建议

### 风险与边界

1. **`cacheAndRegisterPlugin` 中的子目录 rename 边界情况**
   - 当 marketplace 名等于插件名时（如 `exa-mcp-server@exa-mcp-server`），`versionedPath` 可能成为 `cacheResult.path` 的子目录，导致直接 `rename` 报错。代码通过检测 `isSubdirectory` 并使用临时路径迂回解决，但增加了复杂度和潜在的中间状态残留风险。

2. **`installResolvedPlugin` 中根插件的 `marketplaceInstallLocation` 可选性**
   - 对于非本地 source 的根插件，若调用方未传入 `marketplaceInstallLocation`，代码会在 materialize 阶段异步补查。这增加了额外 IO，且若补查失败会导致根插件被跳过（`if (!info) continue`），出现“设置已写入但插件未缓存”的不一致状态。

3. **依赖闭包的政策检查与 settings 写入顺序**
   - 政策检查在 settings 写入之前，这是正确的。但如果 policy 在 settings 写入后发生变化（企业远程策略热更新），已安装的依赖不会被 retroactively 拦截。这不是本模块的 bug，而是整个策略模型的已知限制。

4. **`validatePathWithinBase` 的 Windows 兼容性**
   - 代码使用 `resolve(basePath) + sep` 进行前缀匹配。在 Windows 上，`resolve` 会返回带盘符的路径（如 `C:\foo\bar`），而 `resolvedPath.startsWith(normalizedBase)` 对大小写不敏感（NTFS 默认不敏感），逻辑上是安全的，但未显式处理 UNC 路径（`\\server\share`）等边缘情况。

5. **`installPluginFromMarketplace` 的 analytics 与错误处理耦合**
   - analytics 事件只在成功路径发送，失败路径仅 `logError`。这意味着安装失败率无法直接从 `tengu_plugin_installed` 事件计算，需要依赖 error log 聚合。

6. **本地 source 的路径遍历防护依赖 marketplace 结构**
   - `validatePathWithinBase` 假设 `marketplaceInstallLocation` 是安全的基目录。如果 marketplace manifest 本身被篡改，将 `source` 设为指向 marketplace 目录内但包含敏感数据的子目录，仍然可以读取该数据。

### 改进建议

1. **将 `cacheAndRegisterPlugin` 拆分为更小的原子步骤**
   - 当前函数长达约 100 行，承担了 source 选择、cache、version 计算、目录移动、ZIP 转换、注册等多项职责。可拆分为 `cachePluginSource`、`moveToVersionedPath`、`maybeConvertToZip`、`registerInstallation` 四个子函数，提升可测试性和可读性。

2. **统一要求 `marketplaceInstallLocation` 必须存在**
   - 在 `installResolvedPlugin` 的入口处，对所有 source 类型（不仅是本地 source）强制要求 `marketplaceInstallLocation`，失败时提前返回明确的错误，避免 materialize 阶段的静默跳过。

3. **增加安装失败 analytics 事件**
   - 在 `installPluginFromMarketplace` 的失败分支中发送 `tengu_plugin_install_failed` 事件，包含 `reason`（如 `blocked-by-policy`、`resolution-failed` 等），便于从 dashboard 直接观测安装漏斗和阻塞原因。

4. **`validatePathWithinBase` 增加符号链接解析**
   - 当前使用 `resolve`（会解析部分符号链接），但为确保安全，可在校验前调用 `realpath` 解析所有符号链接，防止通过 symlink 跳出基目录的攻击。

5. **引入安装事务回滚机制**
   - `installResolvedPlugin` 在 settings 写入成功后、materialize 阶段失败时，目前没有回滚已写入的 settings。可考虑在失败时自动从 settings 中删除刚刚添加的插件条目，或至少向用户返回明确的“部分成功”提示，告知哪些依赖已启用但未缓存。
