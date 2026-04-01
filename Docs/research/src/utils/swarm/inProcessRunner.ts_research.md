# inProcessRunner.ts 研究文档

## 场景与职责

`inProcessRunner.ts` 是 Agent Swarm 系统中 In-Process 队友执行的核心引擎。它负责在单个 Node.js 进程中运行多个 AI Agent（队友），通过 AsyncLocalStorage 实现上下文隔离，提供与进程级队友（tmux/iTerm2）等价的功能。

### 核心职责
1. **队友生命周期管理**: 启动、运行、暂停、终止 In-Process 队友
2. **上下文隔离**: 使用 AsyncLocalStorage 和 AgentContext 实现并发安全
3. **权限处理**: 实现队友的工具使用权限请求和审批流程
4. **消息通信**: 通过文件邮箱系统实现队友间通信
5. **进度追踪**: 更新 AppState 中的任务状态和进度
6. **会话压缩**: 自动管理长对话的上下文压缩

## 功能点目的

### 1. 权限处理系统 (`createInProcessCanUseTool`)

为 In-Process 队友提供工具使用权限检查，支持两种模式：

#### 标准路径（优先）
- 使用 Leader 的 `ToolUseConfirm` 对话框
- 显示 Worker Badge（队友名称和颜色）
- 支持权限更新回写（`persistPermissionUpdates`）

#### 邮箱回退路径
- 当 Leader UI 队列不可用时使用
- 通过 `sendPermissionRequestViaMailbox` 发送请求
- 轮询邮箱等待响应（500ms 间隔）

### 2. 队友消息格式化 (`formatAsTeammateMessage`)

将消息格式化为 XML 格式，与 tmux 队友保持一致：
```xml
<teammate_message teammate_id="agent-name" color="blue" summary="brief summary">
Message content
</teammate_message>
```

### 3. 任务认领系统 (`tryClaimNextTask`)

自动从团队任务列表中认领可用任务：
- 查找状态为 `pending` 且无所有者的任务
- 检查依赖任务是否已完成
- 使用 `claimTask` 原子性认领

### 4. 空闲等待循环 (`waitForNextPromptOrShutdown`)

队友完成工作后进入空闲状态，等待：
- **关闭请求**: 来自 Leader 的优雅关闭请求（最高优先级）
- **新消息**: 来自 Leader 或其他队友的消息
- **用户输入**: 通过 `pendingUserMessages` 队列
- **任务分配**: 从任务列表自动认领

轮询间隔：500ms

### 5. 主执行循环 (`runInProcessTeammate`)

核心执行流程：
1. 构建系统提示词（支持 replace/append 模式）
2. 注入团队必需工具（SendMessage、TeamCreate、TaskCreate 等）
3. 运行 Agent 循环（`runAgent`）
4. 处理工具使用和消息流
5. 空闲时发送通知并等待新工作
6. 处理关闭请求（由模型决定是否批准）

### 6. 会话压缩

当 token 数超过阈值时自动压缩：
- 使用 `compactConversation` 生成摘要
- 重置微压缩状态和内容替换状态
- 更新 `task.messages` 以控制内存使用

## 具体技术实现

### 关键数据结构

```typescript
export type InProcessRunnerConfig = {
  identity: TeammateIdentity           // 队友身份
  taskId: string                       // AppState 中的任务 ID
  prompt: string                       // 初始提示词
  agentDefinition?: CustomAgentDefinition  // 可选的 Agent 定义
  teammateContext: TeammateContext     // AsyncLocalStorage 上下文
  toolUseContext: ToolUseContext       // 父级的工具使用上下文
  abortController: AbortController     // 生命周期中止控制器
  model?: string                       // 可选的模型覆盖
  systemPrompt?: string                // 可选的系统提示词覆盖
  systemPromptMode?: 'default' | 'replace' | 'append'
  allowedTools?: string[]              // 自动允许的工具列表
  allowPermissionPrompts?: boolean     // 是否允许权限提示
  description?: string                 // 任务描述（用于摘要）
  invokingRequestId?: string           // 调用请求的 ID（用于链路追踪）
}

export type InProcessRunnerResult = {
  success: boolean
  error?: string
  messages: Message[]
}
```

### 权限处理流程

