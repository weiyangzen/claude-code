# teammateMailbox.ts 研究文档

## 场景与职责

`teammateMailbox.ts` 是 Claude Code 多智能体协作系统的**文件基础消息传递基础设施**。它为 agent swarm 中的 teammate 之间提供可靠的异步通信机制，支持消息、权限请求、计划审批、关闭协调等多种协议消息类型。

### 核心职责

1. **邮箱文件管理**：每个 teammate 拥有独立的 inbox 文件（`~/.claude/teams/{team}/inboxes/{agent}.json`）
2. **并发安全写入**：使用文件锁（proper-lockfile）防止多进程并发写入冲突
3. **协议消息支持**：定义和实现多种结构化消息类型（权限、计划审批、关闭等）
4. **消息生命周期管理**：支持消息的读取、标记已读、清除等操作

### 架构定位

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Team Lead     │◄───►│  Mailbox System  │◄───►│   Teammate A    │
│  (useInboxPoller)│     │  (File-based)    │     │ (useInboxPoller)│
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               ▲                          ▲
                               │                          │
                        ┌──────┴──────┐          ┌────────┴────────┐
                        │ ~/.claude/  │          │ ~/.claude/      │
                        │ teams/{team}│          │ teams/{team}    │
                        │ /inboxes/   │          │ /inboxes/       │
                        │ lead.json   │          │ teammateA.json  │
                        └─────────────┘          └─────────────────┘
```

## 功能点目的

### 1. 基础消息操作

| 功能 | 函数 | 说明 |
|------|------|------|
| 读取邮箱 | `readMailbox()` | 读取指定 agent 的所有消息 |
| 读取未读 | `readUnreadMessages()` | 只返回未读消息 |
| 写入消息 | `writeToMailbox()` | 向指定 agent 的邮箱写入消息 |
| 标记已读 | `markMessageAsReadByIndex()` | 按索引标记单条消息已读 |
| 全部已读 | `markMessagesAsRead()` | 标记所有消息已读 |
| 清除邮箱 | `clearMailbox()` | 清空所有消息 |

### 2. 协议消息类型

#### 2.1 空闲通知（Idle Notification）
```typescript
type IdleNotificationMessage = {
  type: 'idle_notification'
  from: string
  timestamp: string
  idleReason?: 'available' | 'interrupted' | 'failed'
  summary?: string                    // 最后一条 DM 摘要
  completedTaskId?: string
  completedStatus?: 'resolved' | 'blocked' | 'failed'
  failureReason?: string
}
```

#### 2.2 权限请求/响应（Permission Request/Response）
```typescript
type PermissionRequestMessage = {
  type: 'permission_request'
  request_id: string
  agent_id: string
  tool_name: string
  tool_use_id: string
  description: string
  input: Record<string, unknown>
  permission_suggestions: unknown[]
}

type PermissionResponseMessage = 
  | { type: 'permission_response', request_id: string, subtype: 'success', response?: {...} }
  | { type: 'permission_response', request_id: string, subtype: 'error', error: string }
```

#### 2.3 沙箱权限请求/响应（Sandbox Permission）
```typescript
type SandboxPermissionRequestMessage = {
  type: 'sandbox_permission_request'
  requestId: string
  workerId: string
  workerName: string
  workerColor?: string
  hostPattern: { host: string }
  createdAt: number
}
```

#### 2.4 计划审批请求/响应（Plan Approval）
```typescript
const PlanApprovalRequestMessageSchema = z.object({
  type: z.literal('plan_approval_request'),
  from: z.string(),
  timestamp: z.string(),
  planFilePath: z.string(),
  planContent: z.string(),
  requestId: z.string(),
})

