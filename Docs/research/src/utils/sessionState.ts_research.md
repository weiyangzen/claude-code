# sessionState.ts 研究文档

> 文件路径：`src/utils/sessionState.ts`  
> 大小：约 5,309 bytes（150 行）  
> 研究范围：代码、调用方、被调用方、配置、测试、脚本、必要上下文

---

## 一、场景与职责

`sessionState.ts` 是 Claude Code 会话状态机的**权威信号源**。它维护一个极简的三种状态模型（`idle` / `running` / `requires_action`），并通过监听器模式将状态变更广播到多个下游消费者。该模块的设计目标是：

1. **解耦状态产生者与消费者**：任何代码路径（如 `query.ts`、权限对话框、REPL 桥接）只需调用 `notifySessionStateChanged`，无需关心 CCR（Claude Code Remote/IDE 客户端）、SDK 事件流、推送通知等具体实现。
2. **统一权限模式同步**：`notifyPermissionModeChanged` 作为单一 choke point，确保所有改变 `toolPermissionContext.mode` 的路径都能同步到外部系统。
3. **支持外部元数据透传**：`SessionExternalMetadata` 允许将权限模式、待处理动作详情、模型信息、任务摘要等写入可查询的 JSON 字段，供前端/IDE 消费。

### 核心场景

- **主循环状态切换**：`query.ts` / `cli/print.ts` 在模型开始生成时设置为 `running`，在生成结束或需要用户确认时切换为 `idle` 或 `requires_action`。
- **权限请求阻塞**：当工具需要用户确认时，状态变为 `requires_action`，并携带 `RequiresActionDetails`（工具名、描述、tool_use_id、request_id、输入参数）。
- **SDK/Headless 状态流**：`cli/print.ts` 注册监听器，将状态变更转换为 SDK `system:status` 消息或 `session_state_changed` 子类型事件。
- **CCR 外部元数据同步**：`onChangeAppState.ts` 注册 `metadataListener`，将 `permission_mode`、`is_ultraplan_mode`、`pending_action` 等通过 HTTP PUT 同步到 CCR 后端。
- **任务摘要清除**：当状态回到 `idle` 时，自动清除 `task_summary`，防止下一回合短暂显示上一回合的进度摘要。

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| `SessionState` 类型 | 三态模型：`idle`（空闲）、`running`（运行中）、`requires_action`（等待用户动作）。 |
| `RequiresActionDetails` | 描述阻塞原因的结构化数据，支持 proto 序列化（CCR webhook）和 JSON 透传（`external_metadata.pending_action`）。 |
| `SessionExternalMetadata` | 定义可同步到 CCR 后端的元数据键集合，包括 `permission_mode`、`model`、`pending_action`、`post_turn_summary`、`task_summary` 等。 |
| `setSessionStateChangedListener` / `setSessionMetadataChangedListener` / `setPermissionModeChangedListener` | 注册三类监听器，分别消费会话状态、元数据、权限模式变更。 |
| `getSessionState` | 获取当前内存中的会话状态。 |
| `notifySessionStateChanged` | 更新 `currentState` 并触发监听器；自动将 `requires_action` 的 details 镜像到 `metadataListener`；在 `idle` 时清除 `task_summary`；在环境变量开启时向 SDK 事件队列发送 `session_state_changed` 事件。 |
| `notifySessionMetadataChanged` | 触发元数据监听器，用于 `onChangeAppState.ts` 等模块推送外部元数据。 |
| `notifyPermissionModeChanged` | 触发权限模式监听器，确保所有模式变更路径都被 CCR 和 SDK 感知。 |

---

## 三、具体技术实现

### 3.1 关键数据结构与类型

