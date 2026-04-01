# useInboxPoller.ts 深度研究文档

## 场景与职责

`useInboxPoller` 是 Claude Code 多智能体系统（Agent Swarm）中的核心消息轮询钩子，负责在队友（teammate）和团队领导（team lead）之间建立异步通信通道。它是整个团队协作架构的"邮局系统"，确保消息可靠传递。

### 核心场景

1. **队友间消息传递**：团队成员通过文件系统邮箱（mailbox）交换消息
2. **权限请求/响应协调**：工作节点向领导请求工具使用权限，领导审批后返回结果
3. **沙盒网络权限管理**：处理网络访问权限的跨进程协调
4. **计划审批流程**：队友进入计划模式时需获得领导批准
5. **团队关闭协调**：处理团队成员的优雅关闭流程

### 角色区分

- **Team Lead（领导）**：轮询自己的邮箱，处理来自队友的权限请求
- **Teammate（队友）**：轮询自己的邮箱，接收领导的消息和权限响应
- **In-process teammate**：在同进程内运行，使用 AsyncLocalStorage 上下文，不使用此轮询器

---

## 功能点目的

### 1. 消息轮询与分类（核心功能）

每 1 秒轮询一次邮箱文件，将消息按类型分类处理：

| 消息类型 | 处理方 | 用途 |
|---------|-------|------|
| `permission_request` | Leader | 工具使用权限请求 |
| `permission_response` | Teammate | 权限审批结果 |
| `sandbox_permission_request` | Leader | 沙盒网络访问请求 |
| `sandbox_permission_response` | Teammate | 网络权限审批结果 |
| `plan_approval_request` | Leader | 计划模式审批请求 |
| `plan_approval_response` | Teammate | 计划审批结果 |
| `shutdown_request` | Teammate | 关闭请求 |
| `shutdown_approved` | Leader | 关闭确认 |
| `team_permission_update` | Teammate | 权限规则更新 |
| `mode_set_request` | Teammate | 权限模式变更 |
| 普通消息 | 双方 | 常规通信 |

### 2. 权限请求处理（Leader 端）

当领导收到权限请求时：
- 将请求转换为 `ToolUseConfirm` 对象
- 通过 `getLeaderToolUseConfirmQueue()` 注入 REPL 的权限队列
- 用户审批后，通过 `sendPermissionResponseViaMailbox` 发送响应
- 支持桌面通知提醒

### 3. 权限响应处理（Teammate 端）

当队友收到权限响应时：
- 通过 `processMailboxPermissionResponse` 调用注册的回调
- 回调由 `useSwarmPermissionPoller` 中的 `registerPermissionCallback` 注册
- 触发 `onAllow` 或 `onReject` 继续执行

### 4. 沙盒权限协调

独立的沙盒权限流：
- 工作节点请求网络访问时发送 `sandbox_permission_request`
- 领导将请求加入 `workerSandboxPermissions` 队列
- 用户审批后发送 `sandbox_permission_response`
- 工作节点通过 `processSandboxPermissionResponse` 处理

### 5. 计划审批流程

- 队友进入计划模式前发送 `plan_approval_request`
- 领导自动批准（auto-approve）并发送 `plan_approval_response`
- 队友收到批准后退出计划模式，继承领导的权限模式

### 6. 团队关闭流程

- 领导发送 `shutdown_request` 给队友
- 队友处理后发送 `shutdown_approved`
- 领导收到后：
  - 关闭 tmux pane（如果是 pane-based 队友）
  - 从 team file 移除队友
  - 取消分配任务
  - 更新任务状态为 completed

### 7. 消息队列管理

- **空闲状态**：消息立即提交为新对话轮次
- **忙碌状态**：消息加入 `AppState.inbox` 队列，稍后投递
- **消息格式化**：使用 XML 标签 `<teammate_message>` 包装，包含颜色、摘要属性

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
interface Props {
  enabled: boolean                    // 是否启用轮询
  isLoading: boolean                  // 是否正在处理请求
  focusedInputDialog: string | undefined  // 是否有输入对话框聚焦
  onSubmitMessage: (formatted: string) => boolean  // 消息提交回调
}