const PlanApprovalResponseMessageSchema = z.object({
  type: z.literal('plan_approval_response'),
  requestId: z.string(),
  approved: z.boolean(),
  feedback: z.string().optional(),
  timestamp: z.string(),
  permissionMode: PermissionModeSchema().optional(),
})
```

#### 2.5 关闭协调（Shutdown Coordination）
```typescript
type ShutdownRequestMessage = { type: 'shutdown_request', requestId: string, from: string, reason?: string }
type ShutdownApprovedMessage = { type: 'shutdown_approved', requestId: string, from: string, paneId?: string, backendType?: string }
type ShutdownRejectedMessage = { type: 'shutdown_rejected', requestId: string, from: string, reason: string }
```

#### 2.6 任务分配（Task Assignment）
```typescript
type TaskAssignmentMessage = {
  type: 'task_assignment'
  taskId: string
  subject: string
  description: string
  assignedBy: string
  timestamp: string
}
```

#### 2.7 团队权限更新（Team Permission Update）
```typescript
type TeamPermissionUpdateMessage = {
  type: 'team_permission_update'
  permissionUpdate: { type: 'addRules', rules: Array<{...}>, behavior: 'allow' | 'deny' | 'ask', destination: 'session' }
  directoryPath: string
  toolName: string
}
```

#### 2.8 模式设置请求（Mode Set Request）
```typescript
const ModeSetRequestMessageSchema = z.object({
  type: z.literal('mode_set_request'),
  mode: PermissionModeSchema(),  // 来自 SDK
  from: z.string(),
})
```

### 3. 结构化协议消息检测

`isStructuredProtocolMessage()` 函数用于区分普通消息和需要特殊路由的协议消息：

```typescript
export function isStructuredProtocolMessage(messageText: string): boolean {
  const parsed = jsonParse(messageText)
  const type = parsed.type
  return (
    type === 'permission_request' ||
    type === 'permission_response' ||
    type === 'sandbox_permission_request' ||
    type === 'sandbox_permission_response' ||
    type === 'shutdown_request' ||
    type === 'shutdown_approved' ||
    type === 'team_permission_update' ||
    type === 'mode_set_request' ||
    type === 'plan_approval_request' ||
    type === 'plan_approval_response'
  )
}
```

### 4. 最后一条点对点消息摘要

`getLastPeerDmSummary()` 从消息历史中提取最后一条发送给 peer 的 SendMessage 工具调用摘要：

```typescript
export function getLastPeerDmSummary(messages: Message[]): string | undefined {
  // 从后向前遍历消息
  // 在唤醒边界（user prompt with string content）处停止
  // 查找 SendMessage 工具调用，排除发送给 team-lead 的消息
  // 返回格式: "[to {name}] {summary}"
}
```

## 具体技术实现

### 文件锁配置

```typescript
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

### 邮箱路径生成

```typescript
export function getInboxPath(agentName: string, teamName?: string): string {
  const team = teamName || getTeamName() || 'default'
  const safeTeam = sanitizePathComponent(team)
  const safeAgentName = sanitizePathComponent(agentName)
  const inboxDir = join(getTeamsDir(), safeTeam, 'inboxes')
  return join(inboxDir, `${safeAgentName}.json`)
}
```

### 并发安全写入流程

```typescript
export async function writeToMailbox(
  recipientName: string,
  message: Omit<TeammateMessage, 'read'>,
  teamName?: string,
): Promise<void> {
  await ensureInboxDir(teamName)
  const inboxPath = getInboxPath(recipientName, teamName)
  const lockFilePath = `${inboxPath}.lock`
  
  // 1. 确保文件存在（proper-lockfile 需要目标文件存在）
  try {
    await writeFile(inboxPath, '[]', { encoding: 'utf-8', flag: 'wx' })
  } catch (error) {
    if (code !== 'EEXIST') { /* 处理错误 */ }
  }
  
  let release: (() => Promise<void>) | undefined
  try {
    // 2. 获取文件锁
    release = await lockfile.lock(inboxPath, { lockfilePath: lockFilePath, ...LOCK_OPTIONS })
    
    // 3. 获取锁后重新读取（获取最新状态）
    const messages = await readMailbox(recipientName, teamName)
    messages.push({ ...message, read: false })
    
    // 4. 写入更新后的消息
    await writeFile(inboxPath, jsonStringify(messages, null, 2), 'utf-8')
  } finally {
    // 5. 释放锁
    if (release) await release()
  }
}
```

### 消息格式化（XML 输出）

```typescript
export function formatTeammateMessages(messages: Array<{...}>): string {
  return messages
    .map(m => {
      const colorAttr = m.color ? ` color="${m.color}"` : ''
      const summaryAttr = m.summary ? ` summary="${m.summary}"` : ''
      return `<${TEAMMATE_MESSAGE_TAG} teammate_id="${m.from}"${colorAttr}${summaryAttr}>\n${m.text}\n</${TEAMMATE_MESSAGE_TAG}>`
    })
    .join('\n\n')
}
```

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `lockfile.ts` | 延迟加载 proper-lockfile，提供文件锁功能 |
| `envUtils.ts` | `getTeamsDir()` 获取团队目录路径 |
| `tasks.ts` | `sanitizePathComponent()` 路径组件清理 |
| `teammate.ts` | `getTeamName()`, `getAgentName()`, `getTeammateColor()` |
| `slowOperations.ts` | `jsonParse()`, `jsonStringify()` 带性能监控的 JSON 操作 |
| `agentId.ts` | `generateRequestId()` 生成请求 ID |
| `debug.ts` | `logForDebugging()` 调试日志 |
| `errors.ts` | `getErrnoCode()` 错误码提取 |

### 外部调用方

