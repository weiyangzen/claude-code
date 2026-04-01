# sessionRestore.ts 研究文档

> 文件路径：`src/utils/sessionRestore.ts`  
> 大小：约 20,406 bytes（551 行）  
> 研究范围：代码、调用方、被调用方、配置、测试、脚本、必要上下文

---

## 一、场景与职责

`sessionRestore.ts` 是 Claude Code 会话恢复（resume/continue）流程的核心状态重建模块。它负责在以下场景中将持久化在磁盘日志（JSONL transcript）中的会话状态重新加载到运行时的内存状态：

- **CLI `--resume` / `--continue`**：`main.tsx` 启动路径中，用户指定恢复某个历史会话。
- **交互式 `/resume`**：`REPL.tsx` 运行期间，用户通过 slash command 切换到另一个会话。
- **SDK/Headless 恢复**：`cli/print.ts` 在 non-interactive 模式下恢复会话以继续流式输出。
- **Fork 会话**：`--fork-session` 时复制历史消息到新会话 ID，同时正确处理 content replacement 记录。

该模块不直接读取磁盘文件，而是消费由 `conversationRecovery.ts` 预加载的 `ResumeLoadResult`，并在此基础上完成：
1. 会话 ID 切换与元数据恢复
2. Agent 类型与模型覆盖的恢复
3. Worktree 工作目录的恢复/退出
4. File History、Attribution、Context Collapse、Todos 等副状态的重建
5. 为渲染层计算初始 `AppState`

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| `extractTodosFromTranscript` | SDK/非交互式恢复时，从消息链中逆向扫描最后一个 `TodoWrite` tool_use，重建内存中的 todo list（因为 v1 tasks 没有文件持久化）。 |
| `restoreSessionStateFromLog` | 统一恢复 file history、attribution、context collapse、todos 等副状态。被 REPL.tsx（交互式）和 print.ts（SDK）共同调用。 |
| `computeRestoredAttributionState` | 在渲染前预计算 attribution 初始状态，避免首次渲染时状态为空（遵循 CLAUDE.md 的 pre-render state 准则）。 |
| `computeStandaloneAgentContext` | 恢复会话保存的 agent 名称与颜色，用于独立 agent 的上下文展示。 |
| `restoreAgentFromSession` | 恢复会话使用的自定义 agent；若用户 CLI 已显式指定 `--agent` 则尊重用户选择；若 agent 已不可用则回退到默认行为。 |
| `refreshAgentDefinitionsForModeSwitch` | 当 resumed session 的 coordinator/normal 模式与当前不一致时，重新推导内置 agent 定义，保证模式切换后 agent 列表正确。 |
| `restoreWorktreeForResume` | 若会话上次退出时处于 worktree 内，则 `chdir` 回到该目录；若目录已不存在则安全降级并覆盖缓存。 |
| `exitRestoredWorktree` | `/resume` 切换到另一个会话前，先退出当前恢复的 worktree，避免目录/session 指针残留。 |
| `processResumedConversation` | 恢复流程的**主 orchestrator**：协调模式匹配、ID 切换、worktree 恢复、agent 恢复、初始状态计算。 |

---

## 三、具体技术实现

### 3.1 关键数据结构与类型

```ts
// 来自 conversationRecovery.ts 的预加载结果（ResumeLoadResult 子集）
type ResumeResult = {
  messages?: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
}

// 完整加载结果（conversationRecovery.ts 导出同构类型）
type ResumeLoadResult = {
  messages: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contentReplacements?: ContentReplacementRecord[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
  sessionId: UUID | undefined
  agentName?: string
  agentColor?: string
  agentSetting?: string
  customTitle?: string
  tag?: string
  mode?: 'coordinator' | 'normal'
  worktreeSession?: PersistedWorktreeSession | null
  prNumber?: number
  prUrl?: string
  prRepository?: string
}

// 输出给渲染层
type ProcessedResume = {
  messages: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  contentReplacements?: ContentReplacementRecord[]
  agentName: string | undefined
  agentColor: AgentColorName | undefined
  restoredAgentDef: AgentDefinition | undefined
  initialState: AppState
}
```

