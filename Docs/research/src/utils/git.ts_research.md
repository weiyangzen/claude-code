# git.ts 深度研究文档

## 场景与职责

`git.ts` 是 Claude Code CLI 的核心 Git 操作模块，提供全面的 Git 仓库操作功能：

1. **仓库发现**：查找 Git 根目录、解析工作树和子模块
2. **状态查询**：获取分支、HEAD、远程 URL、仓库状态等
3. **变更检测**：检测未推送提交、工作区变更、文件状态
4. **状态保存**：为问题报告保存完整的 Git 状态（包括未跟踪文件）
5. **安全检测**：检测裸仓库攻击向量

该模块是 50+ 个模块的依赖，是 Git 集成的核心基础设施。

## 功能点目的

### 1. 仓库发现
- `findGitRoot`: 向上遍历目录树查找 `.git` 目录或文件
- `findCanonicalGitRoot`: 解析工作树到主仓库根目录
- `resolveCanonicalRoot`: 验证工作树结构的安全性

### 2. 状态查询
- `getIsGit`: 检查当前目录是否在 Git 仓库中
- `getBranch`/`getDefaultBranch`: 获取当前/默认分支
- `getHead`: 获取当前 HEAD SHA
- `getRemoteUrl`: 获取远程 URL
- `getIsClean`: 检查工作区是否干净
- `getChangedFiles`/`getFileStatus`: 获取变更文件列表

### 3. 变更检测
- `getIsHeadOnRemote`: 检查 HEAD 是否在远程
- `hasUnpushedCommits`: 检查是否有未推送提交
- `stashToCleanState`: 暂存所有变更（包括未跟踪文件）

### 4. 状态保存（Issue 报告）
- `preserveGitStateForIssue`: 保存完整的 Git 状态用于问题复现
- `captureUntrackedFiles`: 捕获未跟踪文件内容
- `findRemoteBase`: 查找最佳的远程基准分支

### 5. 仓库标识
- `normalizeGitRemoteUrl`: 规范化远程 URL（支持 SSH/HTTPS/代理格式）
- `getRepoRemoteHash`: 获取仓库的唯一哈希标识
- `getGithubRepo`: 获取 GitHub 仓库名（owner/repo）

### 6. 安全检测
- `isCurrentDirectoryBareGitRepo`: 检测裸仓库攻击向量

## 具体技术实现

### 关键数据结构

```typescript
export type GitRepoState = {
  commitHash: string
  branchName: string
  remoteUrl: string | null
  isHeadOnRemote: boolean
  isClean: boolean
  worktreeCount: number
}

export type PreservedGitState = {
  remote_base_sha: string | null
  remote_base: string | null
  patch: string
  untracked_files: Array<{ path: string; content: string }>
  format_patch: string | null
  head_sha: string | null
  branch_name: string | null
}

export type GitFileStatus = {
  tracked: string[]
  untracked: string[]
}
```

### 关键流程

#### Git 根目录查找
```typescript
const findGitRootImpl = memoizeWithLRU(
  (startPath: string): string | typeof GIT_ROOT_NOT_FOUND => {
    let current = resolve(startPath)
    const root = current.substring(0, current.indexOf(sep) + 1) || sep
    let statCount = 0

    while (current !== root) {
      try {
        const gitPath = join(current, '.git')
        statCount++
        const stat = statSync(gitPath)
        // .git 可以是目录（普通仓库）或文件（工作树/子模块）
        if (stat.isDirectory() || stat.isFile()) {
          return current.normalize('NFC')
        }
      } catch {
        // 继续向上遍历
      }
      const parent = dirname(current)
      if (parent === current) break
      current = parent
    }
    // 检查根目录...
    return GIT_ROOT_NOT_FOUND
  },
  path => path,
  50,  // LRU 缓存大小
)
```

