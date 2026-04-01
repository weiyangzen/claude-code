# useScheduledTasks.ts 深度研究文档

## 场景与职责

`useScheduledTasks` 是一个 React Hook，用于在 REPL 环境中包装和管理定时任务调度器（Cron Scheduler）。它负责挂载调度器、处理任务触发、并将触发的任务注入到命令队列中。

### 核心职责

1. **调度器生命周期管理**：在组件挂载时启动调度器，卸载时停止
2. **任务触发处理**：将触发的定时任务转换为提示词并提交
3. **队友任务路由**：将带有 `agentId` 的任务路由到对应的队友
4. **加载状态协调**：根据 `isLoading` 状态控制任务触发时机
5. **功能开关控制**：通过 `isKairosCronEnabled()` 控制功能启用

### 使用场景

- **团队领导（Team Lead）**：处理 durable 定时任务（持久化到磁盘）
- **队友（Teammate）**：接收分配给特定 agent 的任务
- **Assistant Mode**：绕过 `isLoading` 检查，允许在流式响应期间入队任务

---

## 功能点目的

### 1. 定时任务触发

当 cron 表达式匹配时，将任务提示词通过 `enqueuePendingNotification` 入队到命令队列，优先级为 `'later'`。

### 2. 队友任务注入

对于带有 `agentId` 的任务：
- 查找对应队友任务
- 检查队友状态（非终端状态）
- 调用 `injectUserMessageToTeammate` 注入消息
- 清理孤立任务（队友已不存在）

### 3. 错过任务提示

启动时检测错过的任务（missed tasks），显示系统消息提示用户。

### 4. 工作负载标记

所有 cron 触发的请求标记 `workload: WORKLOAD_CRON`，用于：
- 计费头中的 `cc_workload` 属性
- API 端的 QoS 降级（容量紧张时优先服务人工请求）

---

## 具体技术实现

### 关键数据结构

```typescript
interface Props {
  isLoading: boolean                    // 当前是否正在处理请求
  assistantMode?: boolean               // 是否绕过 isLoading 检查
  setMessages: React.Dispatch<React.SetStateAction<Message[]>>
}

// 来自 cronScheduler.ts
interface CronScheduler {
  start: () => void
  stop: () => void
  getNextFireTime: () => number | null
}

interface CronTask {
  id: string
  prompt: string
  cron: string
  agentId?: string                      // 可选的队友标识
  recurring?: boolean
  durable?: boolean                     // 是否持久化到磁盘
  // ... 其他字段
}
```

### 核心流程

#### 1. Hook 初始化流程
```
useEffect 触发
  ↓
检查 isKairosCronEnabled() → 未启用则直接返回
  ↓
定义 enqueueForLead: 将提示词入队到命令队列
  ↓
创建 CronScheduler:
  - onFire: 处理错过的 durable 任务
  - onFireTask: 处理正常任务触发（支持队友路由）
  - isLoading: 从 ref 读取最新值
  - assistantMode: 控制加载状态绕过
  - getJitterConfig: 获取抖动配置
  - isKilled: 运行时功能开关检查
  ↓
启动调度器 scheduler.start()
  ↓
返回清理函数 scheduler.stop()
```

#### 2. 任务触发处理流程
```
onFireTask 回调
  ↓
检查 task.agentId 是否存在
  ├── 是 → 队友任务路由流程
  └── 否 → 团队领导任务流程
```

**队友任务路由**（行 91-108）：
```typescript
if (task.agentId) {
  const teammate = findTeammateTaskByAgentId(task.agentId, store.getState().tasks)
  if (teammate && !isTerminalTaskStatus(teammate.status)) {
    injectUserMessageToTeammate(teammate.id, task.prompt, setAppState)
    return
  }
  // 队友不存在，清理孤立 cron
  void removeCronTasks([task.id])
}
```

**团队领导任务**（行 110-114）：
```typescript
const msg = createScheduledTaskFireMessage(
  `Running scheduled task (${formatCronFireTime(new Date())})`
)
setMessages(prev => [...prev, msg])
enqueueForLead(task.prompt)
```

### 关键代码路径

