# ExitPlanModeV2Tool.ts 深度研究文档

## 文件元数据
- **路径**: `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
- **大小**: 17,006 bytes
- **类型**: TypeScript 工具实现
- **所属模块**: ExitPlanModeTool

---

## 1. 场景与职责

### 1.1 核心定位
`ExitPlanModeV2Tool` 是 Claude Code 中**计划模式（Plan Mode）**的核心退出工具，负责：

1. **计划审批流程**: 当用户完成计划编写后，通过此工具提交计划供审批
2. **权限模式切换**: 从 `plan` 模式切换回之前的模式（`default` 或 `auto`）
3. **团队协作支持**: 支持 teammate（子代理）向 team lead 提交计划审批请求
4. **计划持久化**: 将编辑后的计划内容写入磁盘

### 1.2 使用场景

| 场景 | 描述 |
|------|------|
| **单人模式** | 用户完成计划编写，调用此工具退出计划模式，开始编码 |
| **Teammate 计划审批** | 作为 teammate 的子代理需要 leader 审批计划后才能继续 |
| **自动模式恢复** | 退出计划模式时恢复之前的 auto 模式状态 |
| **权限恢复** | 恢复进入计划模式前被剥离的危险权限 |

---

## 2. 功能点目的

### 2.1 主要功能

#### 2.1.1 计划审批与退出
- **目的**: 允许用户在完成计划后退出计划模式
- **触发条件**: 仅在 `plan` 模式下可用
- **输出**: 向用户展示计划内容，等待审批

#### 2.1.2 Teammate 计划提交流程
- **目的**: 支持分布式团队中子代理的计划审批
- **流程**:
  1. Teammate 调用 ExitPlanMode
  2. 生成 `plan_approval_request` 消息
  3. 通过 mailbox 发送给 team-lead
  4. 设置 `awaitingLeaderApproval` 状态
  5. 等待 leader 的 `plan_approval_response`

#### 2.1.3 权限模式管理
- **目的**: 安全地切换权限模式
- **功能**:
  - 恢复到进入计划模式前的模式（`prePlanMode`）
  - 处理 auto 模式的断路器逻辑
  - 恢复/剥离危险权限

#### 2.1.4 计划编辑支持
- **目的**: 支持通过 CCR Web UI 或 Ctrl+G 编辑计划
- **实现**: 检测 `input.plan` 字段，如有编辑则写回磁盘

### 2.2 功能开关

| Feature Flag | 用途 |
|-------------|------|
| `TRANSCRIPT_CLASSIFIER` | 启用 auto 模式相关功能 |
| `KAIROS` / `KAIROS_CHANNELS` | 在 channel 模式下禁用此工具 |

---

## 3. 具体技术实现

### 3.1 数据结构

#### 3.1.1 输入 Schema
```typescript
const inputSchema = z.strictObject({
  allowedPrompts: z.array(allowedPromptSchema()).optional()
    // Prompt-based permissions needed to implement the plan
})

