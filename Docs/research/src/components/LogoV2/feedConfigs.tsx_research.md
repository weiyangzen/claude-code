# feedConfigs.tsx 深度研究文档

> 文件路径：`src/components/LogoV2/feedConfigs.tsx`  
> 文件大小：12,205 bytes（编译后产物，含 source map）  
> 研究范围：源码、Feed 类型系统、调用方（LogoV2/FeedColumn）、数据依赖（日志、Release Notes、Onboarding、Referral）

---

## 一、场景与职责

`feedConfigs.tsx` 是 Claude Code 欢迎页**右侧信息栏（Feed Column）的内容工厂**。它不直接渲染 UI，而是提供一组纯函数，把各种业务数据（最近会话、更新日志、 onboarding 步骤、Guest Passes 推广）转换成 `FeedConfig` 对象。这些 `FeedConfig` 再被 `Feed.tsx` / `FeedColumn.tsx` 消费并渲染。

核心职责：
1. **数据到配置的映射**：将原始业务数据格式化为统一的 `FeedConfig` 结构。
2. **空状态处理**：为每种 feed 提供 `emptyMessage` 和 `footer` 提示。
3. **自定义内容支持**：Guest Passes feed 使用 `customContent` 渲染非列表式 UI（图案 + 副标题）。

---

## 二、功能点目的

| 导出函数 | 输入 | 输出 | 目的 |
|----------|------|------|------|
| `createRecentActivityFeed` | `LogOption[]` | `FeedConfig` | 展示最近 3-5 条会话摘要，带相对时间戳，引导用户 `/resume`。 |
| `createWhatsNewFeed` | `string[]` (release notes) | `FeedConfig` | 展示最新更新日志，外部用户看 public changelog，ant 用户看内部 commit 列表。 |
| `createProjectOnboardingFeed` | `Step[]` | `FeedConfig` | 把 onboarding 步骤转成 checklist，并在用户位于 home 目录时追加警告。 |
| `createGuestPassesFeed` | 无（读全局缓存） | `FeedConfig` | 展示 Guest Passes 推广卡片，图案为 3 个 `[✻]`，带奖励金额副标题。 |

---

## 三、具体技术实现

### 3.1 `createRecentActivityFeed`

```tsx
export function createRecentActivityFeed(activities: LogOption[]): FeedConfig {
  const lines: FeedLine[] = activities.map(log => {
    const time = formatRelativeTimeAgo(log.modified)
    const description = log.summary && log.summary !== 'No prompt' ? log.summary : log.firstPrompt
    return { text: description || '', timestamp: time }
  })
  return {
    title: 'Recent activity',
    lines,
    footer: lines.length > 0 ? '/resume for more' : undefined,
    emptyMessage: 'No recent activity',
  }
}
```

- **时间格式化**：`formatRelativeTimeAgo(log.modified)` 来自 `src/utils/format.ts`，输出如 `"2h ago"`、`"3d ago"`。
- **摘要回退**：优先使用 AI 生成的 `summary`，若为空或 `"No prompt"`，则回退到 `firstPrompt`。
- **Footer 引导**：有数据时显示 `/resume for more`，空状态时只显示 `emptyMessage`。

### 3.2 `createWhatsNewFeed`

```tsx
export function createWhatsNewFeed(releaseNotes: string[]): FeedConfig {
  const lines: FeedLine[] = releaseNotes.map(note => {
    if ("external" === 'ant') {   // ← 编译期死代码
      const match = note.match(/^(\d+\s+\w+\s+ago)\s+(.+)$/)
      if (match) return { timestamp: match[1], text: match[2] || '' }
    }
    return { text: note }
  })
  const emptyMessage = "external" === 'ant'
    ? 'Unable to fetch latest claude-cli-internal commits'
    : 'Check the Claude Code changelog for updates'
  return {
    title: "external" === 'ant' ? "What's new [ANT-ONLY: Latest CC commits]" : "What's new",
    lines,
    footer: lines.length > 0 ? '/release-notes for more' : undefined,
    emptyMessage,
  }
}
```

- **编译期分支**：`"external" === 'ant'` 永远为 `false`，这是项目 DCE（死代码消除）模式的一部分。ant 构建流程会在编译前把 `'ant'` 字面量替换掉，使该分支生效。
- **时间戳解析**：ant 分支期望 release notes 格式为 `"3 days ago 修复了某个 bug"`，用正则提取时间戳和正文。
- **数据来源**：`releaseNotes` 由 `LogoV2.tsx` 通过 `getRecentReleaseNotesSync(3)` 获取，底层来自 `src/utils/releaseNotes.ts` 的 GitHub changelog 缓存或 `MACRO.VERSION_CHANGELOG`（ant 构建）。

