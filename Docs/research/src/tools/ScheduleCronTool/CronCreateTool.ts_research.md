# CronCreateTool.ts 深度研究文档

## 1. 场景与职责

### 1.1 定位
`CronCreateTool` 是 Claude Code 定时任务调度系统的核心工具之一，负责创建新的定时任务（Cron Job）。它允许用户通过自然语言或标准 cron 表达式安排提示词（prompt）在指定时间执行，支持循环执行（recurring）或一次性执行（one-shot）两种模式。

### 1.2 使用场景
- **定时提醒**: " remind me at 2:30pm today to check the deploy"
- **周期性检查**: "check my PRs every hour this week"
- **自动化工作流**: 在特定时间触发代码审查、测试运行等任务
- **助手模式内置任务**: 与 `/loop` skill 配合实现自动化循环

### 1.3 核心职责
1. 接收并验证用户输入的 cron 表达式和提示词
2. 支持两种持久化模式：
   - **Session-only** (`durable: false`): 仅内存存储，会话结束即消失
   - **Durable** (`durable: true`): 持久化到 `.claude/scheduled_tasks.json`，跨会话保留
3. 强制执行任务数量上限（MAX_JOBS = 50）
4. 防止队友（teammate）创建 durable 任务（队友不跨会话持久化）
5. 激活调度器（通过 `setScheduledTasksEnabled`）

---

## 2. 功能点目的

### 2.1 输入参数

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `cron` | string | 是 | 标准 5 字段 cron 表达式（M H DoM Mon DoW），本地时区 |
| `prompt` | string | 是 | 任务触发时要执行的提示词 |
| `recurring` | boolean | 否 | `true`（默认）= 循环执行；`false` = 一次性执行 |
| `durable` | boolean | 否 | `true` = 持久化到磁盘；`false`（默认）= 仅会话内有效 |

### 2.2 Cron 表达式规范
- 格式: `minute hour day-of-month month day-of-week`
- 示例:
  - `"*/5 * * * *"` = 每 5 分钟
  - `"0 9 * * 1-5"` = 工作日 9:00
  - `"30 14 28 2 *"` = 每年 2 月 28 日 14:30

### 2.3 负载抖动（Jitter）策略
为避免所有用户在整点（:00、:30）同时触发任务造成负载峰值，系统实现了智能抖动：
- **循环任务**: 在下次执行时间上增加最多 10% 周期延迟（上限 15 分钟）
- **一次性任务**: 如果落在 :00 或 :30，最多提前 90 秒执行
- 抖动量基于任务 ID 的哈希值计算，确保同一任务抖动稳定

### 2.4 任务过期策略
- 循环任务默认 7 天后自动过期（`DEFAULT_MAX_AGE_DAYS = 7`）
- 过期前会最后一次执行，然后自动删除
- `permanent` 标记的任务（系统内置）永不过期

---

## 3. 具体技术实现

### 3.1 输入验证流程

```typescript
async validateInput(input): Promise<ValidationResult>
```

验证步骤（按顺序）：
1. **Cron 语法验证**: 调用 `parseCronExpression()` 检查 5 字段格式
2. **可执行性验证**: 调用 `nextCronRunMs()` 确保未来一年内有匹配时间
3. **数量限制检查**: 当前任务数 >= 50 时拒绝
4. **队友权限检查**: 如果 `input.durable && getTeammateContext()`，拒绝创建（错误码 4）

### 3.2 任务创建流程

```typescript
async call({ cron, prompt, recurring = true, durable = false })
```

执行步骤：
1. **Kill Switch 检查**: `effectiveDurable = durable && isDurableCronEnabled()`
2. **生成任务 ID**: 8 位十六进制 UUID 片段（`randomUUID().slice(0, 8)`）
3. **存储任务**:
   - 如果 `!effectiveDurable`: 调用 `addSessionCronTask()` 存入内存
   - 如果 `effectiveDurable`: 调用 `addCronTask()` 写入 JSON 文件
