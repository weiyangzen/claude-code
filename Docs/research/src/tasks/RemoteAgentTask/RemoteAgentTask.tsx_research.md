# RemoteAgentTask.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`RemoteAgentTask` 是 Claude Code CLI 中负责**远程代理任务执行**的核心模块。它实现了本地 CLI 与远程 Claude Code on the Web (CCR) 会话之间的桥梁，支持以下主要场景：

- **Ultraplan**: 远程多代理计划模式，在云端运行高级计划制定
- **Ultrareview**: 远程代码审查，利用云端资源进行深度代码分析
- **Remote Agent**: 通过 AgentTool 启动的远程隔离代理任务
- **Autofix PR**: 自动修复 PR 中的问题
- **Background PR**: 后台 PR 处理任务

### 1.2 架构位置
```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code CLI (Local)                   │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Local Tasks │  │ RemoteAgent  │  │   CCR Session    │  │
│  │  (Shell/Agent)│  │    Task      │  │   (Cloud)        │  │
│  └──────────────┘  └──────┬───────┘  └──────────────────┘  │
│                           │                                  │
│                    ┌──────▼───────┐                         │
│                    │  Sessions API │ ←── OAuth + REST       │
│                    └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 主要职责
1. **任务生命周期管理**: 注册、轮询、完成、终止远程任务
2. **状态持久化**: 通过 `sessionStorage.ts` 保存任务元数据，支持 `--resume` 恢复
3. **事件轮询**: 定期从 CCR 获取会话事件，更新本地状态
4. **结果提取**: 解析远程会话输出，提取计划、审查结果等
5. **通知机制**: 通过消息队列将任务完成通知传递给用户

---

## 2. 功能点目的

### 2.1 任务类型定义 (RemoteTaskType)
```typescript
const REMOTE_TASK_TYPES = [
  'remote-agent',   // 通用远程代理
  'ultraplan',      // 远程计划模式
  'ultrareview',    // 远程代码审查
  'autofix-pr',     // 自动修复 PR
  'background-pr'   // 后台 PR 处理
] as const;
```

### 2.2 核心功能模块

| 功能模块 | 目的 | 关键函数/类 |
|---------|------|-----------|
| **前置条件检查** | 验证用户是否有权限创建远程会话 | `checkRemoteAgentEligibility()` |
| **任务注册** | 创建任务状态并启动轮询 | `registerRemoteAgentTask()` |
| **任务恢复** | 从持久化存储恢复中断的任务 | `restoreRemoteAgentTasks()` |
| **事件轮询** | 从 CCR 获取最新事件 | `startRemoteSessionPolling()` |
| **结果提取** | 从日志中提取计划/审查结果 | `extractPlanFromLog()`, `extractReviewFromLog()` |
| **任务终止** | 清理资源并归档远程会话 | `RemoteAgentTask.kill()` |
| **完成检查器** | 自定义任务完成判断逻辑 | `registerCompletionChecker()` |

### 2.3 状态流转
```
pending → running → completed/failed/killed
   ↑       ↓
   └── 支持 --resume 恢复