### 3.2 关键流程

#### A. `processResumedConversation` 主流程

1. **Coordinator 模式匹配**
   - 若开启 `COORDINATOR_MODE`，调用 `modeApi.matchSessionMode(result.mode)`。
   - 若当前模式与 resumed session 不一致，向消息链追加一条 system warning 消息，并触发后续 agent 定义刷新。

2. **会话 ID 处理**
   - **非 fork**：使用 resumed session 的 ID（或 `sessionIdOverride`），调用 `switchSession(sid, projectDir)`。
     - 随后 `renameRecordingForSession()` 将 asciicast 录制文件重命名为新 ID。
     - `resetSessionFilePointer()` 清空旧文件指针，避免新会话写到旧文件。
     - `restoreCostStateForSession(sid)` 恢复该会话的成本追踪状态。
   - **fork**：保留启动时生成的新会话 ID。若原会话有 `contentReplacements`，调用 `recordContentReplacement` 将其复制到新会话，防止后续恢复时 tool_use_id 匹配失败导致内容被误标为 `FROZEN`。

3. **会话元数据恢复**
   - `restoreSessionMetadata(result)` 将 `customTitle`、`tag`、`agentName`、`agentColor`、`agentSetting`、`mode`、`worktreeSession`、`pr*` 等写入 `project` 缓存。
   - **fork 场景**：显式将 `worktreeSession` 设为 `undefined`，防止 fork 的退出对话框删除原会话仍在引用的 worktree。

4. **Worktree 恢复**
   - 非 fork 时调用 `restoreWorktreeForResume(result.worktreeSession)`：
     - 若启动时已通过 `--worktree` 创建了全新 worktree（`getCurrentWorktreeSession()` 有值），则优先保存该全新状态。
     - 否则尝试 `process.chdir(worktreeSession.worktreePath)` 进入原 worktree；若目录已不存在，则调用 `saveWorktreeState(null)` 覆盖缓存，避免下次退出时重新持久化一个已删除的路径。
     - 成功进入后，调用 `setCwd()`、`setOriginalCwd()`、`restoreWorktreeSession()`，并清空内存文件缓存与 system prompt 缓存。

5. **文件指针收养**
   - 非 fork 时调用 `adoptResumedSessionFile()`：
     - 将 `project.sessionFile` 指向 resumed transcript 的路径。
     - 立即调用 `reAppendSessionMetadata(true)` 把内存中的元数据写回磁盘，确保退出清理 handler 能正常工作。

6. **Context Collapse 恢复**
   - 若开启 `CONTEXT_COLLAPSE`，通过动态 `require('../services/contextCollapse/persist.js')` 调用 `restoreFromEntries(commits, snapshot)`。
   - **无条件执行**：即使 commits 为空也要调用，以清空可能残留的上一会话的 commit log。

7. **Agent 恢复**
   - `restoreAgentFromSession(agentSetting, currentAgentDefinition, agentDefinitions)`：
     - 若 CLI 已指定 agent（`currentAgentDefinition` 存在），直接返回。
     - 若 session 无 agent，调用 `setMainThreadAgentType(undefined)` 清除可能过时的 bootstrap 状态。
     - 查找匹配的 resumed agent；若找不到则回退并打 debug log。
     - 若用户未显式指定模型且 agent 有非 `inherit` 的模型，调用 `setMainLoopModelOverride()` 覆盖主循环模型。

8. **模式持久化**
   - 若开启 `COORDINATOR_MODE`，调用 `saveMode(coordinator ? 'coordinator' : 'normal')`，使未来 resume 知道当前会话模式。

