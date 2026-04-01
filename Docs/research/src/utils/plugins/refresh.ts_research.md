# 研究报告：src/utils/plugins/refresh.ts

## 场景与职责

`refresh.ts` 是 Claude Code 插件三层模型中的**第三层（Layer 3）**：负责将已物化的插件（Layer 2，磁盘上的 `installed_plugins.json` 和缓存）加载到**当前运行会话的活跃组件**中。它是 `/reload-plugins` 命令和 headless 模式自动刷新的核心实现。

主要职责：
1. **全量刷新插件缓存**：清除所有插件相关的内存缓存，强制从磁盘重新加载。
2. **加载活跃组件**：将插件的命令（commands）、代理（agents）、钩子（hooks）、MCP 服务器、LSP 服务器注入到 AppState 和全局状态中。
3. **触发下游重连**：通过递增 `mcp.pluginReconnectKey` 触发 React effect 重新运行，使 MCP 连接管理器发现新插件服务器。
4. **LSP 管理器重初始化**：确保新启用或禁用的插件 LSP 服务器被正确注册或移除。

## 功能点目的

### 1. `refreshActivePlugins()` — 核心刷新函数
- **目的**：在一个统一的入口中完成插件系统的全量热刷新。
- **调用来源**：
  - `/reload-plugins` 命令（交互式，用户主动触发）
  - `src/cli/print.ts` 的 `refreshPluginState()`（headless 模式，首次查询前自动同步）
  - `performBackgroundPluginInstallations()`（后台自动安装新市场后刷新）

- **NOT 调用来源**（重要设计决策）：
  - `useManagePlugins` 的 `needsRefresh` effect 不再自动调用刷新，而是显示通知引导用户手动执行 `/reload-plugins`。
  - `/plugin` 菜单设置 `needsRefresh` 标志后，同样等待用户手动 `/reload-plugins`。
  - 这一设计（PR 5b/5c）确保了 Layer-3 刷新始终由用户显式触发，避免 mid-conversation 的意外状态变更。

### 2. 缓存清除策略
- **目的**：旧实现中 `needsRefresh` 路径只清除了部分缓存，导致下游 memoized loader 返回陈旧数据。`refreshActivePlugins` 改为调用 `clearAllCaches()` 做**全量清除**。
- **额外清除**：`clearPluginCacheExclusions()` 重新计算孤儿插件过滤器的排除项，因为 `/reload-plugins` 是明确的“磁盘已变更，重新读取”信号。

### 3. 加载时序控制
- **目的**：避免 `#23693` 引入的竞态问题。
- **时序**：
  1. 先 `await loadAllPlugins()`（完整加载，会预热缓存）
  2. 再并行 `await Promise.all([getPluginCommands(), getAgentDefinitionsWithOverrides(...)])`
  3. 再并行加载每个启用插件的 `mcpServers` 和 `lspServers`
  - 若将步骤 2 与步骤 1 并行执行，`getPluginCommands` 可能读取到尚未更新的 `installed_plugins.json` 缓存，导致插件命令缺失。

### 4. 错误合并 (`mergePluginErrors`)
- **目的**：刷新时产生的新错误不应覆盖由其他系统（如 LSP 管理器）记录的现有错误。
- **策略**：
  - 保留 `source === 'lsp-manager'` 或 `source.startsWith('plugin:')` 的现有错误。
  - 对新错误去重，避免同一错误重复出现。

## 具体技术实现

### 关键数据结构

```ts
export type RefreshActivePluginsResult = {
  enabled_count: number
  disabled_count: number
  command_count: number
  agent_count: number
  hook_count: number
  mcp_count: number
  lsp_count: number
  error_count: number
  agentDefinitions: AgentDefinitionsResult
  pluginCommands: Command[]
}

type SetAppState = (updater: (prev: AppState) => AppState) => void
```

### `refreshActivePlugins` 执行流程