type AllowedPrompt = {
  tool: 'Bash'  // 目前仅支持 Bash
  prompt: string  // 语义描述，如 "run tests", "install dependencies"
}
```

#### 3.1.2 输出 Schema
```typescript
const outputSchema = z.object({
  plan: z.string().nullable(),           // 展示给用户的计划内容
  isAgent: z.boolean(),                  // 是否为子代理
  filePath: z.string().optional(),       // 计划文件路径
  hasTaskTool: z.boolean().optional(),   // 是否有 Agent 工具可用
  planWasEdited: z.boolean().optional(), // 用户是否编辑了计划
  awaitingLeaderApproval: z.boolean().optional(), // 是否等待 leader 审批
  requestId: z.string().optional(),      // 审批请求 ID
})
```

### 3.2 关键流程

#### 3.2.1 主调用流程 (`call` 方法)

```
call(input, context)
├── 1. 获取计划文件路径
│   └── getPlanFilePath(context.agentId)
├── 2. 获取计划内容
│   ├── 优先使用 input.plan（CCR UI 编辑）
│   └── 否则从磁盘读取 getPlan()
├── 3. 如有编辑，写回磁盘
│   ├── writeFile(filePath, inputPlan)
│   └── persistFileSnapshotIfRemote()
├── 4. 检查 teammate 审批流程
│   ├── 如果是 teammate 且需要计划模式
│   │   ├── 生成 requestId
│   │   ├── 创建 plan_approval_request
│   │   ├── writeToMailbox('team-lead', approvalRequest)
│   │   ├── 更新 task 状态 setAwaitingPlanApproval()
│   │   └── 返回 awaitingLeaderApproval=true
│   └── 否则继续正常流程
├── 5. 处理权限模式切换
│   ├── 检查 auto mode gate
│   ├── setHasExitedPlanMode(true)
│   ├── setNeedsPlanModeExitAttachment(true)
│   ├── 恢复 prePlanMode
│   ├── 处理 auto mode 状态
│   └── 恢复/剥离危险权限
└── 6. 返回结果
    └── { plan, isAgent, filePath, hasTaskTool, planWasEdited }
```

#### 3.2.2 权限检查流程

```typescript
// 对于 teammate：绕过权限 UI，直接允许
if (isTeammate()) {
  return { behavior: 'allow', updatedInput: input }
}

// 对于普通用户：需要用户确认
return {
  behavior: 'ask',
  message: 'Exit plan mode?',
  updatedInput: input,
}
```

#### 3.2.3 输入验证流程

```typescript
// Teammate 跳过验证（leader 可能处于不同模式）
if (isTeammate()) {
  return { result: true }
}

