# 研究文档：src/commands/privacy-settings/privacy-settings.tsx

## 场景与职责

`src/commands/privacy-settings/privacy-settings.tsx` 是 Claude Code CLI 中 `/privacy-settings` 命令的**实际执行模块**，负责在用户触发命令后完成完整的交互式隐私设置流程。其核心职责包括：

- **资格预检**：在用户调用命令时，二次确认其是否具备使用 Grove 隐私设置功能的资格。
- **数据拉取**：并行从服务端获取用户账户设置（`AccountSettings`）和 Grove 通知配置（`GroveConfig`）。
- **UI 分支渲染**：根据用户是否已接受过 Grove 条款，决定渲染条款同意弹窗（`GroveDialog`）还是日常设置管理弹窗（`PrivacySettingsDialog`）。
- **状态变更反馈**：在用户修改隐私设置后，重新拉取最新状态，向用户回显结果，并上报分析事件。
- **失败降级**：在 API 不可用或用户无资格时，以文本形式回退到网页版隐私设置链接，避免流程卡死。

该文件是 `local-jsx` 类型命令的典型实现，直接操作 React/Ink 组件树，并与后端 API、分析系统、全局配置缓存深度交互。

---

## 功能点目的

### 2.1 资格检查与失败降级

命令执行的第一步是调用 `isQualifiedForGrove()`。如果用户不符合条件（非 consumer 订阅者、无有效 accountId、本地缓存未命中且后台获取未完成），则通过 `onDone(FALLBACK_MESSAGE)` 向用户展示网页版隐私设置链接，并返回 `null`（不渲染任何 UI）。

这确保了：
- 即使用户通过某种方式“看到”了命令（如缓存的命令列表），实际调用时也会被安全拒绝。
- 在 Grove 功能未对该用户开启时，提供可操作的替代路径。

### 2.2 双模式弹窗呈现

根据用户账户设置中的 `grove_enabled` 字段，命令会进入两条不同的渲染路径：

| 用户状态 | 渲染组件 | 交互模式 |
|---------|---------|---------|
| `grove_enabled === null`（从未选择） | `GroveDialog` | 强制/半强制条款同意，用户必须选择 opt-in（帮助改进 Claude：ON）或 opt-out（OFF） |
| `grove_enabled !== null`（已做过选择） | `PrivacySettingsDialog` | 日常管理，用户可通过 Enter/Tab/Space 切换开关状态 |

### 2.3 状态变更追踪与分析

当用户在 `PrivacySettingsDialog` 中切换开关，或在 `GroveDialog` 中做出选择后，命令会：
1. 重新调用 `getGroveSettings()` 获取服务端最新状态。
2. 向用户输出确认消息：`"Help improve Claude" set to <true|false>.`
3. 如果状态确实发生了变化（旧值 ≠ 新值），上报分析事件 `tengu_grove_policy_toggled`。

### 2.4 与启动时 Grove 检查的区分

虽然 `GroveDialog` 组件也被 `src/interactiveHelpers.tsx` 在启动时调用，但 `privacy-settings.tsx` 中的使用场景不同：
- **启动时**：`showIfAlreadyViewed={false}`，用于在用户未查看过通知时主动弹出提醒。
- **命令触发时**：`showIfAlreadyViewed={true}`，即使用户已经查看过，也会再次展示（因为用户主动请求查看隐私设置）。

---

## 具体技术实现

### 3.1 入口函数与类型签名

```ts
export async function call(
  onDone: LocalJSXCommandOnDone,
): Promise<React.ReactNode | null>
```

- `onDone` 是命令框架提供的回调，用于在命令完成时向 REPL 输出结果文本、控制显示方式（`display: 'system'` 或 `'user'`）、或触发后续模型查询。
- 返回 `React.ReactNode` 时，REPL 会将其作为 fullscreen JSX 组件渲染；返回 `null` 时，不渲染任何 UI，仅依赖 `onDone` 的文本输出。

### 3.2 核心执行流程

