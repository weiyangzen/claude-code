# pluginDirectories.ts 研究文档

## 场景与职责

`src/utils/plugins/pluginDirectories.ts` 是 Claude Code 插件系统的**目录配置单一事实来源（single source of truth）**。它负责：

1. **确定插件根目录路径**：根据 CLI 标志、环境变量或默认规则，解析出 `~/.claude/plugins` 或 `~/.claude/cowork_plugins` 等路径。
2. **支持只读种子目录（seed directories）**：允许用户通过环境变量预置一个已填充的插件目录镜像，作为优先读取层，避免首次启动时的网络克隆。
3. **提供插件数据目录**：为每个插件分配持久化的数据目录（`~/.claude/plugins/data/{sanitized-plugin-id}/`），该目录在插件更新时保留，仅在最后一个 scope 卸载时清理。
4. **数据目录生命周期管理**：包括创建（`mkdirSync`）、计算大小（用于卸载确认对话框）、删除（`rm`）等操作。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getPluginsDirectory()` | 返回插件根目录的绝对路径。优先级：`CLAUDE_CODE_PLUGIN_CACHE_DIR` > `~/.claude/(cowork_)plugins`。 |
| `getPluginSeedDirs()` | 返回由 `CLAUDE_CODE_PLUGIN_SEED_DIR` 配置的只读种子目录列表（支持多路径，使用平台路径分隔符）。 |
| `pluginDataDirPath(pluginId)` | 纯函数，返回某插件的数据目录路径（不做 mkdir）。 |
| `getPluginDataDir(pluginId)` | 返回数据目录路径，并**同步** `mkdirSync` 创建（在变量替换的同步路径中被调用）。 |
| `getPluginDataDirSize(pluginId)` | 递归计算数据目录总字节数，返回 `{bytes, human}` 或 `null`（空/不存在）。用于卸载确认 UI。 |
| `deletePluginDataDir(pluginId)` | 在最后一个 scope 卸载时最佳 effort 删除数据目录，失败不抛异常。 |

---

## 具体技术实现

### 关键流程

#### 1. 插件根目录解析流程

```
getPluginsDirectory()
  ├─> process.env.CLAUDE_CODE_PLUGIN_CACHE_DIR ?
  │     └─> expandTilde(envOverride)   // 处理 settings.json 中未经过 shell 展开的 ~
  └─> join(getClaudeConfigHomeDir(), getPluginsDirectoryName())
        └─> getPluginsDirectoryName()
              ├─> getUseCoworkPlugins() ? 'cowork_plugins'   // 来自 --cowork CLI 标志
              ├─> isEnvTruthy(process.env.CLAUDE_CODE_USE_COWORK_PLUGINS) ? 'cowork_plugins'
              └─> 'plugins'
```

#### 2. 种子目录解析流程

```
getPluginSeedDirs()
  ├─> process.env.CLAUDE_CODE_PLUGIN_SEED_DIR
  │     └─> raw.split(path.delimiter)   // Unix ':' / Windows ';'
  │           .filter(Boolean)
  │           .map(expandTilde)
  └─> 返回绝对路径数组（空数组表示未配置）
```

种子目录结构预期与主目录一致：
```
$CLAUDE_CODE_PLUGIN_SEED_DIR/
  known_marketplaces.json
  marketplaces/<name>/...
  cache/<marketplace>/<plugin>/<version>/...
