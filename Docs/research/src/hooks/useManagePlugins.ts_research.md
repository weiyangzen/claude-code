# useManagePlugins.ts 深度研究文档

## 场景与职责

`useManagePlugins` 是 Claude Code 插件系统的核心管理钩子，负责插件的初始加载、状态管理和刷新通知。它是 Layer-3 插件架构的入口点，协调命令、代理、钩子、MCP 和 LSP 组件的加载。

### 核心场景

1. **初始插件加载**：启动时加载所有插件，收集元数据
2. **下架插件处理**：自动检测并卸载被下架的插件
3. **标记插件通知**：向用户显示被标记插件的警告
4. **刷新通知**：当插件状态变化时提示用户运行 `/reload-plugins`

### 与其他组件的关系

- 被 `REPL.tsx` 在挂载时调用
- 与 `/reload-plugins` 命令配合完成插件刷新
- 与 `AppState.plugins` 集成，存储插件状态
- 与通知系统集成，显示插件相关通知

---

## 功能点目的

### 1. 初始插件加载

启动时执行一次（`enabled=true`）：

```typescript
const initialPluginLoad = useCallback(async () => {
  // 1. 加载所有插件
  const { enabled, disabled, errors } = await loadAllPlugins()
  
  // 2. 检测下架插件
  await detectAndUninstallDelistedPlugins()
  
  // 3. 显示标记插件通知
  const flagged = getFlaggedPlugins()
  if (Object.keys(flagged).length > 0) {
    addNotification({ key: 'plugin-delisted-flagged', ... })
  }
  
  // 4. 加载各种组件
  const commands = await getPluginCommands()
  const agents = await loadPluginAgents()
  await loadPluginHooks()
  const mcpServers = await loadPluginMcpServers()
  const lspServers = await loadPluginLspServers()
  
  // 5. 更新 AppState
  setAppState(prev => ({ ...prev, plugins: { enabled, disabled, commands, errors } }))
  
  // 6. 发送遥测
  logEvent('tengu_plugins_loaded', metrics)
}, [])
```

### 2. 下架插件处理

自动检测并卸载：
- 检查已安装插件是否在下架列表中
- 自动卸载被下架的插件
- 记录为标记插件供用户查看

### 3. 组件加载与错误处理

每个组件类型独立加载，错误不互相影响：

| 组件类型 | 加载函数 | 错误处理 |
|---------|---------|---------|
| Commands | `getPluginCommands()` | 记录到 errors 数组 |
| Agents | `loadPluginAgents()` | 记录到 errors 数组 |
| Hooks | `loadPluginHooks()` | 记录到 errors 数组 |
| MCP | `loadPluginMcpServers()` | 记录到 errors 数组 |
| LSP | `loadPluginLspServers()` | 记录到 errors 数组 |

### 4. 刷新通知

当 `plugins.needsRefresh` 为 true 时：
- 显示通知："Plugins changed. Run /reload-plugins to activate."
- **不自动刷新**（避免缓存问题）
- 用户运行 `/reload-plugins` 后通过 `refreshActivePlugins()` 完成刷新

---

## 具体技术实现

### 关键数据结构

```typescript
interface Props {
  enabled?: boolean  // 默认 true
}

// 返回的指标
interface PluginMetrics {
  enabled_count: number
  disabled_count: number
  inline_count: number        // @inline 插件
  marketplace_count: number   // 市场插件
  error_count: number
  skill_count: number         // 命令数量
  agent_count: number
  hook_count: number
  mcp_count: number
  lsp_count: number
  ant_enabled_names?: string  // Ant-only: 启用的插件名列表
  load_failed?: boolean
}
```

### 核心流程

#### 1. 初始加载 Effect

```typescript
useEffect(() => {
  if (!enabled) return
  void initialPluginLoad().then(metrics => {
    const { ant_enabled_names, ...baseMetrics } = metrics
    logEvent('tengu_plugins_loaded', {
      ...baseMetrics,
      has_custom_plugin_cache_dir: !!process.env.CLAUDE_CODE_PLUGIN_CACHE_DIR,
      ...(ant_enabled_names !== undefined && { enabled_names: ant_enabled_names })
    })
    logForDiagnosticsNoPII('info', 'tengu_plugins_loaded', allMetrics)
  })
}, [initialPluginLoad, enabled])
```

#### 2. MCP 服务器计数

```typescript
// 需要单独计数，因为 loadAllPlugins 不填充 mcpServers
const mcpServerCounts = await Promise.all(
  enabled.map(async p => {
    if (p.mcpServers) return Object.keys(p.mcpServers).length
    const servers = await loadPluginMcpServers(p, errors)
    if (servers) p.mcpServers = servers
    return servers ? Object.keys(servers).length : 0
  })
)
const mcp_count = mcpServerCounts.reduce((sum, n) => sum + n, 0)
```

#### 3. LSP 服务器初始化

```typescript
const lspServerCounts = await Promise.all(
  enabled.map(async p => {
    if (p.lspServers) return Object.keys(p.lspServers).length
    const servers = await loadPluginLspServers(p, errors)
    if (servers) p.lspServers = servers
    return servers ? Object.keys(servers).length : 0
  })
)
const lsp_count = lspServerCounts.reduce((sum, n) => sum + n, 0)
reinitializeLspServerManager()  // 重新初始化 LSP 管理器
```

#### 4. 错误合并逻辑

