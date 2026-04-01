# Stats.tsx 组件深度研究文档

> **研究对象**: `src/components/Stats.tsx`  
> **研究范围**: 组件本身及其直接依赖、调用方、数据流、配置  
> **执行器**: kimi (k2p5)  
> **日期**: 2026-04-01

---

## 1. 场景与职责

### 1.1 组件定位

`Stats.tsx` 是 Claude Code CLI 的 **用户统计数据展示组件**，通过 `/stats` 命令触发。它提供一个交互式终端界面，展示用户的 Claude Code 使用统计信息，包括：

- 会话活跃度热力图（GitHub 风格）
- 模型使用统计（输入/输出 token、缓存使用）
- 使用 streak（连续使用天数）
- 趣味对比数据（与经典文学作品 token 量对比）

### 1.2 使用场景

| 场景 | 描述 |
|------|------|
| 用户主动查询 | 用户在 REPL 输入 `/stats` 命令 |
| 数据可视化 | 展示历史使用趋势和模式 |
| 截图分享 | 支持 `Ctrl+S` 将统计截图复制到剪贴板 |
| 多时间范围 | 支持 7天/30天/全部时间 三种视图 |

### 1.3 调用入口

```
用户输入 /stats
    ↓
src/commands/stats/stats.tsx (命令入口)
    ↓
src/components/Stats.tsx (本组件)
```

---

## 2. 功能点目的

### 2.1 核心功能模块

| 模块 | 功能描述 | 用户价值 |
|------|----------|----------|
| **Overview Tab** | 展示总体统计：会话数、消息数、活跃天数、最长会话、streak 等 | 快速了解整体使用情况 |
| **Models Tab** | 展示各模型的 token 使用分布、每日趋势图 | 了解模型偏好和消耗 |
| **Activity Heatmap** | GitHub 风格的年度活跃度热力图 | 可视化使用习惯 |
| **Date Range Selector** | 切换 7d/30d/all 时间范围 | 灵活查看不同时间段 |
| **Screenshot** | Ctrl+S 复制统计截图到剪贴板 | 便于分享 |
| **Fun Factoids** | 与书籍/时间类比的趣味数据 | 增加趣味性 |

### 2.2 特色功能详解

#### 2.2.1 趣味数据对比 (Fun Factoids)

组件内置了两组对比数据：

1. **书籍 Token 对比** (`BOOK_COMPARISONS`): 将用户总 token 消耗与经典文学作品对比（如《小王子》22k tokens、《战争与和平》730k tokens）
2. **时间对比** (`TIME_COMPARISONS`): 将最长会话时长与日常活动对比（如 TED 演讲 18 分钟、电影《盗梦空间》148 分钟）

#### 2.2.2 Shot Stats (Ant 内部功能)

通过 `feature('SHOT_STATS')` 特性开关控制，统计 PR 创建时的 "shot" 次数分布（1-shot、2-5 shot、6-10 shot、11+ shot），用于内部质量分析。

#### 2.2.3 Speculation Time Saved

Ant 内部功能，展示通过 speculation 机制节省的时间。

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 核心类型定义 (来自 `src/utils/stats.ts`)

```typescript
// 每日活动数据
interface DailyActivity {
  date: string;           // YYYY-MM-DD
  messageCount: number;
  sessionCount: number;
  toolCallCount: number;
}

// 每日模型 token 使用
interface DailyModelTokens {
  date: string;
  tokensByModel: { [modelName: string]: number };
}

// Streak 信息
interface StreakInfo {
  currentStreak: number;
  longestStreak: number;
  currentStreakStart: string | null;
  longestStreakStart: string | null;
  longestStreakEnd: string | null;
}

// 会话统计
interface SessionStats {
  sessionId: string;
  duration: number;       // 毫秒
  messageCount: number;
  timestamp: string;
}

// 完整统计数据结构
interface ClaudeCodeStats {
  totalSessions: number;
  totalMessages: number;
  totalDays: number;
  activeDays: number;
  streaks: StreakInfo;
  dailyActivity: DailyActivity[];
  dailyModelTokens: DailyModelTokens[];
  longestSession: SessionStats | null;
  modelUsage: { [modelName: string]: ModelUsage };
  firstSessionDate: string | null;
  lastSessionDate: string | null;
  peakActivityDay: string | null;
  peakActivityHour: number | null;
  totalSpeculationTimeSavedMs: number;
  shotDistribution?: { [shotCount: number]: number };
  oneShotRate?: number;
}
```

