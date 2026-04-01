# installCounts.ts 深度研究文档

## 场景与职责

`installCounts.ts` 实现 Claude Code 插件安装统计数据的获取与缓存层。该模块从官方 Claude 插件统计仓库获取安装量数据，用于在插件浏览/搜索 UI 中显示流行度指标。

核心场景：
1. **插件浏览排序**：按安装量排序插件列表
2. **安装量展示**：格式化显示（如 "1.2K", "36.2K"）
3. **离线支持**：24小时缓存确保弱网环境可用

## 功能点目的

### 1. 安装量数据获取 (`getInstallCounts`)
- **缓存优先**：先尝试加载本地缓存，24小时内有效
- **后台刷新**：缓存过期时异步从 GitHub 获取
- **容错设计**：任何错误返回 `null`，UI 隐藏计数而非显示误导性零值

### 2. 缓存管理
- **存储位置**：`~/.claude/plugins/install-counts-cache.json`
- **格式版本**：`INSTALL_COUNTS_CACHE_VERSION = 1`
- **TTL**：`CACHE_TTL_MS = 24 * 60 * 60 * 1000`（24小时）
- **原子写入**：临时文件 + rename 模式防止损坏

### 3. 遥测集成
- **缓存命中**：上报 `cache_hit` outcome
- **网络获取**：上报 `success` 或 `failure` + 错误分类
- **来源标记**：`install_counts` source 类型

### 4. 格式化显示 (`formatInstallCount`)
- `< 1000`：原始数字（如 "42"）
- `>= 1000`：K 后缀，1位小数（如 "1.2K", "36.2K"）
- `>= 1000000`：M 后缀，1位小数（如 "1.2M"）
- 去除末尾 `.0`（如 "1.0K" → "1K"）

## 具体技术实现

### 核心类型定义
```typescript
type InstallCountsCache = {
  version: number
  fetchedAt: string // ISO 时间戳
  counts: Array<{
    plugin: string // "pluginName@marketplace" 格式
    unique_installs: number
  }>
}

type GitHubStatsResponse = {
  plugins: Array<{
    plugin: string
    unique_installs: number
  }>
}
```

### 缓存加载算法 (`loadInstallCountsCache`)
```
loadInstallCountsCache()
  ├─ 读取缓存文件
  ├─ 基础结构验证（version, fetchedAt, counts 存在）
  ├─ 版本号匹配检查
  ├─ 类型验证（fetchedAt 为 string, counts 为 array）
  ├─ 时间戳有效性验证
  ├─ 条目结构验证（每个条目有 plugin 和 unique_installs）
  ├─ 新鲜度检查（< 24小时）
  └─ 返回验证后的缓存或 null
```

### 原子保存实现
```typescript
async function saveInstallCountsCache(cache: InstallCountsCache): Promise<void> {
  const cachePath = getInstallCountsCachePath()
  const tempPath = `${cachePath}.${randomBytes(8).toString('hex')}.tmp`
  
  try {
    // 确保目录存在
    await getFsImplementation().mkdir(getPluginsDirectory())
    
    // 写入临时文件（权限 0o600）
    await writeFile(tempPath, jsonStringify(cache, null, 2), {
      encoding: 'utf-8',
      mode: 0o600
    })
    
    // 原子重命名
    await rename(tempPath, cachePath)
  } catch (error) {
    // 清理临时文件
    try { await unlink(tempPath) } catch {}
    throw error
  }
}
```

### 数据获取流程
```typescript
async function fetchInstallCountsFromGitHub() {
  const started = performance.now()
  try {
    const response = await axios.get<GitHubStatsResponse>(INSTALL_COUNTS_URL, {
      timeout: 10000 // 10秒超时
    })
    
    if (!response.data?.plugins || !Array.isArray(response.data.plugins)) {
      throw new Error('Invalid response format')
    }
    
    logPluginFetch('install_counts', INSTALL_COUNTS_URL, 'success', duration)
    return response.data.plugins
  } catch (error) {
    logPluginFetch('install_counts', INSTALL_COUNTS_URL, 'failure', duration, classifyFetchError(error))
    throw error
  }
}
```

