# src/utils/stats.ts 研究文档

## 场景与职责

`src/utils/stats.ts` 是 Claude Code 的**会话统计聚合引擎**，为 `/stats` 命令提供底层数据支撑。它负责扫描用户所有项目目录下的会话 JSONL 转录文件，提取并聚合出可供 UI 渲染的量化指标，包括活跃度热力图、模型 token 消耗、连续使用 streak、会话时长分布等。该模块采用**磁盘缓存 + 增量计算**的架构，避免每次打开 `/stats` 都全量重新扫描可能累积数 GB 的历史 transcript。

核心职责：
- 遍历 `~/.claude/projects/` 下所有项目的主会话文件及子代理（subagent） transcript。
- 按日期范围过滤、解析、聚合 transcript 条目，生成 `ClaudeCodeStats`。
- 与 `statsCache.ts` 协作，将“昨天及之前”的确定性历史数据写入缓存，“今天”的实时数据则每次重新计算。
- 提供 `readSessionStartDate` 等性能优化手段，对大文件做 4KB head peek 以跳过无需处理的文件。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `processSessionFiles` | 全量/按日期范围解析 JSONL，产出中间统计 `ProcessedStats` |
| `aggregateClaudeCodeStats` | 带缓存的“全部时间”聚合入口；历史走缓存，今天走实时计算 |
| `aggregateClaudeCodeStatsForRange` | 为 `7d` / `30d` / `all` 提供范围聚合；`all` 走缓存，其余直接扫描 |
| `cacheToStats` | 将磁盘缓存 `PersistedStatsCache` 与今日实时 `ProcessedStats` 合并，生成最终对外类型 `ClaudeCodeStats` |
| `readSessionStartDate` | 仅读取文件头 4KB，快速判断会话首条消息日期，用于大文件跳过优化 |
| `calculateStreaks` | 基于活跃日期计算当前连续天数与历史最长连续天数 |
| `extractShotCountFromMessages` |（ant 内部）从 `gh pr create` 的 bash 命令中提取 shot 次数，用于内部效率统计 |

## 具体技术实现

### 1. 文件发现：`getAllSessionFiles`
- 以 `getProjectsDir()`（即 `~/.claude/projects`）为根，递归读取每个项目目录。
- 主会话文件：项目根目录下直接以 `.jsonl` 结尾的文件。
- 子代理文件：结构为 `{projectDir}/{sessionId}/subagents/agent-{agentId}.jsonl`。
- 使用 `Promise.all` 并行读取所有项目目录，再 `flat()` 合并结果。

### 2. 批量处理与日期过滤：`processSessionFiles`
- **并行批次**：`BATCH_SIZE = 20`，每轮并发处理 20 个文件，控制 I/O 并发度。
- **mtime 预筛**：若指定 `fromDate`，先 `fs.stat` 获取修改时间；若 `mtime < fromDate` 则跳过。
- **4KB 预读优化**：对大于 64KB 的文件，调用 `readSessionStartDate` 读取前 4KB，解析首条 transcript 消息的日期；若首条日期仍早于 `fromDate`，则直接跳过整文件读取。
- **消息分类**：
  - `isTranscriptMessage(entry)` 过滤出用户/助手/附件/系统消息。
  - `speculation-accept` 条目累加 `totalSpeculationTimeSavedMs`。
  - 子代理文件路径包含 `${sep}subagents${sep}`，其消息全部标记为 sidechain；sidechain 消息**不计入 session 数与时长**，但**计入 token 与 tool call**。
- **模型使用聚合**：对 `assistant` 消息，累加 `usage` 字段的 `input_tokens` / `output_tokens` / `cache_*`；跳过 `SYNTHETIC_MODEL`。
- **Shot 统计**（内部）：在 sidechain 过滤前执行，按 `parentSessionId` 去重，避免子代理与主会话重复计数。

### 3. 缓存合并与派生计算：`cacheToStats`
- 输入：磁盘缓存（历史聚合） + 今日实时 `ProcessedStats`（可为 null）。
- 合并逻辑：
  - `dailyActivity` 与 `dailyModelTokens` 按日期做 Map merge，同日期数值相加。
  - `modelUsage` 各指标相加；`contextWindow` 与 `maxOutputTokens` 取最大值。
  - `hourCounts` 按小时累加。
  - `totalSessions` / `totalMessages` 直接相加。
  - `longestSession` 在缓存记录与今日记录中比较 duration 取最大。
  - `firstSessionDate` / `lastSessionDate` 取极值。
- 派生字段：
  - `peakActivityDay`：消息数最多的日期。
  - `peakActivityHour`：会话启动次数最多的小时。
  - `totalDays`：首末会话日期间隔天数 + 1。
  - `streaks`：由 `calculateStreaks` 基于活跃日期集合计算。

### 4. 全量聚合入口：`aggregateClaudeCodeStats`
- 使用 `withStatsCacheLock` 保证并发安全。
- **无缓存**：处理所有历史数据（`toDate: yesterday`），写入缓存。
- **缓存过期**：若 `lastComputedDate < yesterday`，处理 `lastComputedDate+1` 到 `yesterday` 的新增日期，merge 后更新缓存。
- **今天始终实时**：在锁外单独调用 `processSessionFiles(allSessionFiles, {fromDate: today, toDate: today})`，再与缓存合并。

