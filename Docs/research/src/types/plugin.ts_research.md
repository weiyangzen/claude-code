# plugin.ts 研究文档

## 场景与职责

`src/types/plugin.ts` 是 Claude Code CLI 的插件系统核心类型定义文件。插件系统允许用户从 marketplace 或本地目录安装扩展，添加自定义命令、agents、hooks、MCP 服务器和 LSP 服务器。核心职责：

1. **插件定义类型**: 定义内置插件、已加载插件、插件配置的结构
2. **插件组件类型**: 定义插件可提供的组件类型（commands, agents, skills, hooks, output-styles）
3. **插件错误类型**: 定义详细的类型安全错误联合类型，支持精确的错误处理
4. **错误消息生成**: 提供 `getPluginErrorMessage()` 函数生成用户友好的错误消息

该文件是插件系统的类型基础，被 20+ 文件依赖，支持 `/plugin` 命令和插件管理 UI。

## 功能点目的

### 1. 内置插件定义（BuiltinPluginDefinition）
内置插件是随 CLI 一起发布的插件：
- 通过 `{name}@builtin` 标识符引用
- 可在 `/plugin` UI 中启用/禁用
- 支持 skills、hooks、MCP 服务器
- 可通过 `isAvailable()` 函数控制平台可用性
- 支持默认启用状态配置

### 2. 插件仓库配置（PluginRepository）
支持从 Git 仓库加载插件：
- URL 和分支配置
- 可选的 lastUpdated 和 commitSha 用于版本追踪

### 3. 已加载插件（LoadedPlugin）
内存中的插件表示，包含：
- 基础信息：name, manifest, path, source, repository
- 状态：enabled, isBuiltin, sha
- 组件路径：commandsPath(s), agentsPath(s), skillsPath(s), outputStylesPath(s)
- 配置：hooksConfig, mcpServers, lspServers, settings

### 4. 插件组件类型（PluginComponent）
插件可提供的五种组件：
- `commands`: 自定义斜杠命令
- `agents`: 自定义 AI agents
- `skills`: 技能定义
- `hooks`: 生命周期钩子
- `output-styles`: 输出样式

### 5. 类型安全错误系统（PluginError）
25+ 种详细错误类型，每种都有特定的上下文数据：
- **Git 相关**: git-auth-failed, git-timeout
- **网络相关**: network-error, mcpb-download-failed
- **Manifest 相关**: manifest-parse-error, manifest-validation-error
- **Marketplace 相关**: marketplace-not-found, marketplace-load-failed, marketplace-blocked-by-policy
- **MCP/LSP 相关**: mcp-config-invalid, mcp-server-suppressed-duplicate, lsp-config-invalid, lsp-server-start-failed, lsp-server-crashed
- **依赖相关**: dependency-unsatisfied
- **其他**: path-not-found, hook-load-failed, component-load-failed, generic-error

### 6. 插件加载结果（PluginLoadResult）
分类返回加载结果：
- `enabled`: 成功启用的插件
- `disabled`: 被禁用的插件（但加载成功）
- `errors`: 加载过程中遇到的错误

## 具体技术实现

### 关键数据结构

```typescript
// 内置插件定义
interface BuiltinPluginDefinition {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean    // 平台可用性检查
  defaultEnabled?: boolean       // 默认启用状态
}

// 插件仓库配置
interface PluginRepository {
  url: string
  branch: string
  lastUpdated?: string
  commitSha?: string
}

// 插件配置（用户设置）
interface PluginConfig {
  repositories: Record<string, PluginRepository>
}

// 已加载插件（内存表示）
interface LoadedPlugin {
  name: string
  manifest: PluginManifest
  path: string
  source: string
  repository: string
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string                    // Git commit SHA
  commandsPath?: string
  commandsPaths?: string[]        // Manifest 中的额外路径
  commandsMetadata?: Record<string, CommandMetadata>
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

### 插件错误类型系统

```typescript
type PluginError =
  | { type: 'path-not-found'; source: string; plugin?: string; path: string; component: PluginComponent }
  | { type: 'git-auth-failed'; source: string; plugin?: string; gitUrl: string; authType: 'ssh' | 'https' }
  | { type: 'git-timeout'; source: string; plugin?: string; gitUrl: string; operation: 'clone' | 'pull' }
  | { type: 'network-error'; source: string; plugin?: string; url: string; details?: string }
  | { type: 'manifest-parse-error'; source: string; plugin?: string; manifestPath: string; parseError: string }
  | { type: 'manifest-validation-error'; source: string; plugin?: string; manifestPath: string; validationErrors: string[] }
  | { type: 'plugin-not-found'; source: string; pluginId: string; marketplace: string }
  | { type: 'marketplace-not-found'; source: string; marketplace: string; availableMarketplaces: string[] }
  | { type: 'marketplace-load-failed'; source: string; marketplace: string; reason: string }
  | { type: 'mcp-config-invalid'; source: string; plugin: string; serverName: string; validationError: string }
  | { type: 'mcp-server-suppressed-duplicate'; source: string; plugin: string; serverName: string; duplicateOf: string }
  | { type: 'lsp-config-invalid'; source: string; plugin: string; serverName: string; validationError: string }
  | { type: 'lsp-server-start-failed'; source: string; plugin: string; serverName: string; reason: string }
  | { type: 'lsp-server-crashed'; source: string; plugin: string; serverName: string; exitCode: number | null; signal?: string }
  | { type: 'lsp-request-timeout'; source: string; plugin: string; serverName: string; method: string; timeoutMs: number }
  | { type: 'lsp-request-failed'; source: string; plugin: string; serverName: string; method: string; error: string }
  | { type: 'marketplace-blocked-by-policy'; source: string; plugin?: string; marketplace: string; blockedByBlocklist?: boolean; allowedSources: string[] }
  | { type: 'dependency-unsatisfied'; source: string; plugin: string; dependency: string; reason: 'not-enabled' | 'not-found' }
  | { type: 'plugin-cache-miss'; source: string; plugin: string; installPath: string }
  | { type: 'generic-error'; source: string; plugin?: string; error: string }