```ts
export async function refreshActivePlugins(setAppState: SetAppState): Promise<RefreshActivePluginsResult> {
  // 1. 清除所有缓存
  clearAllCaches()
  clearPluginCacheExclusions()

  // 2. 顺序加载插件、命令、代理
  const pluginResult = await loadAllPlugins()
  const [pluginCommands, agentDefinitions] = await Promise.all([
    getPluginCommands(),
    getAgentDefinitionsWithOverrides(getOriginalCwd()),
  ])

  const { enabled, disabled, errors } = pluginResult

  // 3. 并行加载每个启用插件的 MCP/LSP 服务器
  const [mcpCounts, lspCounts] = await Promise.all([
    Promise.all(enabled.map(async p => {
      if (p.mcpServers) return Object.keys(p.mcpServers).length
      const servers = await loadPluginMcpServers(p, errors)
      if (servers) p.mcpServers = servers
      return servers ? Object.keys(servers).length : 0
    })),
    Promise.all(enabled.map(async p => {
      if (p.lspServers) return Object.keys(p.lspServers).length
      const servers = await loadPluginLspServers(p, errors)
      if (servers) p.lspServers = servers
      return servers ? Object.keys(servers).length : 0
    })),
  ])

  // 4. 更新 AppState
  setAppState(prev => ({
    ...prev,
    plugins: {
      ...prev.plugins,
      enabled,
      disabled,
      commands: pluginCommands,
      errors: mergePluginErrors(prev.plugins.errors, errors),
      needsRefresh: false,
    },
    agentDefinitions,
    mcp: {
      ...prev.mcp,
      pluginReconnectKey: prev.mcp.pluginReconnectKey + 1,
    },
  }))

  // 5. 重新初始化 LSP 管理器
  reinitializeLspServerManager()

  // 6. 加载并注册 hooks（失败不阻断其他数据）
  let hook_load_failed = false
  try {
    await loadPluginHooks()
  } catch (e) {
    hook_load_failed = true
    logError(e)
  }

  // 7. 统计 hook 数量并返回结果
  const hook_count = enabled.reduce(...)
  return { enabled_count: enabled.length, ..., error_count: errors.length + (hook_load_failed ? 1 : 0), ... }
}
```

### `mergePluginErrors` 实现

```ts
function mergePluginErrors(existing: PluginError[], fresh: PluginError[]): PluginError[] {
  const preserved = existing.filter(
    e => e.source === 'lsp-manager' || e.source.startsWith('plugin:')
  )
  const freshKeys = new Set(fresh.map(errorKey))
  const deduped = preserved.filter(e => !freshKeys.has(errorKey(e)))
  return [...deduped, ...fresh]
}

function errorKey(e: PluginError): string {
  return e.type === 'generic-error'
    ? `generic-error:${e.source}:${e.error}`
    : `${e.type}:${e.source}`
}
```

### 文件引用

- `loadAllPlugins()` → `src/utils/plugins/pluginLoader.ts`
- `getPluginCommands()` → `src/utils/plugins/loadPluginCommands.ts`
- `getAgentDefinitionsWithOverrides()` → `src/tools/AgentTool/loadAgentsDir.ts`
- `loadPluginMcpServers()` → `src/utils/plugins/mcpPluginIntegration.ts`
- `loadPluginLspServers()` → `src/utils/plugins/lspPluginIntegration.ts`
- `loadPluginHooks()` → `src/utils/plugins/loadPluginHooks.ts`
- `clearAllCaches()` → `src/utils/plugins/cacheUtils.ts`
- `clearPluginCacheExclusions()` → `src/utils/plugins/orphanedPluginFilter.ts`
- `reinitializeLspServerManager()` → `src/services/lsp/manager.ts`
- `getOriginalCwd()` → `src/bootstrap/state.js`

## 依赖与外部交互

### 上游调用方
- `src/commands/reload-plugins/reload-plugins.ts`：用户执行 `/reload-plugins` 命令时调用。
- `src/cli/print.ts`：headless 模式下，`SYNC_PLUGIN_INSTALL` 流程在首次查询前自动刷新。
- `src/services/plugins/PluginInstallationManager.ts`：后台安装完成后调用。
- `src/state/AppStateStore.ts`：某些状态变更路径间接触发。
- `src/hooks/useManagePlugins.ts`：初始加载使用类似逻辑，但**不调用** `refreshActivePlugins`（注释明确说明）。

### 下游依赖
- `pluginLoader.ts`：加载所有插件的元数据。
- `loadPluginCommands.ts` / `loadPluginAgents.ts`（通过 `getAgentDefinitionsWithOverrides`）：加载插件命令和代理。
- `mcpPluginIntegration.ts` / `lspPluginIntegration.ts`：加载 MCP 和 LSP 服务器配置。
- `loadPluginHooks.ts`：加载并注册钩子回调。
- `cacheUtils.ts`：清除所有插件缓存。
- `orphanedPluginFilter.ts`：清除孤儿插件缓存排除项。
- `services/lsp/manager.ts`：重新初始化 LSP 服务器管理器。

## 风险、边界与改进建议

### 已知风险

