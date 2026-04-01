# 研究文档：`src/utils/plugins/pluginLoader.ts`

> 研究范围：代码、类型定义、schema、直接调用方与间接依赖方、配置与测试上下文（如有）。未将 README / docs / Docs 作为研究目标。

---

## 1. 场景与职责

`pluginLoader.ts` 是 Claude Code 插件系统的**核心加载器（Layer-3）**，负责把“意图层”（settings.json 中 `enabledPlugins` + marketplace 元数据）转化为可在当前会话中运行的 `LoadedPlugin` 对象。它处于三层模型的最上层：

- **Layer 1（意图）**：`settings.json` 中的 `enabledPlugins`、`extraKnownMarketplaces` 等。
- **Layer 2（物化）**：`reconciler.ts` / `marketplaceManager.ts` 把 marketplace 克隆/缓存到 `~/.claude/plugins/`。
- **Layer 3（激活）**：`pluginLoader.ts` 读取物化后的目录或缓存，产出 `LoadedPlugin`，供命令、Agent、Hook、MCP、LSP 等子系统消费。

### 主要职责

1. **插件发现**：从三个来源合并插件——session-only（`--plugin-dir`）、marketplace（已安装/已启用）、builtin（内置）。
2. **插件获取与缓存**：支持 local path、npm、git URL、GitHub repo、`git-subdir`（monorepo 子目录）等多种来源；维护 versioned cache、legacy cache、seed cache、ZIP cache。
3. **Manifest 与组件加载**：解析 `plugin.json`（或 marketplace entry 作为 fallback），验证 commands、agents、skills、outputStyles、hooks、settings 的路径与格式。
4. **策略与依赖**：执行企业策略（allowlist/blocklist）的 fail-closed 检查；加载后对依赖做 `verifyAndDemote`（不满足依赖的插件会被降级为 disabled）。
5. **双模式加载**：提供 `loadAllPlugins`（完整加载，可触发网络克隆）和 `loadAllPluginsCacheOnly`（仅读本地缓存，用于启动加速）。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| `loadAllPlugins` / `loadAllPluginsCacheOnly` | 对外暴露的两种加载入口。完整模式用于用户显式刷新（`/reload-plugins`、安装后），缓存模式用于启动期快速恢复已物化的插件，避免阻塞在 git clone。 |
| `cachePlugin` | 把远程/本地插件源下载到一个临时目录，解析 manifest，然后重命名为以插件名命名的 cache 目录。是“首次安装”或“更新”时的原子写入 primitive。 |
| `copyPluginToVersionedCache` | 将已下载/已本地化的插件内容复制到 **versioned cache**（`~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`），支持 ZIP 格式；优先命中 seed cache（只读）。 |
| `installFromGitSubdir` | 针对 monorepo 场景，用 `git clone --filter=tree:0 --no-checkout` + `sparse-checkout set --cone -- <path>` 只拉取子目录，极大降低带宽与磁盘占用。 |
| `gitClone` / `installFromNpm` / `installFromGitHub` | 分别封装 git 克隆（支持 shallow clone、指定 ref/sha）、npm install（带 registry 与 version）、GitHub 快捷克隆。 |
| `createPluginFromPath` | 给定一个插件目录，扫描 `commands/`、`agents/`、`skills/`、`output-styles/`、`hooks/hooks.json`、`.claude-plugin/plugin.json`，组装成 `LoadedPlugin`。 |
| `finishLoadingPluginFromPath` | 共享尾逻辑：在 marketplace entry 与本地 manifest 之间做 **strict / non-strict** 合并或冲突检测。 |
| `mergePluginSources` | 合并 session、marketplace、builtin 三类来源，处理同名覆盖规则：session > marketplace，但 **managed settings 锁定** 的插件不可被 session 覆盖。 |
| `cachePluginSettings` | 把各插件 `settings`（仅 allowlisted 的 `agent` 键）合并后写入同步缓存，供设置级联读取。 |
| `clearPluginCache` | 清除 `loadAllPlugins` 与 `loadAllPluginsCacheOnly` 的 memoize 缓存，并同步清理 settings cache。 |

---

## 3. 具体技术实现

### 3.1 关键流程

