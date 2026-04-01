# spawnMultiAgent.ts 深度研究文档

## 场景与职责

`spawnMultiAgent.ts` 是 Claude Code CLI 中**多智能体（Multi-Agent）队友创建**的核心共享模块。它是 `TeammateTool` 和 `AgentTool` 的底层实现基础，负责：

1. **队友生命周期管理**：创建、配置和启动队友（Teammate）实例
2. **多后端支持**：支持三种执行模式：
   - **In-Process 模式**：在同一 Node.js 进程中使用 AsyncLocalStorage 隔离运行
   - **Split-Pane 模式**：在 tmux/iTerm2 分屏中运行（默认）
   - **Separate-Window 模式**：在独立 tmux 窗口中运行（遗留行为）
3. **环境继承**：将父进程的 CLI 标志、权限模式、模型设置等传播给队友
4. **团队管理**：维护团队文件（team file）和 AppState 中的队友状态

## 功能点目的

### 1. 队友创建 (spawnTeammate)

主入口函数，根据配置和当前环境选择适当的创建方式：
- 检查 `isInProcessEnabled()` 决定是否使用进程内模式
- 预检 pane backend 可用性，必要时回退到 in-process 模式
- 支持 `use_splitpane` 参数控制分屏行为

### 2. 模型解析 (resolveTeammateModel)

处理队友模型配置：
- `inherit` 别名：继承队长的模型（解决 gh-31069 问题）
- `undefined`：使用默认模型（Claude Opus）
- 自定义模型：直接使用指定模型

### 3. 唯一名称生成 (generateUniqueTeammateName)

避免团队内名称冲突：
- 检查团队文件中现有成员名称
- 自动添加数字后缀（如 `tester-2`, `tester-3`）

### 4. CLI 标志继承 (buildInheritedCliFlags)

确保队友继承父进程的重要设置：
- 权限模式（`--dangerously-skip-permissions`, `--permission-mode`）
- 模型覆盖（`--model`）
- 设置路径（`--settings`）
- 内联插件（`--plugin-dir`）
- Chrome 标志（`--chrome` / `--no-chrome`）

### 5. 三种 Spawn 处理器

#### handleSpawnSplitPane
- 在分屏视图中创建队友
- 内部 tmux：分割当前窗口（队长左侧 30%，队友右侧 70%）
- 外部：创建 `claude-swarm` 会话，平铺布局
- iTerm2 + it2：使用原生 iTerm2 分屏

#### handleSpawnSeparateWindow
- 在独立 tmux 窗口中创建队友（遗留行为）
- 每个队友有自己的窗口

#### handleSpawnInProcess
- 使用 AsyncLocalStorage 在同一进程中创建队友
- 通过 `spawnInProcessTeammate()` 创建上下文
- 通过 `startInProcessTeammate()` 启动执行循环

## 具体技术实现

### 关键数据结构

```typescript
// Spawn 输出结构
export type SpawnOutput = {
  teammate_id: string        // 格式: "name@team"
  agent_id: string           // 同上
  agent_type?: string        // 自定义智能体类型
  model?: string             // 使用的模型
  name: string               // 显示名称
  color?: string             // 分配的颜色
  tmux_session_name: string  // tmux 会话名
  tmux_window_name: string   // tmux 窗口名
  tmux_pane_id: string       // tmux pane ID
  team_name?: string         // 团队名
  is_splitpane?: boolean     // 是否分屏模式
  plan_mode_required?: boolean // 是否需要计划模式
}

// Spawn 配置
export type SpawnTeammateConfig = {
  name: string
  prompt: string
  team_name?: string
  cwd?: string
  use_splitpane?: boolean
  plan_mode_required?: boolean
  model?: string
  agent_type?: string
  description?: string
  invokingRequestId?: string  // 用于 API 调用链路追踪
}

// 内部输入类型
 type SpawnInput = {
  name: string
  prompt: string
  team_name?: string
  cwd?: string
  use_splitpane?: boolean
  plan_mode_required?: boolean
  model?: string
  agent_type?: string
  description?: string
  invokingRequestId?: string
}
```

