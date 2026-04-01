# 研究文档：src/services/policyLimits/index.ts

## 场景与职责

`src/services/policyLimits/index.ts` 是 Claude Code CLI 的**组织级策略限制服务**。它的核心职责是从 Anthropic 后端 API 拉取当前用户所属组织配置的策略限制（policy restrictions），并以同步/异步接口的形式供 CLI 各模块查询，从而动态禁用或启用特定功能。

该服务主要面向两类用户群体：
- **Console 用户（API Key）**：所有持有 Anthropic API Key 的用户均符合条件。
- **OAuth 用户（Claude.ai）**：仅 Team 和 Enterprise/C4E 订阅者符合条件，因为这些组织的管理员可在后台配置策略限制（例如禁用远程会话、禁用产品反馈等）。

设计哲学上，该服务明确遵循 **"fail open"** 原则：如果网络请求失败、缓存不可用或用户不符合资格，CLI 默认**不施加任何限制**，以保证用户体验不被阻断。唯一的例外是在 `essential-traffic-only`（HIPAA/ZDR）模式下，某些敏感策略（如 `allow_product_feedback`）会在缓存缺失时 **fail closed**。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **资格判定 (`isPolicyLimitsEligible`)** | 避免向不符合条件的用户（如第三方提供商用户、自定义 Base URL 用户、普通 Pro/Max OAuth 用户）发起无意义的 API 请求，减少服务端负载和本地开销。 |
| **初始化加载 Promise (`initializePolicyLimitsLoadingPromise`)** | 在 CLI 启动早期（`init.ts`）创建一个可等待的 Promise，让下游功能（如 `/remote-control`）能够阻塞等待策略加载完成，避免在策略尚未就绪时做出错误的功能可用性判断。 |
| **带重试的策略获取 (`fetchWithRetry` / `fetchPolicyLimits`)** | 通过 HTTP 向 `/api/claude_code/policy_limits` 端点获取最新策略，支持 ETag 缓存协商（304 Not Modified）和指数退避重试，降低网络抖动影响。 |
| **持久化文件缓存 (`loadCachedRestrictions` / `saveCachedRestrictions`)** | 将策略限制缓存到 `~/.claude/policy-limits.json`，在 fetch 失败或 304 时提供离线回退，文件权限设置为 `0o600` 保护敏感配置。 |
| **后台轮询 (`startBackgroundPolling` / `pollPolicyLimits`)** | 以 1 小时为间隔持续同步策略变更，使管理员在 Web 控制台修改策略后，已运行的 CLI 会话能在合理时间内感知到变化。 |
| **同步策略查询 (`isPolicyAllowed`)** | 提供一个零阻塞的同步接口，供命令注册、组件渲染、工具启用等高频路径快速判断某项功能是否被组织策略允许。 |
| **生命周期管理 (`loadPolicyLimits` / `refreshPolicyLimits` / `clearPolicyLimitsCache`)** | 分别在启动、登录切换、登出时加载、刷新或清除策略状态，确保用户切换账户后策略不会串用。 |

---

## 具体技术实现

### 3.1 核心常量与状态

```ts
const CACHE_FILENAME = 'policy-limits.json'
const FETCH_TIMEOUT_MS = 10000
const DEFAULT_MAX_RETRIES = 5
const POLLING_INTERVAL_MS = 60 * 60 * 1000 // 1 hour
const LOADING_PROMISE_TIMEOUT_MS = 30000
```

模块级状态变量（单例模式）：
- `sessionCache`: 内存中的策略限制对象，供 `isPolicyAllowed` 同步读取。
- `pollingIntervalId`: 后台轮询的 `setInterval` 句柄，支持 `unref()` 避免阻塞进程退出。
- `loadingCompletePromise` / `loadingCompleteResolve`: 控制初始化等待流程，30 秒超时防止死锁。
- `cleanupRegistered`: 确保只向 `cleanupRegistry` 注册一次清理回调。

### 3.2 资格判定逻辑 (`isPolicyLimitsEligible`)

该函数被设计为**完全独立**，明确注释要求不能调用 `getSettings()`，以避免在设置系统初始化期间产生循环依赖。

