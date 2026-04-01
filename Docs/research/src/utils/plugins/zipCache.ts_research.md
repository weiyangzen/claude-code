# 研究文档：src/utils/plugins/zipCache.ts

## 场景与职责

`zipCache.ts` 是 Claude Code **Headless / CCR（容器化、无交互）模式**下的插件存储核心模块。其设计目标是在容器生命周期短、本地磁盘易失的环境中，把插件以 **ZIP 归档**的形式持久化到挂载卷（如 Filestore / GCS 挂载点），并在每次会话启动时解压到本地临时目录供运行时使用。

主要职责包括：
1. **Zip Cache 目录管理**：根据环境变量定位挂载卷上的缓存根目录，维护 `known_marketplaces.json`、`installed_plugins.json`、`marketplaces/`、`plugins/` 四级结构。
2. **会话级临时解压目录（Session Plugin Cache）**：为当前会话创建唯一的本地临时目录（`tmpdir()` 下），在会话结束时清理。
3. **原子写文件**：对挂载卷上的元数据/缓存文件提供“先写临时文件再 rename”的原子写入，防止并发或中断导致半写文件。
4. **ZIP 创建与提取**：
   - `createZipFromDirectory`：把已下载/已克隆的插件目录打包成 ZIP，同时保留 Unix 可执行权限位（`+x`）。
   - `extractZipToDirectory`：把 ZIP 解压到会话临时目录，并恢复权限位。
5. **目录原地转 ZIP**：`convertDirectoryToZipInPlace` 提供“打包 → 原子写入 → 删目录”的标准序列，防止缓存损坏。
6. **来源类型白名单**：`isMarketplaceSourceSupportedByZipCache` 限定 Zip Cache 只支持 `github`/`git`/`url`/`settings` 四种 marketplace 来源，排除本地 `file`/`directory` 与 `npm`。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `isPluginZipCacheEnabled()` | 判断 `CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE` 是否为 truthy，作为整个 Zip Cache 特性的总开关。 |
| `getPluginZipCachePath()` | 返回展开后的 `CLAUDE_CODE_PLUGIN_CACHE_DIR`（支持 `~` 展开），若未启用则返回 `undefined`。 |
| `getZipCacheKnownMarketplacesPath()` / `getZipCacheInstalledPluginsPath()` / `getZipCacheMarketplacesDir()` / `getZipCachePluginsDir()` | 定位 Zip Cache 内部的标准子路径；若特性未启用会显式抛错，防止调用方在错误路径上操作。 |
| `getSessionPluginCachePath()` | 懒创建并缓存会话级临时目录（`claude-plugin-session-<random>`）。使用单例 Promise 模式防止并发创建多个目录。 |
| `cleanupSessionPluginCache()` / `resetSessionPluginCache()` | 会话结束时删除临时目录（供 `cleanupRegistry` 注册）或在测试中重置状态。 |
| `atomicWriteToZipCache(targetPath, data)` | 在同目录写入 `.target.tmp.<random>` 后 `rename` 成目标文件，失败时清理临时文件。 |
| `createZipFromDirectory(sourceDir)` | 递归遍历目录，把文件内容读入内存并生成 ZIP（`fflate.zipSync`，level 6）。 |
| `extractZipToDirectory(zipPath, targetDir)` | 调用 `unzipFile` 解压，再调用 `parseZipModes` 恢复 Unix mode，最后 `chmod`。 |
| `convertDirectoryToZipInPlace(dirPath, zipPath)` | 统一封装“创建 ZIP → 原子写入 → 删除原目录”流程，供安装/缓存路径调用。 |
| `getMarketplaceJsonRelativePath(name)` | 把 marketplace 名称中的非法字符替换为 `-`，生成 `marketplaces/{name}.json`。 |
| `isMarketplaceSourceSupportedByZipCache(source)` | 过滤 Zip Cache 不支持的 marketplace 来源（`file`/`directory`/`npm` 等）。 |

## 具体技术实现