// 轮询间隔常量
const INBOX_POLL_INTERVAL_MS = 1000  // 1秒轮询间隔
```

### 核心流程

#### 1. 获取轮询代理名称

```typescript
function getAgentNameToPoll(appState: AppState): string | undefined {
  // In-process teammates 不使用此轮询器（使用 waitForNextPromptOrShutdown）
  if (isInProcessTeammate()) return undefined
  
  // 普通队友使用 CLAUDE_CODE_AGENT_NAME
  if (isTeammate()) return getAgentName()
  
  // 领导从 teamContext 查找自己的名字
  if (isTeamLead(appState.teamContext)) {
    const leadName = appState.teamContext!.teammates[leadAgentId]?.name
    return leadName || 'team-lead'
  }
  return undefined
}
```

#### 2. 消息分类逻辑

```typescript
// 遍历未读消息，按类型分类
for (const m of unread) {
  const permReq = isPermissionRequest(m.text)
  const permResp = isPermissionResponse(m.text)
  const sandboxReq = isSandboxPermissionRequest(m.text)
  // ... 其他类型检查
  
  if (permReq) permissionRequests.push(m)
  else if (permResp) permissionResponses.push(m)
  // ... 其他分类
}
```

#### 3. 权限请求处理（Leader）

```typescript
if (permissionRequests.length > 0 && isTeamLead(currentAppState.teamContext)) {
  const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()
  
  for (const m of permissionRequests) {
    const entry: ToolUseConfirm = {
      assistantMessage: createAssistantMessage({ content: '' }),
      tool,
      description: parsed.description,
      input: parsed.input,
      toolUseID: parsed.tool_use_id,
      permissionResult: { behavior: 'ask', message: parsed.description },
      workerBadge: { name: parsed.agent_id, color: 'cyan' },
      onAllow(updatedInput, permissionUpdates) {
        void sendPermissionResponseViaMailbox(...)
      },
      onReject(feedback) {
        void sendPermissionResponseViaMailbox(...)
      },
      // ...
    }
    setToolUseConfirmQueue(queue => [...queue, entry])
  }
}
```

#### 4. 消息投递策略

```typescript
if (!isLoading && !focusedInputDialog) {
  // 空闲：立即提交
  const submitted = onSubmitTeammateMessage(formatted)
  if (!submitted) queueMessages()  // 提交失败则入队
} else {
  // 忙碌：加入队列
  queueMessages()
}
```

### 依赖的外部系统

| 依赖模块 | 用途 |
|---------|------|
| `teammateMailbox.ts` | 邮箱文件的读写操作 |
| `permissionSync.ts` | 权限请求/响应的创建和发送 |
| `leaderPermissionBridge.ts` | 与 REPL 权限队列的桥接 |
| `useSwarmPermissionPoller.ts` | 回调注册和处理 |
| `teamHelpers.ts` | 团队成员管理 |
| `teammate.ts` | 身份识别和团队上下文 |

---

## 关键代码路径与文件引用

### 入口与主循环

```
src/hooks/useInboxPoller.ts
├── getAgentNameToPoll()           # 行 81-105: 确定轮询身份
├── useInboxPoller()               # 行 126-969: 主钩子
│   ├── poll()                     # 行 139-873: 轮询回调
│   │   ├── readUnreadMessages()   # 读取未读消息
│   │   ├── 消息分类逻辑            # 行 216-248
│   │   ├── 权限请求处理            # 行 250-364
│   │   ├── 权限响应处理            # 行 366-397
│   │   ├── 沙盒权限处理            # 行 399-495
│   │   ├── 计划审批处理            # 行 599-662
│   │   └── 关闭流程处理            # 行 664-800
│   └── useEffect()                # 行 876-950: 空闲时投递队列消息
└── useInterval()                  # 行 954: 定时轮询
```

### 关键依赖文件

```
src/utils/teammateMailbox.ts       # 邮箱核心操作
├── readUnreadMessages()           # 行 115-125
├── writeToMailbox()               # 行 134-192
├── markMessagesAsRead()           # 行 279-342
└── 消息类型守卫函数                # 行 435-948

src/utils/swarm/permissionSync.ts  # 权限同步
├── sendPermissionResponseViaMailbox()  # 行 734-783
└── sendSandboxPermissionResponseViaMailbox()  # 行 882-927