// 非 plan 模式下拒绝
if (mode !== 'plan') {
  logEvent('tengu_exit_plan_mode_called_outside_plan', {...})
  return {
    result: false,
    message: 'You are not in plan mode...',
    errorCode: 1,
  }
}
```

### 3.3 关键算法

#### 3.3.1 Auto Mode Gate 回退逻辑
```typescript
// 如果 prePlanMode 是 auto 但 gate 已关闭，回退到 default
if (prePlanRaw === 'auto' && !isAutoModeGateEnabled()) {
  const reason = getAutoModeUnavailableReason() ?? 'circuit-breaker'
  gateFallbackNotification = getAutoModeUnavailableNotification(reason)
  restoreMode = 'default'
}
```

#### 3.3.2 危险权限处理
```typescript
// 恢复到 auto 模式：剥离危险权限
if (restoringToAuto) {
  baseContext = stripDangerousPermissionsForAutoMode(baseContext)
}
// 恢复到非 auto 模式：恢复之前剥离的权限
else if (prev.toolPermissionContext.strippedDangerousRules) {
  baseContext = restoreDangerousPermissions(baseContext)
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `../../bootstrap/state.js` | 会话状态管理（hasExitedPlanMode, setNeedsPlanModeExitAttachment 等） |
| `../../Tool.js` | Tool 类型定义和 buildTool 工厂函数 |
| `../../utils/plans.js` | 计划文件操作（getPlan, getPlanFilePath, persistFileSnapshotIfRemote） |
| `../../utils/teammate.js` | Teammate 身份检测（isTeammate, isPlanModeRequired, getAgentName, getTeamName） |
| `../../utils/teammateMailbox.js` | Mailbox 消息写入（writeToMailbox） |
| `../../utils/inProcessTeammateHelpers.js` |  teammate 任务状态管理（findInProcessTeammateTaskId, setAwaitingPlanApproval） |
| `../../utils/permissions/autoModeState.js` | Auto 模式状态管理（setAutoModeActive, isAutoModeActive） |
| `../../utils/permissions/permissionSetup.js` | 权限剥离/恢复（stripDangerousPermissionsForAutoMode, restoreDangerousPermissions, isAutoModeGateEnabled） |
| `../../utils/agentId.js` | Agent ID 格式化（formatAgentId, generateRequestId） |
| `./UI.js` | UI 渲染函数（renderToolUseMessage, renderToolResultMessage, renderToolUseRejectedMessage） |
| `./constants.js` | 工具名称常量 |
| `./prompt.js` | 工具提示词 |

### 4.2 外部调用方

| 文件 | 调用方式 |
|------|---------|
| `src/tools.ts` | 注册到工具列表 `getAllBaseTools()` |
| `src/utils/plans.ts` | 引用 `EXIT_PLAN_MODE_V2_TOOL_NAME` 用于计划恢复 |
| `src/screens/REPL.tsx` | 通过工具系统调用 |
| `src/components/messages/PlanApprovalMessage.tsx` | 处理 plan approval 消息展示 |

### 4.3 关键代码片段

#### 4.3.1 Teammate 计划提交流程
```typescript
// ExitPlanModeV2Tool.ts:263-312
if (isTeammate() && isPlanModeRequired()) {
  if (!plan) {
    throw new Error(`No plan file found at ${filePath}...`)
  }
  const agentName = getAgentName() || 'unknown'
  const teamName = getTeamName()
  const requestId = generateRequestId('plan_approval', formatAgentId(agentName, teamName || 'default'))

  const approvalRequest = {
    type: 'plan_approval_request',
    from: agentName,
    timestamp: new Date().toISOString(),
    planFilePath: filePath,
    planContent: plan,
    requestId,
  }

  await writeToMailbox('team-lead', {...}, teamName)

  // 更新 task 状态
  const agentTaskId = findInProcessTeammateTaskId(agentName, appState)
  if (agentTaskId) {
    setAwaitingPlanApproval(agentTaskId, context.setAppState, true)
  }

  return { data: { plan, isAgent: true, filePath, awaitingLeaderApproval: true, requestId } }
}
```

#### 4.3.2 权限模式切换
```typescript
// ExitPlanModeV2Tool.ts:357-403
context.setAppState(prev => {
  if (prev.toolPermissionContext.mode !== 'plan') return prev
  setHasExitedPlanMode(true)
  setNeedsPlanModeExitAttachment(true)
  
  let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
  
  // Auto mode gate 检查
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
      restoreMode = 'default'
    }
    // 设置 auto mode 状态
    const autoWasUsedDuringPlan = autoModeStateModule?.isAutoModeActive() ?? false
    autoModeStateModule?.setAutoModeActive(finalRestoringAuto)
    if (autoWasUsedDuringPlan && !finalRestoringAuto) {
      setNeedsAutoModeExitAttachment(true)
    }
  }
  
  // 权限剥离/恢复
  const restoringToAuto = restoreMode === 'auto'
  let baseContext = prev.toolPermissionContext
  if (restoringToAuto) {
    baseContext = stripDangerousPermissionsForAutoMode(baseContext)
  } else if (prev.toolPermissionContext.strippedDangerousRules) {
    baseContext = restoreDangerousPermissions(baseContext)
  }
  
  return {
    ...prev,
    toolPermissionContext: {
      ...baseContext,
      mode: restoreMode,
      prePlanMode: undefined,
    },
  }
})
```

---

## 5. 依赖与外部交互

### 5.1 与 Teammate 系统的交互

```
ExitPlanModeV2Tool
├── 检测: isTeammate() + isPlanModeRequired()
├── 生成: PlanApprovalRequestMessage
├── 发送: writeToMailbox('team-lead', message)
└── 状态: setAwaitingPlanApproval(taskId, setAppState, true)

TeammateMailbox (team-lead 端)
├── 接收: plan_approval_request
├── 展示: PlanApprovalRequestDisplay 组件
├── 用户决策: 批准/拒绝
└── 回复: plan_approval_response

InProcessTeammate (teammate 端)
├── 接收: plan_approval_response
├── 处理: handlePlanApprovalResponse()
└── 继续: 开始实施计划
```

### 5.2 与权限系统的交互

```
ExitPlanModeV2Tool
├── 进入时: prepareContextForPlanMode() (EnterPlanModeTool 中)
│   └── 保存 prePlanMode
│   └── 剥离危险权限（如果进入前是 auto）
│
└── 退出时:
    ├── 读取 prePlanMode
    ├── 检查 auto mode gate
    ├── 恢复/剥离权限
    └── 清除 prePlanMode
```

### 5.3 与计划文件系统的交互

```
ExitPlanModeV2Tool
├── getPlanFilePath(agentId) → 获取文件路径
├── getPlan(agentId) → 读取计划内容
├── writeFile(filePath, plan) → 写入编辑后的计划
└── persistFileSnapshotIfRemote() → 远程会话快照
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 模式状态不一致风险
- **风险**: `prePlanMode` 可能因异常退出而丢失
- **缓解**: 工具检查当前模式，非 plan 模式时拒绝执行
- **代码**: `validateInput` 中检查 `mode !== 'plan'`

#### 6.1.2 Teammate 无限等待风险
- **风险**: teammate 提交计划后，如果 leader 不响应会无限等待
- **现状**: 需要 teammate 主动检查 inbox
- **改进**: 添加超时机制或心跳检测

#### 6.1.3 Auto Mode Gate 竞态条件
- **风险**: gate 状态可能在检查和切换之间改变
- **缓解**: 使用缓存值，但可能导致短暂不一致
- **代码**: `isAutoModeGateEnabled()` 使用缓存

### 6.2 边界情况

| 边界情况 | 处理逻辑 |
|---------|---------|
| 空计划 | 允许退出，但会提示用户 |
| 计划文件不存在 | Teammate 模式下抛出错误 |
| CCR UI 编辑计划 | 检测 `input.plan` 并写回磁盘 |
| 非 plan 模式调用 | 拒绝并记录分析事件 |
| Channel 模式 | 工具被禁用（避免审批对话框挂起） |
| 子代理调用 | 禁止（agentId 检查） |

### 6.3 改进建议

#### 6.3.1 架构改进
1. **提取状态机**: 将 plan mode 状态机提取到独立模块
2. **统一权限切换**: 与 `transitionPermissionMode` 进一步整合
3. **异步审批超时**: 为 teammate 审批添加超时机制

#### 6.3.2 代码质量
1. **减少条件导入**: `autoModeStateModule` 和 `permissionSetupModule` 的条件导入增加复杂性
2. **类型安全**: `inputPlan` 的类型检查可以更严格
3. **测试覆盖**: 缺少单元测试，特别是 teammate 流程

#### 6.3.3 功能增强
1. **计划版本控制**: 支持计划的历史版本
2. **审批评论**: 支持 leader 在批准时添加评论
3. **批量审批**: 支持一次审批多个 teammate 的计划

### 6.4 监控与调试

| 分析事件 | 触发条件 |
|---------|---------|
| `tengu_exit_plan_mode_called_outside_plan` | 非 plan 模式下调用 |

| 调试日志 | 用途 |
|---------|------|
| `[auto-mode gate @ ExitPlanModeV2Tool]` | Auto mode gate 回退通知 |

---

## 7. 相关文件索引

### 7.1 同目录文件
- `constants.ts` - 工具名称常量
- `prompt.ts` - 工具提示词
- `UI.tsx` - UI 渲染组件

### 7.2 核心依赖
- `src/Tool.ts` - Tool 基础类型
- `src/tools.ts` - 工具注册
- `src/bootstrap/state.ts` - 全局状态
- `src/utils/plans.ts` - 计划文件操作
- `src/utils/teammate.ts` - Teammate 身份
- `src/utils/teammateMailbox.ts` - Mailbox 消息
- `src/utils/inProcessTeammateHelpers.ts` - Teammate 任务状态
- `src/utils/permissions/permissionSetup.ts` - 权限管理
- `src/utils/permissions/autoModeState.ts` - Auto 模式状态
- `src/utils/agentId.ts` - Agent ID 生成

### 7.3 相关组件
- `src/components/messages/PlanApprovalMessage.tsx` - 计划审批消息展示
- `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` - 进入计划模式工具
