# pluginOperations.ts 研究文档

## 场景与职责

`pluginOperations.ts` 是 Claude Code 插件系统的**核心操作库**，提供纯函数式的插件生命周期管理功能。它是整个插件系统的"业务逻辑层"，被 CLI 命令和交互式 UI 共同依赖。

### 核心定位
- **纯库函数**：无副作用（不调用 `process.exit()`，不写 console）
- **双入口支持**：同时服务 CLI 命令和交互式 UI（ManagePlugins.tsx）
- **Settings-First 架构**：以 settings 为意图层，以 installed_plugins.json 为状态层

### 提供的核心操作

| 操作 | 函数 | 描述 |
|------|------|------|
| 安装 | `installPluginOp()` | 从市场安装插件，支持依赖解析 |
| 卸载 | `uninstallPluginOp()` | 卸载插件，清理数据，检查反向依赖 |
| 启用 | `enablePluginOp()` | 启用插件（settings-first） |
| 禁用 | `disablePluginOp()` | 禁用插件，警告反向依赖 |
| 禁用全部 | `disableAllPluginsOp()` | 批量禁用所有插件 |
| 更新 | `updatePluginOp()` | 非原地更新到新版 |

---

## 功能点目的

### 1. Settings-First 架构

**核心理念**：settings 声明意图，系统负责 materialize。

**传统方式 vs Settings-First**：
```
传统：检查是否安装 → 安装 → 修改 settings
Settings-First：写 settings（声明意图）→ 系统加载时 materialize
```

**优势**：
- 配置即代码：settings.json 是单一事实来源
- 跨设备同步：settings 可以同步，安装状态自动跟随
- 回滚简单：撤销 settings 修改即可

### 2. 依赖解析与安装 (`installPluginOp`)

**目的**：支持插件依赖链的自动安装。

**流程**：
```
1. 在市场查找插件
2. 检查组织策略（是否被 block）
3. 解析传递闭包（transitive closure）
4. 检查跨市场依赖白名单
5. 写 settings（整个闭包一次性写入）
6. 缓存每个依赖到本地
```

**依赖解析结果类型**：
```typescript
type ResolutionResult =
  | { ok: true; closure: string[] }                    // 成功，返回闭包
  | { ok: false; reason: 'cycle'; chain: string[] }    // 循环依赖
  | { ok: false; reason: 'not-found'; missing: string; requiredBy: string }
  | { ok: false; reason: 'cross-marketplace'; dependency: string; requiredBy: string }
```

### 3. 智能 Scope 解析

**目的**：支持多 scope（user/project/local）的插件管理。

**Scope 优先级**（最具体优先）：
```
local > project > user > managed
```

**自动检测逻辑**（`findPluginInSettings`）：
```typescript
const searchOrder: InstallableScope[] = ['local', 'project', 'user']
for (const scope of searchOrder) {
  if (pluginInScope(plugin, scope)) return { pluginId, scope }
}
```

### 4. 卸载清理 (`uninstallPluginOp`)

**目的**：彻底清理插件，包括数据目录和配置。

**清理内容**：
- 从 settings 移除启用记录
- 从 installed_plugins_v2.json 移除安装记录
- 标记版本为孤儿（orphaned）以便后续清理
- 删除插件选项配置（pluginConfigs）
- 删除插件数据目录（可选）

**反向依赖检查**：
```typescript
const reverseDependents = findReverseDependents(pluginId, allPlugins)
// 警告但不阻止：避免 delisted 插件导致无法卸载
```

### 5. 非原地更新 (`updatePluginOp`)

**目的**：安全更新插件，保留旧版本直到重启。

**更新流程**：
```
1. 获取市场最新版本信息
2. 下载/计算新版本
3. 如果版本相同 → 返回 alreadyUpToDate
4. 复制到新版本化目录
5. 更新 installed_plugins.json 中的路径
6. 标记旧版本为孤儿（如无其他引用）
7. 提示用户重启应用
```

**关键设计**：内存中的插件保持不变，磁盘更新，重启后生效。

### 6. 内置插件支持