```ts
export type SessionState = 'idle' | 'running' | 'requires_action'

export type RequiresActionDetails = {
  tool_name: string
  action_description: string
  tool_use_id: string
  request_id: string
  input?: Record<string, unknown>
}

export type SessionExternalMetadata = {
  permission_mode?: string | null
  is_ultraplan_mode?: boolean | null
  model?: string | null
  pending_action?: RequiresActionDetails | null
  post_turn_summary?: unknown
  task_summary?: string | null
}

type SessionStateChangedListener = (
  state: SessionState,
  details?: RequiresActionDetails,
) => void

type SessionMetadataChangedListener = (
  metadata: SessionExternalMetadata,
) => void

type PermissionModeChangedListener = (mode: PermissionMode) => void
```

### 3.2 模块级状态

```ts
let stateListener: SessionStateChangedListener | null = null
let metadataListener: SessionMetadataChangedListener | null = null
let permissionModeListener: PermissionModeChangedListener | null = null

let hasPendingAction = false
let currentState: SessionState = 'idle'
```

- 采用**单例监听器**设计：每类监听器同一时间只能注册一个。这符合当前架构中只有一个 CCR 客户端和一个 SDK 输出通道的实际情况。
- `hasPendingAction` 用于追踪当前是否有未决动作，避免在连续的 `requires_action` → `running` → `requires_action` 转换中重复/遗漏清除信号。

### 3.3 `notifySessionStateChanged` 详细逻辑

```ts
export function notifySessionStateChanged(
  state: SessionState,
  details?: RequiresActionDetails,
): void {
  currentState = state
  stateListener?.(state, details)

  // 1. 将 requires_action 的 details 镜像到 external_metadata
  if (state === 'requires_action' && details) {
    hasPendingAction = true
    metadataListener?.({ pending_action: details })
  } else if (hasPendingAction) {
    hasPendingAction = false
    metadataListener?.({ pending_action: null })
  }

  // 2. idle 时清除 task_summary
  if (state === 'idle') {
    metadataListener?.({ task_summary: null })
  }

  // 3. 可选：向 SDK 事件队列发送事件（需环境变量 CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS）
  if (isEnvTruthy(process.env.CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS)) {
    enqueueSdkEvent({ type: 'system', subtype: 'session_state_changed', state })
  }
}
```

#### 设计要点

- **镜像 details 到 metadata**：CCR 前端通过 `external_metadata.pending_action` 查询当前阻塞原因，而无需扫描整个事件流。当状态离开 `requires_action` 时，发送 `pending_action: null`（符合 RFC 7396 JSON Merge Patch 语义，表示删除键）。
- **`task_summary` 的自动清除**：`task_summary` 由 forked summarizer 在中途生成（约每 5 步/2 分钟），用于展示长期运行回合的进度。回合结束回到 `idle` 时必须清除，否则下一回合会短暂显示旧摘要。
- **SDK 事件为 opt-in**：`CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS` 默认未开启。原因是 CCR web/mobile 客户端的 `isWorking()` 启发式逻辑会把 trailing `idle` 事件误识别为 "Running..."，需要等客户端更新后再全面启用。

### 3.4 `notifyPermissionModeChanged` 的 choke point 设计

```ts
export function notifyPermissionModeChanged(mode: PermissionMode): void {
  permissionModeListener?.(mode)
}
```

- 在 `onChangeAppState.ts` 中，任何导致 `toolPermissionContext.mode` 变化的操作（Shift+Tab 循环、ExitPlanMode 对话框、slash command、`set_permission_mode` 桥接命令等）都会经过 `onChangeAppState` 的 diff 检测，进而调用 `notifyPermissionModeChanged`。
- `cli/print.ts` 注册该监听器，将模式变更转换为 SDK `system:status` 消息，使 VS Code / scmuxd 等消费者实时感知。

---

## 四、关键代码路径与文件引用

### 4.1 本文件内核心导出

