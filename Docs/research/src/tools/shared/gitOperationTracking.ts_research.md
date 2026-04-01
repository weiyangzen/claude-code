# gitOperationTracking.ts 深度研究文档

## 场景与职责

`gitOperationTracking.ts` 是 Claude Code CLI 中用于**Shell-agnostic Git 操作追踪**的核心模块。它的主要职责是：

1. **检测 Git 操作**：在 Bash/PowerShell 命令执行后，解析命令字符串识别 `git commit`、`git push`、`gh pr create` 等操作
2. **收集使用指标**：通过 OTLP 计数器记录 commit 和 PR 数量，用于产品分析
3. **触发分析事件**：发送 `tengu_git_operation` 等分析事件到遥测系统
4. **支持会话关联**：自动将 PR 与会话关联，实现跨会话追踪

该模块设计为 Shell 无关（Shell-agnostic），因为无论是 Bash 还是 PowerShell，调用 git/gh/glab/curl 的外部二进制参数语法相同。

## 功能点目的

### 1. Git 命令检测

检测以下 Git 操作：
- `git commit` / `git commit --amend` - 普通提交和修改提交
- `git push` - 推送操作
- `git cherry-pick` - 拣选提交
- `git merge` / `git rebase` - 分支合并和变基

### 2. GitHub CLI 操作检测

检测 `gh` 命令行工具的 PR 操作：
- `gh pr create` - 创建 PR
- `gh pr edit` - 编辑 PR
- `gh pr merge` - 合并 PR
- `gh pr comment` - 评论 PR
- `gh pr close` - 关闭 PR
- `gh pr ready` - 标记 PR 为就绪

### 3. GitLab 操作检测

检测 `glab mr create` 命令（GitLab CLI）。

### 4. curl-based PR 创建检测

通过正则表达式检测通过 curl 调用 REST API 创建 PR 的操作（支持 Bitbucket、GitHub API、GitLab API）。

### 5. 会话-PR 关联

当检测到 PR 创建时，自动调用 `linkSessionToPR` 将当前会话与 PR 关联，支持跨会话追踪。

## 具体技术实现

### 关键数据结构

```typescript
// 提交类型
export type CommitKind = 'committed' | 'amended' | 'cherry-picked'

// 分支操作类型
export type BranchAction = 'merged' | 'rebased'

// PR 操作类型
export type PrAction = 'created' | 'edited' | 'merged' | 'commented' | 'closed' | 'ready'

// 检测结果结构
export function detectGitOperation(
  command: string,
  output: string,
): {
  commit?: { sha: string; kind: CommitKind }
  push?: { branch: string }
  branch?: { ref: string; action: BranchAction }
  pr?: { number: number; url?: string; action: PrAction }
}
```

### 核心正则表达式

```typescript
// 构建匹配 git 子命令的正则（支持全局选项如 -c key=val）
function gitCmdRe(subcmd: string, suffix = ''): RegExp {
  return new RegExp(
    `\\bgit(?:\\s+-[cC]\\s+\\S+|\\s+--\\S+=\\S+)*\\s+${subcmd}\\b${suffix}`,
  )
}

const GIT_COMMIT_RE = gitCmdRe('commit')
const GIT_PUSH_RE = gitCmdRe('push')
const GIT_CHERRY_PICK_RE = gitCmdRe('cherry-pick')
const GIT_MERGE_RE = gitCmdRe('merge', '(?!-)')  // 排除 --merge 选项
const GIT_REBASE_RE = gitCmdRe('rebase')
```

### 关键解析函数

#### 1. 解析 Git Commit ID
```typescript
export function parseGitCommitId(stdout: string): string | undefined {
  // 匹配: [branch abc1234] message
  // 或:   [branch (root-commit) abc1234] message
  const match = stdout.match(/\[[\w./-]+(?: \(root-commit\))? ([0-9a-f]+)\]/)
  return match?.[1]
}
```