```typescript
function createInProcessCanUseTool(
  identity: TeammateIdentity,
  abortController: AbortController,
  onPermissionWaitMs?: (waitMs: number) => void,
): CanUseToolFn {
  return async (tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision) => {
    // 1. 检查基础权限
    const result = await hasPermissionsToUseTool(...)
    
    // 2. 如果是 'ask'，尝试 Bash 分类器自动批准
    if (result.behavior === 'ask' && tool.name === BASH_TOOL_NAME) {
      const classifierDecision = await awaitClassifierAutoApproval(...)
      if (classifierDecision) return { behavior: 'allow', ... }
    }
    
    // 3. 使用 Leader 的 ToolUseConfirm 对话框
    const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()
    if (setToolUseConfirmQueue) {
      return new Promise<PermissionDecision>(resolve => {
        setToolUseConfirmQueue(queue => [...queue, {
          workerBadge: { name: identity.agentName, color: identity.color },
          onAllow(updatedInput, permissionUpdates, feedback, contentBlocks) { ... },
          onReject(feedback, contentBlocks) { ... },
          // ...
        }])
      })
    }
    
    // 4. 回退到邮箱系统
    return new Promise<PermissionDecision>(resolve => {
      sendPermissionRequestViaMailbox(request)
      // 轮询等待响应...
    })
  }
}
```

### 主循环核心逻辑

```typescript
export async function runInProcessTeammate(config: InProcessRunnerConfig): Promise<InProcessRunnerResult> {
  // 1. 构建 AgentContext 用于分析归因
  const agentContext: AgentContext = {
    agentId: identity.agentId,
    parentSessionId: identity.parentSessionId,
    agentName: identity.agentName,
    teamName: identity.teamName,
    agentColor: identity.color,
    planModeRequired: identity.planModeRequired,
    isTeamLead: false,
    agentType: 'teammate',
    invokingRequestId,
    invocationKind: 'spawn',
    invocationEmitted: false,
  }
  
  // 2. 构建系统提示词
  let teammateSystemPrompt: string
  if (systemPromptMode === 'replace' && systemPrompt) {
    teammateSystemPrompt = systemPrompt
  } else {
    const fullSystemPromptParts = await getSystemPrompt(...)
    systemPromptParts.push(TEAMMATE_SYSTEM_PROMPT_ADDENDUM)
    // ...
    teammateSystemPrompt = systemPromptParts.join('\n')
  }
  
  // 3. 主循环
  while (!abortController.signal.aborted && !shouldExit) {
    // 创建本轮工作的中止控制器（Escape 只中断当前轮次）
    const currentWorkAbortController = createAbortController()
    
    // 检查是否需要压缩
    if (tokenCount > getAutoCompactThreshold(...)) {
      contextMessages = await compactConversation(...)
    }
    
    // 运行 Agent
    await runWithTeammateContext(teammateContext, async () => {
      return runWithAgentContext(agentContext, async () => {
        for await (const message of runAgent({
          canUseTool: createInProcessCanUseTool(identity, currentWorkAbortController),
          // ...
        })) {
          // 处理消息流，更新 AppState
        }
      })
    })
    
    // 标记空闲并发送通知
    await sendIdleNotification(...)
    
    // 等待下一项工作
    const waitResult = await waitForNextPromptOrShutdown(...)
    switch (waitResult.type) {
      case 'shutdown_request':
        // 传递给模型决策
        currentPrompt = formatAsTeammateMessage(...)
        break
      case 'new_message':
        currentPrompt = waitResult.message
        break
      case 'aborted':
        shouldExit = true
        break
    }
  }
}
```

## 关键代码路径与文件引用

### 本文件导出
| 导出项 | 说明 |
|-------|------|
| `InProcessRunnerConfig` | 运行器配置类型 |
| `InProcessRunnerResult` | 运行结果类型 |
| `runInProcessTeammate` | 主执行函数 |
| `startInProcessTeammate` | 后台启动入口 |

### 核心依赖

| 导入路径 | 用途 |
|---------|------|
| `bun:bundle` | feature flag 检查 |
| `@anthropic-ai/sdk` | ContentBlockParam 类型 |
| `../../constants/prompts.js` | `getSystemPrompt` |
| `../../constants/xml.js` | `TEAMMATE_MESSAGE_TAG` |
| `../../hooks/useCanUseTool.js` | `CanUseToolFn` 类型 |
| `../../hooks/useSwarmPermissionPoller.js` | 权限轮询回调 |
| `../../services/analytics/index.js` | 分析事件上报 |
| `../../services/compact/autoCompact.js` | 自动压缩阈值 |
| `../../services/compact/compact.js` | 会话压缩 |
| `../../state/AppState.js` | 应用状态类型 |
| `../../Tool.js` | Tool 和 ToolUseContext 类型 |
| `../../tasks/InProcessTeammateTask/*.js` | 任务状态管理 |
| `../../tools/AgentTool/runAgent.js` | 核心 Agent 运行 |
| `../../tools/BashTool/bashPermissions.js` | Bash 分类器 |
| `../../utils/agentContext.js` | AgentContext |
| `../../utils/permissions/PermissionUpdate.js` | 权限更新 |
| `../../utils/teammateContext.js` | TeammateContext |
| `../../utils/teammateMailbox.js` | 邮箱通信 |
| `./constants.js` | `TEAM_LEAD_NAME` |
| `./leaderPermissionBridge.js` | Leader 权限桥接 |
| `./permissionSync.js` | 权限同步 |
| `./teammatePromptAddendum.js` | 队友提示词附加 |

