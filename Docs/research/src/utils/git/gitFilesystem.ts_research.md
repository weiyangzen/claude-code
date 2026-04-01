# 研究文档：src/utils/git/gitFilesystem.ts

## 场景与职责

`gitFilesystem.ts` 是 Claude Code CLI 的**纯文件系统级 git 状态读取层**，核心目标是在**不启动 git 子进程**的情况下，通过直接读写 `.git/` 目录下的文件来获取和监听仓库状态。这对于热路径（如 prompt 构建、worktree 快速恢复、分支显示）至关重要，因为每次 `git` 子进程调用都有 ~15ms 的启动开销。

主要职责包括：
1. **解析 `.git` 目录位置**：支持普通仓库、worktree、submodule（`.git` 是指针文件的情况）
2. **读取 HEAD 状态**：解析当前分支名或 detached HEAD 的 SHA
3. **解析引用（ref）到 SHA**：支持 loose ref、packed-refs、symref 链式跳转
4. **文件监听与缓存**：通过 `GitFileWatcher` 监听 `.git/HEAD`、`.git/config`、当前分支 ref 文件的变化，并提供缓存的 branch/HEAD/remoteUrl/defaultBranch
5. **worktree 与 shallow 检测**：支持多 worktree 场景和 shallow clone 判断

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `resolveGitDir(startPath?)` | 解析实际 `.git` 目录路径；支持 worktree/submodule 的 `gitdir:` 指针文件；带缓存 |
| `clearResolveGitDirCache()` | 清除 `resolveGitDir` 的缓存；供 `/clear` 和测试使用 |
| `readGitHead(gitDir)` | 解析 `.git/HEAD`，返回分支名或 detached SHA |
| `resolveRef(gitDir, ref)` | 将 ref（如 `refs/heads/main`）解析为 SHA；支持 loose + packed-refs + symref |
| `getCommonDir(gitDir)` | 读取 worktree 的 `commondir` 文件，指向共享 git 目录 |
| `readRawSymref(gitDir, refPath, branchPrefix)` | 读取 loose symref 文件并提取分支名 |
| `isSafeRefName(name)` / `isValidGitSha(s)` | 安全校验：防止篡改的 `.git/HEAD` 或 ref 文件中的路径遍历、参数注入、shell 元字符 |
| `getCachedBranch()` / `getCachedHead()` / `getCachedRemoteUrl()` / `getCachedDefaultBranch()` | 带文件监听的缓存读取接口 |
| `resetGitFileWatcher()` | 重置 watcher；供测试使用 |
| `getHeadForDir(cwd)` | 为任意目录读取 HEAD SHA（不走 watcher 缓存） |
| `readWorktreeHeadSha(worktreePath)` | 直接读取 worktree 的 `.git` 指针文件并解析 HEAD SHA |
| `getRemoteUrlForDir(cwd)` | 为任意目录读取 remote origin URL |
| `isShallowClone()` | 判断是否为 shallow clone |
| `getWorktreeCountFromFs()` | 通过读取 `worktrees/` 目录统计 worktree 数量 |

## 具体技术实现

### 1. resolveGitDir — .git 目录解析

**流程：**
1. 以 `startPath ?? getCwd()` 为起点，调用 `findGitRoot(cwd)` 找到仓库根目录
2. 检查 `join(root, '.git')` 的类型：
   - 是 **文件** → 读取内容，解析 `gitdir: <path>`，返回解析后的绝对路径（worktree/submodule）
   - 是 **目录** → 直接返回该路径（普通仓库）
   - 不存在/出错 → 返回 `null`

**缓存机制：**
- 使用 `Map<string, string | null>` 按 `cwd` 缓存结果
- 缓存不清除的话会持续到进程结束；`clearResolveGitDirCache()` 用于 `/clear` 时刷新

### 2. readGitHead — HEAD 文件解析

`.git/HEAD` 的三种格式：
- `ref: refs/heads/<branch>\n` → 返回 `{ type: 'branch', name }`
- `ref: <other-ref>\n`（如 rebase/bisect 期间的 symref）→ 调用 `resolveRef` 解析为 SHA，返回 `{ type: 'detached', sha }`
- 原始 40/64 位 hex SHA → 返回 `{ type: 'detached', sha }`

**安全校验：**
- 分支名通过 `isSafeRefName()` 校验
- SHA 通过 `isValidGitSha()` 校验
- 若校验失败返回 `null`，防止篡改的 HEAD 文件导致后续 shell 注入或路径遍历

### 3. resolveRef — ref 到 SHA 的解析

**解析优先级：**
1. **Loose ref 文件**：直接读取 `join(dir, ref)` 的内容
   - 内容以 `ref:` 开头 → 递归调用 `resolveRef(dir, target)`（symref 链）
   - 内容是 SHA → 返回
