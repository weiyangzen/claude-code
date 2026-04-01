# officialMarketplaceStartupCheck.ts 深度研究文档

## 场景与职责

`officialMarketplaceStartupCheck.ts` 实现了 Claude Code 启动时**自动安装官方插件市场**的完整逻辑。它处理各种边界情况，包括企业策略限制、Git 可用性、网络问题等，并提供智能的重试机制。

### 核心职责

1. **启动时自动安装**：检查并安装官方市场（如果尚未安装）
2. **策略合规检查**：验证企业策略是否允许安装官方市场
3. **多源获取**：优先尝试 GCS 镜像，失败后可回退到 Git
4. **智能重试**：指数退避重试机制，避免频繁失败请求
5. **状态持久化**：记录安装尝试状态和失败原因

### 在系统架构中的位置

```
启动流程
    ├── main.tsx
    │   └── 各种初始化
    └── hooks/useOfficialMarketplaceNotification.tsx
        └── checkAndInstallOfficialMarketplace()  ← 本文件入口
            ├── 检查重试策略
            ├── 检查环境变量禁用
            ├── 检查是否已安装
            ├── 检查企业策略
            ├── 尝试 GCS 获取
            └── 尝试 Git 克隆
```

---

## 功能点目的

### 1. 重试机制

配置参数：

```typescript
const RETRY_CONFIG = {
  MAX_ATTEMPTS: 10,                    // 最大重试次数
  INITIAL_DELAY_MS: 60 * 60 * 1000,    // 初始延迟：1小时
  BACKOFF_MULTIPLIER: 2,               // 退避乘数
  MAX_DELAY_MS: 7 * 24 * 60 * 60 * 1000, // 最大延迟：1周
}
```

重试策略：
- `policy_blocked`：永久失败，不重试
- `git_unavailable` / `gcs_unavailable` / `unknown`：可重试，指数退避

### 2. 多源获取优先级

```
1. GCS 镜像（inc-5046）
   - 优点：无需 Git，速度快
   - 缺点：需要后端支持发布

2. Git 克隆（fallback）
   - 优点：通用，无需额外基础设施
   - 缺点：需要 Git，较慢
   - 可通过 feature flag 禁用
```

### 3. 状态管理

存储在 `GlobalConfig` 中的字段：

| 字段 | 用途 |
|------|------|
| `officialMarketplaceAutoInstallAttempted` | 是否已尝试安装 |
| `officialMarketplaceAutoInstalled` | 是否成功安装 |
| `officialMarketplaceAutoInstallFailReason` | 失败原因 |
| `officialMarketplaceAutoInstallRetryCount` | 重试次数 |
| `officialMarketplaceAutoInstallLastAttemptTime` | 上次尝试时间 |
| `officialMarketplaceAutoInstallNextRetryTime` | 下次重试时间 |

### 4. 遥测事件

```typescript
logEvent('tengu_official_marketplace_auto_install', {
  installed: boolean,
  skipped: boolean,
  policy_blocked?: boolean,
  git_unavailable?: boolean,
  gcs_unavailable?: boolean,
  failed?: boolean,
  retry_count?: number,
  via_gcs?: boolean,
  macos_xcrun_shim?: boolean,
})
```

---

## 具体技术实现

### 核心流程 (`checkAndInstallOfficialMarketplace`)

```
checkAndInstallOfficialMarketplace()
    │
    ├── 检查是否应该重试 (shouldRetryInstallation)
    │   ├── 超过最大重试次数 → 跳过
    │   ├── 策略阻止 → 跳过
    │   └── 未到下次重试时间 → 跳过
    │
    ├── 检查环境变量禁用
    │   └── CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL
    │       → 标记 policy_blocked，返回
    │
    ├── 检查是否已安装
    │   └── known_marketplaces.json 中存在
    │       → 标记成功，返回
    │
    ├── 检查企业策略
    │   └── isSourceAllowedByPolicy()
    │       → 不允许则标记 policy_blocked，返回
    │
    ├── 尝试 GCS 获取
    │   └── fetchOfficialMarketplaceFromGcs()
    │       ├── 成功 → 注册市场，标记成功，返回
    │       └── 失败 → 继续
    │
    ├── 检查 Git fallback 开关
    │   └── tengu_plugin_official_mkt_git_fallback feature flag
    │       → 禁用则标记 gcs_unavailable，安排重试，返回
    │
    ├── 检查 Git 可用性
    │   └── checkGitAvailable()
    │       ├── 不可用 → 标记 git_unavailable，安排重试，返回
    │       └── 可用 → 继续
    │
    └── 尝试 Git 安装
        └── addMarketplaceSource()
            ├── 成功 → 标记成功，返回
            └── 失败 → 处理错误
                ├── macOS xcrun shim 错误 → 特殊处理
                └── 其他错误 → 标记 unknown，安排重试
```

### 重试逻辑详解

```typescript
function shouldRetryInstallation(config): boolean {
  // 从未尝试 → 应该尝试
  if (!config.officialMarketplaceAutoInstallAttempted) return true
  
  // 已成功 → 不重试
  if (config.officialMarketplaceAutoInstalled) return false
  
  // 超过最大次数 → 不重试
  if (retryCount >= RETRY_CONFIG.MAX_ATTEMPTS) return false
  
  // 策略阻止 → 不重试
  if (failReason === 'policy_blocked') return false
  
  // 未到时间 → 不重试
  if (nextRetryTime && now < nextRetryTime) return false
  
  // 其他情况 → 重试
  return true
}

function calculateNextRetryDelay(retryCount): number {
  const delay = RETRY_CONFIG.INITIAL_DELAY_MS * Math.pow(2, retryCount)
  return Math.min(delay, RETRY_CONFIG.MAX_DELAY_MS)
}
```