#### A. 双模式加载流程（Full vs Cache-Only）

```
loadAllPlugins()
  └─ memoize ── assemblePluginLoadResult(marketplaceLoader=loadPluginsFromMarketplaces({cacheOnly:false}))

loadAllPluginsCacheOnly()
  └─ memoize ── if SYNC_PLUGIN_INSTALL → loadAllPlugins()
                 else assemblePluginLoadResult(marketplaceLoader=loadPluginsFromMarketplaces({cacheOnly:true}))
```

- `assemblePluginLoadResult` 是共享躯干：
  1. 并行加载 marketplace 插件与 session-only 插件（`getInlinePlugins()`）。
  2. 加载 builtin 插件（`getBuiltinPlugins()`）。
  3. `mergePluginSources` 合并三类来源，处理覆盖与 managed settings 锁定。
  4. `verifyAndDemote` 做依赖检查，将被依赖方缺失的插件降级。
  5. `cachePluginSettings` 把 allowlisted 设置写入同步缓存。

- **Full 模式** 会在 `loadPluginFromMarketplaceEntry` 中：
  - 对本地来源调用 `copyPluginToVersionedCache`。
  - 对外部来源先 `cachePlugin`（可能触发 git clone / npm install），再 `copyPluginToVersionedCache`。

- **Cache-Only 模式** 走 `loadPluginFromMarketplaceEntryCacheOnly`：
  - 本地来源直接读取 marketplace 目录（跳过复制）。
  - 外部来源依赖 `installed_plugins.json` 中的 `installPath`；若缺失则报 `plugin-cache-miss`。
  - 若启用了 ZIP cache，仍需把 `.zip` 提取到 session temp dir（这是 invariant：cache-only 不触发网络，但仍需解压）。

#### B. Versioned Cache 与 Seed Cache

缓存路径结构：
```
~/.claude/plugins/cache/{sanitized_marketplace}/{sanitized_plugin}/{sanitized_version}/
```

- `getVersionedCachePath`：主缓存路径。
- `probeSeedCache`：按 `getPluginSeedDirs()` 顺序探测只读 seed 目录；命中则直接返回 seed 路径，**不做复制**。
- `probeSeedCacheAnyVersion`：当计算出的 version 为 `'unknown'` 时，探测 seed 中该插件目录下的唯一子目录（处理首次启动时无法预知版本的情况）。
- ZIP cache 变体：在启用 `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` 时，`copyPluginToVersionedCache` 会把目录原地转换为 `.zip` 文件，后续加载时解压到 session temp dir。

#### C. `git-subdir` 的稀疏克隆实现

`installFromGitSubdir` 是 monorepo 插件的关键技术点：

1. `git clone --depth 1 --filter=tree:0 --no-checkout [--branch ref] <url> <targetPath>.clone`
2. `git sparse-checkout set --cone -- <subdirPath>`（要求 git >= 2.25）
3. 若指定了 `sha`：先 `fetch --depth 1 origin <sha>`（失败则 `--unshallow`），再 `checkout <sha>`。
   若未指定 `sha`：并行执行 `checkout HEAD` 与 `rev-parse HEAD`，后者用于记录 resolved SHA。
4. 把 `<cloneDir>/<subdirPath>` `rename` 到 `targetPath`。
5. `finally` 中删除临时 clone 目录。

该流程确保百万级文件 monorepo 不会把整个 tree 对象下载到本地。

#### D. Manifest 与组件加载

`createPluginFromPath` 的加载顺序：

1. **Manifest**：优先读取 `.claude-plugin/plugin.json`，失败则 fallback 到 `plugin.json`（legacy），再失败则生成默认 manifest。
2. **Auto-detect 目录**：并行检查 `commands/`、`agents/`、`skills/`、`output-styles/` 是否存在。
3. **Manifest 中声明的额外路径**：
   - `commands` 支持三种格式：单路径、路径数组、对象映射（`Record<string, CommandMetadata>`）。对象映射支持 `source`（文件路径）或 `content`（内联 markdown）。
   - `agents` / `skills` / `outputStyles` 支持单路径或路径数组。
   - 所有声明路径都通过 `validatePluginPaths` 做并行 `pathExists` 检查，不存在的路径记录 `path-not-found` 错误但不中断加载。