### 3.3 `createProjectOnboardingFeed`

```tsx
export function createProjectOnboardingFeed(steps: Step[]): FeedConfig {
  const enabledSteps = steps
    .filter(({ isEnabled }) => isEnabled)
    .sort((a, b) => Number(a.isComplete) - Number(b.isComplete))
  const lines: FeedLine[] = enabledSteps.map(({ text, isComplete }) => {
    const checkmark = isComplete ? `${figures.tick} ` : ''
    return { text: `${checkmark}${text}` }
  })
  const warningText = getCwd() === homedir()
    ? 'Note: You have launched claude in your home directory. For the best experience, launch it in a project directory instead.'
    : undefined
  if (warningText) lines.push({ text: warningText })
  return { title: 'Tips for getting started', lines }
}
```

- **步骤过滤与排序**：只展示 `isEnabled === true` 的步骤，未完成的排在前面（`isComplete` 升序）。
- **Checkmark**：使用 `figures.tick`（✔ 或 ✓，取决于平台）标记已完成步骤。
- **Home 目录警告**：如果当前工作目录等于用户 home 目录，追加一条警告提示，引导用户在项目目录启动 Claude。

### 3.4 `createGuestPassesFeed`

```tsx
export function createGuestPassesFeed(): FeedConfig {
  const reward = getCachedReferrerReward()
  const subtitle = reward
    ? `Share Claude Code and earn ${formatCreditAmount(reward)} of extra usage`
    : 'Share Claude Code with friends'
  return {
    title: '3 guest passes',
    lines: [],
    customContent: {
      content: <>
        <Box marginY={1}><Text color="claude">[✻] [✻] [✻]</Text></Box>
        <Text dimColor>{subtitle}</Text>
      </>,
      width: 48,
    },
    footer: '/passes',
  }
}
```

- **奖励读取**：`getCachedReferrerReward()` 从 `GlobalConfig.passesEligibilityCache[orgId].referrer_reward` 读取，若缓存不存在则返回 `null`。
- **金额格式化**：`formatCreditAmount(reward)` 将 `amount_minor_units / 100` 格式化为带货币符号的字符串（如 `$20`、`£15`）。
- **CustomContent**：由于 Guest Passes 不是列表，而是图案卡片，因此使用 `FeedConfig.customContent` 绕过 `Feed.tsx` 的默认列表渲染逻辑。`width: 48` 用于 `calculateFeedWidth` 计算列宽。

---

## 四、关键代码路径与文件引用

### 4.1 本文件内部路径

| 行号区间 | 内容 |
|----------|------|
| `1-10` | 导入：figures、os、React、Ink、类型、工具函数 |
| `11-26` | `createRecentActivityFeed` |
| `27-49` | `createWhatsNewFeed` |
| `50-73` | `createProjectOnboardingFeed` |
| `74-91` | `createGuestPassesFeed` |

### 4.2 上游调用方

- **`src/components/LogoV2/LogoV2.tsx:12, 421`**  
  根据当前状态（`showOnboarding`、`showGuestPassesUpsell`、`showOverageCreditUpsell`）选择调用哪个工厂函数，组合成 `FeedColumn` 的 `feeds` 数组：
  ```tsx
  showOnboarding
    ? [createProjectOnboardingFeed(getSteps()), createRecentActivityFeed(activities)]
    : showGuestPassesUpsell
      ? [createRecentActivityFeed(activities), createGuestPassesFeed()]
      : showOverageCreditUpsell
        ? [createRecentActivityFeed(activities), createOverageCreditFeed()]
        : [createRecentActivityFeed(activities), createWhatsNewFeed(changelog)]
  ```

- **`src/components/LogoV2/FeedColumn.tsx`**  
  接收 `FeedConfig[]`，计算最大宽度，渲染 `Feed` 组件并用 `Divider` 分隔。

### 4.3 下游依赖

- **`src/components/LogoV2/Feed.tsx`**  
  定义 `FeedConfig`、`FeedLine` 类型，并提供 `calculateFeedWidth` 和 `Feed` 渲染组件。
- **`src/projectOnboardingState.ts`**  
  提供 `Step` 类型和 `getSteps()`。
- **`src/services/api/referral.ts`**  
  提供 `formatCreditAmount`、`getCachedReferrerReward`。
- **`src/types/logs.ts`**  
  定义 `LogOption`。
- **`src/utils/cwd.ts`**  
  提供 `getCwd()`（支持 AsyncLocalStorage 覆盖）。
- **`src/utils/format.ts`**  
  提供 `formatRelativeTimeAgo`。
- **`src/utils/releaseNotes.ts`**  
  间接依赖：为 `createWhatsNewFeed` 提供 `releaseNotes` 数据源。

