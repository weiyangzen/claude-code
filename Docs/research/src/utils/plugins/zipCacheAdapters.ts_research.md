# 研究文档：src/utils/plugins/zipCacheAdapters.ts

## 场景与职责

`zipCacheAdapters.ts` 是 `zipCache.ts` 的**I/O 适配层与元数据管理器**，专门负责在 Headless / Zip Cache 模式下读写挂载卷（如 Filestore）上的**离线元数据文件**。它的核心使命是让 ephemeral 容器在每次启动时，无需重新克隆 marketplace 仓库，也能获取到 marketplace 的清单（`marketplace.json`）与已注册 marketplace 的索引（`known_marketplaces.json`）。

具体职责：
1. **读写 `known_marketplaces.json`**：在 Zip Cache 目录中维护一个“离线版”的已知 marketplace 索引，与主配置（`~/.claude/plugins/known_marketplaces.json`）合并后供 ephemeral 容器使用。
2. **读写 marketplace JSON 缓存**：把每个 marketplace 的 `marketplace.json` 复制/保存到 Zip Cache 的 `marketplaces/` 子目录，实现离线可用。
3. **同步 marketplace 数据**：在 Headless 安装流程结束时，执行 `syncMarketplacesToZipCache`，将当前全局配置中的 marketplace 信息同步到 Zip Cache，并与之前已缓存的数据做合并。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `readZipCacheKnownMarketplaces()` | 从 Zip Cache 读取 `known_marketplaces.json`；若文件不存在、JSON 损坏或 schema 校验失败，安全降级返回 `{}`。 |
| `writeZipCacheKnownMarketplaces(data)` | 以原子写方式把 `known_marketplaces.json` 写入 Zip Cache。 |
| `readMarketplaceJson(marketplaceName)` | 从 Zip Cache 的 `marketplaces/{sanitized-name}.json` 读取单个 marketplace 清单；失败返回 `null`。 |
| `saveMarketplaceJsonToZipCache(name, installLocation)` | 从 marketplace 的本地安装位置（克隆后的目录或 URL 源的文件路径）读取 `marketplace.json` 内容，并原子写入 Zip Cache。 |
| `syncMarketplacesToZipCache()` | **主同步入口**：读取当前全局 `known_marketplaces.json`，把每个 entry 对应的 `marketplace.json` 拷贝到 Zip Cache，再与 Zip Cache 里已有的 `known_marketplaces.json` 做对象展开合并，最后写回。 |

## 具体技术实现

### 1. 元数据读取的防御性设计
#### 1.1 `readZipCacheKnownMarketplaces`
```ts
const parsed = KnownMarketplacesFileSchema().safeParse(jsonParse(content))
if (!parsed.success) {
  logForDebugging(`Invalid known_marketplaces.json in zip cache: ${parsed.error.message}`, { level: 'error' })
  return {}
}
```
- 采用 `safeParse` + 静默降级策略，因为数据来自**共享挂载卷**，其他容器可能正在写入或留下损坏文件。
- 任何读取/解析/校验异常都被 `catch` 吞掉并返回 `{}`，避免启动流程崩溃。

#### 1.2 `readMarketplaceJson`
- 先检查 `getPluginZipCachePath()`，若 Zip Cache 未启用直接返回 `null`。
- 使用 `getMarketplaceJsonRelativePath(name)` 生成相对路径：`marketplaces/{sanitized}.json`。
- 同样使用 `safeParse` + `PluginMarketplaceSchema()` 校验，失败返回 `null`。

### 2. Marketplace JSON 内容提取
#### 2.1 `readMarketplaceJsonContent(dir)`
这是一个私有辅助函数，用于从 marketplace 的本地安装位置提取 `marketplace.json` 的原始文本：
```ts
const candidates = [
  join(dir, '.claude-plugin', 'marketplace.json'),
  join(dir, 'marketplace.json'),
  dir, // For URL sources, installLocation IS the marketplace JSON file
]
```
- **目录来源**（`github`/`git` 克隆）：优先尝试 `.claude-plugin/marketplace.json`，再尝试根目录的 `marketplace.json`。
- **URL 来源**：`installLocation` 本身就是下载下来的 JSON 文件路径，因此 `dir` 作为第三个 candidate。
- 对每个 candidate 尝试 `readFile`，`ENOENT` 或 `EISDIR` 被 swallow，继续下一个；全部失败返回 `null`。

### 3. 同步流程 `syncMarketplacesToZipCache`
这是该模块**最关键的入口**，执行步骤如下：

