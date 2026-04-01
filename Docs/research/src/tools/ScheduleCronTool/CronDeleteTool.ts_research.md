# CronDeleteTool.ts 深度研究文档

## 1. 场景与职责

### 1.1 定位
`CronDeleteTool` 是 Claude Code 定时任务调度系统的删除工具，负责通过任务 ID 取消已创建的定时任务。它是 CronCreateTool 的对应操作，构成完整的任务生命周期管理。

### 1.2 使用场景
- **取消提醒**: 用户不再需要之前设置的定时提醒
- **停止周期性任务**: 停止每小时/每天的自动检查
- **清理过期任务**: 删除不再需要的旧任务
- **错误修正**: 删除因误操作创建的错误任务

### 1.3 核心职责
1. 根据任务 ID 查找并删除指定任务
2. 支持删除内存中的 session-only 任务和文件中的 durable 任务
3. 实施权限控制：队友只能删除自己创建的任务
4. 提供清晰的删除确认反馈

---

## 2. 功能点目的

### 2.1 输入参数

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 要删除的任务 ID（由 CronCreateTool 返回） |

### 2.2 权限模型

| 用户类型 | 可见任务 | 可删除任务 |
|---------|---------|-----------|
| Team Lead（无 teammate 上下文） | 所有任务 | 所有任务 |
| Teammate（有 teammate 上下文） | 仅自己创建的任务 | 仅自己创建的任务 |

权限检查逻辑：
```typescript
const ctx = getTeammateContext()
if (ctx && task.agentId !== ctx.agentId) {
  return {
    result: false,
    message: `Cannot delete cron job '${input.id}': owned by another agent`,
    errorCode: 2,
  }
}
```

### 2.3 删除策略
- **Session-only 任务**: 直接从内存数组中移除
- **Durable 任务**: 从 `.claude/scheduled_tasks.json` 文件中移除
- **自动清理**: 如果任务在删除前已被触发（一次性任务），文件可能已被自动清理

---

## 3. 具体技术实现

### 3.1 输入验证流程

```typescript
async validateInput(input): Promise<ValidationResult>
```

验证步骤：
1. **任务存在性检查**: 调用 `listAllCronTasks()` 获取所有任务，查找匹配 ID
2. **权限检查**: 如果当前是 teammate 上下文，验证任务所有权

### 3.2 删除执行流程

```typescript
async call({ id })
```

执行步骤：
1. 调用 `removeCronTasks([id])` 执行删除
2. 返回被删除的任务 ID

### 3.3 底层删除逻辑

```typescript
// src/utils/cronTasks.ts
export async function removeCronTasks(ids: string[], dir?: string): Promise<void>
```

删除策略：
1. **优先检查 session store**: 如果所有 ID 都在 session store 中找到并删除，直接返回
2. **回退到文件操作**: 如果 session store 未完全匹配，读取 JSON 文件，过滤掉指定 ID，写回文件

```typescript
// 删除逻辑伪代码
if (dir === undefined && removeSessionCronTasks(ids) === ids.length) {
  return // 所有任务都在 session store 中，已完成删除
}

const idSet = new Set(ids)
const tasks = await readCronTasks(dir)
const remaining = tasks.filter(t => !idSet.has(t.id))
if (remaining.length === tasks.length) return // 无匹配
await writeCronTasks(remaining, dir)
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心调用链

```
用户输入 → CronDeleteTool
    ↓
validateInput (检查任务存在性、权限)
    ↓
call() → removeCronTasks([id])
    ↓
removeSessionCronTasks() 或 writeCronTasks()
    ↓