9. **初始状态计算**
   - `computeRestoredAttributionState`（若 `includeAttribution`）
   - `computeStandaloneAgentContext(agentName, agentColor)`
   - `updateSessionName(agentName)`
   - `refreshAgentDefinitionsForModeSwitch`（若前面发生了模式切换 warning）
   - 合并成新的 `initialState` 返回。

#### B. `restoreSessionStateFromLog` 副状态恢复流程

该函数被 **REPL.tsx**（交互式 /resume）和 **print.ts**（SDK resume）调用：

1. **File History**：若存在 snapshots，调用 `fileHistoryRestoreStateFromLog`，通过 `setAppState` 回调更新 `AppState.fileHistory`。
2. **Attribution**：在 `feature('COMMIT_ATTRIBUTION')` 开启且存在 snapshots 时，调用 `attributionRestoreStateFromLog`。
3. **Context Collapse**：无条件调用 `restoreFromEntries`，原因同上（防止 stale commit log）。
4. **Todos**：仅在 `!isTodoV2Enabled()`（即非交互式/SKD 使用 v1 todos）且 messages 非空时，调用 `extractTodosFromTranscript` 并写入 `AppState.todos[agentId]`。

#### C. `exitRestoredWorktree` 流程

用于 `/resume` 从一个 worktree 会话切换到另一个会话（或普通会话）时：

1. 获取 `currentWorktreeSession`；若无则直接返回。
2. 调用 `restoreWorktreeSession(null)` 清空运行时 worktree 指针。
3. 清空内存缓存与 prompt 缓存。
4. 尝试 `process.chdir(originalCwd)` 回到原始目录；若失败则静默处理。
5. 调用 `setCwd(originalCwd)` 与 `setOriginalCwd(getCwd())` 恢复 bootstrap cwd 状态。

---

## 四、关键代码路径与文件引用

### 4.1 本文件内核心函数签名

```ts
// src/utils/sessionRestore.ts
export function restoreSessionStateFromLog(
  result: ResumeResult,
  setAppState: (f: (prev: AppState) => AppState) => void,
): void

export function computeRestoredAttributionState(
  result: ResumeResult,
): AttributionState | undefined

export function computeStandaloneAgentContext(
  agentName: string | undefined,
  agentColor: string | undefined,
): AppState['standaloneAgentContext'] | undefined

export function restoreAgentFromSession(
  agentSetting: string | undefined,
  currentAgentDefinition: AgentDefinition | undefined,
  agentDefinitions: AgentDefinitionsResult,
): { agentDefinition: AgentDefinition | undefined; agentType: string | undefined }

export async function refreshAgentDefinitionsForModeSwitch(
  modeWasSwitched: boolean,
  currentCwd: string,
  cliAgents: AgentDefinition[],
  currentAgentDefinitions: AgentDefinitionsResult,
): Promise<AgentDefinitionsResult>

export function restoreWorktreeForResume(
  worktreeSession: PersistedWorktreeSession | null | undefined,
): void

export function exitRestoredWorktree(): void

export async function processResumedConversation(
  result: ResumeLoadResult,
  opts: { forkSession: boolean; sessionIdOverride?: string; transcriptPath?: string; includeAttribution?: boolean },
  context: {
    modeApi: CoordinatorModeApi | null
    mainThreadAgentDefinition: AgentDefinition | undefined
    agentDefinitions: AgentDefinitionsResult
    currentCwd: string
    cliAgents: AgentDefinition[]
    initialState: AppState
  },
): Promise<ProcessedResume>
```

### 4.2 上游调用方

| 调用方文件 | 调用函数 | 场景 |
|-----------|---------|------|
| `src/main.tsx` | `processResumedConversation` | CLI `--resume` / `--continue` 启动路径 |
| `src/cli/print.ts` | `processResumedConversation`, `restoreSessionStateFromLog` | SDK/Headless resume |
| `src/screens/REPL.tsx` | `restoreSessionStateFromLog`, `restoreWorktreeForResume`, `exitRestoredWorktree`, `restoreAgentFromSession`, `computeStandaloneAgentContext` | 交互式 `/resume` slash command |
| `src/screens/ResumeConversation.tsx` | 间接通过 REPL.tsx 或自身调用 | 恢复对话 UI |

