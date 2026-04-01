# officialMarketplaceGcs.ts 深度研究文档

## 场景与职责

`officialMarketplaceGcs.ts` 实现了从 **Google Cloud Storage (GCS) 镜像** 获取官方插件市场的功能。这是 INC-5046 的一部分，旨在避免每次启动都从 GitHub git-clone，提升启动速度并减少对 GitHub 的依赖。

### 核心职责

1. **GCS 镜像获取**：从 `downloads.claude.ai` 获取官方市场的 zip 包
2. **版本检测**：通过 `latest` 指针检查是否有新版本
3. **原子更新**：使用 staging + rename 实现原子替换
4. **权限保留**：解压时恢复 Unix 可执行权限
5. **遥测记录**：记录 fetch 事件用于监控

### 在系统架构中的位置

```
插件市场层
    ├── officialMarketplace.ts           # 常量定义
    ├── officialMarketplaceGcs.ts        # ← 本文件：GCS 获取
    │   └── fetchOfficialMarketplaceFromGcs()
    ├── officialMarketplaceStartupCheck.ts # 启动时调用 GCS 获取
    └── marketplaceManager.ts            # 备选：git 方式
```

---

## 功能点目的

### 1. GCS 获取流程 (`fetchOfficialMarketplaceFromGcs`)

```
1. 安全检查：验证 installLocation 在 marketplacesCacheDir 内
2. 等待滚动空闲：避免与 UI 渲染竞争
3. 获取 latest 指针：~40 字节，Cache-Control: max-age=300
4. 检查本地 sentinel：.gcs-sha 文件记录当前版本
5. 如果版本匹配 → 返回 noop
6. 下载 zip 包：~3.5MB，60秒超时
7. 解压到 staging 目录
8. 写入新的 .gcs-sha
9. 原子替换：rm old → rename staging
10. 记录遥测事件
```

### 2. 安全机制

- **路径验证**：确保 `installLocation` 在 `marketplacesCacheDir` 内，防止误删用户文件
- **staging 策略**：先解压到 `.staging` 目录，成功后再原子替换，避免中途失败留下损坏状态

### 3. 权限保留

- 解析 ZIP 的 central directory 获取 Unix 模式位
- 对可执行文件（mode & 0o111）恢复权限
- 失败静默处理（EPERM/ENOTSUP），保持兼容性

### 4. 错误分类 (`classifyGcsError`)

将错误分类为可监控的类别：

| 类别 | 说明 |
|------|------|
| `timeout` | 请求超时 |
| `http_{status}` | HTTP 错误状态码 |
| `network` | 网络错误（无响应） |
| `fs_{code}` | 文件系统错误（ENOSPC, EACCES 等） |
| `fs_other` | 其他文件系统错误 |
| `zip_parse` | ZIP 解析错误 |
| `empty_latest` | latest 指针为空 |
| `other` | 其他未知错误 |

---

## 具体技术实现

### 关键常量

```typescript
// GCS 基础 URL（CDN 前端）
const GCS_BASE = 'https://downloads.claude.ai/claude-code-releases/plugins/claude-plugins-official'

// ZIP 包内路径前缀（与 titanium seed 共享结构）
const ARC_PREFIX = 'marketplaces/claude-plugins-official/'
```

### 核心流程详解

#### 1. 安全检查

```typescript
const cacheDir = resolve(marketplacesCacheDir)
const resolvedLoc = resolve(installLocation)
if (resolvedLoc !== cacheDir && !resolvedLoc.startsWith(cacheDir + sep)) {
  // 拒绝路径在缓存目录外的请求
  return null
}
```

#### 2. 版本检测

```typescript
// 获取 latest 指针
const latest = await axios.get(`${GCS_BASE}/latest`, {
  responseType: 'text',
  timeout: 10_000,
})
const sha = String(latest.data).trim()

// 检查本地 sentinel
const sentinelPath = join(installLocation, '.gcs-sha')
const currentSha = await readFile(sentinelPath, 'utf8').then(s => s.trim(), () => null)

if (currentSha === sha) {
  return sha  // 无需更新
}
```

#### 3. 下载与解压

```typescript
// 下载
const zipResp = await axios.get(`${GCS_BASE}/${sha}.zip`, {
  responseType: 'arraybuffer',
  timeout: 60_000,
})
const zipBuf = Buffer.from(zipResp.data)

// 解压
const files = await unzipFile(zipBuf)
const modes = parseZipModes(zipBuf)  // 获取权限

// 写入 staging
const staging = `${installLocation}.staging`
await rm(staging, { recursive: true, force: true })
await mkdir(staging, { recursive: true })

for (const [arcPath, data] of Object.entries(files)) {
  if (!arcPath.startsWith(ARC_PREFIX)) continue
  const rel = arcPath.slice(ARC_PREFIX.length)
  const dest = join(staging, rel)
  await mkdir(dirname(dest), { recursive: true })
  await writeFile(dest, data)
  
  // 恢复可执行权限
  const mode = modes[arcPath]
  if (mode && mode & 0o111) {
    await chmod(dest, mode & 0o777).catch(() => {})
  }
}

// 写入 sentinel
await writeFile(join(staging, '.gcs-sha'), sha)
```

