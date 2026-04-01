# pluginBlocklist.ts 研究文档

## 场景与职责

`src/utils/plugins/pluginBlocklist.ts` 负责**插件下架（delisting）检测与自动卸载**。当某个插件从其来源 marketplace 的 manifest 中被移除时，该模块能够：

1. 检测已安装但不再被 marketplace 列出的插件（delisted plugins）。
2. 对符合策略的插件执行**自动卸载**。
3. 将被卸载的插件记录到“flagged”列表，以便在 `/plugins` UI 中向用户展示“该插件已被移除”的提示。

注释中特别提到，该模块早期曾从远程拉取 `security.json` 进行安全判定，但为了避免每周约 2950 万次对 GitHub 的请求（#25447），该远程拉取逻辑已被移除。若未来重新引入，应从 `downloads.claude.ai` 提供服务。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `detectDelistedPlugins(installed, marketplace, marketplaceName)` | 纯函数：对比已安装插件列表与给定 marketplace 的 manifest，返回被下架的插件 ID 列表。 |
| `detectAndUninstallDelistedPlugins()` | 核心入口。遍历所有已知 marketplace，对每个启用了 `forceRemoveDeletedPlugins` 的市场执行检测，自动卸载 delisted 插件并标记为 flagged。 |

---

## 具体技术实现

### 关键流程

#### 下架检测与自动卸载流程

```
detectAndUninstallDelistedPlugins()
  ├─> await loadFlaggedPlugins()          // 预热 flagged 缓存
  ├─> loadInstalledPluginsV2()            // 读取当前已安装插件（内存缓存）
  ├─> await loadKnownMarketplacesConfigSafe()  // 安全读取 known_marketplaces.json（异常不抛）
  └─> for each marketplaceName in knownMarketplaces
        ├─> await getMarketplace(marketplaceName)   // 加载 marketplace manifest
        ├─> if !marketplace.forceRemoveDeletedPlugins → continue
        ├─> detectDelistedPlugins(installed, marketplace, marketplaceName)
        │     ├─> 收集 marketplace.plugins 中所有 name 到 Set
        │     └─> 遍历 installedPlugins.plugins 的 key，筛选以 `@marketplaceName` 结尾的 ID
        │         若 name 不在 Set 中 → 加入 delisted 列表
        └─> for each pluginId in delisted
              ├─> 若已在 alreadyFlagged 中 → skip
              ├─> 检查 installations：必须存在至少一个 user/project/local scope 才继续
              │   （managed-only 插件跳过，由企业管理员处理）
              └─> for each user-controllable installation (user/project/local)
                    ├─> await uninstallPluginOp(pluginId, scope)
                    └─> 异常捕获并记录 debug log，不中断流程
              └─> await addFlaggedPlugin(pluginId)
                  └─> newlyFlagged.push(pluginId)
  return newlyFlagged
```

### 数据结构

- 输入：`InstalledPluginsFileV2`（来自 `installedPluginsManager.ts`）
  - `plugins: Record<string, PluginInstallationEntry[]>`
- 输入：`PluginMarketplace`（来自 `marketplaceManager.ts` / `schemas.ts`）
  - `plugins: PluginMarketplaceEntry[]`
  - `forceRemoveDeletedPlugins?: boolean`
- 输出：`string[]` — 本次新被标记为 flagged 的插件 ID 列表。

### 判定规则

1. **后缀匹配**：插件 ID 必须以 `@${marketplaceName}` 结尾，才被认为是该市场的插件。
2. **名称存在性检查**：将插件 ID 去掉后缀后得到 `pluginName`，检查其是否存在于 `marketplace.plugins` 的 `name` 集合中。
3. **Scope 过滤**：
   - 若一个 delisted 插件的所有安装都是 `managed` scope，则**不自动卸载**（注释说明应由企业管理员处理）。
   - 只对 `user`、`project`、`local` scope 执行 `uninstallPluginOp`。
4. **幂等性**：已存在于 `flagged-plugins.json` 中的插件不会被重复卸载/标记。

---

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 用途 |
|------|------|
| `detectDelistedPlugins(installed, marketplace, marketplaceName)` | 单个市场的下架检测纯函数 |
| `detectAndUninstallDelistedPlugins()` | 全量检测并自动卸载（核心入口） |

### 直接依赖文件

