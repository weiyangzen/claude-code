# Settings Sync Service - 研究文档

## 场景与职责

本文件 (`src/services/settingsSync/index.ts`) 实现 Claude Code 的**设置同步服务**，负责在用户的不同 Claude Code 环境之间同步设置和记忆文件。

### 核心使用场景

| 场景 | 方向 | 触发时机 |
|------|------|----------|
| **交互式 CLI** | 上传（Upload） | 启动时 `main.tsx` preAction hook，后台增量上传本地变更 |
| **CCR (Claude Code Remote)** | 下载（Download） | `print.ts` runHeadless() 启动时，插件安装前拉取远程设置 |
| **中途刷新** | 强制重新下载 | `/reload-plugins` 命令或 SDK `reload_plugins` 消息 |

### 同步范围

同步四类文件：
1. **用户设置** (`~/.claude/settings.json`) - 全局用户偏好
2. **用户记忆** (`~/.claude/CLAUDE.md`) - 全局记忆文件
3. **项目设置** (`.claude/settings.local.json`) - 项目级本地设置
4. **项目记忆** (`CLAUDE.local.md`) - 项目级本地记忆

---

## 功能点目的

### 1. 上传本地设置 (`uploadUserSettingsInBackground`)

**触发条件**（需同时满足）：
- 特性开关 `UPLOAD_USER_SETTINGS` 启用
- GrowthBook 标志 `tengu_enable_settings_sync_push` 为 true
- 当前是交互式 CLI (`getIsInteractive()`)
- 使用 Anthropic OAuth 认证 (`isUsingOAuth()`)

**增量上传逻辑**：
1. 先调用 `fetchUserSettings()` 获取远程当前状态
2. 使用 `pickBy` 对比本地与远程条目，仅上传变更项
3. 若无变更则跳过上传

**设计意图**：减少网络流量，避免不必要的 API 调用。

### 2. 下载远程设置 (`downloadUserSettings` / `redownloadUserSettings`)

| 函数 | 特点 | 使用场景 |
|------|------|----------|
| `downloadUserSettings()` | Promise 缓存（首次调用发起请求，后续复用） | 启动时 fire-and-forget |
| `redownloadUserSettings()` | 绕过缓存，强制新请求，0 次重试 | 用户主动触发 `/reload-plugins` |

**触发条件**：
- 特性开关 `DOWNLOAD_USER_SETTINGS` 启用
- GrowthBook 标志 `tengu_strap_foyer` 为 true
- 使用 Anthropic OAuth 认证
- CCR 模式（`CLAUDE_CODE_REMOTE` 或 `getIsRemoteMode()`）

### 3. 本地条目构建 (`buildEntriesFromLocalFiles`)

从本地文件系统读取同步范围内的文件：
- 项目级文件仅在能获取 `projectId`（git remote hash）时处理
- 单文件大小上限 **500 KB**（与后端限制一致）
- 空文件或仅含空白字符的文件被忽略

### 4. 远程条目应用 (`applyRemoteEntriesToLocal`)

将下载的远程条目写回本地文件：
- 写入前再次校验大小（defense-in-depth）
- 调用 `markInternalWrite()` 抑制文件监听器的误触发
- 写入后清除相关缓存（`resetSettingsCache()` / `clearMemoryFileCaches()`）

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### API 端点与协议

```typescript
// 端点构造
`${getOauthConfig().BASE_API_URL}/api/claude_code/user_settings`

// 认证头
{
  Authorization: `Bearer ${accessToken}`,
  'anthropic-beta': 'oauth-2025-04-20'
}
```

**请求方法**：
- **GET**：获取用户设置（下载）
- **PUT**：上传设置条目（增量更新）

**响应处理**：
- `200`：成功，使用 Zod 模式验证响应格式
- `404`：表示用户无同步数据（`isEmpty: true`）
- `401/403`：认证失败，标记 `skipRetry: true`

### 重试机制

```typescript
const DEFAULT_MAX_RETRIES = 3
const SETTINGS_SYNC_TIMEOUT_MS = 10000 // 10 秒
```