#### 规范化根目录解析（工作树支持）
```typescript
const resolveCanonicalRoot = memoizeWithLRU(
  (gitRoot: string): string => {
    try {
      // 工作树中 .git 是包含 "gitdir: <path>" 的文件
      const gitContent = readFileSync(join(gitRoot, '.git'), 'utf-8').trim()
      if (!gitContent.startsWith('gitdir:')) return gitRoot

      const worktreeGitDir = resolve(gitRoot, gitContent.slice('gitdir:'.length).trim())
      const commonDir = resolve(worktreeGitDir, readFileSync(join(worktreeGitDir, 'commondir'), 'utf-8').trim())

      // 安全验证：
      // 1. worktreeGitDir 必须是 commonDir/worktrees/ 的直接子目录
      // 2. gitdir 文件必须指回 gitRoot/.git
      if (resolve(dirname(worktreeGitDir)) !== join(commonDir, 'worktrees')) {
        return gitRoot
      }
      const backlink = realpathSync(readFileSync(join(worktreeGitDir, 'gitdir'), 'utf-8').trim())
      if (backlink !== join(realpathSync(gitRoot), '.git')) {
        return gitRoot
      }

      // 裸仓库工作树处理
      if (basename(commonDir) !== '.git') {
        return commonDir.normalize('NFC')
      }
      return dirname(commonDir).normalize('NFC')
    } catch {
      return gitRoot
    }
  },
  root => root,
  50,
)
```

#### 远程 URL 规范化
支持多种格式：
- SSH: `git@github.com:owner/repo.git`
- HTTPS: `https://github.com/owner/repo.git`
- SSH URL: `ssh://git@github.com/owner/repo`
- CCR 代理: `http://...@127.0.0.1:PORT/git/owner/repo`

```typescript
export function normalizeGitRemoteUrl(url: string): string | null {
  // SSH 格式
  const sshMatch = trimmed.match(/^git@([^:]+):(.+?)(?:\.git)?$/)
  if (sshMatch) return `${sshMatch[1]}/${sshMatch[2]}`.toLowerCase()

  // HTTPS/SSH URL 格式
  const urlMatch = trimmed.match(/^(?:https?|ssh):\/\/(?:[^@]+@)?([^/]+)\/(.+?)(?:\.git)?$/)
  if (urlMatch) {
    // CCR 代理处理...
    return `${host}/${path}`.toLowerCase()
  }
  return null
}
```

#### 裸仓库攻击检测
```typescript
export function isCurrentDirectoryBareGitRepo(): boolean {
  const fs = getFsImplementation()
  const cwd = getCwd()

  const gitPath = join(cwd, '.git')
  try {
    const stats = fs.statSync(gitPath)
    if (stats.isFile()) return false  // 工作树/子模块
    if (stats.isDirectory()) {
      const gitHeadPath = join(gitPath, 'HEAD')
      try {
        if (fs.statSync(gitHeadPath).isFile()) return false  // 正常仓库
      } catch {
        // .git/HEAD 不存在，继续检查裸仓库指标
      }
    }
  } catch {
    // 没有 .git，继续检查裸仓库指标
  }

  // 检查裸仓库指标：HEAD、objects/、refs/
  try { if (fs.statSync(join(cwd, 'HEAD')).isFile()) return true } catch {}
  try { if (fs.statSync(join(cwd, 'objects')).isDirectory()) return true } catch {}
  try { if (fs.statSync(join(cwd, 'refs')).isDirectory()) return true } catch {}
  return false
}
```

### 文件系统缓存优化

模块大量使用 `git/gitFilesystem.ts` 提供的缓存：
- `getCachedBranch`, `getCachedHead`, `getCachedRemoteUrl`, `getCachedDefaultBranch`
- 使用 `fs.watchFile` 监控 `.git/HEAD`、`.git/config`、分支引用文件
- 文件变更时自动失效缓存

## 关键代码路径与文件引用

