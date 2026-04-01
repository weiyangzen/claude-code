# LocalMainSessionTask.ts 研究文档

## 场景与职责

LocalMainSessionTask 是 Claude Code CLI 中处理**主会话后台化**的核心模块。当用户在查询执行期间按下 `Ctrl+B` 两次时，当前主会话查询会被"后台化"：

- 查询继续在后台运行
- UI 清空并显示新的提示符
- 查询完成时发送通知

该模块复用了 LocalAgentTask 的状态结构，因为行为模式相似（都是后台运行的 agent 类任务）。

### 核心使用场景

1. **用户主动后台化当前查询**：通过 `Ctrl+B` 快捷键触发
2. **启动新的后台会话**：通过 `startBackgroundSession` 函数创建独立的后台查询
3. **前台/后台状态切换**：支持将后台任务重新前台化查看
4. **任务完成通知**：后台任务完成时向用户发送 XML 格式的通知消息

---

## 功能点目的

### 1. 主会话任务注册 (`registerMainSessionTask`)

创建并注册一个新的后台化主会话任务：

- 生成唯一的任务 ID（以 's' 前缀区分于普通 agent 任务的 'a' 前缀）
- 初始化任务输出符号链接到隔离的 transcript 文件
- 支持复用现有的 AbortController（用于后台化正在运行的查询）
- 注册清理回调，在进程退出时清理任务状态

### 2. 任务完成处理 (`completeMainSessionTask`)

当后台查询完成时调用：

- 更新任务状态为 'completed' 或 'failed'
- 仅对仍处于后台状态的任务发送通知（前台化的任务不需要通知）
- 清理任务输出文件

### 3. 任务前台化 (`foregroundMainSessionTask`)

将后台任务切换到前台显示：

- 标记任务为前台状态 (`isBackgrounded: false`)
- 如果有其他前台任务，将其恢复为后台状态
- 返回任务累积的消息用于显示

### 4. 后台会话启动 (`startBackgroundSession`)

启动一个独立的后台查询会话：

- 创建新的后台任务
- 在 agent 上下文中运行查询（支持技能调用的作用域隔离）
- 流式处理查询结果，实时更新任务状态和进度
- 支持中止信号处理

### 5. 任务类型判断 (`isMainSessionTask`)

类型守卫函数，用于判断一个任务是否是主会话任务（vs 普通 agent 任务）。

---

## 具体技术实现

### 关键数据结构

```typescript
// 主会话任务状态，继承自 LocalAgentTaskState
export type LocalMainSessionTaskState = LocalAgentTaskState & {
  agentType: 'main-session'  // 标识为主会话任务
}

// 默认主会话 agent 定义
const DEFAULT_MAIN_SESSION_AGENT: CustomAgentDefinition = {
  agentType: 'main-session',
  whenToUse: 'Main session query',
  source: 'userSettings',
  getSystemPrompt: () => '',
}
```

### 任务 ID 生成

```typescript
const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz'

function generateMainSessionTaskId(): string {
  const bytes = randomBytes(8)
  let id = 's'  // 's' 前缀表示 session 任务
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id
}
```

### 关键流程

#### 后台化流程

1. 用户按下 `Ctrl+B` 两次触发后台化
2. 调用 `registerMainSessionTask` 创建任务
3. 初始化任务输出符号链接（隔离到独立 transcript 文件）
4. 在 agent 上下文中启动 `query()` 调用
5. 流式处理消息，更新任务进度和消息
6. 完成后调用 `completeMainSessionTask`

#### 消息处理流程

```typescript
for await (const event of query({ messages: bgMessages, ...queryParams })) {
  if (abortSignal.aborted) {
    // 处理中止信号
    emitTaskTerminatedSdk(taskId, 'stopped', { summary: description })
    return
  }
  
  // 过滤只保留用户/助手/系统消息
  if (event.type !== 'user' && event.type !== 'assistant' && event.type !== 'system') {
    continue
  }
  
  bgMessages.push(event)
  
  // 写入 transcript
  void recordSidechainTranscript([event], taskId, lastRecordedUuid)
  
  // 统计 token 和工具使用
  if (event.type === 'assistant') {
    for (const block of event.message.content) {
      if (block.type === 'text') {
        tokenCount += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_use') {
        toolCount++
        recentActivities.push({ toolName: block.name, input: block.input })
      }
    }
  }
  
  // 更新 AppState
  setAppState(prev => ({ ...prev, tasks: { ...prev.tasks, [taskId]: updatedTask } }))
}
```

#### 通知消息格式

```xml
<task_notification>
<task_id>sxxxxxxxx</task_id>
<tool_use_id>...</tool_use_id>  <!-- 可选 -->
<output_file>/path/to/output</output_file>
<status>completed|failed</status>
<summary>Background session "description" completed</summary>
</task_notification>
```

### 协议与命令

- **XML 标签常量**：使用 `../constants/xml.js` 中定义的 `TASK_NOTIFICATION_TAG`, `TASK_ID_TAG`, `STATUS_TAG` 等
- **Agent 上下文**：通过 `runWithAgentContext` 在 AsyncLocalStorage 中设置 agentId，实现技能调用的作用域隔离
- **清理注册**：使用 `registerCleanup` 在进程退出时自动清理任务

---

## 关键代码路径与文件引用

### 核心文件

