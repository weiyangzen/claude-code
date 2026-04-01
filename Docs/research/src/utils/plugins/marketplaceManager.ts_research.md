# MarketplaceManager 深度研究文档

> **目标文件**: `src/utils/plugins/marketplaceManager.ts`  
> **研究范围**: 代码实现、依赖关系、数据流、调用链路  
> **生成时间**: 2026-04-01

---

## 1. 场景与职责

### 1.1 核心定位

`marketplaceManager.ts` 是 Claude Code 插件系统的**市场源管理中枢**，负责：

1. **市场源生命周期管理**: 注册、更新、删除、刷新市场源（marketplace）
2. **本地缓存管理**: 将远程市场源（GitHub/URL/Git）缓存到本地文件系统
3. **配置持久化**: 维护 `known_marketplaces.json` 配置文件
4. **种子目录同步**: 支持从只读种子目录（seed dir）注册预置市场源
5. **策略合规检查**: 执行企业策略（allowlist/blocklist）控制市场源访问

### 1.2 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        调用方层 (Callers)                        │
├─────────────────────────────────────────────────────────────────┤
│  CLI handlers    UI Components    Plugin Operations    Reconciler│
│  (plugins.ts)    (ManageMarketplaces.tsx)              (reconciler.ts)│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              MarketplaceManager (本文件核心职责)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Source Mgmt │  │  Cache Layer │  │  Config Persistence  │  │
│  │  - add       │  │  - git clone │  │  - known_markets.json│  │
│  │  - remove    │  │  - url fetch │  │  - settings sync     │  │
│  │  - refresh   │  │  - validate  │  │  - seed registration │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        依赖层 (Dependencies)                     │
├─────────────────────────────────────────────────────────────────┤
│  schemas.ts      marketplaceHelpers.ts    officialMarketplace.ts │
│  pluginDirs.ts   installedPluginsManager  settings/settings.ts   │
│  fetchTelemetry.ts  officialMarketplaceGcs.ts                    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 文件系统结构

```
~/.claude/plugins/
├── known_marketplaces.json          # 市场源配置（本模块管理）
├── marketplaces/                    # 市场源缓存目录
│   ├── claude-plugins-official/     # GitHub源缓存（目录形式）
│   │   └── .claude-plugin/
│   │       └── marketplace.json
│   ├── my-marketplace.json          # URL源缓存（文件形式）
│   └── company-plugins/             # 其他Git源缓存
├── cache/                           # 插件安装缓存（pluginLoader管理）
│   └── {marketplace}/{plugin}/{version}/
├── installed_plugins.json           # 已安装插件元数据
└── data/                            # 插件持久化数据目录
```

---

## 2. 功能点目的

### 2.1 功能矩阵

| 功能类别 | 函数/方法 | 目的 | 关键特性 |
|---------|----------|------|---------|
| **声明管理** | `getDeclaredMarketplaces()` | 聚合所有来源的市场源声明（settings + --add-dir + 隐式官方源） | 三层优先级: implicit < --add-dir < settings |
| **配置读写** | `loadKnownMarketplacesConfig()` | 从磁盘加载市场源配置 | Zod schema 验证，损坏时抛出 ConfigParseError |
| | `saveKnownMarketplacesConfig()` | 保存配置到磁盘 | 原子写入，前置验证 |
| **种子同步** | `registerSeedMarketplaces()` | 从只读种子目录同步市场源 | admin-managed，autoUpdate=false，first-seed-wins |
| **市场源操作** | `addMarketplaceSource()` | 添加新市场源 | 策略检查、source-idempotency、种子保护 |
| | `removeMarketplaceSource()` | 移除市场源 | 清理缓存、级联删除插件、settings 同步 |
| | `refreshMarketplace()` | 刷新单个市场源 | GCS 优先（官方源）、SSH/HTTPS 回退、稀疏检出支持 |
| | `refreshAllMarketplaces()` | 批量刷新所有市场源 | 跳过种子源和 settings 源 |
| **数据获取** | `getMarketplace()` | 获取市场数据（带缓存） | memoized，缓存失效后自动重取 |
| | `getMarketplaceCacheOnly()` | 仅本地缓存读取 | 无网络阻塞，用于启动路径 |
| | `getPluginById()` | 通过 ID 获取插件条目 | 先缓存后网络，支持 `plugin@marketplace` 格式 |
| **Git 操作** | `gitClone()` | 克隆 Git 仓库 | 稀疏检出支持、凭据脱敏、错误增强 |
| | `gitPull()` | 更新 Git 仓库 | 子模块更新、超时控制、SSH/HTTPS 回退 |
| | `reconcileSparseCheckout()` | 稀疏检出状态协调 | 支持 Full→Sparse 转换 |
| **策略合规** | 通过 marketplaceHelpers.ts | 企业策略执行 | allowlist/blocklist/hostPattern |
| **自动更新** | `setMarketplaceAutoUpdate()` | 设置自动更新标志 | 种子源禁止修改 |