### 1. 激活条件与环境变量
- 总开关：`CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE`（经 `isEnvTruthy` 判断）。
- 缓存根目录：`CLAUDE_CODE_PLUGIN_CACHE_DIR`（经 `expandTilde` 支持 `~`）。

### 2. 目录结构约定
```
${CLAUDE_CODE_PLUGIN_CACHE_DIR}/
  ├── known_marketplaces.json
  ├── installed_plugins.json
  ├── marketplaces/
  │   ├── official-marketplace.json
  │   └── company-marketplace.json
  └── plugins/
      ├── official-marketplace/
      │   └── plugin-a/
      │       └── 1.0.0.zip
      └── company-marketplace/
          └── plugin-b/
              └── 2.1.3.zip
```
该结构在 `headlessPluginInstall.ts` 的 `installPluginsForHeadless()` 中被显式 `mkdir` 创建。

### 3. 会话临时目录（Session Cache）
- 使用模块级变量 `sessionPluginCachePath` + `sessionPluginCachePromise` 实现懒加载与并发安全。
- 目录名：`join(tmpdir(), `claude-plugin-session-${randomBytes(8).toString('hex')}`)`。
- 清理由调用方通过 `registerCleanup(cleanupSessionPluginCache)` 在进程退出前执行。

### 4. 原子写入
- 临时文件名：`.${basename(targetPath)}.tmp.${randomBytes(4).toString('hex')}`。
- 写入后 `rename`（POSIX 语义下原子覆盖）。
- 失败时 `rm(tmpPath, { force: true })`  swallow 错误，避免残留。

### 5. ZIP 创建（权限保留与符号链接处理）
#### 5.1 数据结构
```ts
type ZipEntry = [Uint8Array, { os: number; attrs: number }]
```
- `os: 3` 表示 Unix 主机。
- `attrs: (fileStat.mode & 0xffff) << 16` 把 `st_mode` 放到 PKZIP `external_attr` 的高 16 位。

#### 5.2 符号链接策略
- `lstat` 识别符号链接。
- **目录符号链接**：跳过（`continue`），不跟随，防止循环。
- **文件符号链接**：用 `stat` 跟随到真实文件，把目标文件内容读入 ZIP； broken symlink 直接跳过。

#### 5.3 循环检测（Cycle Detection）
- 对**真实目录**（非符号链接目录）使用 `stat(currentDir, { bigint: true })` 获取 `dev: bigint` + `ino: bigint`。
- 生成键 `${dev}:${ino}` 存入 `visited`。
- 若 `dev === 0n && ino === 0n`（ReFS、NFS、FUSE 等常见情况），则**放弃循环检测**（fail open），避免误杀。
- 注释中明确提到 Windows NTFS 的 file index 会溢出 `Number.MAX_SAFE_INTEGER`，因此必须使用 `bigint: true`（参考 `anthropics/claude-code#13893`）。

#### 5.4 打包库
- `fflate` 采用**懒加载**（`await import('fflate')`），避免启动时分配其 ~196KB 的查找表。
- 使用 `zipSync(files, { level: 6 })` 同步压缩。

### 6. ZIP 提取（权限恢复）
- 先通过 `getFsImplementation().readFileBytes(zipPath)` 读取 Buffer。
- `unzipFile(zipBuf)`（来自 `src/utils/dxt/zip.ts`）返回 `Record<string, Uint8Array>`。
- `parseZipModes(zipBuf)`（同样来自 `zip.ts`）手动解析 ZIP Central Directory，提取 `external_attr` 高 16 位的 Unix mode。
- 对每条 entry：
  - 若路径以 `/` 结尾，视为目录，调用 `mkdir`。
  - 否则 `writeFile` 后，若 mode 包含任何执行位（`mode & 0o111`），则 `chmod(fullPath, mode & 0o777)`。
  - `chmod` 失败（`EPERM`/`ENOTSUP`，常见于 NFS root_squash 或某些 FUSE）被 swallow，保证解压不中断。