```ts
const FALLBACK_MESSAGE = 'Review and manage your privacy settings at https://claude.ai/settings/data-privacy-controls';

export async function call(onDone: LocalJSXCommandOnDone): Promise<React.ReactNode | null> {
  // 1. 资格检查
  const qualified = await isQualifiedForGrove();
  if (!qualified) {
    onDone(FALLBACK_MESSAGE);
    return null;
  }

  // 2. 并行拉取设置与配置
  const [settingsResult, configResult] = await Promise.all([
    getGroveSettings(),
    getGroveNoticeConfig(),
  ]);

  // 3. API 失败兜底
  if (!settingsResult.success) {
    onDone(FALLBACK_MESSAGE);
    return null;
  }

  const settings = settingsResult.data;
  const config = configResult.success ? configResult.data : null;

  // 4. 定义完成回调
  async function onDoneWithDecision(decision: GroveDecision) { ... }
  async function onDoneWithSettingsCheck() { ... }

  // 5. 分支渲染
  if (settings.grove_enabled !== null) {
    return <PrivacySettingsDialog settings={settings} domainExcluded={config?.domain_excluded} onDone={onDoneWithSettingsCheck} />;
  }

  return <GroveDialog showIfAlreadyViewed={true} onDone={onDoneWithDecision} location={'settings'} />;
}
```

### 3.3 回调函数详解

#### `onDoneWithDecision`

处理 `GroveDialog` 的关闭/选择结果：

```ts
async function onDoneWithDecision(decision: GroveDecision) {
  if (decision === 'escape' || decision === 'defer') {
    onDone('Privacy settings dialog dismissed', { display: 'system' });
    return;
  }
  await onDoneWithSettingsCheck();
}
```

- `'escape'`：用户在 grace period 结束后按 Esc 退出（强制流程中的“拒绝”）。
- `'defer'`：用户在 grace period 内选择“Not now”延后处理。
- `'accept_opt_in'` / `'accept_opt_out'`：用户做出了明确选择，进入状态确认流程。

#### `onDoneWithSettingsCheck`

处理状态变更后的确认与上报：

```ts
async function onDoneWithSettingsCheck() {
  const updatedSettingsResult = await getGroveSettings();
  if (!updatedSettingsResult.success) {
    onDone('Unable to retrieve updated privacy settings', { display: 'system' });
    return;
  }
  const updatedSettings = updatedSettingsResult.data;
  const groveStatus = updatedSettings.grove_enabled ? 'true' : 'false';
  onDone(`"Help improve Claude" set to ${groveStatus}.`);

  if (settings.grove_enabled !== null && settings.grove_enabled !== updatedSettings.grove_enabled) {
    logEvent('tengu_grove_policy_toggled', {
      state: updatedSettings.grove_enabled as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
      location: 'settings' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
    });
  }
}
```

关键逻辑：
- 分析事件 `tengu_grove_policy_toggled` **仅在旧值不为 null 且新旧值不同**时上报。这意味着：
  - 首次从 `GroveDialog` 做出选择时（旧值为 null），不会触发 `tengu_grove_policy_toggled`。
  - 从 `PrivacySettingsDialog` 切换开关时（旧值为 boolean），会触发该事件。
- 这种设计避免了将首次同意行为与后续的主动切换行为混淆在同一个事件名中。`GroveDialog` 有自己的上报事件（`tengu_grove_policy_submitted`）。

### 3.4 关键数据结构

#### `AccountSettings`（来自 `src/services/api/grove.ts`）

```ts
export type AccountSettings = {
  grove_enabled: boolean | null   // null = 尚未做出选择
  grove_notice_viewed_at: string | null  // ISO 8601 时间戳
}
```

#### `GroveConfig`（来自 `src/services/api/grove.ts`）

```ts
export type GroveConfig = {
  grove_enabled: boolean          // 服务端是否启用 Grove（功能开关）
  domain_excluded: boolean        // 用户是否因域名策略被排除在 opt-in 之外
  notice_is_grace_period: boolean // 是否处于宽限期（宽限期内可 defer）
  notice_reminder_frequency: number | null  // 提醒频率（天）
}
```

#### `GroveDecision`（来自 `src/components/grove/Grove.tsx`）

```ts
export type GroveDecision =
  | 'accept_opt_in'
  | 'accept_opt_out'
  | 'defer'
  | 'escape'
  | 'skip_rendering'
```

