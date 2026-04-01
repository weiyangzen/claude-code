# WorktreeExitDialog.tsx 深度研究文档

## 场景与职责

`WorktreeExitDialog` 是 Claude Code CLI 中用于**工作树（worktree）退出确认**的交互式对话框组件。当用户在使用工作树隔离功能（`--worktree` 模式）后尝试退出 Claude 时，该对话框会显示工作树的当前状态（未提交更改、提交数量等），并让用户选择是保留工作树还是清理删除。

### 核心职责
1. **工作树状态检测**：检测当前工作树的 Git 状态（未提交更改、提交数量）
2. **用户决策**：让用户选择保留工作树（keep）或删除工作树（remove）
3. **Tmux 会话管理**：支持关联的 tmux 会话的保留或终止
4. **清理操作执行**：根据用户选择执行相应的清理或保留操作
5. **状态持久化**：更新会话存储和项目配置

## 功能点目的

### 1. 工作树状态分析
在对话框显示前，自动分析工作树状态：
- **未提交更改**：通过 `git status --porcelain` 检测
- **提交数量**：通过 `git rev-list --count` 检测工作树分支上的新提交
- **静默清理**：如无更改且无提交，自动清理并退出，不显示对话框

### 2. 用户选项
根据是否有 tmux 会话，提供不同选项：

**无 Tmux 会话时：**
- **Keep worktree**：保留工作树，切换回原目录
- **Remove worktree**：删除工作树，清理分支

**有 Tmux 会话时：**
- **Keep worktree and tmux session**：保留工作树和 tmux 会话
- **Keep worktree, kill tmux session**：保留工作树，终止 tmux 会话
- **Remove worktree and tmux session**：删除工作树，终止 tmux 会话

### 3. 取消操作
支持取消退出操作，返回 Claude 会话继续工作。

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type Props = {
  onDone: (result?: string, options?: { display?: CommandResultDisplay }) => void;
  onCancel?: () => void;
};

// 组件内部状态
const [status, setStatus] = useState<'loading' | 'asking' | 'keeping' | 'removing' | 'done'>('loading');
const [changes, setChanges] = useState<string[]>([]);        // 未提交更改文件列表
const [commitCount, setCommitCount] = useState<number>(0);   // 提交数量
const [resultMessage, setResultMessage] = useState<string | undefined>();
```

### 关键流程

#### 1. 状态加载流程（useEffect）
```
组件挂载
    ↓
获取当前工作树会话（getCurrentWorktreeSession）
    ↓
执行 git status --porcelain
    ↓
执行 git rev-list --count <originalHeadCommit>..HEAD
    ↓
判断：changes.length === 0 && commitCount === 0?
    ↓ 是 → 自动清理（silent cleanup）
    ↓ 否 → 显示对话框（setStatus('asking')）
```

#### 2. 自动清理流程
```typescript
if (changeLines.length === 0 && count === 0) {
  setStatus('removing');
  await cleanupWorktree();
  process.chdir(worktreeSession.originalCwd);
  setCwd(worktreeSession.originalCwd);
  recordWorktreeExit();
  getPlansDirectory.cache.clear?.();
  setResultMessage('Worktree removed (no changes)');
  setStatus('done');
}
```

#### 3. 用户选择处理流程
```
用户选择选项
    ↓
handleSelect(value)
    ↓
判断 value 类型：
    ├── 'keep' / 'keep-with-tmux' → 保留工作树
    │   ├── 记录分析事件（logEvent）
    │   ├── 调用 keepWorktree()
    │   ├── 切换回原目录
    │   ├── 记录工作树退出
    │   └── 设置完成消息
    │
    ├── 'keep-kill-tmux' → 保留工作树，终止 tmux
    │   ├── 调用 killTmuxSession()
    │   └── 后续同上
    │
    └── 'remove' / 'remove-with-tmux' → 删除工作树
        ├── 记录分析事件
        ├── 调用 killTmuxSession()（如有 tmux）
        ├── 调用 cleanupWorktree()
        ├── 切换回原目录
        ├── 记录工作树退出
        └── 设置完成消息