### 7. 目录原地转 ZIP
`convertDirectoryToZipInPlace` 是缓存路径的**关键事务**：
1. `createZipFromDirectory(dirPath)` → `Uint8Array`
2. `atomicWriteToZipCache(zipPath, zipData)`
3. `rm(dirPath, { recursive: true, force: true })`

该函数在以下两处被调用：
- `pluginLoader.ts` 的 `copyPluginToVersionedCache`（首次缓存外部插件时）。
- `pluginInstallationHelpers.ts` 的 `cacheAndRegisterPlugin`（安装流程最终归档）。

### 8. Marketplace 来源白名单
`isMarketplaceSourceSupportedByZipCache` 返回 `['github', 'git', 'url', 'settings'].includes(source.source)`。
- 被 `headlessPluginInstall.ts` 用于在 `reconcileMarketplaces` 时 `skip` 不支持的来源。
- 排除原因（代码注释）：
  - `file`/`directory`：installLocation 是用户本地路径，不在 cacheDir 内，对 ephemeral 容器无意义。
  - `npm`：会在 Filestore 挂载点上产生 `node_modules` 膨胀。

## 关键代码路径与文件引用

### 调用方（谁在使用 zipCache.ts）
| 调用文件 | 使用的导出符号 | 场景 |
|---------|---------------|------|
| `src/utils/plugins/pluginLoader.ts` | `convertDirectoryToZipInPlace`, `extractZipToDirectory`, `getSessionPluginCachePath`, `isPluginZipCacheEnabled` | 缓存插件时打包；加载插件时解压到会话目录。 |
| `src/utils/plugins/pluginInstallationHelpers.ts` | `convertDirectoryToZipInPlace`, `isPluginZipCacheEnabled` | `cacheAndRegisterPlugin` 安装最后一步把目录转 ZIP。 |
| `src/utils/plugins/headlessPluginInstall.ts` | `cleanupSessionPluginCache`, `getZipCacheMarketplacesDir`, `getZipCachePluginsDir`, `isMarketplaceSourceSupportedByZipCache`, `isPluginZipCacheEnabled` | Headless 安装流程初始化目录结构、注册清理回调、过滤不支持来源。 |
| `src/utils/plugins/cacheUtils.ts` | `isPluginZipCacheEnabled` | 判断是否在 Zip Cache 模式下跳过孤儿目录清理。 |
| `src/utils/plugins/zipCacheAdapters.ts` | `atomicWriteToZipCache`, `getMarketplaceJsonRelativePath`, `getPluginZipCachePath`, `getZipCacheKnownMarketplacesPath` | 元数据读写适配层。 |

### 被调用方（zipCache.ts 依赖谁）
| 依赖文件 | 使用的导出符号 | 作用 |
|---------|---------------|------|
| `src/utils/dxt/zip.ts` | `parseZipModes`, `unzipFile` | ZIP 解压与权限位解析。 |
| `src/utils/fsOperations.ts` | `getFsImplementation` | 抽象文件系统操作（便于测试/mock）。 |
| `src/utils/debug.ts` | `logForDebugging` | 调试日志。 |
| `src/utils/envUtils.ts` | `isEnvTruthy` | 环境变量 truthy 判断。 |
| `src/utils/permissions/pathValidation.ts` | `expandTilde` | `~` 展开。 |
| `src/utils/plugins/schemas.ts` | `MarketplaceSource` (type) | 类型约束。 |

### 核心流程时序
1. **安装/缓存**：`cacheAndRegisterPlugin` / `copyPluginToVersionedCache` → `convertDirectoryToZipInPlace` → `createZipFromDirectory` → `atomicWriteToZipCache` → `rm` 原目录。
2. **加载（完整路径）**：`loadPluginFromMarketplaceEntry` → 若 `pluginPath.endsWith('.zip')` → `getSessionPluginCachePath` → `extractZipToDirectory` → `finishLoadingPluginFromPath`。
3. **加载（Cache-Only 路径）**：`loadPluginFromMarketplaceEntryCacheOnly` → 同样调用 `extractZipToDirectory` 解压到会话目录。
4. **Headless 启动**：`installPluginsForHeadless` → `mkdir` Zip Cache 目录 → `reconcileMarketplaces` → `syncMarketplacesToZipCache` → `registerCleanup(cleanupSessionPluginCache)`。

