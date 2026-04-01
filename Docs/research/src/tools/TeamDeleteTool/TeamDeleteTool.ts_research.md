# TeamDeleteTool.ts 研究文档

## 场景与职责

`TeamDeleteTool` 是 Claude Code CLI 中 Agent Swarm（智能体集群）功能的核心组件之一，负责在团队协作完成后执行清理工作。当多个 AI Agent 组成的团队完成其任务后，需要有一个机制来安全地解散团队并释放相关资源。

该工具的主要职责包括：
1. **团队资源清理**：删除团队目录和任务目录
2. **Git 工作树清理**：销毁为团队成员创建的 Git worktree
3. **状态重置**：清除团队上下文、颜色分配、消息队列等
4. **安全检查**：确保没有活跃的团队成员时才允许清理

## 功能点目的

### 1. 安全清理机制
- **前置条件检查**：在清理前验证团队中没有活跃的成员（排除 team-lead）
- **优雅关闭依赖**：要求先调用 `requestShutdown` 终止所有队友，防止数据丢失
- **幂等性保证**：通过 `unregisterTeamForSessionCleanup` 防止重复清理

### 2. 资源释放
- 团队配置目录：`~/.claude/teams/{team-name}/`
- 任务数据目录：`~/.claude/tasks/{team-name}/`
- Git worktree：为每个团队成员创建的工作树
- 颜色分配：重置队友颜色映射
- 领导者团队名：清除 `leaderTeamName` 使任务列表 ID 回退到会话 ID

### 3. 分析追踪
- 记录 `tengu_team_deleted` 事件用于分析

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式 - 空对象，无需参数
const inputSchema = lazySchema(() => z.strictObject({}))

// 输出结构
export type Output = {
  success: boolean
  message: string
  team_name?: string
}
```

### 核心流程

```
call() 方法执行流程:
├─ 从 appState 获取 teamName
├─ 如果存在 teamName:
│  ├─ 读取团队配置文件 (readTeamFile)
│  ├─ 过滤出非 team-lead 成员
│  ├─ 检查活跃成员 (isActive !== false)
│  ├─ 如果有活跃成员 → 返回失败
│  ├─ 清理团队目录 (cleanupTeamDirectories)
│  ├─ 注销会话清理 (unregisterTeamForSessionCleanup)
│  ├─ 清除队友颜色 (clearTeammateColors)
│  ├─ 清除领导者团队名 (clearLeaderTeamName)
│  └─ 记录分析事件 (logEvent)
├─ 清除应用状态中的团队上下文和收件箱
└─ 返回操作结果
```

### 安全检查逻辑

```typescript
// 过滤掉 team-lead，只计算非领导成员
const nonLeadMembers = teamFile.members.filter(
  m => m.name !== TEAM_LEAD_NAME,  // TEAM_LEAD_NAME = 'team-lead'
)

// 区分真正活跃的成员和空闲/死亡的成员
// isActive === false 表示空闲（完成回合或崩溃）
const activeMembers = nonLeadMembers.filter(m => m.isActive !== false)