### 调用关系

```
startInProcessTeammate(config)
  └── runInProcessTeammate(config)
        ├── createInProcessCanUseTool()  [权限处理]
        │     ├── hasPermissionsToUseTool()
        │     ├── awaitClassifierAutoApproval()  [Bash 分类器]
        │     ├── getLeaderToolUseConfirmQueue()  [Leader UI]
        │     └── sendPermissionRequestViaMailbox()  [邮箱回退]
        ├── runWithTeammateContext()  [AsyncLocalStorage]
        │     └── runWithAgentContext()
        │           └── runAgent()  [核心 Agent 循环]
        ├── compactConversation()  [会话压缩]
        ├── sendIdleNotification()  [空闲通知]
        └── waitForNextPromptOrShutdown()  [等待工作]
              ├── readMailbox()  [读取消息]
              └── tryClaimNextTask()  [认领任务]
```

## 依赖与外部交互

### 外部系统
1. **文件系统**: 通过 `teammateMailbox.ts` 读写邮箱文件
2. **Analytics**: 上报 `tengu_agent_memory_loaded` 等事件
3. **Perfetto Tracing**: 注册/注销 Agent 追踪

### AppState 交互
- 使用 `toolUseContext.setAppState` 更新任务状态
- 更新字段：status、isIdle、progress、messages、inProgressToolUseIDs 等

### 权限系统交互
- 通过 `leaderPermissionBridge.ts` 访问 Leader 的权限队列
- 支持权限更新回写到 Leader 的共享上下文

## 风险、边界与改进建议

### 风险点

1. **内存泄漏风险**
   - `allMessages` 数组在长时间运行的队友中可能无限增长
   - **缓解**: 已实现 `TEAMMATE_MESSAGES_UI_CAP = 50` 限制 UI 镜像大小
   - 但 `allMessages` 本身仍保存完整历史

2. **竞态条件**
   - 多个队友同时写入邮箱文件
   - **缓解**: `teammateMailbox.ts` 使用文件锁（`lockfile.lock`）

3. **权限回调泄漏**
   - 如果权限请求未正确清理，回调可能残留
   - **缓解**: `cleanup()` 函数在多种路径下被调用

4. **AbortController 嵌套复杂性**
   - `abortController`（生命周期）vs `currentWorkAbortController`（单轮）
   - 需要仔细处理两者的关系和事件监听器的清理

### 边界情况

1. **空任务列表**
   - `tryClaimNextTask` 在无可认领任务时返回 `undefined`
   - 队友继续轮询等待

2. **权限模式切换**
   - 用户可通过 Shift+Tab 切换队友的权限模式
   - 每轮迭代从 AppState 重新读取当前模式

3. **压缩过程中的中断**
   - 压缩可能耗时较长，期间可能被中止
   - 使用隔离的 `isolatedContext` 避免影响主会话

4. **消息队列溢出**
   - `pendingUserMessages` 可能无限增长
   - 需要上限保护机制

### 改进建议

1. **消息历史限制**
   ```typescript
   // 建议添加硬上限，丢弃最旧消息
   const MAX_ALL_MESSAGES = 1000
   if (allMessages.length > MAX_ALL_MESSAGES) {
     allMessages.splice(0, allMessages.length - MAX_ALL_MESSAGES)
   }
   ```

2. **指数退避轮询**
   - 当前固定 500ms 轮询
   - 可改为空闲时延长轮询间隔（500ms → 1000ms → 2000ms）

3. **批量权限处理**
   - 当前每个权限请求单独处理
   - 可支持批量审批提升效率

4. **会话压缩触发优化**
   - 当前仅基于 token 数
   - 可考虑结合消息数和压缩后大小预测

5. **错误恢复增强**
   - 邮箱读取失败时当前仅记录日志
   - 可添加重试和降级策略

6. **性能监控**
   - 添加更多调试指标：轮询次数、权限等待时间、压缩耗时等
   - 便于识别性能瓶颈