#### 任务入队函数（行 71-82）
```typescript
const enqueueForLead = (prompt: string) =>
  enqueuePendingNotification({
    value: prompt,
    mode: 'prompt',
    priority: 'later',        // 低优先级，在 turn 之间处理
    isMeta: true,             // 系统生成，对 UI 隐藏
    workload: WORKLOAD_CRON,  // 用于计费和工作负载追踪
  })
```

#### 调度器创建（行 84-120）
```typescript
const scheduler = createCronScheduler({
  onFire: enqueueForLead,           // 错过任务回调
  onFireTask: task => {             // 正常触发回调
    if (task.agentId) {
      // 队友路由逻辑
    } else {
      // 领导任务逻辑
    }
  },
  isLoading: () => isLoadingRef.current,  // 最新值读取
  assistantMode,
  getJitterConfig: getCronJitterConfig,
  isKilled: () => !isKairosCronEnabled(),  // 运行时开关
})
```

#### 时间格式化（行 129-139）
```typescript
function formatCronFireTime(d: Date): string {
  return d
    .toLocaleString('en-US', {
      month: 'short',
      day: 'numeric',
      hour: 'numeric',
      minute: '2-digit',
    })
    .replace(/,? at |, /, ' ')
    .replace(/ ([AP]M)/, (_, ampm) => ampm.toLowerCase())
  // 输出示例: "Apr 1 2:30pm"
}
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `../state/AppState.js` | AppState 访问和更新 |
| `../Task.js` | `isTerminalTaskStatus` 类型检查 |
| `../tasks/InProcessTeammateTask/InProcessTeammateTask.js` | 队友任务查找和消息注入 |
| `../tools/ScheduleCronTool/prompt.js` | `isKairosCronEnabled` 功能开关 |
| `../utils/cronScheduler.js` | 调度器核心实现 |
| `../utils/cronTasks.js` | `removeCronTasks` 任务清理 |
| `../utils/cronJitterConfig.js` | 抖动配置获取 |
| `../utils/messageQueueManager.js` | `enqueuePendingNotification` 入队 |
| `../utils/messages.js` | `createScheduledTaskFireMessage` |
| `../utils/workloadContext.js` | `WORKLOAD_CRON` 常量 |

### 外部交互

1. **CronScheduler**: 核心调度器，处理文件监听、定时检查、任务触发
2. **AppState**: 读取任务列表、更新队友状态
3. **消息队列**: 通过 `enqueuePendingNotification` 将任务入队到 REPL 命令队列
4. **队友系统**: 通过 `injectUserMessageToTeammate` 向队友发送消息

---

## 风险、边界与改进建议

### 已知风险

1. **Ref 模式依赖**: 使用 `isLoadingRef` 避免 effect 重新运行，但需要确保 ref 及时更新
2. **队友状态竞争**: `findTeammateTaskByAgentId` 读取的状态可能已过期
3. **孤立任务清理**: 异步 `removeCronTasks` 可能在清理前任务再次触发

### 边界情况

1. **Assistant Mode 变化**: 注释说明 `assistantMode` 在会话生命周期内稳定，但代码依赖此假设
2. **任务列表 ID 变化**: `taskListId` 变更时（如创建团队），调度器不会自动重新指向新目录
3. **并发任务触发**: 多个任务同时触发时的队列顺序
4. **队友快速重启**: 队友重启后 agentId 相同但任务 ID 不同，可能误判为孤立任务

### 改进建议

1. **任务触发确认**: 添加任务触发确认机制，确保任务被正确处理
2. **队友状态缓存**: 缓存队友状态减少状态查询开销
3. **任务优先级**: 支持为不同 cron 任务设置不同优先级
4. **批量任务处理**: 多个任务同时触发时批量处理减少消息数量
5. **任务执行历史**: 记录任务执行历史用于调试和审计
6. **错误重试**: 任务提交失败时自动重试机制

### 测试关注点

1. 任务触发时机准确性（考虑抖动）
2. 队友任务路由正确性
3. 孤立任务清理逻辑
4. isLoading 状态变化时的任务入队行为
5. 功能开关动态切换
6. 组件卸载时的资源清理