## 依赖与外部交互

### 外部 NPM 依赖
- **`fflate`**：轻量级纯 JS ZIP 编解码库。仅在运行时懒加载，避免启动开销。

### Node.js 内置模块
- `crypto`：`randomBytes` 用于生成临时目录/文件后缀。
- `fs/promises`：`chmod`, `lstat`, `readdir`, `readFile`, `rename`, `rm`, `stat`, `writeFile`。
- `os`：`tmpdir()`。
- `path`：`basename`, `dirname`, `join`。

### 内部模块
- `zip.ts`：提供解压与权限解析。
- `fsOperations.ts`：提供可替换的 FS 抽象层。
- `debug.ts` / `envUtils.ts` / `permissions/pathValidation.ts`：通用工具。

## 风险、边界与改进建议

### 1. 权限恢复边界
- **风险**：`chmod` 在 NFS root_squash 或某些 FUSE 挂载点上会抛 `EPERM`/`ENOTSUP`。当前代码已 swallow 该错误，但会导致 hooks/scripts 失去 `+x`，可能使依赖可执行脚本的插件运行失败。
- **建议**：在 debug 日志中记录具体哪些文件丢失了执行权限，便于排查。

### 2. 符号链接与循环检测
- **风险**：`collectFilesForZip` 对目录符号链接采取“跳过”策略，若插件合法地使用了符号链接目录（如共享资源），该目录内容将不会被打包。
- **建议**：评估是否需要在严格模式外提供“跟随目录符号链接（带深度限制）”的选项。
- **已处理**：`bigint` dev/ino 循环检测已修复 Windows CI 上的精度问题，注释非常详尽。

### 3. 会话临时目录清理
- **风险**：`cleanupSessionPluginCache` 依赖进程正常退出时注册的 cleanup 回调。若进程被 `SIGKILL` 或容器被强制终止，临时目录会残留。
- **建议**：在启动时扫描并清理过期的 `claude-plugin-session-*` 目录（如 >24h）。

### 4. ZIP64 与大文件
- **风险**：`parseZipModes` 明确不处理 ZIP64（>4GB 或 >65535 entries），会返回空对象 `{}`。对于插件 ZIP 来说通常足够小（~3.5MB），但极端场景（超大 MCPB bundle）可能意外丢失权限。
- **建议**：在 `parseZipModes` 检测到 ZIP64 signature 时增加 warn 日志，提醒权限恢复被跳过。

### 5. 孤儿版本清理
- **风险**：`cacheUtils.ts` 的 `cleanupOrphanedPluginVersionsInBackground` 在 Zip Cache 模式下**完全跳过**，因为原实现基于“遍历子目录”假设，而 Zip Cache 把插件存为 `.zip` 文件。这会导致卸载/更新后的旧 ZIP 长期占用挂载卷空间。
- **建议**：为 Zip Cache 模式实现对应的 `.zip` 孤儿清理逻辑（检查 `installed_plugins.json` 中未引用的 `.zip` 文件并标记删除）。

### 6. 并发原子写入
- **风险**：`atomicWriteToZipCache` 使用同目录临时文件 + `rename`，在 POSIX 上是原子的，但在某些网络文件系统（如旧版 NFS）上 `rename` 可能不是真正的原子操作。
- **建议**：若未来在 NFS 上观察到半写文件，可考虑增加文件校验（如写入后读取前几个字节验证 ZIP magic number）。

### 7. 内存占用
- **风险**：`createZipFromDirectory` 把整个目录内容读入内存（`Record<string, ZipEntry>`）后再调用 `zipSync`。对于非常大的插件（如包含大量模型文件），可能导致 OOM。
- **建议**：评估插件大小上限，或在超过阈值时回退到流式 ZIP 库（如 `archiver`）并直接写入文件系统，而非全内存 `zipSync`。