```typescript
setAppState(prevState => {
  // 保留现有 LSP/插件错误
  const existingLspErrors = prevState.plugins.errors.filter(
    e => e.source === 'lsp-manager' || e.source.startsWith('plugin:')
  )
  
  // 去重：移除已存在的新错误
  const newErrorKeys = new Set(errors.map(e => ...))
  const filteredExisting = existingLspErrors.filter(e => !newErrorKeys.has(key))
  
  const mergedErrors = [...filteredExisting, ...errors]
  
  return { ...prevState, plugins: { ...prevState.plugins, errors: mergedErrors } }
})
```

---

## 关键代码路径与文件引用

```
src/hooks/useManagePlugins.ts
├── Props 类型                     # 行 37-39
├── useManagePlugins()             # 行 37-304: 主钩子
│   ├── initialPluginLoad          # 行 51-266: 初始加载回调
│   │   ├── loadAllPlugins()       # 行 54
│   │   ├── detectAndUninstallDelistedPlugins() # 行 57
│   │   ├── getFlaggedPlugins()    # 行 60
│   │   ├── getPluginCommands()    # 行 76
│   │   ├── loadPluginAgents()     # 行 88
│   │   ├── loadPluginHooks()      # 行 100
│   │   ├── loadPluginMcpServers() # 行 120-128
│   │   ├── loadPluginLspServers() # 行 136-145
│   │   └── setAppState            # 行 148-180
│   ├── useEffect - 初始加载       # 行 269-285
│   └── useEffect - 刷新通知       # 行 293-303
```

### 依赖文件

```
src/utils/plugins/pluginLoader.ts
└── loadAllPlugins()               # 加载所有插件

src/utils/plugins/pluginBlocklist.ts
└── detectAndUninstallDelistedPlugins()  # 下架检测

src/utils/plugins/pluginFlagging.ts
└── getFlaggedPlugins()            # 获取标记插件

src/utils/plugins/loadPluginCommands.ts
└── getPluginCommands()            # 加载命令

src/utils/plugins/loadPluginAgents.ts
└── loadPluginAgents()             # 加载代理

src/utils/plugins/loadPluginHooks.ts
└── loadPluginHooks()              # 加载钩子

src/utils/plugins/mcpPluginIntegration.ts
└── loadPluginMcpServers()         # 加载 MCP

src/utils/plugins/lspPluginIntegration.ts
└── loadPluginLspServers()         # 加载 LSP

src/services/lsp/manager.ts
└── reinitializeLspServerManager() # LSP 管理器

src/context/notifications.ts
└── addNotification()              # 通知系统

src/services/analytics/index.ts
└── logEvent()                     # 遥测
```

---

## 依赖与外部交互

### React Hooks 使用

- `useAppState`: 获取 `needsRefresh` 状态
- `useSetAppState`: 更新插件状态
- `useNotifications`: 添加通知
- `useCallback`: 缓存 `initialPluginLoad`
- `useEffect`: 初始加载和刷新通知

### 与 AppState 的交互

```typescript
const needsRefresh = useAppState(s => s.plugins.needsRefresh)

setAppState(prevState => ({
  ...prevState,
  plugins: {
    ...prevState.plugins,
    enabled,
    disabled,
    commands,
    errors: mergedErrors
  }
}))
```

### 与通知系统的交互

```typescript
// 标记插件通知
addNotification({
  key: 'plugin-delisted-flagged',
  text: 'Plugins flagged. Check /plugins',
  color: 'warning',
  priority: 'high'
})

// 刷新通知
addNotification({
  key: 'plugin-reload-pending',
  text: 'Plugins changed. Run /reload-plugins to activate.',
  color: 'suggestion',
  priority: 'low'
})
```

### 遥测事件

```typescript
logEvent('tengu_plugins_loaded', {
  enabled_count,
  disabled_count,
  inline_count,
  marketplace_count,
  error_count,
  skill_count,
  agent_count,
  hook_count,
  mcp_count,
  lsp_count,
  has_custom_plugin_cache_dir: !!process.env.CLAUDE_CODE_PLUGIN_CACHE_DIR,
  enabled_names: ant_enabled_names  // Ant-only
})
```

---

## 风险、边界与改进建议

### 已知风险

1. **加载失败影响**
   - 单个组件加载失败不影响其他，但会记录错误
   - 严重错误时设置空状态，可能导致功能缺失

2. **缓存不一致**
   - 之前的自动刷新有缓存问题（只清除 loadAllPlugins，下游缓存未清除）
   - 缓解：改为手动 `/reload-plugins` 流程

3. **MCP 计数不准确**
   - `LoadedPlugin.mcpServers` 在 `loadAllPlugins` 后未填充
   - 缓解：单独调用 `loadPluginMcpServers` 计数

### 边界情况

| 场景 | 行为 |
|-----|------|
| enabled=false | 跳过所有加载 |
| loadAllPlugins 失败 | 设置空状态，记录错误 |
| 单个组件加载失败 | 记录错误，继续加载其他 |
| needsRefresh=true | 显示通知，不自动刷新 |
| 无插件 | 发送 0 计数遥测 |

### 改进建议

1. **并行加载优化**
   - 使用 `Promise.allSettled` 并行加载独立组件
   - 减少启动时间

2. **加载进度**
   - 显示加载进度指示器
   - 特别是插件较多时

3. **错误恢复**
   - 提供重试机制
   - 显示详细的错误信息

4. **热重载**
   - 开发模式下支持插件热重载
   - 无需重启 Claude Code

5. **依赖分析**
   - 检测插件依赖冲突
   - 自动解决或提示用户

### 测试建议

1. **单元测试**：
   - 各加载函数的错误处理
   - 错误合并逻辑

2. **集成测试**：
   - 完整加载流程
   - 与 `/reload-plugins` 的集成

3. **性能测试**：
   - 大量插件时的加载时间
   - 内存使用