**重试策略**：
- 使用指数退避（通过 `getRetryDelay()` 计算）
- 认证错误（401/403）不重试
- `redownloadUserSettings()` 明确使用 0 次重试（用户主动触发，fail-fast）

### Promise 缓存模式

```typescript
let downloadPromise: Promise<boolean> | null = null

export function downloadUserSettings(): Promise<boolean> {
  if (downloadPromise) {
    return downloadPromise  // 复用进行中的请求
  }
  downloadPromise = doDownloadUserSettings()
  return downloadPromise
}
```

**设计意图**：避免启动时多个组件重复发起相同的下载请求。

### OAuth 认证检查

```typescript
function isUsingOAuth(): boolean {
  // 仅检查 user:inference，不检查 user:profile
  // 原因：CCR 的文件描述符 token 仅包含 user:inference
  return (
    getAPIProvider() === 'firstParty' &&
    isFirstPartyAnthropicBaseUrl() &&
    tokens?.scopes?.includes('user:inference')
  )
}
```

### 文件大小限制

```typescript
const MAX_FILE_SIZE_BYTES = 500 * 1024 // 500 KB
```

**两处检查**：
1. `tryReadFileForSync()`：上传前读取本地文件时
2. `applyRemoteEntriesToLocal()`：下载后写入本地文件前

### 内部写入标记

```typescript
// 写入前
markInternalWrite(filePath)
// 执行写入
await writeFile(filePath, content)
// changeDetector 在 5 秒内会忽略此文件的变更事件
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 类型 | 说明 |
|------|------|------|
| `uploadUserSettingsInBackground` | `async function` | CLI 上传入口（后台执行） |
| `downloadUserSettings` | `function` | CCR 下载入口（带 Promise 缓存） |
| `redownloadUserSettings` | `function` | 强制重新下载（绕过缓存） |
| `_resetDownloadPromiseForTesting` | `function` | 测试专用：清除 Promise 缓存 |

### 调用方（入口点）

| 文件路径 | 调用方式 | 说明 |
|----------|----------|------|
| `src/main.tsx:964` | 动态 import | `uploadUserSettingsInBackground()`，preAction hook |
| `src/cli/print.ts:514` | 直接调用 | `downloadUserSettings()`，fire-and-forget |
| `src/cli/print.ts:1713` | await | `downloadUserSettings()`，插件安装前等待 |
| `src/cli/print.ts:3073` | await | `redownloadUserSettings()`，SDK `reload_plugins` 消息 |
| `src/commands/reload-plugins/reload-plugins.ts:28` | await | `redownloadUserSettings()`，`/reload-plugins` 命令 |

### 依赖模块

| 文件路径 | 导入内容 | 用途 |
|----------|----------|------|
| `../../bootstrap/state.js` | `getIsInteractive` | 判断交互式模式 |
| `../../constants/oauth.js` | `CLAUDE_AI_INFERENCE_SCOPE`, `getOauthConfig`, `OAUTH_BETA_HEADER` | OAuth 配置 |
| `../../utils/auth.js` | `checkAndRefreshOAuthTokenIfNeeded`, `getClaudeAIOAuthTokens` | Token 管理 |
| `../../utils/claudemd.js` | `clearMemoryFileCaches` | 清除记忆缓存 |
| `../../utils/config.js` | `getMemoryPath` | 获取记忆文件路径 |
| `../../utils/diagLogs.js` | `logForDiagnosticsNoPII` | 诊断日志 |
| `../../utils/errors.js` | `classifyAxiosError` | 错误分类 |
| `../../utils/git.js` | `getRepoRemoteHash` | 获取项目 ID |
| `../../utils/model/providers.js` | `getAPIProvider`, `isFirstPartyAnthropicBaseUrl` | 判断官方端点 |
| `../../utils/settings/internalWrites.js` | `markInternalWrite` | 标记内部写入 |
| `../../utils/settings/settings.js` | `getSettingsFilePathForSource` | 获取设置路径 |
| `../../utils/settings/settingsCache.js` | `resetSettingsCache` | 清除设置缓存 |
| `../analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` | 特性开关 |
| `../analytics/index.js` | `logEvent` | 埋点上报 |
| `../api/withRetry.js` | `getRetryDelay` | 重试延迟计算 |
| `./types.js` | 类型定义 | 类型与常量 |

---

## 依赖与外部交互

### 后端 API 依赖

**端点**：`GET/PUT /api/claude_code/user_settings`

**认证**：OAuth Bearer Token（`user:inference` scope）

**相关 PR**：`anthropic/anthropic#218817`