```

### 错误消息生成

```typescript
export function getPluginErrorMessage(error: PluginError): string {
  switch (error.type) {
    case 'generic-error':
      return error.error
    case 'path-not-found':
      return `Path not found: ${error.path} (${error.component})`
    case 'git-auth-failed':
      return `Git authentication failed (${error.authType}): ${error.gitUrl}`
    case 'git-timeout':
      return `Git ${error.operation} timeout: ${error.gitUrl}`
    case 'network-error':
      return `Network error: ${error.url}${error.details ? ` - ${error.details}` : ''}`
    case 'manifest-parse-error':
      return `Manifest parse error: ${error.parseError}`
    case 'manifest-validation-error':
      return `Manifest validation failed: ${error.validationErrors.join(', ')}`
    case 'plugin-not-found':
      return `Plugin ${error.pluginId} not found in marketplace ${error.marketplace}`
    // ... 更多 case
    case 'mcp-server-suppressed-duplicate': {
      const dup = error.duplicateOf.startsWith('plugin:')
        ? `server provided by plugin "${error.duplicateOf.split(':')[1] ?? '?'}"`
        : `already-configured "${error.duplicateOf}"`
      return `MCP server "${error.serverName}" skipped — same command/URL as ${dup}`
    }
    // ... 更多 case
  }
}
```

## 关键代码路径与文件引用

### 类型定义
- `src/types/plugin.ts` - 本文件，核心类型定义
- `src/utils/plugins/schemas.ts` - PluginManifest, PluginAuthor, CommandMetadata 等 Schema

### 插件加载
- `src/utils/plugins/pluginLoader.ts` - 核心加载逻辑
- `src/utils/plugins/loadPluginCommands.ts` - 加载插件命令
- `src/utils/plugins/loadPluginAgents.ts` - 加载插件 agents
- `src/utils/plugins/loadPluginHooks.ts` - 加载插件 hooks
- `src/utils/plugins/loadPluginOutputStyles.ts` - 加载输出样式

### 插件管理
- `src/utils/plugins/dependencyResolver.ts` - 依赖解析
- `src/utils/plugins/pluginOptionsStorage.ts` - 选项存储
- `src/utils/plugins/refresh.ts` - 插件刷新
- `src/utils/plugins/marketplaceManager.ts` - Marketplace 管理

### MCP/LSP 集成
- `src/utils/plugins/mcpPluginIntegration.ts` - MCP 集成
- `src/utils/plugins/lspPluginIntegration.ts` - LSP 集成

### 服务层
- `src/services/plugins/pluginOperations.ts` - 插件操作
- `src/services/mcp/config.ts` - MCP 配置
- `src/services/lsp/config.ts` - LSP 配置

### UI 组件
- `src/commands/plugin/plugin.tsx` - /plugin 命令主界面
- `src/commands/plugin/ManagePlugins.tsx` - 插件管理
- `src/commands/plugin/ManageMarketplaces.tsx` - Marketplace 管理
- `src/commands/plugin/DiscoverPlugins.tsx` - 插件发现
- `src/commands/plugin/BrowseMarketplace.tsx` - Marketplace 浏览
- `src/commands/plugin/PluginSettings.tsx` - 插件设置
- `src/commands/plugin/PluginOptionsFlow.tsx` - 选项流程
- `src/commands/plugin/PluginErrors.tsx` - 错误展示

### 内置插件
- `src/plugins/builtinPlugins.ts` - 内置插件定义

### 状态管理
- `src/state/AppStateStore.ts` - 应用状态存储

### CLI 处理
- `src/cli/handlers/plugins.ts` - CLI 插件处理

### 遥测
- `src/utils/telemetry/pluginTelemetry.ts` - 插件遥测

### 诊断
- `src/screens/Doctor.tsx` - 诊断界面

## 依赖与外部交互

### 导入依赖
```typescript
import type { LspServerConfig } from '../services/lsp/types.js'           // LSP 配置
import type { McpServerConfig } from '../services/mcp/types.js'           // MCP 配置
import type { BundledSkillDefinition } from '../skills/bundledSkills.js'  // Bundled Skill
import type { CommandMetadata, PluginAuthor, PluginManifest } from '../utils/plugins/schemas.js'  // Schema 类型
import type { HooksSettings } from '../utils/settings/types.js'           // Hooks 配置
```

### 被依赖方（20+ 文件）
主要分布：
- 插件加载系统（`src/utils/plugins/*`）
- 插件 UI（`src/commands/plugin/*`）
- 服务层（`src/services/*`）
- 内置插件（`src/plugins/builtinPlugins.ts`）

## 风险、边界与改进建议

### 潜在风险

1. **LoadedPlugin 的路径字段冗余**
   - `commandsPath` 和 `commandsPaths` 同时存在
   - 前者是主路径，后者是 manifest 中的额外路径
   - 容易混淆，且需要合并处理

2. **PluginError 的 exhaustive 检查**
   - 25+ 种错误类型，`getPluginErrorMessage` 需要处理所有 case
   - 新增错误类型时容易遗漏 switch case
   - TypeScript 会检查 exhaustive，但运行时回退到 implicit undefined

3. **MCP 服务器重复检测**
   - `mcp-server-suppressed-duplicate` 需要比较 command/URL
   - 比较逻辑分散在集成层，容易不一致

4. **依赖解析顺序**
   - `dependency-unsatisfied` 有两个原因：not-enabled 和 not-found
   - 解析顺序影响错误报告（应先检查存在性再检查启用状态）

### 边界情况

1. **内置插件的特殊处理**
   - `isBuiltin: true` 的插件在 UI 中特殊展示
   - 内置插件不可卸载，只能禁用
   - 通过 `isAvailable()` 控制平台可用性

2. **PluginRepository 的 lastUpdated**
   - 用于缓存失效判断
   - 但依赖客户端时钟，可能受时区影响

3. **LSP 服务器崩溃处理**
   - `lsp-server-crashed` 包含 exitCode 和 signal
   - 需要区分正常退出和异常崩溃
   - signal 存在时通常表示被信号终止（如 SIGKILL）

4. **Marketplace 策略阻止**
   - `marketplace-blocked-by-policy` 有两个原因：
     - `blockedByBlocklist: true`: 在阻止列表中
     - `blockedByBlocklist: false`: 不在允许的已知列表中

### 改进建议

1. **LoadedPlugin 路径字段重构**
   ```typescript
   // 建议：统一路径表示
   interface LoadedPlugin {
     paths: {
       commands?: { primary: string; additional?: string[] }
       agents?: { primary: string; additional?: string[] }
       skills?: { primary: string; additional?: string[] }
       outputStyles?: { primary: string; additional?: string[] }
     }
   }
   ```

2. **PluginError 消息国际化**
   ```typescript
   // 建议：增加错误码和参数化消息
   interface PluginError {
     type: 'plugin-not-found'
     errorCode: 'PLUGIN_NOT_FOUND'
     params: { pluginId: string; marketplace: string }
   }
   // 消息模板："Plugin {pluginId} not found in marketplace {marketplace}"
   ```

3. **错误类型分组**
   ```typescript
   // 建议：按领域分组
   type GitPluginError = GitAuthFailed | GitTimeout | ...
   type NetworkPluginError = NetworkError | McpbDownloadFailed | ...
   type ManifestPluginError = ManifestParseError | ManifestValidationError | ...
   ```

4. **LoadedPlugin 验证**
   - 当前无运行时验证
   - 建议增加 Zod Schema 验证加载后的插件对象

5. **内置插件版本管理**
   - 当前 `version?: string` 是可选的
   - 建议内置插件必须提供版本，用于更新检查

6. **错误恢复建议**
   - `getPluginErrorMessage` 仅返回描述
   - 建议增加 `getPluginErrorRecovery()` 返回恢复建议
