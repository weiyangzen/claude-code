# reviewRemote.ts 深度研究文档

> 文件路径：`src/commands/review/reviewRemote.ts`  
> 文件大小：11,926 bytes（316 行，含 source map）  
> 研究日期：2026-04-01  
> 执行器：kimi (k2p5)

---

## 一、场景与职责

### 1.1 模块定位

`reviewRemote.ts` 是 Claude Code CLI 中 `/ultrareview` 命令的**远程执行核心引擎**。它负责将本地的代码审查请求转换为云端 CCR（Claude Code Runtime）会话，并处理所有前置条件检查、配额计费校验、环境参数配置和会话启动逻辑。

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **计费门控** | `checkOverageGate()` — 检查用户是否有足够的免费额度或 Extra Usage 余额 |
| **远程启动** | `launchRemoteReview()` — 创建 CCR 会话并注册远程任务 |
| **配额管理** | 维护会话级 `sessionOverageConfirmed` 标志，避免重复弹窗 |
| **参数解析** | 区分 PR 模式（`args` 为纯数字）和分支模式（无参数或分支名） |
| **环境注入** | 配置 bughunter 环境变量（fleet size、timeout、wallclock 等） |

### 1.3 业务场景

| 场景 | 命令示例 | 处理路径 |
|------|----------|----------|
| PR 审查 | `/ultrareview 42` | PR 模式 → `refs/pull/42/head` → GitHub clone |
| 分支审查 | `/ultrareview` | 分支模式 → `git merge-base` → bundle 上传 |
| 团队订阅 | 任意 `/ultrareview` | 跳过配额检查，直接通过 |

---

## 二、功能点目的

### 2.1 功能矩阵

| 导出函数 | 签名 | 目的 |
|----------|------|------|
| `confirmOverage` | `() => void` | 将会话级 `sessionOverageConfirmed` 设为 true |
| `checkOverageGate` | `() => Promise<OverageGate>` | 综合判断用户是否可以启动 ultrareview |
| `launchRemoteReview` | `(args, context, billingNote?) => Promise<ContentBlockParam[] \| null>` | 启动远程审查会话 |

### 2.2 OverageGate 类型设计

```typescript
export type OverageGate =
  | { kind: 'proceed'; billingNote: string }
  | { kind: 'not-enabled' }
  | { kind: 'low-balance'; available: number }
  | { kind: 'needs-confirm' }
```

该 discriminated union 的设计使得调用方（`ultrareviewCommand.tsx`）可以用清晰的 `switch`/`if-else` 处理所有计费状态，无需嵌套条件判断。

---

## 三、具体技术实现

### 3.1 关键数据结构

#### 3.1.1 会话级确认标志

```typescript
// line 36
let sessionOverageConfirmed = false

// line 38-40
export function confirmOverage(): void {
  sessionOverageConfirmed = true
}
```

- **作用域**：模块级变量，进程内全局有效
- **生命周期**：一次 CLI 会话内持久化，重启 CLI 后重置
- **线程安全**：单进程单线程（Node.js），无需额外同步

#### 3.1.2 Bughunter 配置（GrowthBook）

```typescript
const raw = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, unknown> | null>(
  'tengu_review_bughunter_config',
  null
)
```

该配置控制：
- `enabled`: 功能总开关（`ultrareviewEnabled.ts` 也读取同一 flag）
- `fleet_size`: Agent 并发数（默认 5，上限 20）
- `max_duration_minutes`: 单次审查最大时长（默认 10，上限 25）
- `agent_timeout_seconds`: Agent 超时（默认 600，上限 1800）
- `total_wallclock_minutes`: 总墙钟时间（默认 22，上限 27）

#### 3.1.3 合成环境 ID

```typescript
// line 167
const CODE_REVIEW_ENV_ID = 'env_011111111111111111111113'
```

- 这是一个**合成（synthetic）环境 ID**，对应 Go 代码中的 `taggedid.FromUUID(TagEnvironment, UUID{...,0x02})`
- 版本前缀为 `'01'`，不是 Python 的 legacy `tagged_id()` 格式
- 优势：无需每个组织单独配置 code_review 环境，降低使用门槛

