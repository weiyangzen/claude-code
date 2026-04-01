# prompt.ts 深度研究文档

## 1. 场景与职责

### 1.1 定位
`prompt.ts` 是 ScheduleCronTool 模块的配置中心，负责：
1. 定义三个工具的名称常量
2. 实现功能开关（Feature Flags）控制
3. 构建工具的描述（description）和提示词（prompt）
4. 管理定时任务系统的运行时配置

### 1.2 使用场景
- **功能开关控制**: 通过 GrowthBook 和构建时特性控制定时任务系统的可用性
- **动态提示词生成**: 根据 durable 功能开关状态生成不同的工具提示词
- **工具注册**: 为 CronCreateTool、CronDeleteTool、CronListTool 提供元数据

### 1.3 核心职责
1. **功能开关管理**: `isKairosCronEnabled()`, `isDurableCronEnabled()`
2. **工具名称常量**: `CRON_CREATE_TOOL_NAME`, `CRON_DELETE_TOOL_NAME`, `CRON_LIST_TOOL_NAME`
3. **提示词构建**: `buildCronCreatePrompt()`, `buildCronDeletePrompt()`, `buildCronListPrompt()`
4. **常量定义**: `DEFAULT_MAX_AGE_DAYS`（任务默认过期天数）

---

## 2. 功能点目的

### 2.1 功能开关体系

#### 2.1.1 主开关: `isKairosCronEnabled()`

控制整个定时任务系统的可用性，三层控制：

| 层级 | 控制方式 | 优先级 |
|------|---------|--------|
| 构建时 | `feature('AGENT_TRIGGERS')` | 最高（死代码消除） |
| 环境变量 | `CLAUDE_CODE_DISABLE_CRON` | 中 |
| 运行时 | GrowthBook `tengu_kairos_cron` | 低（默认 true） |

```typescript
export function isKairosCronEnabled(): boolean {
  return feature('AGENT_TRIGGERS')
    ? !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CRON) &&
        getFeatureValue_CACHED_WITH_REFRESH(
          'tengu_kairos_cron',
          true,        // 默认值
          5 * 60 * 1000 // 5 分钟刷新
        )
    : false
}
```

#### 2.1.2 Durable 开关: `isDurableCronEnabled()`

控制持久化功能的可用性，独立于主开关：

```typescript
export function isDurableCronEnabled(): boolean {
  return getFeatureValue_CACHED_WITH_REFRESH(
    'tengu_kairos_cron_durable',
    true,        // 默认值 true（确保 Bedrock/Vertex 用户可用）
    5 * 60 * 1000 // 5 分钟刷新
  )
}
```

**设计考虑**:
- 默认 `true`：确保 Bedrock/Vertex/Foundry 和 DISABLE_TELEMETRY 用户可使用 durable cron
- 不检查 `CLAUDE_CODE_DISABLE_CRON`：这是主开关的责任

### 2.2 工具名称常量

```typescript
export const CRON_CREATE_TOOL_NAME = 'CronCreate'
export const CRON_DELETE_TOOL_NAME = 'CronDelete'
export const CRON_LIST_TOOL_NAME = 'CronList'
```

这些名称用于：
- 工具注册和识别
- 提示词中引用其他工具
- 日志和遥测

### 2.3 默认过期时间

```typescript
export const DEFAULT_MAX_AGE_DAYS =
  DEFAULT_CRON_JITTER_CONFIG.recurringMaxAgeMs / (24 * 60 * 60 * 1000)
// = 7 天
```

循环任务默认 7 天后自动过期，防止无限累积。

---

## 3. 具体技术实现

### 3.1 CronCreate 提示词构建

```typescript
export function buildCronCreatePrompt(durableEnabled: boolean): string
```

提示词结构：
1. **基本说明**: 用途和 cron 格式
2. **One-shot 任务**: `recurring: false` 的用法
3. **Recurring 任务**: `recurring: true` 的用法
4. **避免整点**: 负载分散建议
5. **持久化说明**: 根据 `durableEnabled` 动态生成
6. **运行时行为**: 触发时机和抖动说明
7. **过期策略**: 7 天限制

**关键片段**:
```
## Avoid the :00 and :30 minute marks when the task allows it

Every user who asks for "9am" gets `0 9`, and every user who asks for "hourly" 
gets `0 *` — which means requests from across the planet land on the API at 
the same instant. When the user's request is approximate, pick a minute that 
is NOT 0 or 30:
  "every morning around 9" → "57 8 * * *" or "3 9 * * *" (not "0 9 * * *")
  "hourly" → "7 * * * *" (not "0 * * * *")
```

### 3.2 CronDelete 提示词构建

```typescript
export function buildCronDeletePrompt(durableEnabled: boolean): string
```

简短提示词，根据 durable 开关调整描述：

