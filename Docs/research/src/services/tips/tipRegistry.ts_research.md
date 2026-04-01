# tipRegistry.ts 研究文档

## 场景与职责

`tipRegistry.ts` 是 Claude Code 提示系统的核心注册表模块，负责：

1. **定义所有内置提示**：包含 40+ 个面向不同用户场景的功能提示
2. **管理提示相关性判断**：每个提示都有 `isRelevant` 函数，根据用户状态和环境决定是否展示
3. **支持自定义提示**：通过设置覆盖机制允许用户/组织自定义提示内容
4. **筛选可展示提示**：综合相关性、冷却时间和用户设置，返回当前可展示的提示列表

该模块是提示系统的"内容库"和"筛选引擎"，决定了用户能看到什么样的使用建议。

## 功能点目的

### 1. 内置提示定义（`externalTips` 数组）

包含 40+ 个提示，覆盖以下类别：

| 类别 | 提示示例 | 目标用户 |
|------|----------|----------|
| 新用户引导 | `new-user-warmup` | 启动次数 < 10 |
| 功能发现 | `plan-mode-for-complex-tasks`, `memory-command` | 所有用户 |
| IDE/终端集成 | `terminal-setup`, `vscode-command-install` | 特定终端用户 |
| 高级功能 | `git-worktrees`, `custom-commands`, `custom-agents` | 资深用户 |
| 产品推广 | `desktop-app`, `web-app`, `mobile-app` | 非 Linux 用户 |
| 插件推荐 | `frontend-design-plugin`, `vercel-plugin` | 上下文相关 |
| 实验功能 | `effort-high-nudge`, `subagent-fanout-nudge` | 1P API 客户 |
| 内部功能 | `important-claudemd`, `skillify` | Ant 内部用户 |

### 2. 提示数据结构（`Tip` 类型）

```typescript
interface Tip {
  id: string;                    // 唯一标识符
  content: async (ctx) => string; // 异步内容生成函数
  cooldownSessions: number;      // 冷却会话数
  isRelevant: async (ctx) => boolean; // 相关性判断函数
}
```

### 3. 插件相关性检测（`isMarketplacePluginRelevant`）

根据用户当前工作上下文推荐插件：
- 检查官方市场是否已安装
- 检查插件是否已安装
- 根据文件路径模式匹配（如 `\.(html|css|htm)$`）
- 根据 CLI 命令使用记录匹配（如 `vercel` 命令）

### 4. 自定义提示支持（`getCustomTips`）

允许通过设置覆盖默认提示：
```typescript
// 设置格式
{
  "spinnerTipsOverride": {
    "tips": ["自定义提示 1", "自定义提示 2"],
    "excludeDefault": false  // true 时完全禁用内置提示
  }
}
```

### 5. 提示筛选（`getRelevantTips`）

综合以下因素筛选提示：
1. 相关性检查（`isRelevant` 返回 true）
2. 冷却时间检查（`getSessionsSinceLastShown >= cooldownSessions`）
3. 自定义提示覆盖（`excludeDefault` 选项）

## 具体技术实现

### 关键流程

```
REPL.tsx 请求提示
    ↓
getTipToShowOnSpinner(context)
    ↓
getRelevantTips(context)
    ↓
1. 获取自定义提示（如果有）
2. 如果 excludeDefault=true，直接返回自定义提示
3. 否则，遍历所有内置提示：
   - 调用 isRelevant(context) 检查相关性
   - 调用 getSessionsSinceLastShown 检查冷却
4. 合并内置提示和自定义提示
    ↓
selectTipWithLongestTimeSinceShown 选择最久未展示的
    ↓
展示提示并调用 recordShownTip 记录历史
```

### 数据结构

#### 提示上下文（`TipContext`）

```typescript
interface TipContext {
  theme: Theme;                    // 当前主题
  readFileState?: FileStateCache;  // 已读取文件缓存
  bashTools?: Set<string>;         // 使用的 bash 工具
}
```

#### 内置提示示例

```typescript
{
  id: 'plan-mode-for-complex-tasks',
  content: async () =>
    `Use Plan Mode to prepare for a complex request before making changes. Press ${getShortcutDisplay('chat:cycleMode', 'Chat', 'shift+tab')} twice to enable.`,
  cooldownSessions: 5,
  isRelevant: async () => {
    if (process.env.USER_TYPE === 'ant') return false
    const config = getGlobalConfig()
    const daysSinceLastUse = config.lastPlanModeUse
      ? (Date.now() - config.lastPlanModeUse) / (1000 * 60 * 60 * 24)
      : Infinity
    return daysSinceLastUse > 7
  },
}
```

### 相关性判断逻辑

| 提示 ID | 相关性条件 |
|---------|-----------|
| `new-user-warmup` | `numStartups < 10` |
| `plan-mode-for-complex-tasks` | 7 天内未使用计划模式，且非 Ant 用户 |
| `git-worktrees` | `numStartups > 50` 且 worktree 数量 ≤ 1 |
| `color-when-multi-clauding` | 并发会话数 ≥ 2 且未设置颜色 |
| `terminal-setup` | 根据终端类型检查键位绑定状态 |
| `frontend-design-plugin` | 编辑 HTML/CSS 文件且未安装插件 |
| `effort-high-nudge` | 1P API 客户，支持 effort 的模型，未设置 effort |
| `guest-passes` | 有资格且未访问过 `/passes` |

### 缓存机制

```typescript
// 官方市场安装状态缓存
let _isOfficialMarketplaceInstalledCache: boolean | undefined
```