### 3.2 关键流程实现

#### 3.2.1 checkOverageGate 流程

```typescript
export async function checkOverageGate(): Promise<OverageGate> {
  // 1. Team/Enterprise 直接放行
  if (isTeamSubscriber() || isEnterpriseSubscriber()) {
    return { kind: 'proceed', billingNote: '' }
  }

  // 2. 并行获取配额和使用情况
  const [quota, utilization] = await Promise.all([
    fetchUltrareviewQuota(),
    fetchUtilization().catch(() => null),
  ])

  // 3. 无配额信息 → 放行（服务端会处理计费）
  if (!quota) {
    return { kind: 'proceed', billingNote: '' }
  }

  // 4. 有免费额度 → 放行并附带计数提示
  if (quota.reviews_remaining > 0) {
    return {
      kind: 'proceed',
      billingNote: ` This is free ultrareview ${quota.reviews_used + 1} of ${quota.reviews_limit}.`,
    }
  }

  // 5. 免费额度耗尽 → 检查 Extra Usage
  if (!utilization) {
    return { kind: 'proceed', billingNote: '' }
  }

  const extraUsage = utilization.extra_usage
  if (!extraUsage?.is_enabled) {
    logEvent('tengu_review_overage_not_enabled', {})
    return { kind: 'not-enabled' }
  }

  const monthlyLimit = extraUsage.monthly_limit
  const usedCredits = extraUsage.used_credits ?? 0
  const available =
    monthlyLimit === null || monthlyLimit === undefined
      ? Infinity
      : monthlyLimit - usedCredits

  if (available < 10) {
    logEvent('tengu_review_overage_low_balance', { available })
    return { kind: 'low-balance', available }
  }

  if (!sessionOverageConfirmed) {
    logEvent('tengu_review_overage_dialog_shown', {})
    return { kind: 'needs-confirm' }
  }

  return {
    kind: 'proceed',
    billingNote: ' This review bills as Extra Usage.',
  }
}
```

**设计要点**：
- `fetchUtilization().catch(() => null)`：网络波动时不阻塞用户，降级为服务端计费
- `monthlyLimit === null` 表示无限额度，数学上视为 `Infinity`
- 低余额阈值硬编码为 `$10`，与前端 billing 页面保持一致

#### 3.2.2 launchRemoteReview 流程

```typescript
export async function launchRemoteReview(
  args: string,
  context: ToolUseContext,
  billingNote?: string,
): Promise<ContentBlockParam[] | null> {
  // 1. 检查远程代理资格
  const eligibility = await checkRemoteAgentEligibility()
  if (!eligibility.eligible) {
    const blockers = eligibility.errors.filter(
      e => e.type !== 'no_remote_environment'
    )
    if (blockers.length > 0) {
      // 返回用户可读的错误列表
      return [{ type: 'text', text: `Ultrareview cannot launch:\n${reasons}` }]
    }
  }

  // 2. 解析参数模式
  const prNumber = args.trim()
  const isPrNumber = /^\d+$/.test(prNumber)

  // 3. 构建 bughunter 环境变量
  const commonEnvVars = {
    BUGHUNTER_DRY_RUN: '1',
    BUGHUNTER_FLEET_SIZE: String(posInt(raw?.fleet_size, 5, 20)),
    BUGHUNTER_MAX_DURATION: String(posInt(raw?.max_duration_minutes, 10, 25)),
    BUGHUNTER_AGENT_TIMEOUT: String(posInt(raw?.agent_timeout_seconds, 600, 1800)),
    BUGHUNTER_TOTAL_WALLCLOCK: String(posInt(raw?.total_wallclock_minutes, 22, 27)),
    ...(process.env.BUGHUNTER_DEV_BUNDLE_B64 && {
      BUGHUNTER_DEV_BUNDLE_B64: process.env.BUGHUNTER_DEV_BUNDLE_B64,
    }),
  }

  // 4. PR 模式 vs 分支模式
  if (isPrNumber) {
    // PR 模式...
  } else {
    // 分支模式...
  }

  // 5. 注册远程任务
  registerRemoteAgentTask({
    remoteTaskType: 'ultrareview',
    session,
    command,
    context,
    isRemoteReview: true,
  })

  // 6. 返回启动成功消息
  return [{ type: 'text', text: `Ultrareview launched for ${target}...` }]
}
```

