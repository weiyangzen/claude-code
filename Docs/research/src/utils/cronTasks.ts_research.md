# cronTasks.ts 深度研究文档

## 场景与职责

`cronTasks.ts` 是 Claude Code 定时任务系统的 **任务数据管理层**，负责 `.claude/scheduled_tasks.json` 文件的读写操作、任务生命周期管理以及抖动时间计算。它是调度器的底层存储接口。

### 核心职责

1. **任务文件操作**：读取、写入和更新 `scheduled_tasks.json`
2. **任务 CRUD**：创建、读取、更新、删除任务
3. **抖动计算**：计算带抖动的下次执行时间
4. **错过任务检测**：识别在 Claude 未运行时错过的任务
5. **会话任务管理**：支持内存中的临时任务

### 使用场景

- **任务创建**：`CronCreateTool` 调用 `addCronTask` 创建新任务
- **任务列表**：`CronListTool` 调用 `listAllCronTasks` 获取所有任务
- **任务删除**：`CronDeleteTool` 调用 `removeCronTasks` 删除任务
- **调度器核心**：`cronScheduler.ts` 调用 `readCronTasks` 加载任务
- **抖动应用**：`cronScheduler.ts` 调用 `jitteredNextCronRunMs` 计算执行时间

---

## 功能点目的

### 1. 任务文件读写

**`readCronTasks`**：读取并解析任务文件，返回有效的任务列表。
- 文件不存在或不可读 → 返回空数组
- 无效的任务条目 → 静默跳过（记录调试日志）
- 无效的 cron 表达式 → 静默跳过

**`writeCronTasks`**：将任务列表写回文件。
- 自动创建 `.claude/` 目录
- 剥离运行时字段（`durable`, `agentId`）
- 空列表写入空文件（而非删除），确保 chokidar 触发事件

**`hasCronTasksSync`**：同步检查文件是否有任务（用于启动优化）。

### 2. 任务 CRUD

**`addCronTask`**：添加新任务。
- 生成 8 字符短 ID（UUID 前 8 位）
- 支持持久化（写入文件）和会话级（仅内存）两种模式
- 会话任务直接添加到 `STATE.sessionCronTasks`

**`removeCronTasks`**：删除指定 ID 的任务。
- 先尝试从会话存储删除
- 如果全部是会话任务，跳过文件操作
- 否则从文件过滤并写回

**`markCronTasksFired`**：标记周期性任务已触发。
- 更新 `lastFiredAt` 字段
- 批量写入优化（N 次触发 = 1 次写文件）

**`listAllCronTasks`**：获取所有任务（文件 + 会话）。
- 会话任务标记 `durable: false`
- Daemon 模式（`dir` 指定时）只返回文件任务

### 3. 抖动计算

**`jitteredNextCronRunMs`**：周期性任务的抖动计算。
- 基于任务 ID 的哈希值生成确定性随机数
- 延迟 = 间隔 × `recurringFrac` × 哈希值，上限 `recurringCapMs`
- 目的：分散相同 cron 表达式的任务的执行时间

**`oneShotJitteredNextCronRunMs`**：一次性任务的抖动计算。
- 仅在分钟值匹配 `oneShotMinuteMod` 时应用（默认 :00/:30）
- 提前时间 = `floor + hash × (max - floor)`
- 目的：让用户设置的时间（如 3:00）稍微提前执行，分散负载

**`jitterFrac`**：从任务 ID 生成确定性哈希值。
- 取 ID 前 8 位（16进制）→ 转换为 0-1 之间的浮点数
- 非 16 进制 ID 回退到 0

### 4. 错过任务检测

**`findMissedTasks`**：找出已错过的任务。
- 从 `createdAt` 计算下次执行时间
- 如果下次执行时间 < 当前时间 → 错过
- 适用于一次性任务和周期性任务

---

## 具体技术实现

### 核心数据结构

