# 研究文档：src/commands/passes/passes.tsx

## 场景与职责

本文件是 `/passes` 命令的**实际执行体（Local JSX Command Implementation）**。作为 `local-jsx` 类型命令，它导出一个 `call` 函数，负责：

1. 处理命令被调用时的初始化逻辑（首次访问追踪、配置持久化、分析埋点）。
2. 返回一个 React 节点 `<Passes onDone={onDone} />`，由 Ink 渲染引擎在终端内绘制 Guest Passes 的交互式 UI。

它是命令框架（`src/types/command.ts` 定义的 `LocalJSXCommandCall`）与业务 UI 组件（`src/components/Passes/Passes.tsx`）之间的桥梁。

## 功能点目的

### 1. 首次访问追踪（Upsell 抑制）
当用户第一次运行 `/passes` 时，将 `hasVisitedPasses` 标记为 `true`，并记录当前剩余的 passes 数量到 `passesLastSeenRemaining`。这用于：
- 阻止后续启动时的 Guest Passes upsell 提示（`src/components/LogoV2/GuestPassesUpsell.tsx`）。
- 为 `tipRegistry.ts` 中的 `guest-passes` tip 提供过滤条件。

### 2. 分析埋点
发送 `tengu_guest_passes_visited` 事件，携带 `is_first_visit` 布尔值，用于产品分析用户转化漏斗。

### 3. UI 渲染委托
将实际的终端 UI 渲染委托给 `src/components/Passes/Passes.tsx` 中的 `Passes` 组件，并传入 `onDone` 回调，以便用户在组件内完成交互后正确结束命令生命周期。

## 具体技术实现

### `call` 函数

```ts
export async function call(
  onDone: LocalJSXCommandOnDone,
): Promise<React.ReactNode>
```

执行流程：

1. **读取全局配置**：`const config = getGlobalConfig()`。
2. **判断首次访问**：`const isFirstVisit = !config.hasVisitedPasses`。
3. **首次访问时更新配置**：
   - 获取 `remaining = getCachedRemainingPasses()`。
   - 调用 `saveGlobalConfig(current => ({ ...current, hasVisitedPasses: true, passesLastSeenRemaining: remaining ?? current.passesLastSeenRemaining }))`。
   - 使用函数式 updater，确保基于最新配置写入，避免竞态。
4. **记录分析事件**：`logEvent('tengu_guest_passes_visited', { is_first_visit: isFirstVisit })`。
5. **返回 React 节点**：`return <Passes onDone={onDone} />`。

### `LocalJSXCommandOnDone` 回调

来自 `src/types/command.ts:117-126`：

```ts
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void
```

