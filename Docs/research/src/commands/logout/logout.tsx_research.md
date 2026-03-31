# 研究文档：src/commands/logout/logout.tsx

## 场景与职责

本文件是 `/logout` 命令的**实际实现模块**，负责执行用户登出的完整生命周期：清除凭证、刷新缓存、清理持久化状态、输出 UI 反馈并触发进程优雅退出。它同时导出 `performLogout` 和 `clearAuthRelatedCaches` 两个工具函数，供登录流程、CLI 子命令、OAuth 流等场景复用，确保"切换账号"或"重新登录"时旧状态不会泄漏。

## 功能点目的

1. **`performLogout(options)`**：核心登出逻辑。
   - 先 flush 遥测数据，防止在清除凭证后发送带有旧组织信息的事件。
   - 移除 API key（`removeApiKey`）。
   - 删除安全存储（keychain/明文文件）中的所有数据。
   - 清除与认证相关的各类内存缓存。
   - 更新全局配置，重置 onboarding 状态（可选）并清除 `oauthAccount`。

2. **`clearAuthRelatedCaches()`**：专门用于清除所有因用户/会话/认证变化而需要失效的缓存。
   - 被 `performLogout` 内部调用，也被 `src/cli/handlers/auth.ts` 的 `installOAuthTokens` 在登录成功后调用，确保新旧账号之间无缓存污染。

3. **`call()`**：`local-jsx` 命令的入口函数。
   - 调用 `performLogout({ clearOnboarding: true })`。
   - 返回 Ink `<Text>` 组件显示成功消息。
   - 通过 `setTimeout(..., 200)` 触发 `gracefulShutdownSync(0, 'logout')`，在短暂展示消息后退出进程。

## 具体技术实现

### 关键流程

#### `performLogout` 执行时序

```
1. flushTelemetry()          // 懒加载 ~1.1MB OpenTelemetry，确保事件不泄漏
2. removeApiKey()            // 清除 keychain/配置文件中的 API key
3. secureStorage.delete()    // 清空整个安全存储（含 OAuth tokens、trusted device token 等）
4. clearAuthRelatedCaches()  // 清除各类 memoized 缓存
5. saveGlobalConfig(updater) // 重置 oauthAccount、onboarding 等配置字段
```

#### `clearAuthRelatedCaches` 清除清单

| 缓存目标 | 清除方式 | 用途 |
|---------|---------|------|
| `getClaudeAIOAuthTokens` | `.cache?.clear?.()` | OAuth token 的 memoized 读取 |
| `clearTrustedDeviceTokenCache()` | 函数调用 | 桥接（remote control）受信任设备 token |
| `clearBetasCaches()` | 函数调用 | 模型 beta headers 的 memoization |
| `clearToolSchemaCache()` | 函数调用 | 工具 schema 的 session 缓存 |
| `resetUserCache()` | 函数调用 | 用户核心数据（email、org、subscription） |
| `refreshGrowthBookAfterAuthChange()` | 函数调用 | 触发 GrowthBook 客户端重建，使用新 auth headers |
| `getGroveNoticeConfig` / `getGroveSettings` | `.cache?.clear?.()` | Grove 隐私通知配置的 memoized API 结果 |
| `clearRemoteManagedSettingsCache()` | `await` | 远程托管设置（企业策略）的 session + 文件缓存 |
| `clearPolicyLimitsCache()` | `await` | 组织策略限制的 session + 文件缓存 |

### 配置更新逻辑（`saveGlobalConfig`）

当 `clearOnboarding: true` 时（仅 `/logout` 命令入口，CLI `auth logout` 为 `false`）：
- `hasCompletedOnboarding = false`
- `subscriptionNoticeCount = 0`
- `hasAvailableSubscription = false`
- `customApiKeyResponses.approved = []`（若存在）
- `oauthAccount = undefined`

### 懒加载设计

```ts
const { flushTelemetry } = await import('../../utils/telemetry/instrumentation.js')
```