```

### 核心工具函数

#### 1. 记录工作树退出
```typescript
function recordWorktreeExit(): void {
  // 使用 inline require 打破循环依赖
  (require('../utils/sessionStorage.js') as typeof import('../utils/sessionStorage.js'))
    .saveWorktreeState(null);
}
```

#### 2. 选项生成逻辑
```typescript
const options = hasTmuxSession ? [
  {
    label: 'Keep worktree and tmux session',
    value: 'keep-with-tmux',
    description: `Stays at ${worktreeSession.worktreePath}. Reattach with: tmux attach -t ${worktreeSession.tmuxSessionName}`
  },
  {
    label: 'Keep worktree, kill tmux session',
    value: 'keep-kill-tmux',
    description: `Keeps worktree at ${worktreeSession.worktreePath}, terminates tmux session.`
  },
  {
    label: 'Remove worktree and tmux session',
    value: 'remove-with-tmux',
    description: removeDescription
  }
] : [
  {
    label: 'Keep worktree',
    value: 'keep',
    description: `Stays at ${worktreeSession.worktreePath}`
  },
  {
    label: 'Remove worktree',
    value: 'remove',
    description: removeDescription
  }
];
```

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/WorktreeExitDialog.tsx`

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/commands.js` | `CommandResultDisplay` 类型 |
| `src/services/analytics/index.js` | `logEvent` 分析事件 |
| `src/utils/debug.js` | `logForDebugging` 调试日志 |
| `../ink.js` | `Box`, `Text` 组件 |
| `../utils/execFileNoThrow.js` | 执行 Git 命令 |
| `../utils/plans.js` | `getPlansDirectory` 计划目录管理 |
| `../utils/Shell.js` | `setCwd` 设置当前工作目录 |
| `../utils/worktree.js` | 工作树核心操作 |
| `./CustomSelect/select.js` | `Select` 单选组件 |
| `./design-system/Dialog.js` | 对话框容器组件 |
| `./Spinner.js` | `Spinner` 加载指示器 |

### worktree.js 核心函数
| 函数 | 用途 |
|-----|------|
| `getCurrentWorktreeSession()` | 获取当前工作树会话信息 |
| `cleanupWorktree()` | 清理删除工作树 |
| `keepWorktree()` | 保留工作树 |
| `killTmuxSession()` | 终止 tmux 会话 |

### 调用方
| 文件路径 | 调用场景 |
|---------|---------|
| `src/screens/REPL.tsx` | 退出流程中检测工作树会话 |
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | 退出工作树工具 |

### 调用链
```
REPL.tsx / ExitWorktreeTool.ts
    ↓
WorktreeExitDialog
    ↓
Select (单选交互)
Dialog (容器渲染)
worktree.js (核心操作)
```

## 依赖与外部交互

### 1. Git 命令执行
组件通过 `execFileNoThrow` 执行以下 Git 命令：

```typescript
// 检测未提交更改
await execFileNoThrow('git', ['status', '--porcelain']);

// 检测提交数量
await execFileNoThrow('git', ['rev-list', '--count', `${worktreeSession.originalHeadCommit}..HEAD`]);
```

### 2. 工作树会话类型
```typescript
type WorktreeSession = {
  originalCwd: string;        // 原始工作目录
  worktreePath: string;       // 工作树路径
  worktreeName: string;       // 工作树名称
  worktreeBranch?: string;    // 工作树分支
  originalBranch?: string;    // 原始分支
  originalHeadCommit?: string;// 原始 HEAD 提交
  sessionId: string;          // 会话 ID
  tmuxSessionName?: string;   // Tmux 会话名称
  hookBased?: boolean;        // 是否基于 hook
  creationDurationMs?: number;// 创建耗时
  usedSparsePaths?: boolean;  // 是否使用稀疏检出
};
```

### 3. 状态管理交互
- **`saveWorktreeState(null)`**：清除工作树状态持久化
- **`getPlansDirectory.cache.clear?.()`**：清除计划目录缓存
- **`setCwd()`**：更新当前工作目录状态

### 4. 分析事件
```typescript
// 保留工作树事件
logEvent('tengu_worktree_kept', {
  commits: commitCount,
  changed_files: changes.length
});