4. **Hooks**：
   - 自动加载 `hooks/hooks.json`（若存在）。
   - 若 manifest 声明了 `hooks`（文件路径或 inline object），则合并；通过 `realpath` 去重，strict 模式下对重复 hooks 文件报错。
5. **Settings**：读取 `settings.json`（插件目录内）或 `manifest.settings`，经 `PluginSettingsSchema`（`SettingsSchema.pick({agent:true}).strip()`）过滤后保留。

#### E. 企业策略（Enterprise Policy）

在 `loadPluginsFromMarketplaces` 中：

- 先 `loadKnownMarketplacesConfigSafe()` 读取已知 marketplace 配置（Safe 变体：解析失败返回 `{}`，防止坏配置导致整个插件系统崩溃）。
- 若配置了 `strictKnownMarketplaces`（allowlist）或 `blockedMarketplaces`（非空 blocklist），则 `hasEnterprisePolicy = true`。
- **Fail-closed**：若某插件所属的 marketplace 在 `knownMarketplaces` 中找不到配置，且策略已激活，则直接阻断并记录 `marketplace-blocked-by-policy` 错误。
- 若 marketplace 配置存在，再调用 `isSourceAllowedByPolicy` 判断其 source 是否在 allowlist / 不在 blocklist。

### 3.2 关键数据结构

#### `LoadedPlugin`（`src/types/plugin.ts`）

```ts
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string
  repository: string
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string
  commandsPath?: string
  commandsPaths?: string[]
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

#### `PluginError`（Discriminated Union）

包含 20+ 种错误变体，如 `path-not-found`、`hook-load-failed`、`marketplace-blocked-by-policy`、`plugin-cache-miss`、`dependency-unsatisfied`、`generic-error` 等。`pluginLoader.ts` 是这些错误的主要生产方之一。

#### `PluginMarketplaceEntry`（`src/utils/plugins/schemas.ts`）

```ts
type PluginMarketplaceEntry = {
  name: string
  source: PluginSource
  strict?: boolean // default true
  commands?: ...
  agents?: ...
  skills?: ...
  hooks?: ...
  outputStyles?: ...
  // 其他 manifest 字段
}
```

`strict` 字段决定 marketplace entry 与本地 `plugin.json` 的关系：
- `strict=true`（默认）：本地 `plugin.json` 必须存在；marketplace entry 只能**补充**组件路径。
- `strict=false`：允许没有 `plugin.json`，此时 marketplace entry 充当完整 manifest；若同时存在 `plugin.json` 与 marketplace 组件声明，则视为冲突，加载失败。

### 3.3 协议与命令

#### Git 命令序列（`gitClone`）

```bash
git clone --depth 1 --recurse-submodules --shallow-submodules \
  [--branch <ref>] [--no-checkout] <url> <targetPath>

