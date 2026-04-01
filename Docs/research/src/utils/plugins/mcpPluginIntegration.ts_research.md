# mcpPluginIntegration.ts 深度研究文档

## 场景与职责

`mcpPluginIntegration.ts` 是 Claude Code 插件系统中负责 **MCP (Model Context Protocol) 服务器集成** 的核心模块。它桥接插件系统与 MCP 服务层，使插件能够通过声明式配置提供 MCP 服务器功能。

### 核心职责

1. **插件 MCP 服务器加载**：从插件的 manifest、`.mcp.json` 文件或 `.mcpb` 包中加载 MCP 服务器配置
2. **环境变量解析**：处理插件特定的变量替换（`${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PLUGIN_DATA}`、`${user_config.X}`）
3. **配置作用域管理**：为插件提供的 MCP 服务器添加 `plugin:` 前缀作用域，避免命名冲突
4. **用户配置集成**：支持插件声明 `userConfig`，在启用时提示用户输入配置值
5. **通道配置管理**：处理插件声明的 `channels`（如 Telegram、Slack 等消息通道）的配置

### 在系统架构中的位置

```
插件层 (Plugin Layer)
    ├── pluginLoader.ts          # 加载插件元数据
    ├── mcpPluginIntegration.ts  # ← 本文件：MCP 配置集成
    └── mcpbHandler.ts           # MCPB 包处理

服务层 (Service Layer)
    └── services/mcp/config.ts   # MCP 配置管理
        └── getPluginMcpServers() 调用本模块
```

---

## 功能点目的

### 1. 多源 MCP 配置加载 (`loadPluginMcpServers`)

支持三种配置来源，按优先级合并：

| 来源 | 优先级 | 说明 |
|------|--------|------|
| `.mcp.json` | 最低 | 插件目录下的默认配置文件 |
| `manifest.mcpServers` | 高 | 插件 manifest 中声明的配置 |
| 内联配置 | 最高 | manifest 中的直接配置对象 |

### 2. MCPB 包支持 (`loadMcpServersFromMcpb`)

支持从 `.mcpb` 或 `.dxt` 格式的 MCP Bundle 文件加载服务器：
- 支持本地文件路径和远程 URL
- 自动下载、缓存、解压
- 处理用户配置需求（`needs-config` 状态）

### 3. 变量替换系统 (`resolvePluginMcpEnvironment`)

三层变量替换，按优先级从高到低：

```
1. 插件变量: ${CLAUDE_PLUGIN_ROOT}, ${CLAUDE_PLUGIN_DATA}
2. 用户配置: ${user_config.KEY}
3. 环境变量: ${ENV_VAR}
```

### 4. 配置作用域隔离 (`addPluginScopeToServers`)

为每个插件的 MCP 服务器添加前缀，避免冲突：
- 原始服务器名：`my-server`
- 作用域化后：`plugin:plugin-name:my-server`

### 5. 未配置通道检测 (`getUnconfiguredChannels`)

检测插件 manifest 中声明的 `channels` 是否有未完成的用户配置，用于在插件启用后弹出配置对话框。

---

## 具体技术实现

### 关键数据结构

```typescript
// 未配置通道（用于 UI 提示）
export type UnconfiguredChannel = {
  server: string           // MCP 服务器名称
  displayName: string      // 显示名称
  configSchema: UserConfigSchema  // 配置项 schema
}

// 插件 MCP 服务器加载结果
plugin.mcpServers?: Record<string, McpServerConfig>
```

### 核心流程

#### 1. 提取所有插件的 MCP 服务器 (`extractMcpServersFromPlugins`)