```

---

## 3. 具体技术实现

### 3.1 数据结构

#### 3.1.1 RemoteAgentTaskState
```typescript
type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent';
  remoteTaskType: RemoteTaskType;
  remoteTaskMetadata?: RemoteTaskMetadata;
  sessionId: string;           // CCR 会话 ID
  command: string;             // 执行的命令
  title: string;               // 任务标题
  todoList: TodoList;          // 待办事项列表
  log: SDKMessage[];           // 会话日志
  isLongRunning?: boolean;     // 是否长期运行
  pollStartedAt: number;       // 轮询开始时间
  isRemoteReview?: boolean;    // 是否为远程审查
  reviewProgress?: {           // 审查进度
    stage?: 'finding' | 'verifying' | 'synthesizing';
    bugsFound: number;
    bugsVerified: number;
    bugsRefuted: number;
  };
  isUltraplan?: boolean;
  ultraplanPhase?: UltraplanPhase;
};
```

#### 3.1.2 RemoteAgentMetadata (持久化)
```typescript
type RemoteAgentMetadata = {
  taskId: string;
  remoteTaskType: string;
  sessionId: string;
  title: string;
  command: string;
  spawnedAt: number;
  toolUseId?: string;
  isLongRunning?: boolean;
  isUltraplan?: boolean;
  isRemoteReview?: boolean;
  remoteTaskMetadata?: Record<string, unknown>;
};
```

### 3.2 关键流程

#### 3.2.1 任务注册流程
```typescript
function registerRemoteAgentTask(options: {
  remoteTaskType: RemoteTaskType;
  session: { id: string; title: string };
  command: string;
  context: TaskContext;
  // ... 其他选项
}): { taskId: string; sessionId: string; cleanup: () => void }
```

流程步骤：
1. 生成任务 ID (`generateTaskId('remote_agent')`)
2. 初始化任务输出文件 (`initTaskOutput`)
3. 创建任务状态对象 (`RemoteAgentTaskState`)
4. 注册任务到 AppState (`registerTask`)
5. 持久化元数据到磁盘 (`persistRemoteAgentMetadata`)
6. 启动事件轮询 (`startRemoteSessionPolling`)

#### 3.2.2 事件轮询机制
```typescript
function startRemoteSessionPolling(taskId: string, context: TaskContext): () => void
```

轮询参数：
- **轮询间隔**: 1000ms (POLL_INTERVAL_MS)
- **审查超时**: 30分钟 (REMOTE_REVIEW_TIMEOUT_MS)
- **稳定空闲检测**: 5次连续空闲 (STABLE_IDLE_POLLS)

轮询逻辑：
```typescript
const poll = async (): Promise<void> => {
  // 1. 获取任务状态
  const task = appState.tasks?.[taskId] as RemoteAgentTaskState;
  
  // 2. 调用 CCR API 获取新事件
  const response = await pollRemoteSessionEvents(task.sessionId, lastEventId);
  
  // 3. 追加新事件到日志
  if (response.newEvents.length > 0) {
    accumulatedLog = [...accumulatedLog, ...response.newEvents];
    appendTaskOutput(taskId, deltaText);
  }
  
  // 4. 检查会话状态
  if (response.sessionStatus === 'archived') {
    // 任务完成，发送通知
    enqueueRemoteNotification(taskId, task.title, 'completed', ...);
    return;
  }
  
  // 5. 检查完成条件
  const checker = completionCheckers.get(task.remoteTaskType);
  if (checker) {
    const result = await checker(task.remoteTaskMetadata);
    if (result !== null) {
      // 任务完成
      return;
    }
  }
  
  // 6. 处理远程审查特定逻辑
  if (task.isRemoteReview) {
    // 提取审查内容
    cachedReviewContent = extractReviewTagFromLog(response.newEvents);
    // 解析进度
    newProgress = parseReviewProgress(response.newEvents);
  }
  
  // 7. 更新任务状态
  updateTaskState(taskId, setAppState, prevTask => ({ ... }));
  
  // 8. 继续轮询
  setTimeout(poll, POLL_INTERVAL_MS);
};
```

#### 3.2.3 任务恢复流程
```typescript
async function restoreRemoteAgentTasks(context: TaskContext): Promise<void>
```

恢复步骤：
1. 扫描 `remote-agents/` 目录获取所有持久化的元数据
2. 对每个任务调用 `fetchSession(sessionId)` 获取远程状态
3. 如果远程会话已归档或 404，删除本地元数据
4. 否则重建任务状态并重新启动轮询

### 3.3 协议与 API

#### 3.3.1 CCR Sessions API 端点
| 端点 | 方法 | 用途 |
|-----|------|------|
| `/v1/sessions` | POST | 创建新会话 |
| `/v1/sessions/{id}` | GET | 获取会话状态 |
| `/v1/sessions/{id}/events` | GET | 获取会话事件 |
| `/v1/sessions/{id}/archive` | POST | 归档会话 |

#### 3.3.2 事件类型 (SDKMessage)
```typescript
type SDKMessage = 
  | { type: 'assistant'; message: { content: ContentBlock[] } }
  | { type: 'user'; message: { content: ContentBlock[] } }
  | { type: 'result'; subtype: 'success' | 'error_during_execution' | ... }
  | { type: 'system'; subtype: 'hook_progress' | 'hook_response' | 'hook_started'; stdout: string; hook_event?: string };
