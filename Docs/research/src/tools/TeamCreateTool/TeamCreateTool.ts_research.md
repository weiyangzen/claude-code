# TeamCreateTool.ts 研究文档

## 场景与职责

`TeamCreateTool.ts` 是 Claude Code 多智能体协同（Agent Swarm/Teammate）系统的入口工具，负责在用户请求创建团队时初始化一个多智能体工作组。它是整个 Swarm 功能链路的起点：只有先通过 `TeamCreate` 成功创建团队，后续才能使用 `AgentTool` 招募队友、`TaskCreateTool/TaskUpdateTool` 分配任务、`SendMessageTool` 进行团队通信等。

该工具面向的是当前会话的"领导者"（Leader）角色。每个领导者同时只能管理一个团队；如果已存在团队，则必须先用 `TeamDelete` 解散当前团队，才能创建新团队。

## 功能点目的

1. **团队唯一性保障**：接受用户提供的 `team_name`，若该名称已被占用，则自动生成一个随机词组 slug（如 `gleaming-brewing-phoenix`）作为替代，避免硬失败。
2. **领导者身份注册**：为当前会话生成确定性的 `leadAgentId`（格式 `team-lead@teamName`），并将其写入团队配置文件，使后续队友能够通过读取配置文件发现领导者。
3. **任务列表绑定**：团队与任务列表 1:1 绑定（Team = TaskList）。创建团队时会同步初始化任务目录，并设置 `leaderTeamName`，确保领导者的任务操作与队友看到的是同一个任务空间。
4. **会话级生命周期管理**：将新创建的团队注册到会话清理集合中，确保会话异常退出时能够自动清理遗留的团队目录和任务目录（解决 gh-32730 遗留问题）。
5. **分析埋点**：记录 `tengu_team_created` 事件，用于追踪团队创建频率、队友模式（tmux / in-process）等。

## 具体技术实现

### 输入 Schema

使用 `zod/v4` 的 `strictObject` 定义，通过 `lazySchema` 延迟构造以避免模块加载时立即求值：

```ts
{
  team_name: string,      // 必填：团队名称
  description?: string,   // 可选：团队描述/用途
  agent_type?: string,    // 可选：团队领导的角色类型
}
```

### 核心执行流程（`call` 方法）

1. **单团队限制检查**
   - 从 `context.getAppState()` 读取 `appState.teamContext.teamName`。
   - 若已存在，抛出 `Error`："Already leading team ... A leader can only manage one team at a time."

2. **名称去重**
   - 调用 `generateUniqueTeamName(providedName)`：
     - 使用同步的 `readTeamFile(providedName)` 检查 `~/.claude/teams/{sanitizedName}/config.json` 是否存在。
     - 若存在，调用 `generateWordSlug()` 生成三段式随机词组（`adjective-verb-noun`）。

3. **构建领导者元数据**
   - `leadAgentId = formatAgentId(TEAM_LEAD_NAME, finalTeamName)` → `team-lead@finalTeamName`
   - `leadAgentType = agent_type || TEAM_LEAD_NAME`
   - `leadModel = parseUserSpecifiedModel(appState.mainLoopModelForSession ?? appState.mainLoopModel ?? getDefaultMainLoopModel())`

4. **写入团队配置文件**
   - 构造 `TeamFile` 对象：
     ```ts
     {
       name: finalTeamName,
       description,
       createdAt: Date.now(),
       leadAgentId,
       leadSessionId: getSessionId(),
       members: [{
         agentId: leadAgentId,
         name: TEAM_LEAD_NAME,
         agentType: leadAgentType,
         model: leadModel,
         joinedAt: Date.now(),
         tmuxPaneId: '',
         cwd: getCwd(),
         subscriptions: [],
       }]
     }
     ```
   - 调用 `writeTeamFileAsync(finalTeamName, teamFile)` 异步写入磁盘。
   - 调用 `registerTeamForSessionCleanup(finalTeamName)` 将会话与团队关联，用于退出时清理。