#### 步骤 1：读取全局配置（安全版）
```ts
const knownMarketplaces = await loadKnownMarketplacesConfigSafe()
```
- 使用 `marketplaceManager.ts` 提供的 `Safe` 变体， corrupted config 不抛错而返回 `{}`。
- 注释说明：该函数在启动路径中运行，若抛错会级联到捕获 `loadAllPlugins` 失败的同一块 `try/catch` 中，因此必须安全。

#### 步骤 2：逐个保存 marketplace JSON
```ts
for (const [name, entry] of Object.entries(knownMarketplaces)) {
  if (!entry.installLocation) continue
  try {
    await saveMarketplaceJsonToZipCache(name, entry.installLocation)
  } catch (error) {
    logForDebugging(`Failed to save marketplace JSON for ${name}: ${error}`)
  }
}
```
- 对每个有 `installLocation` 的 marketplace，调用 `saveMarketplaceJsonToZipCache`。
- 单条失败被 `catch` 并记录 debug 日志，不影响其他 marketplace 的同步。

#### 步骤 3：合并并写回 Zip Cache 的 `known_marketplaces.json`
```ts
const zipCacheKnownMarketplaces = await readZipCacheKnownMarketplaces()
const mergedKnownMarketplaces: KnownMarketplacesFile = {
  ...zipCacheKnownMarketplaces,
  ...knownMarketplaces,
}
await writeZipCacheKnownMarketplaces(mergedKnownMarketplaces)
```
- **合并策略**：对象展开，`knownMarketplaces` 覆盖 `zipCacheKnownMarketplaces` 的同名键。
- **设计意图**：ephemeral 容器在每次启动时可能会丢失全局配置（如 `~/.claude/plugins/known_marketplaces.json` 不在持久化卷上），而 Zip Cache 目录在挂载卷上是持久的。通过合并，确保即使当前全局配置为空，之前已同步的 marketplace 记录仍然保留在 Zip Cache 中。

### 4. 原子写入复用
- `writeZipCacheKnownMarketplaces` 和 `saveMarketplaceJsonToZipCache` 都复用 `zipCache.ts` 提供的 `atomicWriteToZipCache`，保证对共享挂载卷的写入是原子的。

## 关键代码路径与文件引用

### 调用方
| 调用文件 | 使用的导出符号 | 场景 |
|---------|---------------|------|
| `src/utils/plugins/headlessPluginInstall.ts` | `syncMarketplacesToZipCache` | Headless 插件安装流程在 marketplace reconcile 完成后无条件调用，保证离线数据最新。 |

### 被调用方 / 依赖
| 依赖文件 | 使用的导出符号 | 作用 |
|---------|---------------|------|
| `src/utils/plugins/zipCache.ts` | `atomicWriteToZipCache`, `getMarketplaceJsonRelativePath`, `getPluginZipCachePath`, `getZipCacheKnownMarketplacesPath` | 原子写、路径生成、Zip Cache 根目录获取。 |
| `src/utils/plugins/marketplaceManager.ts` | `loadKnownMarketplacesConfigSafe` | 安全读取全局 `known_marketplaces.json`。 |
| `src/utils/plugins/schemas.ts` | `KnownMarketplacesFileSchema`, `PluginMarketplaceSchema`, 以及对应类型 | JSON Schema 校验。 |
| `src/utils/slowOperations.ts` | `jsonParse`, `jsonStringify` | 带慢操作检测的 JSON 解析/序列化。 |
| `src/utils/debug.ts` | `logForDebugging` | 调试日志。 |

### 数据流时序
```
headlessPluginInstall.ts
  └── syncMarketplacesToZipCache()
        ├── loadKnownMarketplacesConfigSafe()  [读取全局配置]
        ├── saveMarketplaceJsonToZipCache()    [把每个 marketplace.json 拷贝到 Zip Cache]
        │     └── readMarketplaceJsonContent() [从 installLocation 提取 JSON 文本]
        ├── readZipCacheKnownMarketplaces()    [读取旧 Zip Cache 索引]
        └── writeZipCacheKnownMarketplaces()   [合并后原子写回]
              └── atomicWriteToZipCache()      [来自 zipCache.ts]
```

## 依赖与外部交互

### 外部 NPM 依赖
- 无直接外部依赖（仅通过 `slowOperations.ts` 间接使用标准 JSON 操作）。

### Node.js 内置模块
- `fs/promises`：`readFile`。
- `path`：`join`。

