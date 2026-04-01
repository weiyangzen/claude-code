# CoordinatorAgentStatus.tsx 研究文档

## 场景与职责

`CoordinatorAgentStatus.tsx`（实际导出名为 `CoordinatorTaskPanel`）是 Claude Code CLI 中用于**显示和管理后台 Agent 任务**的核心 UI 组件。它位于终端界面的底部，在 prompt 输入框下方渲染，为用户提供：

1. **后台任务可视化**：显示所有正在运行或已完成的 local_agent 任务列表
2. **任务状态监控**：实时展示任务运行时长、token 消耗、消息队列状态
3. **快速切换导航**：允许用户通过键盘或鼠标快速切换到特定 Agent 的会话视图
4. **任务生命周期管理**：自动清理已结束的任务，支持手动停止/清除

该组件是 Claude Code 多 Agent 协作功能的关键 UI 入口，让用户能够在主会话和多个子 Agent 之间无缝切换。

## 功能点目的

### 1. 可见任务筛选 (`getVisibleAgentTasks`)
- **目的**：从 AppState 中筛选出需要在面板中显示的任务
- **逻辑**：
  - 过滤条件：`isPanelAgentTask(t) && t.evictAfter !== 0`
  - 排序规则：按 `startTime` 升序排列（先启动的任务在前）
  - `evictAfter` 机制：
    - `undefined`：任务正在运行或被保留，始终显示
    - `timestamp`：任务已结束，显示到该时间点为止
    - `0`：立即隐藏（用户手动清除）

### 2. 任务自动清理机制
- **1秒定时器**：通过 `setInterval` 每秒检查一次任务状态
- **清理条件**：`isPanelAgentTask(t) && (t.evictAfter ?? Infinity) <= now`
- **清理动作**：调用 `evictTerminalTask(t.id, setAppState)` 从 AppState 中移除任务
- **设计考量**：清理逻辑集中在组件内，通过 `tasksRef` 避免闭包问题

### 3. 主行显示 (`MainLine`)
- **功能**：始终显示 "main" 选项，代表返回主会话
- **交互**：点击可调用 `exitTeammateView(setAppState)` 退出 Agent 视图
- **视觉状态**：
  - `isViewed`：当前正在查看主会话（黑色圆点标记）
  - `isSelected`：当前键盘选中状态（显示指针符号）

### 4. Agent 行显示 (`AgentLine`)
每行展示一个 Agent 任务的详细信息：
- **名称**：从 `agentNameRegistry` 反向查找 Agent 名称
- **描述**：优先显示 `progress.summary`，否则显示 `task.description`
- **状态图标**：运行中显示 `▶`，暂停/结束显示 `⏸`
- **运行时长**：动态计算，考虑暂停时间 (`totalPausedMs`)
- **Token 计数**：显示输入/输出 token 数量及方向箭头
- **消息队列**：显示待处理消息数量（如有）
- **操作提示**：选中时显示 `x to stop/clear` 提示

### 5. 选中状态管理
- 从 `AppState` 读取 `coordinatorTaskIndex` 和 `footerSelection`
- 当 `footerSelection === 'tasks'` 时，使用 `coordinatorTaskIndex` 确定选中项
- 索引 0 对应 "main"，索引 1+ 对应具体 Agent 任务

## 具体技术实现

### 关键数据结构

```typescript
// LocalAgentTaskState（来自 LocalAgentTask.js）
interface LocalAgentTaskState {
  id: string;                    // 任务唯一标识
  type: 'local_agent';           // 任务类型
  status: TaskStatus;            // running | pending | completed | failed | killed
  startTime: number;             // 启动时间戳
  endTime?: number;              // 结束时间戳
  totalPausedMs?: number;        // 累计暂停时长
  evictAfter?: number;           // 清理截止时间（undefined=不清理）
  retain: boolean;               // 是否保留（正在查看时）
  description: string;           // 任务描述
  progress?: {
    summary?: string;            // 进度摘要
    tokenCount?: number;         // token 数量
    lastActivity?: 'in' | 'out'; // 最后活动方向
  };
  pendingMessages: Message[];    // 待处理消息队列
  // ... 其他字段
}
```

### 关键流程

#### 任务渲染流程
```
CoordinatorTaskPanel()
  ├── getVisibleAgentTasks(tasks) → 筛选并排序任务
  ├── 1秒定时器设置（如 hasTasks）
  ├── MainLine 渲染（索引 0）
  └── visibleTasks.map() → AgentLine 渲染（索引 1+）
```

#### 任务清理流程
```
setInterval(1000ms)
  ├── 遍历 tasksRef.current
  ├── 检查 evictAfter 是否过期
  └── 调用 evictTerminalTask(taskId, setAppState)
        └── 从 AppState.tasks 中删除对应任务
```

#### 视图切换流程
```
点击 AgentLine
  └── enterTeammateView(taskId, setAppState)
        ├── 设置 viewingAgentTaskId = taskId
        ├── 设置 viewSelectionMode = 'viewing-agent'
        └── 设置 task.retain = true, evictAfter = undefined

点击 MainLine
  └── exitTeammateView(setAppState)
        ├── 设置 viewingAgentTaskId = undefined
        ├── 设置 viewSelectionMode = 'none'
        └── 释放之前的任务（retain=false，设置 evictAfter）
```

### 性能优化

1. **React Compiler 缓存**：大量使用 `_c(n)` 模式进行 memoization
2. **定时器优化**：仅在 `hasTasks` 为 true 时启动定时器
3. **引用稳定性**：使用 `tasksRef` 避免定时器回调中的闭包问题
4. **条件渲染**：无可见任务时返回 `null`，不渲染任何内容