```

### 3.4 关键命令

#### 3.4.1 前置条件检查
```typescript
export async function checkRemoteAgentEligibility({
  skipBundle = false
}: { skipBundle?: boolean } = {}): Promise<RemoteAgentPreconditionResult>
```

检查项：
- 用户是否已登录 Claude.ai (`checkNeedsClaudeAiLogin`)
- 是否有可用的远程环境 (`checkHasRemoteEnvironment`)
- 是否在 Git 仓库中 (`checkIsInGitRepo`)
- 是否有 GitHub 远程 (`checkHasGitRemote`)
- GitHub App 是否已安装 (`checkGithubAppInstalled`)
- 组织策略是否允许 (`isPolicyAllowed`)

#### 3.4.2 结果提取函数

**提取计划** (`extractPlanFromLog`):
- 从后向前扫描助手消息
- 查找 `<ultraplan>...</ultraplan>` 标签
- 返回计划文本

**提取审查结果** (`extractReviewFromLog`):
- 优先扫描 `hook_progress`/`hook_response` 系统消息
- 查找 `<remote-review>...</remote-review>` 标签
- 回退到助手消息文本拼接

**提取审查标签** (`extractReviewTagFromLog`):
- 仅返回明确的 `<remote-review>` 标签内容
- 不返回回退文本，用于增量扫描

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 职责 |
|---------|------|
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 主实现文件，包含所有核心逻辑 |
| `src/Task.ts` | Task 类型定义、任务 ID 生成、基础状态创建 |
| `src/tasks.ts` | 任务注册表，统一导出所有任务类型 |
| `src/tasks/types.ts` | TaskState 联合类型定义 |

### 4.2 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/utils/teleport.tsx` | CCR 会话创建、事件轮询、归档 (`pollRemoteSessionEvents`, `archiveRemoteSession`) |
| `src/utils/teleport/api.ts` | Sessions API 调用 (`fetchSession`, `sendEventToRemoteSession`) |
| `src/utils/sessionStorage.ts` | 远程代理元数据持久化 (`writeRemoteAgentMetadata`, `listRemoteAgentMetadata`) |
| `src/utils/task/framework.ts` | 任务框架通用函数 (`registerTask`, `updateTaskState`) |
| `src/utils/task/diskOutput.ts` | 任务输出文件管理 (`initTaskOutput`, `appendTaskOutput`, `evictTaskOutput`) |
| `src/utils/messageQueueManager.ts` | 消息队列管理 (`enqueuePendingNotification`) |
| `src/utils/background/remote/remoteSession.ts` | 前置条件检查 (`checkBackgroundRemoteSessionEligibility`) |
| `src/utils/ultraplan/ccrSession.ts` | Ultraplan 专用轮询逻辑 (`pollForApprovedExitPlanMode`) |

### 4.3 调用方文件

| 文件路径 | 调用方式 |
|---------|---------|
| `src/commands/ultraplan.tsx` | `registerRemoteAgentTask({ remoteTaskType: 'ultraplan', ... })` |
| `src/commands/review/reviewRemote.ts` | `registerRemoteAgentTask({ remoteTaskType: 'ultrareview', ... })` |
| `src/tools/AgentTool/AgentTool.tsx` | `registerRemoteAgentTask({ remoteTaskType: 'remote-agent', ... })` |
| `src/state/AppStateStore.ts` | `restoreRemoteAgentTasks(context)` 在恢复会话时调用 |

### 4.4 关键代码路径图

```
启动远程任务:
┌─────────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│  /ultraplan     │────→│  launchUltraplan()      │────→│ teleportToRemote │
│  /ultrareview   │     │  (ultraplan.tsx)        │     │ (teleport.tsx)   │
│  AgentTool      │     └─────────────────────────┘     └────────┬─────────┘
└─────────────────┘                                            │
                                                               ↓
┌─────────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│ startDetached   │←────│ registerRemoteAgentTask │←────│  Sessions API    │
│ Poll (ultraplan)│     │ (RemoteAgentTask.tsx)   │     │  (创建会话)      │
└─────────────────┘     └───────────┬─────────────┘     └──────────────────┘
                                    │
                                    ↓
                    ┌───────────────────────────────┐
                    │ startRemoteSessionPolling()   │
                    │ • 每 1s 轮询 CCR 事件          │
                    │ • 更新任务状态                 │
                    │ • 检查完成条件                 │
                    └───────────────────────────────┘

任务恢复 (--resume):
┌─────────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│  switchSession  │────→│ restoreRemoteAgentTasks │────→│ listRemoteAgent  │
│  (sessionStorage)│     │ (RemoteAgentTask.tsx)   │     │ Metadata()       │
└─────────────────┘     └───────────┬─────────────┘     └────────┬─────────┘
                                    │                          │
                                    ↓                          ↓
                    ┌─────────────────────────┐     ┌──────────────────┐
                    │  fetchSession()         │←────│ 读取元数据文件   │
                    │  (检查远程状态)          │     │ (remote-agents/) │
                    └───────────┬─────────────┘     └──────────────────┘
                                │
                                ↓
                    ┌─────────────────────────┐
                    │ 重新注册运行中任务       │
                    │ 启动轮询                │
                    └─────────────────────────┘
```

