# src/utils/logoV2Utils.ts 研究文档

## 场景与职责

`logoV2Utils.ts` 为 Claude Code CLI 的启动界面（Logo V2 和 CondensedLogo 组件）提供纯计算型的布局和数据准备逻辑。由于 Ink（React for CLI）的渲染是同步的，所有数据准备和尺寸计算必须在渲染前完成。该模块将复杂的布局数学、文本格式化、数据获取逻辑从 React 组件中剥离出来，使组件保持声明式且易于测试。

主要职责：
1. **布局计算**：根据终端宽度计算 Logo V2 的左右面板宽度。
2. **文本格式化**：欢迎消息、路径截断、模型/账单信息格式化。
3. **最近活动加载**：预加载最近会话列表供 Logo V2 右侧展示。
4. **发布说明加载**：为 ants 和外部用户分别获取最近的发布说明。
5. **Logo 显示数据聚合**：版本号、当前路径、账单类型、代理名称等。

调用方：
- `src/components/LogoV2/LogoV2.tsx`
- `src/components/LogoV2/CondensedLogo.tsx`
- `src/setup.ts`

## 功能点目的

### 1. `getLayoutMode` / `calculateLayoutDimensions`
根据终端列数决定布局模式：
- `columns >= 70` → `horizontal`（左右分栏）
- 否则 → `compact`（上下堆叠）

`calculateLayoutDimensions` 计算左右面板的精确宽度，考虑边框填充 (`BORDER_PADDING=4`)、内容填充 (`CONTENT_PADDING=2`)、分隔线 (`DIVIDER_WIDTH=1`) 和 `optimalLeftWidth`。

### 2. `calculateOptimalLeftWidth`
基于三个内容元素（欢迎消息、截断后的 cwd、模型信息行）的实际字符串宽度，计算左面板的最佳宽度。使用 `stringWidth`（CJK/emoji 感知）而非 `String.prototype.length`。

### 3. `formatWelcomeMessage`
根据用户名生成欢迎语：
- 有用户名且长度 ≤ 20：`Welcome back {username}!`
- 否则：`Welcome back!`

### 4. `truncatePath`
智能路径截断函数，是 `src/utils/format.ts` 中 `truncatePathMiddle` 的增强/变体。特点：
- 使用 `stringWidth` 进行宽度感知测量。
- 优先保留路径的首尾部分，用 `…` 省略中间。
- 处理 Unix 根路径（`first === ''`）的特殊情况。
- 在可用空间内尽可能保留更多的中间路径段。

### 5. `getRecentActivity` / `getRecentActivitySync`
异步预加载最近 3 条有意义的非当前会话记录：
- 过滤 sidechain 会话、当前会话、包含 "I apologize" 的摘要。
- 要求至少存在 `summary` 或 `firstPrompt`（且不为 "No prompt"）。
- 使用 Promise 缓存（`cachePromise`），防止并发重复加载。

### 6. `getLogoDisplayData`
聚合 Logo 显示所需的基础数据：
- `version`：优先使用 `DEMO_VERSION` 环境变量，否则 `MACRO.VERSION`。
- `cwd`：显示路径，直连服务器时附加服务器域名。
- `billingType`：订阅者显示订阅名，否则显示 "API Usage Billing"。
- `agentName`：从初始设置中读取 `--agent` 参数。

### 7. `formatModelAndBilling`
根据可用宽度决定模型名和账单信息的展示方式：
- 若总宽度足够，显示在同一行（`model · billing`）。
- 否则拆分为两行，并分别截断。

### 8. `getRecentReleaseNotesSync`
获取最近发布说明：
- **Ant 用户**：使用构建时打包的 `MACRO.VERSION_CHANGELOG`（Git 提交日志），直接返回前 N 条。
- **外部用户**：从内存缓存的 changelog 解析，按版本号降序取最近 3 个版本的说明，再截取前 `maxItems` 条。

## 具体技术实现

### 布局常量
```typescript
const MAX_LEFT_WIDTH = 50
const MAX_USERNAME_LENGTH = 20
const BORDER_PADDING = 4
const DIVIDER_WIDTH = 1
const CONTENT_PADDING = 2
```

### 路径截断算法核心逻辑
```typescript
// 多段路径的截断策略
let available = maxLength - firstWidth - lastWidth - ellipsisWidth - 2 * separatorWidth
const middleParts = []
for (let i = parts.length - 2; i > 0; i--) {
  const part = parts[i]
  if (part && stringWidth(part) + separatorWidth <= available) {
    middleParts.unshift(part)
    available -= stringWidth(part) + separatorWidth
  } else {
    break
  }
}
```

该算法从倒数第二个段开始向中间遍历，尽可能多地保留靠近末尾的中间段。这是一种偏向保留路径深层结构的策略（因为深层目录名通常比浅层更具区分性）。

### 最近活动缓存
```typescript
let cachedActivity: LogOption[] = []
let cachePromise: Promise<LogOption[]> | null = null

export async function getRecentActivity(): Promise<LogOption[]> {
  if (cachePromise) return cachePromise
  cachePromise = loadMessageLogs(10)
    .then(logs => { /* 过滤和截取 */ })
    .catch(() => { cachedActivity = []; return cachedActivity })
  return cachePromise
}
```

注意：`loadMessageLogs(10)` 加载最近 10 条消息日志，然后过滤出最多 3 条。这里加载 10 条是为了给过滤留足余量。

