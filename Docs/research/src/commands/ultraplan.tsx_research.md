# Research: src/commands/ultraplan.tsx

## 场景与职责

`ultraplan.tsx` 实现了 `/ultraplan` 斜杠命令，这是 Claude Code CLI 的高级多智能体计划模式功能。该命令允许用户在本地终端发起一个远程计划会话，由 Claude Code on the web (CCR) 使用最强大的模型（Opus 4.6）来制定详细的实施计划。

**核心场景：**
1. 用户需要复杂任务的深度计划（估计 10-30 分钟）
2. 用户希望在终端保持可用的同时，让远程服务并行处理计划
3. 计划完成后，用户可以选择在远程执行或传回本地终端执行
4. 支持关键词触发（用户在普通输入中包含 "ultraplan" 字样）

**定位：** 这是一个 `local-jsx` 类型的命令，内部仅对 ant 构建启用（`"external" === 'ant'`），属于内部功能。

---

## 功能点目的

### 1. 远程计划会话启动
- 创建 CCR 远程会话，使用 `plan` 权限模式
- 默认使用 Opus 4.6 模型（可通过 GrowthBook 配置覆盖）
- 支持可选的 seedPlan（草稿计划）用于细化

### 2. 计划审批流程
- 远程模型生成计划后，在浏览器中展示审批对话框
- 用户可选择：
  - **Approve in CCR**: 在远程直接执行计划
  - **Teleport back**: 将计划传回本地终端执行
  - **Reject**: 拒绝计划，让模型重新生成

### 3. 状态管理与轮询
- 使用 `startDetachedPoll` 进行异步轮询（最长 30 分钟）
- 通过 `ExitPlanModeScanner` 解析远程事件流
- 维护 `UltraplanPhase` 状态：`running` → `needs_input` → `plan_ready`

### 4. 任务生命周期管理
- 集成 `RemoteAgentTask` 框架
- 支持任务取消 (`stopUltraplan`)
- 会话恢复支持（通过 `--resume`）

---

## 具体技术实现

### 关键数据结构

```typescript
// 轮询结果
interface PollResult {
  plan: string;
  rejectCount: number;
  executionTarget: 'local' | 'remote';
}

// AppState 中的 ultraplan 相关状态
interface AppState {
  ultraplanSessionUrl?: string;      // 当前会话 URL
  ultraplanLaunching?: boolean;      // 是否正在启动
  ultraplanPendingChoice?: {         // 等待用户选择
    plan: string;
    sessionId: string;
    taskId: string;
  };
  ultraplanLaunchPending?: {         // 启动前确认
    blurb: string;
  };
}
```

### 核心流程

#### 1. 命令入口 (`call` 函数)
```
用户输入 /ultraplan <prompt>
    ↓
检查是否已有活动会话（防重复启动）
    ↓
有参数? → 显示预启动对话框 (ultraplanLaunchPending)
无参数? → 显示使用说明
```

#### 2. 启动流程 (`launchDetached`)
```
checkRemoteAgentEligibility()  // 检查登录状态、GitHub 等
    ↓
teleportToRemote({
  initialMessage: prompt,
  permissionMode: 'plan',
  ultraplan: true,
  ...
})  // 创建远程会话
    ↓
registerRemoteAgentTask()  // 注册任务
    ↓
startDetachedPoll()  // 开始轮询
```

#### 3. 轮询流程 (`startDetachedPoll`)
```
pollForApprovedExitPlanMode()  // 轮询远程事件
    ↓
ExitPlanModeScanner.ingest()  // 解析事件流
    ↓
检测到计划完成:
  - executionTarget='remote' → 标记完成，通知用户
  - executionTarget='local'  → 设置 ultraplanPendingChoice，显示选择对话框
```

### 关键代码路径

#### Prompt 构建 (`buildUltraplanPrompt`)
```typescript
export function buildUltraplanPrompt(blurb: string, seedPlan?: string): string {
  const parts: string[] = [];
  if (seedPlan) {
    parts.push('Here is a draft plan to refine:', '', seedPlan, '');
  }
  parts.push(ULTRAPLAN_INSTRUCTIONS);  // 从 prompt.txt 加载
  if (blurb) {
    parts.push('', blurb);
  }
  return parts.join('\n');
}
```

- Prompt 包装在 `<system-reminder>` 中，CCR 浏览器会隐藏脚手架内容
- 故意避免使用 "ultraplan" 字样，防止远程 CCR 的 keyword detection 自触发

#### 模型选择 (`getUltraplanModel`)
```typescript
function getUltraplanModel(): string {
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_ultraplan_model', 
    ALL_MODEL_CONFIGS.opus46.firstParty  // 默认 claude-opus-4-6
  );
}
```

#### 会话创建参数
```typescript
const requestBody = {
  title: options.ultraplan ? `ultraplan: ${sessionTitle}` : sessionTitle,
  events: [
    // 1. set_permission_mode control_request (如果指定了 permissionMode)
    // 2. 用户初始消息
  ],
  session_context: {
    sources: [gitSource],  // GitHub 或 bundle
    outcomes: [gitOutcome],
    model: options.model ?? getMainLoopModel(),
  },
  environment_id: selectedEnvironment.environment_id
};
```

