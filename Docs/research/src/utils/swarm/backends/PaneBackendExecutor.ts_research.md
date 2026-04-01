# PaneBackendExecutor.ts 深度研究文档

## 场景与职责

PaneBackendExecutor.ts 实现了 **PaneBackend 到 TeammateExecutor 的适配器模式**，将底层窗口后端（tmux/iTerm2）包装为统一的 `TeammateExecutor` 接口。这是 Agent Swarm 架构中的关键抽象层，使上层代码无需关心具体使用的是 tmux 还是 iTerm2。

**核心定位：**
- 作为 PaneBackend（tmux/iTerm2）的统一包装器
- 实现 TeammateExecutor 接口，与 InProcessBackend 接口兼容
- 处理命令构建、环境变量传递、清理注册等通用逻辑

**架构意义：**
```
上层代码 (TeammateTool)
    ↓
TeammateExecutor 接口
    ↓
├─ InProcessBackend（直接实现）
└─ PaneBackendExecutor（包装 PaneBackend）
        ↓
    ├─ TmuxBackend
    └─ ITermBackend
```

---

## 功能点目的

### 1. 后端适配与包装
- 将 `PaneBackend` 接口适配为 `TeammateExecutor` 接口
- 统一处理两种后端类型的差异

### 2. 队友创建与启动
- `spawn()`: 创建窗口分屏并启动 Claude CLI 进程
  - 分配队友颜色
  - 创建窗口分屏
  - 构建启动命令（含身份标识、继承的 CLI 标志）
  - 发送命令到窗口
  - 注册清理处理器
  - 发送初始提示到邮箱

### 3. 消息传递
- `sendMessage()`: 通过文件邮箱向队友发送消息
- 与 InProcessBackend 使用相同的邮箱机制

### 4. 生命周期控制
- `terminate()`: 发送优雅关闭请求（通过邮箱）
- `kill()`: 强制关闭窗口分屏
- `isActive()`: 检查队友是否活跃（基于内部映射）

### 5. 命令构建
- 构建 Claude CLI 启动命令
- 继承 leader 的 CLI 标志（权限模式、模型、设置等）
- 转发必要的环境变量

---

## 具体技术实现

### 关键数据结构

```typescript
class PaneBackendExecutor implements TeammateExecutor {
  readonly type: BackendType
  private backend: PaneBackend
  private context: ToolUseContext | null = null
  
  // 跟踪已创建的队友
  private spawnedTeammates: Map<string, { 
    paneId: string;      // 窗口 ID
    insideTmux: boolean  // 是否在 tmux 内部
  }>
  
  private cleanupRegistered = false
}
```

### 核心流程

#### 1. 队友创建流程（spawn）

```
1. 验证 ToolUseContext 已设置
2. 分配队友颜色（assignTeammateColor）
3. 创建窗口分屏（backend.createTeammatePaneInSwarmView）
4. 检测是否在 tmux 内部（isInsideTmux）
5. 如果是首个队友且在 tmux 内，启用窗口边框状态
6. 构建启动命令：
   a. 获取二进制路径（getTeammateCommand）
   b. 构建队友身份参数（--agent-id, --agent-name, --team-name, --agent-color, --parent-session-id）
   c. 构建继承的 CLI 标志（buildInheritedCliFlags）
   d. 处理模型覆盖
   e. 构建环境变量（buildInheritedEnvVars）
   f. 组合完整命令：cd <cwd> && env <vars> <binary> <args> <flags>
7. 发送命令到窗口（backend.sendCommandToPane）
8. 跟踪队友信息（spawnedTeammates Map）
9. 注册清理处理器（首次创建时）
10. 发送初始提示到邮箱（writeToMailbox）
```

**关键代码（行 79-209）：**