### 3.5 API 调用与缓存机制

本文件直接依赖 `src/services/api/grove.ts` 提供的以下函数：

| 函数 | 作用 | 缓存策略 |
|------|------|---------|
| `isQualifiedForGrove()` | 判断用户是否有资格 | 基于 `globalConfig.groveConfigCache`，24h TTL，非阻塞 |
| `getGroveSettings()` | 获取账户设置 | `memoize` 会话级缓存，失败时自动 `cache.clear()` |
| `getGroveNoticeConfig()` | 获取 Grove 配置 | `memoize` 会话级缓存，3s 超时 |

缓存失效点：
- `updateGroveSettings()`（在 `PrivacySettingsDialog` 和 `GroveDialog` 内部调用）成功后会清除 `getGroveSettings` 的缓存。
- `markGroveNoticeViewed()` 成功后也会清除该缓存。
- 登出命令（`src/commands/logout/logout.tsx`）会同时清除 `getGroveSettings` 和 `getGroveNoticeConfig` 的缓存。

### 3.6 隐私级别兼容

`src/services/api/grove.ts` 中的 API 函数会检查 `isEssentialTrafficOnly()`（来自 `src/utils/privacyLevel.ts`）。当环境变量 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 被设置时：
- `getGroveSettings()` 直接返回 `{ success: false }`。
- `getGroveNoticeConfig()` 直接返回 `{ success: false }`。
- `isQualifiedForGrove()` 因依赖缓存，若缓存不存在则返回 false。

这导致 `privacy-settings.tsx` 会走 `onDone(FALLBACK_MESSAGE)` 的降级路径，不在 CLI 内渲染任何弹窗，也不发起任何网络请求。

---

## 关键代码路径与文件引用

### 4.1 命令执行完整链路

```
用户输入 /privacy-settings
  ↓
src/screens/REPL.tsx → processUserInput / processSlashCommand.tsx
  ↓
匹配到 Command 对象 → command.load() → import('./privacy-settings.js')
  ↓
src/commands/privacy-settings/privacy-settings.tsx::call(onDone)
  ↓
  ├─ isQualifiedForGrove() ──→ src/services/api/grove.ts
  ├─ getGroveSettings() ─────→ src/services/api/grove.ts ──→ Anthropic OAuth API
  ├─ getGroveNoticeConfig() ──→ src/services/api/grove.ts ──→ Anthropic OAuth API
  ↓
  分支 A: settings.grove_enabled !== null
    → <PrivacySettingsDialog /> ──→ src/components/grove/Grove.tsx
    → 用户切换 → updateGroveSettings() ──→ src/services/api/grove.ts
    → onDoneWithSettingsCheck() → onDone(...)
  
  分支 B: settings.grove_enabled === null
    → <GroveDialog showIfAlreadyViewed={true} location="settings" />
      ──→ src/components/grove/Grove.tsx
    → 用户选择 → updateGroveSettings(true/false)
    → onDoneWithDecision() → onDoneWithSettingsCheck() → onDone(...)
```

### 4.2 直接依赖文件

| 依赖 | 路径 | 用途 |
|------|------|------|
| React | `'react'` | JSX 运行时 |
| `GroveDialog`, `PrivacySettingsDialog`, `GroveDecision` | `../../components/grove/Grove.js` → `src/components/grove/Grove.tsx` | UI 组件 |
| `logEvent` | `../../services/analytics/index.js` → `src/services/analytics/index.ts` | 分析事件上报 |
| `getGroveNoticeConfig`, `getGroveSettings`, `isQualifiedForGrove` | `../../services/api/grove.js` → `src/services/api/grove.ts` | Grove API 封装 |
| `LocalJSXCommandOnDone` | `../../types/command.js` → `src/types/command.ts` | `onDone` 类型定义 |

### 4.3 相关分析事件