---

## 依赖与外部交互

### 内部依赖

| 模块 | 用途 |
|------|------|
| `../utils/teleport.tsx` | `teleportToRemote`, `archiveRemoteSession`, `pollRemoteSessionEvents` |
| `../utils/ultraplan/ccrSession.ts` | `pollForApprovedExitPlanMode`, `ExitPlanModeScanner`, `UltraplanPollError` |
| `../utils/ultraplan/keyword.ts` | 关键词触发检测（`hasUltraplanKeyword`） |
| `../tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `checkRemoteAgentEligibility`, `registerRemoteAgentTask`, `RemoteAgentTask` |
| `../state/AppStateStore.js` | `AppState` 类型定义 |
| `../utils/task/framework.js` | `updateTaskState` |
| `../utils/messageQueueManager.js` | `enqueuePendingNotification` |
| `../services/analytics/index.js` | `logEvent` |
| `../services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` |
| `../utils/model/configs.js` | `ALL_MODEL_CONFIGS` |
| `../constants/product.js` | `getRemoteSessionUrl` |
| `../constants/figures.js` | `DIAMOND_OPEN` |
| `../bridge/types.js` | `REMOTE_CONTROL_DISCONNECTED_MSG` |

### 外部 API 交互

1. **Sessions API** (`/v1/sessions`)
   - POST: 创建远程会话
   - 需要 OAuth token 和 organization UUID

2. **Session Events API** (`/v1/sessions/{id}/events`)
   - GET: 轮询会话事件流
   - 用于检测 ExitPlanMode 工具调用结果

3. **Archive API** (`/v1/sessions/{id}/archive`)
   - POST: 归档会话（停止远程执行）

### 配置与环境变量

| 变量 | 说明 |
|------|------|
| `ULTRAPLAN_PROMPT_FILE` | 开发环境覆盖 prompt.txt 的路径（仅 ant 构建） |
| `SESSION_INGRESS_URL` | 会话入口 URL |
| `tengu_ultraplan_model` | GrowthBook feature flag，用于覆盖默认模型 |

---

## 风险、边界与改进建议

### 已知风险

1. **OAuth Token 过期** (代码注释 TODO)
   - 30 分钟的轮询期间 OAuth token 可能过期
   - 当前未实现 token 刷新机制
   - **影响**: 轮询可能因 401 失败

2. **重复启动防护**
   - 依赖 `ultraplanSessionUrl` 和 `ultraplanLaunching` 状态
   - 竞态条件：快速连续调用可能绕过检查
   - **缓解**: `setAppState` 同步设置 `ultraplanLaunching` 标志

3. **网络错误处理**
   - `MAX_CONSECUTIVE_FAILURES = 5` 次网络错误后放弃轮询
   - 短暂网络中断可能导致任务失败

4. **Bundle 上传失败**
   - 大仓库可能超过 bundle 大小限制
   - 失败原因包括：`empty_repo`, `too_large`, `git_error`

### 边界情况

1. **任务取消时机**
   - 用户取消后，轮询的 `shouldStop` 回调会触发
   - 需要正确处理 `status !== 'running'` 的提前返回

2. **会话恢复**
   - `--resume` 时从 sidecar 恢复 `RemoteAgentMetadata`
   - 需要重新连接并继续轮询

3. **Plan 文本提取失败**
   - 如果远程返回的计划没有 `"## Approved Plan:"` 标记
   - 会抛出 `extract_marker_missing` 错误

### 改进建议

1. **Token 刷新机制**
   ```typescript
   // 在 pollForApprovedExitPlanMode 的循环中添加
   if (tokenExpiringSoon()) {
     await refreshOAuthToken();
   }
   ```

2. **ExitPlanModeScanner 集成**
   - 代码注释 TODO(#23985): 将 `ExitPlanModeScanner` 移入 `startRemoteSessionPolling`
   - 当前 ultraplan 和普通 remote agent 任务有两个并行的轮询逻辑

3. **更优雅的降级**
   - 当 bundle 失败且 GitHub 不可用时，提供更清晰的错误信息
   - 考虑支持本地计划模式作为 fallback

4. **进度可视化**
   - 当前仅显示 `running`/`needs_input`/`plan_ready` 三态
   - 可考虑从远程事件流中提取更多进度信息

5. **测试覆盖**
   - 需要更多针对 `ExitPlanModeScanner` 的单元测试
   - 轮询超时和网络错误的集成测试

---

## 文件引用汇总

| 文件路径 | 引用关系 |
|---------|---------|
| `src/commands/ultraplan.tsx` | 本文件 |
| `src/utils/teleport.tsx` | `teleportToRemote`, `archiveRemoteSession`, `pollRemoteSessionEvents` |
| `src/utils/ultraplan/ccrSession.ts` | `pollForApprovedExitPlanMode`, `UltraplanPollError` |
| `src/utils/ultraplan/keyword.ts` | 关键词触发检测 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 任务注册、资格检查 |
| `src/utils/ultraplan/prompt.txt` | 系统指令（构建时内联） |
| `src/utils/model/configs.ts` | 模型配置 |
| `src/constants/product.ts` | 会话 URL 生成 |
| `src/state/AppStateStore.ts` | AppState 类型 |