判定流程（短路返回）：
1. `getAPIProvider() !== 'firstParty'` → `false`（第三方提供商用户跳过）。
2. `!isFirstPartyAnthropicBaseUrl()` → `false`（自定义 Base URL 用户跳过）。
3. 尝试获取 API Key（`getAnthropicApiKeyWithSource({ skipRetrievingKeyFromApiKeyHelper: true })`），成功 → `true`。
4. 检查 OAuth Token：无 `accessToken` → `false`。
5. 检查 Scope：不包含 `CLAUDE_AI_INFERENCE_SCOPE` → `false`。
6. 检查订阅类型：非 `enterprise` 且非 `team` → `false`。
7. 其余情况 → `true`。

### 3.3 认证头构造 (`getAuthHeaders`)

同样避免依赖 `getSettings()`。优先级：
1. API Key（`x-api-key` 头）。
2. OAuth Bearer Token + `anthropic-beta: oauth-2025-04-20`。
3. 无认证 → 返回 `error` 字段，触发 `skipRetry: true`。

### 3.4 获取与重试机制 (`fetchWithRetry` / `fetchPolicyLimits`)

`fetchWithRetry` 实现了一个简单的 for 循环重试：
- 最大重试次数 `DEFAULT_MAX_RETRIES = 5`，即最多 6 次尝试。
- 退避延迟通过 `getRetryDelay(attempt)` 计算（来自 `src/services/api/withRetry.js`）。
- 若 `lastResult.skipRetry === true`（如认证错误），立即终止重试。

`fetchPolicyLimits`（单次请求）细节：
- 先调用 `checkAndRefreshOAuthTokenIfNeeded()` 确保 OAuth Token 未过期。
- 构造请求头，若提供 `cachedChecksum` 则附加 `If-None-Match: "<checksum>"`。
- `axios.get` 的 `validateStatus` 仅接受 `200 | 304 | 404`。
- **304**：返回 `restrictions: null` 作为"缓存有效"信号。
- **404**：返回空对象 `{}`，表示该用户/组织无策略限制或功能未启用。
- **200**：用 `PolicyLimitsResponseSchema().safeParse()` 校验响应体格式。
- 错误分类：通过 `classifyAxiosError` 区分 `auth` / `timeout` / `network` / `http` / `other`。

### 3.5 缓存一致性：校验和 (`computeChecksum` / `sortKeysDeep`)

为了可靠地使用 HTTP ETag 协商，服务在本地计算策略内容的 SHA-256 校验和：
- `sortKeysDeep` 递归地对对象键进行字母排序、对数组元素递归处理，确保 JSON 序列化结果稳定。
- `computeChecksum` 使用 `jsonStringify(sorted)` 后计算 `sha256:...` 格式的校验和。
- 该校验和作为 `If-None-Match` 的值发送给服务端。

### 3.6 文件缓存读写

- **读 (`loadCachedRestrictions`)**：同步读取 `~/.claude/policy-limits.json`，用 `safeParseJSON` 解析后通过 Zod Schema 校验，失败返回 `null`。
- **写 (`saveCachedRestrictions`)**：异步写入，文件模式 `0o600`，内容以 2 空格缩进的 JSON 存储。
- **删除**：当服务端返回 404（空限制）时，主动 `unlink` 缓存文件，避免本地残留过期策略。

### 3.7 策略查询的 fail-open 与例外 (`isPolicyAllowed`)

```ts
export function isPolicyAllowed(policy: string): boolean {
  const restrictions = getRestrictionsFromCache()
  if (!restrictions) {
    // essential-traffic-only 模式下，特定策略 fail closed
    if (isEssentialTrafficOnly() && ESSENTIAL_TRAFFIC_DENY_ON_MISS.has(policy)) {
      return false
    }
    return true // fail open
  }
  const restriction = restrictions[policy]
  if (!restriction) {
    return true // 未知策略 = 允许
  }
  return restriction.allowed
}
```

`ESSENTIAL_TRAFFIC_DENY_ON_MISS` 当前仅包含 `'allow_product_feedback'`。这是为 HIPAA 等零数据保留（ZDR）组织设计的：如果缓存不可用，宁可禁用反馈功能，也不能在合规模式下意外启用它。

### 3.8 后台轮询与进程清理