### 核心函数实现

#### 1. 模型解析

```typescript
export function resolveTeammateModel(
  inputModel: string | undefined,
  leaderModel: string | null,
): string {
  if (inputModel === 'inherit') {
    return leaderModel ?? getDefaultTeammateModel(leaderModel)
  }
  return inputModel ?? getDefaultTeammateModel(leaderModel)
}

function getDefaultTeammateModel(leaderModel: string | null): string {
  const configured = getGlobalConfig().teammateDefaultModel
  if (configured === null) {
    // 用户选择了 "Default" - 跟随队长
    return leaderModel ?? getHardcodedTeammateModelFallback()
  }
  if (configured !== undefined) {
    return parseUserSpecifiedModel(configured)
  }
  return getHardcodedTeammateModelFallback()
}
```

#### 2. CLI 标志构建

```typescript
function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []
  
  // 权限模式传播（plan mode 优先于 bypass permissions）
  if (planModeRequired) {
    // 不继承 bypass 权限
  } else if (permissionMode === 'bypassPermissions' || getSessionBypassPermissionsMode()) {
    flags.push('--dangerously-skip-permissions')
  } else if (permissionMode === 'acceptEdits') {
    flags.push('--permission-mode acceptEdits')
  } else if (permissionMode === 'auto') {
    flags.push('--permission-mode auto')
  }
  
  // 模型覆盖
  const modelOverride = getMainLoopModelOverride()
  if (modelOverride) {
    flags.push(`--model ${quote([modelOverride])}`)
  }
  
  // 设置路径
  const settingsPath = getFlagSettingsPath()
  if (settingsPath) {
    flags.push(`--settings ${quote([settingsPath])}`)
  }
  
  // 内联插件
  const inlinePlugins = getInlinePlugins()
  for (const pluginDir of inlinePlugins) {
    flags.push(`--plugin-dir ${quote([pluginDir])}`)
  }
  
  // Chrome 标志
  const chromeFlagOverride = getChromeFlagOverride()
  if (chromeFlagOverride === true) {
    flags.push('--chrome')
  } else if (chromeFlagOverride === false) {
    flags.push('--no-chrome')
  }
  
  return flags.join(' ')
}
```

#### 3. Split-Pane Spawn 流程

```typescript
async function handleSpawnSplitPane(input: SpawnInput, context: ToolUseContext): Promise<{ data: SpawnOutput }> {
  // 1. 解析模型
  const model = resolveTeammateModel(input.model, getAppState().mainLoopModel)
  
  // 2. 获取/验证团队名
  const teamName = input.team_name || appState.teamContext?.teamName
  if (!teamName) throw new Error('team_name is required...')
  
  // 3. 生成唯一名称
  const uniqueName = await generateUniqueTeammateName(name, teamName)
  const sanitizedName = sanitizeAgentName(uniqueName)
  const teammateId = formatAgentId(sanitizedName, teamName)
  
  // 4. 检测后端（tmux/iTerm2）
  let detectionResult = await detectAndGetBackend()
  
  // 5. iTerm2 需要 it2 设置
  if (detectionResult.needsIt2Setup && context.setToolJSX) {
    const setupResult = await new Promise<'installed' | 'use-tmux' | 'cancelled'>(resolve => {
      context.setToolJSX!({
        jsx: React.createElement(It2SetupPrompt, { onDone: resolve, tmuxAvailable }),
        shouldHidePromptInput: true,
      })
    })
    if (setupResult === 'cancelled') throw new Error('Teammate spawn cancelled')
    // 重新检测后端
    resetBackendDetection()
    detectionResult = await detectAndGetBackend()
  }
  
  // 6. 分配颜色
  const teammateColor = assignTeammateColor(teammateId)
  
  // 7. 创建 pane
  const { paneId, isFirstTeammate } = await createTeammatePaneInSwarmView(sanitizedName, teammateColor)
  
  // 8. 构建启动命令
  const binaryPath = getTeammateCommand()
  const teammateArgs = [
    `--agent-id ${quote([teammateId])}`,
    `--agent-name ${quote([sanitizedName])}`,
    `--team-name ${quote([teamName])}`,
    `--agent-color ${quote([teammateColor])}`,
    `--parent-session-id ${quote([getSessionId()])}`,
    plan_mode_required ? '--plan-mode-required' : '',
    agent_type ? `--agent-type ${quote([agent_type])}` : '',
  ].filter(Boolean).join(' ')
  
  const inheritedFlags = buildInheritedCliFlags({ planModeRequired, permissionMode })
  const envStr = buildInheritedEnvVars()
  const spawnCommand = `cd ${quote([workingDir])} && env ${envStr} ${quote([binaryPath])} ${teammateArgs}${flagsStr}`
  
  // 9. 发送命令到 pane
  await sendCommandToPane(paneId, spawnCommand, !insideTmux)
  
  // 10. 注册任务和团队文件
  registerOutOfProcessTeammateTask(...)
  await writeTeamFileAsync(teamName, teamFile)
  
  // 11. 通过 mailbox 发送初始指令
  await writeToMailbox(sanitizedName, { from: TEAM_LEAD_NAME, text: prompt, timestamp: ... }, teamName)
  
  return { data: {...} }
}
```