```
遍历所有已启用插件
    ├── 加载插件 MCP 配置 (loadPluginMcpServers)
    │   ├── 尝试 .mcp.json
    │   ├── 尝试 manifest.mcpServers (支持字符串路径、数组、内联对象)
    │   └── 如果是 MCPB，调用 mcpbHandler 处理
    ├── 解析环境变量 (resolvePluginMcpEnvironment)
    │   ├── 替换 ${CLAUDE_PLUGIN_ROOT} → 插件安装路径
    │   ├── 替换 ${CLAUDE_PLUGIN_DATA} → 插件数据目录
    │   ├── 替换 ${user_config.X} → 用户配置值
    │   └── 展开一般环境变量
    └── 添加作用域前缀 (addPluginScopeToServers)
        └── 格式: plugin:{pluginName}:{serverName}
```

#### 2. 环境变量解析详解 (`resolvePluginMcpEnvironment`)

```typescript
// 处理不同类型的 MCP 服务器
switch (config.type) {
  case 'stdio':
    // 解析 command, args, env
    // 注入 CLAUDE_PLUGIN_ROOT 和 CLAUDE_PLUGIN_DATA
  case 'sse':
  case 'http':
  case 'ws':
    // 解析 URL 和 headers
  case 'sse-ide':
  case 'ws-ide':
  case 'sdk':
  case 'claudeai-proxy':
    // 透传，不做修改
}
```

#### 3. 用户配置构建 (`buildMcpUserConfig`)

合并两级用户配置：
- **顶级配置**：`manifest.userConfig` → 存储于 `pluginConfigs[pluginId].options`
- **通道特定配置**：`channels[].userConfig` → 存储于 `pluginConfigs[pluginId].mcpServers[serverName]`

通道特定配置优先级更高。

### 错误处理

定义了多种 `PluginError` 子类型：

| 错误类型 | 场景 |
|----------|------|
| `mcpb-download-failed` | MCPB 文件下载失败 |
| `mcpb-extract-failed` | MCPB 解压失败 |
| `mcpb-invalid-manifest` | Manifest 解析或验证失败 |
| `mcp-config-invalid` | 环境变量缺失或配置无效 |
| `generic-error` | 其他未分类错误 |

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `loadPluginMcpServers` | 加载单个插件的 MCP 配置 | `extractMcpServersFromPlugins`, `getPluginMcpServers` |
| `extractMcpServersFromPlugins` | 批量提取所有插件的 MCP 服务器 | `services/mcp/config.ts` |
| `getPluginMcpServers` | 获取特定插件的 MCP 服务器（带缓存） | `services/mcp/config.ts` |
| `getUnconfiguredChannels` | 获取未配置的通道列表 | `ManagePlugins.tsx` |
| `resolvePluginMcpEnvironment` | 解析 MCP 配置中的环境变量 | 内部使用，也可用于测试 |
| `addPluginScopeToServers` | 为服务器名添加插件作用域 | 内部使用 |

### 调用关系图

```
services/mcp/config.ts
    ├── getAllMcpConfigs()
    │   └── extractMcpServersFromPlugins()  ← 入口
    │       ├── loadPluginMcpServers()
    │       │   ├── loadMcpServersFromFile()  ← .mcp.json
    │       │   └── loadMcpServersFromMcpb()  ← .mcpb/.dxt
    │       │       └── mcpbHandler.loadMcpbFile()
    │       └── resolvePluginMcpEnvironment()
    │           ├── substitutePluginVariables()
    │           │   └── pluginOptionsStorage.ts
    │           ├── substituteUserConfigVariables()
    │           │   └── pluginOptionsStorage.ts
    │           └── expandEnvVarsInString()
    │               └── services/mcp/envExpansion.ts
    └── getPluginMcpServers()  ← 单个插件获取
        └── loadPluginMcpServers()

commands/plugin/ManagePlugins.tsx
    └── getUnconfiguredChannels()  ← 检测未配置通道

hooks/useManagePlugins.ts
    └── extractMcpServersFromPlugins()

cli/handlers/plugins.ts
    └── extractMcpServersFromPlugins()
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `mcpbHandler.ts` | MCPB 文件加载、用户配置读写/验证 |
| `pluginOptionsStorage.ts` | 插件选项存储、变量替换辅助函数 |
| `pluginDirectories.ts` | 获取插件数据目录 (`getPluginDataDir`) |
| `services/mcp/types.ts` | `McpServerConfig`, `ScopedMcpServerConfig` 类型 |
| `services/mcp/envExpansion.ts` | 通用环境变量展开 |
| `types/plugin.ts` | `LoadedPlugin`, `PluginError` 类型 |

### 外部包依赖

| 包 | 用途 |
|----|------|
| `@anthropic-ai/mcpb` | MCPB manifest 类型定义（通过 mcpbHandler 间接使用） |

### 配置存储位置

```
~/.claude/settings.json
    └── pluginConfigs
        └── {pluginId}              # 格式: "pluginName@marketplace"
            ├── options             # 非敏感用户配置
            └── mcpServers
                └── {serverName}    # 通道特定配置