任务从内存/文件中移除
```

### 4.2 关键文件依赖

| 文件路径 | 用途 |
|---------|------|
| `src/Tool.ts` | `buildTool`, `ToolDef`, `ValidationResult` |
| `src/utils/cronTasks.ts` | `getCronFilePath`, `listAllCronTasks`, `removeCronTasks` |
| `src/utils/lazySchema.ts` | `lazySchema`（延迟加载 Zod schema） |
| `src/utils/teammateContext.ts` | `getTeammateContext`（权限检查） |
| `src/tools/ScheduleCronTool/prompt.ts` | 工具名称、描述、功能开关 |
| `src/tools/ScheduleCronTool/UI.tsx` | 结果渲染组件 |

### 4.3 工具定义

```typescript
export const CronDeleteTool = buildTool({
  name: CRON_DELETE_TOOL_NAME, // 'CronDelete'
  searchHint: 'cancel a scheduled cron job',
  maxResultSizeChars: 100_000,
  shouldDefer: true, // 需要 ToolSearch 后才能使用
  // ...
})
```

---

## 5. 依赖与外部交互

### 5.1 与任务存储的交互

CronDeleteTool 不直接操作存储，而是通过 `cronTasks.ts` 提供的抽象：

```typescript
// 统一删除接口
await removeCronTasks([id])
```

该接口自动处理：
- Session store 删除（内存操作）
- JSON 文件读写（异步文件操作）
- 并发安全（通过文件锁和原子写入）

### 5.2 与队友系统的交互

```typescript
// 验证时检查队友上下文
const ctx = getTeammateContext()
if (ctx && task.agentId !== ctx.agentId) {
  // 拒绝删除
}
```

队友任务隔离机制：
- 队友创建的任务带有 `agentId` 标记
- 队友只能看到 `agentId === ctx.agentId` 的任务
- Team lead（无上下文）可以看到所有任务

### 5.3 与功能开关的交互

```typescript
isEnabled() {
  return isKairosCronEnabled()
}
```

工具仅在 `isKairosCronEnabled()` 返回 true 时可用。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 竞态条件
- **风险**: 任务在验证和删除之间被触发并自动删除（一次性任务）
- **行为**: `removeCronTasks` 是幂等的，无匹配时静默返回
- **影响**: 用户可能收到 "任务不存在" 的错误，尽管验证时存在

#### 6.1.2 权限绕过风险
- **风险**: 如果 teammate 上下文验证逻辑有漏洞，可能删除其他队友的任务
- **当前防护**: 严格检查 `task.agentId !== ctx.agentId`

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 任务 ID 不存在 | 验证失败，返回错误码 1 |
| 队友尝试删除他人任务 | 验证失败，返回错误码 2 |
| 任务已被自动删除 | `removeCronTasks` 静默返回，无错误 |
| 删除 durable 任务时文件被锁 | 等待锁释放后重试（底层实现） |
| 并发删除同一任务 | 先执行者成功，后执行者无操作（幂等） |

### 6.3 改进建议

#### 6.3.1 批量删除支持
- **建议**: 支持一次删除多个任务
- **实现**: 修改输入 schema 接受 `ids: string[]`
- **注意**: 需要保持向后兼容，支持单 ID 字符串

#### 6.3.2 删除确认机制
- **建议**: 支持 `--force` 或交互式确认
- **场景**: 防止误删重要任务
- **实现**: 在 `validateInput` 或 `checkPermissions` 中添加确认逻辑

#### 6.3.3 删除历史记录
- **建议**: 记录删除操作，便于审计
- **实现**: 扩展日志或新增删除记录文件

#### 6.3.4 任务搜索/过滤删除
- **建议**: 支持按 cron 表达式或 prompt 内容搜索并删除
- **实现**: 扩展输入参数，或新增 `CronSearchTool`

### 6.4 测试建议

建议添加以下测试场景：
1. **正常删除**: session-only 和 durable 任务的删除
2. **权限测试**: 队友只能删除自己的任务
3. **并发测试**: 同时删除同一任务
4. **边界测试**: 删除不存在的任务、删除已被自动清理的任务
5. **集成测试**: 删除后立即列出，验证任务确实消失

---

## 7. 附录

### 7.1 错误码定义

| 错误码 | 含义 |
|--------|------|
| 1 | 任务 ID 不存在 |
| 2 | 无权删除（属于其他 agent） |

### 7.2 相关工具

| 工具 | 关系 |
|------|------|
| `CronCreateTool` | 创建任务，返回 ID 供删除使用 |
| `CronListTool` | 列出任务，获取可删除的 ID |

### 7.3 UI 渲染

删除结果的 UI 渲染由 `UI.tsx` 中的 `renderDeleteResultMessage` 处理：

```typescript
export function renderDeleteResultMessage(output: DeleteOutput): React.ReactNode {
  return <MessageResponse>
      <Text>
        Cancelled <Text bold>{output.id}</Text>
      </Text>
    </MessageResponse>;
}
```

显示格式：
```
⎿  Cancelled a1b2c3d4
```