#### 2. 解析 Git Push 分支
```typescript
function parseGitPushBranch(output: string): string | undefined {
  // 匹配: "abc..def  branch -> branch"
  // 或:   "* [new branch] branch -> branch"
  // 或:   " + abc...def  branch -> branch (forced update)"
  const match = output.match(
    /^\s*[+\-*!= ]?\s*(?:\[new branch\]|\S+\.\.+\S+)\s+\S+\s*->\s*(\S+)/m,
  )
  return match?.[1]
}
```

#### 3. 解析 PR URL
```typescript
function parsePrUrl(url: string): { prNumber: number; prUrl: string; prRepository: string } | null {
  const match = url.match(/https:\/\/github\.com\/([^/]+\/[^/]+)\/pull\/(\d+)/)
  if (match?.[1] && match?.[2]) {
    return {
      prNumber: parseInt(match[2], 10),
      prUrl: url,
      prRepository: match[1],
    }
  }
  return null
}
```

### 核心追踪函数

#### detectGitOperation - 检测操作（用于 UI 摘要）

```typescript
export function detectGitOperation(command: string, output: string): {...}
```

流程：
1. 检测 commit/cherry-pick，提取 SHA
2. 检测 push，提取分支名
3. 检测 merge/rebase，提取目标 ref
4. 检测 PR 操作，提取 PR 号和 URL

#### trackGitOperations - 追踪并记录（用于遥测）

```typescript
export function trackGitOperations(
  command: string,
  exitCode: number,
  stdout?: string,
): void
```

流程：
1. 检查 exitCode === 0（仅成功操作）
2. 匹配 commit → 记录 `tengu_git_operation` + `commit`/`commit_amend` 事件，增加 commitCounter
3. 匹配 push → 记录 `push` 事件
4. 匹配 gh pr 操作 → 记录对应操作事件
5. 匹配 pr create → 增加 prCounter，并异步关联会话到 PR
6. 匹配 glab mr create → 增加 prCounter
7. 匹配 curl POST 到 PR 端点 → 增加 prCounter

## 关键代码路径与文件引用

### 调用方（入口点）

| 文件 | 调用方式 | 用途 |
|------|----------|------|
| `src/tools/BashTool/BashTool.tsx` | `trackGitOperations(command, exitCode, stdout)` | Bash 命令执行后追踪 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | `trackGitOperations(command, exitCode, stdout)` | PowerShell 命令执行后追踪 |
| `src/services/tools/toolExecution.ts` | `detectGitOperation(command, output)` | 工具执行摘要生成 |
| `src/utils/collapseReadSearch.ts` | `detectGitOperation(command, output)` | 折叠搜索结果摘要 |

### 被调用方（依赖）

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.ts` | `getCommitCounter`, `getPrCounter` | 获取 OTLP 计数器 |
| `src/services/analytics/index.ts` | `logEvent`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 发送分析事件 |
| `src/utils/sessionStorage.ts` | `linkSessionToPR` (动态导入) | 会话-PR 关联 |

### 代码路径示例

```
BashTool.tsx (执行命令)
  ↓
执行完成，获取 exitCode 和 stdout/stderr
  ↓
trackGitOperations(command, exitCode, stdout)
  ↓
1. 检测 GIT_COMMIT_RE → logEvent('tengu_git_operation', {operation: 'commit'})
   → getCommitCounter()?.add(1)
2. 检测 GH_PR_ACTIONS → logEvent('tengu_git_operation', {operation: 'pr_create'})
   → getPrCounter()?.add(1)
   → 动态导入 sessionStorage.ts → linkSessionToPR(sessionId, prNumber, prUrl, prRepository)
