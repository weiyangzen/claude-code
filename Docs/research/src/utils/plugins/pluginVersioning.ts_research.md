# 研究报告：src/utils/plugins/pluginVersioning.ts

## 场景与职责

`pluginVersioning.ts` 是 Claude Code 插件系统的**版本计算与解析模块**。它为来自不同来源的插件计算一个稳定、可比较的版本标识，用于：

1. **版本化缓存路径**：插件安装后存放在 `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`，版本字符串决定了缓存目录名。
2. **更新检测**：通过比较已安装版本与市场条目中的版本，判断插件是否需要更新。
3. **缓存键区分**：对于 git-subdir 等复杂来源，确保同一 monorepo 不同子目录的插件不会共享同一个缓存目录。

## 功能点目的

### 1. `calculatePluginVersion()` — 插件版本计算
- **目的**：为任意来源的插件生成一个统一的版本字符串。
- **优先级策略**（从高到低）：
  1. `plugin.json` 中的显式 `version` 字段（最高优先级）
  2. 市场条目提供的 `providedVersion`
  3. 预解析的 git commit SHA（`gitCommitSha`）
  4. 从安装路径动态读取的 git commit SHA
  5. `'unknown'`（最后兜底）

- **特殊处理 — `git-subdir`**：
  - 由于 `git-subdir` 来源在克隆后只保留子目录（`.git` 被丢弃），无法从安装路径读取 SHA。
  - 因此依赖调用方在克隆阶段预捕获 `gitCommitSha`。
  - 为避免同一 monorepo 不同子目录的插件碰撞缓存，版本字符串格式为 `{shortSha}-{pathHash}`，其中 `pathHash` 是规范化子目录路径的 SHA256 前 8 位。
  - 路径规范化规则必须与后端 squashfs cron 完全一致（`\`→`/`，去掉 `./` 前缀，去掉尾部 `/`）。

### 2. `getGitCommitSha()` — 读取目录的 git HEAD
- **目的**：为基于 git 的插件（本地开发、GitHub 来源等）提供版本标识。
- **实现**：委托给 `src/utils/git/gitFilesystem.ts` 的 `getHeadForDir()`。

### 3. `getVersionFromPath()` / `isVersionedPath()` — 从缓存路径反解版本
- **目的**：给定一个插件安装路径，判断它是否是版本化缓存路径，并提取版本字符串。
- **规则**：路径必须包含 `.../plugins/cache/{marketplace}/{plugin}/{version}/` 结构，从 `cache` 后第 3 段提取版本。

## 具体技术实现

### 关键函数签名

```ts
export async function calculatePluginVersion(
  pluginId: string,
  source: PluginSource,
  manifest?: PluginManifest,
  installPath?: string,
  providedVersion?: string,
  gitCommitSha?: string,
): Promise<string>
```

### `git-subdir` 版本计算细节

```ts
if (typeof source === 'object' && source.source === 'git-subdir') {
  const normPath = source.path
    .replace(/\\/g, '/')
    .replace(/^\.\//, '')
    .replace(/\/+$/, '')
  const pathHash = createHash('sha256')
    .update(normPath)
    .digest('hex')
    .substring(0, 8)
  return `${shortSha}-${pathHash}`
}
```

### 路径版本提取算法

```ts
export function getVersionFromPath(installPath: string): string | null {
  const parts = installPath.split('/').filter(Boolean)
  const cacheIndex = parts.findIndex(
    (part, i) => part === 'cache' && parts[i - 1] === 'plugins'
  )
  if (cacheIndex === -1) return null
  const componentsAfterCache = parts.slice(cacheIndex + 1)
  if (componentsAfterCache.length >= 3) {
    return componentsAfterCache[2] || null
  }
  return null
}
```

### 文件引用

- `getHeadForDir()` → `src/utils/git/gitFilesystem.ts`
- `PluginManifest` / `PluginSource` → `src/utils/plugins/schemas.ts`

## 依赖与外部交互

### 上游调用方
- `src/utils/plugins/pluginInstallationHelpers.ts`：`cacheAndRegisterPlugin` 在缓存插件后调用，计算版本并移动到版本化目录。
- `src/utils/plugins/pluginLoader.ts`：`loadAllPlugins` 在加载插件时调用，为没有显式版本的插件计算版本；`copyPluginToVersionedCache` 也依赖此函数。
- `src/services/plugins/pluginOperations.ts`：更新插件时比较新旧版本。
- `src/utils/plugins/installedPluginsManager.ts`：V1→V2 迁移等场景下读取已安装插件的版本信息。

### 下游依赖
- `src/utils/git/gitFilesystem.ts`：读取 git HEAD SHA。
- `src/utils/plugins/schemas.ts`：`PluginManifest` 和 `PluginSource` 类型定义。
- Node.js `crypto`：用于 `git-subdir` 路径哈希。

## 风险、边界与改进建议

### 已知风险

1. **`'unknown'` 版本的碰撞问题**
   - 当插件来源既不是 git 也没有显式 `version` 时，回退到 `'unknown'`。多个不同版本的此类插件会共享同一个 `.../unknown/` 缓存目录，导致缓存污染或更新检测失效。
   - 当前缓解措施：大多数官方/常用来源（GitHub、git、npm、本地目录）都能提供 git SHA 或显式版本，`'unknown'` 主要出现在极少数异常来源。

2. **`getVersionFromPath` 的平台路径假设**
   - 该函数使用 `split('/')` 硬编码正斜杠。在 Windows 上，若 `installPath` 使用反斜杠（如 `C:\Users\...\.claude\plugins\cache\...`），`getVersionFromPath` 会返回 `null`。
   - 影响：Windows 环境下 `isVersionedPath` 判断失效，可能导致某些依赖路径反解版本的逻辑（如孤儿清理、更新检测）出错。

3. **`git-subdir` 路径规范化与后端耦合**
   - 注释明确说明 `git-subdir` 的规范化规则必须与后端 cron `api/…/plugins_official_squashfs/job.py` 的 `_validate_subdir()` 完全一致。若任一方修改而未同步，会导致官方市场与本地缓存的键不一致，引发缓存未命中或重复下载。

4. **`calculatePluginVersion` 的异步签名与实际同步逻辑**
   - 函数标记为 `async`，但内部只有在 `installPath` 分支会 `await getGitCommitSha()`。若调用方提供了 `manifest.version` 或 `providedVersion`，函数会立即返回，无需 await。
   - 这本身不是 bug，但调用方必须始终 await，否则在 `installPath` 分支会拿到 Promise 而非字符串。

### 边界情况

- **空 `installPath` + 无 `manifest.version` + 无 `providedVersion` + 无 `gitCommitSha`** → 返回 `'unknown'`。
- **`gitCommitSha` 长度处理**：`substring(0, 12)` 取前 12 位，即使传入的是短 SHA（如 7 位）也不会报错，只是结果更短。
- **路径尾部斜杠**：`git-subdir` 的规范化会去掉所有尾部 `/`，确保 `./foo/` 和 `./foo` 生成相同的哈希。

### 改进建议

1. **修复 Windows 路径兼容性**
   - `getVersionFromPath` 应在分割前统一将反斜杠替换为正斜杠：
     ```ts
     const normalizedPath = installPath.replace(/\\/g, '/')
     ```
   - 这是低风险高回报的修复，建议立即实施。

2. **减少 `'unknown'` 的使用**
   - 对于本地目录来源（`directory` / `file`），若无法获取 git SHA，可考虑使用目录内容的哈希（如 `mtime` + 文件列表的摘要）或时间戳，避免多个版本共享 `unknown`。
   - 但需注意性能：计算目录哈希可能很昂贵，应在安装时一次性完成并持久化到 `installed_plugins.json`。

3. **提取路径规范化常量**
   - `git-subdir` 的规范化逻辑（`replace(/\\/g, '/')` 等）应与后端共享一个规范文档或测试用例，防止未来漂移。可考虑在 CI 中增加一个跨语言的一致性测试。

4. **版本比较工具函数**
   - 当前模块只负责“计算版本字符串”，但更新检测需要比较两个版本。对于 semver 可直接用字符串比较或 semver 库；对于 `git-subdir` 的 `{sha}-{hash}` 格式，需要比较 SHA 部分。建议在本模块增加 `isNewerVersion(oldVersion, newVersion)` 或 `versionEquals(a, b)` 工具函数，统一更新检测逻辑。