---

## 5. 依赖与外部交互

### 5.1 外部 API 依赖

#### 5.1.1 Claude.ai Sessions API
- **认证**: OAuth 2.0 (Claude.ai 账号)
- **基础 URL**: `https://api.claude.ai/v1/sessions`
- **关键 Header**: 
  - `Authorization: Bearer {accessToken}`
  - `anthropic-beta: ccr-byoc-2025-07-29`
  - `x-organization-uuid: {orgUUID}`

#### 5.1.2 GitHub API (前置条件检查)
- 检查 GitHub App 安装状态
- 检查 GitHub Token 同步状态

### 5.2 内部模块依赖

```
RemoteAgentTask.tsx
├── @anthropic-ai/sdk/resources (ToolUseBlock)
├── ../../constants/product.js (getRemoteSessionUrl)
├── ../../constants/xml.js (各种 XML 标签)
├── ../../entrypoints/agentSdkTypes.js (SDKMessage)
├── ../../Task.js (Task, TaskContext, TaskStateBase)
├── ../../tools/TodoWriteTool/TodoWriteTool.js
├── ../../utils/background/remote/remoteSession.js
├── ../../utils/debug.js (logForDebugging)
├── ../../utils/log.js (logError)
├── ../../utils/messageQueueManager.js (enqueuePendingNotification)
├── ../../utils/messages.js (extractTag, extractTextContent)
├── ../../utils/sdkEventQueue.js (emitTaskTerminatedSdk)
├── ../../utils/sessionStorage.js (RemoteAgentMetadata 持久化)
├── ../../utils/slowOperations.js (jsonStringify)
├── ../../utils/task/diskOutput.js (任务输出文件)
├── ../../utils/task/framework.js (registerTask, updateTaskState)
├── ../../utils/teleport/api.js (fetchSession)
├── ../../utils/teleport.js (pollRemoteSessionEvents, archiveRemoteSession)
└── ../../utils/todo/types.js (TodoList)
```

### 5.3 配置依赖

| 配置项 | 来源 | 用途 |
|-------|------|------|
| `SESSION_INGRESS_URL` | 环境变量 | 构建远程会话 URL |
| `CCR_FORCE_BUNDLE` | 环境变量 | 强制使用 bundle 模式 |
| `CCR_ENABLE_BUNDLE` | 环境变量 | 启用 bundle 模式 |
| `BUGHUNTER_*` | GrowthBook / 环境变量 | 审查任务配置 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 超时风险
- **远程审查超时**: 30分钟硬编码超时 (`REMOTE_REVIEW_TIMEOUT_MS`)
- **风险**: 大型代码库审查可能超时
- **缓解**: 使用 `STABLE_IDLE_POLLS` (5次) 避免短暂空闲误判

#### 6.1.2 网络可靠性
- **轮询依赖**: 每秒轮询 CCR API，网络中断可能导致任务状态丢失
- **风险**: `isTransientNetworkError` 仅重试 5xx 错误，4xx 直接失败
- **缓解**: 恢复机制 (`restoreRemoteAgentTasks`) 可在重新连接后恢复任务

#### 6.1.3 竞态条件
- **状态更新竞态**: `updateTaskState` 使用函数式更新，但轮询和 kill 可能并发
- **风险**: `raceTerminated` 标志用于检测，但仍有窗口期
- **代码位置**: `startRemoteSessionPolling` 函数中的 `raceTerminated` 检查

#### 6.1.4 存储泄漏
- **元数据残留**: 如果任务完成但清理失败，`remote-agents/` 目录可能残留文件
- **风险**: `--resume` 时可能尝试恢复已不存在的会话
- **缓解**: `restoreRemoteAgentTasks` 会检查远程状态，404 时清理本地元数据