### macOS xcrun shim 特殊处理

macOS 上的 `/usr/bin/git` 是一个 xcrun shim，即使未安装 Xcode CLT 也存在：

```typescript
if (errorMessage.includes('xcrun: error:')) {
  markGitUnavailable()  // 毒化缓存
  // 不记录重试状态，下次启动重新检测
  return { installed: false, skipped: true, reason: 'git_unavailable' }
}
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `checkAndInstallOfficialMarketplace` | 主入口：检查和安装官方市场 | `useOfficialMarketplaceNotification.tsx` |
| `isOfficialMarketplaceAutoInstallDisabled` | 检查环境变量禁用 | 内部使用 |
| `shouldRetryInstallation` | 重试决策逻辑 | 内部使用 |
| `calculateNextRetryDelay` | 计算下次重试延迟 | 内部使用 |

### 导出常量

| 常量 | 用途 |
|------|------|
| `RETRY_CONFIG` | 重试配置参数 |

### 导出类型

| 类型 | 用途 |
|------|------|
| `OfficialMarketplaceSkipReason` | 跳过原因枚举 |
| `OfficialMarketplaceCheckResult` | 检查结果类型 |

### 调用关系图

```
hooks/useOfficialMarketplaceNotification.tsx
    └── useEffect
        └── checkAndInstallOfficialMarketplace()
            ├── getGlobalConfig()                    # utils/config.ts
            ├── shouldRetryInstallation()
            ├── isOfficialMarketplaceAutoInstallDisabled()
            ├── loadKnownMarketplacesConfig()        # marketplaceManager.ts
            ├── isSourceAllowedByPolicy()            # marketplaceHelpers.ts
            ├── fetchOfficialMarketplaceFromGcs()    # officialMarketplaceGcs.ts
            │   └── 成功: addMarketplaceSource()     # marketplaceManager.ts
            ├── getFeatureValue_CACHED_MAY_BE_STALE() # services/analytics/growthbook.ts
            ├── checkGitAvailable()                  # gitAvailability.ts
            │   └── 失败: markGitUnavailable()
            └── addMarketplaceSource()               # marketplaceManager.ts
                └── 失败处理
                    ├── markGitUnavailable()         # gitAvailability.ts
                    └── saveGlobalConfig()           # utils/config.ts
```

---

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `officialMarketplace.ts` | 官方市场常量 |
| `officialMarketplaceGcs.ts` | GCS 获取 |
| `marketplaceManager.ts` | 市场管理（addMarketplaceSource, loadKnownMarketplacesConfig） |
| `marketplaceHelpers.ts` | 策略检查（isSourceAllowedByPolicy） |
| `gitAvailability.ts` | Git 可用性检查 |
| `utils/config.ts` | 全局配置读写 |
| `utils/debug.ts` | 调试日志 |
| `utils/envUtils.ts` | 环境变量检查 |
| `utils/errors.ts` | 错误处理 |
| `utils/log.ts` | 错误日志 |
| `services/analytics/growthbook.ts` | Feature flag |
| `services/analytics/index.ts` | 遥测 |

### 配置存储

```
~/.claude/config.json
    └── officialMarketplaceAutoInstallAttempted: boolean
    └── officialMarketplaceAutoInstalled: boolean
    └── officialMarketplaceAutoInstallFailReason: string
    └── officialMarketplaceAutoInstallRetryCount: number
    └── officialMarketplaceAutoInstallLastAttemptTime: number
    └── officialMarketplaceAutoInstallNextRetryTime: number
```

### 环境变量

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` | 禁用自动安装 |

---

## 风险、边界与改进建议

### 已知风险

1. **配置保存失败**
   - 风险：`saveGlobalConfig` 可能失败，导致重试状态丢失
   - 缓解：捕获错误，记录日志，返回 `configSaveFailed` 标记

2. **Git 可用性误判**
   - 风险：`which git` 通过但 `git clone` 失败（如 macOS xcrun shim）
   - 缓解：特殊处理 xcrun 错误，毒化缓存

3. **Feature flag 缓存**
   - 风险：`getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期值
   - 现状：函数名已提示风险，默认启用 fallback

4. **时间回拨**
   - 风险：系统时间回拨可能导致 `nextRetryTime` 逻辑异常
   - 现状：未处理，依赖 NTP 同步

### 边界情况

| 场景 | 行为 |
|------|------|
| 首次启动 | 尝试安装，记录状态 |
| 已安装 | 快速跳过，更新状态 |
| 策略阻止 | 永久跳过，不重试 |
| Git 不可用 | 指数退避重试，最多10次 |
| GCS 失败且 fallback 禁用 | 标记 gcs_unavailable，安排重试 |
| 配置保存失败 | 继续返回失败结果，但标记 configSaveFailed |
| 未知错误 | 标记 unknown，安排重试 |

### 改进建议

1. **后台静默安装**
   - 建议：启动后延迟几秒再尝试，避免阻塞 UI

2. **用户通知**
   - 建议：多次失败后提示用户手动安装

3. **网络检测**
   - 建议：检测网络状态，离线时跳过尝试

4. **配置校验**
   - 建议：定期校验已安装市场的完整性

5. **降级策略**
   - 建议：GCS 和 Git 都失败时，尝试使用预置的 seed 目录

---

## 测试要点

1. **重试逻辑**：验证指数退避计算和最大次数限制
2. **状态转换**：验证各种场景下的状态变更
3. **错误处理**：验证 xcrun shim、网络错误、权限错误的处理
4. **策略检查**：验证企业策略阻止的正确性
5. **并发安全**：验证多进程同时调用的行为