**目的**：特殊处理内置插件（如 `@filesystem`、`@git`）。

**特殊逻辑**：
```typescript
if (isBuiltinPluginId(plugin)) {
  // 内置插件不使用 installed_plugins.json
  // 直接写入 user-scope settings
  // 跳过缓存和 materialization
}
```

---

## 具体技术实现

### 类型系统

```typescript
// 可安装 scope（不含 managed）
export const VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local'] as const
export type InstallableScope = (typeof VALID_INSTALLABLE_SCOPES)[number]

// 可更新 scope（包含 managed）
export const VALID_UPDATE_SCOPES: readonly PluginScope[] = [
  'user', 'project', 'local', 'managed'
]

// 操作结果
export type PluginOperationResult = {
  success: boolean
  message: string
  pluginId?: string
  pluginName?: string
  scope?: PluginScope
  reverseDependents?: string[]  // 反向依赖警告
}

// 更新结果
export type PluginUpdateResult = {
  success: boolean
  message: string
  pluginId?: string
  newVersion?: string
  oldVersion?: string
  alreadyUpToDate?: boolean
  scope?: PluginScope
}
```

### 核心函数实现

#### `installPluginOp` (行 321-418)

```typescript
export async function installPluginOp(
  plugin: string,
  scope: InstallableScope = 'user'
): Promise<PluginOperationResult> {
  // 1. 验证 scope
  assertInstallableScope(scope)

  // 2. 解析插件标识符
  const { name: pluginName, marketplace: marketplaceName } = parsePluginIdentifier(plugin)

  // 3. 在市场查找插件
  let foundPlugin: PluginMarketplaceEntry | undefined
  let foundMarketplace: string | undefined
  // ... 查找逻辑（支持指定市场或全市场搜索）

  // 4. 调用核心安装逻辑
  const result = await installResolvedPlugin({
    pluginId,
    entry,
    scope,
    marketplaceInstallLocation,
  })

  // 5. 处理各种失败情况
  switch (result.reason) {
    case 'local-source-no-location': ...
    case 'settings-write-failed': ...
    case 'resolution-failed': ...
    case 'blocked-by-policy': ...
    case 'dependency-blocked-by-policy': ...
  }
}
```

#### `setPluginEnabledOp` (行 573-747)

**状态检查逻辑**（处理跨 scope 覆盖）：
```typescript
// 检查是否是覆盖操作（高优先级 scope 覆盖低优先级）
const isOverride = scope && SCOPE_PRECEDENCE[scope] > SCOPE_PRECEDENCE[found.scope]

// 确定当前状态
const isCurrentlyEnabled =
  scope && !isOverride
    ? scopeSettingsValue === true  // 显式 scope：检查该 scope 的设置
    : getPluginEditableScopes().has(pluginId)  // 自动检测：检查合并后的状态
```

**禁用时的反向依赖捕获**：
```typescript
// 在写 settings 前捕获（因为写完后缓存会被清除）
if (!enabled) {
  const { enabled: loadedEnabled, disabled } = await loadAllPlugins()
  const rdeps = findReverseDependents(pluginId, [...loadedEnabled, ...disabled])
  if (rdeps.length > 0) reverseDependents = rdeps
}
```

#### `updatePluginOp` / `performPluginUpdate` (行 829-1088)

**版本计算**：
```typescript
if (typeof entry.source !== 'string') {
  // 远程插件：下载后计算版本
  const cacheResult = await cachePlugin(entry.source, { manifest: { name: entry.name } })
  newVersion = await calculatePluginVersion(
    pluginId, entry.source, cacheResult.manifest, cacheResult.path,
    entry.version, cacheResult.gitCommitSha
  )
} else {
  // 本地插件：从源路径计算版本
  newVersion = await calculatePluginVersion(
    pluginId, entry.source, pluginManifest, sourcePath, entry.version
  )
}
```

**新旧版本判断**：
```typescript
const isUpToDate =
  installation.version === newVersion ||
  installation.installPath === versionedPath ||
  installation.installPath === zipPath
```

---

## 关键代码路径与文件引用

### 核心函数位置