### 6.2 边界情况

#### 6.2.1 会话归档
- **行为**: 用户点击 kill 后，调用 `archiveRemoteSession` 归档远程会话
- **边界**: 归档是"尽力而为"，失败会留下孤儿会话直到被回收
- **代码**: `RemoteAgentTask.kill()` 和 `archiveRemoteSession()`

#### 6.2.2 Bundle 模式
- **触发条件**: `useBundle: true` 或环境变量 `CCR_FORCE_BUNDLE=1`
- **边界**: 大型仓库可能超过 bundle 大小限制
- **处理**: `createAndUploadGitBundle` 返回失败，任务启动失败

#### 6.2.3 审查结果提取
- **多生产者问题**: bughunter 模式 (hook) 和 prompt 模式 (assistant) 产生不同事件形状
- **处理**: `extractReviewFromLog` 优先扫描 hook 事件，回退到 assistant 消息
- **边界**: 大型 JSON 负载可能跨多个 hook 事件分割，需要拼接处理

### 6.3 改进建议

#### 6.3.1 架构改进
1. **统一轮询逻辑**: 
   - 当前: Ultraplan 使用独立的 `startDetachedPoll` + `ExitPlanModeScanner`
   - 建议: 将 `ExitPlanModeScanner` 集成到 `startRemoteSessionPolling` (TODO #23985)

2. **指数退避轮询**:
   - 当前: 固定 1s 轮询间隔
   - 建议: 空闲时增加轮询间隔，减少 API 调用

3. **批量事件处理**:
   - 当前: 每次轮询处理 `newEvents`
   - 建议: 支持更大批量，减少状态更新频率

#### 6.3.2 可靠性改进
1. **断线重连**:
   - 当前: 网络错误仅重置空闲计数器
   - 建议: 实现指数退避重连，保持任务状态

2. **进度持久化**:
   - 当前: 仅持久化任务身份，不保存进度
   - 建议: 定期保存 `reviewProgress` 到元数据

3. **孤儿会话清理**:
   - 当前: 依赖服务器端 TTL
   - 建议: 客户端定期扫描并清理孤儿会话

#### 6.3.3 可观测性改进
1. **详细指标**:
   - 添加轮询延迟、API 错误率、任务持续时间等指标

2. **调试信息**:
   - 增强 `logForDebugging` 输出，包含更多上下文

3. **用户反馈**:
   - 在 UI 中显示轮询状态和最后一次成功轮询时间

#### 6.3.4 代码质量
1. **类型安全**:
   - `completionCheckers` Map 使用 `RemoteTaskType` 作为 key，但获取时可能返回 undefined
   - 建议: 添加更严格的类型检查

2. **错误处理**:
   - 部分 `void` 调用的 Promise 错误未被捕获
   - 建议: 统一使用 `void promise.catch(...)` 模式

3. **测试覆盖**:
   - 关键路径如 `extractReviewFromLog` 的多种回退场景需要单元测试
   - 轮询逻辑的状态转换需要更全面的测试

---

## 7. 附录

### 7.1 相关 Issue/PR 参考
- `#23985`: 将 ExitPlanModeScanner 集成到 RemoteAgentTask 轮询器
- `#22546`: Task 接口简化 (移除 spawn/render 的多态调用)
- `#22051`: Ultrareview bundle 模式支持

### 7.2 环境变量汇总
| 变量名 | 用途 |
|-------|------|
| `SESSION_INGRESS_URL` | 远程会话 URL 前缀 |
| `CCR_FORCE_BUNDLE` | 强制使用 git bundle 模式 |
| `CCR_ENABLE_BUNDLE` | 启用 bundle 模式 |
| `BUGHUNTER_DEV_BUNDLE_B64` | 开发用 bundle 覆盖 |
| `BUGHUNTER_*` | 审查配置参数 |

### 7.3 文件存储位置
```
~/.claude-code/projects/{sanitizedCwd}/
└── {sessionId}/
    ├── {sessionId}.jsonl          # 主会话文件
    ├── subagents/                 # 子代理会话
    │   └── agent-{agentId}.jsonl
    └── remote-agents/             # 远程代理元数据
        └── remote-agent-{taskId}.meta.json
```
