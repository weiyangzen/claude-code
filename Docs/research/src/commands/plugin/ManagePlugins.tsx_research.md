# ManagePlugins.tsx 深度研究文档

> 研究对象：`src/commands/plugin/ManagePlugins.tsx` 及其直接/间接依赖（调用方、被调用方、配置、测试上下文、脚本）。
> 研究范围：仅限代码、脚本、配置、测试及必要实现上下文；不涉及 README、docs、Docs、markdown 等文档。

---

## 1. 场景与职责

### 1.1 所在位置与调用链

`ManagePlugins.tsx` 是 Claude Code CLI 的 `/plugin`（别名 `/plugins`、`/marketplace`）命令族中“已安装插件”管理界面的核心 React 组件。它位于命令渲染链的末端：

```
CLI 输入 /plugin manage|enable|disable|uninstall <plugin>
  → src/commands/plugin/index.tsx   (Command 定义，immediate: true)
  → src/commands/plugin/plugin.tsx  (call() → <PluginSettings />)
  → src/commands/plugin/PluginSettings.tsx (标签页容器，状态机路由)
  → src/commands/plugin/ManagePlugins.tsx  (Installed 标签页内容)
```

在 `PluginSettings.tsx` 中，`getInitialViewState()` 将 `manage`/`enable`/`disable`/`uninstall` 子命令统一映射到 `viewState.type === 'manage-plugins'`，随后渲染 `<ManagePlugins />` 并传入 `targetPlugin`、`targetMarketplace`、`action` 等可选参数，实现“列表浏览”或“直达操作”两种模式。

### 1.2 核心职责

1. **统一展示已安装插件与 MCP Server**：将传统插件（LoadedPlugin）和 MCP Server（stdio/sse/http/claudeai-proxy）合并为同一份列表，按 scope 分组展示。
2. **插件生命周期操作 UI**：支持 Enable / Disable / Update / Uninstall，以及内置插件的有限操作（仅 enable/disable）。
3. **失败插件与下架插件的治理**：展示加载失败的插件（orphan errors）和被 marketplace 下架的插件（flagged plugins），并提供移除/忽略入口。
4. **MCPB 配置向导集成**：对包含 `.mcpb`/`.dxt` 文件的插件，支持“Configure”入口，拉起 `PluginOptionsDialog` / `PluginOptionsFlow` 进行交互式配置。
5. **项目级卸载的冲突处理**：当插件在 `.claude/settings.json`（project scope，团队共享）中被启用时，卸载前会提示用户改为在 `.claude/settings.local.json` 中禁用，避免影响其他成员。
6. **数据目录清理确认**：卸载最后一个 scope 的插件前，若该插件存在 `${CLAUDE_PLUGIN_DATA}` 持久数据，会提示用户是否一并删除。

---

## 2. 功能点目的

### 2.1 统一列表（Unified List）

Claude Code 的插件体系经历了从“纯插件”到“插件 + MCP Server”的演进。`ManagePlugins` 的设计目标是将以下四类实体合并到单一列表中，降低用户认知成本：

- **正常插件**（`type: 'plugin'`）：从 marketplace 安装并成功加载的 `LoadedPlugin`。
- **失败插件**（`type: 'failed-plugin'`）：在 `loadAllPlugins()` 中未能加载、但 settings 中仍声明为启用的插件。
- **下架插件**（`type: 'flagged-plugin'`）：被 marketplace 下架（delisted），由后台标记到 `flagged-plugins.json` 中的插件。
- **MCP Server**（`type: 'mcp'`）：包括插件子 MCP（`plugin:<pluginName>:<serverName>`）和独立 MCP（如 IDE 集成、用户手动添加的 server）。

### 2.2 Scope 分层与权限控制

插件系统支持多 scope 安装/启用：

| Scope | 含义 | 可安装 | 可卸载 | 可启用/禁用 |
|-------|------|--------|--------|-------------|
| `builtin` | 内置插件 | 否 | 否 | 是 |
| `managed` | 组织策略强制安装 | 否（仅策略） | 否 | 否（仅更新） |
| `user` | 用户级（`~/.claude/settings.json`） | 是 | 是 | 是 |
| `project` | 项目级（`.claude/settings.json`） | 是 | 是 | 是 |
| `local` | 本地覆盖（`.claude/settings.local.json`） | 是 | 是 | 是 |

`ManagePlugins` 通过 `isInstallableScope()` 和 `filterManagedDisabledPlugins()` 等守卫，确保用户不能对企业策略管控的插件执行越权操作。