### 4.3 下游被调用方

| 被调用模块/文件 | 函数/符号 | 用途 |
|----------------|----------|------|
| `src/bootstrap/state.js` | `switchSession`, `setMainThreadAgentType`, `setMainLoopModelOverride`, `setOriginalCwd`, `getMainLoopModelOverride`, `getSessionId` | 修改全局 bootstrap 状态 |
| `src/utils/sessionStorage.ts` | `adoptResumedSessionFile`, `recordContentReplacement`, `resetSessionFilePointer`, `restoreSessionMetadata`, `saveMode`, `saveWorktreeState` | 持久化元数据与文件指针管理 |
| `src/utils/worktree.ts` | `getCurrentWorktreeSession`, `restoreWorktreeSession` | worktree 运行时状态 |
| `src/utils/conversationRecovery.ts` | `loadConversationForResume`（调用方预调用） | 加载原始对话数据 |
| `src/utils/fileHistory.ts` | `fileHistoryRestoreStateFromLog` | 恢复文件历史 |
| `src/utils/commitAttribution.ts` | `attributionRestoreStateFromLog`, `restoreAttributionStateFromSnapshots` | 恢复 attribution 状态 |
| `src/utils/asciicast.ts` | `renameRecordingForSession` | 重命名录制文件 |
| `src/cost-tracker.js` | `restoreCostStateForSession` | 恢复成本状态 |
| `src/tools/AgentTool/loadAgentsDir.js` | `getActiveAgentsFromList`, `getAgentDefinitionsWithOverrides` | Agent 定义查询与刷新 |
| `src/services/contextCollapse/persist.js` | `restoreFromEntries`（动态 require） | 恢复 context collapse 提交日志 |
| `src/utils/concurrentSessions.ts` | `updateSessionName` | 更新并发会话名称 |
| `src/constants/systemPromptSections.js` | `clearSystemPromptSections` | 清除 prompt 缓存 |
| `src/utils/claudemd.ts` | `clearMemoryFileCaches` | 清除内存文件缓存 |
| `src/utils/plans.ts` | `getPlansDirectory` | 清除 plans 目录缓存 |

---

## 五、依赖与外部交互

### 5.1 运行时依赖

- **`bun:bundle` 的 `feature()`**：大量功能开关（`COMMIT_ATTRIBUTION`、`CONTEXT_COLLAPSE`、`COORDINATOR_MODE`）通过运行时 feature flag 控制，决定某些恢复分支是否执行。
- **`crypto` 的 `UUID`**：用于类型声明，实际不直接生成 UUID。
- **`path.dirname`**：用于从 `transcriptPath` 推导项目目录（跨目录 resume）。

### 5.2 状态系统交互

该模块是**状态恢复的中转站**：
- 读取端：消费 `conversationRecovery.ts` 的 `ResumeLoadResult`（来自磁盘 JSONL）。
- 写入端：通过 `setAppState` 回调、直接调用 bootstrap state mutators、以及 `sessionStorage.ts` 的缓存函数，将状态写回内存。

### 5.3 与 worktree 子系统的耦合

`restoreWorktreeForResume` / `exitRestoredWorktree` 与 `worktree.ts` 紧密配合：
- 使用 `process.chdir` 作为 **TOCTOU-safe** 的存在性检查（注释明确说明）。
- 不设置 `projectRoot`，以匹配 `EnterWorktreeTool` 的行为（skills/history 锚定在原始项目）。

### 5.4 与 context collapse 的耦合

通过**动态 `require`** 引入 `../services/contextCollapse/persist.js`，避免在功能关闭时静态打包该模块，减少 bundle 体积。

---