```typescript
type CronTask = {
  id: string                    // 任务唯一标识（8字符短 UUID）
  cron: string                  // 5字段 cron 表达式
  prompt: string                // 触发时执行的提示
  createdAt: number             // 创建时间戳（epoch ms）
  lastFiredAt?: number          // 上次触发时间（周期性任务）
  recurring?: boolean           // 是否周期性
  permanent?: boolean           // 是否永久（不过期）
  durable?: boolean             // 运行时：是否持久化到磁盘
  agentId?: string              // 运行时：所属 teammate 的 agent ID
}

type CronFile = { tasks: CronTask[] }

type CronJitterConfig = {
  recurringFrac: number         // 周期性任务延迟比例
  recurringCapMs: number        // 周期性任务延迟上限
  oneShotMaxMs: number          // 一次性任务最大提前
  oneShotFloorMs: number        // 一次性任务最小提前
  oneShotMinuteMod: number      // 一次性任务抖动触发的分钟模数
  recurringMaxAgeMs: number     // 周期性任务最大存活时间
}
```

### 文件路径

```typescript
const CRON_FILE_REL = join('.claude', 'scheduled_tasks.json')
// 实际路径: <projectRoot>/.claude/scheduled_tasks.json
```

### 抖动计算算法

**周期性任务**：
```
t1 = nextCronRunMs(cron, fromMs)           // 下次执行时间
t2 = nextCronRunMs(cron, t1)               // 再下次执行时间（用于计算间隔）
interval = t2 - t1                          // 任务间隔
jitter = min(hash(id) * recurringFrac * interval, recurringCapMs)
return t1 + jitter                          // 延迟后的执行时间
```

**一次性任务**：
```
t1 = nextCronRunMs(cron, fromMs)
if t1.getMinutes() % oneShotMinuteMod !== 0:
    return t1                               // 不匹配模数，不抖动
lead = oneShotFloorMs + hash(id) * (oneShotMaxMs - oneShotFloorMs)
return max(t1 - lead, fromMs)               // 提前执行，但不早于创建时间
```

### 默认值