```

#### 3. 数据目录生命周期

- **路径生成**：`pluginDataDirPath(pluginId)` 使用 `sanitizePluginId` 将插件 ID 中的非 `[a-zA-Z0-9\-_]` 字符替换为 `-`，防止路径遍历。
- **创建**：`getPluginDataDir` 是**同步**的（`mkdirSync`），因为它在 `substitutePluginVariables` 的同步 `String.replace` 回调中被调用。
- **大小计算**：`getPluginDataDirSize` 使用异步递归 walk，对每个文件 `stat` 求和。对 broken symlink 的 `ENOENT` 做了单文件级 catch，避免一个坏链接导致整个对话框被跳过。
- **删除**：`deletePluginDataDir` 使用 `rm(..., {recursive: true, force: true})`，失败仅记录 warn log，不抛异常——与 `deletePluginOptions` 等函数保持一致。

### 数据结构

- `PLUGINS_DIR = 'plugins'`
- `COWORK_PLUGINS_DIR = 'cowork_plugins'`
- 环境变量：
  - `CLAUDE_CODE_PLUGIN_CACHE_DIR`：显式覆盖插件根目录。
  - `CLAUDE_CODE_USE_COWORK_PLUGINS`：启用 cowork 模式目录。
  - `CLAUDE_CODE_PLUGIN_SEED_DIR`：只读种子目录（PATH 风格多路径）。

---

## 关键代码路径与文件引用

### 本文件导出

| 导出 | 用途 |
|------|------|
| `getPluginsDirectory(): string` | 主插件根目录 |
| `getPluginSeedDirs(): string[]` | 种子目录列表 |
| `pluginDataDirPath(pluginId): string` | 数据目录路径（纯函数，无 IO） |
| `getPluginDataDir(pluginId): string` | 数据目录路径（确保存在，同步 mkdir） |
| `getPluginDataDirSize(pluginId)` | 异步计算目录大小 |
| `deletePluginDataDir(pluginId)` | 异步删除目录 |

### 直接依赖文件

- `src/bootstrap/state.ts`：`getUseCoworkPlugins`
- `src/utils/envUtils.ts`：`getClaudeConfigHomeDir`, `isEnvTruthy`
- `src/utils/errors.ts`：`errorMessage`, `isFsInaccessible`
- `src/utils/format.ts`：`formatFileSize`
- `src/utils/permissions/pathValidation.ts`：`expandTilde`
- `src/utils/debug.ts`：`logForDebugging`
- `src/utils/fsOperations.ts`：`getFsImplementation`

### 调用方文件（广泛）

| 调用方 | 使用的导出 |
|--------|-----------|
| `src/utils/plugins/pluginLoader.ts` | `getPluginsDirectory`, `getPluginSeedDirs` |
| `src/utils/plugins/marketplaceManager.ts` | `getPluginsDirectory`, `getPluginSeedDirs`, `deletePluginDataDir` |
| `src/utils/plugins/installedPluginsManager.ts` | `getPluginsDirectory` |
| `src/utils/plugins/pluginFlagging.ts` | `getPluginsDirectory` |
| `src/utils/plugins/pluginOptionsStorage.ts` | `getPluginDataDir` |
| `src/utils/plugins/mcpPluginIntegration.ts` | `getPluginDataDir` |
| `src/utils/plugins/lspPluginIntegration.ts` | `getPluginDataDir` |
| `src/utils/plugins/orphanedPluginFilter.ts` | `getPluginsDirectory` |
| `src/utils/plugins/installCounts.ts` | `getPluginsDirectory` |
| `src/utils/hooks.ts` | `getPluginDataDir` |
| `src/commands/plugin/ManagePlugins.tsx` | `getPluginDataDirSize`, `pluginDataDirPath` |
| `src/main.tsx` | `getPluginSeedDirs` |
| `src/services/plugins/pluginOperations.ts` | `deletePluginDataDir` |

---

## 依赖与外部交互

### 外部系统/环境变量

- **`CLAUDE_CODE_PLUGIN_CACHE_DIR`**：最高优先级目录覆盖。注释特别指出需要 `expandTilde`，因为当该变量通过 `settings.json` 的 `env` 字段设置时，shell 不会展开 `~`，否则会在 cwd 下创建字面量 `~` 目录（gh-30794 / CC-212）。
- **`CLAUDE_CODE_USE_COWORK_PLUGINS`**：布尔环境变量，启用 cowork 插件目录。
- **`--cowork` CLI 标志**：通过 `bootstrap/state.ts` 中的 `getUseCoworkPlugins()` 读取，优先级高于环境变量。
- **`CLAUDE_CODE_PLUGIN_SEED_DIR`**：支持多路径（`:` 或 `;` 分隔），用于容器镜像预置插件场景。
- **`~/.claude/`**：默认配置主目录，由 `getClaudeConfigHomeDir()` 返回。

### 文件系统交互

- `mkdirSync(dir, { recursive: true })`：同步创建数据目录。
- `readdir` + `stat`：计算目录大小。
- `rm(..., { recursive: true, force: true })`：删除数据目录。

---

## 风险、边界与改进建议

### 风险与边界

1. **`expandTilde` 的必要性隐含配置注入风险**
   - 虽然 `expandTilde` 解决了 `~` 未展开的问题，但如果 `CLAUDE_CODE_PLUGIN_CACHE_DIR` 被恶意设置为指向系统敏感目录（如 `/etc`），后续插件缓存、marketplace 克隆、数据目录都会写到该位置。当前调用方（如 `pluginLoader.ts`）没有对该返回值做二次校验。

2. **同步 `mkdirSync` 在热路径**
   - `getPluginDataDir` 是同步的，且在 MCP/LSP server env 构建、hook env 构建等路径中被**同步**调用。虽然单个 `mkdirSync` 开销小，但如果插件数量多且每次加载都触发，可能在启动时累积成微卡顿。

3. **种子目录的只读假设**
   - 代码注释和逻辑都假设 seed dir 是只读的，但没有任何文件系统权限层面的强制（如 `chmod` 或 `readonly` 挂载检查）。如果 seed dir 被意外写入了，行为未定义。

4. **数据目录大小计算的 TOCTOU**
   - `getPluginDataDirSize` 是异步递归 walk，在遍历过程中文件可能被删除或新增，导致计算结果与删除时的实际大小不一致。不过该函数仅用于卸载确认对话框的参考显示，不影响卸载逻辑正确性。

5. **`sanitizePluginId` 的碰撞风险**
   - 所有非 `[a-zA-Z0-9\-_]` 字符都被替换为 `-`。例如 `plugin@marketplace` 和 `plugin-marketplace` 会生成相同的数据目录名。虽然 marketplace 名和插件名本身通常不会设计成这种冲突形式，但理论上是可能的。

### 改进建议

1. **对 `CLAUDE_CODE_PLUGIN_CACHE_DIR` 增加路径白名单/黑名单校验**
   - 在返回前检查目标路径是否在用户主目录下，或至少拒绝指向 `/etc`、`/usr`、`/bin` 等系统目录的值，防止配置注入导致系统文件被覆盖。

2. **将 `getPluginDataDir` 的同步 mkdir 改为按需懒加载**
   - 可考虑在首次真正需要写入数据时才创建目录（例如在子进程实际写入前），而不是在每次 env 构建时都同步 `mkdirSync`。不过需要评估改动对同步代码路径的连锁影响。

3. **为 seed dir 增加存在性与可读性校验日志**
   - 在 `getPluginSeedDirs` 返回后，增加一层对目录存在性和可读性的快速探测，并在 debug log 中输出实际生效的种子路径，便于容器场景排障。

4. **改进 `sanitizePluginId` 避免碰撞**
   - 可考虑使用更稳定的编码方式（如 base64url 或 percent-encoding）替代简单的 `-` 替换，确保不同插件 ID 不会映射到同一目录名。
