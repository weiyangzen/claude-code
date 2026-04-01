# 研究报告：src/utils/plugins/pluginStartupCheck.ts

## 场景与职责

`pluginStartupCheck.ts` 是 Claude Code 插件系统在**启动阶段和安装阶段**的核心状态协调模块。它桥接了“用户意图层”（settings.json 中的 `enabledPlugins`）与“全局安装层”（`installed_plugins.json`），负责：

1. **计算当前启用的插件列表**：跨所有设置源（`--add-dir`、policy、user、project、local、flag）合并，得出最终哪些插件处于启用状态。
2. **追踪可编辑作用域**：为每个启用的插件确定“用户可以在哪个作用域修改它的启用状态”。
3. **发现缺失插件**：找出“已启用但未安装”的插件，为自动安装提供输入。
4. **批量安装插件**：`installSelectedPlugins` 是安装流程的底层实现之一，负责将插件缓存到磁盘并更新设置。

## 功能点目的

### 1. `checkEnabledPlugins()` — 权威的“是否启用”检查
- **目的**：在启动、刷新、安装等流程中，获取当前应当激活的插件完整列表。
- **优先级设计**（从低到高）：
  1. `--add-dir` 插件（会话级，最低优先级）
  2. 合并设置（`getInitialSettings()`，policy > local > project > user）
- **行为**：合并设置中的显式 `false` 可以覆盖 `--add-dir` 的启用；`true` 可以补充 `--add-dir` 未列出的插件。

### 2. `getPluginEditableScopes()` — 作用域所有权追踪
- **目的**：当用户在 UI 中启用/禁用插件时，系统需要知道“应该往哪个 settings 源写回”。
- **优先级设计**（从低到高）：
  1. `addDir` → 映射为 `'flag'`（会话级，不持久化）
  2. `managed`（policySettings）→ 用户不可编辑
  3. `user`
  4. `project`
  5. `local`
  6. `flag`（flagSettings）→ 会话级，不持久化
- **行为**：后处理的作用域覆盖前面的结果。`managed` 先处理是因为用户无法编辑它，最终应落在最高优先级的**用户可控**作用域上。

### 3. `getInstalledPlugins()` — 读取已安装插件
- **目的**：获取 `installed_plugins.json` 中记录的全局安装列表。
- **设计**：
  - 触发 `migrateFromEnabledPlugins()` 将旧版 `enabledPlugins` 设置同步到 V2 安装记录（后台异步，不阻塞）。
  - 使用 `getInMemoryInstalledPlugins()` 获取 V2 格式的内存快照。

### 4. `findMissingPlugins()` — 发现待安装插件
- **目的**：在 `checkEnabledPlugins()` 之后，找出“已启用但未安装”的插件，供自动安装流程使用。
- **设计**：先同步过滤出未安装的 ID，再并行调用 `getPluginById()` 在市场数据中确认这些插件是否存在（不存在的插件会在上层报错处理）。

### 5. `installSelectedPlugins()` — 批量安装
- **目的**：为 headless 模式、CLI 命令、UI 安装操作提供统一的批量安装能力。
- **设计**：
  - 支持 `user`/`project`/`local` 三种安装作用域。
  - 对每个插件：从 marketplace 查找 → 区分本地/外部来源进行缓存/注册 → 写入对应 settings 源的 `enabledPlugins`。
  - 返回 `{ installed: string[], failed: Array<{name, error}> }`。

## 具体技术实现

### 关键数据结构

```ts
export type PluginInstallResult = {
  installed: string[]
  failed: Array<{ name: string; error: string }>
}

type InstallableScope = Exclude<PluginScope, 'managed'>
```

### `checkEnabledPlugins` 合并算法

```ts
const enabledPlugins: string[] = []

// 1. 先加入 --add-dir 插件
for (const [pluginId, value] of Object.entries(addDirPlugins)) {
  if (pluginId.includes('@') && value) enabledPlugins.push(pluginId)
}

// 2. 合并设置覆盖 --add-dir
for (const [pluginId, value] of Object.entries(settings.enabledPlugins ?? {})) {
  if (!pluginId.includes('@')) continue
  const idx = enabledPlugins.indexOf(pluginId)
  if (value) {
    if (idx === -1) enabledPlugins.push(pluginId)
  } else {
    if (idx !== -1) enabledPlugins.splice(idx, 1)
  }
}
```

### `getPluginEditableScopes` 算法

```ts
const result = new Map<string, ExtendedPluginScope>()

// addDir 先处理
for (const [pluginId, value] of Object.entries(addDirPlugins)) {
  if (value === true) result.set(pluginId, 'flag')
  else if (value === false) result.delete(pluginId)
}

// 标准源按优先级顺序处理
const scopeSources = [
  { scope: 'managed', source: 'policySettings' },
  { scope: 'user', source: 'userSettings' },
  { scope: 'project', source: 'projectSettings' },
  { scope: 'local', source: 'localSettings' },
  { scope: 'flag', source: 'flagSettings' },
]

for (const { scope, source } of scopeSources) {
  const settings = getSettingsForSource(source)
  for (const [pluginId, value] of Object.entries(settings?.enabledPlugins ?? {})) {
    if (value === true) result.set(pluginId, scope)
    else if (value === false) result.delete(pluginId)
  }
}
```

### `installSelectedPlugins` 流程

1. 根据 `scope` 获取 `projectPath`（非 user 作用域时使用 `getCwd()`）。
2. 通过 `scopeToSettingSource(scope)` 获取对应的 settings 源。
3. 遍历 `pluginsToInstall`：
   - `getPluginById(pluginId)` 查找市场条目。
   - 若来源为本地（`isLocalPluginSource`）→ `registerPluginInstallation`（直接注册本地路径）。
   - 若为外部来源 → `cacheAndRegisterPlugin`（克隆/缓存到 `~/.claude/plugins/cache`）。
   - 将 `pluginId` 标记为 `enabledPlugins[pluginId] = true`。
