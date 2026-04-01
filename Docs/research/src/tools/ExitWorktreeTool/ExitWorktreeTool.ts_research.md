# ExitWorktreeTool.ts 研究文档

## 场景与职责

`ExitWorktreeTool.ts` 实现了 `ExitWorktree` 工具，用于**退出当前会话中由 `EnterWorktreeTool` 创建的 git worktree 会话**，并将会话恢复到原始工作目录。它是 `EnterWorktreeTool` 的语义逆操作，构成 worktree 隔离功能的"出口"闸门。

该工具仅在 `isWorktreeModeEnabled()` 为 true 时通过 `src/tools.ts` 注册到工具池（`getAllBaseTools`），且被显式标记为 `shouldDefer: true`（需要通过 ToolSearch 才能调用）。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `action: 'keep'` | 保留 worktree 目录和分支，仅将 CLI 会话的 CWD 切回原始目录 |
| `action: 'remove'` | 强制删除 worktree 目录，并清理对应的临时分支 |
| `discard_changes` 校验 | 防止在存在未提交文件或未推送 commit 时误删工作成果 |
| 会话状态恢复 | 重置 `originalCwd`、`projectRoot`、系统提示缓存、内存文件缓存等 CWD 依赖状态 |
| tmux 会话清理 | `remove` 时杀死关联的 tmux 会话；`keep` 时保留并提示用户如何重新 attach |

## 具体技术实现

### 1. 输入/输出 Schema

```ts
// 输入
{
  action: z.enum(['keep', 'remove']),
  discard_changes: z.boolean().optional()
}

// 输出
{
  action: 'keep' | 'remove',
  originalCwd: string,
  worktreePath: string,
  worktreeBranch?: string,
  tmuxSessionName?: string,
  discardedFiles?: number,
  discardedCommits?: number,
  message: string,
}
```

### 2. 核心私有函数

#### `countWorktreeChanges(worktreePath, originalHeadCommit): Promise<ChangeSummary | null>`

- 使用 `git -C <path> status --porcelain` 统计未提交文件数
- 使用 `git rev-list --count <originalHeadCommit>..HEAD` 统计 worktree 分支上新增 commit 数
- **fail-closed 设计**：任何 git 命令非零退出或 `originalHeadCommit` 缺失时返回 `null`，调用方必须将 `null` 视为"状态未知，禁止删除"

#### `restoreSessionToOriginalCwd(originalCwd, projectRootIsWorktree): void`

- 调用 `setCwd(originalCwd)` 恢复进程 CWD
- 调用 `setOriginalCwd(originalCwd)` 恢复原始目录状态
- 仅在 `--worktree` 启动模式下（`projectRootIsWorktree === true`）才恢复 `projectRoot`，避免 mid-session `EnterWorktree` 破坏"稳定项目身份"契约
- 清理 CWD 依赖缓存：
  - `saveWorktreeState(null)`
  - `clearSystemPromptSections()`
  - `clearMemoryFileCaches()`
  - `getPlansDirectory.cache.clear?.()`
  - `updateHooksConfigSnapshot()`（仅 `--worktree` 启动时）

### 3. 工具生命周期方法

#### `validateInput(input)`

1. **作用域守卫**：通过 `getCurrentWorktreeSession()` 检查当前会话是否由 `EnterWorktree` 创建。若不是，返回 `result: false`，并附带 `errorCode: 1`。
2. **删除安全守卫**：若 `action === 'remove' && !discard_changes`，调用 `countWorktreeChanges()`：
   - 返回 `null` → `errorCode: 3`，拒绝删除并要求显式确认
   - 存在变更 → `errorCode: 2`，列出未提交文件数和/或 commit 数，要求用户确认后重试

#### `call(input)`

1. 再次防御性检查 `getCurrentWorktreeSession()`（防止 validation→call 之间出现竞态）
2. 在 `keepWorktree()` / `cleanupWorktree()` 清空 session 之前，提前捕获 `session` 字段
3. 通过 `getProjectRoot() === getOriginalCwd()` 判断是否为 `--worktree` 启动模式
4. 再次调用 `countWorktreeChanges()` 获取执行时点的精确统计（用于 analytics 和输出文案）
5. 分支处理：
   - **`keep`** → `await keepWorktree()` → `restoreSessionToOriginalCwd()` → 记录 `tengu_worktree_kept` 事件
   - **`remove`** → `await killTmuxSession(tmuxSessionName)`（如存在）→ `await cleanupWorktree()` → `restoreSessionToOriginalCwd()` → 记录 `tengu_worktree_removed` 事件
6. 构造人类可读的消息返回给模型

### 4. 其他元数据