src/utils/swarm/leaderPermissionBridge.ts  # 领导桥接
├── getLeaderToolUseConfirmQueue()  # 行 34-36

src/hooks/useSwarmPermissionPoller.ts  # 权限轮询
├── processMailboxPermissionResponse()   # 行 124-156
├── processSandboxPermissionResponse()   # 行 201-226
└── registerPermissionCallback()         # 行 82-89

src/utils/swarm/teamHelpers.ts     # 团队管理
├── removeTeammateFromTeamFile()   # 行 188-227
└── setMemberMode()                # 行 357-389

src/utils/inProcessTeammateHelpers.ts  # 进程内队友
├── findInProcessTeammateTaskId()  # 行 33-46
└── handlePlanApprovalResponse()   # 行 77-83
```

---

## 依赖与外部交互

### 与 AppState 的交互

```typescript
// 读取状态
const store = useAppStateStore()
const currentAppState = store.getState()

// 更新状态
setAppState(prev => ({
  ...prev,
  inbox: { messages: [...] },
  workerSandboxPermissions: { queue: [...] },
  teamContext: { teammates: {...} }
}))
```

### 与文件系统的交互

邮箱文件位置：`~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

使用文件锁（lockfile）防止并发写入冲突：
```typescript
const lockFilePath = `${inboxPath}.lock`
release = await lockfile.lock(inboxPath, { lockfilePath, ...LOCK_OPTIONS })
```

### 与通知系统的交互

```typescript
import { sendNotification } from '../services/notifier.js'

void sendNotification({
  message: `${firstParsed.agent_id} needs permission for ${firstParsed.tool_name}`,
  notificationType: 'worker_permission_prompt',
}, terminal)
```

### 与工具权限系统的交互

通过 `ToolUseConfirm` 对象与 REPL 的权限对话框集成，支持：
- `BashPermissionRequest`
- `FileEditToolDiff`
- 其他工具特定的权限 UI

---

## 风险、边界与改进建议

### 已知风险

1. **消息丢失风险**
   - 当前实现：消息标记为已读后才从邮箱移除
   - 风险：如果标记为已读后、处理前崩溃，消息可能丢失
   - 缓解：消息先加入 AppState 队列，再标记已读

2. **并发冲突**
   - 使用文件锁防止并发写入，但锁超时可能导致冲突
   - 重试机制：10 次重试，5-100ms 退避

3. **身份混淆**
   - In-process teammates 不使用此轮询器，依赖 AsyncLocalStorage
   - 如果上下文切换不当，可能导致消息路由错误

4. **性能问题**
   - 每 1 秒轮询一次，大量队友时可能产生文件系统压力
   - 消息分类是 O(n) 操作，大量消息时可能阻塞

### 边界情况

| 场景 | 行为 |
|-----|------|
| 邮箱文件不存在 | 返回空数组，正常处理 |
| 消息格式损坏 | 作为普通消息处理 |
| 权限回调未注册 | 记录警告，消息丢弃 |
| 团队上下文缺失 | 跳过处理，等待下次轮询 |
| 同时收到多条消息 | 批量处理，按类型分类 |

### 改进建议

1. **事件驱动替代轮询**
   - 使用文件系统监视（fs.watch）替代定时轮询
   - 减少不必要的文件系统操作

2. **消息持久化**
   - 引入 WAL（Write-Ahead Log）确保消息不丢失
   - 处理完成后再确认消费

3. **批量处理优化**
   - 限制单次处理消息数量，避免阻塞
   - 大消息量时采用流式处理

4. **身份验证增强**
   - 消息签名验证，防止伪造
   - 特别是权限相关消息需要严格验证来源

5. **监控与可观测性**
   - 添加消息处理延迟指标
   - 监控邮箱文件大小，防止无限增长

6. **优雅降级**
   - 文件系统不可用时切换到内存队列
   - 网络模式下使用 WebSocket 替代文件邮箱

### 测试建议

1. **单元测试**：模拟各种消息类型的处理
2. **集成测试**：多进程队友间的消息传递
3. **压力测试**：大量消息并发处理
4. **故障注入**：文件锁超时、磁盘满等异常情况
