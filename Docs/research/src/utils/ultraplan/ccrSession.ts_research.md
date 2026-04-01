# ccrSession.ts 研究文档

## 场景与职责

`ccrSession.ts` 是 Claude Code CLI 中 `/ultraplan` 功能的核心模块，负责与远程 Claude Code on the web (CCR) 会话进行交互，实现计划模式的轮询和状态管理。

**核心场景：**
1. **Ultraplan 功能**：用户通过 `/ultraplan <prompt>` 或关键词触发，在远程 CCR 会话中生成详细计划
2. **计划审批流程**：远程 CCR 生成计划后，等待用户在浏览器中审批（批准/拒绝/编辑）
3. **Teleport 模式**：用户可以选择将计划传回本地 CLI 执行（而非在远程执行）

**文件位置**：`src/utils/ultraplan/ccrSession.ts` (349 行)

---

## 功能点目的

### 1. ExitPlanModeScanner - 事件流状态机

纯状态ful的分类器，用于处理 CCR 事件流并提取 ExitPlanMode 工具的结果。

**状态转换：**
```
running → (turn ends, no ExitPlanMode) → needs_input
needs_input → (user replies in browser) → running
running → (ExitPlanMode emitted, no result yet) → plan_ready
plan_ready → (rejected) → running
plan_ready → (approved) → poll resolves, pill removed
```

**优先级**（从高到低）：approved > terminated > rejected > pending > unchanged

### 2. pollForApprovedExitPlanMode - 主轮询函数

异步轮询远程会话，等待用户批准计划。

**关键特性：**
- 30 分钟超时 (`ULTRAPLAN_TIMEOUT_MS = 30 * 60 * 1000`)
- 3 秒轮询间隔 (`POLL_INTERVAL_MS = 3000`)
- 最大连续失败 5 次 (`MAX_CONSECUTIVE_FAILURES = 5`)
- 支持阶段回调 (`onPhaseChange`): running → needs_input → plan_ready
- 支持外部停止检查 (`shouldStop`)

### 3. 计划提取与解析

**两种执行目标：**
- `'remote'`: 用户在浏览器中批准，远程 CCR 直接执行
- `'local'`: 用户点击 "teleport back to terminal"，计划传回本地执行

**标记提取：**
- 批准标记：`## Approved Plan:\n` 或 `## Approved Plan (edited by user):\n`
- Teleport 标记：`__ULTRAPLAN_TELEPORT_LOCAL__\n`

---

## 具体技术实现

### 数据结构

```typescript
// 扫描结果类型
export type ScanResult =
  | { kind: 'approved'; plan: string }
  | { kind: 'teleport'; plan: string }
  | { kind: 'rejected'; id: string }
  | { kind: 'pending' }
  | { kind: 'terminated'; subtype: string }
  | { kind: 'unchanged' }

// 轮询结果
export type PollResult = {
  plan: string
  rejectCount: number
  executionTarget: 'local' | 'remote'
}

// Ultraplan 阶段
export type UltraplanPhase = 'running' | 'needs_input' | 'plan_ready'
```

### ExitPlanModeScanner 核心逻辑

```typescript
class ExitPlanModeScanner {
  private exitPlanCalls: string[] = []        // ExitPlanMode 工具调用 ID 列表
  private results = new Map<string, ToolResultBlockParam>()  // 工具结果映射
  private rejectedIds = new Set<string>()     // 被拒绝的调用 ID
  private terminated: { subtype: string } | null = null
  private rescanAfterRejection = false
  everSeenPending = false

  // 检查是否有待处理的计划（有 tool_use 但没有 tool_result）
  get hasPendingPlan(): boolean {
    const id = this.exitPlanCalls.findLast(c => !this.rejectedIds.has(c))
    return id !== undefined && !this.results.has(id)
  }

  // 摄入新事件并返回当前状态
  ingest(newEvents: SDKMessage[]): ScanResult
}
```

**ingest 方法流程：**
1. 遍历新事件，分类处理：
   - `assistant` 消息：提取 `tool_use` 类型的 ExitPlanMode 调用
   - `user` 消息：提取 `tool_result` 结果
   - `result` 消息（非 success）：标记会话终止

2. 逆序扫描 ExitPlanMode 调用：
   - 跳过已拒绝的 ID
   - 无结果 → `pending`
   - `is_error === true` → 检查是否为 teleport 标记
   - 正常结果 → `approved`

### 轮询流程

```typescript
export async function pollForApprovedExitPlanMode(
  sessionId: string,
  timeoutMs: number,
  onPhaseChange?: (phase: UltraplanPhase) => void,
  shouldStop?: () => boolean,
): Promise<PollResult>
```

**轮询循环：**
1. 检查超时和停止信号
2. 调用 `pollRemoteSessionEvents` 获取新事件
3. 使用 `ExitPlanModeScanner.ingest()` 处理事件
4. 根据结果类型处理：
   - `approved`/`teleport`: 返回计划
   - `terminated`: 抛出错误
   - `rejected`: 记录拒绝，继续轮询
   - `pending`: 更新阶段状态
5. 计算当前阶段并触发回调
6. 等待 3 秒后继续

### 阶段判断逻辑

```typescript
const quietIdle =
  (sessionStatus === 'idle' || sessionStatus === 'requires_action') &&
  newEvents.length === 0

const phase: UltraplanPhase = scanner.hasPendingPlan
  ? 'plan_ready'
  : quietIdle
    ? 'needs_input'
    : 'running'
```