### 2.3 自动导航与自动执行（Auto-Navigate / Auto-Action）

当用户通过 CLI 子命令直接操作（如 `/plugin uninstall foo`）时，`ManagePlugins` 并非仅打开列表，而是：

1. 在 `marketplaces` 和 `pluginStates` 加载完成后，`useEffect` 根据 `targetPlugin` 自动定位到对应插件；
2. 若找到，自动将 `viewState` 切换到 `plugin-details` 或 `failed-plugin-details`；
3. 再通过 `pendingAutoActionRef` 触发对应的 `handleSingleOperation(action)`，实现“一键直达”。

若目标插件未安装，则通过 `setResult()` 向用户返回明确错误信息，而非静默停留在列表页。

---

## 3. 具体技术实现

### 3.1 组件结构与状态机

`ManagePlugins` 是一个大型函数组件（约 2200 行编译后源码），内部维护多层状态：

#### Props

```ts
type Props = {
  setViewState: (state: ParentViewState) => void;   // 返回上层（如 menu）
  setResult: (result: string | null) => void;       // 操作结果提示
  onManageComplete?: () => void | Promise<void>;    // 操作完成回调（标记 plugins changed）
  onSearchModeChange?: (isActive: boolean) => void; // 搜索模式通知父级
  targetPlugin?: string;                             // 自动导航目标
  targetMarketplace?: string;                        // 限定 marketplace
  action?: 'enable' | 'disable' | 'uninstall';       // 自动执行动作
};
```

#### 内部 ViewState（页面级状态机）

```ts
type ViewState =
  | 'plugin-list'           // 主列表
  | 'plugin-details'        // 插件详情菜单
  | 'configuring'           // MCPB 配置中
  | { type: 'plugin-options' }           // 插件选项流（manifest.userConfig）
  | { type: 'configuring-options'; schema: PluginOptionSchema }
  | 'confirm-project-uninstall'          // 项目级启用冲突确认
  | { type: 'confirm-data-cleanup'; size: { bytes: number; human: string } }
  | { type: 'flagged-detail'; plugin: FlaggedPluginInfo }
  | { type: 'failed-plugin-details'; plugin: FailedPluginInfo }
  | { type: 'mcp-detail'; client: MCPServerConnection }
  | { type: 'mcp-tools'; client: MCPServerConnection }
  | { type: 'mcp-tool-detail'; client: MCPServerConnection; tool: Tool };
```

状态切换通过 `setViewState` 完成，Escape/Back 逻辑在 `handleBack` 中集中处理，形成可逆的导航栈。

### 3.2 统一列表的数据合成（`unifiedItems` useMemo）

这是组件最核心的数据流，约 260 行逻辑，分为以下阶段：

#### 阶段 A：收集插件子 MCP

遍历 `mcpClients`（来自 `useAppState(s => s.mcp.clients)`），将 `name` 以 `plugin:` 开头的 client 按 `pluginName` 分组，构建 `pluginMcpMap`。

#### 阶段 B：构建插件项

遍历 `pluginStates`（由 `loadInstalledPlugins` effect 生成），为每个插件计算：

- `isEnabled`：从 `getSettings_DEPRECATED().enabledPlugins[pluginId]` 读取（merged settings，尊重 local > project > user 的覆盖层级）。
- `errors`：从 `useAppState(s => s.plugins.errors)` 中过滤出与该插件相关的 `PluginError`。
- `scope`：内置插件为 `builtin`，其余从 V2 `installed_plugins.json` 读取。
- `pendingToggle` / `pendingUpdate`：UI 层面的临时标记。

#### 阶段 C：收集孤儿错误（Orphan Errors）

有些插件在 settings 中被声明为启用，但 `loadAllPlugins()` 完全未能加载（连 `LoadedPlugin` 对象都没有）。这些错误通过 `orphanErrorsBySource` 收集，生成 `failed-plugin` 类型的列表项。

#### 阶段 D：收集独立 MCP

排除 `name === 'ide'` 和 `name.startsWith('plugin:')` 的 client，剩余为独立 MCP。

#### 阶段 E：收集下架插件（Flagged）

从 `getFlaggedPlugins()` 读取，生成 `flagged-plugin` 项。首次渲染后会调用 `markFlaggedPluginsSeen(flaggedIds)`，48 小时后自动从 `flagged-plugins.json` 清除。

#### 阶段 F：按 Scope 分组并排序