### 2.2 关键设计决策

#### 2.2.1 双层状态模型（Intent vs State）

```
┌─────────────────────────────────────────────────────────────┐
│  Intent Layer (声明层)                                       │
│  - settings.json: extraKnownMarketplaces                    │
│  - --add-dir 目录中的 settings                              │
│  - 隐式官方源（当启用官方插件时）                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼  reconciler.ts: diffMarketplaces()
┌─────────────────────────────────────────────────────────────┐
│  State Layer (状态层)                                        │
│  - known_marketplaces.json: 实际缓存位置和元数据              │
│  - marketplaces/: 本地缓存内容                               │
└─────────────────────────────────────────────────────────────┘
```

**设计目的**: 
- 支持版本控制：项目级 settings.json 可提交到 git
- 支持跨设备同步：声明随项目走，状态在每个设备上 materialize
- 支持意图覆盖：settings 中的声明可覆盖已缓存的市场源

#### 2.2.2 Source-Idempotency（源幂等性）

```typescript
// addMarketplaceSource() 中的幂等检查
for (const [existingName, existingEntry] of Object.entries(existingConfig)) {
  if (isEqual(existingEntry.source, resolvedSource)) {
    return { name: existingName, alreadyMaterialized: true, resolvedSource }
  }
}
```

**设计目的**: 避免重复克隆相同源，支持快速重试和并发安全。

#### 2.2.3 GCS 镜像优先（官方源）

```typescript
// refreshMarketplace() 中的官方源特殊处理
if (name === OFFICIAL_MARKETPLACE_NAME) {
  const sha = await fetchOfficialMarketplaceFromGcs(installLocation, cacheDir)
  if (sha !== null) return // GCS 成功，跳过 git
  // GCS 失败时根据 feature flag 决定是否回退到 git
}
```

**设计目的**: 减少 GitHub API 压力，提高启动速度。

---

## 3. 具体技术实现

### 3.1 关键数据类型

```typescript
// 来自 schemas.ts
interface KnownMarketplace {
  source: MarketplaceSource           // 源定义（github/url/git/file/directory/settings）
  installLocation: string             // 本地缓存路径
  lastUpdated: string                 // ISO 8601 时间戳
  autoUpdate?: boolean                // 是否自动更新
}

interface PluginMarketplace {
  name: string                        // 市场源名称（kebab-case）
  owner: PluginAuthor                 // 维护者信息
  plugins: PluginMarketplaceEntry[]   // 插件条目列表
  forceRemoveDeletedPlugins?: boolean // 是否强制删除已下架插件
  allowCrossMarketplaceDependenciesOn?: string[] // 跨市场依赖白名单
}

// 本文件定义的声明类型
type DeclaredMarketplace = {
  source: MarketplaceSource
  installLocation?: string
  autoUpdate?: boolean
  sourceIsFallback?: boolean  // 关键：隐式声明标记
}
```

### 3.2 市场源源类型（MarketplaceSource）

