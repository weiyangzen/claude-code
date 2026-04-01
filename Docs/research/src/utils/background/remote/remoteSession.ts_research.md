# remoteSession.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`src/utils/background/remote/remoteSession.ts` 是 Claude Code CLI 中后台远程会话（Background Remote Session）的核心类型定义和资格检查模块。它定义了远程会话的数据结构和前置条件检查逻辑。

### 1.2 核心职责
- **类型定义**: 定义后台远程会话的数据结构（`BackgroundRemoteSession`）
- **前置条件类型**: 定义前置条件失败的类型（`BackgroundRemoteSessionPrecondition`）
- **资格检查**: 实现 `checkBackgroundRemoteSessionEligibility()` 函数，检查用户是否有资格创建后台远程会话
- **Bundle 种子模式支持**: 支持通过本地 Git bundle 创建远程会话（无需 GitHub 远程）

### 1.3 使用场景
| 场景 | 说明 |
|------|------|
| 远程 Agent 任务 | `RemoteAgentTask.tsx` 使用此模块检查任务资格 |
| Ultraplan 远程执行 | 计划模式远程执行时的资格验证 |
| Ultrareview 远程代码审查 | 远程代码审查前的条件检查 |
| Autofix PR | 自动修复 PR 功能的远程会话创建 |

---

## 2. 功能点目的

### 2.1 类型定义

#### BackgroundRemoteSession
后台远程会话的完整状态表示：
```typescript
type BackgroundRemoteSession = {
  id: string                    // 会话唯一标识
  command: string               // 执行的命令
  startTime: number             // 开始时间戳
  status: 'starting' | 'running' | 'completed' | 'failed' | 'killed'
  todoList: TodoList            // 待办事项列表
  title: string                 // 会话标题
  type: 'remote_session'        // 固定类型标识
  log: SDKMessage[]             // 会话日志消息
}
```

#### BackgroundRemoteSessionPrecondition
前置条件失败的联合类型：
```typescript
type BackgroundRemoteSessionPrecondition =
  | { type: 'not_logged_in' }           // 未登录 Claude.ai
  | { type: 'no_remote_environment' }   // 无可用远程环境
  | { type: 'not_in_git_repo' }         // 不在 Git 仓库中
  | { type: 'no_git_remote' }           // 无 Git 远程配置
  | { type: 'github_app_not_installed' }// GitHub App 未安装
  | { type: 'policy_blocked' }          // 组织策略阻止
```

### 2.2 资格检查流程
`checkBackgroundRemoteSessionEligibility()` 按优先级执行以下检查：

1. **策略检查**（最高优先级）
   - 调用 `isPolicyAllowed('allow_remote_sessions')`
   - 如果被阻止，立即返回，跳过后续检查

2. **并行检查**（Promise.all）
   - `checkNeedsClaudeAiLogin()` - 是否需要登录
   - `checkHasRemoteEnvironment()` - 是否有远程环境
   - `detectCurrentRepositoryWithHost()` - 检测当前仓库

3. **Git 相关检查**
   - `checkIsInGitRepo()` - 是否在 Git 仓库中
   - Bundle 种子模式检查（可选跳过）
   - `checkGithubAppInstalled()` - GitHub App 安装检查

---

## 3. 具体技术实现

### 3.1 关键流程

#### checkBackgroundRemoteSessionEligibility 流程图
```
开始
│
├─► 检查组织策略 (isPolicyAllowed)
│   └─► 如果被阻止 → 返回 ['policy_blocked']
│
├─► 并行执行三项检查
│   ├─► checkNeedsClaudeAiLogin()
│   ├─► checkHasRemoteEnvironment()
│   └─► detectCurrentRepositoryWithHost()
│
├─► 评估登录状态
│   └─► 需要登录 → 添加 'not_logged_in'
│
├─► 评估远程环境
│   └─► 无环境 → 添加 'no_remote_environment'
│
├─► 评估 Git 状态
│   ├─► 不在 Git 仓库 → 添加 'not_in_git_repo'
│   ├─► Bundle 种子模式开启 → 跳过远程检查
│   ├─► 无 Git 远程 → 添加 'no_git_remote'
│   └─► GitHub 仓库但 App 未安装 → 添加 'github_app_not_installed'
│
└─► 返回错误数组（空数组表示全部通过）
```

### 3.2 Bundle 种子模式

Bundle 种子模式允许用户在没有 GitHub 远程的情况下创建远程会话：

```typescript
// 行 75-79
const bundleSeedGateOn =
  !skipBundle &&
  (isEnvTruthy(process.env.CCR_FORCE_BUNDLE) ||
    isEnvTruthy(process.env.CCR_ENABLE_BUNDLE) ||
    (await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bundle_seed_enabled')))
```