4. 调用 `updateSettingsForSource(settingSource, { ...settings, enabledPlugins: updatedEnabledPlugins })` 持久化。

### 文件引用

- `getAddDirEnabledPlugins()` → `src/utils/plugins/addDirPluginSettings.ts`
- `getInMemoryInstalledPlugins()` / `migrateFromEnabledPlugins()` → `src/utils/plugins/installedPluginsManager.ts`
- `getPluginById()` → `src/utils/plugins/marketplaceManager.ts`
- `cacheAndRegisterPlugin()` / `registerPluginInstallation()` → `src/utils/plugins/pluginInstallationHelpers.ts`
- `scopeToSettingSource()` / `SETTING_SOURCE_TO_SCOPE` → `src/utils/plugins/pluginIdentifier.ts`
- `isLocalPluginSource()` / `PluginScope` → `src/utils/plugins/schemas.ts`
- `getInitialSettings()` / `getSettingsForSource()` / `updateSettingsForSource()` → `src/utils/settings/settings.ts`
- `getCwd()` → `src/utils/cwd.js`

## 依赖与外部交互

### 上游调用方
- `src/services/plugins/pluginOperations.ts`：大量调用 `getPluginEditableScopes`、`findMissingPlugins`、`installSelectedPlugins`。
- `src/commands/thinkback/thinkback.tsx`：启动检查中调用。
- `src/commands/plugin/PluginSettings.tsx`、`ManagePlugins.tsx`：UI 层获取可编辑作用域和启用列表。
- `src/cli/print.ts`（通过 `pluginOperations` 间接）：headless 模式自动安装。

### 下游依赖
- `addDirPluginSettings.ts`：读取 `--add-dir` 目录下的插件设置。
- `installedPluginsManager.ts`：V2 安装记录的读写与迁移。
- `marketplaceManager.ts`：按 ID 查找插件市场条目。
- `pluginInstallationHelpers.ts`：实际的缓存和注册逻辑。
- `pluginIdentifier.ts`：作用域与 settings 源之间的映射。
- `settings.ts`：所有 settings 读写操作。

## 风险、边界与改进建议

### 已知风险

1. **`getInstalledPlugins` 的异步迁移竞态**
   - `getInstalledPlugins` 中 `migrateFromEnabledPlugins()` 被 `void` 丢弃不 await。这意味着在迁移完成前，`getInMemoryInstalledPlugins()` 可能读取到旧数据。虽然 `getInMemoryInstalledPlugins` 内部也会触发迁移，但两个入口并行调用时存在竞态窗口。

2. **`findMissingPlugins` 的静默吞错**
   - 当 `getInstalledPlugins` 或 `getPluginById` 抛出异常时，`findMissingPlugins` 捕获后返回空数组 `[]`。这会导致上层认为“没有缺失插件”，从而跳过自动安装。在启动路径中，这可能使用户启用的插件未被加载且无任何报错提示。

3. **`installSelectedPlugins` 的部分失败处理**
   - 循环中某个插件安装失败时，已成功的插件已被缓存并写入 `installed_plugins.json`，但 `enabledPlugins` 的更新是在循环结束后统一写入 settings。若 settings 写入失败，会出现“已安装但未启用”的不一致状态。
   - 反之，若某个插件的 `cacheAndRegisterPlugin` 成功但 `updateSettingsForSource` 之前抛错，已缓存的插件会成为孤儿版本（7 天后由后台清理）。

4. **`checkEnabledPlugins` 与 `getPluginEditableScopes` 的语义差异**
   - `checkEnabledPlugins` 使用 `getInitialSettings()`（policy 最高优先级）。
   - `getPluginEditableScopes` 使用 `getSettingsForSource` 逐个源读取，且 `managed` 优先级最低（因为不可编辑）。
   - 注释已明确警告“不要用 `getPluginEditableScopes` 判断插件是否启用”，但新开发者仍可能误用。两个函数名称差异不够大，容易混淆。

### 边界情况

- **无效 pluginId**：两个函数都跳过不含 `@` 的键（`continue`），防止旧版裸名或损坏数据进入列表。
- **`enabledPlugins` 值为版本字符串**：注释提到未来 P2 可能用版本字符串表示“启用并固定到该版本”。当前实现将其忽略（只处理 `true`/`false`）。
- **`--add-dir` 与标准源冲突**：`getPluginEditableScopes` 会打印 debug 日志记录覆盖行为。

### 改进建议

1. **统一启用检查入口**
   - 考虑将 `checkEnabledPlugins` 和 `getPluginEditableScopes` 的公共逻辑提取为共享的迭代器，减少重复代码，并降低误用风险。

2. **增强 `findMissingPlugins` 的错误传播**
   - 建议返回 `{ missing: string[], errors: Array<{pluginId, error}> }` 而非在异常时静默返回 `[]`。让调用方（如 `thinkback.tsx` 或 `print.ts`）能向用户或日志报告问题。

3. **`installSelectedPlugins` 的事务性改进**
   - 可将 `enabledPlugins` 的增量更新改为“每成功一个就写一次”，或至少先收集所有成功的 ID，再统一写入。若写入失败，应尝试回滚或返回明确的 `settings-write-failed` 错误。

4. **增加作用域冲突检测**
   - 当一个插件在多个可编辑作用域（如 user 和 project）同时启用时，`getPluginEditableScopes` 只会返回最高优先级作用域。这在当前设计中是正确的，但 UI 层可能需要提示用户“该插件也在其他作用域启用了”。