| 类型 | 结构 | 用途 | 缓存形式 |
|-----|------|------|---------|
| `github` | `{source:'github', repo:'owner/repo', ref?, path?, sparsePaths?}` | GitHub 仓库 | 目录克隆 |
| `git` | `{source:'git', url:'https://...', ref?, path?, sparsePaths?}` | 任意 Git URL | 目录克隆 |
| `url` | `{source:'url', url:'https://...', headers?}` | 直接 JSON URL | 单文件缓存 |
| `file` | `{source:'file', path:'/abs/path.json'}` | 本地 JSON 文件 | 引用原文件 |
| `directory` | `{source:'directory', path:'/abs/path'}` | 本地目录 | 引用原目录 |
| `settings` | `{source:'settings', name, plugins[], owner?}` | 内联声明 | 合成 marketplace.json |
| `npm` | `{source:'npm', package:'name'}` | NPM 包 | 未实现 |
| `hostPattern` | `{source:'hostPattern', hostPattern:'^github\\.corp\\.com$'}` | 策略匹配 | N/A（仅策略） |
| `pathPattern` | `{source:'pathPattern', pathPattern:'^/opt/approved/'}` | 路径策略匹配 | N/A（仅策略） |

### 3.3 核心流程

#### 3.3.1 添加市场源流程

```
addMarketplaceSource(source, onProgress)
│
├─► 解析本地路径（相对→绝对）
│
├─► 策略检查（isSourceAllowedByPolicy）
│   └─► 若在 blocklist → 抛错
│   └─► 若 strictKnownMarketplaces 存在且不匹配 → 抛错
│
├─► Source-Idempotency 检查
│   └─► 若源已存在 → 返回 alreadyMaterialized=true
│
├─► loadAndCacheMarketplace(source) ─────────────────────────────┐
│   │                                                            │
│   ├─► 根据源类型分发:                                          │
│   │   ├─ url → cacheMarketplaceFromUrl() → axios GET → 验证 schema
│   │   ├─ github → SSH/HTTPS 智能选择 → cacheMarketplaceFromGit()
│   │   ├─ git → cacheMarketplaceFromGit()
│   │   ├─ file/directory → 直接使用本地路径
│   │   └─ settings → 合成 marketplace.json 写入缓存
│   │                                                            │
│   ├─► 验证 marketplace.json schema (PluginMarketplaceSchema)   │
│   │                                                            │
│   └─► 重命名缓存目录为市场源实际名称（防御路径遍历）            │
│                                                                │
├─► 验证官方源名称保留（validateOfficialNameSource）
│   └─► 如 claude-plugins-official 必须来自 anthropics 组织
│
├─► 处理名称冲突（name collision）
│   └─► 若已存在且为种子源 → 抛错（admin-managed 保护）
│   └─► 若已存在且非种子 → 清理旧缓存，覆盖配置
│
├─► 更新 known_marketplaces.json
│
└─► 返回 {name, alreadyMaterialized, resolvedSource}
```

#### 3.3.2 Git 缓存流程（cacheMarketplaceFromGit）

```
cacheMarketplaceFromGit(gitUrl, cachePath, ref?, sparsePaths?, onProgress?)
│
├─► reconcileSparseCheckout() ──► 若需 Full→Sparse 转换则返回错误
│
├─► 尝试 gitPull()（增量更新）
│   ├─► 成功 → 完成
│   └─► 失败 → 继续以下步骤
│
├─► 清理旧目录（若存在）
│
├─► gitClone() ─────────────────────────────────────────────────┐
│   │                                                            │
│   ├─► 构建参数:                                                │
│   │   ├─ core.sshCommand=ssh -o BatchMode=yes -o StrictHostKeyChecking=yes
│   │   ├─ --depth 1（浅克隆）                                   │
│   │   ├─ --filter=blob:none --no-checkout（稀疏模式）          │
│   │   ├─ --recurse-submodules --shallow-submodules（完整模式） │
│   │   └─ --branch ref（若指定）                                │
│   │                                                            │
│   ├─► 执行克隆                                                 │
│   │                                                            │
│   ├─► 稀疏模式后处理:                                          │
│   │   ├─ git sparse-checkout set --cone $sparsePaths           │
│   │   └─ git checkout HEAD                                     │
│   │                                                            │
│   └─► 错误增强（超时、SSH 认证、主机密钥等）                    │
│                                                                │
├─► gitSubmoduleUpdate()（若非稀疏模式且有 .gitmodules）
│
└─► 验证 marketplace.json 存在
```