5. **初始化任务系统**
   - `taskListId = sanitizeName(finalTeamName)`
   - `resetTaskList(taskListId)`：加锁清空旧任务文件，并记录 high-water mark 防止 ID 复用。
   - `ensureTasksDir(taskListId)`：确保 `~/.claude/tasks/{taskListId}/` 目录存在。
   - `setLeaderTeamName(sanitizeName(finalTeamName))`：设置模块级变量 `leaderTeamName`，使 `getTaskListId()` 后续返回团队名而非 session ID。

6. **更新全局 AppState**
   - 通过 `context.setAppState` 写入 `teamContext`：
     - `teamName`, `teamFilePath`, `leadAgentId`
     - `teammates` 映射表中先注册领导者自身，包含 `assignTeammateColor(leadAgentId)` 分配的颜色。
   - 注释明确说明：**不设置 `CLAUDE_CODE_AGENT_ID` 环境变量**，因为领导者不是"teammate"，`isTeammate()` 必须返回 false，否则会导致收件箱轮询逻辑异常。

7. **返回结果**
   ```ts
   { team_name, team_file_path, lead_agent_id }
   ```

### 工具元数据配置

- `name`: `TEAM_CREATE_TOOL_NAME`（值为 `'TeamCreate'`）
- `searchHint`: `'create a multi-agent swarm team'`
- `shouldDefer: true`：该工具被延迟加载，模型需先通过 `ToolSearch` 发现它才能调用。
- `maxResultSizeChars: 100_000`
- `isEnabled()`: 委托 `isAgentSwarmsEnabled()`，受功能开关控制。
- `validateInput()`: 校验 `team_name` 非空。
- `toAutoClassifierInput()`: 返回 `input.team_name` 用于自动分类器。
- `mapToolResultToToolResultBlockParam()`: 将结果 JSON 序列化为 text 类型的 tool_result。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 本文件，工具主实现 |
| `src/tools/TeamCreateTool/constants.ts` | `TEAM_CREATE_TOOL_NAME` 常量 |
| `src/tools/TeamCreateTool/prompt.ts` | 模型 prompt（工具使用说明） |
| `src/tools/TeamCreateTool/UI.tsx` | `renderToolUseMessage` 实现 |
| `src/Tool.ts` | `Tool`、`ToolDef`、`buildTool` 类型与工厂 |
| `src/tools.ts` | 通过 lazy `require()` 注册本工具到全局 tools 列表（打破循环依赖） |
| `src/utils/swarm/teamHelpers.ts` | `readTeamFile`、`writeTeamFileAsync`、`registerTeamForSessionCleanup`、`sanitizeName`、`getTeamFilePath` |
| `src/utils/swarm/constants.ts` | `TEAM_LEAD_NAME`（`'team-lead'`） |
| `src/utils/swarm/teammateLayoutManager.ts` | `assignTeammateColor` |
| `src/utils/swarm/backends/registry.ts` | `getResolvedTeammateMode` |
| `src/utils/tasks.ts` | `resetTaskList`、`ensureTasksDir`、`setLeaderTeamName` |
| `src/utils/agentId.ts` | `formatAgentId` |
| `src/utils/agentSwarmsEnabled.ts` | `isAgentSwarmsEnabled`（功能开关） |
| `src/utils/model/model.ts` | `parseUserSpecifiedModel`、`getDefaultMainLoopModel` |
| `src/utils/words.ts` | `generateWordSlug` |
| `src/utils/cwd.ts` | `getCwd` |
| `src/bootstrap/state.ts` | `getSessionId` |
| `src/services/analytics/index.ts` | `logEvent` |
| `src/state/AppStateStore.ts` | `AppState.teamContext` 类型定义 |

## 依赖与外部交互

### 调用方

