# 研究文档：src/utils/user.ts

## 场景与职责

本模块负责收集、组装和缓存**核心用户数据（CoreUserData）**，为整个应用的 analytics（Statsig、GrowthBook）、遥测、用户识别和个性化功能提供统一的数据源。它是连接认证系统（OAuth/API Key）、本地配置（`~/.claude/settings.json`）、git 环境、以及 CI 环境（GitHub Actions）的枢纽。

数据消费方包括：
- GrowthBook feature flag 评估（`getUserForGrowthBook`）；
- Statsig 事件上报（`getCoreUserData` 作为基础 payload）；
- 会话恢复、成本追踪、远程设置同步等需要用户身份标识的子系统。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `initUser()` | 启动时异步预取用户 email，使后续 `getCoreUserData()` 可同步返回完整数据。 |
| `getCoreUserData` | Memoized 同步函数，返回核心用户数据对象。可选参数 `includeAnalyticsMetadata` 控制是否附加订阅类型、速率限制层级、首次 token 时间等敏感信息。 |
| `getUserForGrowthBook()` | 包装 `getCoreUserData(true)`，专供 GrowthBook 初始化使用。 |
| `resetUserCache()` | 在登录/登出/账户切换时清空所有缓存，确保下次调用获取最新身份。 |
| `getGitEmail` | 异步通过 `git config --get user.email` 获取用户 git 邮箱，用于 ant 内部用户的身份识别。 |

## 具体技术实现

### 1. Email 异步预取与缓存

```ts
let cachedEmail: string | undefined | null = null
let emailFetchPromise: Promise<string | undefined> | null = null
```

- `initUser()` 在应用启动早期调用，触发 `getEmailAsync()`。
- 使用 `cachedEmail === null` 作为“尚未获取”的哨兵，`emailFetchPromise` 防止并发重复请求。
- 获取完成后，调用 `getCoreUserData.cache.clear?.()` 清除 memoize 缓存，确保下一次 `getCoreUserData` 调用能带上新 email。

### 2. 核心用户数据结构 `CoreUserData`

| 字段 | 来源 |
|------|------|
| `deviceId` | `getOrCreateUserID()`（本地配置中的稳定 UUID） |
| `sessionId` | `getSessionId()`（`src/bootstrap/state.ts`） |
| `email` | `getEmail()`（OAuth → COO_CREATOR → git email 的优先级链） |
| `appVersion` | `MACRO.VERSION`（构建时注入） |
| `platform` | `getHostPlatformForAnalytics()`（`src/utils/env.ts`） |
| `organizationUuid` / `accountUuid` | `getOauthAccountInfo()`（仅当活跃使用 OAuth 时） |
| `userType` | `process.env.USER_TYPE`（`ant` 或 `external`） |
| `subscriptionType` / `rateLimitTier` / `firstTokenTime` | 仅在 `includeAnalyticsMetadata === true` 时附加 |
| `githubActionsMetadata` | 当 `GITHUB_ACTIONS` 为 truthy 时，从对应环境变量提取 |

### 3. Email 解析优先级链 `getEmailAsync()`

1. **OAuth 邮箱**：若用户当前通过 OAuth 认证，直接使用 `oauthAccountInfo.emailAddress`。
2. **Ant-only 分支**（`USER_TYPE === 'ant'`）：
   - `COO_CREATOR` 环境变量 → `'{COO_CREATOR}@anthropic.com'`。
   - `git config --get user.email`（通过 `execa` 异步执行）。
3. **外部用户**：若未走 OAuth，返回 `undefined`（不阻塞、不暴露本地 git 信息）。

### 4. Memoize 缓存策略

`getCoreUserData` 使用 `lodash-es/memoize` 缓存，缓存 key 仅为 `includeAnalyticsMetadata?: boolean`（布尔值或 undefined）。这意味着：

- 在同一布尔参数下，即使底层依赖（如 OAuth 账户、git email、环境变量）发生变化，返回值也不会更新。
- `resetUserCache()` 通过 `getCoreUserData.cache.clear?.()` 手动打破缓存，是状态同步的主要手段。

### 5. Git Email 获取

`getGitEmail` 也是 memoized 的，使用 `execa('git config --get user.email', { shell: true, reject: false, cwd: getCwd() })`：