#### 3.3.3 SSH/HTTPS 智能回退逻辑

```typescript
// GitHub 源的特殊处理
const sshConfigured = await isGitHubSshLikelyConfigured()

if (sshConfigured) {
  try {
    await cacheMarketplaceFromGit(sshUrl, ...)
  } catch {
    // SSH 失败，回退到 HTTPS
    await cacheMarketplaceFromGit(httpsUrl, ...)
  }
} else {
  try {
    await cacheMarketplaceFromGit(httpsUrl, ...)
  } catch {
    // HTTPS 失败，回退到 SSH
    await cacheMarketplaceFromGit(sshUrl, ...)
  }
}
```

**SSH 配置检测** (`isGitHubSshLikelyConfigured`):
```typescript
// 使用 StrictHostKeyChecking=yes（非 accept-new）
// 未知主机失败闭合，防止 MITM 攻击
const result = await execFileNoThrow('ssh', [
  '-T', '-o', 'BatchMode=yes',
  '-o', 'ConnectTimeout=2',
  '-o', 'StrictHostKeyChecking=yes',
  'git@github.com'
])
// GitHub SSH 总是返回 exit code 1 但包含 "successfully authenticated"
return result.code === 1 && result.stderr?.includes('successfully authenticated')
```

### 3.4 错误处理与增强

| 错误场景 | 原始错误 | 增强后消息 |
|---------|---------|-----------|
| SSH 主机密钥变更 | `REMOTE HOST IDENTIFICATION HAS CHANGED` | "SSH host key for this marketplace's git host has changed... Remove the stale entry with: ssh-keygen -R <host>" |
| SSH 主机密钥验证失败 | `Host key verification failed` | "SSH host key verification failed... Connect once manually to add it..." |
| SSH 认证失败 | `Permission denied (publickey)` | "SSH authentication failed. Please ensure your SSH keys are configured..." |
| 超时 | `timed out` | "Git clone timed out after Xs... Set CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS to increase it" |
| HTTPS 认证失败 | `Authentication failed` | "HTTPS authentication failed. Please ensure your credential helper is configured..." |

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
marketplaceManager.ts
├── 直接导入
│   ├── schemas.ts                    # 所有 Zod schema 和类型定义
│   ├── marketplaceHelpers.ts         # 策略检查、源格式化、错误处理
│   ├── officialMarketplace.ts        # OFFICIAL_MARKETPLACE_NAME/SOURCE 常量
│   ├── officialMarketplaceGcs.ts     # GCS 镜像获取（inc-5046）
│   ├── pluginDirectories.ts          # getPluginsDirectory(), getPluginSeedDirs()
│   ├── installedPluginsManager.ts    # removeAllPluginsForMarketplace()
│   ├── cacheUtils.ts                 # markPluginVersionOrphaned()
│   ├── fetchTelemetry.ts             # logPluginFetch(), classifyFetchError()
│   ├── addDirPluginSettings.ts       # getAddDirEnabledPlugins(), getAddDirExtraMarketplaces()
│   ├── pluginIdentifier.ts           # parsePluginIdentifier()
│   ├── pluginOptionsStorage.ts       # deletePluginOptions()
│   ├── settings/settings.ts          # getSettingsForSource(), updateSettingsForSource()
│   ├── services/analytics/growthbook.ts  # getFeatureValue_CACHED_MAY_BE_STALE()
│   └── 通用工具函数
│       ├── errors.ts                 # ConfigParseError, errorMessage, isENOENT
│       ├── execFileNoThrow.ts        # git 命令执行
│       ├── git.ts                    # gitExe()
│       ├── fsOperations.ts           # getFsImplementation()
│       ├── log.ts                    # logError()
│       ├── debug.ts                  # logForDebugging()
│       ├── slowOperations.ts         # jsonParse, jsonStringify
│       └── envUtils.ts               # isEnvTruthy()
│
└── 被调用方（反向依赖）
    ├── reconciler.ts                 # diffMarketplaces(), reconcileMarketplaces()
    ├── services/plugins/pluginOperations.ts  # installPluginOp, updatePluginOp
    ├── commands/plugin/AddMarketplace.tsx
    ├── commands/plugin/ManageMarketplaces.tsx
    ├── commands/plugin/DiscoverPlugins.tsx
    └── cli/handlers/plugins.ts