Scope 的显示优先级硬编码为：

```ts
const scopeOrder: Record<string, number> = {
  flagged: -1,
  project: 0,
  local: 1,
  user: 2,
  enterprise: 3,
  managed: 4,
  dynamic: 5,
  builtin: 6,
};
```

每组内部先排插件（含嵌套子 MCP），再排独立 MCP，均按名称字母序。

### 3.3 搜索与分页

- **搜索**：使用 `useSearchInput` hook 处理输入，`/` 或任意可打印字符进入搜索模式，实时过滤 `unifiedItems`（按 `name` 和 `description` 匹配）。
- **分页**：使用本地 `usePagination` hook（`maxVisible: 8`），实现连续滚动（continuous scrolling），而非传统翻页。选中项始终保持在可视区域内。

### 3.4 插件操作的核心流程

#### Enable / Disable

调用 `enablePluginOp(pluginId)` / `disablePluginOp(pluginId)`（来自 `src/services/plugins/pluginOperations.ts`）。

**关键设计**：`ManagePlugins` 在调用时**不传递 scope**，而是让 `pluginOperations.ts` 通过 `findPluginInSettings()` 自动检测最具体的 scope（local > project > user）。这是为了避免 `installed_plugins.json` 中的安装 scope 与 `settings.json` 中的启用 scope 不一致时触发跨 scope 守卫（cross-scope guard）。

操作成功后：
1. `clearAllCaches()` 清除插件缓存；
2. 若操作后插件处于启用状态，自动跳转到 `plugin-options`（`PluginOptionsFlow`），检查是否有未配置的 `manifest.userConfig` 或 channel userConfig；
3. 否则返回结果消息，提示用户运行 `/reload-plugins` 生效。

#### Update

调用 `updatePluginOp(pluginId, pluginScope)`。该函数执行**非原地更新（non-inplace update）**：

1. 从 marketplace 获取最新版本信息；
2. 下载/计算新版本号；
3. 复制到新版本缓存目录；
4. 更新 `installed_plugins.json` 中的 `installPath`（仅写磁盘，不写内存）；
5. 旧版本若不再被引用则标记为 orphan，供后续 GC。

若 `alreadyUpToDate === true`，直接返回提示并关闭界面。

#### Uninstall

流程最为复杂，涉及多层确认：

1. **内置/托管守卫**：`builtin` 和 `managed` scope 直接拒绝。
2. **项目级启用检查**：若 `isPluginEnabledAtProjectScope(pluginId)` 为 true，跳转到 `confirm-project-uninstall` 视图，建议用户改为在 `localSettings` 中写 `enabledPlugins[pluginId] = false`。
3. **数据目录检查**：若这是最后一个 scope 的安装（`installs.length <= 1`），且存在 `${CLAUDE_PLUGIN_DATA}`，则跳转到 `confirm-data-cleanup` 视图，提示 `y` 删除 / `n` 保留 / `esc` 取消。
4. **执行卸载**：调用 `uninstallPluginOp(pluginId, pluginScope, deleteDataDir)`，该函数会：
   - 从对应 settings source 中删除 `enabledPlugins` 键；
   - 从 `installed_plugins.json` 移除该 scope 的安装记录；
   - 若为最后 scope，删除插件选项（`deletePluginOptions`）和数据目录（可选）。

### 3.5 MCPB 配置流程

当插件详情菜单中的“Configure”被选中时：

1. 从 `selectedPlugin.plugin.manifest.mcpServers` 中找出 `.mcpb`/`.dxt` 路径；
2. 调用 `loadMcpbFile(mcpbPath, pluginPath, pluginId, undefined, undefined, true)`，最后一个参数 `forceConfigDialog = true` 确保即使已有缓存也进入配置；
3. 若返回 `status === 'needs-config'`，进入 `configuring` 视图，渲染 `PluginOptionsDialog`；
4. 用户填写后，再次调用 `loadMcpbFile(..., providedUserConfig)` 生成最终的 `McpServerConfig`；
5. 配置保存到 `settings.json`（非敏感字段）和 secureStorage/keychain（敏感字段），通过 `saveMcpServerUserConfig` 实现分存。

### 3.6 失败插件的恢复卸载

在 `failed-plugin-details` 视图中，用户只能执行“Remove”。该路径会：

