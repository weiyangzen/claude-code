# SendMessageTool.ts 研究文档

## 场景与职责

SendMessageTool 是 Claude Code 中 Agent Swarm（代理集群）系统的核心通信工具，负责实现团队成员之间的消息传递功能。它是多代理协作架构中的关键组件，支持以下场景：

1. **团队内部通信**： teammates（团队成员）之间发送点对点消息
2. **广播消息**：team-lead 向所有团队成员广播消息
3. **结构化协议消息**：支持 shutdown_request/shutdown_response、plan_approval_response 等协议消息
4. **跨会话通信**：通过 UDS（Unix Domain Socket）和 Bridge 机制实现跨机器/跨会话的消息传递
5. **In-Process 队友管理**：自动恢复已停止的 in-process 队友并传递消息

该工具是 Agent Swarm 功能的核心依赖，只有在 `isAgentSwarmsEnabled()` 返回 true 时才会启用。

## 功能点目的

### 1. 消息路由与分发
- **点对点消息**：通过 `handleMessage` 将消息写入指定队友的 mailbox
- **广播消息**：通过 `handleBroadcast` 向团队所有成员发送消息（排除发送者自身）
- **跨会话消息**：支持 `uds:` 和 `bridge:` 地址格式的跨会话通信

### 2. 结构化消息协议
支持三种结构化消息类型：
- `shutdown_request` / `shutdown_response`：团队成员关闭请求/响应协议
- `plan_approval_response`：计划审批响应（approve/reject）

### 3. In-Process 队友生命周期管理
- 检测目标 agent 是否为 in-process teammate
- 自动恢复已停止的 in-process 队友
- 消息队列管理（对于运行中的 agent，消息会在下一个 tool round 时传递）

### 4. 权限与安全检查
- `checkPermissions`：对 bridge 地址的消息发送需要用户显式确认
- `validateInput`：验证输入参数，包括地址格式、消息类型限制等

## 具体技术实现

### 关键数据结构

```typescript
// 输入 Schema
{
  to: string;        // 接收者：队友名称、"*" 广播、uds:/path 或 bridge:session-id
  summary?: string;  // 5-10 字摘要（纯文本消息必需）
  message: string | StructuredMessage;  // 消息内容
}

// 结构化消息联合类型
StructuredMessage = 
  | { type: 'shutdown_request', reason?: string }
  | { type: 'shutdown_response', request_id: string, approve: boolean, reason?: string }
  | { type: 'plan_approval_response', request_id: string, approve: boolean, feedback?: string }

// 输出类型
SendMessageToolOutput = MessageOutput | BroadcastOutput | RequestOutput | ResponseOutput
```

### 核心流程

#### 1. 工具调用入口 (`call` 方法)
```
call(input, context, canUseTool, assistantMessage)
  ├── UDS/Bridge 地址处理 (feature UDS_INBOX)
  │     ├── bridge:scheme → postInterClaudeMessage()
  │     └── uds:scheme → sendToUdsSocket()
  ├── In-Process Teammate 路由
  │     ├── 检查 agentNameRegistry
  │     ├── 如果运行中 → queuePendingMessage()
  │     └── 如果已停止 → resumeAgentBackground()
  ├── 纯文本消息
  │     ├── to === "*" → handleBroadcast()
  │     └── 否则 → handleMessage()
  └── 结构化消息
        ├── shutdown_request → handleShutdownRequest()
        ├── shutdown_response → handleShutdownApproval/Rejection()
        └── plan_approval_response → handlePlanApproval/Rejection()
```

#### 2. Mailbox 写入流程 (`handleMessage`)
```typescript
async function handleMessage(recipientName, content, summary, context):
  1. 获取发送者信息（名称、颜色）
  2. 调用 writeToMailbox() 写入收件箱
  3. 返回 MessageOutput 包含路由信息
```

#### 3. 广播流程 (`handleBroadcast`)
```typescript
async function handleBroadcast(content, summary, context):
  1. 验证团队上下文存在
  2. 读取 teamFile 获取所有成员
  3. 排除发送者自身
  4. 循环调用 writeToMailbox() 向每个成员发送
  5. 返回 BroadcastOutput 包含接收者列表
```

#### 4. Shutdown 协议处理