`startBackgroundPolling`：
- 检查是否已在轮询、用户是否仍符合条件。
- `setInterval(..., POLLING_INTERVAL_MS)`，并调用 `.unref()` 避免阻止 Node/Bun 进程自然退出。
- 首次启动时向 `registerCleanup` 注册 `stopBackgroundPolling`，确保 CLI 退出时清理句柄。

`pollPolicyLimits`：
- 通过 `jsonStringify` 对比轮询前后的 `sessionCache`，若发生变化则记录调试日志。
- 所有异常被静默捕获，后台轮询**绝不 fail closed**。

### 3.9 测试辅助函数

`_resetPolicyLimitsForTesting`：
- 仅清除内存状态和轮询句柄，**不做文件 I/O**。
- 注释明确说明这是为了供 `preload` 阶段的 `beforeEach` 快速重置使用，避免 `clearPolicyLimitsCache` 的异步文件操作拖慢测试。

---

## 关键代码路径与文件引用

### 4.1 本模块内部路径

| 符号 | 行号 | 说明 |
|------|------|------|
| `isPolicyLimitsEligible` | 167-211 | 资格判定入口 |
| `initializePolicyLimitsLoadingPromise` | 94-114 | 初始化可等待 Promise |
| `waitForPolicyLimitsToLoad` | 217-221 | 阻塞等待初始化完成 |
| `fetchWithRetry` | 267-295 | 带指数退避的重试包装 |
| `fetchPolicyLimits` | 300-386 | 单次 HTTP 获取 |
| `fetchAndLoadPolicyLimits` | 432-495 | 整合 fetch + 缓存更新 |
| `isPolicyAllowed` | 510-526 | 同步策略查询 |
| `loadPolicyLimits` | 556-575 | CLI 启动时调用 |
| `refreshPolicyLimits` | 581-590 | 登录后刷新 |
| `clearPolicyLimitsCache` | 595-608 | 登出/重置时清除 |
| `startBackgroundPolling` | 635-653 | 启动后台轮询 |
| `_resetPolicyLimitsForTesting` | 79-84 | 测试快速重置 |

### 4.2 上游调用方（谁在使用 policyLimits）

| 调用方文件 | 使用的 API | 用途 |
|------------|-----------|------|
| `src/entrypoints/init.ts` | `initializePolicyLimitsLoadingPromise`, `isPolicyLimitsEligible` | 启动早期初始化 Promise |
| `src/main.tsx` | `loadPolicyLimits`, `waitForPolicyLimitsToLoad`, `isPolicyAllowed` | preAction 中加载策略；后续多处判断功能可用性 |
| `src/entrypoints/cli.tsx` | `waitForPolicyLimitsToLoad`, `isPolicyAllowed('allow_remote_control')` | `claude remote-control` 快速路径检查 |
| `src/commands/bridge/bridge.tsx` | `waitForPolicyLimitsToLoad`, `isPolicyAllowed('allow_remote_control')` | `/remote-control` 命令前置检查 |
| `src/bridge/initReplBridge.ts` | `waitForPolicyLimitsToLoad`, `isPolicyAllowed('allow_remote_control')` | REPL Bridge 启动前检查组织策略 |
| `src/commands/remote-env/index.ts` | `isPolicyAllowed('allow_remote_sessions')` | 命令注册时控制 `remote-env` 可见性 |
| `src/commands/remote-setup/index.ts` | `isPolicyAllowed('allow_remote_sessions')` | 命令注册时控制 `web-setup` 可见性 |
| `src/commands/feedback/index.ts` | `isPolicyAllowed('allow_product_feedback')` | 控制 `/feedback` 命令是否可用 |
| `src/skills/bundled/scheduleRemoteAgents.ts` | `isPolicyAllowed('allow_remote_sessions')` | `/schedule` skill 启用条件 |
| `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts` | `isPolicyAllowed('allow_remote_sessions')` | 远程触发工具启用条件 |
| `src/utils/background/remote/remoteSession.ts` | `isPolicyAllowed('allow_remote_sessions')` | 后台远程会话资格检查 |
| `src/utils/teleport.tsx` | `isPolicyAllowed('allow_remote_sessions')` | `--teleport` 恢复远程会话前检查 |
| `src/components/FeedbackSurvey/useFeedbackSurvey.tsx` | `isPolicyAllowed('allow_product_feedback')` | 反馈调查弹窗及转录分享请求检查 |
| `src/components/FeedbackSurvey/useMemorySurvey.tsx` | `isPolicyAllowed('allow_product_feedback')` | 内存调查弹窗检查 |
| `src/commands/login/login.tsx` | `refreshPolicyLimits` | 登录成功后刷新策略 |
| `src/commands/logout/logout.tsx` | `clearPolicyLimitsCache` | 登出时清除策略缓存 |