// 删除工作树事件
logEvent('tengu_worktree_removed', {
  commits: commitCount,
  changed_files: changes.length
});
```

## 风险、边界与改进建议

### 风险点

#### 1. 循环依赖处理
**风险**：`recordWorktreeExit` 使用 inline require 打破循环依赖。
```typescript
// sessionStorage → commands → exit → ExitFlow → WorktreeExitDialog
```
**影响**：中 - 如果模块结构改变，可能导致运行时错误。
**建议**：考虑重构模块依赖关系，或使用依赖注入。

#### 2. Git 命令失败处理
**风险**：Git 命令失败时（如不在 git 仓库），错误处理可能不够完善。
**影响**：中 - 可能导致状态检测不准确。
**建议**：增加更完善的错误处理和用户提示。

#### 3. 状态竞争条件
**风险**：`status` 状态变更和 `onDone` 回调可能存在竞争条件。
**影响**：低 - 在极端情况下可能导致回调多次触发。
**建议**：使用 ref 或更严格的状态管理。

### 边界情况

#### 1. 无工作树会话
```typescript
if (!worktreeSession) {
  onDone('No active worktree session found', { display: 'system' });
  return null;
}
```

#### 2. 快速状态切换
- `loading` → `removing` → `done`（自动清理路径）
- 需要确保 `onDone` 只在 `status === 'done'` 时触发一次

#### 3. Tmux 会话不存在
`killTmuxSession` 失败时继续执行，不阻塞流程：
```typescript
if (worktreeSession.tmuxSessionName) {
  await killTmuxSession(worktreeSession.tmuxSessionName);
}
```

#### 4. 清理失败
`cleanupWorktree` 失败时记录错误但仍标记为完成：
```typescript
try {
  await cleanupWorktree();
} catch (error) {
  logForDebugging(`Failed to clean up worktree: ${error}`, { level: 'error' });
  setResultMessage('Worktree cleanup failed, exiting anyway');
}
```

### 改进建议

#### 1. 更详细的状态报告
当前仅显示更改文件数量和提交数量，可扩展为：
```typescript
// 建议：显示具体更改文件列表
subtitle = `You have ${changes.length} uncommitted files:\n${changes.slice(0, 5).join('\n')}${changes.length > 5 ? '\n...' : ''}`;
```

#### 2. 强制删除选项
当工作树清理失败时，提供强制删除选项：
```typescript
{
  label: 'Force remove worktree',
  value: 'force-remove',
  description: 'Remove worktree even if cleanup failed'
}
```

#### 3. 提交推送提醒
当有未推送提交时提醒用户：
```typescript
// 检测未推送提交
const { stdout: unpushed } = await execFileNoThrow('git', [
  'rev-list', '--count', 'HEAD', '--not', '--remotes'
]);
```

#### 4. 工作树状态快照
在退出前创建状态快照，支持后续恢复：
```typescript
await createWorktreeSnapshot(worktreeSession);
```

#### 5. 批量操作支持
支持同时管理多个工作树：
```typescript
type Props = {
  worktreeSessions: WorktreeSession[];
  onDone: (results: WorktreeExitResult[]) => void;
};
```

### 测试建议
1. **单元测试**：
   - 状态检测逻辑（有/无更改、有/无提交）
   - 选项生成逻辑（有/无 tmux）
   - 消息生成逻辑（各种组合情况）

2. **集成测试**：
   - 与 worktree.js 的集成
   - Git 命令执行和错误处理
   - 会话状态持久化

3. **边界测试**：
   - 无工作树会话
   - Git 命令失败
   - 快速取消/选择
   - 大数量更改文件

4. **E2E 测试**：
   - 完整的工作树创建-使用-退出流程
   - Tmux 会话生命周期