```

### 4.2 关键代码路径详解

#### 4.2.1 启动时种子同步路径

```
entry: performStartupChecks.tsx 或 pluginStartupCheck.ts
│
├─► registerSeedMarketplaces()
│   ├─► getPluginSeedDirs() ──► process.env.CLAUDE_CODE_PLUGIN_SEED_DIR
│   ├─► readSeedKnownMarketplaces(seedDir)
│   │   └─► 读取 {seedDir}/known_marketplaces.json
│   ├─► findSeedMarketplaceLocation(seedDir, name)
│   │   └─► 探测 {seedDir}/marketplaces/{name} 或 {name}.json
│   └─► 合并到主配置（seed entries 强制 autoUpdate=false）
```

#### 4.2.2 插件发现时的市场源加载路径

```
entry: DiscoverPlugins.tsx 或 pluginOperations.ts
│
├─► loadKnownMarketplacesConfig()
│   └─► 读取 ~/.claude/plugins/known_marketplaces.json
│
├─► getMarketplace(name) [memoized]
│   ├─► 尝试 readCachedMarketplace() ──► 本地缓存命中
│   └─► 缓存失效 → loadAndCacheMarketplace() ──► 网络获取
│
└─► 遍历 marketplace.plugins 展示可安装插件
```

#### 4.2.3 市场源刷新路径

```
entry: ManageMarketplaces.tsx 或 cli/handlers/plugins.ts
│
├─► refreshMarketplace(name, onProgress)
│   ├─► 清除 memoized 缓存: getMarketplace.cache?.delete(name)
│   ├─► 检查种子源保护: seedDirFor() → 若存在则抛错
│   ├─► 官方源 GCS 优先: fetchOfficialMarketplaceFromGcs()
│   ├─► GitHub 源: SSH/HTTPS 智能回退
│   │   └─► cacheMarketplaceFromGit() → gitPull() / gitClone()
│   ├─► URL 源: cacheMarketplaceFromUrl() → axios GET
│   └─► 本地源: 仅验证文件存在
```

### 4.3 配置文件格式

#### 4.3.1 known_marketplaces.json

```json
{
  "claude-plugins-official": {
    "source": {
      "source": "github",
      "repo": "anthropics/claude-plugins-official"
    },
    "installLocation": "/home/user/.claude/plugins/marketplaces/claude-plugins-official",
    "lastUpdated": "2026-04-01T10:30:00.000Z",
    "autoUpdate": true
  },
  "company-internal": {
    "source": {
      "source": "url",
      "url": "https://internal.company.com/marketplace.json",
      "headers": {
        "Authorization": "Bearer ***"
      }
    },
    "installLocation": "/home/user/.claude/plugins/marketplaces/company-internal.json",
    "lastUpdated": "2026-03-15T08:00:00.000Z"
  }
}
```

#### 4.3.2 种子目录结构

```
$CLAUDE_CODE_PLUGIN_SEED_DIR/
├── known_marketplaces.json          # 种子市场源声明
└── marketplaces/
    ├── claude-plugins-official/     # 预置市场源内容
    │   └── .claude-plugin/
    │       └── marketplace.json
    └── another-marketplace.json