```

## 依赖与外部交互

### 内部依赖

1. **`src/bootstrap/state.ts`**
   - `getCommitCounter()`: 获取 commit 计数器（OTLP Counter）
   - `getPrCounter()`: 获取 PR 计数器（OTLP Counter）
   - `getSessionId()`: 获取当前会话 ID（用于 PR 关联）

2. **`src/services/analytics/index.ts`**
   - `logEvent()`: 同步发送分析事件
   - `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`: 类型标记，确保不记录敏感信息

3. **`src/utils/sessionStorage.ts`** (动态导入)
   - `linkSessionToPR()`: 将会话 ID 与 PR 信息关联存储

### 外部系统交互

1. **OTLP 指标系统**
   - 通过 `getCommitCounter()` 和 `getPrCounter()` 获取的计数器是 OTLP (OpenTelemetry Protocol) 计数器
   - 计数器在 `src/bootstrap/state.ts` 的 `setMeter()` 中初始化
   - 指标最终发送到遥测后端（如 Datadog）

2. **分析事件系统**
   - `logEvent()` 将事件发送到分析接收器（Analytics Sink）
   - Sink 在应用启动时通过 `attachAnalyticsSink()` 附加
   - 事件最终路由到 Datadog 和 1P 事件日志

## 风险、边界与改进建议

### 风险点

1. **正则表达式误匹配**
   - 风险：`git commit` 正则可能匹配到 `git log` 输出中的示例文本
   - 缓解：通过检查命令字符串（而非仅输出）来确认操作类型
   - 代码体现：`if (GIT_COMMIT_RE.test(command) || isCherryPick)` 先检查命令

2. **PR URL 解析失败**
   - 风险：`gh pr create` 输出格式变化可能导致 PR URL 提取失败
   - 缓解：使用 `parsePrNumberFromText` 作为备用方案，从文本中提取 PR 号

3. **curl PR 检测误报**
   - 风险：curl 检测可能误匹配非 PR 的 POST 请求
   - 缓解：要求同时满足 `isCurlPost` 和 `isPrEndpoint` 两个条件
   - 正则：`/https?:\/\/[^\s'"]*\/(pulls|pull-requests|merge[-_]requests)(?!\/\d)/i`

4. **动态导入循环依赖**
   - 风险：`linkSessionToPR` 的动态导入可能因循环依赖失败
   - 缓解：使用 `void import(...).then(...)` 异步处理，不阻塞主流程

### 边界条件

1. **非零退出码**：`trackGitOperations` 在 `exitCode !== 0` 时直接返回，不记录失败操作
2. **无输出情况**：解析函数在输出为空时返回 `undefined`，不会崩溃
3. **多次 commit**：单次命令中多个 commit 操作只记录第一次检测到的
4. **非 GitHub URL**：`parsePrUrl` 仅支持 GitHub URL，GitLab/Bitbucket 的 PR URL 不会被解析

### 改进建议

1. **支持更多 Git 平台**
   ```typescript
   // 当前仅支持 GitHub
   const match = url.match(/https:\/\/github\.com\/.../)
   // 建议扩展支持 GitLab、Bitbucket、Azure DevOps 等
   ```

2. **增强 curl 检测准确性**
   - 当前仅检测 URL 路径中的 `pulls`、`pull-requests`、`merge_requests`
   - 建议增加对请求体的检测，确认是创建操作而非查询操作

3. **添加更多 Git 操作检测**
   - `git revert` - 回滚提交
   - `git reset` - 重置分支
   - `git tag` - 标签操作
   - `git fetch` / `git pull` - 获取操作

4. **性能优化**
   - 当前每次命令执行都运行多个正则匹配
   - 考虑使用单一预编译正则或 Trie 结构优化多模式匹配

5. **测试覆盖**
   - 建议添加针对各种边缘情况的单元测试：
     - 带全局选项的 git 命令（`git -c commit.gpgsign=false commit`）
     - 多行输出中的 PR URL 提取
     - 强制推送的检测（`git push --force`）

6. **错误处理增强**
   - 当前 `linkSessionToPR` 的失败被静默忽略（`void` 调用）
   - 建议添加调试日志记录失败情况