#### 3.1.2 组件内部状态

```typescript
// 日期范围类型
type StatsDateRange = '7d' | '30d' | 'all';

// 组件 Props
interface Props {
  onClose: (result?: string, options?: { display?: CommandResultDisplay }) => void;
}

// 加载结果类型
interface StatsResult {
  type: 'success' | 'error' | 'empty';
  data?: ClaudeCodeStats;
  message?: string;
}
```

### 3.2 关键流程

#### 3.2.1 数据加载流程

```
Stats 组件挂载
    ↓
createAllTimeStatsPromise() 创建 Promise
    ↓
React Suspense 等待数据
    ↓
aggregateClaudeCodeStatsForRange('all') 调用
    ↓
processSessionFiles() 处理会话文件
    ↓
返回 StatsResult
```

**数据加载策略**：
- 使用 React 19 的 `use()` hook 读取 Promise
- 全量数据（all-time）通过 Suspense 阻塞加载
- 7d/30d 范围切换时，非阻塞加载，显示 spinner

#### 3.2.2 日期范围切换流程

```typescript
// 使用 useEffect 监听 dateRange 变化
useEffect(() => {
  if (dateRange === "all") return;  // all 已预加载
  if (statsCache[dateRange]) return; // 已有缓存
  
  setIsLoadingFiltered(true);
  aggregateClaudeCodeStatsForRange(dateRange)
    .then(data => {
      setStatsCache(prev => ({ ...prev, [dateRange]: data }));
      setIsLoadingFiltered(false);
    });
}, [dateRange, statsCache]);
```

#### 3.2.3 截图功能流程

```
Ctrl+S 触发
    ↓
handleScreenshot(stats, activeTab, setCopyStatus)
    ↓
renderStatsToAnsi(stats, activeTab) 生成 ANSI 文本
    ↓
copyAnsiToClipboard(ansiText)
    ↓
ansiToPng() 转换为 PNG
    ↓
平台特定命令复制到剪贴板
    (macOS: osascript, Linux: xclip/xsel, Windows: PowerShell)
```

### 3.3 图表生成技术

#### 3.3.1 热力图 (`generateHeatmap`)

- **库**: 自定义实现 (`src/utils/heatmap.ts`)
- **算法**: 
  - 基于百分位数计算活跃度强度 (p25, p50, p75)
  - 使用 Unicode 块字符表示强度: `·` `░` `▒` `▓` `█`
  - 颜色: Claude 橙色 (`#da7756`)
  - 布局: 52 周 × 7 天，带月份标签

#### 3.3.2 Token 趋势图 (`generateTokenChart`)

- **库**: `asciichart` (npm 包)
- **特性**:
  - 最多显示 top 3 模型
  - 使用主题色 (suggestion, success, warning)
  - 自适应终端宽度
  - 生成 X 轴日期标签

### 3.4 键盘交互

| 按键 | 功能 |
|------|------|
| `Esc` / `Ctrl+C` / `Ctrl+D` | 关闭统计面板 |
| `Tab` | 切换 Overview/Models Tab |
| `r` | 循环切换日期范围 (all → 7d → 30d) |
| `Ctrl+S` | 复制截图到剪贴板 |
| `↑/↓` | Models Tab 中滚动模型列表 |

---

## 4. 关键代码路径与文件引用

### 4.1 文件依赖图

