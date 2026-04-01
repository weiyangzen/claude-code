# 研究报告：src/utils/plugins/pluginOptionsStorage.ts

## 场景与职责

`pluginOptionsStorage.ts` 是 Claude Code 插件系统的**用户配置持久化与变量替换中枢**。它负责：

1. **插件用户选项的读写与分存**：插件在 `manifest.userConfig` 中声明的用户可配置字段（类型为 `McpbUserConfigurationOption`），在启用时提示用户输入值后，需要持久化。该模块将字段按 `sensitive` 标志分流：
   - `sensitive: true` → 安全存储（macOS 钥匙串 / 其他平台的 `.credentials.json`）
   - 非敏感字段 → `settings.json` 的 `pluginConfigs[pluginId].options`

2. **变量替换**：提供 `${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PLUGIN_DATA}`、`${user_config.KEY}` 三种占位符的替换能力，供 MCP/LSP 服务器命令行参数、环境变量、Hook 命令、Skill/Agent 内容等使用。

3. **配置缺失检测**：在插件启用流程（`PluginOptionsFlow`）中，判断哪些字段尚未配置或验证失败，决定是否需要弹出配置对话框。

4. **卸载清理**：在插件最后一个作用域被卸载时，清理其全部配置（`deletePluginOptions`）。

## 功能点目的

### 1. 安全与非敏感配置的分存 (`loadPluginOptions` / `savePluginOptions` / `deletePluginOptions`)
- **目的**：解决 H1 #3617646 等安全问题，防止敏感信息（如 Telegram/Discord bot token）以明文落入 `settings.json` 或 `.env` 等世界可读文件。
- **设计**：
  - 读取时合并两者，`secureStorage` 在键冲突时优先。
  - 写入时先写 `secureStorage`，失败则抛错，避免旧明文被覆盖后丢失。
  - 支持“跨存储清洗”：当某个键从敏感变为非敏感（或反之）时，自动从另一侧存储中删除旧值。

### 2. 内存缓存 (`memoize`)
- **目的**：macOS 钥匙串读取会同步 spawn `security find-generic-password`（约 50–100ms），对每工具调用都读一次不可接受。
- **设计**：`loadPluginOptions` 使用 `lodash-es/memoize`，按 `pluginId` 缓存整个会话；通过 `clearPluginOptionsCache` 在 `/reload-plugins` 或设置变更时刷新。

### 3. 变量替换 (`substitutePluginVariables` / `substituteUserConfigVariables` / `substituteUserConfigInContent`)
- **目的**：让插件作者无需硬编码绝对路径，也无需在 manifest 中明文写 secret。
- **设计**：
  - `substitutePluginVariables`：替换 `${CLAUDE_PLUGIN_ROOT}`（版本级安装目录）和 `${CLAUDE_PLUGIN_DATA}`（持久数据目录）。Windows 下将反斜杠规范化为正斜杠，避免 shell 转义问题。
  - `substituteUserConfigVariables`：严格替换 `${user_config.KEY}`，缺失则抛错，用于命令行/环境变量等必须正确填充的场景。
  - `substituteUserConfigInContent`：宽松替换，用于 Skill/Agent 文本内容；敏感键替换为占位符，防止 secret 进入模型上下文。

### 4. 未配置选项检测 (`getUnconfiguredOptions`)
- **目的**：在插件启用后，自动判断是否需要弹出用户配置对话框。
- **设计**：加载已保存值，用 `validateUserConfig` 校验；仅返回校验失败的字段子集。

## 具体技术实现

### 关键数据结构

```ts
export type PluginOptionValues = UserConfigValues  // Record<string, string | number | boolean | string[]>
export type PluginOptionSchema = UserConfigSchema  // Record<string, McpbUserConfigurationOption>
```

存储键格式：
- 插件级选项：`plugin.source`（即 `"${name}@${marketplace}"`）
- MCP 服务器级选项（在 `mcpbHandler.ts` 中）：`${pluginId}/${serverName}`

### 关键流程

#### `savePluginOptions` 写入流程
1. 按 `schema[key].sensitive` 拆分 `nonSensitive` 和 `sensitive`。
2. **先写 secureStorage**：
   - 读取现有 `pluginSecrets[pluginId]`。
   - 过滤掉本次要写入的非敏感键（清洗）。
   - 调用 `storage.update()` 写入；失败则 `logError` + `throw`。
3. **后写 settings.json**：
   - 读取现有 `pluginConfigs[pluginId].options`。
   - 对本次要写入的敏感键，显式设为 `undefined`（利用 `updateSettingsForSource` 的 `mergeWith` 定制器实现删除）。
   - 调用 `updateSettingsForSource('userSettings', settings)`；失败则 `logError` + `throw`。
4. 调用 `clearPluginOptionsCache()` 使下次读取生效。

#### `deletePluginOptions` 删除流程
1. **settings 侧**：构造 `{ [pluginId]: undefined }` 的 `Partial<PluginConfigs>`，通过 `updateSettingsForSource` 删除整条记录；同时清理遗留的 `mcpServers` 子键。
2. **secureStorage 侧**：读取 `pluginSecrets`，过滤掉 `pluginId` 及所有 `${pluginId}/` 前缀的复合键；写回剩余条目或 `undefined`。
3. 两个操作均为 best-effort（失败只 warn 不 throw），因为卸载本身已成功，不希望清理副作用被用户感知为“卸载失败”。