| 函数 | 行号 | 描述 |
|------|------|------|
| `assertInstallableScope` | 90-98 | Scope 运行时验证 |
| `isInstallableScope` | 104-108 | Scope 类型守卫 |
| `getProjectPathForScope` | 114-116 | 获取 scope 对应的项目路径 |
| `isPluginEnabledAtProjectScope` | 128-132 | 检查插件是否在项目 scope 启用 |
| `findPluginInSettings` | 180-201 | 在 settings 中查找插件 |
| `findPluginByIdentifier` | 206-223 | 在已加载插件中查找 |
| `resolveDelistedPluginId` | 230-251 | 解析已下架插件 |
| `getPluginInstallationFromV2` | 258-299 | 从 V2 数据获取安装信息 |
| `installPluginOp` | 321-418 | 安装操作 |
| `uninstallPluginOp` | 427-558 | 卸载操作 |
| `setPluginEnabledOp` | 573-747 | 启用/禁用核心逻辑 |
| `enablePluginOp` | 756-761 | 启用包装 |
| `disablePluginOp` | 770-775 | 禁用包装 |
| `disableAllPluginsOp` | 782-812 | 禁用全部 |
| `updatePluginOp` | 829-890 | 更新操作入口 |
| `performPluginUpdate` | 896-1088 | 更新核心实现 |

### 依赖文件

```typescript
// 路径与文件系统
import { dirname, join } from 'path'
import { getOriginalCwd } from '../../bootstrap/state.js'
import { getFsImplementation } from '../../utils/fsOperations.js'

// 插件类型与常量
import { isBuiltinPluginId } from '../../plugins/builtinPlugins.js'
import type { LoadedPlugin, PluginManifest } from '../../types/plugin.js'

// 错误处理
import { isENOENT, toError } from '../../utils/errors.js'
import { logError } from '../../utils/log.js'

// 缓存管理
import { clearAllCaches, markPluginVersionOrphaned } from '../../utils/plugins/cacheUtils.js'

// 依赖解析
import {
  findReverseDependents,
  formatReverseDependentsSuffix,
} from '../../utils/plugins/dependencyResolver.js'

// 安装记录管理
import {
  loadInstalledPluginsFromDisk,
  loadInstalledPluginsV2,
  removePluginInstallation,
  updateInstallationPathOnDisk,
} from '../../utils/plugins/installedPluginsManager.js'

// 市场管理
import {
  getMarketplace,
  getPluginById,
  loadKnownMarketplacesConfig,
} from '../../utils/plugins/marketplaceManager.js'

// 目录与数据
import { deletePluginDataDir } from '../../utils/plugins/pluginDirectories.js'
import { deletePluginOptions } from '../../utils/plugins/pluginOptionsStorage.js'

// 插件标识与安装
import {
  parsePluginIdentifier,
  scopeToSettingSource,
} from '../../utils/plugins/pluginIdentifier.js'
import {
  formatResolutionError,
  installResolvedPlugin,
} from '../../utils/plugins/pluginInstallationHelpers.js'

// 插件加载与缓存
import {
  cachePlugin,
  copyPluginToVersionedCache,
  getVersionedCachePath,
  getVersionedZipCachePath,
  loadAllPlugins,
  loadPluginManifest,
} from '../../utils/plugins/pluginLoader.js'

// 策略与版本
import { isPluginBlockedByPolicy } from '../../utils/plugins/pluginPolicy.js'
import { getPluginEditableScopes } from '../../utils/plugins/pluginStartupCheck.js'
import { calculatePluginVersion } from '../../utils/plugins/pluginVersioning.js'

// Schema 与 Settings
import type { PluginMarketplaceEntry, PluginScope } from '../../utils/plugins/schemas.js'
import { getSettingsForSource, updateSettingsForSource } from '../../utils/settings/settings.js'

// 工具
import { plural } from '../../utils/stringUtils.js'
```

### 调用方

- `src/services/plugins/pluginCliCommands.ts` - CLI 命令包装层
- `src/commands/plugin/ManagePlugins.tsx` - 交互式 UI

---