#### 4. In-Process Spawn 流程

```typescript
async function handleSpawnInProcess(input: SpawnInput, context: ToolUseContext): Promise<{ data: SpawnOutput }> {
  // 1. 解析模型和验证团队
  const model = resolveTeammateModel(input.model, getAppState().mainLoopModel)
  const teamName = input.team_name || appState.teamContext?.teamName
  
  // 2. 生成 ID 和颜色
  const uniqueName = await generateUniqueTeammateName(name, teamName)
  const sanitizedName = sanitizeAgentName(uniqueName)
  const teammateId = formatAgentId(sanitizedName, teamName)
  const teammateColor = assignTeammateColor(teammateId)
  
  // 3. 查找自定义智能体定义
  let agentDefinition: CustomAgentDefinition | undefined
  if (agent_type) {
    const allAgents = context.options.agentDefinitions.activeAgents
    const foundAgent = allAgents.find(a => a.agentType === agent_type)
    if (foundAgent && isCustomAgent(foundAgent)) {
      agentDefinition = foundAgent
    }
  }
  
  // 4. 创建 in-process 队友
  const config: InProcessSpawnConfig = { name: sanitizedName, teamName, prompt, color: teammateColor, planModeRequired, model }
  const result = await spawnInProcessTeammate(config, context)
  
  if (!result.success) throw new Error(result.error)
  
  // 5. 启动执行循环
  if (result.taskId && result.teammateContext && result.abortController) {
    startInProcessTeammate({
      identity: { agentId: teammateId, agentName: sanitizedName, teamName, color: teammateColor, planModeRequired, parentSessionId },
      taskId: result.taskId,
      prompt,
      description: input.description,
      model,
      agentDefinition,
      teammateContext: result.teammateContext,
      toolUseContext: { ...context, messages: [] },  // 清空 messages 避免内存泄漏
      abortController: result.abortController,
      invokingRequestId: input.invokingRequestId,
    })
  }
  
  // 6. 更新 AppState（自动注册队长如果需要）
  setAppState(prev => {
    const needsLeaderSetup = !prev.teamContext?.leadAgentId
    // ... 构建 teammates map
  })
  
  // 7. 注册到团队文件
  await writeTeamFileAsync(teamName, teamFile)
  
  // 注意：in-process 队友不通过 mailbox 接收初始指令
  return { data: {...} }
}
```

#### 5. 任务注册（Out-of-Process）

