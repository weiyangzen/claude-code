# worktree.ts 研究文档

## 场景与职责

`worktree.ts` 是 Claude Code CLI 的 Git 工作区（worktree）管理核心模块，提供完整的 worktree 生命周期管理功能。支持用户创建隔离的工作环境进行并行开发，同时保持与主仓库的关联。

**核心使用场景：**
- `--worktree` 命令行参数创建隔离工作区
- `--tmux` 集成，在 tmux 会话中运行 Claude
- Agent 工具的隔离工作区创建
- PR 检出（支持 `#123` 或 GitHub URL 格式）
- 工作区清理和过期工作区自动回收

## 功能点目的

### 1. 工作区创建与管理
- **`createWorktreeForSession`**：为用户会话创建/恢复工作区
- **`createAgentWorktree`**：为 Agent 工具创建轻量级工作区
- **`getOrCreateWorktree`**：底层 worktree 创建/恢复逻辑

### 2. 工作区生命周期管理
- **`keepWorktree`**：保留工作区（退出时不删除）
- **`cleanupWorktree`**：清理当前工作区
- **`removeAgentWorktree`**：移除 Agent 工作区

### 3. Tmux 集成
- **`execIntoTmuxWorktree`**：`--worktree --tmux` 快速路径处理
- **`createTmuxSessionForWorktree`**：为工作区创建 tmux 会话
- **`isTmuxAvailable`** / **`getTmuxInstallInstructions`**：tmux 环境检测

### 4. 过期工作区清理
- **`cleanupStaleAgentWorktrees`**：清理超过 30 天的临时工作区
- **`hasWorktreeChanges`**：检测工作区是否有未提交更改

### 5. 配置和钩子支持
- **`.worktreeinclude 支持`**：复制被 gitignore 但需要的文件
- **Hooks 支持**：`WorktreeCreate`/`WorktreeRemove` 钩子允许自定义 VCS

## 具体技术实现

### 工作区命名规范

```typescript
// Slug 验证规则
const VALID_WORKTREE_SLUG_SEGMENT = /^[a-zA-Z0-9._-]+$/
const MAX_WORKTREE_SLUG_LENGTH = 64

// 嵌套 slug 扁平化（解决 git D/F 冲突）
// user/feature → user+feature
function flattenSlug(slug: string): string {
  return slug.replaceAll('/', '+')
}

// 分支命名
function worktreeBranchName(slug: string): string {
  return `worktree-${flattenSlug(slug)}`
}

// 路径生成
function worktreePathFor(repoRoot: string, slug: string): string {
  return join(worktreesDir(repoRoot), flattenSlug(slug))
}
```

### 工作区创建流程

```
createWorktreeForSession(sessionId, slug, tmuxSessionName, options)
├── validateWorktreeSlug(slug)  // 验证 slug 格式
├── 检查 WorktreeCreate Hook
│   └── 如果存在 → executeWorktreeCreateHook(slug)
│       └── 返回 hookResult.worktreePath
└── 否则 → Git worktree 流程
    ├── findGitRoot(getCwd())  // 查找 git 根目录
    ├── getOrCreateWorktree(repoRoot, slug, options)
    │   ├── 检查工作区是否已存在（fast resume）
    │   │   └── readWorktreeHeadSha(worktreePath)
    │   ├── 如果是 PR → fetch PR 分支
    │   ├── 否则 → fetch 默认分支（如果本地不存在）
    │   ├── git worktree add -B <branch> <path> <base>
    │   └── 如果配置了 sparsePaths → 配置 sparse-checkout
    └── performPostCreationSetup(repoRoot, worktreePath)
        ├── 复制 settings.local.json
        ├── 配置 git hooks 路径
        ├── 创建目录符号链接（如 node_modules）
        └── 复制 .worktreeinclude 指定的文件
```

### PR 检出支持

```typescript
// 支持的格式
parsePRReference("#123")        // → 123
parsePRReference("https://github.com/owner/repo/pull/123")  // → 123
parsePRReference("https://ghe.example.com/owner/repo/pull/123")  // → 123

// PR 检出流程
if (options?.prNumber) {
  await execFileNoThrow(gitExe(), [
    'fetch', 'origin', `pull/${options.prNumber}/head`
  ])
  baseBranch = 'FETCH_HEAD'
}
```