**关键洞察：**
- `hasPendingPlan` 优先于 session_status
- `quietIdle` 表示会话空闲且没有新事件（用户可能在浏览器中回复）
- 事件流动时始终视为 `running`，即使 session_status 显示 idle

### 错误处理

**UltraplanPollError 类型：**
```typescript
export type PollFailReason =
  | 'terminated'           // 远程会话终止
  | 'timeout_pending'      // 超时，但已看到 pending 状态
  | 'timeout_no_plan'      // 超时，从未到达 plan 阶段
  | 'extract_marker_missing' // 批准结果缺少标记
  | 'network_or_unknown'   // 网络错误或其他未知错误
  | 'stopped'              // 被调用方停止
```

---

## 关键代码路径与文件引用

### 调用链

```
/ultraplan 命令 (src/commands/ultraplan.tsx)
  └── launchUltraplan()
      └── launchDetached()
          └── teleportToRemote() (src/utils/teleport.tsx)
          └── registerRemoteAgentTask() (src/tasks/RemoteAgentTask/RemoteAgentTask.tsx)
          └── startDetachedPoll() (src/commands/ultraplan.tsx)
              └── pollForApprovedExitPlanMode() (本文件)
                  └── pollRemoteSessionEvents() (src/utils/teleport.tsx)
                  └── ExitPlanModeScanner.ingest()
```

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/utils/teleport.tsx` | `pollRemoteSessionEvents`, `archiveRemoteSession` |
| `src/tools/ExitPlanModeTool/constants.ts` | `EXIT_PLAN_MODE_V2_TOOL_NAME` |
| `src/entrypoints/agentSdkTypes.ts` | `SDKMessage` 类型 |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/sleep.ts` | `sleep` |
| `src/utils/teleport/api.ts` | `isTransientNetworkError` |

### 被调用方

| 文件 | 用途 |
|------|------|
| `src/commands/ultraplan.tsx` | 主调用方，`pollForApprovedExitPlanMode` |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `UltraplanPhase` 类型，任务状态管理 |

---

## 依赖与外部交互

### API 调用

**pollRemoteSessionEvents** (`src/utils/teleport.tsx`):
- 调用 CCR Events API: `GET /v1/sessions/{sessionId}/events`
- 支持分页（最多 50 页）
- 返回 `SDKMessage[]`, `lastEventId`, `sessionStatus`

**Session Status 值：**
- `'idle'`: 会话空闲
- `'running'`: 正在运行
- `'requires_action'`: 需要用户操作
- `'archived'`: 已归档

### 工具结果格式

**ExitPlanMode 工具结果（批准）：**
```
## Approved Plan:
<plan content here>
```

或编辑后的版本：
```
## Approved Plan (edited by user):
<edited plan content>
```

**Teleport 结果（拒绝中的特殊标记）：**
```
__ULTRAPLAN_TELEPORT_LOCAL__
<plan content here>
```

### 与 RemoteAgentTask 的集成

`ExitPlanModeScanner` 产生的阶段状态通过 `ultraplanPhase` 字段同步到 `RemoteAgentTaskState`：

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx
export type RemoteAgentTaskState = TaskStateBase & {
  // ...
  ultraplanPhase?: Exclude<UltraplanPhase, 'running'>
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **网络可靠性**
   - 30 分钟轮询期间可能遇到 ~600 次 API 调用
   - 任何非零的 5xx 错误率都可能导致轮询失败
   - 已实现 `MAX_CONSECUENT_FAILURES = 5` 的容错机制

2. **OAuth Token 过期**
   - 代码注释指出：TODO(prod-hardening): OAuth token may go stale over the 30min poll
   - 建议：实现 token 刷新机制

3. **空计划或分支问题**
   - `extractApprovedPlan` 在缺少标记时会抛出错误
   - 可能原因：远程命中 empty-plan 或 isAgent 分支

### 边界情况

1. **批量事件处理**
   - `pollRemoteSessionEvents` 每轮最多获取 50 页事件
   - 一个批次可能同时包含批准结果和后续 `result` 消息
   - 实现优先返回批准结果（即使会话随后崩溃）

2. **拒绝与迭代**
   - 用户可以多次拒绝计划并在浏览器中迭代
   - `rejectedIds` Set 跟踪已拒绝的调用 ID
   - `rescanAfterRejection` 确保拒绝后重新扫描

3. **会话终止检测**
   - 仅 `result` 消息的 error subtypes 触发终止
   - `result(success)` 在每轮 CCR 后触发，不代表终止

### 改进建议

1. **架构改进**
   - 代码注释提到 TODO(#23985): 将 `ExitPlanModeScanner` 整合到 `startRemoteSessionPolling` 中，移除 `startDetachedPoll`
   - 当前有两个轮询循环：`startDetachedPoll` (ccrSession) 和 `startRemoteSessionPolling` (RemoteAgentTask)

2. **错误处理增强**
   - 区分不同类型的网络错误（401/403/404/5xx）
   - 为 401 添加自动重试与 token 刷新

3. **可观测性**
   - 添加更多调试日志记录轮询状态变化
   - 记录每次 ingest 的事件数量和类型

4. **性能优化**
   - 考虑使用 WebSocket 替代轮询（如果 CCR 支持）
   - 实现指数退避策略减少 API 调用

### 测试建议

- 单元测试：`ExitPlanModeScanner` 是纯函数，可使用合成事件进行测试
- 集成测试：模拟 `pollRemoteSessionEvents` 返回各种事件序列
- 边界测试：拒绝后重新批准、空计划、网络故障恢复