4. **激活调度器**: `setScheduledTasksEnabled(true)`
5. **返回结果**: 包含 `id`, `humanSchedule`, `recurring`, `durable`

### 3.3 数据结构

**任务存储格式（内存/文件）**:
```typescript
type CronTask = {
  id: string           // 8 位十六进制，如 "a1b2c3d4"
  cron: string         // "0 9 * * 1-5"
  prompt: string       // 要执行的提示词
  createdAt: number    // 创建时间戳（毫秒）
  lastFiredAt?: number // 上次执行时间（仅循环任务）
  recurring?: boolean  // 是否循环
  permanent?: boolean  // 是否永久（系统任务）
  durable?: boolean    // 运行时标记，false 表示仅内存
  agentId?: string     // 创建者队友 ID（仅内存任务）
}
```

### 3.4 文件存储位置

```typescript
const CRON_FILE_REL = join('.claude', 'scheduled_tasks.json')
// 示例: /project/.claude/scheduled_tasks.json
```

文件格式:
```json
{
  "tasks": [
    {
      "id": "a1b2c3d4",
      "cron": "0 9 * * 1-5",
      "prompt": "Check PRs",
      "createdAt": 1712345678901,
      "recurring": true
    }
  ]
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心调用链

```
用户输入 → CronCreateTool
    ↓
validateInput (检查 cron 有效性、任务数量限制)
    ↓
call() → addCronTask() / addSessionCronTask()
    ↓
setScheduledTasksEnabled(true) → 激活调度器
    ↓
scheduler.check() 每秒检查 → 触发任务
```

### 4.2 关键文件依赖

| 文件路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `buildTool`, `ToolDef`, `ValidationResult` |
| `src/utils/cron.ts` | `parseCronExpression`, `cronToHuman` |
| `src/utils/cronTasks.ts` | `addCronTask`, `listAllCronTasks`, `nextCronRunMs` |
| `src/utils/lazySchema.ts` | `lazySchema`（延迟加载 Zod schema） |
| `src/utils/semanticBoolean.ts` | `semanticBoolean`（支持 "true"/"false" 字符串） |
| `src/utils/teammateContext.ts` | `getTeammateContext`（队友权限检查） |
| `src/bootstrap/state.ts` | `setScheduledTasksEnabled` |
| `src/tools/ScheduleCronTool/prompt.ts` | 工具名称、描述、功能开关 |
| `src/tools/ScheduleCronTool/UI.tsx` | 结果渲染组件 |

### 4.3 功能开关控制

```typescript
// prompt.ts
export function isKairosCronEnabled(): boolean {
  return feature('AGENT_TRIGGERS')
    ? !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CRON) &&
        getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron', true, KAIROS_CRON_REFRESH_MS)
    : false
}

export function isDurableCronEnabled(): boolean {
  return getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron_durable', true, KAIROS_CRON_REFRESH_MS)
}
```

- `AGENT_TRIGGERS`: 构建时特性开关（死代码消除）
- `tengu_kairos_cron`: GrowthBook 运行时开关（5 分钟刷新）
- `CLAUDE_CODE_DISABLE_CRON`: 本地环境变量覆盖

---

## 5. 依赖与外部交互

### 5.1 与调度器的交互

```typescript
// 创建任务后激活调度器
setScheduledTasksEnabled(true)
```

调度器（`cronScheduler.ts`）通过 `useScheduledTasks` hook 在 REPL 中运行：
- 每秒检查一次（`CHECK_INTERVAL_MS = 1000`）
- 使用 `chokidar` 监视文件变化
- 通过 PID 锁防止多会话重复执行

### 5.2 与队友系统的交互

```typescript
// 队友只能创建 session-only 任务
if (input.durable && getTeammateContext()) {
  return {
    result: false,
    message: 'durable crons are not supported for teammates...',
    errorCode: 4,
  }
}