```ts
// src/utils/sessionState.ts
export type SessionState = 'idle' | 'running' | 'requires_action'
export type RequiresActionDetails = { ... }
export type SessionExternalMetadata = { ... }

export function setSessionStateChangedListener(cb: SessionStateChangedListener | null): void
export function setSessionMetadataChangedListener(cb: SessionMetadataChangedListener | null): void
export function setPermissionModeChangedListener(cb: PermissionModeChangedListener | null): void

export function getSessionState(): SessionState
export function notifySessionStateChanged(state: SessionState, details?: RequiresActionDetails): void
export function notifySessionMetadataChanged(metadata: SessionExternalMetadata): void
export function notifyPermissionModeChanged(mode: PermissionMode): void
```

### 4.2 上游调用方（状态变更生产者）

| 调用方文件 | 调用函数 | 场景说明 |
|-----------|---------|---------|
| `src/state/onChangeAppState.ts` | `notifySessionMetadataChanged`, `notifyPermissionModeChanged` | AppState diff 检测后的统一出口 |
| `src/cli/print.ts` | `notifySessionStateChanged`, `setPermissionModeChangedListener`, `setSessionStateChangedListener`, `setSessionMetadataChangedListener` | SDK/Headless 模式的状态流桥接 |
| `src/cli/remoteIO.ts` | `setSessionStateChangedListener` | 远程 IO 会话状态监听 |
| `src/cli/structuredIO.ts` | `setSessionStateChangedListener`, `setSessionMetadataChangedListener` | 结构化 IO 状态监听 |

> 注：`notifySessionStateChanged('running' / 'idle' / 'requires_action')` 的实际调用点分散在 `query.ts`、`cli/print.ts` 等主循环代码中，通过全局搜索 `notifySessionStateChanged` 可定位。

### 4.3 下游被调用方（状态变更消费者）

| 被调用模块/文件 | 函数/符号 | 用途 |
|----------------|----------|------|
| `src/utils/sdkEventQueue.ts` | `enqueueSdkEvent` | 将状态变更镜像到 SDK 事件流 |
| `src/utils/envUtils.ts` | `isEnvTruthy` | 判断环境变量是否开启 SDK 事件发射 |
| `src/utils/permissions/PermissionMode.ts` | `PermissionMode` (type) | 权限模式类型定义 |

### 4.4 监听器注册与消费链路

```
notifySessionStateChanged('requires_action', details)
  ├─► stateListener ──► cli/print.ts ──► SDK stream: system:status
  ├─► metadataListener ──► onChangeAppState.ts ──► CCR PUT external_metadata
  │                      (pending_action: details)
  └─► (opt-in) enqueueSdkEvent ──► sdkEventQueue.ts ──► drainSdkEvents()

notifyPermissionModeChanged(mode)
  └─► permissionModeListener ──► cli/print.ts ──► SDK stream: system:status
```

---

## 五、依赖与外部交互

### 5.1 与 `onChangeAppState.ts` 的协同

`onChangeAppState.ts` 是 `AppState` 变更的观察者。它通过对比 `oldState` 和 `newState`，在 `toolPermissionContext.mode` 变化时：
1. 调用 `notifyPermissionModeChanged(newMode)` 通知 SDK。
2. 若外部化后的模式（`toExternalPermissionMode`）确实变化，调用 `notifySessionMetadataChanged({ permission_mode, is_ultraplan_mode })` 通知 CCR。

这种分层设计确保了：
- **内部模式**（如 `auto`、`bubble`）不会泄漏到 CCR。
- **所有模式变更路径**都被捕获，无需在每个 mutations 点手动添加通知代码。

### 5.2 与 SDK 事件队列的交互

`enqueueSdkEvent` 仅在 `getIsNonInteractiveSession()` 返回 true 时实际入队。这意味着：
- **TUI 模式**：SDK 事件被丢弃，不会累积到上限（1000）。
- **Headless/SDK 模式**：事件被收集，最终由 `drainSdkEvents()` 刷入输出流。

### 5.3 环境变量控制

| 环境变量 | 作用 |
|---------|------|
| `CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS` | 开启后，`notifySessionStateChanged` 会向 `sdkEventQueue` 发送 `session_state_changed` 子类型事件。 |

---

## 六、风险、边界与改进建议