### 核心导出
- `findGitRoot` / `findCanonicalGitRoot` - 仓库根目录查找
- `getIsGit` / `isAtGitRoot` / `dirIsInGitRepo` - Git 状态检查
- `getBranch` / `getDefaultBranch` / `getHead` / `getRemoteUrl` - 基本信息
- `getIsClean` / `getChangedFiles` / `getFileStatus` - 变更检测
- `getIsHeadOnRemote` / `hasUnpushedCommits` - 远程状态
- `normalizeGitRemoteUrl` / `getRepoRemoteHash` / `getGithubRepo` - 仓库标识
- `preserveGitStateForIssue` - 状态保存
- `stashToCleanState` - 暂存变更
- `isCurrentDirectoryBareGitRepo` - 安全检测
- `gitExe` - 缓存的 Git 可执行文件路径

### 依赖关系

**被以下模块导入**（50+ 个）：
- `src/utils/fsOperations.ts` - 文件系统操作
- `src/utils/getWorktreePaths.ts` - 工作树检测
- `src/utils/ghPrStatus.ts` - PR 状态
- `src/utils/gitDiff.ts` - Git 差异
- `src/utils/detectRepository.ts` - 仓库检测
- `src/services/...` - 多个服务模块
- `src/tools/...` - 多个工具模块
- ... 等

**依赖的模块**：
- `src/utils/cwd.ts` - 当前工作目录
- `src/utils/debug.ts` - 调试日志
- `src/utils/diagLogs.ts` - 诊断日志
- `src/utils/execFileNoThrow.ts` - 命令执行
- `src/utils/fsOperations.ts` - 文件系统操作
- `src/utils/git/gitFilesystem.ts` - Git 文件系统缓存
- `src/utils/log.ts` - 错误日志
- `src/utils/memoize.ts` - 缓存工具
- `src/utils/which.ts` - 命令查找
- `src/constants/files.ts` - 文件类型常量

### 文件位置
- 源码：`src/utils/git.ts` (926 行)

## 依赖与外部交互

### Node.js 内置模块
- `crypto` - SHA256 哈希
- `fs` - 同步文件操作
- `fs/promises` - 异步文件操作
- `path` - 路径操作

### 项目内部依赖
- `src/utils/cwd.ts`
- `src/utils/debug.ts`
- `src/utils/diagLogs.ts`
- `src/utils/execFileNoThrow.ts`
- `src/utils/fsOperations.ts`
- `src/utils/git/gitFilesystem.ts`
- `src/utils/log.ts`
- `src/utils/memoize.ts`
- `src/utils/which.ts`
- `src/constants/files.ts`

### 外部依赖
- `lodash-es/memoize.js` - 记忆化

### 外部工具
- Git 命令行工具

## 风险、边界与改进建议

### 已知风险

1. **缓存一致性**：`findGitRoot` 的 LRU 缓存可能在目录结构变更后返回过期结果
2. **符号链接竞争**：`resolveCanonicalRoot` 中的 `realpathSync` 存在 TOCTOU 风险
3. **性能问题**：`preserveGitStateForIssue` 可能读取大量未跟踪文件
4. **Git 版本依赖**：某些命令可能需要特定版本的 Git

### 边界情况

1. **浅克隆**：`preserveGitStateForIssue` 检测浅克隆并回退到简单模式
2. **分离 HEAD**：分支名显示为 'HEAD'
3. **无远程**：远程相关函数返回 null
4. **工作树**：正确处理主仓库和工作树的关系
5. **子模块**：作为独立仓库处理
6. **合并/变基状态**：由调用方通过 `gitDiff.ts` 的 `isInTransientGitState` 处理

### 安全考虑

1. **工作树验证**：`resolveCanonicalRoot` 验证工作树结构防止路径遍历攻击
2. **裸仓库检测**：`isCurrentDirectoryBareGitRepo` 防止恶意仓库执行钩子
3. **文件大小限制**：`captureUntrackedFiles` 有 500MB/文件和 5GB/总计限制

### 改进建议

1. **异步化**：将更多同步操作改为异步，避免阻塞事件循环
2. **错误分类**：细化错误类型，区分 Git 错误、文件系统错误、权限错误
3. **增量更新**：为 `preserveGitStateForIssue` 添加增量模式
4. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
5. **配置化**：允许通过配置自定义某些行为（如文件大小限制）
6. **性能监控**：添加更多性能指标收集