**Shutdown Request** (`handleShutdownRequest`):
- 生成确定性 requestId: `generateRequestId('shutdown', targetName)`
- 创建 shutdown_request 消息
- 写入目标 mailbox

**Shutdown Approval** (`handleShutdownApproval`):
- 验证请求合法性
- 获取自身 paneId 和 backendType
- 发送 shutdown_approved 消息给 team-lead
- **关键逻辑**：
  - In-process 队友：通过 abortController.abort() 终止
  - 其他：调用 gracefulShutdown(0, 'other')

#### 5. Plan Approval 处理

**Plan Approval** (`handlePlanApproval`):
- 仅 team-lead 可执行
- 继承当前 permission mode（plan mode → default）
- 发送 plan_approval_response 消息

### 地址解析协议

```typescript
// parseAddress 函数解析规则
to = "uds:/path/to.sock"    → { scheme: 'uds', target: '/path/to.sock' }
to = "bridge:session_xxx"   → { scheme: 'bridge', target: 'session_xxx' }
to = "/bare/socket/path"    → { scheme: 'uds', target: '/bare/socket/path' } (legacy)
to = "teammate_name"        → { scheme: 'other', target: 'teammate_name' }
```

### 输入验证规则 (`validateInput`)

1. `to` 不能为空
2. `to` 不能包含 `@` 符号（单团队限制）
3. bridge/uds 地址必须有非空 target
4. 纯文本消息必须有 summary
5. 结构化消息不能广播（to !== "*"）
6. 结构化消息不能跨会话发送
7. shutdown_response 必须发送给 team-lead
8. 拒绝 shutdown 必须提供 reason

### 权限检查 (`checkPermissions`)

- **Bridge 消息**：返回 `behavior: 'ask'`，需要用户显式确认
- **安全原因**：跨机器消息可能涉及提示注入风险
- **决策标记**：`decisionReason: { type: 'safetyCheck', classifierApprovable: false }`

## 关键代码路径与文件引用

### 核心文件
- `src/tools/SendMessageTool/SendMessageTool.ts` - 主实现（917 行）
- `src/tools/SendMessageTool/constants.ts` - 工具名称常量
- `src/tools/SendMessageTool/prompt.ts` - 工具提示模板
- `src/tools/SendMessageTool/UI.tsx` - UI 渲染组件

### 依赖文件

#### 工具框架
- `src/Tool.ts` - Tool 接口定义和 buildTool 辅助函数

#### 队友/团队系统
- `src/utils/teammate.ts` - 队友身份获取（getAgentName, getTeamName, isTeamLead 等）
- `src/utils/teammateMailbox.ts` - Mailbox 读写操作（writeToMailbox, readMailbox 等）
- `src/utils/swarm/teamHelpers.ts` - 团队文件操作（readTeamFileAsync, TeamFile 类型）
- `src/utils/swarm/constants.ts` - 常量定义（TEAM_LEAD_NAME）

#### Agent 任务管理
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` - In-process 队友任务管理
  - `findTeammateTaskByAgentId()` - 按 agentId 查找任务
- `src/tasks/InProcessTeammateTask/types.ts` - InProcessTeammateTaskState 类型定义
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx` - Local agent 任务管理
  - `queuePendingMessage()` - 消息队列管理
  - `isLocalAgentTask()` - 类型守卫

#### Agent 恢复
- `src/tools/AgentTool/resumeAgent.ts` - `resumeAgentBackground()` 函数

#### 工具与辅助
- `src/utils/peerAddress.ts` - `parseAddress()` 地址解析
- `src/utils/agentSwarmsEnabled.ts` - `isAgentSwarmsEnabled()` 功能开关
- `src/utils/agentId.ts` - `generateRequestId()` 请求 ID 生成
- `src/utils/semanticBoolean.ts` - `semanticBoolean()` Zod schema 辅助
- `src/utils/lazySchema.ts` - `lazySchema()` 延迟 schema 构造

#### 桥接/UDS（动态导入）
- `src/bridge/peerSessions.ts` - `postInterClaudeMessage()`（bridge 消息）
- `src/utils/udsClient.ts` - `sendToUdsSocket()`（UDS 消息）

## 依赖与外部交互

### 运行时依赖