2. **Packed-refs**：读取 `join(dir, 'packed-refs')`，逐行匹配
   - 跳过 `#` 注释行和 `^` peeled tag 行
   - 格式：`<sha> <refname>`
3. **Worktree common dir 回退**：若当前 gitDir 未找到，尝试 `getCommonDir(gitDir)` 再解析

### 4. GitFileWatcher — 文件监听与缓存

**设计：** 单例模式（`const gitWatcher = new GitFileWatcher()`），懒启动（`ensureStarted()`）。

**监听文件：**
- `.git/HEAD` — 分支切换、detached HEAD 变化
- `.git/config`（或 commonDir 下的 config）— remote URL 变化
- `refs/heads/<current-branch>` — 当前分支有新 commit

**实现细节：**
- 使用 Node.js `fs.watchFile(path, { interval: WATCH_INTERVAL_MS }, callback)` 轮询监听
- `WATCH_INTERVAL_MS` 在测试环境为 10ms，生产环境为 1000ms
- HEAD 变化时触发 `onHeadChanged()`：先 `invalidate()` 缓存，然后 `waitForScrollIdle()` 后重新设置分支 ref 监听器
- 分支切换时，会 `unwatchFile` 旧分支 ref 文件，并 `watchFile` 新分支 ref 文件
- 通过 `registerCleanup` 注册进程退出时的 `unwatchFile` 清理

**缓存语义（get<T>）：**
- `dirty = false` 时直接返回缓存值
- `dirty = true` 或首次访问时，执行异步 `compute()`
- **竞态处理**：在 `compute()` 开始之前先清 `dirty = false`；若计算期间文件又变了，`invalidate()` 会重新置 `dirty = true`，下次 `get()` 会再次计算，避免返回陈旧值

**缓存键与计算函数：**
| 键 | 计算函数 | 说明 |
|---|---------|------|
| `branch` | `computeBranch` | 当前分支名，detached 时返回 `'HEAD'` |
| `head` | `computeHead` | 当前 HEAD 的 SHA |
| `remoteUrl` | `computeRemoteUrl` | `remote.origin.url` |
| `defaultBranch` | `computeDefaultBranch` | 默认分支（从 `origin/HEAD` symref 或 `main`/`master` 推断） |

### 5. 安全校验函数

**`isSafeRefName(name)`：**
- 拒绝空字符串、`-` 开头、`/` 开头
- 拒绝包含 `..` 的字符串
- 拒绝路径组件为 `.` 或空（如 `foo/./bar`、`foo//bar`）
- 只允许 ASCII 字母、数字、`/`、`.`、`_`、`+`、`-`、`@`
- 覆盖所有合法 git 分支名，同时拒绝 shell 元字符、空格、反引号、`$`、反斜杠等

**`isValidGitSha(s)`：**
- 只接受完整长度：40 位 hex（SHA-1）或 64 位 hex（SHA-256）
- 拒绝缩写 SHA 和任意非 hex 字符

## 关键代码路径与文件引用

### 内部调用关系

```
resolveGitDir
  ├── findGitRoot (from src/utils/git.js)
  └── parseGitConfigValue (from gitConfigParser.ts)

readGitHead
  ├── isSafeRefName
  ├── isValidGitSha
  └── resolveRef
        ├── resolveRefInDir
        │     ├── isSafeRefName
        │     └── isValidGitSha
        └── getCommonDir

GitFileWatcher
  ├── resolveGitDir
  ├── getCommonDir
  ├── readGitHead
  ├── resolveRef
  ├── parseGitConfigValue
  ├── waitForScrollIdle (src/bootstrap/state.js)
  └── registerCleanup (src/utils/cleanupRegistry.js)
```

### 外部调用方

| 调用方文件 | 调用符号 | 用途 |
|-----------|---------|------|
| `src/utils/git.ts` | `resolveGitDir`, `getCachedBranch`, `getCachedHead`, `getCachedRemoteUrl`, `getCachedDefaultBranch`, `getWorktreeCountFromFs`, `isShallowCloneFs` | 封装为 `getGitDir`, `getBranch`, `getHead`, `getRemoteUrl`, `getDefaultBranch`, `getWorktreeCount`, `isShallowClone` |
| `src/utils/worktree.ts` | `resolveGitDir`, `getCommonDir`, `readWorktreeHeadSha`, `resolveRef` | worktree 创建/恢复、hooksPath 配置、base branch 解析 |
| `src/utils/commitAttribution.ts` | `getRemoteUrlForDir`, `resolveGitDir` | 仓库分类（内部/外部）、transient git 状态检测 |
| `src/utils/githubRepoPathMapping.ts` | `getRemoteUrlForDir` | 验证本地路径是否对应预期的 GitHub repo |
| `src/utils/plugins/pluginVersioning.ts` | `getHeadForDir` | 计算插件目录的 git SHA 作为版本 |
| `src/utils/plugins/installedPluginsManager.ts` | `getHeadForDir` | 同上 |
| `src/utils/deepLink/banner.ts` | `getCommonDir` | 读取 `FETCH_HEAD` 的 mtime，判断仓库是否陈旧 |
| `src/commands/clear/caches.ts` | `clearResolveGitDirCache` | `/clear` 时刷新 git 目录解析缓存 |