### 发布说明来源差异
```typescript
if (process.env.USER_TYPE === 'ant') {
  const changelog = MACRO.VERSION_CHANGELOG
  if (changelog) {
    const commits = changelog.trim().split('\n').filter(Boolean)
    return commits.slice(0, maxItems)
  }
  return []
}
```

Ant 用户直接读取构建宏注入的提交日志，无需网络请求，因此该函数可以是**同步的**。外部用户需要依赖异步获取并缓存的 `getStoredChangelogFromMemory()`，但该函数设计为在 `setup.ts` 预加载后由同步调用方使用，因此如果缓存未准备好会返回空数组。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/logoV2Utils.ts:35-75` | `getLayoutMode` / `calculateLayoutDimensions` |
| `src/utils/logoV2Utils.ts:80-92` | `calculateOptimalLeftWidth` |
| `src/utils/logoV2Utils.ts:97-102` | `formatWelcomeMessage` |
| `src/utils/logoV2Utils.ts:108-180` | `truncatePath` 智能路径截断 |
| `src/utils/logoV2Utils.ts:189-219` | `getRecentActivity` 最近活动预加载 |
| `src/utils/logoV2Utils.ts:242-267` | `getLogoDisplayData` 显示数据聚合 |
| `src/utils/logoV2Utils.ts:272-305` | `formatModelAndBilling` 模型/账单格式化 |
| `src/utils/logoV2Utils.ts:312-350` | `getRecentReleaseNotesSync` 发布说明 |
| `src/ink/stringWidth.ts` | `stringWidth` CJK/emoji 感知宽度计算 |
| `src/utils/format.ts` | `truncate`, `truncateToWidth`, `truncateToWidthNoEllipsis` |
| `src/utils/sessionStorage.ts` | `loadMessageLogs` |
| `src/utils/auth.ts` | `getSubscriptionName`, `isClaudeAISubscriber` |
| `src/utils/settings/settings.ts` | `getInitialSettings` |
| `src/utils/releaseNotes.ts` | `getStoredChangelogFromMemory`, `parseChangelog` |
| `src/utils/semver.ts` | `gt` 版本比较 |

## 依赖与外部交互

### 内部依赖
- `../bootstrap/state.js`：`getSessionId`, `getDirectConnectServerUrl`
- `../ink/stringWidth.js`：`stringWidth`
- `../types/logs.js`：`LogOption`
- `./auth.js`：`getSubscriptionName`, `isClaudeAISubscriber`
- `./cwd.js`：`getCwd`
- `./file.js`：`getDisplayPath`
- `./format.js`：`truncate`, `truncateToWidth`, `truncateToWidthNoEllipsis`
- `./releaseNotes.js`：`getStoredChangelogFromMemory`, `parseChangelog`
- `./semver.js`：`gt`
- `./sessionStorage.js`：`loadMessageLogs`
- `./settings/settings.js`：`getInitialSettings`

### 调用方
- `src/components/LogoV2/LogoV2.tsx`
- `src/components/LogoV2/CondensedLogo.tsx`
- `src/setup.ts`

## 风险、边界与改进建议

### 风险与边界
1. **`truncatePath` 与 `format.ts` 中 `truncatePathMiddle` 的重复**：`logoV2Utils.ts` 中的 `truncatePath` 和 `format.ts` 导出的 `truncatePathMiddle` 功能高度重叠。维护两者增加了不一致的风险。实际上 `format.ts` 已经重新导出了 `truncatePathMiddle`（来自 `./truncate.js`），而 `logoV2Utils.ts` 似乎实现了自己的版本。
2. **`getRecentActivity` 的缓存不会过期**：`cachePromise` 和 `cachedActivity` 在进程生命周期内不会刷新。如果用户在会话期间删除了某些历史会话，Logo V2 仍会显示旧数据。
3. **`getRecentReleaseNotesSync` 对外部用户的空结果**：如果 `setup.ts` 的 `checkForReleaseNotes()` 尚未完成（或失败），外部用户调用 `getRecentReleaseNotesSync` 会返回空数组。这是设计上的同步限制，但可能导致启动界面有时不显示发布说明。
4. **`MAX_LEFT_WIDTH = 50` 的硬编码**：在超宽终端（如 300+ 列）中，左面板仍被限制在 50 列，可能导致右侧大量空白未被充分利用。
5. **`loadMessageLogs(10)` 的 I/O 成本**：虽然只加载 10 条，但 `loadMessageLogs` 内部可能需要扫描和排序所有消息日志。如果消息日志目录很大，这个调用可能比预期更慢。

### 改进建议
1. **统一路径截断实现**：将 `logoV2Utils.ts` 中的 `truncatePath` 逻辑合并到 `src/utils/truncate.ts` 或 `format.ts` 中，消除重复代码。
2. **增加缓存刷新机制**：在 `getRecentActivity` 中暴露一个 `refreshRecentActivity()` 函数，或在会话状态变化时自动清除 `cachePromise`。
3. **异步发布说明加载**：为外部用户提供一个 `getRecentReleaseNotes()` 异步版本，在缓存未命中时主动触发 `fetchAndStoreChangelog()`，避免同步版本返回空数组。
4. **响应式左面板宽度**：考虑根据终端总宽度动态调整 `MAX_LEFT_WIDTH`（如 `Math.min(50, columns * 0.3)`），在宽终端中展示更多左侧内容。
5. **延迟加载最近活动**：`LogoV2` 组件可能不需要在每次渲染时都等待 `getRecentActivity()`。可以考虑先渲染基础布局，待活动数据加载完成后再更新右侧内容（如果 Ink 支持这种异步更新模式）。
