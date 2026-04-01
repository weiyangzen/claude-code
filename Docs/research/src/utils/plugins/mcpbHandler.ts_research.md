# mcpbHandler.ts 深度研究文档

## 场景与职责

`mcpbHandler.ts` 是 Claude Code 插件系统中负责 **MCP Bundle (MCPB)** 文件处理的核心模块。MCPB 是一种打包格式，包含 MCP 服务器的完整运行时（代码、配置、manifest），支持从远程 URL 或本地路径加载。

### 核心职责

1. **MCPB 文件下载与缓存**：从 URL 下载 MCPB 文件，本地文件直接读取
2. **内容验证与解压**：验证 ZIP 格式，解压到缓存目录，保留可执行权限
3. **Manifest 解析**：解析并验证 `manifest.json`，生成 MCP 服务器配置
4. **用户配置管理**：处理 `user_config` 的加载、验证、保存（支持敏感值分离存储）
5. **缓存管理**：基于内容 hash 的缓存，支持本地文件的 mtime 检测更新

### 在系统架构中的位置

```
插件层 (Plugin Layer)
    ├── mcpPluginIntegration.ts  # MCP 配置集成
    │   └── loadMcpServersFromMcpb()
    │       └── loadMcpbFile()   # ← 本文件主要入口
    └── mcpbHandler.ts           # ← 本文件

工具层
    └── utils/dxt/
        ├── zip.ts               # ZIP 解压和权限解析
        └── helpers.ts           # Manifest 验证
```

---

## 功能点目的

### 1. MCPB 文件识别 (`isMcpbSource`)

简单通过后缀名识别：`.mcpb` 或 `.dxt`

### 2. 智能缓存系统

```
缓存目录: {pluginPath}/.mcpb-cache/
    ├── {sourceHash}.metadata.json  # 元数据（source, hash, extractedPath, cachedAt）
    ├── {sourceHash}.mcpb           # 下载的原始文件（URL 来源）
    └── {contentHash}/              # 解压后的内容
        └── manifest.json
```

缓存策略：
- **URL 来源**：基于内容 hash，不自动更新（需显式刷新）
- **本地文件**：基于 mtime 检测变更

### 3. 用户配置分离存储

| 配置类型 | 存储位置 | 用途 |
|----------|----------|------|
| 非敏感值 | `settings.json` `pluginConfigs[pluginId].mcpServers[serverName]` | 普通配置项 |
| 敏感值 | Keychain / `.credentials.json` `pluginSecrets[{pluginId}/{serverName}]` | API 密钥、令牌等 |

### 4. 安全特性

- **路径遍历防护**：ZIP 解压时验证路径安全
- **可执行权限保留**：解析 ZIP 的 Unix 模式位，恢复 `+x` 权限
- **敏感值清理**：保存时自动从错误位置清理敏感值（schema 变更时）

---

## 具体技术实现

### 关键数据结构

```typescript
// MCPB 加载成功结果
export type McpbLoadResult = {
  manifest: McpbManifest           // DXT manifest
  mcpConfig: McpServerConfig       // 生成的 MCP 配置
  extractedPath: string            // 解压路径
  contentHash: string              // 内容 SHA256 前16位
}

// 需要用户配置的结果
export type McpbNeedsConfigResult = {
  status: 'needs-config'
  manifest: McpbManifest
  extractedPath: string
  contentHash: string
  configSchema: UserConfigSchema   // 配置项 schema
  existingConfig: UserConfigValues // 已保存的配置
  validationErrors: string[]       // 验证错误
}

// 缓存元数据
type McpbCacheMetadata = {
  source: string
  contentHash: string
  extractedPath: string
  cachedAt: string      // ISO 时间戳
  lastChecked: string   // ISO 时间戳
}

// 用户配置值类型
export type UserConfigValues = Record<string, string | number | boolean | string[]>

// 用户配置 schema（来自 DXT manifest）
export type UserConfigSchema = Record<string, McpbUserConfigurationOption>
```

### 核心流程

#### 1. MCPB 加载流程 (`loadMcpbFile`)

```
loadMcpbFile(source, pluginPath, pluginId, onProgress?, providedUserConfig?, forceConfigDialog?)
    │
    ├── 检查缓存 (loadCacheMetadata)
    │   └── checkMcpbChanged()  # 本地文件检查 mtime
    │
    ├── 缓存命中且未变更
    │   ├── 加载 manifest.json
    │   ├── 检查 user_config
    │   │   ├── 需要配置 → 返回 McpbNeedsConfigResult
    │   │   └── 有配置或不需要 → 生成 MCP 配置 → 返回 McpbLoadResult
    │   └── 返回结果
    │
    └── 缓存未命中或已变更
        ├── 下载/读取文件
        │   ├── URL → downloadMcpb() → 保存到缓存目录
        │   └── 本地路径 → readFileBytes()
        ├── 生成 contentHash (SHA256)
        ├── 解压 ZIP (unzipFile)
        ├── 解析 Unix 权限 (parseZipModes)
        ├── 提取文件到缓存目录 (extractMcpbContents)
        │   └── 保留可执行权限 (chmod)
        ├── 解析 manifest.json
        ├── 检查 user_config
        │   ├── 需要配置 → 保存缓存元数据 → 返回 McpbNeedsConfigResult
        │   └── 有配置或不需要 → 生成 MCP 配置
        ├── 保存缓存元数据
        └── 返回 McpbLoadResult
```