### 4.3 下游依赖（policyLimits 依赖谁）

| 依赖文件 | 使用的符号 | 说明 |
|----------|-----------|------|
| `src/constants/oauth.ts` | `CLAUDE_AI_INFERENCE_SCOPE`, `getOauthConfig`, `OAUTH_BETA_HEADER` | OAuth 配置与 Scope 常量 |
| `src/utils/auth.ts` | `checkAndRefreshOAuthTokenIfNeeded`, `getAnthropicApiKeyWithSource`, `getClaudeAIOAuthTokens` | 认证状态读取 |
| `src/utils/errors.ts` | `classifyAxiosError` | Axios 错误分类 |
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir` | 缓存文件目录 |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly` | HIPAA/ZDR 模式检测 |
| `src/utils/cleanupRegistry.ts` | `registerCleanup` | 进程退出清理注册 |
| `src/utils/debug.ts` | `logForDebugging` | 调试日志 |
| `src/utils/sleep.ts` | `sleep` | 重试间隔 |
| `src/utils/slowOperations.ts` | `jsonStringify` | 稳定 JSON 序列化 |
| `src/utils/userAgent.ts` | `getClaudeCodeUserAgent` | 请求 User-Agent |
| `src/utils/model/providers.ts` | `getAPIProvider`, `isFirstPartyAnthropicBaseUrl` | 提供商与 Base URL 判断 |
| `src/services/api/withRetry.ts` | `getRetryDelay` | 退避延迟计算 |
| `src/utils/json.ts` | `safeParseJSON` | 安全 JSON 解析 |
| `./types.ts` | `PolicyLimitsResponseSchema`, `PolicyLimitsFetchResult`, `PolicyLimitsResponse` | 类型与校验模式 |

---

## 依赖与外部交互

### 5.1 HTTP API 交互

**端点**：`GET ${getOauthConfig().BASE_API_URL}/api/claude_code/policy_limits`

**请求头**：
- `x-api-key` 或 `Authorization: Bearer <token>`
- `User-Agent: <getClaudeCodeUserAgent()>`
- `If-None-Match: "<sha256:checksum>"`（可选，当本地有缓存时）
- `anthropic-beta: oauth-2025-04-20`（OAuth 场景）

**可接受响应状态**：200, 304, 404

**响应体格式**（Zod Schema 定义于 `types.ts`）：
```json
{
  "restrictions": {
    "allow_remote_control": { "allowed": false },
    "allow_remote_sessions": { "allowed": false },
    "allow_product_feedback": { "allowed": false }
  }
}
```
仅包含被**显式阻止**的策略；缺失的键默认视为允许。

### 5.2 文件系统交互

- **读**：`fs.readFileSync(join(getClaudeConfigHomeDir(), 'policy-limits.json'), 'utf-8')`（同步，因为 `isPolicyAllowed` 可能被同步上下文调用）。
- **写**：`fs/promises.writeFile(..., { mode: 0o600 })`（异步）。
- **删**：`fs/promises.unlink(...)`（异步，404 响应时清理）。

### 5.3 与远程管理设置（Remote Managed Settings）的平行关系

`policyLimits` 与 `src/services/remoteManagedSettings/` 在架构上高度对称：
- 两者都在 `init.ts` 中初始化 loading promise。
- 两者都在 `main.tsx` 的 `preAction` hook 中异步加载。
- 两者都在 `login.tsx` 中刷新、在 `logout.tsx` 中清除。
- 两者都使用文件缓存、ETag 协商、后台轮询、fail-open 策略。

这种平行关系意味着对其中一个服务的修改（如轮询间隔、超时时间、缓存路径）通常需要同步审视另一个服务，以保持一致性。

---

## 风险、边界与改进建议

### 6.1 已知风险