- **`src/tools.ts`**：通过 `getTeamCreateTool()` 的 getter + `require()` 动态引入，打破 `tools.ts → TeamCreateTool → ... → tools.ts` 的循环依赖。
- **`src/coordinator/coordinatorMode.ts`**：将 `TEAM_CREATE_TOOL_NAME` 列入 `INTERNAL_WORKER_TOOLS`，在 Coordinator 模式下限制可用工具集。
- **`src/utils/swarm/inProcessRunner.ts`**：引用 `TEAM_CREATE_TOOL_NAME` 常量，用于 in-process 队友的权限/工具白名单。
- **`src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`**：引用常量，用于 plan mode 退出时的工具上下文判断。
- **`src/utils/permissions/classifierDecision.ts`**：将 `TEAM_CREATE_TOOL_NAME` 加入 `SAFE_YOLO_ALLOWLISTED_TOOLS`，使自动模式分类器跳过对该工具的安全检查。
- **`src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx`**：引用常量用于 UI 渲染中的工具名匹配。

### 被调用方/依赖模块

- **文件系统**：通过 `teamHelpers.ts` 读写 `~/.claude/teams/{team}/config.json`；通过 `tasks.ts` 操作 `~/.claude/tasks/{team}/`。
- **状态管理**：通过 `ToolUseContext.setAppState` 更新全局 `AppState.teamContext`。
- **分析系统**：调用 `logEvent('tengu_team_created', {...})` 进行埋点。
- **模型解析**：调用 `parseUserSpecifiedModel` 获取领导者当前使用的模型名称。

## 风险、边界与改进建议

### 风险与边界

1. **同步文件读取阻塞事件循环**
   - `generateUniqueTeamName` 中调用 `readTeamFile`（sync 版本）进行名称冲突检查。虽然团队配置文件很小，但在高频并发场景下仍可能短暂阻塞事件循环。

2. **单团队限制无持久化校验**
   - `call` 方法仅检查内存中的 `appState.teamContext`，如果进程重启或状态被意外清空，理论上可能绕过该限制。不过实际场景中 `appState` 是单会话生命周期内的唯一状态源，风险较低。

3. **名称冲突时的静默重命名**
   - 当用户提供的 `team_name` 已存在时，系统会自动生成随机名称而不告知用户原始名称被占用。这可能导致用户困惑，但避免了硬失败。

4. **不设置 `CLAUDE_CODE_AGENT_ID` 的隐式约定**
   - 代码注释详细解释了为什么不设置环境变量，但这是一个关键的隐式约定。如果未来有人误加，会破坏 `isTeammate()` 的判定，导致领导者被当作普通队友，进而引发收件箱轮询和消息路由错误。

5. **任务列表与团队名的紧耦合**
   - `setLeaderTeamName` 使用模块级变量，意味着同一进程内无法同时管理多个团队。这与"一个领导者只能有一个团队"的产品设计一致，但在测试并行化时需要特别注意状态隔离。

### 改进建议

1. **异步化名称冲突检查**
   - 将 `generateUniqueTeamName` 改为 async，使用 `readTeamFileAsync` 替代 sync 版本，避免阻塞事件循环。

2. **增强结果反馈**
   - 当发生自动重命名时，可在返回的 `Output` 中增加一个 `wasRenamed: boolean` 字段，或在 `renderToolResultMessage` 中向用户展示提示。

3. **统一状态初始化封装**
   - `call` 方法中涉及文件系统、任务系统、AppState、分析埋点等多个副作用，可考虑提取一个 `initializeTeam` 纯业务函数，使 `call` 方法更薄、更易单元测试。

4. **补充单元测试**
   - 当前仓库中未找到针对 `TeamCreateTool` 的专项测试。建议补充：
     - 正常创建团队的端到端模拟（mock 文件系统 + AppState）。
     - 名称冲突时的自动重命名逻辑。
     - 单团队限制的校验逻辑。
     - `TeamFile` 序列化与 `AppState.teamContext` 更新的一致性断言。