## 依赖与外部交互

### 三层架构关系

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Intent (Settings)                                  │
│  - ~/.claude/settings.json                                   │
│  - .claude/settings.json (project)                           │
│  - .claude/local.json                                        │
└─────────────────────────────────────────────────────────────┘
                            ↑
         pluginOperations.ts 写 settings 声明意图
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Materialization (~/.claude/plugins/)              │
│  - known_marketplaces.json                                   │
│  - installed_plugins.json (V2)                               │
│  - cache/                                                    │
└─────────────────────────────────────────────────────────────┘
                            ↑
         marketplaceManager.ts / pluginLoader.ts 负责
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Active Components (AppState)                      │
│  - 加载的插件对象                                            │
│  - commands, agents, hooks, MCP servers                      │
└─────────────────────────────────────────────────────────────┘
```

### 与 pluginInstallationHelpers.ts 的关系

```typescript
// pluginOperations.ts 调用 installResolvedPlugin 核心逻辑
import { installResolvedPlugin } from './pluginInstallationHelpers.js'

// 分工：
// - pluginOperations.ts: 参数准备、错误转换、结果格式化
// - pluginInstallationHelpers.ts: 依赖解析、settings 写入、缓存
```

### 与 dependencyResolver.ts 的关系

```typescript
// 依赖解析
import { resolveDependencyClosure } from './dependencyResolver.js'

// 反向依赖检查
import { findReverseDependents } from './dependencyResolver.js'
```

---

## 风险、边界与改进建议

### 已知风险

1. **Settings 写入失败**
   - 风险：`updateSettingsForSource` 可能返回错误
   - 处理：返回 `settings-write-failed` 结果，不抛出异常

2. **依赖循环**
   - 风险：A 依赖 B，B 依赖 A
   - 处理：`resolveDependencyClosure` 检测循环并返回 `cycle` 错误

3. **跨市场依赖安全**
   - 风险：恶意插件通过依赖链引入不受信任代码
   - 处理：默认阻止跨市场依赖，需显式白名单 `allowCrossMarketplaceDependenciesOn`

4. **版本计算不确定性**
   - 风险：本地插件的版本计算依赖 git SHA，可能失败
   - 处理：使用 `unknown` 作为回退版本

### 边界情况

| 场景 | 处理 |
|------|------|
| 插件已在目标 scope 安装 | 返回"already installed"错误 |
| 插件在其他 scope 安装 | 提示用户使用正确 scope |
| 插件已下架（delisted） | 通过 V2 installed_plugins.json 解析 |
| 组织策略阻止 | 返回 `blocked-by-policy` 错误 |
| 依赖被策略阻止 | 返回 `dependency-blocked-by-policy` 错误 |
| 本地源插件无市场位置 | 返回 `local-source-no-location` 错误 |
| 更新时源路径不存在 | 返回友好错误信息 |

### 改进建议

1. **事务性安装**
   - 当前：settings 写入和缓存是分开的，可能部分失败
   - 建议：实现两阶段提交，失败时回滚 settings

2. **并发控制**
   - 当前：无显式并发控制，依赖文件系统锁
   - 建议：增加操作锁，防止并行安装冲突

3. **更好的错误上下文**
   - 当前：错误消息相对通用
   - 建议：包含更多上下文（尝试了哪些市场、哪些依赖失败等）

4. **依赖版本约束**
   - 当前：依赖解析只检查存在性
   - 建议：支持语义化版本约束（`^1.0.0`, `>=2.0.0` 等）

5. **安装回滚**
   - 当前：失败时残留部分安装文件
   - 建议：实现完整的回滚机制

6. **增量更新**
   - 当前：更新时复制整个插件
   - 建议：支持增量更新（如使用 git pull）

### 代码质量观察

1. **类型安全**：大量使用 TypeScript 类型，包括 branded types 用于 PII 标记
2. **单一职责**：每个函数职责清晰，复杂逻辑委托给专门模块
3. **错误处理**：使用结果对象而非异常作为主要错误传播机制
4. **注释质量**：复杂逻辑有详细注释，特别是边界情况处理