- `isDestructive(input)`：当 `action === 'remove'` 时返回 `true`，触发权限系统的破坏性操作确认
- `toAutoClassifierInput(input)`：返回 `input.action`，供自动模式安全分类器使用
- `mapToolResultToToolResultBlockParam`：将 `message` 包装为 `tool_result` 块

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | 本文件，工具主实现 |
| `src/tools/ExitWorktreeTool/constants.ts` | `EXIT_WORKTREE_TOOL_NAME = 'ExitWorktree'` |
| `src/tools/ExitWorktreeTool/prompt.ts` | 模型提示词（scope、参数说明、行为描述） |
| `src/tools/ExitWorktreeTool/UI.tsx` | 工具使用/结果渲染组件 |
| `src/utils/worktree.ts` | `getCurrentWorktreeSession`、`keepWorktree`、`cleanupWorktree`、`killTmuxSession` |
| `src/bootstrap/state.ts` | `getOriginalCwd`、`getProjectRoot`、`setOriginalCwd`、`setProjectRoot`、`setCwdState` |
| `src/utils/sessionStorage.ts` | `saveWorktreeState`（写入/清除持久化 worktree 状态） |
| `src/constants/systemPromptSections.ts` | `clearSystemPromptSections`（清除系统提示缓存） |
| `src/utils/claudemd.ts` | `clearMemoryFileCaches`（清除 CLAUDE.md 内存缓存） |
| `src/utils/plans.ts` | `getPlansDirectory`（计划文件目录，带 memoize 缓存） |
| `src/utils/hooks/hooksConfigSnapshot.ts` | `updateHooksConfigSnapshot`（恢复 hooks 配置快照） |
| `src/utils/Shell.ts` | `setCwd`（同步修改进程 CWD 与状态） |
| `src/services/analytics/index.ts` | `logEvent`（记录 `tengu_worktree_kept` / `tengu_worktree_removed`） |
| `src/tools.ts` | 工具注册池，`isWorktreeModeEnabled()` 条件注入 `ExitWorktreeTool` |
| `src/constants/tools.ts` | `ASYNC_AGENT_ALLOWED_TOOLS` 包含 `EXIT_WORKTREE_TOOL_NAME`，允许异步 agent 使用 |

## 依赖与外部交互

### 外部进程调用

通过 `execFileNoThrow` 执行 git 子进程：
- `git -C <worktreePath> status --porcelain`
- `git -C <worktreePath> rev-list --count <originalHeadCommit>..HEAD`

这些调用在 `countWorktreeChanges` 中完成，用于判断 worktree 是否"干净"。

### 与 `EnterWorktreeTool` 的配对关系

- `EnterWorktreeTool.call()` 创建 worktree 后会设置 `currentWorktreeSession`、修改 `originalCwd`、清除缓存
- `ExitWorktreeTool` 是上述操作的精确逆操作：恢复 `originalCwd`、清除 session、再次清除缓存
- 两者通过模块级可变状态 `currentWorktreeSession`（位于 `src/utils/worktree.ts`）进行协调

### 状态变更时序

```
validateInput()
  └─> getCurrentWorktreeSession() 检查
      └─> countWorktreeChanges() 预检（仅 remove）

call()
  └─> 再次 getCurrentWorktreeSession()
      └─> countWorktreeChanges() 精确计数
          └─> keepWorktree() / cleanupWorktree() + killTmuxSession()
              └─> restoreSessionToOriginalCwd()
                  └─> logEvent()
```

## 风险、边界与改进建议

### 风险与边界

1. **竞态条件（Race）**
   - `currentWorktreeSession` 是模块级可变状态。`validateInput` 与 `call` 之间如果用户通过其他路径（如并发工具调用）修改了状态，可能导致 `call` 中抛出 `"Not in a worktree session"`。代码已做防御性检查，但无法完全消除模块级可变状态的风险。

2. **fail-closed 的 UX 代价**
   - `countWorktreeChanges` 返回 `null` 时（如 git 锁文件、索引损坏、hook-based worktree 无 `originalHeadCommit`），工具会拒绝删除。这虽然安全，但在某些边缘场景下可能让用户困惑，因为错误信息仅提示"无法验证状态"。

3. **`projectRootIsWorktree` 判断的隐式契约**
   - 通过 `getProjectRoot() === getOriginalCwd()` 推断是否为 `--worktree` 启动。该等式在 mid-session `EnterWorktree` 后不成立（因为 `EnterWorktreeTool` 不修改 `projectRoot`）。如果未来启动逻辑改变，该推断可能失效。

4. **tmux 会话的孤儿问题**
   - `keep` 模式下 tmux 会话被保留，但如果用户后续手动删除 worktree 目录，tmux 会话可能变成"孤儿"（仍在运行但 CWD 指向已删除目录）。当前没有自动清理机制。

5. **Analytics 计数在 git 失败时回退为 0**
   - `call()` 中 `countWorktreeChanges` 返回 `null` 时，通过 `?? { changedFiles: 0, commits: 0 }` 回退。这会导致 analytics 事件低估实际被丢弃的工作量，但出于安全考虑这是可接受的。

### 改进建议

1. **显式模式标记替代推断**
   - 建议在 `WorktreeSession` 中增加一个布尔字段 `isStartupWorktree`（或类似），显式记录该 worktree 是否由 `--worktree` 启动标志创建，从而消除对 `getProjectRoot() === getOriginalCwd()` 的隐式依赖。

2. **更细粒度的错误码与文案**
   - 当 `countWorktreeChanges` 返回 `null` 时，可以区分是"git 命令失败"还是"缺少 originalHeadCommit"，向用户提供更有针对性的重试建议（如"请检查 git 状态" vs "该 worktree 由 hook 创建，无法自动判断变更，请手动确认"）。

3. **tmux 会话存活提示增强**
   - 当前 `keep` 模式仅返回一条文本提示。可以考虑在 `saveWorktreeState` 中记录 tmux 会话名，并在 `--resume` 时检测并提示用户有挂起的 tmux 会话。

4. **单元测试覆盖**
   - 目前未找到针对 `ExitWorktreeTool` 的专用单元测试。建议增加对以下场景的测试：
     - `validateInput` 在无 active session 时返回 errorCode 1
     - `discard_changes: false` 且存在未提交文件时返回 errorCode 2
     - `countWorktreeChanges` 在 git 失败时返回 null
     - `restoreSessionToOriginalCwd` 正确区分 `--worktree` 启动与 mid-session 进入
