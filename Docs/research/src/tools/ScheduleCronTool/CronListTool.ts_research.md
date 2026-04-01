# CronListTool.ts 深度研究文档

## 1. 场景与职责

### 1.1 定位
`CronListTool` 是 Claude Code 定时任务调度系统的查询工具，负责列出所有活动的定时任务。它为用户和模型提供任务可见性，支持任务管理和调试。

### 1.2 使用场景
- **查看当前任务**: "show me my scheduled tasks"
- **确认任务创建**: 验证新任务是否成功添加
- **准备删除**: 查找要删除的任务 ID
- **调试**: 检查任务状态、调度时间等
- **助手模式**: 自动列出任务作为上下文

### 1.3 核心职责
1. 列出所有活动的定时任务（session-only 和 durable）
2. 为每个任务生成人类可读的调度描述
3. 实施权限控制：队友只能看到自己创建的任务
4. 提供并发安全保证（`isConcurrencySafe: true`）
5. 标记为只读操作（`isReadOnly: true`）

---

## 2. 功能点目的

### 2.1 输入参数

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 无 | - | - | 该工具不接受任何输入参数 |

### 2.2 输出结构

```typescript
{
  jobs: [
    {
      id: string           // 任务 ID
      cron: string         // 原始 cron 表达式
      humanSchedule: string // 人类可读的描述，如 "Every day at 9:00 AM"
      prompt: string       // 任务提示词
      recurring?: boolean  // 是否循环（仅当 true 时包含）
      durable?: boolean    // 是否持久化（仅当 false 时包含，即 session-only）
    }
  ]
}
```

### 2.3 权限模型

| 用户类型 | 可见任务 |
|---------|---------|
| Team Lead（无 teammate 上下文） | 所有任务（文件 + 内存） |
| Teammate（有 teammate 上下文） | 仅自己创建的任务（通过 `agentId` 过滤） |

过滤逻辑：
```typescript
const ctx = getTeammateContext()
const tasks = ctx
  ? allTasks.filter(t => t.agentId === ctx.agentId)
  : allTasks
```

### 2.4 人类可读描述

使用 `cronToHuman()` 函数将 cron 表达式转换为自然语言：

| Cron 表达式 | humanSchedule |
|------------|---------------|
| `"*/5 * * * *"` | "Every 5 minutes" |
| `"0 9 * * *"` | "Every day at 9:00 AM" |
| `"0 9 * * 1-5"` | "Weekdays at 9:00 AM" |
| `"0 0 * * 0"` | "Every Sunday at 12:00 AM" |

---

## 3. 具体技术实现

### 3.1 数据获取流程

```typescript
async call()
```

执行步骤：
1. **获取所有任务**: `listAllCronTasks()` 合并文件任务和 session 任务
2. **权限过滤**: 如果是 teammate，按 `agentId` 过滤
3. **格式化输出**: 为每个任务生成人类可读的调度描述
4. **返回结果**: 按输出 schema 组装数据

### 3.2 任务格式化

```typescript
const jobs = tasks.map(t => ({
  id: t.id,
  cron: t.cron,
  humanSchedule: cronToHuman(t.cron),
  prompt: t.prompt,
  ...(t.recurring ? { recurring: true } : {}),
  ...(t.durable === false ? { durable: false } : {}),
}))
```

格式化规则：
- `recurring`: 仅当为 `true` 时包含（默认为一次性任务）
- `durable`: 仅当为 `false` 时包含（默认为 durable，省略表示）

### 3.3 底层数据合并

```typescript
// src/utils/cronTasks.ts
export async function listAllCronTasks(dir?: string): Promise<CronTask[]>
```

合并逻辑：
1. 读取文件任务: `readCronTasks(dir)`
2. 获取 session 任务: `getSessionCronTasks()`
3. 为 session 任务标记 `durable: false`
4. 合并返回: `[...fileTasks, ...sessionTasks]`

---

## 4. 关键代码路径与文件引用

### 4.1 核心调用链

```
用户输入 → CronListTool
    ↓
call() → listAllCronTasks()
    ↓
readCronTasks() + getSessionCronTasks()
    ↓
权限过滤（如果是 teammate）
    ↓
格式化（cronToHuman）
    ↓
返回任务列表
```

### 4.2 关键文件依赖

| 文件路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `buildTool`, `ToolDef` |
| `src/utils/cron.ts` | `cronToHuman`（人类可读描述） |
| `src/utils/cronTasks.ts` | `listAllCronTasks`（获取所有任务） |
| `src/utils/format.ts` | `truncate`（截断长文本） |
| `src/utils/lazySchema.ts` | `lazySchema`（延迟加载 Zod schema） |
| `src/utils/teammateContext.ts` | `getTeammateContext`（权限过滤） |
| `src/tools/ScheduleCronTool/prompt.ts` | 工具名称、描述、功能开关 |
| `src/tools/ScheduleCronTool/UI.tsx` | 结果渲染组件 |

### 4.3 工具定义特性

```typescript
export const CronListTool = buildTool({
  name: CRON_LIST_TOOL_NAME, // 'CronList'
  searchHint: 'list active cron jobs',
  maxResultSizeChars: 100_000,
  shouldDefer: true,
  isConcurrencySafe: true,  // 可并发执行
  isReadOnly: true,         // 只读操作
  // ...
})
```