if (activeMembers.length > 0) {
  return {
    data: {
      success: false,
      message: `Cannot cleanup team with ${activeMembers.length} active member(s): ${memberNames}. Use requestShutdown to gracefully terminate teammates first.`,
    },
  }
}
```

### 工具定义配置

```typescript
export const TeamDeleteTool: Tool<InputSchema, Output> = buildTool({
  name: TEAM_DELETE_TOOL_NAME,           // 'TeamDelete'
  searchHint: 'disband a swarm team and clean up',
  maxResultSizeChars: 100_000,
  shouldDefer: true,                      // 延迟执行

  userFacingName() { return '' },         // 空字符串表示不显示给用户

  isEnabled() {
    return isAgentSwarmsEnabled()         // 检查功能开关
  },

  async description() {
    return 'Clean up team and task directories when the swarm is complete'
  },

  async prompt() {
    return getPrompt()                    // 从 prompt.ts 获取
  },

  mapToolResultToToolResultBlockParam(data, toolUseID) {
    // 将结果映射为 Anthropic API 的 tool_result 格式
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: [{ type: 'text', text: jsonStringify(data) }],
    }
  },

  async call(_input, context) { /* ... */ },

  renderToolUseMessage,      // 来自 UI.tsx
  renderToolResultMessage,   // 来自 UI.tsx
})
```

## 关键代码路径与文件引用

### 直接依赖

| 路径 | 用途 |
|------|------|
| `./constants.js` | `TEAM_DELETE_TOOL_NAME` 常量 |
| `./prompt.js` | `getPrompt()` 获取工具提示 |
| `./UI.tsx` | UI 渲染函数 |
| `../../Tool.js` | `Tool`, `ToolDef`, `buildTool` 基础类型 |
| `../../services/analytics/index.js` | `logEvent` 分析日志 |
| `../../utils/agentSwarmsEnabled.js` | `isAgentSwarmsEnabled()` 功能开关 |
| `../../utils/lazySchema.js` | 延迟加载 Zod schema |
| `../../utils/slowOperations.js` | `jsonStringify` 慢操作包装 |
| `../../utils/swarm/constants.js` | `TEAM_LEAD_NAME` |
| `../../utils/swarm/teamHelpers.js` | 核心清理函数 |
| `../../utils/swarm/teammateLayoutManager.js` | 颜色管理 |
| `../../utils/tasks.js` | `clearLeaderTeamName` |

### 被调用方

| 路径 | 用途 |
|------|------|
| `../../tools.ts` | 工具注册，通过 `getTeamDeleteTool()` 懒加载 |
| `../../coordinator/coordinatorMode.ts` | `INTERNAL_WORKER_TOOLS` 集合成员 |
| `../../utils/permissions/classifierDecision.ts` | `SAFE_YOLO_ALLOWLISTED_TOOLS` 白名单 |

## 依赖与外部交互

### 核心工具函数 (`teamHelpers.js`)

```typescript
// 读取团队配置
readTeamFile(teamName: string): TeamFile | null

// 清理团队目录和任务目录
cleanupTeamDirectories(teamName: string): Promise<void>
  ├─ 销毁 Git worktree (git worktree remove --force)
  ├─ 删除团队目录 (~/.claude/teams/{team-name}/)
  └─ 删除任务目录 (~/.claude/tasks/{team-name}/)

// 注销会话清理跟踪
unregisterTeamForSessionCleanup(teamName: string): void
```

### 状态管理

```typescript
// 清除队友颜色分配
clearTeammateColors(): void

// 清除领导者团队名，使 getTaskListId() 回退到会话 ID
clearLeaderTeamName(): void

// 应用状态更新
setAppState(prev => ({
  ...prev,
  teamContext: undefined,
  inbox: { messages: [] },
}))
```

### 功能开关

```typescript
isAgentSwarmsEnabled(): boolean
  ├─ Ant 构建：始终启用
  ├─ 外部构建：需要 CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 或 --agent-teams
  └─ 需要 GrowthBook killswitch 'tengu_amber_flint' 启用
```

## 风险、边界与改进建议

### 风险点

1. **竞态条件**
   - 在检查活跃成员和实际清理之间，成员状态可能发生变化
   - 建议：考虑使用文件锁或原子操作

2. **部分清理失败**
   - 如果 `cleanupTeamDirectories` 部分失败，可能导致资源泄漏
   - 当前实现使用 `force: true` 的 `rm` 操作，但错误仅记录不传播

3. **Team Name 为空**
   - 如果 `appState.teamContext?.teamName` 为 undefined，工具会返回成功但仅清理状态
   - 这可能掩盖配置问题

### 边界情况

1. **重复调用**
   - 通过 `unregisterTeamForSessionCleanup` 防止会话关闭时的重复清理
   - 但工具本身可以被多次调用（如果 teamContext 被重新设置）

2. **活跃成员检测**
   - `isActive === false` 表示空闲，但 undefined/true 都表示活跃
   - 新创建的成员可能还没有 `isActive` 字段

3. **UI 抑制**
   - `renderToolResultMessage` 始终返回 `null`，结果完全抑制
   - 这是设计选择（批量的 shutdown 消息覆盖此消息）

### 改进建议

1. **增强日志记录**
   - 添加更详细的清理步骤日志
   - 记录每个被删除的资源路径

2. **事务性清理**
   - 实现两阶段提交：先标记清理中，再执行清理
   - 支持清理失败时的回滚或重试

3. **配置验证**
   - 在工具调用前验证 teamContext 存在
   - 提供更清晰的错误消息

4. **并发控制**
   - 添加工具级别的锁，防止并发调用导致的状态不一致

5. **测试覆盖**
   - 添加单元测试覆盖各种边界情况
   - 模拟部分失败场景
