# src/utils/statsCache.ts 研究文档

## 场景与职责

`src/utils/statsCache.ts` 是 Claude Code `/stats` 功能的**磁盘缓存管理层**。它的核心使命是将“已完结的历史日期”的统计结果持久化到本地磁盘，避免用户每次打开 `/stats` 都重新扫描可能数量庞大的 JSONL 会话文件。该模块通过版本化缓存结构、原子写、内存级互斥锁等手段，保证缓存的可靠性、可迁移性与并发安全。

核心职责：
- 定义并维护 `PersistedStatsCache` 的持久化 schema 及版本（当前 v3）。
- 提供 `withStatsCacheLock` 进程内互斥锁，防止并发读写导致缓存损坏。
- 实现缓存的加载、版本迁移、校验、保存与合并。
- 提供日期工具函数（`toDateString`、`getTodayDateString`、`isDateBefore` 等），供上层 `stats.ts` 统一使用。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `withStatsCacheLock` | 进程内异步互斥锁，确保同一时刻只有一个缓存读写操作在执行 |
| `loadStatsCache` | 从磁盘加载缓存；若版本不匹配则尝试迁移；若结构非法或无法迁移则返回空缓存 |
| `saveStatsCache` | 原子写缓存：先写临时文件，再 `fsync`，最后 `rename` 覆盖目标文件 |
| `mergeCacheWithNewStats` | 将新计算出的某日/某段统计增量合并到已有缓存中，按日期、模型、小时等维度做数值累加 |
| `migrateStatsCache` | 将 v1/v2 旧缓存升级至当前 v3 schema，尽量保留历史聚合数据 |
| 日期工具函数 | 统一 ISO `YYYY-MM-DD` 格式处理，避免上层重复实现 |

## 具体技术实现

### 1. 缓存 Schema：`PersistedStatsCache`

```ts
export type PersistedStatsCache = {
  version: number              // 当前固定为 3
  lastComputedDate: string | null  // 已完全计算的最后日期（含）
  dailyActivity: DailyActivity[]
  dailyModelTokens: DailyModelTokens[]
  modelUsage: { [modelName: string]: ModelUsage }
  totalSessions: number
  totalMessages: number
  longestSession: SessionStats | null
  firstSessionDate: string | null
  hourCounts: { [hour: number]: number }
  totalSpeculationTimeSavedMs: number
  shotDistribution?: { [shotCount: number]: number }  // v3 新增（内部）
}
```

设计原则：**用有界聚合字段替代无界原始数组**。例如：
- 不保存所有 `sessionStats[]`，只保留 `totalSessions`、`totalMessages`、`longestSession`。
- `dailyActivity` 与 `dailyModelTokens` 按天聚合，上限为使用天数。
- `hourCounts` 固定最多 24 个键。
- 这样可确保缓存文件大小不会随会话数量线性膨胀。

### 2. 进程内互斥锁：`withStatsCacheLock`

- 使用模块级变量 `statsCacheLockPromise: Promise<void> | null` 作为锁状态。
- 进入时循环 `await` 已有锁；执行完 `fn()` 后在 `finally` 中释放。
- **限制**：仅为单进程内存锁，对多进程并发无防护能力。考虑到 CLI 通常为单实例，当前设计足够。

### 3. 缓存加载与版本迁移：`loadStatsCache`

加载流程：
1. `fs.readFile(cachePath, 'utf-8')`。
2. `jsonParse` 后检查 `version`。
3. 若 `version !== STATS_CACHE_VERSION`（当前 3）：
   - 调用 `migrateStatsCache`。
   - 若迁移失败（版本过旧或结构非法），记录 debug log，返回空缓存。
   - 若迁移成功，**立即回写** `saveStatsCache(migrated)`，避免每次启动都重复迁移。
4. 若 `feature('SHOT_STATS')` 开启但缓存缺少 `shotDistribution`，记录 debug log 并**返回空缓存**，强制触发全量重算以补齐内部指标。
5. 基础结构校验：确保 `dailyActivity`、`dailyModelTokens` 为数组，`totalSessions` / `totalMessages` 为数字。

迁移策略（`migrateStatsCache`）：
- 可接受版本范围：`[MIN_MIGRATABLE_VERSION, STATS_CACHE_VERSION]`，即 `[1, 3]`。
- 对缺失字段使用默认值填充（`??`），但 `shotDistribution` 在迁移时**保留 `undefined`**，以便触发第 4 步的强制重算逻辑。
- 注释中明确指出：旧版本可能在子代理 token 等字段存在欠计数，但“保留历史聚合优于直接丢弃”。

### 4. 原子写：`saveStatsCache`

- 临时文件路径：`${cachePath}.${randomBytes(8).toString('hex')}.tmp`
- 使用 `fs/promises.open(tempPath, 'w', 0o600)` 创建文件，写入 `jsonStringify(cache, null, 2)` 后调用 `handle.sync()` 强制刷盘。
- 最后 `fs.rename(tempPath, cachePath)` 完成原子替换。
- 异常时尝试 `unlink` 临时文件，忽略清理错误。
- 目录不存在时，通过 `fs.mkdir(configDir)` 自动创建（忽略已存在错误）。

### 5. 缓存合并：`mergeCacheWithNewStats`

该函数用于 `aggregateClaudeCodeStats` 在增量更新历史日期时，将新计算出的 `ProcessedStats` 合并进旧缓存：