```typescript
const DEFAULT_CRON_JITTER_CONFIG: CronJitterConfig = {
  recurringFrac: 0.1,                    // 10% 间隔
  recurringCapMs: 15 * 60 * 1000,        // 15 分钟
  oneShotMaxMs: 90 * 1000,               // 90 秒
  oneShotFloorMs: 0,                     // 0 秒
  oneShotMinuteMod: 30,                  // :00 和 :30
  recurringMaxAgeMs: 7 * 24 * 60 * 60 * 1000,  // 7 天
}
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `readCronTasks` | 91-140 | 读取任务文件 |
| `hasCronTasksSync` | 146-158 | 同步检查是否有任务 |
| `writeCronTasks` | 165-182 | 写入任务文件 |
| `addCronTask` | 194-219 | 添加任务 |
| `removeCronTasks` | 231-248 | 删除任务 |
| `markCronTasksFired` | 261-278 | 标记任务已触发 |
| `listAllCronTasks` | 288-296 | 获取所有任务 |
| `nextCronRunMs` | 302-307 | 计算下次执行时间 |
| `jitteredNextCronRunMs` | 381-398 | 周期性任务抖动计算 |
| `oneShotJitteredNextCronRunMs` | 421-445 | 一次性任务抖动计算 |
| `findMissedTasks` | 453-458 | 检测错过任务 |

### 导出类型

| 类型 | 行号 | 用途 |
|------|------|------|
| `CronTask` | 30-70 | 任务数据结构 |
| `CronJitterConfig` | 315-346 | 抖动配置 |
| `DEFAULT_CRON_JITTER_CONFIG` | 348-355 | 默认配置值 |

### 依赖文件

```
cronTasks.ts
├── 被调用方（上游）
│   ├── src/utils/cronScheduler.ts           # 调度器核心
│   ├── src/utils/cronJitterConfig.ts        # 配置获取
│   ├── src/tools/ScheduleCronTool/CronCreateTool.ts   # 创建工具
│   ├── src/tools/ScheduleCronTool/CronListTool.ts     # 列表工具
│   ├── src/tools/ScheduleCronTool/CronDeleteTool.ts   # 删除工具
│   └── src/tools/ScheduleCronTool/prompt.ts           # 提示词
├── 被依赖模块（下游）
│   ├── src/bootstrap/state.js               # 会话任务状态
│   ├── src/utils/cron.js                    # cron 计算
│   ├── src/utils/debug.js                   # 调试日志
│   ├── src/utils/errors.js                  # 错误处理
│   ├── src/utils/fsOperations.js            # FS 抽象
│   ├── src/utils/json.js                    # JSON 解析
│   ├── src/utils/log.js                     # 错误日志
│   └── src/utils/slowOperations.js          # JSON 序列化
└── Node.js 内置
    ├── crypto                               # randomUUID
    ├── fs (readFileSync)                    # 同步读取
    └── fs/promises                          # 异步文件操作
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `crypto` | `randomUUID()` 生成任务 ID |
| `fs` | `readFileSync` 用于 `hasCronTasksSync` |
| `fs/promises` | 异步文件操作 |
| `path` | 路径拼接 |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.js` | `addSessionCronTask`, `getProjectRoot`, `getSessionCronTasks`, `removeSessionCronTasks` | 会话任务管理 |
| `src/utils/cron.js` | `computeNextCronRun`, `parseCronExpression` | cron 计算 |
| `src/utils/debug.js` | `logForDebugging` | 调试日志 |
| `src/utils/errors.js` | `isFsInaccessible` | 错误分类 |
| `src/utils/fsOperations.js` | `getFsImplementation` | FS 抽象 |
| `src/utils/json.js` | `safeParseJSON` | 安全 JSON 解析 |
| `src/utils/log.js` | `logError` | 错误日志 |
| `src/utils/slowOperations.js` | `jsonStringify` | JSON 序列化 |

---

## 风险、边界与改进建议

### 已知风险

1. **文件竞争**
   - 多进程同时读写 `scheduled_tasks.json` 可能导致数据丢失
   - 当前依赖 chokidar 事件和锁机制缓解，但非原子操作

2. **任务 ID 冲突**
   - 8 字符短 UUID 理论上可能冲突（概率极低：1/16^8 ≈ 1/4e9）
   - 冲突会导致任务被覆盖

3. **抖动可预测性**
   - `jitterFrac` 基于任务 ID 是确定性的，可能被利用预测执行时间
   - 对于安全敏感场景不适用

4. **时间计算精度**
   - `nextCronRunMs` 使用本地时间，DST 切换时可能异常
   - 闰秒等边界情况未处理

5. **内存与文件不一致**
   - 会话任务仅存在于内存，进程崩溃时丢失
   - 文件写入失败时无重试机制

### 边界情况

| 场景 | 处理 |
|------|------|
| 文件不存在 | `readCronTasks` 返回空数组 |
| 文件权限不足 | `isFsInaccessible` 捕获，返回空数组 |
| JSON 解析失败 | `safeParseJSON` 返回 null，视为空文件 |
| 任务字段缺失 | 验证失败，跳过该任务 |
| cron 表达式无效 | 验证失败，跳过该任务 |
| 删除不存在的 ID | 静默忽略 |
| 任务数量超过 MAX_JOBS | `CronCreateTool` 层面拦截 |

### 改进建议

1. **原子性增强**
   - 使用文件锁或原子写入（write-then-rename）防止数据损坏
   - 添加写入确认机制

2. **ID 生成**
   - 考虑使用完整 UUID 或检查冲突
   - 添加创建时间戳到 ID 增加唯一性

3. **容错性**
   - 添加文件写入失败重试
   - 实现文件损坏时的备份恢复

4. **性能优化**
   - 缓存文件内容，减少重复读取
   - 使用流式写入处理大量任务

5. **可观测性**
   - 添加任务文件操作指标
   - 记录任务生命周期事件

6. **功能扩展**
   - 支持任务标签/分类
   - 添加任务执行历史记录
   - 支持任务优先级

### 相关 Issue/PR 参考

- `#19931`: 定时任务功能基础实现
- `#20467`: Brief 模式工具结果处理