### 布局与样式

- 使用 Ink 组件：`Box`, `Text`, `wrapText`
- 布局方向：`flexDirection="column"`
- 顶部间距：`marginTop={1}`
- 文本截断：使用 `wrapText(displayDescription, availableForDesc, "truncate-end")`
- 宽度计算：动态计算可用宽度，考虑前缀、后缀、token 计数等

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/CoordinatorAgentStatus.tsx`

### 直接依赖
| 导入路径 | 用途 |
|---------|------|
| `../constants/figures.js` | 图标常量（BLACK_CIRCLE, PAUSE_ICON, PLAY_ICON） |
| `../hooks/useTerminalSize.js` | 获取终端尺寸 |
| `../ink/stringWidth.js` | 计算字符串显示宽度 |
| `../ink.js` | Ink UI 组件（Box, Text, wrapText） |
| `../state/AppState.js` | 应用状态类型和 hooks |
| `../state/teammateViewHelpers.js` | 视图切换辅助函数 |
| `../tasks/LocalAgentTask/LocalAgentTask.js` | LocalAgentTaskState 类型 |
| `../utils/format.js` | 格式化函数（formatDuration, formatNumber） |
| `../utils/task/framework.js` | evictTerminalTask 函数 |
| `./tasks/taskStatusUtils.js` | isTerminalStatus 工具函数 |

### 相关依赖文件
- `/home/sansha/Github/claude-code-instructkr/src/state/teammateViewHelpers.ts`
  - `enterTeammateView()`: 进入 Agent 视图
  - `exitTeammateView()`: 退出 Agent 视图
  - `stopOrDismissAgent()`: 停止/清除 Agent
  - `PANEL_GRACE_MS = 30_000`: 任务结束后保留显示的时间

- `/home/sansha/Github/claude-code-instructkr/src/utils/task/framework.ts`
  - `evictTerminalTask()`: 从 AppState 中驱逐已结束的任务
  - `PANEL_GRACE_MS = 30_000`: 与 teammateViewHelpers 保持一致

- `/home/sansha/Github/claude-code-instructkr/src/tasks/types.ts`
  - `TaskState` 联合类型
  - `BackgroundTaskState` 联合类型
  - `isBackgroundTask()`: 判断是否为后台任务

## 依赖与外部交互

### 与 AppState 的交互

```typescript
// 读取的状态
const tasks = useAppState(s => s.tasks);
const viewingAgentTaskId = useAppState(s => s.viewingAgentTaskId);
const agentNameRegistry = useAppState(s => s.agentNameRegistry);
const coordinatorTaskIndex = useAppState(s => s.coordinatorTaskIndex);
const tasksSelected = useAppState(s => s.footerSelection === 'tasks');

// 写入操作（通过 setAppState）
const setAppState = useSetAppState();
```

### 与任务系统的交互

1. **LocalAgentTask**：主要展示的任务类型
2. **任务状态变更**：通过 AppState 的 tasks 对象监听
3. **任务清理**：调用 `evictTerminalTask` 进行清理

### 与视图系统的交互

- `enterTeammateView`: 切换到 Agent 视图时调用
- `exitTeammateView`: 返回主会话时调用
- 这些函数会修改 `viewingAgentTaskId` 和任务的 `retain`/`evictAfter` 状态

## 风险、边界与改进建议

### 已知风险

1. **定时器泄漏风险**
   - 风险：如果组件卸载时定时器未正确清理，可能导致内存泄漏
   - 缓解：useEffect 返回 cleanup 函数清除 interval
   - 现状：代码中已正确处理

2. **任务引用过期**
   - 风险：定时器回调中的任务引用可能过期
   - 缓解：使用 `tasksRef` 始终访问最新状态
   - 现状：代码中已正确处理

3. **性能问题（大量任务）**
   - 风险：如果同时有大量任务运行，每秒遍历所有任务可能消耗性能
   - 现状：通常任务数量较少，不构成问题

### 边界情况

1. **任务在渲染期间被清除**
   - 处理：组件通过 `visibleTasks` 长度检查，无任务时返回 null

2. **终端宽度变化**
   - 处理：使用 `useTerminalSize` 监听尺寸变化，动态调整文本截断

3. **任务描述过长**
   - 处理：使用 `wrapText` 进行截断，保留 `availableForDesc` 计算逻辑

4. **Agent 名称未注册**
   - 处理：`nameByAgentId.get(task.id)` 可能返回 undefined，组件已处理

### 改进建议

1. **虚拟列表优化**
   - 当任务数量很多时，考虑使用虚拟列表只渲染可见项
   - 当前实现会渲染所有可见任务

2. **清理策略可配置**
   - 当前 `PANEL_GRACE_MS` 是硬编码的 30 秒
   - 建议：允许用户配置任务在面板中保留的时长

3. **批量清理优化**
   - 当前每秒遍历所有任务
   - 建议：维护一个按 evictAfter 排序的优先队列，减少遍历次数

4. **更好的空状态处理**
   - 当前无任务时直接返回 null
   - 建议：可考虑显示一个折叠的指示器，提示用户有后台任务功能

5. **TypeScript 类型完善**
   - `useCoordinatorTaskCount` 函数当前返回硬编码的 0
   - 建议：完善该 hook 的实现，或移除如果不需要

### 测试建议

1. **单元测试**：测试 `getVisibleAgentTasks` 的筛选和排序逻辑
2. **集成测试**：测试与 `teammateViewHelpers` 的交互
3. **边界测试**：测试大量任务、快速切换、任务清理等场景
4. **性能测试**：测试长时间运行时的内存使用情况