## 关键代码路径与文件引用

### 内部依赖
```
installCounts.ts
  ├─ axios: HTTP 客户端
  ├─ crypto.randomBytes: 临时文件随机后缀
  ├─ fs/promises: 文件操作
  ├─ path.join: 路径拼接
  ├─ debug.ts: logForDebugging
  ├─ errors.ts: errorMessage, getErrnoCode
  ├─ fsOperations.ts: getFsImplementation
  ├─ log.ts: logError
  ├─ slowOperations.ts: jsonParse, jsonStringify
  ├─ fetchTelemetry.ts: classifyFetchError, logPluginFetch
  └─ pluginDirectories.ts: getPluginsDirectory
```

### 外部调用方
| 调用方 | 用途 |
|--------|------|
| `marketplaceUI.ts` / `DiscoverPlugins.tsx` | 获取安装量用于排序和显示 |
| `pluginSearch.ts` | 搜索结果排序 |

### 调用链
```
UI 组件
  └─ getInstallCounts()
      ├─ 缓存命中路径
      │   ├─ loadInstallCountsCache()
      │   └─ logPluginFetch('install_counts', ..., 'cache_hit', 0)
      └─ 缓存未命中路径
          ├─ fetchInstallCountsFromGitHub()
          │   ├─ axios.get() → GitHub raw content
          │   └─ logPluginFetch(..., 'success'/'failure', ...)
          ├─ saveInstallCountsCache() [原子写入]
          └─ 返回 Map<pluginId, count>
```

## 依赖与外部交互

### 数据源
- **URL**：`https://raw.githubusercontent.com/anthropics/claude-plugins-official/refs/heads/stats/stats/plugin-installs.json`
- **格式**：JSON，包含插件 ID 和唯一安装数
- **更新频率**：由官方市场仓库定期生成

### 文件系统交互
- **读取**：`~/.claude/plugins/install-counts-cache.json`
- **写入**：临时文件 → 原子重命名
- **权限**：0o600（用户只读）

## 风险、边界与改进建议

### 已知风险

1. **GitHub Raw 内容可靠性**
   - 风险：GitHub 可能限流或变更 raw content URL 格式
   - 现状：10秒超时，失败时返回 null
   - 缓解：遥测监控失败率

2. **缓存损坏处理**
   - 风险：磁盘损坏或并发写入导致 JSON 无效
   - 现状：严格验证，无效时视为缓存未命中
   - 局限：丢失缓存但可恢复

3. **隐私考虑**
   - 风险：安装量数据可能暴露使用模式
   - 现状：数据来自公共仓库，无用户特定信息

4. **大列表性能**
   - 风险：插件数量增长导致 Map 构建变慢
   - 现状：O(n) 遍历，当前规模可忽略

### 边界条件

| 场景 | 行为 |
|------|------|
| 缓存文件不存在 | 视为缓存未命中，从网络获取 |
| 缓存版本不匹配 | 视为无效，从网络获取 |
| 缓存条目格式错误 | 整缓存视为无效 |
| 网络请求超时 | 记录错误，返回 null |
| 响应格式无效 | 抛出错误，返回 null |
| 计数为 0 | 正常显示 "0" |
| 负计数 | 正常格式化（如 "-1.2K"）|

### 改进建议

1. **备用数据源**
   - 当前：仅 GitHub raw
   - 建议：增加 GCS 镜像，提高可靠性

2. **增量更新**
   - 当前：全量替换
   - 建议：支持增量更新 API，减少带宽

3. **本地安装量统计**
   - 当前：仅官方统计
   - 建议：混合本地安装事件，提供更实时数据

4. **缓存预热**
   - 当前：首次请求触发获取
   - 建议：启动时后台预热，减少首次加载延迟

5. **格式扩展**
   - 当前：仅唯一安装数
   - 建议：增加周增长率、趋势指示等