#### 3.2.3 PR 模式详细逻辑

```typescript
if (isPrNumber) {
  const repo = await detectCurrentRepositoryWithHost()
  if (!repo || repo.host !== 'github.com') {
    logEvent('tengu_review_remote_precondition_failed', {})
    return null
  }
  session = await teleportToRemote({
    initialMessage: null,
    description: `ultrareview: ${repo.owner}/${repo.name}#${prNumber}`,
    signal: context.abortController.signal,
    branchName: `refs/pull/${prNumber}/head`,
    environmentId: CODE_REVIEW_ENV_ID,
    environmentVariables: {
      BUGHUNTER_PR_NUMBER: prNumber,
      BUGHUNTER_REPOSITORY: `${repo.owner}/${repo.name}`,
      ...commonEnvVars,
    },
  })
  command = `/ultrareview ${prNumber}`
  target = `${repo.owner}/${repo.name}#${prNumber}`
}
```

- `refs/pull/N/head` 是 GitHub 的 PR 引用格式，CCR 后端会以此检出代码
- 必须检测到 `github.com` 远程，否则返回 `null`（调用方会显示通用失败消息）

#### 3.2.4 分支模式详细逻辑

```typescript
} else {
  const baseBranch = (await getDefaultBranch()) || 'main'

  // 计算 merge-base SHA（因为远程容器没有 origin/* refs）
  const { stdout: mbOut, code: mbCode } = await execFileNoThrow(
    gitExe(),
    ['merge-base', baseBranch, 'HEAD'],
    { preserveOutputOnError: false },
  )
  const mergeBaseSha = mbOut.trim()
  if (mbCode !== 0 || !mergeBaseSha) {
    return [{
      type: 'text',
      text: `Could not find merge-base with ${baseBranch}. Make sure you're in a git repo with a ${baseBranch} branch.`,
    }]
  }

  // 提前检查空 diff，避免启动无意义的容器
  const { stdout: diffStat, code: diffCode } = await execFileNoThrow(
    gitExe(),
    ['diff', '--shortstat', mergeBaseSha],
    { preserveOutputOnError: false },
  )
  if (diffCode === 0 && !diffStat.trim()) {
    return [{
      type: 'text',
      text: `No changes against the ${baseBranch} fork point. Make some commits or stage files first.`,
    }]
  }

  session = await teleportToRemote({
    initialMessage: null,
    description: `ultrareview: ${baseBranch}`,
    signal: context.abortController.signal,
    useBundle: true,  // 打包本地工作目录
    environmentId: CODE_REVIEW_ENV_ID,
    environmentVariables: {
      BUGHUNTER_BASE_BRANCH: mergeBaseSha,
      ...commonEnvVars,
    },
  })
  if (!session) {
    return [{
      type: 'text',
      text: 'Repo is too large. Push a PR and use `/ultrareview <PR#>` instead.',
    }]
  }
  command = '/ultrareview'
  target = baseBranch
}
```

**设计要点**：
- 使用 `merge-base SHA` 而不是分支名，因为 CCR 容器在 bundle-clone 后会执行 `git remote remove origin`，导致 `origin/main` 等引用消失
- `useBundle: true` 触发 `createAndUploadGitBundle()`，将本地未提交改动也打包上传
- 空 diff 提前拦截，节省云端资源

### 3.3 环境变量配置算法

```typescript
const posInt = (v: unknown, fallback: number, max?: number): number => {
  if (typeof v !== 'number' || !Number.isFinite(v)) return fallback
  const n = Math.floor(v)
  if (n <= 0) return fallback
  return max !== undefined && n > max ? fallback : n
}
```

- **防御性编程**：GrowthBook 缓存可能返回错误类型或越界值，此时回退到安全默认值
- **上限设计**：`total_wallclock_minutes` 上限 27 分钟，为 `RemoteAgentTask` 的 30 分钟轮询超时留出 3 分钟收尾时间

---

## 四、关键代码路径与文件引用

### 4.1 调用链路

```
用户输入 /ultrareview [args]
    │
    ├──→ src/commands/review.ts
    │    └── ultrareview: Command { type: 'local-jsx', load: () => import('./review/ultrareviewCommand.js') }
    │
    ├──→ src/commands/review/ultrareviewCommand.tsx
    │    ├── checkOverageGate() ──→ 本文件 line 52
    │    └── launchRemoteReview() ──→ 本文件 line 128
    │
    ├──→ src/utils/teleport.tsx:730
    │    └── teleportToRemote(options)
    │
    └──→ src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:386
         └── registerRemoteAgentTask({ remoteTaskType: 'ultrareview', isRemoteReview: true })