### 直接读写的文件

- `join(root, '.git')`（作为文件或目录）
- `join(gitDir, 'config')`
- `join(gitDir, 'HEAD')`
- `join(gitDir, 'commondir')`
- `join(gitDir, ref)`（loose ref）
- `join(gitDir, 'packed-refs')`
- `join(gitDir, 'refs/remotes/origin/HEAD')`
- `join(gitDir, 'shallow')`
- `join(commonDir, 'worktrees')` 目录
- `join(worktreePath, '.git')`（`readWorktreeHeadSha`）

## 依赖与外部交互

### 导入依赖

```typescript
import { unwatchFile, watchFile } from 'fs'
import { readdir, readFile, stat } from 'fs/promises'
import { join, resolve } from 'path'
import { waitForScrollIdle } from '../../bootstrap/state.js'
import { registerCleanup } from '../cleanupRegistry.js'
import { getCwd } from '../cwd.js'
import { findGitRoot } from '../git.js'
import { parseGitConfigValue } from './gitConfigParser.js'
```

### 无 git 子进程

模块完全通过文件系统操作实现，不调用任何 `git` 命令。这是其存在的主要价值。

## 风险、边界与改进建议

### 已知限制与风险

1. **watchFile 的轮询开销**：
   - 使用 `fs.watchFile`（轮询模式）而非 `fs.watch`（事件驱动）。在大型 monorepo 或高频操作场景下，每秒轮询可能带来轻微 CPU 开销。
   - 优势是跨平台稳定、支持监听不存在的文件（新分支首次 commit 时 ref 文件从无到有）。

2. **packed-refs 的线性扫描**：
   - 每次缓存失效时，若 loose ref 不存在，会逐行扫描 `packed-refs`。对于超大型仓库（refs 数量极多），这可能成为瓶颈。
   - 实际观察：packed-refs 通常几万行以内，扫描耗时在毫秒级。

3. **worktree 分支 ref 不存在**：
   - 新创建的分支在首次 commit 前没有 loose ref 文件。`watchFile` 可以监听不存在的文件，所以能正确触发；但 `resolveRef` 会返回 `null`，`computeHead` 会回退到空字符串。

4. **安全性假设**：
   - `isSafeRefName` 和 `isValidGitSha` 是防御性编程，针对".git/HEAD 被恶意篡改"的假设威胁模型。但 `.git` 目录本身通常由 git 保护（文件权限），实际攻击面较小。
   - 允许列表（allowlist）策略足够保守，可能误杀某些极端分支名（如包含非 ASCII 字符），但这类分支名在常规开发中极为罕见。

5. **缓存生命周期**：
   - `resolveGitDirCache` 是进程级永久缓存。若用户在会话期间手动移动 `.git` 目录或切换 worktree，缓存不会自动失效（除非调用 `clearResolveGitDirCache`）。

### 改进建议

1. **增加单元测试**：
   - 模拟 `.git` 为指针文件的场景（worktree/submodule）
   - 模拟 `packed-refs` 解析（含注释、peeled tags、多行）
   - 模拟 `GitFileWatcher` 的缓存失效与竞态条件
   - 安全校验边界测试（`isSafeRefName` 对危险字符的拒绝、`isValidGitSha` 对缩写 SHA 的拒绝）

2. **考虑 packed-refs 索引优化**：
   - 若未来遇到 packed-refs 扫描瓶颈，可在首次扫描后构建 `Map<ref, sha>` 缓存，并在 `packed-refs` 文件变化时重建。

3. **watchFile → watch 的渐进迁移**：
   - 在支持 `fs.watch` 的平台上，可考虑对存在的文件使用 `fs.watch`（更低开销），对不存在的文件保留 `fs.watchFile`。需要仔细处理平台差异和重复触发问题。

4. **更完善的错误日志**：
   - 当前多处使用 `catch { return null }` 的静默失败模式。对于诊断 worktree/HEAD 解析问题，可考虑在 `logForDebugging` 级别记录异常原因。