- `shell: true` 允许直接传递整条命令字符串。
- `reject: false` 确保命令失败时不会抛异常，而是返回 `undefined`。
- `cwd` 使用当前工作目录，以尊重用户可能设置的本地 git 配置。

## 关键代码路径与文件引用

- **主实现**：`src/utils/user.ts`（194 行）
- **认证信息**：`src/utils/auth.ts`（`getOauthAccountInfo`、`getSubscriptionType`、`getRateLimitTier`）
- **本地配置/用户 ID**：`src/utils/config.ts`（`getOrCreateUserID`、`getGlobalConfig`）
- **会话状态**：`src/bootstrap/state.ts`（`getSessionId`）
- **平台信息**：`src/utils/env.ts`（`getHostPlatformForAnalytics`）
- **CWD**：`src/utils/cwd.ts`（`getCwd`）

## 依赖与外部交互

- **`execa`**：执行 `git config` 子进程。
- **`lodash-es/memoize.js`**：`getCoreUserData` 和 `getGitEmail` 的缓存。
- **`src/services/analytics/index.js`**：消费方通过 `getCoreUserData` 获取数据后发送事件。
- **`src/utils/auth.js`**：OAuth 账户信息、订阅类型、速率限制层级。
- **`src/utils/config.js`**：用户 ID、全局配置（`claudeCodeFirstTokenDate`）。
- **`src/bootstrap/state.js`**：会话 ID。
- **`src/utils/cwd.js`**：当前工作目录。
- **`src/utils/env.js`**：主机平台信息。

## 风险、边界与改进建议

### 风险

1. **Memoize 缓存 key 过粗**：`getCoreUserData` 只按 `includeAnalyticsMetadata` 缓存。若 mid-session 发生 OAuth 账户切换（如用户通过 `/login` 换号），但参数相同，缓存不会自动刷新。虽然 `resetUserCache()` 是标准做法，但若某个调用方忘记调用，后续 analytics 会继续上报旧身份，导致数据污染。
2. **`shell: true` 的潜在安全隐患**：`getGitEmail` 使用 `shell: true` 执行固定字符串命令，风险较低。但如果 `getCwd()` 返回的路径被恶意构造（包含 shell 元字符），理论上可能引发命令注入。由于 `cwd` 是传给 `execa` 的选项而非直接拼接到命令中，`execa` 内部会对其进行转义，但实际安全性取决于 `execa` 版本实现。
3. **Email 获取的阻塞窗口**：`initUser()` 若未被调用（如某些非标准入口点），`getEmail()` 会直接返回 `undefined`。这会导致 GrowthBook/Statsig 初始化时缺少 email，可能影响基于用户身份的实验分组准确性。

### 边界

- **外部用户无 git email**：`getEmailAsync()` 在非 ant 构建中，若 OAuth 无邮箱，直接返回 `undefined`，不会尝试读取本地 git 配置。这是隐私设计的一部分。
- **GitHub Actions 元数据仅在 CI 环境附加**：普通本地运行不会包含 `actor`、`repository` 等字段。
- **OAuth 信息“仅活跃时包含”**：即使 keychain/secure storage 中存有过期 OAuth token，只要当前未走 OAuth 认证流程，`organizationUuid` 和 `accountUuid` 就不会出现在 `CoreUserData` 中。

### 改进建议

1. **细化 memoize key 或引入依赖哈希**：将 `getCoreUserData` 的缓存 key 扩展为包含 `deviceId + sessionId + oauthAccountUuid` 等稳定标识的复合 key，或改用基于依赖项变更的显式缓存失效机制，减少对 `resetUserCache()` 的人工调用依赖。
2. **移除 `shell: true`**：将 `getGitEmail` 改为 `execa('git', ['config', '--get', 'user.email'], { cwd: getCwd(), reject: false })`，彻底消除任何潜在的 shell 注入面。
3. **增加 `initUser()` 的强制调用检查**：在 `getCoreUserData` 中增加开发模式断言（`process.env.NODE_ENV === 'development'`），若 `cachedEmail === null` 且未显式标记为“跳过 email”，则打印 warning，提醒开发者调用 `initUser()`。
4. **缓存 GitHub Actions 元数据**：`githubActionsMetadata` 目前每次 `getCoreUserData` 调用都重新从 `process.env` 读取。由于环境变量在进程生命周期内不变，可将其也 memoize 以减少对象分配。