| 文件 | 调用目的 |
|------|----------|
| `useInboxPoller.ts` | 读取未读消息、标记已读、处理协议消息 |
| `permissionSync.ts` | 创建/解析权限请求和响应消息 |
| `leaderPermissionBridge.ts` | 发送权限响应 |
| `inProcessRunner.ts` | 创建空闲通知消息 |
| `TeamDeleteTool.ts` | 发送关闭请求 |
| `useTeammateShutdownNotification.ts` | 处理关闭消息 |
| `sendShutdownRequestToMailbox()` | 发送关闭请求 |

### 消息类型检测函数

```typescript
isIdleNotification(messageText): IdleNotificationMessage | null
isPermissionRequest(messageText): PermissionRequestMessage | null
isPermissionResponse(messageText): PermissionResponseMessage | null
isSandboxPermissionRequest(messageText): SandboxPermissionRequestMessage | null
isSandboxPermissionResponse(messageText): SandboxPermissionResponseMessage | null
isShutdownRequest(messageText): ShutdownRequestMessage | null
isShutdownApproved(messageText): ShutdownApprovedMessage | null
isShutdownRejected(messageText): ShutdownRejectedMessage | null
isPlanApprovalRequest(messageText): PlanApprovalRequestMessage | null
isPlanApprovalResponse(messageText): PlanApprovalResponseMessage | null
isTaskAssignment(messageText): TaskAssignmentMessage | null
isTeamPermissionUpdate(messageText): TeamPermissionUpdateMessage | null
isModeSetRequest(messageText): ModeSetRequestMessage | null
isStructuredProtocolMessage(messageText): boolean
```

## 依赖与外部交互

### 文件系统布局

```
~/.claude/
└── teams/
    └── {team_name}/
        ├── inboxes/
        │   ├── team-lead.json
        │   ├── researcher.json
        │   ├── tester.json
        │   └── ...
        ├── team.json          # 团队配置
        └── permissions/       # 权限请求存储
            ├── pending/
            └── resolved/
```

### 与 useInboxPoller 的交互

```typescript
// useInboxPoller.ts 中的消息处理流程
const unread = await readUnreadMessages(agentName, teamName)
for (const message of unread) {
  if (isPermissionRequest(message.text)) {
    // 路由到权限处理队列
  } else if (isShutdownRequest(message.text)) {
    // 处理关闭请求
  } else if (isPlanApprovalResponse(message.text)) {
    // 处理计划审批响应
  } else {
    // 作为普通消息提交给 LLM
  }
}
```

### 与权限系统的交互

权限请求通过 mailbox 在 worker 和 leader 之间传递：

1. **Worker** 遇到权限提示 → 创建 `PermissionRequestMessage` → 写入 leader 的 mailbox
2. **Leader** 的 `useInboxPoller` 检测到请求 → 显示权限对话框
3. **Leader** 用户批准/拒绝 → 创建 `PermissionResponseMessage` → 写入 worker 的 mailbox
4. **Worker** 的 `useInboxPoller` 检测到响应 → 继续执行

## 风险、边界与改进建议

### 已知风险

1. **文件锁性能**：高并发场景下（10+ teammate），文件锁竞争可能导致延迟
2. **文件系统限制**：依赖文件系统的原子操作，某些网络文件系统可能不支持
3. **消息丢失**：如果进程崩溃时消息未写入磁盘，可能导致消息丢失
4. **消息堆积**：长时间运行的 teammate 可能积累大量未读消息

### 边界情况

1. **空消息处理**：`isStructuredProtocolMessage` 对非 JSON 消息返回 false，但 `jsonParse` 可能抛出异常
2. **并发写入**：两个进程同时写入同一 mailbox 时，文件锁确保顺序执行，但后写入者可能覆盖前者
3. **磁盘空间**：未清理的 mailbox 文件可能占用大量磁盘空间
4. **跨团队消息**：当前实现假设消息在同一团队内传递，跨团队消息需要显式 teamName

### 改进建议

1. **消息压缩**：对于大量消息的场景，考虑使用压缩或分页机制

2. **消息过期**：添加消息 TTL 机制，自动清理过期消息

3. **批量操作**：当前 `markMessagesAsRead` 需要重新写入整个文件，对于大文件效率低

4. **索引优化**：对于频繁访问的 mailbox，考虑维护内存索引

5. **错误恢复**：增强文件损坏时的恢复机制（如备份文件、CRC 校验）

6. **监控指标**：
   - 邮箱文件大小分布
   - 文件锁等待时间
   - 消息读写延迟
   - 协议消息类型分布

7. **协议版本控制**：为未来协议变更添加版本号字段

8. **消息确认机制**：当前是"最多一次"传递，考虑添加确认机制实现"至少一次"