```

### 4.2 核心文件清单

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `src/commands/review/reviewRemote.ts` | 316 | **本文件**：配额检查、远程启动核心 |
| `src/commands/review/ultrareviewCommand.tsx` | 58 | JSX 命令入口，渲染计费确认对话框 |
| `src/commands/review/UltrareviewOverageDialog.tsx` | 96 | Extra Usage 确认弹窗 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 856 | 远程任务注册与轮询 |
| `src/utils/teleport.tsx` | 1226 | CCR 会话创建（ teleportToRemote ） |
| `src/services/api/ultrareviewQuota.ts` | 38 | 配额 API 客户端 |
| `src/services/api/usage.ts` | 63 | Extra Usage API 客户端 |
| `src/utils/background/remote/remoteSession.ts` | 98 | 远程会话前置条件检查 |
| `src/utils/background/remote/preconditions.ts` | 235 | 详细前置条件实现 |

### 4.3 代码位置速查

| 元素 | 行号 |
|------|------|
| `sessionOverageConfirmed` | 36 |
| `confirmOverage` | 38-40 |
| `OverageGate` 类型 | 42-47 |
| `checkOverageGate` | 52-113 |
| `launchRemoteReview` | 128-316 |
| `CODE_REVIEW_ENV_ID` | 167 |
| `commonEnvVars` | 190-203 |
| PR 模式逻辑 | 208-228 |
| 分支模式逻辑 | 230-292 |

---

## 五、依赖与外部交互

### 5.1 内部依赖图谱

```
reviewRemote.ts
├── @anthropic-ai/sdk/resources/messages.js
│   └── ContentBlockParam
├── ../../services/analytics/growthbook.js
│   └── getFeatureValue_CACHED_MAY_BE_STALE
├── ../../services/analytics/index.js
│   └── logEvent
├── ../../services/api/ultrareviewQuota.js
│   └── fetchUltrareviewQuota
├── ../../services/api/usage.js
│   └── fetchUtilization
├── ../../Tool.js
│   └── ToolUseContext
├── ../../tasks/RemoteAgentTask/RemoteAgentTask.js
│   ├── checkRemoteAgentEligibility
│   ├── formatPreconditionError
│   ├── getRemoteTaskSessionUrl
│   └── registerRemoteAgentTask
├── ../../utils/auth.js
│   ├── isEnterpriseSubscriber
│   └── isTeamSubscriber
├── ../../utils/detectRepository.js
│   └── detectCurrentRepositoryWithHost
├── ../../utils/execFileNoThrow.js
├── ../../utils/git.js
│   ├── getDefaultBranch
│   └── gitExe
└── ../../utils/teleport.js
    └── teleportToRemote