### 过期工作区清理策略

```typescript
// 临时工作区模式（只清理匹配这些模式的）
const EPHEMERAL_WORKTREE_PATTERNS = [
  /^agent-a[0-9a-f]{7}$/,           // Agent 工具工作区
  /^wf_[0-9a-f]{8}-[0-9a-f]{3}-\d+$/, // Workflow 工作区
  /^wf-\d+$/,                        // 旧版 Workflow 工作区
  /^bridge-[A-Za-z0-9_]+(-[A-Za-z0-9_]+)*$/, // Bridge 工作区
  /^job-[a-zA-Z0-9._-]{1,55}-[0-9a-f]{8}$/, // 模板任务工作区
]

// 安全检查（fail-closed）
async function cleanupStaleAgentWorktrees(cutoffDate: Date): Promise<number> {
  // 1. 只匹配临时工作区模式（永不删除用户命名的工作区）
  // 2. 跳过当前会话的工作区
  // 3. 检查 git status（有更改则跳过）
  // 4. 检查未推送提交（有则跳过）
}
```

### Tmux 集成架构

```
execIntoTmuxWorktree(args)
├── 平台检查（Windows 不支持）
├── tmux 可用性检查
├── 解析 --worktree 和 --tmux 参数
├── 解析 PR 引用（如果有）
├── 生成随机 slug（如果未提供）
├── validateWorktreeSlug()
├── 创建/恢复工作区
├── 生成 tmux 会话名（repoName_worktree-branch）
├── 构建新的参数列表（移除 --tmux 和 --worktree）
├── 检测 tmux 前缀键（检查是否与 Claude 快捷键冲突）
├── 设置环境变量（CLAUDE_CODE_TMUX_*）
└── 启动 tmux 会话
    ├── iTerm2 + Control Mode（-CC）：原生标签/面板集成
    ├── 已在 tmux 中：switch-client 到兄弟会话
    └── 标准模式：attach-session -A
```

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `validateWorktreeSlug` | 66 | 验证工作区名称格式 |
| `createWorktreeForSession` | 702 | 为用户会话创建工作区 |
| `createAgentWorktree` | 902 | 为 Agent 工具创建工作区 |
| `removeAgentWorktree` | 961 | 移除 Agent 工作区 |
| `cleanupWorktree` | 813 | 清理当前工作区 |
| `keepWorktree` | 780 | 保留工作区 |
| `cleanupStaleAgentWorktrees` | 1058 | 清理过期工作区 |
| `execIntoTmuxWorktree` | 1180 | --tmux 快速路径 |
| `copyWorktreeIncludeFiles` | 391 | 复制 .worktreeinclude 文件 |
| `hasWorktreeChanges` | 1144 | 检测工作区更改 |

### 内部辅助函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `getOrCreateWorktree` | 235 | 底层 worktree 创建/恢复 |
| `performPostCreationSetup` | 510 | 工作区创建后配置 |
| `symlinkDirectories` | 102 | 创建目录符号链接 |
| `worktreeBranchName` | 221 | 生成分支名 |
| `worktreePathFor` | 225 | 生成工作区路径 |
| `flattenSlug` | 217 | 扁平化嵌套 slug |
| `parsePRReference` | 633 | 解析 PR 引用 |

### 类型定义

| 类型 | 行号 | 说明 |
|------|------|------|
| `WorktreeSession` | 140 | 工作区会话信息 |
| `WorktreeCreateResult` | 180 | 创建结果类型 |

### 依赖文件