## 关键代码路径与文件引用

### 导出函数

| 函数 | 导出类型 | 被引用文件 |
|------|----------|------------|
| `getRelevantTips` | 命名导出 | `tipScheduler.ts` |

### 依赖关系

```
tipRegistry.ts
├── 导入: tipHistory.ts (getSessionsSinceLastShown)
├── 导入: config.ts (getGlobalConfig, saveGlobalConfig)
├── 导入: settings.ts (getSettings_DEPRECATED, getInitialSettings)
├── 导入: 各种工具函数（IDE检测、平台检测、模型检测等）
├── 被 tipScheduler.ts 导入
└── 被 REPL.tsx 间接使用
```

### 导入的依赖模块

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `./tipHistory.js` | `getSessionsSinceLastShown` | 检查提示冷却状态 |
| `../../utils/config.js` | `getGlobalConfig` | 读取用户配置 |
| `../../utils/settings/settings.js` | `getSettings_DEPRECATED`, `getInitialSettings` | 读取用户设置 |
| `../../utils/ide.js` | IDE 检测函数 | 判断 IDE 集成相关提示 |
| `../../utils/platform.js` | `getPlatform` | 平台特定提示 |
| `../../utils/auth.js` | `is1PApiCustomer` | 1P API 客户专属提示 |
| `../analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` | 实验功能提示 |

## 依赖与外部交互

### 与配置系统的交互

- **读取配置**：`getGlobalConfig()` 获取 `numStartups`、`lastPlanModeUse`、`tipsHistory` 等
- **读取设置**：`getSettings_DEPRECATED()` 获取 `spinnerTipsEnabled`、`statusLine` 等

### 与实验系统的交互

使用 GrowthBook 功能标志控制实验性提示：
```typescript
const variant = getFeatureValue_CACHED_MAY_BE_STALE<'off' | 'copy_a' | 'copy_b'>(
  'tengu_tide_elm',  // effort-high-nudge 实验
  'off'
)
```

### 与插件系统的交互

检测插件安装状态和市场可用性：
```typescript
const config = await loadKnownMarketplacesConfigSafe()
_isOfficialMarketplaceInstalledCache = OFFICIAL_MARKETPLACE_NAME in config
```

### 与 IDE 检测的交互

检测运行中的 IDE 和安装状态：
```typescript
const runningIDEs = await detectRunningIDEsCached()
const lockfiles = await getSortedIdeLockfiles()
```

## 风险、边界与改进建议

### 潜在风险

1. **性能问题**：`getRelevantTips` 会并行执行所有提示的 `isRelevant` 函数，如果有大量提示或复杂的判断逻辑，可能影响启动性能
   
2. **外部依赖过多**：导入了 20+ 个模块，任何一个模块的故障都可能影响提示系统

3. **硬编码提示内容**：提示文本直接写在代码中，国际化和动态更新困难

4. **实验功能耦合**：实验提示与 GrowthBook 标志紧密耦合，标志名称散落在代码各处

### 边界情况

| 场景 | 行为 |
|------|------|
| `spinnerTipsEnabled = false` | `getTipToShowOnSpinner` 返回 `undefined`（在 tipScheduler.ts 中处理）|
| `excludeDefault = true` 且自定义提示为空 | 返回空数组，不展示任何提示 |
| `isRelevant` 抛出异常 | 该提示被排除（通过 `try-catch` 在部分提示中处理）|
| 所有提示都在冷却中 | 返回空数组，等待冷却结束 |
| 新用户（`numStartups = 1`） | 只展示新用户相关提示 |

### 改进建议

1. **提示内容外部化**：将提示文本移到配置文件或远程配置，支持：
   - 动态更新提示内容
   - 多语言国际化
   - A/B 测试不同文案

2. **性能优化**：
   ```typescript
   // 建议：按需加载提示，而非一次性检查所有
   const priorityTips = getPriorityTipsForUserContext(context)
   const relevantTips = await filterRelevant(priorityTips)
   ```

3. **更细粒度的冷却控制**：
   ```typescript
   // 当前：基于会话数
   cooldownSessions: 5
   
   // 建议：支持多种冷却策略
   cooldown: {
     type: 'sessions' | 'time' | 'once',
     value: number  // 会话数或毫秒
   }
   ```

4. **提示分类和优先级**：
   ```typescript
   interface Tip {
     // ... 现有字段
     category: 'onboarding' | 'feature' | 'promotion'
     priority: number
     maxShows?: number  // 最多展示次数
   }
   ```

5. **更好的错误处理**：当前部分提示有 `try-catch`，建议统一封装：
   ```typescript
   const safeIsRelevant = async (tip: Tip, context: TipContext) => {
     try {
       return await tip.isRelevant(context)
     } catch (error) {
       logForDebugging(`Tip ${tip.id} relevance check failed: ${error}`)
       return false
     }
   }
   ```

6. **提示效果追踪**：当前只记录展示次数，建议追踪：
   - 用户是否按照提示执行了操作
   - 提示的转化率
   - 用户对提示的反馈

### 代码组织建议

当前所有提示定义在一个文件中（约 600 行），建议按类别拆分：

```
src/services/tips/
├── index.ts           # 导出 getRelevantTips
├── types.ts           # Tip, TipContext 类型
├── registry.ts        # 注册表逻辑
├── categories/
│   ├── onboarding.ts  # 新用户提示
│   ├── features.ts    # 功能发现提示
│   ├── ide.ts         # IDE 集成提示
│   └── promotions.ts  # 产品推广提示
```