**触发条件**（满足任一）：
- `CCR_FORCE_BUNDLE=1` - 强制使用 bundle 模式
- `CCR_ENABLE_BUNDLE=1` - 启用 bundle 模式
- Feature flag `tengu_ccr_bundle_seed_enabled` 开启

**行为**: 当 bundle 种子模式开启时，只需要 `checkIsInGitRepo()` 通过，跳过 `no_git_remote` 和 `github_app_not_installed` 检查。

### 3.3 数据结构

#### 错误类型到用户消息的映射（在 RemoteAgentTask.tsx 中）
```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx 行 146-161
function formatPreconditionError(error: BackgroundRemoteSessionPrecondition): string {
  switch (error.type) {
    case 'not_logged_in':
      return 'Please run /login and sign in with your Claude.ai account (not Console).'
    case 'no_remote_environment':
      return 'No cloud environment available. Set one up at https://claude.ai/code/onboarding?magic=env-setup'
    case 'not_in_git_repo':
      return 'Background tasks require a git repository. Initialize git or run from a git repository.'
    case 'no_git_remote':
      return 'Background tasks require a GitHub remote. Add one with `git remote add origin REPO_URL`.'
    case 'github_app_not_installed':
      return 'The Claude GitHub app must be installed on this repository first.\nhttps://github.com/apps/claude/installations/new'
    case 'policy_blocked':
      return "Remote sessions are disabled by your organization's policy. Contact your organization admin to enable them."
  }
}
```

### 3.4 协议与配置

| 环境变量 | 用途 |
|----------|------|
| `CCR_FORCE_BUNDLE` | 强制使用 bundle 种子模式 |
| `CCR_ENABLE_BUNDLE` | 启用 bundle 种子模式 |

| Feature Flag | 用途 |
|--------------|------|
| `tengu_ccr_bundle_seed_enabled` | 控制 bundle 种子功能的灰度发布 |

---

## 4. 关键代码路径与文件引用

### 4.1 导出类型和函数
```typescript
// src/utils/background/remote/remoteSession.ts

// 类型导出
export type BackgroundRemoteSession = { ... }
export type BackgroundRemoteSessionPrecondition = 
  | { type: 'not_logged_in' }
  | { type: 'no_remote_environment' }
  | { type: 'not_in_git_repo' }
  | { type: 'no_git_remote' }
  | { type: 'github_app_not_installed' }
  | { type: 'policy_blocked' }

// 函数导出
export async function checkBackgroundRemoteSessionEligibility(
  options?: { skipBundle?: boolean }
): Promise<BackgroundRemoteSessionPrecondition[]>
```

### 4.2 调用方文件

| 调用方 | 使用的函数/类型 | 用途 |
|--------|----------------|------|
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `checkBackgroundRemoteSessionEligibility`, `BackgroundRemoteSessionPrecondition` | 远程 Agent 任务资格检查 |
| `src/utils/teleport.tsx` | `checkGithubAppInstalled`（通过 preconditions.js） | Teleport 预检 |

### 4.3 核心代码片段

#### 资格检查主函数
```typescript
// 行 45-98
export async function checkBackgroundRemoteSessionEligibility({
  skipBundle = false,
}: {
  skipBundle?: boolean
} = {}): Promise<BackgroundRemoteSessionPrecondition[]> {
  const errors: BackgroundRemoteSessionPrecondition[] = []

  // Check policy first - if blocked, no need to check other preconditions
  if (!isPolicyAllowed('allow_remote_sessions')) {
    errors.push({ type: 'policy_blocked' })
    return errors
  }

  const [needsLogin, hasRemoteEnv, repository] = await Promise.all([
    checkNeedsClaudeAiLogin(),
    checkHasRemoteEnvironment(),
    detectCurrentRepositoryWithHost(),
  ])

  if (needsLogin) {
    errors.push({ type: 'not_logged_in' })
  }

  if (!hasRemoteEnv) {
    errors.push({ type: 'no_remote_environment' })
  }

  // When bundle seeding is on, in-git-repo is enough
  const bundleSeedGateOn =
    !skipBundle &&
    (isEnvTruthy(process.env.CCR_FORCE_BUNDLE) ||
      isEnvTruthy(process.env.CCR_ENABLE_BUNDLE) ||
      (await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bundle_seed_enabled')))

  if (!checkIsInGitRepo()) {
    errors.push({ type: 'not_in_git_repo' })
  } else if (bundleSeedGateOn) {
    // has .git/, bundle will work — skip remote+app checks
  } else if (repository === null) {
    errors.push({ type: 'no_git_remote' })
  } else if (repository.host === 'github.com') {
    const hasGithubApp = await checkGithubAppInstalled(
      repository.owner,
      repository.name,
    )
    if (!hasGithubApp) {
      errors.push({ type: 'github_app_not_installed' })
    }
  }

  return errors
}
```