# 若指定了 sha：
git fetch --depth 1 origin <sha>   # fallback: git fetch --unshallow
git checkout <sha>
```

#### NPM 命令序列（`installFromNpm`）

```bash
npm install <package>[@<version>] --prefix <npmCachePath> [--registry <url>]
```

然后 `copyDir(packagePath, targetPath)`。

#### ZIP 操作（`zipCache.ts`）

- `convertDirectoryToZipInPlace(cachePath, zipPath)`：目录 → ZIP，成功后删除原目录。
- `extractZipToDirectory(zipPath, extractDir)`：使用内部 `unzipFile` / `parseZipModes` 解压。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件核心函数分布

| 函数 | 行号 | 说明 |
|------|------|------|
| `getPluginCachePath` / `getVersionedCachePath` / `getVersionedZipCachePath` | ~50-90 | 缓存路径计算 |
| `probeSeedCache` / `probeSeedCacheAnyVersion` | ~95-140 | Seed cache 探测 |
| `copyDir` | ~180-250 | 递归目录复制，含 symlink 循环检测 |
| `copyPluginToVersionedCache` | ~260-370 | 版本化缓存写入 |
| `installFromNpm` | ~400-450 | NPM 安装 primitive |
| `gitClone` | ~460-540 | Git 克隆 primitive |
| `installFromGitSubdir` | ~600-740 | Monorepo 稀疏克隆 |
| `cachePlugin` | ~800-920 | 外部源下载 + manifest 解析 |
| `loadPluginManifest` | ~1147-1213 | Manifest 加载与校验 |
| `loadPluginHooks` | ~1224-1242 | Hooks JSON 加载 |
| `validatePluginPaths` | ~1265-1307 | 并行路径存在性检查 |
| `createPluginFromPath` | ~1348-1770 | 从目录构建 LoadedPlugin |
| `loadPluginsFromMarketplaces` | ~1888-2089 | Marketplace 插件发现与策略检查 |
| `loadPluginFromMarketplaceEntryCacheOnly` | ~2098-2174 | Cache-only 单插件加载 |
| `loadPluginFromMarketplaceEntry` | ~2191-2410 | Full 单插件加载 |
| `finishLoadingPluginFromPath` | ~2420-2917 | Manifest/entry 合并共享尾 |
| `loadSessionOnlyPlugins` | ~2928-2993 | `--plugin-dir` 插件加载 |
| `mergePluginSources` | ~3009-3064 | 来源合并与覆盖规则 |
| `loadAllPlugins` | ~3096-3108 | 完整加载入口（memoized） |
| `loadAllPluginsCacheOnly` | ~3137-3146 | 缓存加载入口（memoized） |
| `assemblePluginLoadResult` | ~3155-3211 | 共享加载躯干 |
| `clearPluginCache` | ~3225-3243 | 缓存清除 |
| `cachePluginSettings` | ~3281-3295 | 设置缓存写入 |

### 4.2 直接调用方（Import 关系）

```
src/main.tsx
  → clearPluginCache, loadAllPluginsCacheOnly

src/QueryEngine.ts
  → loadAllPluginsCacheOnly

src/hooks/useManagePlugins.ts
  → loadAllPlugins

src/commands/thinkback/thinkback.tsx
  → loadAllPlugins

src/commands/plugin/PluginOptionsFlow.tsx
  → loadAllPlugins

src/commands/plugin/ManagePlugins.tsx
  → loadAllPlugins

src/commands/plugin/ManageMarketplaces.tsx
  → loadAllPlugins

src/cli/handlers/plugins.ts
  → loadAllPlugins

src/cli/print.ts
  → loadAllPluginsCacheOnly

src/services/plugins/pluginOperations.ts
  → loadAllPlugins, cachePlugin, copyPluginToVersionedCache, createPluginFromPath, ...

src/services/plugins/PluginInstallationManager.ts
  → clearPluginCache

src/services/lsp/config.ts
  → loadAllPluginsCacheOnly

src/services/mcp/config.ts
  → loadAllPluginsCacheOnly

src/utils/plugins/loadPluginCommands.ts
  → loadAllPluginsCacheOnly

src/utils/plugins/loadPluginHooks.ts
  → clearPluginCache, loadAllPluginsCacheOnly

src/utils/plugins/loadPluginOutputStyles.ts
  → loadAllPluginsCacheOnly

src/utils/plugins/loadPluginAgents.ts
  → loadAllPluginsCacheOnly

src/utils/plugins/refresh.ts
  → loadAllPlugins

src/utils/plugins/headlessPluginInstall.ts
  → clearPluginCache

src/utils/plugins/performStartupChecks.tsx
  → clearPluginCache

src/utils/plugins/installedPluginsManager.ts
  → getPluginCachePath, getVersionedCachePath

src/utils/plugins/cacheUtils.ts
  → clearPluginCache, getPluginCachePath

src/utils/plugins/pluginInstallationHelpers.ts
  → getVersionedCachePath, resolvePluginPath, copyDir, cachePlugin, copyPluginToVersionedCache