```

---

## 5. 依赖与外部交互

### 5.1 外部系统交互

| 外部系统 | 交互方式 | 用途 | 超时/限制 |
|---------|---------|------|----------|
| GitHub (git) | `git clone` / `git pull` | 克隆官方及第三方市场源 | 120s 默认（可配置） |
| GCS (downloads.claude.ai) | `axios.get()` | 官方市场源镜像（inc-5046） | 10s (latest) / 60s (zip) |
| 自定义 URL | `axios.get()` | 用户配置的 URL 源 | 10s |
| SSH 代理 | `ssh -T git@github.com` | 预检 SSH 配置 | 3s |
| 文件系统 | `fs.readFile/writeFile/rm` | 缓存管理 | N/A |

### 5.2 环境变量

| 变量 | 用途 | 默认值 |
|-----|------|-------|
| `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` | Git 操作超时 | 120000 (120s) |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | 插件缓存根目录 | `~/.claude/plugins` |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | 只读种子目录（PATH-like） | 未设置 |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | 使用 cowork_plugins 目录 | `false` |
| `CLAUDE_CODE_REMOTE` | CCR 模式（强制 HTTPS） | `false` |
| `GIT_TERMINAL_PROMPT` | 禁用 git 凭证提示 | `0` |
| `GIT_ASKPASS` | 禁用 git GUI 凭证 | `''` |

### 5.3 策略配置（通过 settings.json）

```json
{
  "strictKnownMarketplaces": [
    {"source": "github", "repo": "anthropics/claude-plugins-official"},
    {"source": "hostPattern", "hostPattern": "^github\\.mycompany\\.com$"}
  ],
  "blockedMarketplaces": [
    {"source": "github", "repo": "untrusted/repo"}
  ]
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 路径遍历风险（已缓解）

**风险**: 恶意 marketplace.json 使用 `../` 或绝对路径作为名称，导致文件写入/删除到任意位置。

**缓解措施**:
```typescript
// loadAndCacheMarketplace() 中的防御
const resolvedFinal = resolve(finalCachePath)
const resolvedCacheDir = resolve(cacheDir)
if (!resolvedFinal.startsWith(resolvedCacheDir + sep)) {
  throw new Error(`Marketplace name '${marketplace.name}' resolves to a path outside...`)
}
```

**状态**: 已修复（gh-32793, gh-32661 相关）。

#### 6.1.2 凭据泄露风险（已缓解）

**风险**: Git URL 中包含的用户名/密码可能出现在日志或错误消息中。

**缓解措施**:
```typescript
function redactUrlCredentials(urlString: string): string {
  const parsed = new URL(urlString)
  if (parsed.username) parsed.username = '***'
  if (parsed.password) parsed.password = '***'
  return parsed.toString()
}
```

**状态**: 已在 `gitClone()` 和日志中使用。

#### 6.1.3 种子源权限提升风险（已缓解）

**风险**: 用户尝试修改种子源（admin-managed）的 autoUpdate 或尝试删除。

**缓解措施**: 所有修改操作检查 `seedDirFor()`，若为种子源则抛错并给出指导。

#### 6.1.4 并发修改风险（部分缓解）

**风险**: 多个 Claude Code 实例同时修改 `known_marketplaces.json`。

**现状**: 无文件级锁，依赖 Node.js 单进程模型。启动时的 `registerSeedMarketplaces()` 是幂等的，但并发 `addMarketplaceSource()` 可能导致竞态。

### 6.2 边界情况

| 场景 | 行为 |
|-----|------|
| 网络完全离线 | `getMarketplaceCacheOnly()` 返回缓存数据；`getMarketplace()` 缓存失效时抛出错误 |
| 市场源 JSON 损坏 | `loadKnownMarketplacesConfig()` 抛出 `ConfigParseError`；`loadKnownMarketplacesConfigSafe()` 返回 `{}` |
| Git 仓库被强制推送 | `gitPull()` 失败 → 删除目录 → `gitClone()` 重新克隆 |
| 稀疏检出路径变更 | `reconcileSparseCheckout()` 检测到 Sparse→Full 转换需求 → 触发重新克隆 |
| 官方源 GCS 失败 | 根据 `tengu_plugin_official_mkt_git_fallback` feature flag 决定是否回退到 git |
| 本地文件源被删除 | `getMarketplace()` 缓存失效后尝试重新读取 → 抛出 ENOENT 错误 |

### 6.3 改进建议

#### 6.3.1 高优先级

1. **文件级并发锁**
   - 问题: 多进程并发修改 `known_marketplaces.json` 可能导致数据丢失
   - 建议: 使用 `proper-lockfile` 或类似的跨平台文件锁

2. **增量更新优化**
   - 问题: URL 源每次刷新都全量下载，即使内容未变
   - 建议: 支持 ETag/If-None-Match 或 Last-Modified 条件请求

3. **网络失败指数退避**
   - 问题: 网络抖动时频繁重试可能加剧问题
   - 建议: 添加指数退避机制，避免在恢复前反复尝试

#### 6.3.2 中优先级

4. **市场源健康检查**
   - 问题: 损坏的市场源只能在访问时才发现
   - 建议: 添加后台健康检查，提前发现并标记问题源

5. **缓存大小限制**
   - 问题: 长期使用的市场源缓存可能无限增长
   - 建议: 添加 LRU 驱逐策略或大小限制

6. **更好的离线体验**
   - 问题: 首次启动无网络时无法使用任何插件
   - 建议: 支持从种子目录预加载插件（不只是市场源声明）

#### 6.3.3 低优先级

7. **NPM 源类型实现**
   - 当前 `npm` source type 抛出 `not yet implemented`
   - 评估是否有实际需求

8. **市场源签名验证**
   - 当前依赖 HTTPS 和 Git 的传输安全
   - 考虑添加市场源内容签名验证（类似 VS Code 扩展签名）

---

## 7. 附录

### 7.1 关键函数签名速查

```typescript
// 配置管理
export async function loadKnownMarketplacesConfig(): Promise<KnownMarketplacesFile>
export async function saveKnownMarketplacesConfig(config: KnownMarketplacesFile): Promise<void>

// 市场源操作
export async function addMarketplaceSource(
  source: MarketplaceSource,
  onProgress?: MarketplaceProgressCallback
): Promise<{ name: string; alreadyMaterialized: boolean; resolvedSource: MarketplaceSource }>

export async function removeMarketplaceSource(name: string): Promise<void>
export async function refreshMarketplace(name: string, onProgress?: MarketplaceProgressCallback): Promise<void>
export async function refreshAllMarketplaces(): Promise<void>

// 数据获取
export const getMarketplace: ((name: string) => Promise<PluginMarketplace>) & { cache?: Map<string, Promise<PluginMarketplace>> }
export async function getMarketplaceCacheOnly(name: string): Promise<PluginMarketplace | null>
export async function getPluginById(pluginId: string): Promise<{ entry: PluginMarketplaceEntry; marketplaceInstallLocation: string } | null>

// 声明管理
export function getDeclaredMarketplaces(): Record<string, DeclaredMarketplace>
export async function registerSeedMarketplaces(): Promise<boolean>

// Git 操作（内部）
export async function gitClone(gitUrl: string, targetPath: string, ref?: string, sparsePaths?: string[]): Promise<{ code: number; stderr: string }>
export async function gitPull(cwd: string, ref?: string, options?: { disableCredentialHelper?: boolean; sparsePaths?: string[] }): Promise<{ code: number; stderr: string }>
export async function reconcileSparseCheckout(cwd: string, sparsePaths: string[] | undefined): Promise<{ code: number; stderr: string }>
```

### 7.2 相关 Issue/PR 参考

- **inc-5046**: GCS 镜像迁移（官方市场源）
- **gh-32793/gh-32661**: 路径遍历修复（installLocation 损坏保护）
- **gh-30696**: Git 子模块更新修复
- **gh-28373**: Git clone 空 stderr 错误增强
- **gh-31256**: Azure DevOps Git URL 支持（移除 `.git` 后缀强制检查）
- **#19708**: 相对路径迁移（legacy entries 处理）
- **#29608**: 项目级安装可见性修复

### 7.3 测试要点

1. **网络故障模拟**: 断开网络后验证缓存-only 路径
2. **损坏配置恢复**: 手动损坏 `known_marketplaces.json` 验证错误处理
3. **并发测试**: 同时启动多个实例添加市场源
4. **权限测试**: 尝试修改种子源验证保护机制
5. **Git 协议切换**: 模拟 SSH 失败验证 HTTPS 回退
6. **稀疏检出**: 测试 Full↔Sparse 转换场景