| 事件名 | 触发位置 | 说明 |
|--------|---------|------|
| `tengu_grove_policy_toggled` | `privacy-settings.tsx:42` | 用户在 `PrivacySettingsDialog` 中切换了开关 |
| `tengu_grove_policy_viewed` | `Grove.tsx:168` | `GroveDialog` 被展示 |
| `tengu_grove_policy_submitted` | `Grove.tsx:199/208` | 用户在 `GroveDialog` 中提交了选择 |
| `tengu_grove_policy_dismissed` | `Grove.tsx:216` | 用户选择了 defer |
| `tengu_grove_policy_escaped` | `Grove.tsx:223` | 用户按 Esc 退出 |
| `tengu_grove_privacy_settings_viewed` | `Grove.tsx:461` | `PrivacySettingsDialog` 被展示 |

---

## 依赖与外部交互

### 5.1 内部依赖模块

| 模块 | 用途 |
|------|------|
| `src/components/grove/Grove.tsx` | 提供 `GroveDialog` 和 `PrivacySettingsDialog` 两个 React 组件，以及 `GroveDecision` 类型 |
| `src/services/analytics/index.ts` | 提供 `logEvent` 函数，用于上报用户行为分析事件 |
| `src/services/api/grove.ts` | 提供 Grove 相关的所有 API 调用和资格判断逻辑 |
| `src/types/command.ts` | 提供 `LocalJSXCommandOnDone` 类型，定义 `onDone` 回调的签名 |

### 5.2 外部系统交互

通过 `src/services/api/grove.ts` 间接与以下外部系统交互：

- **Anthropic OAuth API**
  - `GET /api/oauth/account/settings` — 读取账户设置
  - `PATCH /api/oauth/account/settings` — 更新 `grove_enabled`
  - `POST /api/oauth/account/grove_notice_viewed` — 标记已查看
  - `GET /api/claude_code_grove` — 读取 Grove 功能配置

- **分析后端**
  - 通过 `logEvent` 异步上报到 Datadog 或 1P 事件收集系统。

### 5.3 运行时环境变量影响