### 6.1 已知风险与边界

1. **单例监听器的覆盖风险**
   - 当前每类监听器只有一个模块级变量。若多个消费者（如测试 mock、多个 CCR 连接）同时注册，后注册的会覆盖先注册的。
   - 生产环境中目前只有一个合法消费者，但扩展性受限。

2. **`hasPendingAction` 的隐式状态机**
   - `hasPendingAction` 是一个布尔标志，用于决定何时发送 `pending_action: null`。
   - 边界：如果某处直接修改了 `currentState` 而未经过 `notifySessionStateChanged`，`hasPendingAction` 可能与实际状态不一致。但模块已导出 `notifySessionStateChanged` 作为唯一合法变更入口，违规属于编程错误。

3. **`task_summary` 清除的时序假设**
   - 代码假设 `idle` 状态一定意味着"回合真正结束"。若未来出现"回合结束但立即开始后台任务"的场景，`task_summary` 的清除可能过早。
   - 当前注释已说明 `idle` 是在 `heldBackResult` flush 之后触发的，因此后台任务点会替代显示，实际影响可控。

4. **`RequiresActionDetails.input` 的类型安全**
   - `input` 被定义为 `Record<string, unknown>`，前端消费时需要进行运行时校验。注释已说明这是为了让前端"无需 proto 往返即可迭代 shape"。
   - 风险：若前端直接强转类型，可能因 CLI 侧字段变更导致运行时错误。

5. **无单元测试覆盖**
   - 仓库中未找到针对 `sessionState.ts` 的单元测试。`notifySessionStateChanged` 的多个分支（`requires_action` → `idle` → `running`、环境变量开关、`task_summary` 清除）均缺乏自动化验证。

### 6.2 改进建议

1. **支持多监听器注册**
   - 将单例监听器改为数组 `Set<Listener>`，允许注册多个消费者，并提供 `removeSessionStateChangedListener(handle)` 以支持组件生命周期管理。这能提升模块的可测试性和可扩展性。

2. **引入状态变更日志/追踪**
   - 在 `notifySessionStateChanged` 中添加 `logForDebugging` 调用，记录每次状态转换（如 `idle -> running`、`running -> requires_action`），便于排查 CCR/SDK 状态不同步的问题。

3. **将 `task_summary` 清除与后台任务状态关联**
   - 若未来架构演进，可考虑将 `task_summary: null` 的发送条件从 `state === 'idle'` 细化为 "主回合结束且无活跃后台 summarizer"，避免过早清除。

4. **为 `RequiresActionDetails` 增加 Zod 校验模式（可选）**
   - 在前端/桥接层共享一个 `RequiresActionDetailsSchema`，确保 `input` 字段在跨进程/跨网络传输时有基本的运行时类型安全。

5. **补充单元测试**
   - 建议测试以下场景：
     - `notifySessionStateChanged('running')` 触发 `stateListener` 且不影响 `metadataListener`。
     - `notifySessionStateChanged('requires_action', details)` 同时触发 `stateListener` 和 `metadataListener` 的 `pending_action`。
     - 连续两次 `requires_action` 不会重复发送 `pending_action`；从 `requires_action` 到 `running` 会发送 `pending_action: null`。
     - `idle` 状态自动发送 `task_summary: null`。
     - 环境变量开启/关闭时 `enqueueSdkEvent` 的调用与否。
     - `notifyPermissionModeChanged` 正确触发注册的监听器。

6. **考虑将 `SessionExternalMetadata` 的 `post_turn_summary` 类型具体化**
   - 当前注释说明 `post_turn_summary` 是 opaque/unknown，以避免 `sdk.d.ts` 泄漏内部导入路径。可考虑在 `src/entrypoints/agentSdkTypes.ts` 中定义一个轻量接口并在此处复用，在保持类型安全的同时避免 bundle 泄漏。

---

*文档生成时间：2026-04-01*  
*研究执行器：kimi (model=k2p5)*