### 5. 范围聚合入口：`aggregateClaudeCodeStatsForRange`
- `all` → 直接调用 `aggregateClaudeCodeStats()`。
- `7d` / `30d` → 计算 `fromDate`，直接调用 `processSessionFiles`（**不走缓存**），再经 `processedStatsToClaudeCodeStats` 转换。

### 6. `readSessionStartDate` 的健壮性设计
- 读取前 4KB，按最后一个换行符截断，避免读到半截 JSON。
- 逐行 `jsonParse`，仅当 `entry.type` 属于 `TRANSCRIPT_MESSAGE_TYPES`（user/assistant/attachment/system/progress）且 `isSidechain !== true` 时，才信任其 `timestamp`。
- **关键安全细节**：不能简单字符串搜索 `timestamp`，因为 `file-history-snapshot` 条目会嵌套携带旧会话的 `snapshot.timestamp`，会导致 resumed session 被误判为旧日期而丢弃。

## 关键代码路径与文件引用

```
aggregateClaudeCodeStats()
├── getAllSessionFiles()
│   └── getProjectsDir()  → ~/.claude/projects
├── withStatsCacheLock()
│   ├── loadStatsCache()          ← src/utils/statsCache.ts
│   ├── processSessionFiles()     ← 本文件
│   │   ├── readSessionStartDate() ← 4KB peek
│   │   ├── readJSONLFile()       ← src/utils/json.ts
│   │   ├── isTranscriptMessage() ← src/utils/sessionStorage.ts
│   │   └── getFsImplementation() ← src/utils/fsOperations.ts
│   ├── mergeCacheWithNewStats()  ← src/utils/statsCache.ts
│   └── saveStatsCache()          ← src/utils/statsCache.ts
└── cacheToStats()
    └── calculateStreaks()

aggregateClaudeCodeStatsForRange('7d'|'30d')
└── processSessionFiles() → processedStatsToClaudeCodeStats()
```

**主要调用方**：
- `src/components/Stats.tsx`：UI 层通过 `aggregateClaudeCodeStatsForRange` 拉取数据渲染热力图、模型图表、趣味 facts。
- `src/commands/stats/index.ts`：命令入口懒加载 `src/commands/stats/stats.tsx`，后者再引用 `Stats` 组件。

## 依赖与外部交互

| 依赖模块 | 用途 |
|----------|------|
| `src/utils/statsCache.ts` | 缓存加载、保存、合并、锁、日期工具函数 |
| `src/utils/sessionStorage.ts` | `getProjectsDir`、`isTranscriptMessage` |
| `src/utils/fsOperations.ts` | `getFsImplementation` 提供抽象文件系统接口 |
| `src/utils/json.ts` | `readJSONLFile` 流式解析 JSONL |
| `src/utils/errors.ts` | `errorMessage`、`isENOENT` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/messages.ts` | `SYNTHETIC_MODEL` |
| `src/utils/shell/shellToolUtils.ts` | `SHELL_TOOL_NAMES` 用于 shot 提取 |
| `src/types/logs.ts` | `Entry`、`TranscriptMessage` 类型定义 |
| `src/entrypoints/agentSdkTypes.ts` | `ModelUsage` 类型 |
| `bun:bundle` | `feature()` 特性开关（`SHOT_STATS`） |

## 风险、边界与改进建议

### 风险与边界

1. **Invalid Date 导致崩溃**  
   代码已显式处理：若 `firstTimestamp` 或 `lastTimestamp` 为 `Invalid Date`（某些远程/部分写入的 transcript 缺少 `timestamp`），直接 `continue` 跳过该会话，避免 `toISOString()` 抛出 `RangeError`。

2. **大文件性能**  
   64KB 阈值 + 4KB head peek 是有效的第一道防线，但若文件数量极多（数百个项目 × 数千个会话），`getAllSessionFiles` 的 `readdir` 仍可能成为瓶颈。当前批次大小 20 是一个经验值。

3. **子代理 token 归属**  
   子代理消息全部视为 sidechain，不计入 session 数与时长，但 token 和 tool call 会计入当日统计。这在“按 session 数平均”时可能导致理解偏差。

4. **SHOT_STATS 的缓存失效**  
   当内部开启 `SHOT_STATS` 而缓存是旧版本（无 `shotDistribution`）时，`loadStatsCache` 会强制返回空缓存，触发全量重算。这是设计上的有意行为，但会导致首次 `/stats` 加载变慢。

5. **日期范围边界**  
   `isDateBefore` 采用字符串字典序比较（`YYYY-MM-DD`），在格式统一的前提下正确。若未来引入时区或格式变更，需同步修改。

6. **并发安全**  
   `withStatsCacheLock` 是**进程内内存锁**，对多进程同时运行 `claude` 的场景无法防止缓存竞争。考虑到 CLI 通常为单实例运行，当前设计可接受。

### 改进建议

- **增量文件级缓存**：目前缓存粒度是“天”，可考虑在 `getAllSessionFiles` 层维护文件 mtime → 已处理签名的映射，进一步减少重复读取。
- **异步流式 JSONL 解析**：`readJSONLFile` 当前可能一次性加载整个文件到内存；对 GB 级 transcript 可改用流式解析，降低峰值内存。
- **SHOT_STATS 去重更精确**：当前按 `parentSessionId` 去重，若同一父会话存在多个子代理文件且都触发 shot 提取，仅第一次有效。可考虑将 shot 信息写入主会话元数据，避免事后推断。
- **多进程锁**：若未来支持多实例并行，可将内存锁升级为基于文件描述符的 advisory lock（如 `flock`）。