1. **Mailbox 系统** (`src/utils/teammateMailbox.ts`)
   - 文件路径：`~/.claude/teams/{team_name}/inboxes/{agent_name}.json`
   - 使用文件锁（lockfile）防止并发写入冲突
   - 消息格式：`{ from, text, timestamp, read, color?, summary? }`

2. **团队文件** (`src/utils/swarm/teamHelpers.ts`)
   - 路径：`~/.claude/teams/{team_name}/config.json`
   - 包含成员列表、leadAgentId、权限模式等信息

3. **In-Process 队友运行时**
   - AsyncLocalStorage 提供上下文隔离
   - `agentNameRegistry` 用于名称到 agentId 的映射
   - `pendingMessages` 队列用于消息暂存

4. **跨会话通信（UDS_INBOX feature）**
   - UDS：Unix Domain Socket 本地通信
   - Bridge：通过 Anthropic 服务器的远程控制会话通信

### 外部调用

| 函数 | 来源 | 用途 |
|------|------|------|
| `writeToMailbox` | `teammateMailbox.ts` | 写入收件箱 |
| `readTeamFileAsync` | `teamHelpers.ts` | 读取团队配置 |
| `findTeammateTaskByAgentId` | `InProcessTeammateTask.tsx` | 查找队友任务 |
| `queuePendingMessage` | `LocalAgentTask.tsx` | 消息入队 |
| `resumeAgentBackground` | `resumeAgent.ts` | 恢复已停止的 agent |
| `generateRequestId` | `agentId.ts` | 生成请求 ID |
| `isAgentSwarmsEnabled` | `agentSwarmsEnabled.ts` | 功能开关检查 |
| `postInterClaudeMessage` | `peerSessions.ts` | Bridge 消息发送（动态导入） |
| `sendToUdsSocket` | `udsClient.ts` | UDS 消息发送（动态导入） |

## 风险、边界与改进建议

### 已知风险

1. **并发写入风险**
   - Mailbox 使用文件锁，但在极端并发下仍可能出现竞态条件
   - 锁超时配置：10 次重试，5-100ms 退避

2. **跨会话消息限制**
   - 结构化消息不能跨会话发送（仅纯文本）
   - Bridge 消息需要用户显式确认，可能中断自动化流程

3. **In-Process 队友恢复失败**
   - 如果 transcript 被清理，无法恢复已停止的 agent
   - 错误消息：`"Agent ... has no transcript to resume"`

4. **Shutdown 竞态条件**
   - In-process 队友的 abortController 可能无法及时找到
   - 有 fallback 到 gracefulShutdown 的机制

### 边界情况

1. **空团队广播**
   - 如果团队中只有自己，广播返回成功但 recipients 为空数组

2. **重复 agentId**
   - `findTeammateTaskByAgentId` 优先返回 running 状态的任务

3. **Bridge 断开**
   - 在权限检查和实际发送之间可能断开连接
   - 有二次检查：`if (!getReplBridgeHandle() || !isReplBridgeActive())`

4. **消息大小限制**
   - `maxResultSizeChars: 100_000` - 工具结果大小限制

### 改进建议

1. **可靠性改进**
   - 添加消息发送确认机制（当前仅依赖 mailbox 写入成功）
   - 考虑添加消息重试逻辑（特别是对于跨会话消息）

2. **可观测性**
   - 当前仅有 debug 日志，建议添加结构化指标
   - 消息发送延迟、失败率、队列深度等

3. **错误处理**
   - 部分错误返回 success: false，但可能应该抛出异常让上层处理
   - 跨会话消息的错误信息可以更具体

4. **性能优化**
   - 广播时顺序写入 mailbox，可以改为并行
   - 大团队广播可能是性能瓶颈（当前是 O(n)）

5. **代码组织**
   - `call` 方法较长（~170 行），可以进一步拆分为子函数
   - UDS/Bridge 处理逻辑与 teammate 处理逻辑可以分离

6. **类型安全**
   - 部分类型断言（`as typeof import(...)`）可以改进
   - 动态导入的类型定义可以更严格

### 测试建议

1. 单元测试：验证各种输入验证规则
2. 集成测试：模拟 mailbox 读写、团队文件操作
3. 并发测试：多 agent 同时写入同一 mailbox
4. 故障测试：网络断开、文件权限问题、磁盘满等场景