注释明确说明：OpenTelemetry 依赖包约 1.1MB，若静态导入会拖慢启动速度；仅在真正执行 logout 时动态加载。

## 关键代码路径与文件引用

### 直接依赖（被 import）

| 文件 | 导入符号 | 作用 |
|------|---------|------|
| `src/bridge/trustedDevice.js` | `clearTrustedDeviceTokenCache` | 清除桥接受信任设备 token 缓存 |
| `src/ink.js` | `Text` | Ink UI 组件 |
| `src/services/analytics/growthbook.js` | `refreshGrowthBookAfterAuthChange` | 认证变化后重建 GrowthBook 客户端 |
| `src/services/api/grove.js` | `getGroveNoticeConfig`, `getGroveSettings` | Grove 通知配置（缓存清除） |
| `src/services/policyLimits/index.js` | `clearPolicyLimitsCache` | 清除组织策略限制缓存 |
| `src/services/remoteManagedSettings/index.js` | `clearRemoteManagedSettingsCache` | 清除远程托管设置缓存 |
| `src/utils/auth.js` | `getClaudeAIOAuthTokens`, `removeApiKey` | OAuth token 缓存读取、API key 移除 |
| `src/utils/betas.js` | `clearBetasCaches` | 清除 beta header 缓存 |
| `src/utils/config.js` | `saveGlobalConfig` | 持久化全局配置更新 |
| `src/utils/gracefulShutdown.js` | `gracefulShutdownSync` | 同步触发优雅退出 |
| `src/utils/secureStorage/index.js` | `getSecureStorage` | 获取平台对应的安全存储实现 |
| `src/utils/toolSchemaCache.js` | `clearToolSchemaCache` | 清除工具 schema 缓存 |
| `src/utils/user.js` | `resetUserCache` | 重置用户数据缓存 |

### 调用方（外部使用）

| 文件 | 调用符号 | 场景 |
|------|---------|------|
| `src/cli/handlers/auth.ts:52` | `performLogout({ clearOnboarding: false })` | `installOAuthTokens`：登录新账号前清旧状态 |
| `src/cli/handlers/auth.ts:109` | `clearAuthRelatedCaches()` | `installOAuthTokens`：登录成功后再次刷新缓存 |
| `src/cli/handlers/auth.ts:323` | `performLogout({ clearOnboarding: false })` | `authLogout()`：CLI `claude auth logout` 子命令 |
| `src/commands/logout/index.ts` | `load: () => import('./logout.js')` | `/logout` slash command 懒加载入口 |

### 类型与调度相关

| 文件 | 说明 |
|------|------|
| `src/types/command.ts:131-135` | `LocalJSXCommandCall` 类型定义，`call` 函数签名来源 |
| `src/types/command.ts:140-142` | `LocalJSXCommandModule` 类型，`call` 为必需导出 |

## 依赖与外部交互

### 安全存储（Secure Storage）

`getSecureStorage()` 根据平台返回不同实现（`src/utils/secureStorage/index.ts`）：
- **macOS**：`createFallbackStorage(macOsKeychainStorage, plainTextStorage)` — 优先 keychain，失败回退到明文文件。
- **Linux/其他**：`plainTextStorage` — 当前无 libsecret 支持，直接写文件。

`secureStorage.delete()` 会**清空整个存储容器**，这意味着不仅 OAuth tokens，任何其他保存在同一存储中的敏感数据（如 trusted device token）也会被一并删除。

### 遥测 Flush

`flushTelemetry()` 来自 `src/utils/telemetry/instrumentation.ts`，负责将 OpenTelemetry 的 spans、metrics、logs 批量发送到后端。在清除凭证**之前**调用是关键设计：若顺序颠倒，后续 flush 可能因缺少 auth headers 失败，或更糟——在已切换账号后仍携带旧账号的 org UUID 发送事件，导致数据归属错误。

### GrowthBook 刷新