Keychain / .credentials.json
    └── pluginSecrets
        └── {pluginId}              # 敏感配置值
        └── {pluginId}/{serverName} # 通道特定敏感配置
```

---

## 风险、边界与改进建议

### 已知风险

1. **环境变量解析失败导致服务器启动失败**
   - 风险：如果 `${user_config.X}` 引用的配置不存在，`substituteUserConfigVariables` 会抛出错误
   - 缓解：已在 `extractMcpServersFromPlugins` 和 `getPluginMcpServers` 中添加 per-server try/catch

2. **敏感信息泄露**
   - 风险：用户配置可能包含敏感信息（API 密钥等）
   - 缓解：`user_config` 支持 `sensitive: true` 标记，会存储到 keychain 而非 settings.json

3. **MCPB 缓存不一致**
   - 风险：本地 MCPB 文件修改后，缓存可能未失效
   - 缓解：`checkMcpbChanged` 会检查 mtime，URL 来源的 MCPB 需要显式更新

4. **服务器名称冲突**
   - 风险：不同插件可能声明同名 MCP 服务器
   - 缓解：通过 `addPluginScopeToServers` 添加 `plugin:{name}:` 前缀

### 边界情况

| 场景 | 行为 |
|------|------|
| 插件禁用 | 返回 `undefined`，不加载 MCP 服务器 |
| MCPB 需要配置 | 返回 `null`（非错误），用户可通过 `/plugin` 菜单配置 |
| 环境变量缺失 | 记录警告，添加 `mcp-config-invalid` 错误 |
| 配置验证失败 | 单服务器失败不影响其他服务器加载 |
| 无 userConfig 声明 | 跳过 `loadMcpServerUserConfig` 调用，避免不必要的 keychain 读取 (~50-100ms) |

### 改进建议

1. **缓存优化**
   - 当前：`plugin.mcpServers` 缓存的是未解析环境变量的配置
   - 建议：考虑添加解析后配置的缓存，避免每次重复解析

2. **错误粒度**
   - 当前：部分错误使用 `generic-error` 类型
   - 建议：细化错误类型，提供更多上下文（如具体缺失的环境变量名）

3. **配置热重载**
   - 当前：用户配置变更后需要重新加载插件
   - 建议：支持 MCP 服务器配置的热更新，无需重启

4. **敏感配置审计**
   - 建议：添加敏感配置访问日志，便于安全审计

5. **MCPB 版本管理**
   - 当前：MCPB 缓存基于内容 hash
   - 建议：支持显式版本声明，便于回滚和管理

---

## 测试要点

1. **多源配置合并**：验证 `.mcp.json`、manifest、内联配置的优先级
2. **变量替换**：验证三层变量替换的正确性和优先级
3. **错误隔离**：验证单个服务器配置失败不影响其他服务器
4. **作用域隔离**：验证同名服务器在不同插件中不会冲突
5. **用户配置流程**：验证 `needs-config` 状态的正确检测和处理