| 文件 | 导入内容 |
|------|----------|
| `src/utils/config.ts` | `saveCurrentProjectConfig` |
| `src/utils/cwd.ts` | `getCwd` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/errors.ts` | `errorMessage`, `getErrnoCode` |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrow`, `execFileNoThrowWithCwd` |
| `src/utils/git/gitConfigParser.ts` | `parseGitConfigValue` |
| `src/utils/git/gitFilesystem.ts` | `getCommonDir`, `readWorktreeHeadSha`, `resolveGitDir`, `resolveRef` |
| `src/utils/git.ts` | `findCanonicalGitRoot`, `findGitRoot`, `getBranch`, `getDefaultBranch`, `gitExe` |
| `src/utils/hooks.ts` | `executeWorktreeCreateHook`, `executeWorktreeRemoveHook`, `hasWorktreeCreateHook` |
| `src/utils/path.ts` | `containsPathTraversal` |
| `src/utils/platform.ts` | `getPlatform` |
| `src/utils/settings/settings.ts` | `getInitialSettings`, `getRelativeSettingsFilePathForSource` |
| `src/utils/sleep.ts` | `sleep` |
| `src/utils/swarm/backends/detection.ts` | `isInITerm2` |

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `bun:bundle` | `feature` 函数（特性开关） |
| `chalk` | 终端颜色输出 |
| `child_process` | `spawnSync` |
| `fs/promises` | 文件系统操作 |
| `ignore` | .gitignore 模式匹配 |
| `path` | 路径操作 |

## 依赖与外部交互

### Git 命令交互

```typescript
// 工作区创建
git worktree add -B <branch> <path> <base>

// PR 检出
git fetch origin pull/<prNumber>/head

// Sparse checkout（如果配置）
git sparse-checkout set --cone -- <paths>

// 工作区移除
git worktree remove --force <path>
git branch -D <branch>

// 过期检测
git status --porcelain -uno
git rev-list --max-count=1 HEAD --not --remotes
```

### 配置文件交互

```typescript
// 保存活动工作区会话
saveCurrentProjectConfig(current => ({
  ...current,
  activeWorktreeSession: currentWorktreeSession ?? undefined,
}))
```

### 环境变量

```typescript
// 防止 git/SSH 提示输入凭证（会挂起 CLI）
const GIT_NO_PROMPT_ENV = {
  GIT_TERMINAL_PROMPT: '0',
  GIT_ASKPASS: '',
}

// Tmux 环境变量
CLAUDE_CODE_TMUX_SESSION      // 会话名
CLAUDE_CODE_TMUX_PREFIX       // 前缀键
CLAUDE_CODE_TMUX_PREFIX_CONFLICTS  // 是否有快捷键冲突
```

## 风险、边界与改进建议

### 已知风险

1. **路径遍历风险**
   - `validateWorktreeSlug` 防止 `..` 和绝对路径
   - `containsPathTraversal` 检查目录符号链接目标

2. **Git 凭证提示挂起**
   - 使用 `GIT_TERMINAL_PROMPT=0` 和 `GIT_ASKPASS=''` 禁用交互式提示
   - `stdin: 'ignore'` 关闭标准输入

3. **并发问题**
   - `cleanupStaleAgentWorktrees` 可能与其他进程冲突
   - 使用 `--force` 标志强制移除

4. **Tmux 前缀冲突**
   - Claude 使用 `ctrl+b`, `ctrl+c`, `ctrl+d` 等快捷键
   - 如果 tmux 前缀也是这些键，会有冲突
   - 代码检测并报告冲突

### 边界情况

1. **工作区已存在**
   - `getOrCreateWorktree` 通过 `readWorktreeHeadSha` 快速检测
   - 存在则直接返回（fast resume）

2. **Sparse checkout 失败**
   - 如果配置失败，会执行清理（tearDown）
   - 移除部分创建的工作区，避免残留

3. **Hook 失败**
   - Hook 执行失败会抛出错误
   - 调用方需要处理错误

4. **Windows 不支持 Tmux**
   - `execIntoTmuxWorktree` 检查 `process.platform === 'win32'`
   - 返回错误提示

### 改进建议

1. **工作区健康检查**
   - 添加定期检查工作区完整性的机制
   - 检测损坏的 git 引用或缺失的文件

2. **并发创建保护**
   - 使用文件锁防止同时创建同名工作区
   - 避免竞态条件导致的 git 错误

3. **工作区使用统计**
   - 记录工作区访问时间，优化过期策略
   - 区分"活跃"和"闲置"工作区

4. **增量同步**
   - 对于长时间保留的工作区，提供增量同步功能
   - 从主仓库同步最新更改

5. **可视化工具**
   - 添加命令列出所有工作区及其状态
   - 显示工作区的最后访问时间、分支、更改状态

6. **配置扩展**
   - 支持 `.worktreeignore` 排除特定文件
   - 支持工作区模板（预配置的开发环境）