`refreshGrowthBookAfterAuthChange()` 会完全销毁旧 GrowthBook 客户端并重建，因为 `apiHostRequestHeaders` 在创建后不可变。logout 后 auth headers 失效，必须重建；否则后续 feature flag 读取会使用旧的 Bearer token，导致 401 或错误的实验分组。

### 进程退出

`call()` 中的 `setTimeout(() => gracefulShutdownSync(0, 'logout'), 200)`：
- 200ms 的延迟是为了让 Ink 有时间渲染成功消息。
- `gracefulShutdownSync` 设置 `process.exitCode`，执行终端模式清理（alt screen、mouse tracking、kitty keyboard 等），运行 cleanup hooks，flush 分析数据，最终 `process.exit(0)`。

## 风险、边界与改进建议

### 风险与边界

1. **`secureStorage.delete()` 过于宽泛**
   - 当前实现是"全量删除"而非"选择性删除"。如果未来在安全存储中存放了与 auth 无关的数据（如用户自定义的加密笔记），logout 会误删。
   - 相关代码：`secureStorage.delete()` 在 `plainTextStorage` 和 `macOsKeychainStorage` 中都是整库/整服务清空。

2. **`performLogout` 与 `call` 的 `clearOnboarding` 差异**
   - `/logout` 命令（`call`）会重置 onboarding，而 CLI `auth logout`（`authLogout`）不会。这一差异是设计上的（防止用户误操作后重新经历 onboarding），但文档中未明确说明，可能导致维护者困惑。

3. **缓存清除顺序敏感**
   - `resetUserCache()` 必须在 `refreshGrowthBookAfterAuthChange()` 之前执行，因为 GrowthBook 重建时需要读取最新的用户属性（如 org UUID、email）。当前代码顺序正确，但未来重构时容易颠倒。

4. **`clearAuthRelatedCaches` 中部分操作未 `await`**
   - `clearRemoteManagedSettingsCache()` 和 `clearPolicyLimitsCache()` 是 async 的，在 `clearAuthRelatedCaches` 中被 `await`；但 `getClaudeAIOAuthTokens.cache?.clear?.()` 等同步操作若抛出异常（虽然概率极低），会中断后续清除步骤。

5. **无测试覆盖**
   - `src/commands/logout/` 目录下无任何 `.test.ts` 或 `.test.tsx` 文件。`performLogout` 涉及大量外部副作用（keychain、文件 I/O、网络 flush），缺少测试意味着重构风险高。

6. **OpenTelemetry 懒加载失败无降级**
   - 若 `flushTelemetry` 的动态导入失败（如网络包损坏），`performLogout` 会直接抛出异常，导致登出流程中断。虽然外层调用方（如 `authLogout`）有 `try/catch`，但 TUI `/logout` 路径（`call`）未显式捕获。

### 改进建议

1. **细化安全存储清理粒度**
   - 将 `secureStorage.delete()` 替换为显式的字段级删除（如 `secureStorage.update({ apiKey: undefined, oauthTokens: undefined, ... })`），避免误删非认证数据。

2. **统一异常处理**
   - 在 `call()` 中包裹 `try/catch`，对 `performLogout` 的异常输出友好的错误消息（如 "Failed to log out: <reason>"），而不是让 Ink 渲染层暴露原始堆栈。

3. **补充集成测试**
   - 使用 mock 的 `secureStorage`、`saveGlobalConfig` 和 `removeApiKey` 编写测试，验证：
     - `performLogout` 的调用顺序（flush → removeApiKey → delete → caches → config）。
     - `clearOnboarding: true/false` 时配置更新的差异。
     - `clearAuthRelatedCaches` 确实清除了所有列出的缓存。

4. **文档化 `clearOnboarding` 语义**
   - 在 `performLogout` 的 JSDoc 中明确说明 `clearOnboarding` 的行为差异及使用场景。

5. **考虑将 `flushTelemetry` 的导入失败降级为警告**
   - 若 OpenTelemetry 模块不可用，记录 debug 日志后继续执行登出，避免阻塞用户退出。