#### 4. 原子替换

```typescript
await rm(installLocation, { recursive: true, force: true })
await rename(staging, installLocation)
```

#### 5. 遥测

```typescript
logEvent('tengu_plugin_remote_fetch', {
  source: 'marketplace_gcs',
  host: 'downloads.claude.ai',
  is_official: true,
  outcome: 'updated' | 'noop' | 'failed',
  duration_ms: number,
  bytes?: number,
  sha?: string,
  error_kind?: string,
})
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `fetchOfficialMarketplaceFromGcs` | 主入口：从 GCS 获取官方市场 | `officialMarketplaceStartupCheck.ts` |
| `classifyGcsError` | 错误分类（也用于遥测） | 内部使用 |

### 调用关系图

```
officialMarketplaceStartupCheck.ts
    └── checkAndInstallOfficialMarketplace()
        └── fetchOfficialMarketplaceFromGcs(installLocation, cacheDir)
            ├── axios.get(latest)      # 获取版本指针
            ├── readFile(.gcs-sha)     # 检查本地版本
            ├── axios.get({sha}.zip)   # 下载 zip
            ├── unzipFile()            # utils/dxt/zip.ts
            ├── parseZipModes()        # utils/dxt/zip.ts
            ├── rm() / mkdir() / writeFile() / chmod()  # 解压到 staging
            ├── rename()               # 原子替换
            └── logEvent()             # 遥测

marketplaceManager.ts (潜在使用)
    └── 可能作为刷新机制的一部分
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `utils/dxt/zip.ts` | ZIP 解压 (`unzipFile`, `parseZipModes`) |
| `utils/debug.ts` | 调试日志 (`logForDebugging`) |
| `utils/errors.ts` | 错误处理 (`errorMessage`, `getErrnoCode`) |
| `bootstrap/state.ts` | 等待滚动空闲 (`waitForScrollIdle`) |
| `services/analytics/index.ts` | 遥测 (`logEvent`) |

### 外部包依赖

| 包 | 用途 |
|----|------|
| `axios` | HTTP 请求 |

### 环境变量

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | 种子目录（间接相关） |

---

## 风险、边界与改进建议

### 已知风险

1. **GCS 服务不可用**
   - 风险：CDN 或 GCS 故障导致获取失败
   - 缓解：调用方（`officialMarketplaceStartupCheck.ts`）会回退到 git 方式

2. **zip 包损坏**
   - 风险：下载过程中网络中断导致文件损坏
   - 缓解：`unzipFile` 会验证 ZIP 完整性，失败时返回 null

3. **磁盘空间不足**
   - 风险：解压需要额外磁盘空间（~3.5MB zip → ~?MB 解压）
   - 现状：无显式检查，依赖文件系统错误处理

4. **并发更新冲突**
   - 风险：多个进程同时更新同一目录
   - 缓解：原子 rename 操作，但 rm + rename 非原子，可能短暂不存在

5. **权限恢复失败**
   - 风险：某些文件系统（NFS root_squash, FUSE）不支持 chmod
   - 缓解：失败静默处理，保持兼容性

### 边界情况

| 场景 | 行为 |
|------|------|
| latest 指针为空 | 抛出错误，返回 null |
| 本地无 .gcs-sha | 视为首次获取，继续下载 |
| 下载超时 | 返回 null，记录 timeout 错误 |
| 解压失败 | 返回 null，记录 zip_parse 错误 |
| staging 目录已存在 | 先删除再创建（force: true） |
| rename 失败 | 抛出错误，staging 目录保留供排查 |
| 路径在缓存目录外 | 拒绝操作，返回 null |

### 改进建议

1. **断点续传**
   - 建议：支持 Range 请求，大文件下载中断可恢复

2. **磁盘空间检查**
   - 建议：下载前检查可用空间，避免半拉子文件

3. **并发控制**
   - 建议：添加文件锁，防止多进程同时更新

4. **校验和验证**
   - 建议：下载后验证 SHA256，确保完整性

5. **备份机制**
   - 建议：更新前备份旧版本，失败可回滚

6. **增量更新**
   - 建议：支持 diff 更新，减少下载量

---

## 测试要点

1. **版本检测**：验证 latest 指针读取和 sentinel 比较
2. **缓存命中**：验证相同版本跳过下载
3. **解压正确性**：验证文件内容和权限
4. **原子替换**：验证更新过程中不会读取到半拉子状态
5. **错误处理**：验证各种错误场景的正确分类和处理
6. **安全检查**：验证路径 outside 缓存目录被拒绝