```

### 5.2 外部 API 交互

| API | 端点 | 用途 | 超时 |
|-----|------|------|------|
| Ultrareview Quota | `GET /v1/ultrareview/quota` | 获取免费审查额度 | 5s |
| Usage | `GET /api/oauth/usage` | 获取 Extra Usage 状态 | 5s |
| Sessions | `POST /v1/sessions` | 创建 CCR 会话 | 无（由 teleport.tsx 处理） |

### 5.3 GrowthBook Feature Flag

| Flag | 用途 | 读取方式 |
|------|------|----------|
| `tengu_review_bughunter_config` | 控制 ultrareview 开关及 bughunter 参数 | `getFeatureValue_CACHED_MAY_BE_STALE` |

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 硬编码阈值

```typescript
if (available < 10)  // line 99
```

- `$10` 最低余额阈值硬编码，若后端策略变更需要同步修改代码
- **建议**：从 GrowthBook 配置或 API 响应中读取阈值

#### 6.1.2 `no_remote_environment` 被忽略

```typescript
const blockers = eligibility.errors.filter(
  e => e.type !== 'no_remote_environment'
)
```

- 合成环境 ID 使得 `no_remote_environment` 不再是阻塞条件
- 风险：如果后端某天不再支持合成环境，此过滤逻辑会导致启动失败但错误信息不完整

#### 6.1.3 PR 模式非 GitHub 仓库返回 `null`

```typescript
if (!repo || repo.host !== 'github.com') {
  logEvent('tengu_review_remote_precondition_failed', {})
  return null
}
```

- 返回 `null` 时，`ultrareviewCommand.tsx` 会显示通用错误消息："Ultrareview failed to launch the remote session..."
- 用户无法得知具体原因是 "not a GitHub repo"
- **建议**：返回明确的 `ContentBlockParam[]` 错误提示

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| `args` 为空字符串 | 进入分支模式，审查当前分支相对 base 的改动 |
| `args` 为纯数字但带前导空格 | `args.trim()` 后正常识别为 PR 模式 |
| `args` 为分支名（如 `feature/x`） | 当前代码按分支模式处理，但 `getDefaultBranch()` 仍作为 base |
| merge-base 失败 | 返回明确错误，提示用户检查 git 仓库和 base 分支 |
| 空 diff | 返回明确错误，提示用户先提交或暂存 |
| bundle 上传失败（仓库过大） | 返回明确错误，建议改用 PR 模式 |
| `fetchUtilization` 网络错误 | `.catch(() => null)` 降级，不阻塞启动 |

### 6.3 改进建议

#### 6.3.1 架构层面

1. **拆分职责**
   - `reviewRemote.ts` 当前 316 行，同时负责配额检查、参数解析、环境构建、会话启动
   - 建议拆分为：
     - `overageGate.ts` — 配额与计费逻辑
     - `reviewLaunch.ts` — 会话启动逻辑
     - `reviewEnv.ts` — 环境变量构建

2. **支持非 GitHub 托管**
   - 当前 PR 模式仅限 `github.com`
   - 若 CCR 后端支持 GitLab/Bitbucket PR 引用，可扩展 `repo.host` 检查

#### 6.3.2 可观测性

1. **增加结构化日志**
   - 当前 `logEvent` 仅记录事件名，缺少启动耗时、参数模式、仓库大小等维度
   - 建议增加：
     - `tengu_review_remote_launch_duration_ms`
     - `tengu_review_remote_mode` (`pr` / `branch`)

2. **错误分类细化**
   - `tengu_review_remote_precondition_failed` 事件缺少具体错误类型的 payload
   - 建议将 `precondition_errors` 从字符串拼接改为数组上报

#### 6.3.3 技术债务

| 问题 | 位置 | 建议 |
|------|------|------|
| TODO #22051 | line 7-9 | 待实现 `useBundleMode` 以支持本地未提交状态捕获 |
| 硬编码环境 ID | line 167 | 考虑从配置或 API 获取，避免后端变更导致失效 |
| 硬编码 wallclock 上限 | line 198 | 27 分钟上限与 30 分钟轮询超时耦合，建议集中配置 |

#### 6.3.4 安全考虑

1. **环境变量泄露**
   - `BUGHUNTER_DEV_BUNDLE_B64` 从 `process.env` 透传到远程环境
   - 需要确保 CCR 容器的环境变量隔离性

2. **Bundle 内容**
   - `useBundle: true` 会将本地工作目录（含未提交改动）打包上传
   - 建议增加敏感文件扫描（如 `.env`、密钥文件），或在用户首次使用时明确提示