**durableEnabled = true**:
```
Cancel a cron job previously scheduled with CronCreate. Removes it from 
.claude/scheduled_tasks.json (durable jobs) or the in-memory session store 
(session-only jobs).
```

**durableEnabled = false**:
```
Cancel a cron job previously scheduled with CronCreate. Removes it from 
the in-memory session store.
```

### 3.3 CronList 提示词构建

```typescript
export function buildCronListPrompt(durableEnabled: boolean): string
```

同样根据 durable 开关调整：

**durableEnabled = true**:
```
List all cron jobs scheduled via CronCreate, both durable 
(.claude/scheduled_tasks.json) and session-only.
```

**durableEnabled = false**:
```
List all cron jobs scheduled via CronCreate in this session.
```

### 3.4 描述构建

```typescript
export function buildCronCreateDescription(durableEnabled: boolean): string
```

用于工具描述（Tool.description），比 prompt 更简短：

**durableEnabled = true**:
```
Schedule a prompt to run at a future time — either recurring on a cron 
schedule, or once at a specific time. Pass durable: true to persist to 
.claude/scheduled_tasks.json; otherwise session-only.
```

**durableEnabled = false**:
```
Schedule a prompt to run at a future time within this Claude session — 
either recurring on a cron schedule, or once at a specific time.
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖关系

```
prompt.ts
├── bun:bundle (feature)
├── services/analytics/growthbook.js (getFeatureValue_CACHED_WITH_REFRESH)
├── utils/cronTasks.js (DEFAULT_CRON_JITTER_CONFIG)
└── utils/envUtils.js (isEnvTruthy)
```

### 4.2 被依赖关系

```
CronCreateTool.ts ──┐
CronDeleteTool.ts ──┼──> prompt.ts
CronListTool.ts ────┘
```

### 4.3 关键文件依赖

| 文件路径 | 用途 |
|---------|------|
| `bun:bundle` | `feature()` 构建时特性检查 |
| `src/services/analytics/growthbook.js` | GrowthBook 功能开关 |
| `src/utils/cronTasks.js` | `DEFAULT_CRON_JITTER_CONFIG` |
| `src/utils/envUtils.js` | `isEnvTruthy` 环境变量解析 |

---

## 5. 依赖与外部交互

### 5.1 与 GrowthBook 的交互

```typescript
getFeatureValue_CACHED_WITH_REFRESH(
  'tengu_kairos_cron',      // 或 'tengu_kairos_cron_durable'
  true,                      // 默认值
  5 * 60 * 1000             // 刷新间隔（5 分钟）
)
```

特点：
- **缓存**: 同步读取，避免阻塞
- **后台刷新**: TTL 到期后后台更新
- **默认值**: GrowthBook 不可用时使用

### 5.2 与构建系统的交互

```typescript
import { feature } from 'bun:bundle'
```

`feature('AGENT_TRIGGERS')` 是构建时特性：
- 如果为 `false`，整个定时任务模块会被死代码消除
- 相关工具不会被打包到最终产物

### 5.3 与环境变量的交互

```typescript
import { isEnvTruthy } from '../../utils/envUtils.js'
```

支持的环境变量：
- `CLAUDE_CODE_DISABLE_CRON`: 设置为 truthy 值时禁用定时任务

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 功能开关耦合
- **风险**: 两个开关（主开关和 durable 开关）可能产生意外组合
- **场景**: 
  - 主开关关闭时 durable 开关仍可能为 true（但工具已被禁用）
  - 需要确保工具实现正确处理 `isEnabled()`

#### 6.1.2 提示词长度
- **风险**: `buildCronCreatePrompt` 生成的提示词较长
- **影响**: 增加 token 消耗
- **缓解**: 这是必要的指导信息，无法大幅缩减

#### 6.1.3 刷新延迟
- **风险**: 5 分钟刷新间隔意味着功能开关变化有延迟
- **影响**: 紧急关闭/开启需要等待最多 5 分钟
- **缓解**: 可通过重启客户端立即生效

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| GrowthBook 不可用 | 使用默认值 `true` |
| `AGENT_TRIGGERS = false` | 整个模块被消除，`isKairosCronEnabled` 不存在 |
| 环境变量和 GB 冲突 | 环境变量优先（在代码中先检查） |
| 刷新期间调用 | 返回缓存值，后台更新 |

### 6.3 改进建议

#### 6.3.1 提示词国际化
- **建议**: 支持多语言提示词
- **实现**: 根据用户语言设置选择提示词
- **挑战**: 需要维护多语言版本

#### 6.3.2 提示词模板化
- **建议**: 使用模板引擎生成提示词
- **实现**: 
  ```typescript
  const template = `
  Schedule a prompt to run at a future time...
  {{#if durableEnabled}}
  ## Durability
  ...
  {{/if}}
  `;
  ```

#### 6.3.3 动态配置
- **建议**: 将更多参数（如 MAX_JOBS、默认过期时间）放入 GrowthBook
- **实现**: 新增 `tengu_kairos_cron_limits` 配置
- **好处**: 无需发版即可调整限制

#### 6.3.4 提示词版本控制
- **建议**: 为提示词添加版本号
- **实现**: 
  ```typescript
  export const CRON_PROMPT_VERSION = '2024-01-15';
  ```
- **好处**: 便于追踪提示词变更影响

#### 6.3.5 A/B 测试支持
- **建议**: 支持提示词的 A/B 测试
- **实现**: 通过 GrowthBook 返回不同提示词变体
- **场景**: 测试不同措辞对模型行为的影响

### 6.4 测试建议

建议添加以下测试：
1. **开关测试**: 各开关组合下的行为
2. **提示词测试**: 生成的提示词包含必要信息
3. **缓存测试**: 刷新机制正确工作
4. **默认值测试**: GrowthBook 不可用时使用默认值

---

## 7. 附录

### 7.1 完整提示词示例

**CronCreate（durableEnabled = true）**:
```
Schedule a prompt to be enqueued at a future time. Use for both recurring 
schedules and one-shot reminders.