```

### 4.3 强依赖的下游/上游文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/types/plugin.ts` | 类型定义 | `LoadedPlugin`、`PluginError`、`PluginLoadResult` |
| `src/utils/plugins/schemas.ts` | 类型+校验 | `PluginManifestSchema`、`PluginHooksSchema`、`PluginIdSchema`、`PluginMarketplaceEntrySchema`、`PluginSourceSchema` 等 |
| `src/utils/plugins/zipCache.ts` | 下游工具 | ZIP 缓存的转换与解压 |
| `src/utils/plugins/pluginVersioning.ts` | 下游工具 | 版本计算（manifest → git SHA → unknown） |
| `src/utils/plugins/dependencyResolver.ts` | 下游工具 | `verifyAndDemote`、依赖闭包解析 |
| `src/utils/plugins/installedPluginsManager.ts` | 上游数据 | 提供 `getInMemoryInstalledPlugins`、V1/V2 安装元数据 |
| `src/utils/plugins/marketplaceManager.ts` | 上游数据 | `getMarketplaceCacheOnly`、`getPluginByIdCacheOnly`、`loadKnownMarketplacesConfigSafe` |
| `src/utils/plugins/marketplaceHelpers.ts` | 策略工具 | `isSourceAllowedByPolicy`、`getBlockedMarketplaces`、`getStrictKnownMarketplaces` |
| `src/utils/plugins/pluginDirectories.ts` | 路径工具 | `getPluginsDirectory`、`getPluginSeedDirs` |
| `src/utils/settings/settingsCache.ts` | 下游缓存 | `setPluginSettingsBase`、`clearPluginSettingsBase`、`resetSettingsCache` |
| `src/utils/settings/types.ts` | 类型定义 | `HooksSettings`、`SettingsSchema` |
| `src/utils/execFileNoThrow.ts` | 执行工具 | 封装 `child_process.execFile`，返回 `{code, stdout, stderr}` |
| `src/utils/git.ts` | 执行工具 | `gitExe()` 返回 git 可执行路径 |

---

## 5. 依赖与外部交互

### 5.1 运行时外部依赖

- **Git 可执行文件**：`gitExe()` 必须在 PATH 上；`git-subdir` 要求 git >= 2.25（`sparse-checkout` cone mode）。
- **NPM 可执行文件**：`installFromNpm` 直接调用 `npm` 命令行。
- **文件系统**：大量使用 `fs/promises`（`readdir`、`stat`、`copyFile`、`symlink`、`rm` 等），并通过 `getFsImplementation()` 做一层抽象（用于测试 mock 或 bundled mode 替换）。
- **网络**：`cachePlugin` 中的 git clone / npm install 会触发网络请求；`loadPluginsFromMarketplaces` 在 Full 模式下可能因此阻塞。

### 5.2 环境变量

| 变量 | 影响 |
|------|------|
| `CLAUDE_CODE_REMOTE` | 若为 truthy，`installFromGitHub` 使用 `https://github.com/...` 而非 SSH `git@github.com:...` |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | 若为 truthy，`loadAllPluginsCacheOnly` 会回退到 `loadAllPlugins()`，强制在首次查询前阻塞安装 |
| `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` | 启用 ZIP cache 模式 |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | ZIP cache 的挂载目录 |

### 5.3 配置来源

- `settings.json` 中的 `enabledPlugins`：决定哪些插件被加载。
- `settings.json` 中的 `extraKnownMarketplaces`、`strictKnownMarketplaces`、`blockedMarketplaces`：决定 marketplace 发现与策略。
- `--plugin-dir` CLI flag / SDK `plugins` option：通过 `getInlinePlugins()` 传入，生成 session-only 插件。
- `--add-dir` 相关配置：通过 `getAddDirEnabledPlugins()` 以最低优先级合并到 `enabledPlugins`。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险与边界行为

#### 1. 双 memoize 缓存的竞态与预热逻辑

`loadAllPlugins` 与 `loadAllPluginsCacheOnly` 各自有独立的 `lodash-es/memoize` 缓存。`loadAllPlugins` 完成后会手动向 `loadAllPluginsCacheOnly.cache` 写入结果（`cache.set(undefined, Promise.resolve(result))`），以避免 `refresh.ts` 中先 full load、后 cache-only 消费时出现 `plugin-cache-miss`。

- **风险**：若外部代码在 full load 完成前并发调用 `loadAllPluginsCacheOnly`，会读到旧缓存或触发 cache miss。
- **边界**：`clearPluginCache` 必须同时清除两个缓存，否则会出现状态不一致。

#### 2. `git-subdir` 的 git 版本依赖