### 文件引用

- `getSecureStorage()` → `src/utils/secureStorage/index.ts`
- `validateUserConfig()` → `src/utils/plugins/mcpbHandler.ts`
- `getPluginDataDir()` → `src/utils/plugins/pluginDirectories.ts`
- `getSettings_DEPRECATED()` / `updateSettingsForSource()` → `src/utils/settings/settings.ts`

## 依赖与外部交互

### 上游调用方
- `src/services/plugins/pluginOperations.ts`：卸载时调用 `deletePluginOptions`。
- `src/commands/plugin/PluginOptionsFlow.tsx`、`PluginOptionsDialog.tsx`、`ManagePlugins.tsx`：配置对话框读写选项。
- `src/utils/plugins/mcpPluginIntegration.ts`、`lspPluginIntegration.ts`、`loadPluginCommands.ts`、`loadPluginAgents.ts`：变量替换与选项加载。
- `src/utils/plugins/marketplaceManager.ts`：卸载时清理选项。
- `src/utils/plugins/cacheUtils.ts`：缓存清除时联动清除选项缓存。

### 下游依赖
- `../secureStorage/index.ts`：平台相关的安全存储抽象（macOS 钥匙串 / 回退到明文文件）。
- `../settings/settings.ts`：非敏感配置的读写。
- `./mcpbHandler.ts`：配置校验逻辑（`validateUserConfig`）。
- `./pluginDirectories.ts`：`getPluginDataDir` 用于 `${CLAUDE_PLUGIN_DATA}` 替换。

## 风险、边界与改进建议

### 已知风险

1. **项目级配置泄漏到用户设置（TODO 注释）**
   - `getSettings_DEPRECATED` 返回的是**合并后**的设置（跨所有作用域）。在 `savePluginOptions` 中直接修改并写回 `userSettings`，可能导致项目级 `pluginConfigs` 被意外固化到 `~/.claude/settings.json`。
   - 当前安全是因为 `pluginConfigs` 只在用户作用域写入，但一旦支持项目级插件选项，就会触发此问题。

2. **macOS 钥匙串同步阻塞**
   - `storage.read()` 在 macOS 上会同步 spawn keychain 进程。虽然 `memoize` 缓解了同一会话的重复调用，但**首次调用**仍会阻塞事件循环 50–100ms。在热路径（如每工具调用）中已通过 memoize 避免，但在 `/reload-plugins` 后的首次调用仍无法避免。

3. **敏感键在 Skill 内容中的泄露风险**
   - `substituteUserConfigInContent` 已做防护：敏感键替换为 `[sensitive option 'key' not available in skill content]`。但插件作者若将敏感键用于非 Skill 的文本字段（如命令描述），仍可能间接泄露。

4. **卸载清理的时序问题**
   - `deletePluginOptions` 只在“最后一个安装作用域”卸载时调用。若插件在多个作用域安装，从一个作用域卸载不会清理配置，这是设计意图；但调用方（`pluginOperations.ts`）必须正确判断“是否为最后作用域”，否则可能过早丢失用户配置。

### 边界情况

- **空 schema**：`getUnconfiguredOptions` 对空 `manifest.userConfig` 直接返回 `{}`，不触发配置对话框。
- **缺失 `plugin.source`**：`substitutePluginVariables` 中若 `plugin.source` 缺失，则 `${CLAUDE_PLUGIN_DATA}` 保持原样不替换（用于无插件上下文的 hook/skill root）。
- **Windows 路径规范化**：`substitutePluginVariables` 将反斜杠替换为正斜杠，防止 shell 把 `\` 当成转义序列。

### 改进建议

1. **拆分 `getSettings_DEPRECATED` 的使用**
   - 在 `savePluginOptions` 和 `deletePluginOptions` 中，应使用 `getSettingsForSource('userSettings')` 而非合并视图，彻底消除项目级配置泄漏风险。

2. **异步化安全存储读取**
   - 若 `getSecureStorage` 未来支持异步 API，可将 `loadPluginOptions` 改为 async，避免 macOS 首次调用阻塞事件循环。

3. **统一 composite key 常量**
   - MCP 服务器级选项使用 `${pluginId}/${serverName}` 作为 composite key，该逻辑分散在 `mcpbHandler.ts`（`serverSecretsKey`）和本文件（`deletePluginOptions` 的 prefix 过滤）。建议将 composite key 构造/解析逻辑提取到共享工具函数，降低漂移风险。

4. **增强 `deletePluginOptions` 的 settings 侧 scrub**
   - 当前仅删除 `pluginConfigs[pluginId]` 顶层对象，但未递归删除 `pluginConfigs[pluginId].mcpServers` 子树（该子树由 `mcpbHandler.ts` 管理）。虽然顶层删除后 `updateSettingsForSource` 会移除整个键，但注释中提到“legacy mcpServers sub-key”说明历史上存在分离存储。应确保两者一致清理。