1. 优先调用 `uninstallPluginOp(pluginId, scope, false)`（`deleteDataDir = false`，保留数据，因为这是恢复路径）；
2. 若失败（插件从未安装，仅在 settings 中有启用记录），则回退到手动遍历 `userSettings`/`projectSettings`/`localSettings`，将 `enabledPlugins[pluginId]` 设为 `undefined`（通过 `updateSettingsForSource` 的 mergeWith 语义实现删除）。

---

## 4. 关键代码路径与文件引用

### 4.1 目标文件

- `src/commands/plugin/ManagePlugins.tsx` — 主组件，约 2200 行（含 source map）。

### 4.2 直接调用方

- `src/commands/plugin/PluginSettings.tsx` — 标签页容器，负责将 `viewState` 路由到 `ManagePlugins`。
- `src/commands/plugin/plugin.tsx` — `/plugin` 命令的入口，渲染 `<PluginSettings />`。
- `src/commands/plugin/index.tsx` — `Command` 定义，声明 `name: 'plugin'`，`immediate: true`。

### 4.3 直接子组件 / 辅助组件

- `src/commands/plugin/UnifiedInstalledCell.tsx` — 统一列表的单元格渲染，处理 plugin / flagged-plugin / failed-plugin / mcp 四种类型及不同状态图标。
- `src/commands/plugin/PluginOptionsDialog.tsx` — 通用配置对话框，支持敏感字段掩码、多字段步进输入。
- `src/commands/plugin/PluginOptionsFlow.tsx` — 配置向导流，串联 top-level options 和 per-channel MCP options。
- `src/commands/plugin/usePagination.ts` — 连续滚动分页 hook。
- `src/commands/plugin/PluginErrors.tsx` — 错误格式化与引导（`formatErrorMessage`、`getErrorGuidance`）。

### 4.4 核心服务层依赖

- `src/services/plugins/pluginOperations.ts` — 插件操作的纯库函数：
  - `enablePluginOp` / `disablePluginOp` / `uninstallPluginOp` / `updatePluginOp`
  - `isInstallableScope` / `isPluginEnabledAtProjectScope` / `getPluginInstallationFromV2`
- `src/utils/plugins/pluginLoader.ts` — 插件加载器：
  - `loadAllPlugins()` — 返回 `{ enabled: LoadedPlugin[], disabled: LoadedPlugin[] }`
  - 缓存路径计算、版本化缓存、git/npm/local 安装逻辑
- `src/utils/plugins/installedPluginsManager.ts` — V2 安装元数据管理：
  - `loadInstalledPluginsV2()` / `loadInstalledPluginsFromDisk()`
  - `removePluginInstallation()` / `updateInstallationPathOnDisk()`
- `src/utils/plugins/marketplaceManager.ts` — Marketplace 读取：
  - `getMarketplace(marketplaceName)` / `getPluginById(pluginId)`
- `src/utils/plugins/mcpbHandler.ts` — MCPB/DXT 文件处理：
  - `isMcpbSource()` / `loadMcpbFile()` / `saveMcpServerUserConfig()` / `validateUserConfig()`
- `src/utils/plugins/pluginOptionsStorage.ts` — 插件选项存储：
  - `loadPluginOptions()` / `savePluginOptions()` / `deletePluginOptions()` / `getUnconfiguredOptions()`
- `src/utils/plugins/pluginFlagging.ts` — 下架插件标记：
  - `getFlaggedPlugins()` / `markFlaggedPluginsSeen()` / `removeFlaggedPlugin()`
- `src/utils/plugins/pluginDirectories.ts` — 目录工具：
  - `getPluginDataDirSize()` / `pluginDataDirPath()` / `deletePluginDataDir()`
- `src/utils/plugins/pluginPolicy.ts` — 策略检查：
  - `isPluginBlockedByPolicy()`
- `src/utils/plugins/pluginStartupCheck.ts` — 启动检查：
  - `getPluginEditableScopes()`
- `src/utils/plugins/cacheUtils.ts` — 缓存清理：
  - `clearAllCaches()`

### 4.5 状态与设置依赖

- `src/state/AppState.ts` — `useAppState` 读取 `mcp.clients`、`mcp.tools`、`plugins.errors`、`plugins.installationStatus`。
- `src/utils/settings/settings.ts` — `getSettings_DEPRECATED()`、`getSettingsForSource()`、`updateSettingsForSource()`。

### 4.6 MCP 相关组件

- `src/components/mcp/MCPStdioServerMenu.tsx`
- `src/components/mcp/MCPRemoteServerMenu.tsx`
- `src/components/mcp/MCPToolListView.tsx`
- `src/components/mcp/MCPToolDetailView.tsx`
- `src/services/mcp/MCPConnectionManager.ts` — `useMcpToggleEnabled()`