### 4.4 结果格式化输出

```typescript
mapToolResultToToolResultBlockParam(output, toolUseID)
```

输出格式：
```
a1b2c3d4 — Every day at 9:00 AM (recurring): Check PRs
e5f6g7h8 — Weekdays at 9:00 AM (one-shot) [session-only]: Run tests
```

- 使用 `truncate(j.prompt, 80, true)` 截断长提示词
- 标记循环任务 `(recurring)` 和一次性任务 `(one-shot)`
- 标记 session-only 任务 `[session-only]`

---

## 5. 依赖与外部交互

### 5.1 与任务存储的交互

```typescript
const allTasks = await listAllCronTasks()
```

该接口自动处理：
- 文件读取（异步）
- 内存访问（同步）
- 数据合并和标记

### 5.2 与队友系统的交互

```typescript
const ctx = getTeammateContext()
const tasks = ctx
  ? allTasks.filter(t => t.agentId === ctx.agentId)
  : allTasks
```

队友隔离机制：
- 通过 `agentId` 字段识别任务所有者
- Team lead（无上下文）看到所有任务
- 队友只能看到自己的任务

### 5.3 与功能开关的交互

```typescript
isEnabled() {
  return isKairosCronEnabled()
}
```

工具仅在定时任务系统启用时可用。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 任务列表可能过时
- **风险**: 列表获取和显示之间存在时间窗口，任务可能已被删除或触发
- **行为**: 这是最终一致性设计，不影响正确性
- **缓解**: 用户应理解列表是快照，非实时

#### 6.1.2 大量任务性能
- **风险**: 接近 50 个任务上限时，列表可能较长
- **当前**: 无分页或过滤机制
- **影响**: 输出可能较长，但 `truncate` 限制了提示词显示长度

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 无任务 | 返回 `jobs: []`，UI 显示 "No scheduled jobs." |
| 只有 session-only 任务 | 正常列出，标记 `[session-only]` |
| 只有 durable 任务 | 正常列出，不标记（默认） |
| 混合任务 | 按文件任务 + session 任务顺序列出 |
| 队友无任务 | 返回空列表 |
| 任务有无效 cron | `readCronTasks` 会自动过滤，不显示 |

### 6.3 改进建议

#### 6.3.1 过滤和搜索
- **建议**: 支持按状态、类型、时间范围过滤
- **实现**: 扩展输入参数
  ```typescript
  inputSchema: z.object({
    type: z.enum(['all', 'recurring', 'one-shot']).optional(),
    durable: z.boolean().optional(),
    search: z.string().optional(), // 搜索 prompt 内容
  })
  ```

#### 6.3.2 分页支持
- **建议**: 大量任务时支持分页
- **实现**: 添加 `limit` 和 `offset` 参数

#### 6.3.3 排序选项
- **建议**: 支持按创建时间、下次执行时间排序
- **实现**: 添加 `sortBy` 和 `sortOrder` 参数

#### 6.3.4 详细信息模式
- **建议**: 支持显示更多字段（如 `createdAt`, `lastFiredAt`）
- **实现**: 添加 `detailed: boolean` 参数

#### 6.3.5 下次执行时间显示
- **建议**: 显示任务的下次执行时间
- **实现**: 调用 `nextCronRunMs()` 或 `jitteredNextCronRunMs()` 计算

### 6.4 测试建议

建议添加以下测试场景：
1. **空列表**: 无任务时的行为
2. **权限测试**: 队友只能看到自己的任务
3. **混合任务**: session-only 和 durable 同时存在
4. **格式化测试**: `humanSchedule` 正确生成
5. **截断测试**: 长提示词正确截断
6. **并发测试**: 列表获取期间任务被修改

---

## 7. 附录

### 7.1 输出示例

**有任务时**:
```
a1b2c3d4 — Every day at 9:00 AM (recurring): Check PRs and review code
e5f6g7h8 — Every 5 minutes (recurring) [session-only]: Monitor build status
```

**无任务时**:
```
No scheduled jobs.
```

### 7.2 相关工具

| 工具 | 关系 |
|------|------|
| `CronCreateTool` | 创建任务，可被列出 |
| `CronDeleteTool` | 删除任务，需要列表获取 ID |

### 7.3 UI 渲染

列表结果的 UI 渲染由 `UI.tsx` 中的 `renderListResultMessage` 处理：

```typescript
export function renderListResultMessage(output: ListOutput): React.ReactNode {
  if (output.jobs.length === 0) {
    return <MessageResponse>
        <Text dimColor>No scheduled jobs</Text>
      </MessageResponse>;
  }
  return <MessageResponse>
      {output.jobs.map(j => <Text key={j.id}>
          <Text bold>{j.id}</Text> <Text dimColor>{j.humanSchedule}</Text>
        </Text>)}
    </MessageResponse>;
}
```

显示格式：
```
⎿  a1b2c3d4 Every day at 9:00 AM
   e5f6g7h8 Weekdays at 9:00 AM
```

### 7.4 性能考虑

- **时间复杂度**: O(n)，n 为任务数（目前上限 50）
- **文件 I/O**: 每次调用读取一次 JSON 文件
- **缓存**: 无内置缓存，依赖文件系统缓存
- **优化建议**: 可考虑添加短期缓存（如 1 秒）减少重复读取