## 六、风险、边界与改进建议

### 6.1 已知风险与边界

1. **动态 require 的维护成本**
   - `CONTEXT_COLLAPSE` 和 `COORDINATOR_MODE` 分支使用 `require()` 进行条件加载。
   - 风险：路径字符串硬编码，若模块重构时路径变更，编译期无法发现，运行时可能抛 `MODULE_NOT_FOUND`。
   - 缓解：现有代码已用 `as typeof import(...)` 做类型断言，但仍需人工保证路径同步。

2. **Worktree 目录竞态（TOCTOU）**
   - `restoreWorktreeForResume` 使用 `process.chdir` 作为存在性检查，但目录在 `chdir` 成功和被调用方使用之间仍可能被删除。
   - 实际影响较小，因为后续文件操作会自然抛出 `ENOENT`。

3. **Fork 会话的 content replacement 种子**
   - 注释详细解释了为什么 fork 时必须复制 `contentReplacements`：否则新会话的 tool_use_id 会找不到对应替换记录，导致 `FROZEN` 分类，进而发送完整内容造成缓存未命中和永久超额。
   - 边界：若原会话的 replacement 记录非常多，复制可能带来一定 I/O 开销。

4. **Attribution 恢复的 feature flag 与 `includeAttribution` 选项**
   - `computeRestoredAttributionState` 只在 `feature('COMMIT_ATTRIBUTION')` 和 `result.attributionSnapshots` 都存在时才返回值。
   - `processResumedConversation` 还受 `opts.includeAttribution` 控制；若调用方误设为 `false`，即使快照存在也会跳过恢复。

5. **Agent 恢复的竞争条件**
   - `restoreAgentFromSession` 会修改 `setMainThreadAgentType` 和 `setMainLoopModelOverride`（全局 bootstrap 状态）。
   - 若并发恢复多个会话（理论上 Claude Code 主线程单会话，但测试/异常路径中可能），可能导致状态覆盖。

6. **Todo 恢复的 v1/v2 分裂**
   - `extractTodosFromTranscript` 仅对 v1 todos（`!isTodoV2Enabled()`）生效。
   - 交互式模式使用文件级 v2 tasks，因此 `AppState.todos` 在交互式恢复路径中实际上不被填充，这是设计上的 but 对跨模式 resume 可能造成困惑。

### 6.2 改进建议

1. **统一动态 require 的容错**
   - 建议将 `require('../services/contextCollapse/persist.js')` 包装成带 try/catch 的辅助函数，若模块缺失则打印明确警告而非直接崩溃。

2. **减少 `processResumedConversation` 的参数复杂度**
   - 当前 `opts` + `context` 参数较多，可考虑将 `context` 中的 `modeApi`、`agentDefinitions`、`initialState` 等合并为一个 `ResumeContext` 类型，提升可读性。

3. **Worktree 恢复的原子性**
   - 考虑将 `chdir` + `setCwd` + `setOriginalCwd` + `restoreWorktreeSession` 封装为一个原子操作，并在任何一步失败时统一回滚，避免半恢复状态。

4. **测试覆盖**
   - 当前仓库中未找到针对 `sessionRestore.ts` 的单元测试文件。
   - 建议为 `restoreAgentFromSession`、`refreshAgentDefinitionsForModeSwitch`、`computeRestoredAttributionState` 等纯函数补充单元测试；为 `processResumedConversation` 补充集成测试（mock `conversationRecovery.ts` 和 `sessionStorage.ts`）。

5. **文档化 cross-directory resume 逻辑**
   - `switchSession(sid, opts.transcriptPath ? dirname(opts.transcriptPath) : null)` 这一行承载了跨目录恢复的核心逻辑，但分散在 `main.tsx` 和本文件中，建议在 `conversationRecovery.ts` 或本模块顶部补充架构注释。

---

*文档生成时间：2026-04-01*  
*研究执行器：kimi (model=k2p5)*