---

## 5. 依赖与外部交互

### 5.1 文件系统交互

`ManagePlugins` 本身不直接操作文件系统，但通过依赖链间接涉及以下路径：

| 路径/文件 | 用途 |
|-----------|------|
| `~/.claude/plugins/installed_plugins.json` | V2 安装元数据（scope、installPath、version 等） |
| `~/.claude/plugins/flagged-plugins.json` | 下架插件标记与 seenAt 时间戳 |
| `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/` | 插件版本化缓存目录 |
| `~/.claude/plugins/data/<pluginId>/` | 插件持久数据目录（`${CLAUDE_PLUGIN_DATA}`） |
| `.claude/settings.json` | project scope 的启用状态 |
| `.claude/settings.local.json` | local scope 的启用状态 |
| `~/.claude/settings.json` | user scope 的启用状态及 pluginConfigs |
| `pluginPath/../.claude-plugin/marketplace.json` | 读取原始 marketplace 数据以检测 MCPB |

### 5.2 网络交互

- `getMarketplace()` 和 `getPluginById()` 可能触发 marketplace 数据加载（本地缓存优先）。
- `loadMcpbFile()` 在 MCPB 为 URL 时会通过 `axios` 下载（120s 超时）。
- `updatePluginOp()` 对远程插件会执行 git clone / npm install。

### 5.3 安全/密钥链交互

- `saveMcpServerUserConfig()` 和 `savePluginOptions()` 将 `sensitive: true` 的字段写入 `secureStorage`（macOS 钥匙串或 `.credentials.json`）。
- `loadPluginOptions()` 每次读取时会同步调用 `security find-generic-password`（经 memoize 缓存，每会话每插件一次）。

### 5.4 进程/子命令交互

- 插件更新/缓存过程中会调用 `git`、`npm` 等外部命令（封装在 `pluginLoader.ts`）。
- MCP Server 的启用/禁用通过 `useMcpToggleEnabled()` 触发，最终影响 MCP 连接管理器的子进程生命周期。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险与边界

#### A. `getSettings_DEPRECATED` 的合并语义陷阱

`ManagePlugins` 大量使用 `getSettings_DEPRECATED()` 读取合并后的 settings。这在判断 `isEnabled` 时是正确的（需要尊重 local > project > user 的覆盖层级），但在**写入**时存在隐患：

- 代码中 `confirm-project-uninstall` 的处理直接写 `localSettings`，这是安全的；
- 但 `PluginOptionsDialog` / `PluginOptionsFlow` 在保存配置时，使用 `getSettings_DEPRECATED()` 作为基线，再 `updateSettingsForSource('userSettings', ...)`。如果未来支持 project-scoped plugin options，这种“读取合并、写入 user”的模式会把 project 层的配置泄漏到 user 层（`pluginOptionsStorage.ts` 中已有 TODO 注释指出此问题）。

#### B. `unifiedItems` 的同步 `getFlaggedPlugins()`

`getFlaggedPlugins()` 返回模块级缓存，若 `loadFlaggedPlugins()` 尚未被调用（如某些非标准启动路径），可能返回空对象，导致 flagged 插件不显示。虽然正常 REPL 启动流程会调用，但 headless 或测试环境可能遗漏。

#### C. 孤儿错误（Orphan Errors）的 scope 推断

对于完全加载失败的插件，`ManagePlugins` 通过 `getPluginEditableScopes()` 推断 scope。若返回 `undefined` 或 `'flag'`，代码会回退到 `'user'`。这可能导致失败插件被错误地归类到 User scope，而不是它实际声明的 scope。

#### D. MCP Server 详情视图的类型转换重复

`mcp-detail`、`mcp-tools`、`mcp-tool-detail` 三个视图都包含几乎相同的 `MCPServerConnection → ServerInfo` 转换逻辑（约 30 行 × 3 处）。若新增 transport 类型（如 `websocket`），需要修改三处，容易遗漏。

#### E. 搜索模式下的键盘冲突

搜索模式通过 `useSearchInput` 处理输入，但 `plugin:toggle`（Space）的 keybinding 在列表非搜索模式下仍然注册。若用户快速从搜索模式退出后按 Space，可能意外触发 toggle。当前通过 `isActive: viewState === 'plugin-list' && !isSearchMode` 守卫，但状态切换的竞态窗口理论上存在。