- `src/services/plugins/pluginOperations.ts`：`uninstallPluginOp`
- `src/utils/plugins/installedPluginsManager.ts`：`loadInstalledPluginsV2`
- `src/utils/plugins/marketplaceManager.ts`：`getMarketplace`, `loadKnownMarketplacesConfigSafe`
- `src/utils/plugins/pluginFlagging.ts`：`addFlaggedPlugin`, `getFlaggedPlugins`, `loadFlaggedPlugins`
- `src/utils/plugins/schemas.ts`：`InstalledPluginsFileV2`, `PluginMarketplace`
- `src/utils/debug.ts`：`logForDebugging`
- `src/utils/errors.ts`：`errorMessage`

### 调用方文件

- `src/hooks/useManagePlugins.ts`：在 `initialPluginLoad()`（REPL mount 时的初始插件加载）中调用，确保用户每次进入交互模式时都会执行下架检测。
- `src/utils/plugins/headlessPluginInstall.ts`：在 `installPluginsForHeadless()` 中调用，为非交互式/headless 模式（如 CCR）提供同样的 delisting  enforcement。
- `src/main.tsx`（print/headless 路径）：通过 `headlessPluginInstall.ts` 间接调用。

---

## 依赖与外部交互

### 外部文件/配置

- **`installed_plugins.json`**（V2）：读取已安装插件列表及每个插件的 scope/projectPath。
- **`known_marketplaces.json`**：获取已知市场列表，用于遍历检测。
- **Marketplace manifest（`marketplace.json`）**：通过 `getMarketplace()` 从本地缓存（或远程克隆）加载，作为“当前官方插件列表”的基准。
- **`flagged-plugins.json`**：通过 `pluginFlagging.ts` 读写，记录已下架并被自动卸载的插件，供 UI 展示。

### 与卸载系统的交互

- 直接调用 `uninstallPluginOp(pluginId, scope)`，该函数会：
  1. 从对应 scope 的 settings 中移除 `enabledPlugins` 条目；
  2. 从 `installed_plugins.json` 中移除该 scope 的安装记录；
  3. 将旧版本缓存标记为 orphaned，供后台 GC 清理；
  4. 若是最后一个 scope，则删除插件数据和配置。

---

## 风险、边界与改进建议

### 风险与边界

1. **`security.json` 已移除，纯基于 manifest 存在性判断**
   - 当前逻辑仅判断“插件是否还在 marketplace.json 的 `plugins` 数组中”。若 marketplace 作者因临时构建故障短暂移除某插件，该插件会被立即判定为 delisted 并自动卸载。没有额外的安全/原因字段做分级处理。

2. **`forceRemoveDeletedPlugins` 默认行为**
   - 该字段由 marketplace manifest 控制。若第三方市场未设置或设置为 `false`，则该市场下的 delisted 插件不会被自动清理，用户可能长期运行一个“已不存在于市场”的插件。

3. **managed-only 插件跳过卸载**
   - 设计上尊重企业管理员权限，但这也意味着如果企业策略强制推送的某个插件被官方下架，且用户没有 user-scope 安装，则该插件会保留在系统中，直到管理员干预。

4. **异常安全**
   - 内部对每个市场的加载、每个插件的卸载都包裹了 `try/catch`，单个失败不会中断整体流程。但失败仅记录到 debug log，用户和 telemetry 均无法直接感知某个特定插件的卸载失败。

5. **与 marketplace 加载的耦合**
   - `getMarketplace()` 可能触发 git pull 或网络请求。在 `useManagePlugins` 的初始加载路径中，delisting 检测与插件加载串行执行，若 marketplace 网络缓慢，会拖慢整个 `initialPluginLoad` 的完成时间。

### 改进建议

1. **引入下架原因与宽限期**
   - 若未来恢复远程安全文件，建议不仅返回“是否下架”，还返回下架原因（security issue / deprecated / renamed）和宽限期时间戳，避免误杀或立即强制卸载。

2. **将 delisting 检测异步化或延后**
   - 当前在 `useManagePlugins` 的 mount 路径中同步 `await`，可考虑将其推迟到首次 `/plugins` 打开时或一个独立的低优先级后台任务中，减少启动阻塞。

3. **增加 telemetry 事件**
   - 对 `detectAndUninstallDelistedPlugins` 的结果（检测到的数量、成功卸载数量、失败数量）发送专门的 analytics 事件，便于监控 marketplace 健康度和卸载成功率。

4. **用户可配置的自动卸载开关**
   - 在 settings 中增加 `autoUninstallDelistedPlugins` 全局开关，允许高级用户选择“仅标记不自动卸载”，提升控制权。