`installFromGitSubdir` 依赖 `sparse-checkout set --cone`，需要 git >= 2.25。代码中在失败时会将 stderr 透传给用户，但不会做版本预检；在旧 git 环境中报错信息可能不够直观。

#### 3. Version = `'unknown'` 的无限重克隆

当外部来源（如 git ref-tracked）无法计算出版本时，`calculatePluginVersion` 返回 `'unknown'`。每次 full load 都会因为 versioned cache miss 而重新 clone。这在设计上是“freshness mechanism”，但对用户而言意味着每次 `/reload-plugins` 都会重新下载。

- 缓解：seed cache 的 `probeSeedCacheAnyVersion` 可以在首次启动时命中预置插件；CCR 镜像构建场景受益于此。

#### 4. Symlink 循环与 `copyDir`

`copyDir` 对 symbolic link 做了循环检测：通过 `realpath` 解析目标，若目标在源树内则创建相对 symlink，否则创建绝对 symlink。但若 symlink 链极其复杂或存在跨文件系统的循环，仍可能触发栈溢出或 ENOENT。

#### 5. 企业策略的 fail-closed 与 UX 权衡

当 `known_marketplaces.json` 损坏或被清空时，`loadKnownMarketplacesConfigSafe` 返回 `{}`。若此时存在 active policy，插件会被全部阻断并显示 `marketplace-blocked-by-policy`，而不是 `plugin-not-found`。这对用户来说可能困惑——明明只是 marketplace 配置丢了，却显示策略错误。

#### 6. ZIP cache 的损坏处理

在 `loadPluginFromMarketplaceEntry` 的 ZIP 解压路径中，若解压失败会删除该 `.zip` 文件并抛出异常。下一次 full load 会重新触发 `cachePlugin` 重新下载。这是合理的自修复，但在只读挂载场景（如某些容器）中，删除可能报 EACCES，导致错误升级。

#### 7. `mergePluginSources` 中 managed settings 的匹配粒度

managed names 的比较基于 `plugin.name`（即 manifest.name）。如果 marketplace entry 的 `name` 与 manifest 的 `name` 不一致（marketplace 配置错误），managed settings 的锁定会失效，session 插件可能意外覆盖。

#### 8. `verifyAndDemote` 的固定点循环

依赖不满足时插件被 demoted（`enabled = false`），但 demotion 可能连锁导致其他插件也不满足。代码使用 `while (changed)` 固定点循环处理。该循环的复杂度为 O(N²) 最坏情况，N 为插件数量；在正常使用中 N 很小，不构成性能问题。

### 6.2 改进建议

1. **增加 git 版本预检**：在 `installFromGitSubdir` 入口处增加 `git --version` 检查（或复用 `checkGitAvailable()` 的增强版），当版本 < 2.25 时给出更明确的错误提示。
2. **细化 unknown version 的缓存策略**：可考虑在 `installed_plugins.json` 中记录最后一次成功 clone 的 resolved SHA，即使 version 为 `'unknown'` 也能用该 SHA 做缓存键，减少不必要的重克隆。
3. **统一错误上下文**：`marketplace-blocked-by-policy` 在 `knownMarketplaces` 为空时，可在错误消息中追加提示“marketplace configuration missing or corrupted”，帮助用户区分策略阻断与配置损坏。
4. **测试覆盖**：当前仓库中未找到针对 `pluginLoader.ts` 的单元测试文件（`__tests__/pluginLoader*.test.ts` 不存在）。建议为核心函数（`createPluginFromPath`、`mergePluginSources`、`verifyAndDemote` 的交互、`copyDir` 的 symlink 处理）补充集成测试。
5. **ZIP cache 只读挂载兼容**：在删除损坏 ZIP 前检查文件系统是否可写，若不可写则跳过删除、仅记录错误，避免 EACCES 升级。
6. **减少 `finishLoadingPluginFromPath` 的重复代码**：该函数对 commands/agents/skills/outputStyles/hooks 的处理存在大量结构相似的代码块，可考虑提取一个通用的“manifest entry supplement”辅助函数，降低维护成本。

---

*文档生成时间：基于代码库当前 HEAD 的静态分析。*