| 环境变量 | 影响路径 | 效果 |
|----------|---------|------|
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `src/services/api/grove.ts` | 禁用 Grove API 调用，命令降级为 fallback message |
| `CLAUDE_CODE_OAUTH_TOKEN` / `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | `src/utils/auth.ts` | 决定 OAuth 认证是否可用，间接影响 `isQualifiedForGrove()` |
| `DISABLE_TELEMETRY` | `src/services/analytics/index.ts` | 可能抑制分析事件的上报（取决于 analytics sink 实现） |

---

## 风险、边界与改进建议

### 6.1 已知风险

1. **API 失败导致功能完全不可用**
   - 如果 `getGroveSettings()` 返回 `success: false`（网络故障、OAuth 过期、服务端错误），命令会直接 fallback 到网页链接，用户无法在 CLI 内完成任何操作。
   - 虽然 `getGroveSettings` 在失败时会清除自身缓存（`grove.ts:81`），但连续失败会使用户体验持续降级。

2. **并发状态更新竞争**
   - `PrivacySettingsDialog` 通过 `useInput` 监听键盘事件，每次 `Enter/Tab/Space` 都会立即调用 `updateGroveSettings(newValue)`。
   - 如果用户快速连按切换键，可能产生多个并发的 PATCH 请求，导致服务端状态与本地 UI 状态短暂不一致。
   - 当前实现没有 `inflight` 锁或 debounce 机制。

3. **分析事件上报条件过于严格**
   - `tengu_grove_policy_toggled` 仅在 `settings.grove_enabled !== null` 时上报。这意味着首次通过 `GroveDialog` 做出选择的用户不会触发此事件。
   - 如果数据分析团队期望统计“所有状态变更”，可能会遗漏首次同意/拒绝的数据（虽然 `GroveDialog` 有独立的 `tengu_grove_policy_submitted` 事件，但事件名不同，分析时需要额外关联）。

4. **`configResult` 失败时的静默降级**
   - `configResult`（`getGroveNoticeConfig` 的结果）失败时，代码仅将其设为 `null`：
     ```ts
     const config = configResult.success ? configResult.data : null;
     ```
   - 这会导致 `PrivacySettingsDialog` 的 `domainExcluded` 为 `undefined`，用户可能看到一个可交互的开关，但实际上其域名可能被服务端排除。虽然 `getGroveSettings()` 返回的 `grove_enabled` 已经是服务端的真实值，但 UI 上缺少 `domainExcluded` 的提示。

5. **`onDoneWithSettingsCheck` 的闭包陷阱**
   - `onDoneWithSettingsCheck` 内部引用了外层 `settings` 变量（用于判断状态是否变化）。
   - 如果 `PrivacySettingsDialog` 被长时间保持打开，而期间其他流程（如启动时的 Grove 检查）修改了全局缓存，闭包中的 `settings` 仍是旧值，可能导致分析事件漏报或误报。

### 6.2 边界情况

- **用户从未登录 claude.ai**：`isQualifiedForGrove()` 返回 false，直接 fallback。
- **OAuth token 过期**：API 调用触发 `withOAuth401Retry`，尝试刷新 token 后重试一次；若刷新失败则返回 `{ success: false }`，命令 fallback。
- **用户已选择但服务端重置为 null**：下次调用 `/privacy-settings` 会重新展示 `GroveDialog`。
- **domainExcluded 为 true**：`PrivacySettingsDialog` 显示 `false (for emails with your domain)` 且禁用键盘交互，但视觉上没有明确的“禁用”样式（如灰色 checkbox）。
- **Fullscreen 模式**：REPL 在 fullscreen 模式下会以居中 modal 渲染返回的 JSX，命令本身无需关心布局容器。

### 6.3 改进建议

1. **增加请求去重/防抖**
   在 `PrivacySettingsDialog` 的 `useInput` 处理中，为 `updateGroveSettings` 增加简单的 `inflight` 锁：
   ```ts
   const [isUpdating, setIsUpdating] = useState(false);
   useInput(async (input, key) => {
     if (isUpdating) return;
     if (!domainExcluded && (key.tab || key.return || input === ' ')) {
       setIsUpdating(true);
       const newValue = !groveEnabled;
       setGroveEnabled(newValue);
       await updateGroveSettings(newValue);
       setIsUpdating(false);
     }
   });
   ```

2. **优化 API 失败时的用户体验**
   当前 fallback 仅给出一个网页链接。可考虑在 `settingsResult.success === false` 时：
   - 若本地有缓存的 `globalConfig` 历史值，渲染一个只读弹窗显示最后已知状态，并附带“重试”按钮。
   - 或至少向用户说明失败原因（如“网络连接失败，请稍后重试”而非仅一个链接）。

3. **统一首次选择与后续切换的分析事件**
   考虑在 `onDoneWithSettingsCheck` 中，无论旧值是否为 null，只要最终值与初始值不同就上报 `tengu_grove_policy_toggled`。或者增加一个专门的事件（如 `tengu_grove_policy_initial_choice`）来捕获首次选择，使分析漏斗更完整。

4. **处理 `configResult` 失败时的 `domainExcluded` 不确定性**
   当 `configResult.success === false` 时，可考虑：
   - 向用户显示一个温和的警告（如“无法获取最新域名策略，显示状态可能不完整”）。
   - 或保守地将 `domainExcluded` 视为 `true`（禁用交互），直到配置成功获取。

5. **补充单元测试**
   当前未找到针对 `privacy-settings.tsx` 的测试文件。建议补充以下测试：
   - `call()` 在 `qualified === false` 时返回 `null` 并调用 `onDone(FALLBACK_MESSAGE)`。
   - `call()` 在 `settingsResult.success === false` 时的降级行为。
   - `call()` 在 `grove_enabled === null` 时渲染 `GroveDialog`。
   - `call()` 在 `grove_enabled !== null` 时渲染 `PrivacySettingsDialog`。
   - `onDoneWithSettingsCheck` 在状态变化时上报 `tengu_grove_policy_toggled`。
   - `onDoneWithSettingsCheck` 在状态未变化时不上报事件。
   - `onDoneWithSettingsCheck` 在 `updatedSettingsResult.success === false` 时输出错误消息。

6. **将 `FALLBACK_MESSAGE` 提取为配置常量**
   当前 `FALLBACK_MESSAGE` 是文件级硬编码字符串。如果网页 URL 发生变更，需要修改源码。建议将其提取到 `src/constants/urls.ts` 或类似位置，便于统一维护。