```typescript
function registerOutOfProcessTeammateTask(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  { teammateId, sanitizedName, teamName, teammateColor, prompt, plan_mode_required, paneId, insideTmux, backendType, toolUseId }: {...}
): void {
  const taskId = generateTaskId('in_process_teammate')
  const description = `${sanitizedName}: ${prompt.substring(0, 50)}...`
  const abortController = new AbortController()
  
  const taskState: InProcessTeammateTaskState = {
    ...createTaskStateBase(taskId, 'in_process_teammate', description, toolUseId),
    type: 'in_process_teammate',
    status: 'running',
    identity: { agentId: teammateId, agentName: sanitizedName, teamName, color: teammateColor, planModeRequired, parentSessionId: getSessionId() },
    prompt,
    abortController,
    awaitingPlanApproval: false,
    permissionMode: planModeRequired ? 'plan' : 'default',
    isIdle: false,
    shutdownRequested: false,
    lastReportedToolCount: 0,
    lastReportedTokenCount: 0,
    pendingUserMessages: [],
  }
  
  registerTask(taskState, setAppState)
  
  // 监听 abort 信号，杀死 pane
  abortController.signal.addEventListener('abort', () => {
    if (isPaneBackend(backendType)) {
      void getBackendByType(backendType).killPane(paneId, !insideTmux)
    }
  }, { once: true })
}
```

## 关键代码路径与文件引用

### 调用方（入口点）

| 文件 | 调用方式 | 用途 |
|------|----------|------|
| `src/tools/TeammateTool/TeammateTool.tsx` | `spawnTeammate(config, context)` | 创建队友 |
| `src/tools/AgentTool/AgentTool.tsx` | `spawnTeammate(config, context)` | 创建智能体 |

### 被调用方（依赖）

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.ts` | `getSessionId`, `getMainLoopModelOverride`, `getFlagSettingsPath`, `getInlinePlugins`, `getChromeFlagOverride`, `getSessionBypassPermissionsMode` | 获取会话状态和 CLI 标志 |
| `src/state/AppState.ts` | `AppState` 类型 | 类型定义 |
| `src/Task.ts` | `createTaskStateBase`, `generateTaskId` | 任务状态创建 |
| `src/Tool.ts` | `ToolUseContext` 类型 | 类型定义 |
| `src/tasks/InProcessTeammateTask/types.ts` | `InProcessTeammateTaskState` 类型 | 类型定义 |
| `src/utils/agentId.ts` | `formatAgentId` | 生成智能体 ID |
| `src/utils/bash/shellQuote.ts` | `quote` | Shell 参数转义 |
| `src/utils/bundledMode.ts` | `isInBundledMode` | 检测打包模式 |
| `src/utils/config.ts` | `getGlobalConfig` | 获取全局配置 |
| `src/utils/cwd.ts` | `getCwd` | 获取当前工作目录 |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrow` | 执行子进程 |
| `src/utils/model/model.ts` | `parseUserSpecifiedModel` | 解析模型字符串 |
| `src/utils/permissions/PermissionMode.ts` | `PermissionMode` 类型 | 类型定义 |
| `src/utils/swarm/backends/detection.ts` | `isTmuxAvailable` | 检测 tmux 可用性 |
| `src/utils/swarm/backends/registry.ts` | `detectAndGetBackend`, `getBackendByType`, `isInProcessEnabled`, `markInProcessFallback`, `resetBackendDetection` | 后端检测和管理 |
| `src/utils/swarm/backends/teammateModeSnapshot.ts` | `getTeammateModeFromSnapshot` | 获取队友模式 |
| `src/utils/swarm/backends/types.ts` | `BackendType`, `isPaneBackend` | 类型定义 |
| `src/utils/swarm/constants.ts` | `SWARM_SESSION_NAME`, `TEAM_LEAD_NAME`, `TEAMMATE_COMMAND_ENV_VAR`, `TMUX_COMMAND` | 常量定义 |
| `src/utils/swarm/It2SetupPrompt.tsx` | `It2SetupPrompt` | iTerm2 设置提示 |
| `src/utils/swarm/inProcessRunner.ts` | `startInProcessTeammate` | 启动 in-process 队友 |
| `src/utils/swarm/spawnInProcess.ts` | `spawnInProcessTeammate`, `InProcessSpawnConfig` | 创建 in-process 队友 |
| `src/utils/swarm/spawnUtils.ts` | `buildInheritedEnvVars` | 构建继承的环境变量 |
| `src/utils/swarm/teamHelpers.ts` | `readTeamFileAsync`, `sanitizeAgentName`, `sanitizeName`, `writeTeamFileAsync` | 团队文件操作 |
| `src/utils/swarm/teammateLayoutManager.ts` | `assignTeammateColor`, `createTeammatePaneInSwarmView`, `enablePaneBorderStatus`, `isInsideTmux`, `sendCommandToPane` | Pane 布局管理 |
| `src/utils/swarm/teammateModel.ts` | `getHardcodedTeammateModelFallback` | 获取默认模型 |
| `src/utils/task/framework.ts` | `registerTask` | 注册任务 |
| `src/utils/teammateMailbox.ts` | `writeToMailbox` | Mailbox 写入 |
| `src/tools/AgentTool/loadAgentsDir.ts` | `CustomAgentDefinition`, `isCustomAgent` | 自定义智能体类型 |