1. **`loadPluginHooks` 的失败隔离**
   - `loadPluginHooks()` 被包裹在 `try/catch` 中，失败时只增加 `error_count`，不会导致整个刷新失败。这是正确的：用户不希望因为某个插件的 hook 配置错误就丢失所有命令/代理/MCP。
   - 但 `hook_load_failed` 只计为 1 个错误，丢失了具体是哪个插件、什么错误的详细信息。调用方（`/reload-plugins`）只显示 "1 error"，用户需要去 `/doctor` 查看。

2. **`setAppState` 的竞态**
   - `refreshActivePlugins` 接受 `setAppState` 回调，在内部调用一次。若用户在刷新过程中又触发了其他状态更新（如快速连续执行两次 `/reload-plugins`），React 的 `setAppState` 批处理机制通常能保护一致性，但 `loadAllPlugins` 等缓存清除操作是全局副作用，无法被 React 批处理保护。
   - 当前没有显式的“刷新中”锁，连续快速调用可能导致缓存被清除两次、加载交错。

3. **MCP/LSP 服务器加载的二次遍历**
   - `loadPluginMcpServers` 和 `loadPluginLspServers` 对每个启用插件各调用一次。虽然这些函数内部有 memoize，但 `refreshActivePlugins` 显式将结果写回 `p.mcpServers` / `p.lspServers`，这是为了“预热缓存槽”。
   - 这种写回修改了 `loadAllPlugins()` 返回的 `LoadedPlugin` 对象（虽然对象引用本身是可变的），在严格不可变假设下可能存在隐患。

4. **LSP 管理器的无条件重初始化**
   - `reinitializeLspServerManager()` 在每次刷新时都调用，即使没有任何插件提供 LSP 服务器。注释说明这是为了“移除最后一个 LSP 插件时也能清除过期配置”，且“只是配置解析，服务器是懒启动的”。
   - 但若 LSP 管理器的重初始化成本未来增加（如加入更多 I/O），每次 `/reload-plugins` 的延迟会上升。

### 边界情况

- **无启用插件**：`enabled` 为空数组时，MCP/LSP/hook 的 `Promise.all` 会立即解析为空数组，流程正常。
- **LSP 管理器从未初始化**：headless 子命令路径中 LSP 管理器可能从未初始化。`reinitializeLspServerManager()` 内部会处理为 no-op。
- **Hook 加载失败后的旧 hook 状态**：`loadPluginHooks` 内部采用“先清除再注册”的原子对。若 `loadPluginHooks` 抛错，旧 hook 已在 `clearAllCaches()` 阶段被 `pruneRemovedPluginHooks` 清除（只保留仍启用插件的 hook），因此不会留下完全过时的 hook。

### 改进建议

1. **增加刷新互斥锁**
   - 增加一个模块级布尔标志 `isRefreshing`，在 `refreshActivePlugins` 开始时设为 `true`，结束时（包括异常路径）设为 `false`。若调用时检测到已有刷新在进行中，可选择：
     - 直接返回 "Refresh already in progress"
     - 或排队等待当前刷新完成后再次执行
   - 这能避免快速连续 `/reload-plugins` 导致的全局缓存竞态。

2. **细化 hook 错误信息**
   - 将 `hook_load_failed` 从布尔值改为收集具体错误对象，并合并到 `errors` 数组中（使用 `source: 'plugin-hooks'`）。这样 `/reload-plugins` 的输出能直接显示 "Failed to load plugin hooks: ..."，改善可观测性。

3. **延迟 LSP 重初始化判断**
   - 可在 `refreshActivePlugins` 中记录上一次刷新的 `lsp_count`，若本次与上次均为 0 且中间没有插件变更，则跳过 `reinitializeLspServerManager`。但鉴于当前成本极低，优先级不高。

4. **提取 `mergePluginErrors` 为公共工具**
   - `useManagePlugins.ts` 中也有几乎相同的错误合并逻辑（保留 LSP 错误、去重）。建议将 `mergePluginErrors` 提取到 `src/utils/plugins/errorUtils.ts` 或类似位置，供 Layer-3 的所有入口统一使用。

5. **增加刷新耗时诊断**
   - `refreshActivePlugins` 是用户可感知的同步操作（`/reload-plugins` 会阻塞直到完成）。建议在 `withDiagnosticsTiming` 中包装主要阶段（`loadAllPlugins`、`getPluginCommands`、`loadPluginMcpServers` 等），将耗时写入诊断日志，便于排查“刷新慢”问题。