#### F. `PluginComponentsDisplay` 的 I/O 密集

该子组件在 `useEffect` 中对每个插件的 `commandsPath`、`agentsPath`、`skillsPath` 等执行多次 `fs.readdir`。在插件数量多、或路径位于慢速网络文件系统时，可能导致列表渲染延迟。虽然返回 `null` 时不阻塞 UI，但 effect 仍会执行。

### 6.2 改进建议

#### 1. 提取 `ServerInfo` 转换函数

将 `client.config.type → StdioServerInfo | SSEServerInfo | HTTPServerInfo | ClaudeAIServerInfo` 的转换逻辑提取为独立工具函数（如 `src/components/mcp/serverInfoHelpers.ts`），供 `ManagePlugins` 和其他 MCP 菜单复用，消除三处重复。

#### 2. 延迟加载 `PluginComponentsDisplay`

仅在用户进入 `plugin-details` 视图后才加载组件清单，而非在列表渲染时就为每个插件触发 `fs.readdir`。或者将组件信息缓存到 `LoadedPlugin` / marketplace 索引中，避免运行时遍历文件系统。

#### 3. 明确 `getSettings_DEPRECATED` 在写路径中的使用范围

对 `savePluginOptions` 和 `saveMcpServerUserConfig` 增加 scope 参数，确保写入时以目标 scope 的原始 settings 为基线，而不是合并后的全局视图。这能为未来 project-scoped options 铺平道路。

#### 4. 增加 `ManagePlugins` 的单元测试覆盖

当前未找到针对 `ManagePlugins` 的专用测试文件。建议补充以下场景的测试：

- `unifiedItems` 的排序与分组逻辑（可提取为纯函数测试）；
- `handleSingleOperation` 的各分支（enable/disable/update/uninstall）与错误处理；
- 自动导航逻辑（`targetPlugin` 匹配成功/失败/失败插件回退）；
- `confirm-project-uninstall` 和 `confirm-data-cleanup` 的状态转换。

由于组件依赖大量全局状态（`useAppState`、`fs`），测试时需要对 `AppState` 和 `pluginOperations` 进行 mock。

#### 5. 统一 `unifiedTypes` 的类型定义

`UnifiedInstalledItem` 类型目前似乎仅在编译后的 `.js` 导入中存在，源码中 `src/commands/plugin/unifiedTypes.ts` 缺失（或已被内联/重命名）。建议确保类型定义文件与导入路径一致，避免 TypeScript 编译或源码阅读时的困惑。

#### 6. 优化 `useMemo` 依赖

`unifiedItems` 的 `useMemo` 依赖数组包含 `pluginStates`、`mcpClients`、`pluginErrors`、`pendingToggles`、`flaggedPlugins`。其中 `mcpClients` 和 `pluginErrors` 是数组/对象引用，若上层状态管理未做稳定化，可能导致 `unifiedItems` 在无关更新时重复计算。建议在上游 `AppState` 中使用 Immer 或结构共享，确保引用稳定性。

---

## 7. 附录：关键类型推断

由于 `src/commands/plugin/unifiedTypes.ts` 源码文件在当前仓库中不可见，根据 `ManagePlugins.tsx` 和 `UnifiedInstalledCell.tsx` 的用法，可推断 `UnifiedInstalledItem` 的联合类型结构如下：

```ts
type UnifiedInstalledItem =
  | {
      type: 'plugin';
      id: string;
      name: string;
      description?: string;
      marketplace: string;
      scope: string;
      isEnabled: boolean;
      errorCount: number;
      errors: PluginError[];
      plugin: LoadedPlugin;
      pendingEnable?: boolean;
      pendingUpdate?: boolean;
      pendingToggle?: 'will-enable' | 'will-disable';
    }
  | {
      type: 'flagged-plugin';
      id: string;
      name: string;
      marketplace: string;
      scope: 'flagged';
      reason: string;
      text: string;
      flaggedAt: string;
    }
  | {
      type: 'failed-plugin';
      id: string;
      name: string;
      marketplace: string;
      scope: string;
      errorCount: number;
      errors: PluginError[];
    }
  | {
      type: 'mcp';
      id: string;
      name: string;
      description?: string;
      scope: string;
      status: 'connected' | 'disabled' | 'pending' | 'needs-auth' | 'failed';
      client: MCPServerConnection;
      indented?: boolean;
    };
```

该类型是 `ManagePlugins` 与 `UnifiedInstalledCell` 之间的核心契约。