```typescript
async spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult> {
  const agentId = formatAgentId(config.name, config.teamName)
  
  // 1. 分配颜色
  const teammateColor = config.color ?? assignTeammateColor(agentId)
  
  // 2. 创建窗口
  const { paneId, isFirstTeammate } = await this.backend.createTeammatePaneInSwarmView(
    config.name, teammateColor
  )
  
  // 3. 检测环境
  const insideTmux = await isInsideTmux()
  
  // 4. 启用边框状态（tmux 首个队友）
  if (isFirstTeammate && insideTmux) {
    await this.backend.enablePaneBorderStatus()
  }
  
  // 5. 构建命令
  const binaryPath = getTeammateCommand()
  const teammateArgs = [
    `--agent-id ${quote([agentId])}`,
    `--agent-name ${quote([config.name])}`,
    `--team-name ${quote([config.teamName])}`,
    `--agent-color ${quote([teammateColor])}`,
    `--parent-session-id ${quote([config.parentSessionId || getSessionId()])}`,
    config.planModeRequired ? '--plan-mode-required' : '',
  ].filter(Boolean).join(' ')
  
  // 构建继承的标志
  let inheritedFlags = buildInheritedCliFlags({...})
  if (config.model) {
    // 移除继承的 --model，添加队友特定的模型
    inheritedFlags = inheritedFlags
      .split(' ')
      .filter((flag, i, arr) => flag !== '--model' && arr[i - 1] !== '--model')
      .join(' ')
    inheritedFlags = `${inheritedFlags} --model ${quote([config.model])}`
  }
  
  const envStr = buildInheritedEnvVars()
  const spawnCommand = `cd ${quote([workingDir])} && env ${envStr} ${quote([binaryPath])} ${teammateArgs}${flagsStr}`
  
  // 6. 发送命令
  await this.backend.sendCommandToPane(paneId, spawnCommand, !insideTmux)
  
  // 7. 跟踪队友
  this.spawnedTeammates.set(agentId, { paneId, insideTmux })
  
  // 8. 注册清理
  if (!this.cleanupRegistered) {
    this.cleanupRegistered = true
    registerCleanup(async () => {
      for (const [id, info] of this.spawnedTeammates) {
        await this.backend.killPane(info.paneId, !info.insideTmux)
      }
      this.spawnedTeammates.clear()
    })
  }
  
  // 9. 发送初始提示
  await writeToMailbox(config.name, {
    from: 'team-lead',
    text: config.prompt,
    timestamp: new Date().toISOString(),
  }, config.teamName)
  
  return { success: true, agentId, paneId }
}
```

#### 2. 命令构建细节

**队友身份参数：**
```typescript
const teammateArgs = [
  `--agent-id ${quote([agentId])}`,           // 完整 ID: name@team
  `--agent-name ${quote([config.name])}`,     // 显示名称
  `--team-name ${quote([config.teamName])}`,  // 团队名称
  `--agent-color ${quote([teammateColor])}`,  // UI 颜色
  `--parent-session-id ${quote([...])}`,      // 父会话关联
  config.planModeRequired ? '--plan-mode-required' : '',
]
```

**继承的 CLI 标志（buildInheritedCliFlags）：**
- `--dangerously-skip-permissions`（如果 leader 是 bypass 模式）
- `--permission-mode acceptEdits`（如果 leader 是 acceptEdits 模式）
- `--model <model>`（leader 的模型覆盖）
- `--settings <path>`（leader 的设置路径）
- `--plugin-dir <dir>`（leader 的插件目录）
- `--teammate-mode <mode>`（队友模式）
- `--chrome` / `--no-chrome`（Chrome 标志）

**环境变量（buildInheritedEnvVars）：**
```typescript
const TEAMMATE_ENV_VARS = [
  // API 提供商选择
  'CLAUDE_CODE_USE_BEDROCK',
  'CLAUDE_CODE_USE_VERTEX',
  'CLAUDE_CODE_USE_FOUNDRY',
  // 自定义端点
  'ANTHROPIC_BASE_URL',
  // 配置目录
  'CLAUDE_CONFIG_DIR',
  // CCR 标记
  'CLAUDE_CODE_REMOTE',
  'CLAUDE_CODE_REMOTE_MEMORY_DIR',
  // 代理设置
  'HTTPS_PROXY', 'HTTP_PROXY', 'NO_PROXY',
  // CA 证书
  'SSL_CERT_FILE', 'NODE_EXTRA_CA_CERTS',
]
```

#### 3. 优雅关闭流程（terminate）

```
1. 解析 agentId 获取 agentName 和 teamName
2. 构建 shutdown request 消息
3. 写入队友邮箱
4. 队友检测到后决定是否退出
```

**关键代码（行 252-290）：**

```typescript
async terminate(agentId: string, reason?: string): Promise<boolean> {
  const { agentName, teamName } = parseAgentId(agentId)
  
  const shutdownRequest = {
    type: 'shutdown_request',
    requestId: `shutdown-${agentId}-${Date.now()}`,
    from: 'team-lead',
    reason,
  }
  
  await writeToMailbox(agentName, {
    from: 'team-lead',
    text: jsonStringify(shutdownRequest),
    timestamp: new Date().toISOString(),
  }, teamName)
  
  return true
}
```

#### 4. 强制终止流程（kill）

```
1. 从 spawnedTeammates Map 查找队友信息
2. 调用 backend.killPane() 关闭窗口
3. 从 Map 中移除记录
```

**关键代码（行 295-320）：**

```typescript
async kill(agentId: string): Promise<boolean> {
  const teammateInfo = this.spawnedTeammates.get(agentId)
  if (!teammateInfo) return false
  
  const { paneId, insideTmux } = teammateInfo
  const killed = await this.backend.killPane(paneId, !insideTmux)
  
  if (killed) {
    this.spawnedTeammates.delete(agentId)
  }
  return killed
}
```

#### 5. 活跃状态检查（isActive）