### 代码路径示例

```
TeammateTool.tsx (用户调用 teammate 工具)
  ↓
spawnTeammate(config, context)
  ↓
handleSpawn(input, context)
  ↓
检查 isInProcessEnabled()
  ├─ true → handleSpawnInProcess(input, context)
  │         ↓
  │         spawnInProcessTeammate(config, context)
  │         ↓
  │         startInProcessTeammate({...})  // 启动执行循环
  │
  └─ false → detectAndGetBackend()  // 预检后端可用性
             ↓
             如果失败且 mode === 'auto' → markInProcessFallback() → handleSpawnInProcess
             ↓
             handleSpawnSplitPane(input, context)
             ↓
             1. detectAndGetBackend()  // 检测 tmux/iTerm2
             2. createTeammatePaneInSwarmView(name, color)  // 创建 pane
             3. buildInheritedCliFlags()  // 构建 CLI 标志
             4. sendCommandToPane(paneId, spawnCommand)  // 发送启动命令
             5. registerOutOfProcessTeammateTask(...)  // 注册任务
             6. writeToMailbox(...)  // 发送初始指令
```

## 依赖与外部交互

### 内部依赖架构

```
spawnMultiAgent.ts
├── 状态管理
│   ├── src/bootstrap/state.ts (会话状态、CLI 标志)
│   └── src/state/AppState.ts (AppState 类型)
├── 任务系统
│   ├── src/Task.ts (任务状态基础)
│   ├── src/tasks/InProcessTeammateTask/types.ts (队友任务状态)
│   └── src/utils/task/framework.ts (任务注册)
├── 后端系统
│   ├── src/utils/swarm/backends/registry.ts (后端检测)
│   ├── src/utils/swarm/backends/detection.ts (tmux 检测)
│   └── src/utils/swarm/backends/types.ts (类型定义)
├── Pane 管理
│   ├── src/utils/swarm/teammateLayoutManager.ts (布局管理)
│   └── src/utils/swarm/spawnInProcess.ts (in-process 创建)
├── 团队管理
│   └── src/utils/swarm/teamHelpers.ts (团队文件读写)
├── 通信
│   └── src/utils/teammateMailbox.ts (Mailbox 系统)
└── 工具
    ├── src/utils/bash/shellQuote.ts (Shell 转义)
    ├── src/utils/execFileNoThrow.ts (进程执行)
    └── src/utils/model/model.ts (模型解析)
```

### 外部系统交互

1. **tmux**
   - 通过 `execFileNoThrow(TMUX_COMMAND, [...])` 执行 tmux 命令
   - 创建/管理会话、窗口、panes
   - 发送按键命令到 pane