#### 2. 用户配置加载 (`loadMcpServerUserConfig`)

```typescript
// 合并两个来源的配置
const nonSensitive = settings.pluginConfigs?.[pluginId]?.mcpServers?.[serverName]
const sensitive = secureStorage.pluginSecrets?.[`${pluginId}/${serverName}`]
return { ...nonSensitive, ...sensitive }  // 敏感值优先级更高
```

#### 3. 用户配置保存 (`saveMcpServerUserConfig`)

```
saveMcpServerUserConfig(pluginId, serverName, config, schema)
    │
    ├── 按 sensitive 标记分离配置
    │   ├── sensitive: true → sensitive 对象
    │   └── 其他 → nonSensitive 对象
    │
    ├── 准备清理集合
    │   ├── sensitiveKeysInThisSave: 本次保存的敏感 key
    │   └── nonSensitiveKeysInThisSave: 本次保存的非敏感 key
    │
    ├── 先写 secureStorage（失败则抛出，不碰 settings.json）
    │   ├── 读取现有敏感配置
    │   ├── 清理其中非敏感的 key（schema 变更处理）
    │   └── 写入合并后的配置
    │
    └── 再写 settings.json
        ├── 读取现有配置
        ├── 清理其中敏感的 key（通过设为 undefined）
        └── 写入合并后的配置
```

#### 4. 配置验证 (`validateUserConfig`)

支持多种类型验证：
- `string` / `number` / `boolean` / `file` / `directory`
- `multiple: true` 支持字符串数组
- `required` 必填检查
- `min` / `max` 数值范围

### 下载与解压实现

#### 下载 (`downloadMcpb`)

```typescript
// 使用 axios，配置：
// - timeout: 120000ms (2分钟)
// - responseType: 'arraybuffer'
// - maxRedirects: 5
// - onDownloadProgress: 进度回调

// 错误分类：
// - 网络错误、超时、HTTP 错误状态码
// - 通过 classifyFetchError 分类用于遥测
```

#### 解压与权限 (`extractMcpbContents`)

```typescript
// 1. 创建解压目录
// 2. 遍历 ZIP 内容（过滤目录条目）
// 3. 自动创建父目录
// 4. 判断文本/二进制文件（通过后缀）
// 5. 写入文件
// 6. 恢复可执行权限（如果 mode & 0o111）
//    - 失败静默处理（EPERM/ENOTSUP）
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `isMcpbSource` | 判断是否为 MCPB 文件 | `mcpPluginIntegration.ts` |
| `loadMcpbFile` | 主入口：加载 MCPB 文件 | `mcpPluginIntegration.ts` |
| `loadMcpServerUserConfig` | 加载用户配置 | `mcpPluginIntegration.ts`, `ManagePlugins.tsx` |
| `saveMcpServerUserConfig` | 保存用户配置 | `PluginOptionsFlow.tsx` |
| `validateUserConfig` | 验证配置值 | `mcpPluginIntegration.ts`, `PluginOptionsFlow.tsx` |
| `checkMcpbChanged` | 检查 MCPB 是否变更 | `loadMcpbFile`（内部） |

### 调用关系图

```
mcpPluginIntegration.ts
    ├── isMcpbSource()
    └── loadMcpServersFromMcpb()
        └── loadMcpbFile()  ← 主入口
            ├── loadCacheMetadata()
            ├── checkMcpbChanged()
            │   └── isUrl()
            ├── downloadMcpb()  # URL 来源
            │   └── logPluginFetch()  # 遥测
            ├── unzipFile()  # utils/dxt/zip.ts
            ├── parseZipModes()  # utils/dxt/zip.ts
            ├── extractMcpbContents()
            ├── parseAndValidateManifestFromBytes()  # utils/dxt/helpers.ts
            ├── loadMcpServerUserConfig()  # 读取现有配置
            ├── validateUserConfig()
            ├── saveMcpServerUserConfig()  # 保存新配置
            └── generateMcpConfig()  # 生成 MCP 配置
                └── @anthropic-ai/mcpb.getMcpConfigForManifest()

commands/plugin/PluginOptionsFlow.tsx
    ├── loadMcpServerUserConfig()
    ├── validateUserConfig()
    └── saveMcpServerUserConfig()

commands/plugin/ManagePlugins.tsx
    └── loadMcpServerUserConfig()