| 文件路径 | 用途 |
|---------|------|
| `src/tasks/LocalMainSessionTask.ts` | 本文件，主会话后台化逻辑 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | LocalAgentTaskState 类型定义，被复用的状态结构 |
| `src/Task.ts` | TaskStateBase 基础类型，任务 ID 生成工具 |

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/query.ts` | `query()` 函数，执行实际的 LLM 查询 |
| `src/utils/agentContext.ts` | `runWithAgentContext`, `SubagentContext` - agent 上下文管理 |
| `src/utils/sessionStorage.ts` | `getAgentTranscriptPath`, `recordSidechainTranscript` - 会话存储 |
| `src/utils/task/diskOutput.ts` | `initTaskOutputAsSymlink`, `evictTaskOutput`, `getTaskOutputPath` - 任务输出管理 |
| `src/utils/task/framework.ts` | `registerTask`, `updateTaskState` - 任务框架 |
| `src/utils/messageQueueManager.ts` | `enqueuePendingNotification` - 消息队列 |
| `src/utils/sdkEventQueue.ts` | `emitTaskTerminatedSdk` - SDK 事件 |
| `src/utils/cleanupRegistry.ts` | `registerCleanup` - 清理注册 |
| `src/utils/abortController.ts` | `createAbortController` - 中止控制器 |
| `src/services/tokenEstimation.ts` | `roughTokenCountEstimation` - Token 估算 |
| `src/constants/xml.ts` | XML 标签常量 |

### 调用方文件

| 文件路径 | 用途 |
|---------|------|
| `src/screens/REPL.tsx` | 主 REPL 界面，处理 Ctrl+B 快捷键 |
| `src/commands/clear/conversation.ts` | `/clear` 命令处理，清理后台任务 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | 发送消息工具，可能与后台任务交互 |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | Shell 任务，可能相关 |

---

## 依赖与外部交互

### 类型依赖

```typescript
// 从 LocalAgentTask 导入
import type { LocalAgentTaskState } from './LocalAgentTask/LocalAgentTask.js'

// 从 Task 导入
import type { SetAppState } from '../Task.js'
import { createTaskStateBase } from '../Task.js'

// 从 agent 工具导入
import type { AgentDefinition, CustomAgentDefinition } from '../tools/AgentTool/loadAgentsDir.js'

// 从类型系统导入
import type { Message } from '../types/message.js'
import { asAgentId } from '../types/ids.js'
```

### 外部服务交互

1. **会话存储系统** (`sessionStorage.ts`)
   - `getAgentTranscriptPath`: 获取 agent transcript 文件路径
   - `recordSidechainTranscript`: 记录旁路 transcript

2. **任务磁盘输出** (`task/diskOutput.ts`)
   - `initTaskOutputAsSymlink`: 初始化任务输出为符号链接
   - `evictTaskOutput`: 清理任务输出
   - `getTaskOutputPath`: 获取任务输出路径

3. **任务框架** (`task/framework.ts`)
   - `registerTask`: 注册任务到 AppState
   - `updateTaskState`: 更新任务状态

4. **消息队列** (`messageQueueManager.ts`)
   - `enqueuePendingNotification`: 将通知加入队列

5. **SDK 事件队列** (`sdkEventQueue.ts`)
   - `emitTaskTerminatedSdk`: 发送任务终止 SDK 事件

6. **Agent 上下文** (`agentContext.ts`)
   - `runWithAgentContext`: 在 agent 上下文中运行代码

7. **清理注册表** (`cleanupRegistry.ts`)
   - `registerCleanup`: 注册进程退出时的清理回调

---

## 风险、边界与改进建议

### 已知风险

1. **Transcript 文件隔离风险**
   - 代码注释明确说明：不要使用 `getTranscriptPath()`（主会话文件），否则在 `/clear` 后写入会损坏对话
   - 使用隔离路径确保任务能在 `/clear` 后存活

2. **通知重复风险**
   - `enqueueMainSessionNotification` 使用原子检查设置 `notified` 标志防止重复通知
   - 中止路径也需要检查 `alreadyNotified` 避免重复发送 SDK 事件

3. **内存泄漏风险**
   - `recentActivities` 数组限制为 `MAX_RECENT_ACTIVITIES` (5) 个
   - 消息数组 `bgMessages` 会持续累积，长时间运行的任务可能占用较多内存

4. **前台/后台状态竞争**
   - `foregroundMainSessionTask` 需要处理前一个前台任务的恢复
   - 状态更新不是原子操作，可能存在竞态条件

### 边界情况

1. **任务前台化后完成**
   - 如果任务在前台化后完成，不发送 XML 通知（用户正在观看）
   - 但仍需发送 SDK 事件 `emitTaskTerminatedSdk`

2. **中止信号处理**
   - 流式处理中检查 `abortSignal.aborted`
   - 需要区分是 `chat:killAgents` 路径还是 `stopTask` 路径

3. **空消息处理**
   - 过滤掉非 user/assistant/system 类型的消息
   - 确保只处理实际对话内容

4. **Token 估算精度**
   - 使用 `roughTokenCountEstimation` 进行粗略估算
   - 可能与实际 API 计费有偏差

### 改进建议

1. **消息数量限制**
   - 考虑对 `bgMessages` 数组设置上限，避免长时间运行任务的内存问题
   - 可以参考 `InProcessTeammateTask` 的 `TEAMMATE_MESSAGES_UI_CAP` 模式

2. **错误处理增强**
   - `recordSidechainTranscript` 的错误仅记录到调试日志
   - 考虑增加更健壮的错误处理和重试机制

3. **进度更新优化**
   - 当前每收到一条消息就更新 AppState
   - 可以考虑批量更新或节流以减少重渲染

4. **类型安全**
   - `isMainSessionTask` 类型守卫可以进一步增强，检查更多字段

5. **测试覆盖**
   - 建议增加单元测试覆盖：
     - 任务注册/完成/前台化流程
     - 中止信号处理
     - 通知去重逻辑
     - 边界情况（空消息、异常消息格式等）