```
Stats.tsx
├── 直接依赖
│   ├── react/compiler-runtime (_c 编译器缓存)
│   ├── bun:bundle (feature 标志)
│   ├── asciichart (图表库)
│   ├── chalk (ANSI 颜色)
│   ├── figures (符号)
│   ├── strip-ansi (ANSI 剥离)
│   └── react (核心)
│
├── 项目内部依赖
│   ├── ../commands.js (CommandResultDisplay 类型)
│   ├── ../hooks/useTerminalSize.js (终端尺寸)
│   ├── ../ink/colorize.js (颜色应用)
│   ├── ../ink/stringWidth.js (字符串宽度计算)
│   ├── ../ink/styles.js (Color 类型)
│   ├── ../ink.js (Ansi, Box, Text, useInput)
│   ├── ../keybindings/useKeybinding.js (键盘绑定)
│   ├── ../utils/config.js (getGlobalConfig)
│   ├── ../utils/format.js (formatDuration, formatNumber)
│   ├── ../utils/heatmap.js (generateHeatmap)
│   ├── ../utils/model/model.js (renderModelName)
│   ├── ../utils/screenshotClipboard.js (copyAnsiToClipboard)
│   ├── ../utils/stats.js (核心统计逻辑)
│   ├── ../utils/systemTheme.js (resolveThemeSetting)
│   ├── ../utils/theme.js (getTheme, themeColorToAnsi)
│   ├── ./design-system/Pane.js (容器组件)
│   ├── ./design-system/Tabs.js (Tab 组件)
│   └── ./Spinner.js (加载动画)
│
└── 被调用
    └── ../commands/stats/stats.tsx (命令入口)
```

### 4.2 核心依赖文件详解

#### 4.2.1 `src/utils/stats.ts` - 统计逻辑核心

| 函数 | 职责 |
|------|------|
| `aggregateClaudeCodeStats()` | 聚合所有会话统计（带缓存） |
| `aggregateClaudeCodeStatsForRange()` | 按日期范围聚合 |
| `processSessionFiles()` | 处理会话 JSONL 文件 |
| `getAllSessionFiles()` | 获取所有项目会话文件 |
| `calculateStreaks()` | 计算连续使用天数 |
| `cacheToStats()` | 缓存数据转换为展示数据 |
| `readSessionStartDate()` | 高效读取会话起始日期（4KB peek） |
| `extractShotCountFromMessages()` | 提取 PR shot 次数 |

#### 4.2.2 `src/utils/statsCache.ts` - 缓存管理

- **缓存文件**: `~/.claude/stats-cache.json`
- **版本**: `STATS_CACHE_VERSION = 3`
- **策略**: 
  - 历史数据（昨天及之前）持久化缓存
  - 今日数据实时计算（会话进行中）
  - 文件锁防止并发写入 (`withStatsCacheLock`)

#### 4.2.3 `src/utils/heatmap.ts` - 热力图生成

```typescript
// 核心算法
function calculatePercentiles(dailyActivity: DailyActivity[]): Percentiles {
  const counts = dailyActivity
    .map(a => a.messageCount)
    .filter(c => c > 0)
    .sort((a, b) => a - b);
  
  return {
    p25: counts[Math.floor(counts.length * 0.25)],
    p50: counts[Math.floor(counts.length * 0.5)],
    p75: counts[Math.floor(counts.length * 0.75)],
  };
}
```

#### 4.2.4 `src/utils/screenshotClipboard.ts` - 截图功能

- **流程**: ANSI → PNG → 剪贴板
- **渲染**: 纯 TS 实现，无 WASM，无系统字体依赖
- **跨平台**: macOS (osascript), Linux (xclip/xsel), Windows (PowerShell)

### 4.3 代码行数统计

| 文件 | 行数 | 说明 |
|------|------|------|
| `Stats.tsx` | ~1228 行 | 主组件 |
| `utils/stats.ts` | ~1061 行 | 统计逻辑 |
| `utils/statsCache.ts` | ~434 行 | 缓存管理 |
| `utils/heatmap.ts` | ~198 行 | 热力图 |
| `utils/screenshotClipboard.ts` | ~121 行 | 截图 |

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

| 依赖 | 用途 |
|------|------|
| `asciichart` | Token 趋势折线图 |
| `chalk` | ANSI 颜色处理 |
| `figures` | 终端符号（箭头、圆点等） |
| `strip-ansi` | 移除 ANSI 转义序列 |

### 5.2 内部服务依赖

| 服务 | 用途 |
|------|------|
| `feature('SHOT_STATS')` | Ant 内部 shot 统计开关 |
| `feature('external') === 'ant'` | Ant 内部功能标识 |
| `getGlobalConfig()` | 读取用户主题设置 |
| `useTerminalSize()` | 监听终端尺寸变化 |

### 5.3 数据存储

| 存储位置 | 内容 |
|----------|------|
| `~/.claude/stats-cache.json` | 统计缓存（历史数据） |
| `~/.claude/projects/*/\*.jsonl` | 会话记录文件 |
| `~/.claude/projects/*/{sessionId}/subagents/*.jsonl` | 子代理会话 |