2. **iTerm2 (macOS)**
   - 通过 `it2` CLI 工具控制 iTerm2
   - 创建原生分屏（比 tmux 更好的 macOS 集成）

3. **文件系统**
   - 团队文件存储在 `.claude/teams/<teamName>.json`
   - 通过 `readTeamFileAsync` / `writeTeamFileAsync` 读写

## 风险、边界与改进建议

### 风险点

1. **后端检测失败**
   - 风险：iTerm2 用户未安装 `it2` CLI，且未安装 tmux
   - 缓解：`handleSpawn` 中预检后端，失败时回退到 in-process 模式（仅 auto 模式）
   - 代码体现：
     ```typescript
     try {
       await detectAndGetBackend()
     } catch (error) {
       if (getTeammateModeFromSnapshot() !== 'auto') throw error
       markInProcessFallback()
       return handleSpawnInProcess(input, context)
     }
     ```

2. **名称冲突**
   - 风险：团队中已存在同名队友
   - 缓解：`generateUniqueTeammateName` 自动添加数字后缀
   - 边界：仅检查团队文件中的成员，不检查正在创建中的

3. **模型继承问题**
   - 风险：`inherit` 被直接传递给 `--model` 标志导致错误（gh-31069）
   - 缓解：`resolveTeammateModel` 将 `inherit` 解析为队长模型或默认值

4. **权限继承风险**
   - 风险：队友继承 `--dangerously-skip-permissions` 可能导致安全问题
   - 缓解：`plan_mode_required` 时阻止继承 bypass 权限
   - 代码体现：
     ```typescript
     if (planModeRequired) {
       // Don't inherit bypass permissions when plan mode is required
     }
     ```

5. **循环依赖**
   - 风险：动态导入 `sessionStorage.ts` 可能导致循环依赖
   - 缓解：使用 `void import(...).then(...)` 异步处理

### 边界条件

1. **非交互式会话**
   - `isInProcessEnabled()` 在 `getIsNonInteractiveSession()` 时强制返回 true
   - 因为 tmux 队友在没有终端 UI 时无意义

2. **团队名缺失**
   - 如果 `input.team_name` 和 `appState.teamContext?.teamName` 都为空
   - 抛出错误要求先调用 `spawnTeam`

3. **团队文件不存在**
   - 如果 `readTeamFileAsync(teamName)` 返回 null
   - 抛出错误 `"Team "${teamName}" does not exist. Call spawnTeam first...`

4. **iTerm2 设置取消**
   - 用户在 `It2SetupPrompt` 中选择取消
   - 抛出错误 `'Teammate spawn cancelled - iTerm2 setup required'`

### 改进建议

1. **增强后端检测**
   - 当前：仅检测 tmux 和 iTerm2
   - 建议：添加对 Windows Terminal、VS Code 终端等的支持

2. **改进错误恢复**
   - 当前：pane 创建失败时抛出错误
   - 建议：自动回退到 in-process 模式，并记录警告

3. **优化资源管理**
   - 当前：队友异常退出时可能留下僵尸 pane
   - 建议：添加 pane 健康检查，自动清理死亡的 pane

4. **增强模型继承**
   - 当前：仅支持 `inherit` 别名
   - 建议：支持更复杂的模型策略，如 `leader-model-tier`（继承队长模型级别而非具体模型）

5. **改进团队文件一致性**
   - 当前：团队文件和 AppState 可能不一致（如崩溃后）
   - 建议：启动时验证团队文件与 AppState 的一致性

6. **添加更多测试覆盖**
   - 建议测试场景：
     - 后端检测失败后的回退
     - 名称冲突解决
     - CLI 标志继承的各种组合
     - iTerm2 设置流程

7. **性能优化**
   - 当前：每次 spawn 都读取团队文件
   - 建议：添加团队文件缓存，减少 I/O

8. **安全增强**
   - 当前：通过 shell 命令字符串传递参数
   - 建议：使用参数数组传递，避免 shell 注入风险