```
1. 检查 spawnedTeammates Map 中是否存在记录
2. 如果存在，假设为活跃（不查询后端）
```

**注意：** 当前实现是"尽力而为"的检查，窗口可能存在但内部进程已退出。

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/swarm/backends/types.ts` | `TeammateExecutor`, `PaneBackend`, `BackendType` 类型 |
| `src/utils/swarm/backends/detection.ts` | `isInsideTmux()` |
| `src/utils/swarm/spawnUtils.ts` | `getTeammateCommand()`, `buildInheritedCliFlags()`, `buildInheritedEnvVars()` |
| `src/utils/swarm/teammateLayoutManager.ts` | `assignTeammateColor()` |
| `src/utils/swarm/constants.ts` | `getSessionId()` |
| `src/utils/teammateMailbox.ts` | `writeToMailbox()` |
| `src/utils/agentId.ts` | `formatAgentId()`, `parseAgentId()` |
| `src/utils/bash/shellQuote.ts` | `quote()` |
| `src/utils/cleanupRegistry.ts` | `registerCleanup()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/slowOperations.ts` | `jsonStringify()` |

### 关键代码位置

- **类定义**: 行 39-345
- **工厂函数**: 行 350-354
- **spawn 方法**: 行 79-209
- **sendMessage 方法**: 行 216-244
- **terminate 方法**: 行 252-290
- **kill 方法**: 行 295-320
- **isActive 方法**: 行 329-344

---

## 依赖与外部交互

### 与 Registry 的交互

PaneBackendExecutor 由 `registry.ts` 创建：

```typescript
// registry.ts
async function getPaneBackendExecutor(): Promise<TeammateExecutor> {
  const detection = await detectAndGetBackend()
  cachedPaneBackendExecutor = createPaneBackendExecutor(detection.backend)
  return cachedPaneBackendExecutor
}
```

### 与 PaneBackend 的交互

PaneBackendExecutor 包装具体的 PaneBackend 实现：

```typescript
constructor(backend: PaneBackend) {
  this.backend = backend
  this.type = backend.type
}
```

调用委托：
- `createTeammatePaneInSwarmView()` → `backend.createTeammatePaneInSwarmView()`
- `sendCommandToPane()` → `backend.sendCommandToPane()`
- `killPane()` → `backend.killPane()`

### 与 Cleanup Registry 的交互

首次创建队友时注册清理处理器：

```typescript
registerCleanup(async () => {
  for (const [id, info] of this.spawnedTeammates) {
    await this.backend.killPane(info.paneId, !info.insideTmux)
  }
  this.spawnedTeammates.clear()
})
```

确保 leader 退出时关闭所有队友窗口。

### 与 Mailbox 系统的交互

使用统一的文件邮箱系统与队友通信：

```typescript
await writeToMailbox(
  config.name,           // 队友名称
  {
    from: 'team-lead',
    text: config.prompt, // 初始提示或 shutdown request
    timestamp: new Date().toISOString(),
  },
  config.teamName        // 团队名称
)
```

---

## 风险、边界与改进建议

### 已知风险

1. **命令注入风险**
   - 构建命令时涉及多个用户可控输入
   - **缓解**: 使用 `quote()` 函数对所有参数进行 shell 转义

2. **环境变量泄漏**
   - 转发的环境变量可能包含敏感信息
   - **缓解**: 仅转发必要的环境变量，且经过审查

3. **Map 状态不一致**
   - spawnedTeammates Map 可能与实际窗口状态不同步
   - 用户手动关闭窗口不会更新 Map
   - **影响**: isActive() 可能返回过时的 true

4. **清理处理器泄漏**
   - 注册的清理处理器在进程退出时才执行
   - 如果队友提前终止，清理处理器仍会尝试关闭已关闭的窗口

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| Context 未设置 | 返回错误，不创建队友 |
| 队友不在 spawnedTeammates 中 | kill/isActive 返回 false |
| killPane 失败 | 保留 Map 中的记录，返回 false |
| 模型覆盖 | 移除继承的 --model，添加队友特定的模型 |
| 非交互式会话 | 通过 useExternalSession 标志控制 |

### 改进建议

1. **状态同步增强**
   - 定期查询后端验证窗口是否存在
   - 在 isActive() 中实现真正的状态检查

2. **错误处理增强**
   - 区分临时错误和永久错误
   - 对临时错误实现重试机制

3. **命令构建优化**
   - 考虑使用数组形式的命令参数，避免字符串拼接
   - 使用 spawn 而非 shell 执行，消除注入风险

4. **资源追踪**
   - 添加队友创建时间戳
   - 实现超时自动清理机制

5. **与 InProcessBackend 统一**
   - 考虑提取更多通用逻辑到基类或工具函数
   - 减少代码重复（如 sendMessage 实现几乎相同）

6. **性能优化**
   - 批量发送邮箱消息
   - 减少重复的状态查询