### 4.4 与 RemoteAgentTask 的集成

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx 行 114-141
export type RemoteAgentPreconditionResult = 
  | { eligible: true }
  | { eligible: false; errors: BackgroundRemoteSessionPrecondition[] }

export async function checkRemoteAgentEligibility({
  skipBundle = false
}: { skipBundle?: boolean } = {}): Promise<RemoteAgentPreconditionResult> {
  const errors = await checkBackgroundRemoteSessionEligibility({ skipBundle })
  if (errors.length > 0) {
    return { eligible: false, errors }
  }
  return { eligible: true }
}
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

| 依赖模块 | 用途 |
|----------|------|
| `src/entrypoints/agentSdkTypes.js` | `SDKMessage` 类型 |
| `src/services/analytics/growthbook.js` | Feature flag 检查 |
| `src/services/policyLimits/index.js` | 组织策略检查 |
| `src/utils/detectRepository.js` | 仓库检测 |
| `src/utils/envUtils.js` | 环境变量检查 |
| `src/utils/todo/types.js` | `TodoList` 类型 |
| `./preconditions.js` | 前置条件检查函数 |

### 5.2 依赖关系图

```
remoteSession.ts
├── agentSdkTypes.js (SDKMessage)
├── todo/types.js (TodoList)
├── services/analytics/growthbook.js (checkGate_CACHED_OR_BLOCKING)
├── services/policyLimits/index.js (isPolicyAllowed)
├── utils/detectRepository.js (detectCurrentRepositoryWithHost)
├── utils/envUtils.js (isEnvTruthy)
└── ./preconditions.js
    ├── checkNeedsClaudeAiLogin
    ├── checkHasRemoteEnvironment
    ├── checkIsInGitRepo
    └── checkGithubAppInstalled
```

### 5.3 被依赖关系

```
RemoteAgentTask.tsx
└── remoteSession.ts
    ├── checkBackgroundRemoteSessionEligibility
    └── BackgroundRemoteSessionPrecondition
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 影响 |
|------|------|------|
| **策略检查顺序** | 策略检查在并行检查之前，但返回时跳过了其他检查 | 用户无法同时看到多个问题 |
| **Bundle 模式竞态** | `checkGate_CACHED_OR_BLOCKING` 可能阻塞 | 启动延迟 |
| **GHE 支持缺失** | GitHub Enterprise 仓库不检查 App 安装 | GHE 用户可能遇到运行时错误 |
| **错误累积** | 所有错误一次性返回，可能过多 | 用户被信息淹没 |

### 6.2 边界情况

1. **策略阻止**: 如果 `isPolicyAllowed` 返回 false，其他检查不会执行，用户只看到策略错误
2. **Bundle 模式与 GitHub 远程并存**: Bundle 模式开启时，即使有 GitHub 远程也跳过 App 检查
3. **非 GitHub 主机**: 对于 GHE 或其他 Git 主机，不检查 App 安装（只检查 github.com）
4. **并发调用**: 资格检查函数是无状态的，可以安全并发调用

### 6.3 改进建议

#### 短期改进
1. **错误优先级排序**: 将错误按严重程度排序，最重要的放在前面
2. **并行优化**: 将 `checkIsInGitRepo()` 也放入 `Promise.all` 并行执行
3. **更细粒度的策略检查**: 区分读取策略和写入策略

#### 中期改进
1. **GHE 支持**: 添加对 GitHub Enterprise App 安装的检查
2. **缓存机制**: 缓存资格检查结果，减少重复检查
3. **渐进式检查**: 根据用户操作动态决定需要哪些检查

#### 长期改进
1. **统一检查框架**: 与 `preconditions.ts` 合并，提供统一的检查接口
2. **配置化检查**: 允许调用方配置需要哪些检查
3. **实时状态更新**: 监听登录状态、策略变化，实时更新资格状态

### 6.4 测试建议

| 测试场景 | 验证点 |
|----------|--------|
| 策略阻止 | 验证策略错误单独返回，不执行其他检查 |
| Bundle 模式 | 验证开启后跳过 GitHub 相关检查 |
| 多重错误 | 验证所有错误都被收集并返回 |
| 非 GitHub 仓库 | 验证 GHE 仓库的处理逻辑 |
| 并发调用 | 验证多次并发调用不会产生竞态条件 |

---

## 7. 相关文档链接

- [preconditions.ts 研究文档](./preconditions.ts_research.md)
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` - 远程 Agent 任务实现
- `src/utils/teleport.tsx` - Teleport 主逻辑
- `src/services/policyLimits/index.ts` - 组织策略限制服务