1. **同步文件读取阻塞事件循环**
   - `loadCachedRestrictions` 和 `getRestrictionsFromCache` 在同步上下文中使用 `fsReadFileSync`。虽然文件很小（通常 < 1KB），但在某些环境（如网络文件系统、高负载容器）中仍可能产生微卡顿。`isPolicyAllowed` 被大量 UI 渲染路径调用，累积效应不可忽视。

2. **ETag 校验和基于内容而非服务端 ETag**
   - 服务使用本地计算的 SHA-256 作为 `If-None-Match` 值，而非服务端返回的 ETag。如果服务端未来改变响应体格式（如增加字段）但保持语义不变，本地校验和会失效，导致不必要的 200 响应和缓存重写。

3. **后台轮询无变更通知机制**
   - 轮询发现策略变更后仅记录调试日志，没有向 `AppState` 或事件总线广播。这意味着某些依赖 `isPolicyAllowed` 的 React 组件不会自动重渲染，用户可能需要重启 CLI 才能看到命令/工具的可见性变化。

4. **`isPolicyLimitsEligible` 与 `getAuthHeaders` 的认证逻辑重复**
   - 两者都实现了"先 API Key 后 OAuth"的优先级判断。如果 `auth.ts` 中的认证源逻辑发生变化（如新增一种 token 来源），这里可能出现不一致。

5. **测试重置函数未覆盖文件缓存**
   - `_resetPolicyLimitsForTesting` 不删除磁盘缓存，若测试用例间共享文件系统（如未使用 `tmpfs` 隔离），可能出现测试污染。

### 6.2 边界情况

- **用户从 Enterprise 切换到 Pro**：登出时 `clearPolicyLimitsCache` 会清除缓存，但 `sessionCache` 在内存中。若登出不彻底（如某些测试场景未调用 `clearPolicyLimitsCache`），`isPolicyAllowed` 可能继续使用旧企业的策略。
- **网络完全不可用 + 无缓存**：`fetchAndLoadPolicyLimits` 返回 `null`，`isPolicyAllowed` 对所有策略返回 `true`（fail open）。在 `essential-traffic-only` 模式下，`allow_product_feedback` 返回 `false`。
- **304 响应后进程崩溃**：缓存文件已存在，下次启动可直接从文件恢复，数据不丢失。
- **策略键名拼写错误**：服务端返回了未知的策略键，由于 `isPolicyAllowed` 对未知键返回 `true`，拼写错误会导致策略实际上不生效。这要求前后端对策略键名有严格的契约。

### 6.3 改进建议

1. **引入变更事件广播**
   - 在 `pollPolicyLimits` 检测到 `sessionCache` 变化时，触发一个全局事件（如 `onPolicyLimitsChanged`），让 `commands.ts`、工具注册表、React 组件等订阅方能够动态刷新功能可用性，而无需重启 CLI。

2. **统一认证头构造**
   - 考虑将 `getAuthHeaders` 中的逻辑委托给 `auth.ts` 中一个低层级的、不依赖 `getSettings()` 的辅助函数，消除与 `auth.ts` 的隐式重复。

3. **异步化缓存预热路径**
   - 对于 `isPolicyAllowed` 的同步调用方，可以保留 `sessionCache` 的同步读取；但在首次调用且 `sessionCache` 为空时，可以触发一个后台异步读取（`loadCachedRestrictions` 的异步版本），减少同步文件 I/O 的阻塞时间。

4. **服务端 ETag 优先**
   - 若 API 响应头中包含 `ETag`，优先使用服务端提供的 ETag 进行缓存协商，而非本地计算的校验和。这需要在 `fetchPolicyLimits` 中解析响应头的 `etag` 字段并持久化到缓存文件元数据中。

5. **增强测试隔离**
   - 为 `_resetPolicyLimitsForTesting` 增加可选的 `clearDiskCache` 参数，或在测试框架中通过 `mock` 覆盖 `getCachePath`，确保每个测试用例使用独立的临时目录。

6. **策略键名校验**
   - 在 `PolicyLimitsResponseSchema` 或 `fetchPolicyLimits` 中增加一个已知策略键的白名单/警告机制，当服务端返回未预期的键时记录 `warn` 日志，帮助前后端快速发现契约漂移。