utils/plugins/pluginOptionsStorage.ts
    └── validateUserConfig()  # 复用验证逻辑
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `utils/dxt/zip.ts` | ZIP 解压 (`unzipFile`, `parseZipModes`) |
| `utils/dxt/helpers.ts` | Manifest 验证 (`parseAndValidateManifestFromBytes`) |
| `utils/secureStorage/index.ts` | 敏感配置存储 (`getSecureStorage`) |
| `utils/settings/settings.ts` | 非敏感配置存储 (`getSettings_DEPRECATED`, `updateSettingsForSource`) |
| `utils/systemDirectories.ts` | 系统目录 (`getSystemDirectories`) |
| `utils/fsOperations.ts` | 文件系统抽象 (`getFsImplementation`) |
| `utils/errors.ts` | 错误处理工具 |
| `utils/debug.ts` | 调试日志 (`logForDebugging`) |
| `utils/log.ts` | 错误日志 (`logError`) |
| `utils/slowOperations.ts` | JSON 解析/序列化 |
| `utils/plugins/fetchTelemetry.ts` | 下载遥测 (`logPluginFetch`, `classifyFetchError`) |

### 外部包依赖

| 包 | 用途 |
|----|------|
| `@anthropic-ai/mcpb` | DXT/MCPB manifest 类型、验证 schema、配置生成 |
| `axios` | HTTP 下载 |
| `fflate` | ZIP 解压（通过 zip.ts 间接使用） |

### 存储键设计

```typescript
// 服务器密钥格式：pluginId/serverName
// pluginId 格式: "pluginName@marketplace"
// serverName 来自 DXT manifest.name

function serverSecretsKey(pluginId: string, serverName: string): string {
  return `${pluginId}/${serverName}`
}
// 示例: "my-plugin@official/claude-plugins-official/my-server"
```

---

## 风险、边界与改进建议

### 已知风险

1. **ZIP 炸弹攻击**
   - 风险：恶意 MCPB 文件可能是 ZIP 炸弹
   - 缓解：`unzipFile` 在 `utils/dxt/zip.ts` 中实现了多重检查：
     - 单文件大小限制 (512MB)
     - 总解压大小限制 (1GB)
     - 文件数量限制 (10万)
     - 压缩比限制 (50:1)

2. **路径遍历攻击**
   - 风险：ZIP 中的 `../../../etc/passwd` 路径
   - 缓解：`isPathSafe` 函数检查路径遍历模式

3. **敏感值泄露到 settings.json**
   - 风险：配置 schema 变更（sensitive 标记变化）导致敏感值存储位置错误
   - 缓解：`saveMcpServerUserConfig` 实现了双向清理：
     - 保存敏感值时清理 secureStorage 中的非敏感值
     - 保存非敏感值时清理 settings.json 中的敏感值

4. **并发下载问题**
   - 风险：同一 MCPB 并发下载可能导致文件损坏
   - 现状：无显式锁机制，依赖文件系统原子操作

5. **macOS keychain 性能**
   - 风险：`loadMcpServerUserConfig` 每次调用都读取 keychain (~50-100ms)
   - 缓解：调用方（如 `mcpPluginIntegration.ts`）使用 memoize 缓存

### 边界情况

| 场景 | 行为 |
|------|------|
| 缓存目录被手动删除 | `checkMcpbChanged` 检测到 `ENOENT`，重新下载/解压 |
| 本地 MCPB 文件被修改 | mtime 检测触发重新加载 |
| user_config 验证失败 | 返回 `needs-config` 状态，不生成 MCP 配置 |
| 敏感配置写入失败 | 抛出错误，不修改 settings.json（保持旧值） |
| ZIP 权限恢复失败 | 静默忽略（EPERM/ENOTSUP），继续解压 |
| 空 user_config | 跳过配置验证，直接生成 MCP 配置 |

### 改进建议

1. **并发控制**
   - 建议：添加文件锁或内存锁，防止同一 MCPB 并发下载/解压

2. **缓存清理策略**
   - 当前：无自动清理，缓存无限增长
   - 建议：添加 LRU 清理或基于时间的过期策略

3. **下载恢复**
   - 当前：下载失败需从头开始
   - 建议：支持 Range 请求，实现断点续传

4. **配置迁移**
   - 当前：schema 变更时清理旧值，但不迁移数据
   - 建议：支持配置版本和自动迁移

5. **遥测增强**
   - 当前：仅记录下载事件
   - 建议：添加缓存命中率、解压时间等指标

6. **Manifest 验证优化**
   - 当前：每次加载都验证 manifest
   - 建议：缓存验证结果，hash 不变跳过验证

---

## 测试要点

1. **缓存机制**：验证缓存命中、失效、重新加载
2. **用户配置分离**：验证敏感/非敏感值的正确分离和清理
3. **权限保留**：验证 Unix 可执行权限正确恢复
4. **错误恢复**：验证下载失败、解压失败、验证失败的错误处理
5. **配置验证**：验证各种类型、必填、范围的验证逻辑
6. **并发安全**：验证多线程/进程环境下的文件操作安全