// 创建时记录队友 ID
const id = await addCronTask(cron, prompt, recurring, effectiveDurable, getTeammateContext()?.agentId)
```

队友创建的任务：
- 仅存储在内存中（`durable: false`）
- 触发时通过 `agentId` 路由到对应队友的消息队列

### 5.3 与 GrowthBook 的交互

通过 `prompt.ts` 中的函数获取功能开关状态：
- `isKairosCronEnabled()`: 控制整个定时任务系统
- `isDurableCronEnabled()`: 控制持久化功能

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 任务数量上限硬编码
```typescript
const MAX_JOBS = 50
```
- **风险**: 大型项目可能需要更多定时任务
- **缓解**: 当前限制可防止滥用，但应考虑配置化

#### 6.1.2 时间解析边界
- Cron 表达式使用本地时区，跨时区协作时可能产生混淆
- DST（夏令时）切换时，固定小时的任务可能跳过或重复执行

#### 6.1.3 文件锁竞争
- 多会话同时启动时，锁竞争可能导致任务重复执行
- 锁文件（`.claude/scheduled_tasks.lock`）依赖 PID 存活检测

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 无效 cron 表达式 | 验证失败，返回错误码 1 |
| cron 在未来一年无匹配 | 验证失败，返回错误码 2 |
| 任务数达到上限 | 验证失败，返回错误码 3 |
| 队友尝试创建 durable 任务 | 验证失败，返回错误码 4 |
| Kill switch 关闭 durable | `effectiveDurable` 强制为 false，不报错 |
| 任务触发时调度器被 kill | `isKilled()` 检查阻止执行 |

### 6.3 改进建议

#### 6.3.1 任务编辑功能
- **建议**: 支持修改现有任务的 cron 表达式或 prompt，无需删除重建
- **实现**: 新增 `CronUpdateTool`，或扩展 `CronCreateTool` 支持 `upsert` 模式

#### 6.3.2 任务暂停/恢复
- **建议**: 支持临时禁用任务而不删除
- **实现**: 新增 `enabled` 字段，或支持 `pause`/`resume` 操作

#### 6.3.3 任务执行历史
- **建议**: 记录任务执行历史，便于调试和审计
- **实现**: 新增 `.claude/scheduled_tasks.log` 或扩展 JSON 格式

#### 6.3.4 更灵活的 cron 语法
- **建议**: 支持自然语言输入（如 "every 5 minutes"）
- **实现**: 集成 `@breejs/later` 或类似库，或扩展 `/loop` skill 的解析能力

#### 6.3.5 任务分组/标签
- **建议**: 支持按项目或类别组织任务
- **实现**: 新增 `tags` 或 `group` 字段，扩展 `CronListTool` 过滤能力

### 6.4 测试建议

当前未发现针对 ScheduleCronTool 的单元测试。建议添加：
1. **验证逻辑测试**: 各种无效输入的处理
2. **边界测试**: 任务数上限、时间边界（闰年、DST）
3. **并发测试**: 多会话同时创建/删除任务
4. **持久化测试**: 重启后 durable 任务恢复
5. **队友隔离测试**: 队友只能看到自己的任务

---

## 7. 附录

### 7.1 错误码定义

| 错误码 | 含义 |
|--------|------|
| 1 | 无效 cron 表达式 |
| 2 | Cron 表达式在未来一年无匹配 |
| 3 | 任务数量达到上限（50） |
| 4 | 队友不支持 durable 任务 |

### 7.2 相关 Skill

- `/loop`: 封装 CronCreateTool，提供简化的间隔语法（如 `5m`, `1h`）

### 7.3 相关 Issue/PR

- GH #31759: GrowthBook 默认值的讨论
- GH #19931: Cron 作为多天长会话的主要驱动力
- GH #20425: Assistant 模式不再强制 --proactive