### 内部模块交互
- **`zipCache.ts`**：提供底层原子写与路径计算。
- **`marketplaceManager.ts`**：提供全局 marketplace 配置的安全读取。
- **`schemas.ts`**：提供 Zod schema 进行运行时校验。
- **`slowOperations.ts`**：所有 JSON 操作都经过 `jsonParse` / `jsonStringify` 包装，便于慢操作监控。

## 风险、边界与改进建议

### 1. 合并策略导致“僵尸”Marketplace 残留
- **风险**：`syncMarketplacesToZipCache` 使用对象展开合并：
  ```ts
  { ...zipCacheKnownMarketplaces, ...knownMarketplaces }
  ```
  这意味着如果某个 marketplace 曾经存在过 Zip Cache 中，但之后用户从全局配置里**删除了**它，Zip Cache 中的旧记录不会被清除，因为 `knownMarketplaces` 不再包含该键，但展开操作不会主动删除旧键。
- **后果**：ephemeral 容器可能继续“看到”一个已经废弃的 marketplace，导致加载过时的插件清单或产生误导性的 marketplace 列表。
- **建议**：在合并前显式计算差集，删除 `zipCacheKnownMarketplaces` 中已不存在于 `knownMarketplaces` 的键，再写入。

### 2. 静默降级掩盖配置损坏
- **风险**：`readZipCacheKnownMarketplaces` 和 `loadKnownMarketplacesConfigSafe` 在 schema 校验失败时都返回 `{}`。这在启动路径上是合理的防御性设计，但会导致：
  - 用户无法感知挂载卷上的元数据已损坏。
  - 下一次 `syncMarketplacesToZipCache` 会用空对象覆盖旧的合并数据，可能**永久丢失**之前缓存的 marketplace 记录（虽然这些记录可以从远程重新克隆，但在离线场景下就是致命问题）。
- **建议**：
  - 在返回 `{}` 前增加 `logError`（而不仅是 `logForDebugging`），把损坏事件记录到错误日志，便于运维排查。
  - 考虑把损坏的 JSON 文件重命名为 `.corrupted.<timestamp>`，而不是让后续同步直接覆盖。

### 3. URL 来源的 `installLocation` 假设
- **风险**：`readMarketplaceJsonContent` 把 `dir`（即 `installLocation`）本身作为第三个 candidate 直接 `readFile`。
  这依赖于 `url` 来源的 marketplace 其 `installLocation` 是一个**文件路径**而非目录。若未来 `url` 来源的实现改为把文件下载到某个目录下并重新命名，`readMarketplaceJsonContent` 会读到目录导致 `EISDIR` 错误，从而返回 `null`。
- **建议**：在注释或文档中明确约定 `url` 来源的 `installLocation` 必须是文件路径；或在读取 `dir` 前先用 `stat` 确认其为文件。

### 4. 共享挂载卷的并发写入
- **风险**：多个 ephemeral 容器可能同时启动并调用 `syncMarketplacesToZipCache`，对同一个 `known_marketplaces.json` 执行原子写。虽然单文件的 `rename` 是原子的，但并发容器可能基于略微不同版本的旧文件进行合并，导致最终文件内容取决于 `rename` 的先后顺序（last-write-wins）。
- **建议**：
  - 对于绝大多数场景，这种竞态是可接受的，因为最终写入的数据都来自同一个全局配置源。
  - 若未来需要更强的一致性，可考虑引入文件锁（如 `proper-lockfile`）或基于对象存储的 CAS（compare-and-swap）机制。

### 5. `saveMarketplaceJsonToZipCache` 的单点失败
- **风险**：`syncMarketplacesToZipCache` 的循环中，单个 marketplace 的 `saveMarketplaceJsonToZipCache` 失败只会记录 debug 日志。如果失败的是用户最关心的 marketplace，且失败原因是挂载卷空间不足（`ENOSPC`），后续所有 marketplace 的保存也会失败，但日志里只会看到多条 debug 记录。
- **建议**：对 `ENOSPC`/`EACCES` 等系统性错误，在首次失败后提前中断循环并向上抛出/记录更醒目的错误，避免无意义的重复失败。

### 6. 缺少对 `marketplace.json` 变更的增量检测
- **风险**：每次 `syncMarketplacesToZipCache` 都会无条件读取并重新写入所有 marketplace JSON，即使内容完全没有变化。对于大量 marketplace 的场景，这会产生不必要的 I/O。
- **建议**：在写入前对比读取到的内容与目标文件现有内容（或对比 mtime/hash），仅在真正变化时执行原子写。