### 文件系统交互

| 操作 | 路径来源 | 说明 |
|------|----------|------|
| 读取 | `getSettingsFilePathForSource()` | 用户/项目设置 JSON |
| 读取 | `getMemoryPath()` | CLAUDE.md 记忆文件 |
| 写入 | 同上 | 下载时写回本地 |

### 缓存交互

| 缓存 | 操作 | 时机 |
|------|------|------|
| `settingsCache` | `resetSettingsCache()` | 写入设置文件后 |
| `memoryFileCaches` | `clearMemoryFileCaches()` | 写入记忆文件后 |
| `internalWrites` | `markInternalWrite()` | 写入文件前 |

---

## 风险、边界与改进建议

### 当前风险与边界

1. **Fail-open 设计**
   - 所有错误都被捕获并记录，不会阻塞启动或插件安装。
   - 风险：用户可能在不知情的情况下使用了过期的本地设置。

2. **Promise 缓存永不自动失效**
   - `downloadUserSettings()` 的 Promise 缓存在进程生命周期内持续有效。
   - 如果启动时下载失败，后续调用会立即返回失败的 Promise，不会重试，直到进程重启。

3. **Mid-session 通知责任外移**
   - `redownloadUserSettings()` 的注释明确说明：调用方需要负责在返回 `true` 后调用 `settingsChangeDetector.notifyChange('userSettings')`。
   - 当前由 `reload-plugins.ts` 和 `print.ts` 两处维护，如果未来新增调用点容易遗漏。

4. **Git Remote 依赖**
   - `getRepoRemoteHash()` 基于 git remote URL 的 SHA256 前 16 位。
   - 如果项目没有 git remote（本地新建目录），`projectId` 为 `null`，项目级 settings/memory 将**不会**被同步。

5. **大小限制与静默丢弃**
   - 单文件 500 KB 上限超过时，`tryReadFileForSync()` 和 `applyRemoteEntriesToLocal()` 均会静默返回 `null` 或跳过写入，不会向用户报错。

6. **上传的增量判断是字符串全等**
   - `pickBy` 使用 `remoteEntries[key] !== value` 做严格字符串比较。
   - 如果本地文件仅因换行符（CRLF/LF）或末尾空行差异导致字符串不等，也会触发上传。

7. **OAuth Scope 限制**
   - 仅检查 `user:inference`，不检查 `user:profile`。
   - 原因：CCR 文件描述符 token 仅包含 `user:inference`。

### 改进建议

1. **Promise 缓存增加失败重试机制**
   ```typescript
   // 在 downloadPromise 失败后设置短时间冷却期
   if (result.success === false && !result.skipRetry) {
     setTimeout(() => { downloadPromise = null }, 30000)
   }
   ```

2. **统一设置变更通知**
   - 考虑通过事件总线或回调参数，让 `redownloadUserSettings` 在内部安全地触发通知，降低调用方遗漏的风险。

3. **增加大小超限警告**
   - 当文件超过 500 KB 时，通过 `logEvent` 上报或向用户显示警告，避免静默失败。

4. **规范化内容比较**
   - 上传前对内容进行规范化处理（统一换行符、去除末尾空白），减少不必要的同步流量。

5. **无 Git Remote 时的提示**
   - 当 `projectId` 为 `null` 时，在诊断日志中记录原因，帮助排查项目级设置未同步的问题。

6. **考虑添加测试覆盖**
   - 当前 `src/services/settingsSync/` 目录下无测试文件，建议添加单元测试覆盖：
     - `buildEntriesFromLocalFiles` 的各种边界情况
     - `applyRemoteEntriesToLocal` 的缓存清除逻辑
     - Promise 缓存的正确性