Uses standard 5-field cron in the user's local timezone: minute hour day-of-month 
month day-of-week. "0 9 * * *" means 9am local — no timezone conversion needed.

## One-shot tasks (recurring: false)

For "remind me at X" or "at <time>, do Y" requests — fire once then auto-delete.
Pin minute/hour/day-of-month/month to specific values:
  "remind me at 2:30pm today to check the deploy" → cron: "30 14 <today_dom> <today_month> *", recurring: false
  "tomorrow morning, run the smoke test" → cron: "57 8 <tomorrow_dom> <tomorrow_month> *", recurring: false

## Recurring jobs (recurring: true, the default)

For "every N minutes" / "every hour" / "weekdays at 9am" requests:
  "*/5 * * * *" (every 5 min), "0 * * * *" (hourly), "0 9 * * 1-5" (weekdays at 9am local)

## Avoid the :00 and :30 minute marks when the task allows it

Every user who asks for "9am" gets `0 9`, and every user who asks for "hourly" 
gets `0 *` — which means requests from across the planet land on the API at 
the same instant. When the user's request is approximate, pick a minute that 
is NOT 0 or 30:
  "every morning around 9" → "57 8 * * *" or "3 9 * * *" (not "0 9 * * *")
  "hourly" → "7 * * * *" (not "0 * * * *")
  "in an hour or so, remind me to..." → pick whatever minute you land on, don't round

Only use minute 0 or 30 when the user names that exact time and clearly means 
it ("at 9:00 sharp", "at half past", coordinating with a meeting). When in 
doubt, nudge a few minutes early or late — the user will not notice, and the 
fleet will.

## Durability

By default (durable: false) the job lives only in this Claude session — nothing 
is written to disk, and the job is gone when Claude exits. Pass durable: true 
to write to .claude/scheduled_tasks.json so the job survives restarts. Only use 
durable: true when the user explicitly asks for the task to persist ("keep doing 
this every day", "set this up permanently"). Most "remind me in 5 minutes" / 
"check back in an hour" requests should stay session-only.

## Runtime behavior

Jobs only fire while the REPL is idle (not mid-query). Durable jobs persist to 
.claude/scheduled_tasks.json and survive session restarts — on next launch they 
resume automatically. One-shot durable tasks that were missed while the REPL was 
closed are surfaced for catch-up. Session-only jobs die with the process. The 
scheduler adds a small deterministic jitter on top of whatever you pick: recurring 
tasks fire up to 10% of their period late (max 15 min); one-shot tasks landing on 
:00 or :30 fire up to 90 s early. Picking an off-minute is still the bigger lever.

Recurring tasks auto-expire after 7 days — they fire one final time, then are 
deleted. This bounds session lifetime. Tell the user about the 7-day limit when 
scheduling recurring jobs.

Returns a job ID you can pass to CronDelete.
```

### 7.2 相关配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `KAIROS_CRON_REFRESH_MS` | 5 * 60 * 1000 | 功能开关刷新间隔 |
| `DEFAULT_MAX_AGE_DAYS` | 7 | 循环任务默认过期天数 |
| `MAX_JOBS` | 50 | 任务数量上限（在 CronCreateTool 中定义） |

### 7.3 GrowthBook 配置

| Feature Flag | 类型 | 默认 | 说明 |
|-------------|------|------|------|
| `tengu_kairos_cron` | boolean | true | 主开关 |
| `tengu_kairos_cron_durable` | boolean | true | Durable 功能开关 |
| `tengu_kairos_cron_config` | JSON | DEFAULT_CRON_JITTER_CONFIG | 抖动配置 |