- `dailyActivity`：按 `date` 做 Map 合并，同日期 `messageCount` / `sessionCount` / `toolCallCount` 相加。
- `dailyModelTokens`：按 `date` → `model` 二级 Map 合并，token 数相加。
- `modelUsage`：同名模型各字段相加；`contextWindow` / `maxOutputTokens` 取最大值。
- `hourCounts`：按小时数值累加。
- `totalSessions` / `totalMessages`：直接相加。
- `longestSession`：比较 duration 取最大。
- `firstSessionDate`：取时间戳最小值。
- `totalSpeculationTimeSavedMs`：相加。
- `shotDistribution`（若 `SHOT_STATS` 开启）：shot 次数 → 会话数 相加。

### 6. 日期工具函数

| 函数 | 行为 |
|------|------|
| `toDateString(date)` | `date.toISOString().split('T')[0]`，非法日期会抛错 |
| `getTodayDateString()` | `toDateString(new Date())` |
| `getYesterdayDateString()` | 当前日期减 1 天后转字符串 |
| `isDateBefore(a, b)` | 字典序比较 `a < b`（要求格式均为 `YYYY-MM-DD`） |

## 关键代码路径与文件引用

```
aggregateClaudeCodeStats()  ← src/utils/stats.ts
├── withStatsCacheLock()
│   └── loadStatsCache()
│       ├── getStatsCachePath() → ~/.claude/stats-cache.json
│       ├── jsonParse()         ← src/utils/slowOperations.ts
│       └── migrateStatsCache()
│           └── saveStatsCache()  (迁移成功后立即回写)
├── processSessionFiles()       ← src/utils/stats.ts
├── mergeCacheWithNewStats()
│   └── 按维度 Map merge
└── saveStatsCache()
    ├── jsonStringify()         ← src/utils/slowOperations.ts
    ├── open(tempPath).sync()
    └── rename(tempPath, cachePath)
```

**主要调用方**：
- `src/utils/stats.ts`：`aggregateClaudeCodeStats` 是唯一的缓存读写调用方。
- `src/utils/heatmap.ts`：导入 `toDateString` 用于热力图日期格式化。

## 依赖与外部交互

| 依赖模块 | 用途 |
|----------|------|
| `src/utils/stats.ts` | 导入 `DailyActivity`、`DailyModelTokens`、`SessionStats` 类型 |
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir` 确定缓存根目录 `~/.claude` |
| `src/utils/fsOperations.ts` | `getFsImplementation` 提供抽象文件系统接口 |
| `src/utils/slowOperations.ts` | `jsonParse`、`jsonStringify` 安全序列化/反序列化 |
| `src/utils/debug.ts` | `logForDebugging` 记录版本迁移、强制重算等诊断信息 |
| `src/utils/log.ts` | `logError` 在保存失败时记录异常 |
| `src/entrypoints/agentSdkTypes.ts` | `ModelUsage` 类型 |
| `bun:bundle` | `feature('SHOT_STATS')` 特性开关 |
| Node.js 内置 | `crypto.randomBytes`、`fs/promises.open`、`path.join` |

## 风险、边界与改进建议

### 风险与边界

1. **缓存版本强制重算的代价**  
   当 `SHOT_STATS` 开启且旧缓存无 `shotDistribution` 时，`loadStatsCache` 会直接返回空缓存，导致 `aggregateClaudeCodeStats` 下次运行时全量重新扫描所有历史文件。对于拥有大量历史会话的用户，这可能造成 `/stats` 首次加载明显变慢（数秒级）。

2. **进程内锁的局限性**  
   `withStatsCacheLock` 仅在同进程内有效。若用户同时运行多个 `claude` 实例（如不同终端窗口），仍可能出现读写竞争。虽然 `saveStatsCache` 使用原子 `rename` 可保证最终文件不会半写，但读取方可能读到旧版本或触发 `jsonParse` 失败（概率极低）。

3. **缓存文件权限**  
   临时文件以 `0o600` 创建，但 `rename` 后的最终文件权限取决于操作系统默认 umask。在共享机器上，缓存中的使用数据（模型、token 数、活跃日期）属于隐私信息，建议显式 `chmod` 最终文件。

4. **日期边界假设**  
   整个缓存策略基于“昨天及之前的数据不会再变”这一假设。若用户手动修改历史 transcript 文件（如恢复备份、清理旧文件），缓存将不会感知，导致统计与文件实际内容不一致。当前无 checksum 或文件级签名机制。

5. **迁移的欠计数风险**  
   注释中已承认：v2 缓存缺少子代理 token，迁移后这些历史数据会永久欠计数。由于 transcript 可能已被 `cleanupPeriodDays` 清理，无法重新计算，这是可接受的工程权衡。

### 改进建议

- **文件级 advisory lock**：将 `withStatsCacheLock` 升级为基于 `fs-ext` 或 Node.js `fs.open` + `flock` 的跨进程锁，彻底消除多实例竞争。
- **缓存校验和机制**：在 `getAllSessionFiles` 层维护 `{filePath, mtime, size}` 的摘要，当检测到历史文件发生变化时，仅使对应日期区间的缓存失效并局部重算。
- **延迟后台重算**：在 `SHOT_STATS` 缺失场景下，可先返回旧缓存保证 UI 响应，再在后台 fork 或异步任务中完成全量重算并更新缓存。
- **压缩或二进制格式**：当用户拥有数年历史时，JSON 格式的 `stats-cache.json` 可能膨胀到数 MB。可考虑使用 MessagePack 或 Snappy 压缩，减少磁盘 I/O。
- **缓存权限加固**：在 `saveStatsCache` 的 `rename` 成功后显式调用 `fs.chmod(cachePath, 0o600)`，确保隐私数据不被同机器其他用户读取。