---

## 五、依赖与外部交互

### 5.1 运行时依赖

| 模块 | 用途 |
|------|------|
| `figures` | 跨平台 tick 符号（✔ / ✓） |
| `os` (`homedir`) | Home 目录检测 |
| `react` | `customContent` 中的 JSX |
| `src/ink.js` (`Box`, `Text`) | Guest Passes 卡片渲染 |
| `src/projectOnboardingState.js` | Onboarding 步骤类型与数据 |
| `src/services/api/referral.js` | 推荐奖励缓存读取与格式化 |
| `src/types/logs.js` | `LogOption` 类型 |
| `src/utils/cwd.js` | 当前工作目录 |
| `src/utils/format.js` | 相对时间格式化 |

### 5.2 数据流图

```
LogoV2.tsx
    │
    ├─► createRecentActivityFeed(activities)
    │       └─► formatRelativeTimeAgo ──► src/utils/format.ts
    │
    ├─► createWhatsNewFeed(changelog)
    │       └─► releaseNotes from getRecentReleaseNotesSync(3)
    │
    ├─► createProjectOnboardingFeed(getSteps())
    │       ├─► figures.tick
    │       ├─► getCwd() vs homedir()
    │       └─► Step[] from src/projectOnboardingState.ts
    │
    └─► createGuestPassesFeed()
            ├─► getCachedReferrerReward()
            ├─► formatCreditAmount()
            └─► customContent JSX ──► Feed.tsx 特殊渲染
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险

1. **`"external" === 'ant'` 死代码模式的理解成本**  
   这是项目统一的 DCE 写法，但对新开发者来说非常反直觉。若误删或误改，可能导致 ant 构建产物泄漏字符串或外部构建包含内部提示。

2. **`getCachedReferrerReward()` 可能返回 `null` 的静默失败**  
   如果 eligibility cache 为空（冷启动），`createGuestPassesFeed` 会优雅降级到 `"Share Claude Code with friends"`，但用户看不到奖励金额，可能降低转化率。

3. **Home 目录警告的误报**  
   `getCwd() === homedir()` 使用严格字符串相等。若用户通过符号链接进入 home 目录，或 `homedir()` 与 `getCwd()` 的解析路径不一致（如 trailing slash），警告不会触发。

4. **`createWhatsNewFeed` 的正则脆弱性**  
   ant 分支依赖 `note.match(/^(\d+\s+\w+\s+ago)\s+(.+)$/)`。如果 commit message 格式变化（如 `"1 week ago"` 变成 `"7 days ago"` 以外的表达），解析会失败，整条 note 被当成纯文本展示，时间戳和正文不再分离。

5. **无异常边界**  
   所有工厂函数都是纯函数，没有 `try/catch`。如果输入的 `activities` 包含非法日期，`formatRelativeTimeAgo` 可能抛出（虽然该函数内部对 `Date` 对象做了安全处理）。

### 6.2 边界情况

- **空 `activities`**：`lines` 为空数组，`footer` 消失，展示 `emptyMessage: 'No recent activity'`。
- **空 `releaseNotes`**：同理，展示外部/ ant 对应的 `emptyMessage`。
- **Onboarding 步骤全部 `isEnabled: false`**：`enabledSteps` 为空，`lines` 为空，只展示标题 `Tips for getting started`（没有 `emptyMessage`）。
- **CWD 被 AsyncLocalStorage 覆盖**：`getCwd()` 会读取覆盖值，因此并发 agent 场景下每个 agent 看到的 home 目录警告是独立的。

### 6.3 改进建议

| 优先级 | 建议 | 理由 |
|--------|------|------|
| 中 | 把 `"external" === 'ant'` 替换为更明确的编译期宏或 `feature()` 调用 | 降低认知负担，减少 DCE 误操作风险。 |
| 中 | 在 `createGuestPassesFeed` 增加缓存缺失时的主动触发逻辑 | 当前冷启动看不到奖励金额，可在返回前尝试 `fetchAndStorePassesEligibility()`（非阻塞）。 |
| 低 | Home 目录检测使用 `path.resolve` 规范化后再比较 | 消除符号链接、trailing slash 导致的 false negative。 |
| 低 | 为 `createWhatsNewFeed` 的 ant 分支增加更宽松的正则或 fallback | 兼容 `"X weeks ago"`、 `"Yesterday"` 等非标准格式。 |
| 低 | 抽离 `width: 48` 为常量或根据内容动态计算 | `customContent.width` 目前硬编码，若副标题翻译变长可能导致布局错位。 |

---

*文档生成时间：2026-04-01*  
*执行器：kimi (k2p5)*