### 5.4 会话文件格式

```typescript
// 会话文件: {projectDir}/{sessionId}.jsonl
// 子代理: {projectDir}/{sessionId}/subagents/agent-{agentId}.jsonl

interface Entry {
  type: 'user' | 'assistant' | 'system' | 'attachment' | 'progress' | 'speculation-accept' | ...;
  timestamp: string;
  isSidechain?: boolean;
  // ... 其他字段
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 性能风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 大文件处理 | 会话文件可能非常大（长时间会话） | 批处理（BATCH_SIZE=20）、日期过滤预检查 |
| 并发访问 | 多进程同时读写缓存 | 文件锁 (`withStatsCacheLock`) |
| 缓存失效 | 缓存版本升级时数据丢失 | 版本迁移逻辑 (`migrateStatsCache`) |

#### 6.1.2 数据准确性风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 时间戳异常 | 部分会话记录可能缺少时间戳 | 过滤无效日期会话 |
| 子代理统计 | 子代理消息标记为 sidechain，但 token 仍需统计 | 特殊处理子代理文件 |
|  resumed 会话 | 旧会话恢复后 mtime 更新但 startDate 不变 | 大文件时 peek 检查 startDate |

#### 6.1.3 兼容性风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 终端颜色 | Apple Terminal 对 24-bit 色支持不佳 | 自动降级到 256 色 |
| 截图剪贴板 | Linux 需要 xclip/xsel | 清晰的错误提示 |

### 6.2 边界情况

1. **空数据**: 新用户无历史会话时显示 "No stats available yet"
2. **单一会话**: 热力图和趋势图需要至少 2 天数据才显示
3. **大量模型**: Models Tab 最多显示 4 个模型，支持滚动
4. **窄终端**: 热力图最小 10 周，最大 52 周，自适应终端宽度

### 6.3 改进建议

#### 6.3.1 性能优化

```typescript
// 建议: 添加 Web Worker 支持处理大文件
// 当前: 主线程批处理 20 个文件
// 优化: 使用 Worker 池并行处理
```

#### 6.3.2 功能增强

| 建议 | 优先级 | 说明 |
|------|--------|------|
| 导出 JSON/CSV | 中 | 支持原始数据导出 |
| 自定义时间范围 | 低 | 除 7d/30d/all 外支持自定义 |
| 趋势预测 | 低 | 基于历史数据预测使用趋势 |
| 多设备同步 | 低 | 跨设备统计聚合 |

#### 6.3.3 代码质量

1. **类型安全**: 部分 `any` 类型可收紧（如 React Compiler 生成的 `$` 数组）
2. **测试覆盖**: 缺乏单元测试，特别是 `generateFunFactoid` 和图表生成逻辑
3. **国际化**: 当前硬编码英文，需 i18n 支持

#### 6.3.4 架构改进

```
当前: Stats.tsx 包含所有逻辑 (~1200 行)
建议: 
  - StatsContainer.tsx (数据获取)
  - StatsOverview.tsx (概览视图)
  - StatsModels.tsx (模型视图)
  - useStatsScreenshot.ts (截图逻辑)
```

### 6.4 监控指标建议

| 指标 | 用途 |
|------|------|
| `stats_load_duration_ms` | 数据加载耗时 |
| `stats_cache_hit_rate` | 缓存命中率 |
| `stats_screenshot_success` | 截图成功率 |
| `stats_tab_switch_count` | Tab 切换频率（功能使用度） |

---

## 7. 附录

### 7.1 主题颜色映射

```typescript
// 图表使用的主题色
const colors = [
  themeColorToAnsi(theme.suggestion),  // 蓝色系
  themeColorToAnsi(theme.success),     // 绿色系
  themeColorToAnsi(theme.warning),     // 黄色系
];
```

### 7.2 模型名称渲染

```typescript
// 内部模型名称 → 展示名称
claude-opus-4-6 → Opus 4.6
claude-opus-4-6[1m] → Opus 4.6 (1M context)
claude-sonnet-4-6 → Sonnet 4.6
// ... 详见 src/utils/model/model.ts
```

### 7.3 相关命令

| 命令 | 说明 |
|------|------|
| `/stats` | 打开统计面板 |
| `/usage` | 查看当前会话用量 |
| `/cost` | 查看成本统计 |

---

*文档结束*