`Passes` 组件通过调用 `onDone` 来结束命令，例如：
- 用户按 Esc / Ctrl+C 取消时：`onDone('Guest passes dialog dismissed', { display: 'system' })`。
- 用户按 Enter 复制链接后：`onDone('Referral link copied to clipboard!')`。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands/passes/index.ts:21` | `load: () => import('./passes.js')` 指向本文件的编译产物 |
| `src/components/Passes/Passes.tsx` | 实际渲染 Guest Passes UI 的 Ink 组件 |
| `src/services/analytics/index.ts` | `logEvent` 实现，事件队列 + sink 模式 |
| `src/services/api/referral.ts:162-168` | `getCachedRemainingPasses()` 实现 |
| `src/utils/config.ts:313-316` | `hasVisitedPasses`、`passesLastSeenRemaining` 的配置类型 |
| `src/utils/config.ts:797-866` | `saveGlobalConfig()` 实现（带锁、回退、auth-loss 防护） |
| `src/components/LogoV2/GuestPassesUpsell.tsx` | 读取 `hasVisitedPasses` 和 `passesLastSeenRemaining` 的调用方 |
| `src/services/tips/tipRegistry.ts:591-608` | `guest-passes` tip，使用 `hasVisitedPasses` 做过滤 |

## 依赖与外部交互

### 直接依赖

- `react` — 返回 JSX 节点。
- `../../components/Passes/Passes.js` — `Passes` UI 组件。
- `../../services/analytics/index.js` — `logEvent`。
- `../../services/api/referral.js` — `getCachedRemainingPasses`。
- `../../types/command.js` — `LocalJSXCommandOnDone` 类型。
- `../../utils/config.js` — `getGlobalConfig`, `saveGlobalConfig`。

### 下游组件 `Passes.tsx` 的关键行为

`src/components/Passes/Passes.tsx` 是本命令的核心 UI，其内部逻辑值得一并研究：

#### 状态管理
- `loading`：初始为 `true`，在 `useEffect` 中异步加载数据。
- `isAvailable`：资格检查通过后设为 `true`。
- `passStatuses`：每个 pass 的可用状态数组。
- `referralLink`：推荐链接，供用户复制。
- `referrerReward`：奖励信息，用于显示带金额的文案。

#### 数据加载流程（`useEffect` 中的 `loadPassesData`）
1. 调用 `getCachedOrFetchPassesEligibility()`：
   - 若返回 `null` 或 `eligible === false`，显示 `"Guest passes are not currently available."`。
2. 提取 `referral_link`、`referrer_reward` 和 `campaign`。
3. 调用 `fetchReferralRedemptions(campaign)` 获取兑换记录：
   - 失败则同样显示不可用。
4. 根据 `redemptions` 和 `limit`（默认 3）构建 `PassStatus[]`。

#### 用户交互
- **Enter**：若 `referralLink` 存在，调用 `setClipboard(referralLink)` 复制到剪贴板，并发送 `tengu_guest_passes_link_copied` 事件，然后 `onDone('Referral link copied to clipboard!')`。
- **Esc / `confirm:no` keybinding**：调用 `handleCancel`，`onDone('Guest passes dialog dismissed', { display: 'system' })`。
- **Ctrl+C**：通过 `useExitOnCtrlCDWithKeybindings` 处理，行为同上。

#### 视觉呈现
- 使用 `Pane` 组件作为容器。
- 可用 pass 显示为 ASCII 艺术票券（`┌──────────┐`），已兑换的显示为灰色划掉的票券（`┌─────────╱`）。
- 文案根据 `referrerReward` 是否存在动态切换，并附带不同的支持文章链接。

## 风险、边界与改进建议

### 风险

1. **无异常捕获的初始化逻辑**：`call` 函数内部对 `getGlobalConfig()`、`saveGlobalConfig()` 和 `logEvent()` 的调用**没有包裹 try-catch**。如果配置系统出现意外（如磁盘权限问题、JSON 损坏），整个 `/passes` 命令会崩溃，用户体验较差。
2. **`saveGlobalConfig` 的同步写文件开销**：虽然 `saveGlobalConfig` 内部实现了文件锁和缓存写透（write-through），但它仍然是同步文件 I/O。在极端并发场景（如快速连续调用命令）下可能成为瓶颈。
3. **`fetchReferralRedemptions` 网络失败导致 UI 降级**：在 `Passes` 组件中，如果 redemptions 接口失败，UI 会直接降级为 `"not currently available"`，用户无法区分是网络问题还是真的不可用。

### 边界

- `getCachedRemainingPasses()` 可能返回 `null`（无缓存或 orgId 不存在）。此时 `passesLastSeenRemaining` 保持原值（`?? current.passesLastSeenRemaining`），不会覆盖为 `null`。
- `isFirstVisit` 的判断完全基于本地配置。如果用户手动删除 `~/.claude.json` 或切换机器，会再次被视为首次访问。
- `Passes` 组件的 `useEffect` 依赖数组为空（`[]`），只在挂载时加载一次数据。如果用户长时间停留在该 UI 中，不会自动刷新资格状态。

### 改进建议

1. **增强 `call` 函数的健壮性**：
   ```ts
   export async function call(onDone: LocalJSXCommandOnDone): Promise<React.ReactNode> {
     try {
       const config = getGlobalConfig()
       const isFirstVisit = !config.hasVisitedPasses
       if (isFirstVisit) {
         const remaining = getCachedRemainingPasses()
         saveGlobalConfig(current => ({
           ...current,
           hasVisitedPasses: true,
           passesLastSeenRemaining: remaining ?? current.passesLastSeenRemaining,
         }))
       }
       logEvent('tengu_guest_passes_visited', { is_first_visit: isFirstVisit })
     } catch (err) {
       logError(err as Error)
       // 继续渲染 UI，不因配置写入失败阻断用户
     }
     return <Passes onDone={onDone} />
   }
   ```

2. **区分网络错误与资格不足**：在 `Passes` 组件中，可以单独捕获 `fetchReferralRedemptions` 的异常并显示更友好的提示，例如 `"Unable to load pass details. Please check your connection and try again."`，而不是统一显示 `"not currently available"`。

3. **自动刷新机制**：考虑到用户可能在 `/passes` 页面停留较长时间，可以为 `useEffect` 添加定时刷新（如每 30 秒），或在窗口重新获得焦点时刷新资格数据，避免用户看到过期的 pass 状态。

4. **减少 `saveGlobalConfig` 调用频率**：当前 `call` 函数在每次 `/passes` 调用时都会写配置（即使 `hasVisitedPasses` 已经是 `true` 时不会写，但首次访问时必写）。可以考虑将 `hasVisitedPasses` 的标记延迟到用户实际与 `Passes` 组件交互后再写，但目前逻辑已足够简单，改动收益有限。
