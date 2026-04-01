# Teleport 功能研究文档

## 1. 场景与职责

### 1.1 功能定位

Teleport 是 Claude Code CLI 的核心功能之一，用于实现**本地 CLI 会话与远程 Claude Code Web (CCR) 会话之间的双向互操作**。它允许用户：

1. **从 CLI 创建远程 CCR 会话** (`--remote` 或内部调用)：将本地工作上下文推送到云端，在远程容器中继续执行
2. **从 CLI 恢复远程 CCR 会话** (`--teleport [sessionId]`): 将远程会话拉取到本地，在本地 CLI 中继续对话

### 1.2 典型使用场景

| 场景 | 描述 |
|------|------|
| 跨设备工作 | 用户在办公室使用 Claude Code Web，回家后通过 `claude --teleport <sessionId>` 在本地 CLI 恢复会话 |
| 长时间任务 | 本地启动任务后推送到远程容器，关闭本地终端，任务在云端继续运行 |
| 团队协作 | 会话可以在不同开发者之间传递（通过共享 session ID） |
| 代码审查 | 通过 teleport 将审查会话同步到本地进行深度检查 |

### 1.3 目标文件状态

**`src/commands/teleport/index.js` 当前是一个存根（stub）文件**：

```javascript
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

该文件仅作为命令注册系统的占位符存在，实际 teleport 功能通过以下方式提供：
- CLI 参数 `--teleport` 和 `--remote` 在 `main.tsx` 中直接处理
- 相关 UI 组件和工具函数分散在 `src/utils/teleport.tsx`、`src/utils/teleport/` 目录下

---

## 2. 功能点目的

### 2.1 核心功能模块

```
┌─────────────────────────────────────────────────────────────────┐
│                        Teleport 功能架构                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  远程会话创建 │  │  远程会话恢复 │  │  会话状态同步        │  │
│  │  (toRemote)  │  │  (resume)    │  │  (poll/events)       │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Git Bundle  │  │  环境选择    │  │  权限请求桥接        │  │
│  │  (seed)      │  │  (env)       │  │  (permission)        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 功能点详细说明

#### 2.2.1 远程会话创建 (`teleportToRemote`)

**目的**：在 Claude.ai 云端创建一个新的 CCR 会话，将本地代码上下文传送过去

**关键特性**：
- **双模式代码传输**：
  - GitHub 模式：通过 GitHub App 从仓库克隆（需要 GitHub 授权）
  - Bundle 模式：创建 git bundle 上传（适用于本地-only 仓库或无 GitHub 授权场景）
- **自动环境选择**：优先选择 `anthropic_cloud` 环境，支持 BYOC (Bring Your Own Compute)
- **分支管理**：自动生成 `claude/<task-name>` 分支，支持重用结果分支

#### 2.2.2 远程会话恢复 (`teleportResumeCodeSession`)

**目的**：将云端 CCR 会话恢复到本地 CLI

**关键特性**：
- **仓库验证**：确保本地仓库与远程会话关联的仓库匹配
- **分支切换**：自动检出远程会话对应的分支
- **消息恢复**：获取远程会话的完整对话历史

#### 2.2.3 Git Bundle 系统 (`createAndUploadGitBundle`)

**目的**：为没有 GitHub 授权的仓库提供代码传输能力

**降级策略**：
```
--all (完整仓库) → HEAD (当前分支) → squashed-root (单提交快照)
```

**WIP 处理**：通过 `git stash create` 捕获未提交更改，打包到 bundle 中

#### 2.2.4 远程会话管理 (`RemoteSessionManager`)

**目的**：管理本地 CLI 与远程 CCR 容器之间的实时连接

**职责**：
- WebSocket 连接管理（订阅远程事件）
- HTTP POST 发送用户消息
- 权限请求/响应桥接（将远程工具权限请求映射到本地 UI）
- 自动重连和超时检测

---

## 3. 具体技术实现

### 3.1 关键流程

#### 3.1.1 远程会话创建流程

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  用户输入    │────▶│  前置条件检查    │────▶│  仓库检测        │
│  --remote   │     │  (登录/Git状态)  │     │  (GitHub/Bundle) │
└─────────────┘     └─────────────────┘     └────────┬────────┘
                                                     │
                         ┌───────────────────────────┘
                         ▼
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  返回会话ID  │◀────│  调用 Sessions  │◀────│  环境选择        │
│  和 URL      │     │  API 创建会话   │     │  (anthropic_cloud)│
└─────────────┘     └─────────────────┘     └─────────────────┘
```

**代码路径**：`src/utils/teleport.tsx`:`teleportToRemote()` (行 730-1190)

**关键决策逻辑**：
```typescript
// 1. 检查 GitHub App 是否安装（预检）
const ghViable = await checkGithubAppInstalled(owner, repo, signal);

// 2. 如果 GitHub 不可行且功能开关开启，使用 Bundle 模式
if (!ghViable && bundleSeedGateOn) {
  const bundle = await createAndUploadGitBundle({...});
  seedBundleFileId = bundle.fileId;
}

// 3. 选择环境（优先 anthropic_cloud）
const cloudEnv = environments.find(env => env.kind === 'anthropic_cloud');

// 4. 调用 Sessions API 创建会话
const response = await axios.post(url, {
  title: sessionTitle,
  events: initialMessage ? [userEvent] : [],
  session_context: {
    sources: gitSource ? [gitSource] : [],
    seed_bundle_file_id: seedBundleFileId,
    outcomes: gitOutcome ? [gitOutcome] : [],
  },
  environment_id: environmentId,
}, { headers });
```

#### 3.1.2 远程会话恢复流程

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  用户输入    │────▶│  验证仓库匹配    │────▶│  获取会话日志    │
│  --teleport │     │  (本地vs远程)   │     │  (v2 API/ingress)│
│  <sessionId>│     └─────────────────┘     └────────┬────────┘
└─────────────┘                                      │
                         ┌───────────────────────────┘
                         ▼
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  启动 REPL  │◀────│  处理消息恢复    │◀────│  检出目标分支    │
│  继续会话   │     │  (添加恢复标记)  │     │  (git checkout) │
└─────────────┘     └─────────────────┘     └─────────────────┘
```

**代码路径**：`src/utils/teleport.tsx`:`teleportResumeCodeSession()` (行 430-503)

**仓库验证逻辑**：
```typescript
const repoValidation = await validateSessionRepository(sessionData);
switch (repoValidation.status) {
  case 'match':
  case 'no_repo_required':
    // 继续恢复
    break;
  case 'not_in_repo':
    // 错误：需要在正确的仓库中运行
    throw new TeleportOperationError(`You must run claude --teleport ${sessionId} from a checkout of ${sessionRepo}`);
  case 'mismatch':
    // 错误：仓库不匹配
    throw new TeleportOperationError(`You must run claude --teleport ${sessionId} from a checkout of ${sessionDisplay}.\nThis repo is ${currentDisplay}`);
}
```

#### 3.1.3 Bundle 创建流程

```
┌─────────────────┐
│  git stash create│──▶ 捕获未提交更改 (refs/seed/stash)
└────────┬────────┘
         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  git bundle     │────▶│  检查大小限制   │────▶│  上传到 Files   │
│  create --all   │     │  (100MB默认)   │     │  API            │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │
         ▼ (如果超过限制)
┌─────────────────┐     ┌─────────────────┐
│  HEAD-only      │────▶│  squashed-root  │
│  (当前分支)     │     │  (单提交快照)   │
└─────────────────┘     └─────────────────┘
```

**代码路径**：`src/utils/teleport/gitBundle.ts`:`createAndUploadGitBundle()` (行 152-292)

### 3.2 数据结构

#### 3.2.1 SessionResource (会话资源)

```typescript
// src/utils/teleport/api.ts
export type SessionResource = {
  type: 'session'
  id: string
  title: string | null
  session_status: SessionStatus  // 'requires_action' | 'running' | 'idle' | 'archived'
  environment_id: string
  created_at: string
  updated_at: string
  session_context: SessionContext
}

export type SessionContext = {
  sources: SessionContextSource[]        // Git 仓库或知识库
  cwd: string
  outcomes: Outcome[] | null             // 结果分支配置
  custom_system_prompt: string | null
  append_system_prompt: string | null
  model: string | null
  seed_bundle_file_id?: string           // Bundle 模式文件 ID
  github_pr?: { owner: string; repo: string; number: number }
  reuse_outcome_branches?: boolean
}
```

#### 3.2.2 GitSource (代码源)

```typescript
export type GitSource = {
  type: 'git_repository'
  url: string                           // e.g., https://github.com/owner/repo
  revision?: string | null              // 分支或 commit
  allow_unrestricted_git_push?: boolean // 是否允许推送到任意分支
}
```

#### 3.2.3 TeleportRemoteResponse (恢复响应)

```typescript
// src/utils/conversationRecovery.ts
export type TeleportRemoteResponse = {
  log: Message[]    // 会话消息历史
  branch?: string   // 关联的 git 分支
}
```

### 3.3 API 协议

#### 3.3.1 Sessions API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/v1/sessions` | POST | 创建新会话 |
| `/v1/sessions` | GET | 列出用户会话 |
| `/v1/sessions/{id}` | GET | 获取会话详情 |
| `/v1/sessions/{id}` | PATCH | 更新会话标题 |
| `/v1/sessions/{id}/events` | POST | 发送用户消息 |
| `/v1/sessions/{id}/events` | GET | 轮询会话事件 |
| `/v1/sessions/{id}/archive` | POST | 归档会话 |
| `/v1/code/sessions/{id}/teleport-events` | GET | 获取 teleport 事件 (v2) |
| `/v1/session_ingress/session/{id}` | GET | 获取会话日志 (旧版) |

#### 3.3.2 请求头规范

```typescript
const headers = {
  'Authorization': `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': 'ccr-byoc-2025-07-29',  // CCR BYOC 功能标志
  'x-organization-uuid': orgUUID,
}
```

### 3.4 命令/接口

#### 3.4.1 CLI 参数

```bash
# 创建远程会话并附加
claude --remote "实现用户认证功能"

# 恢复特定远程会话
claude --teleport <session-id>

# 交互式选择要恢复的会话
claude --teleport
```

**代码位置**：`src/main.tsx` (行 3863-3864)

```typescript
program.addOption(new Option('--teleport [session]', 'Resume a teleport session, optionally specify session ID').hideHelp());
```

#### 3.4.2 核心函数接口

```typescript
// 远程会话创建
async function teleportToRemote(options: {
  initialMessage: string | null;
  branchName?: string;
  title?: string;
  description?: string;
  model?: string;
  permissionMode?: PermissionMode;
  signal: AbortSignal;
  useBundle?: boolean;
  skipBundle?: boolean;
  reuseOutcomeBranch?: string;
  githubPr?: { owner: string; repo: string; number: number };
}): Promise<TeleportToRemoteResponse | null>;

// 远程会话恢复
async function teleportResumeCodeSession(
  sessionId: string, 
  onProgress?: TeleportProgressCallback
): Promise<TeleportRemoteResponse>;

// 带进度 UI 的恢复
async function teleportWithProgress(
  root: Root, 
  sessionId: string
): Promise<TeleportResult>;
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心实现文件

| 文件路径 | 职责 | 关键导出 |
|---------|------|---------|
| `src/commands/teleport/index.js` | **存根文件**，仅注册命令占位 | `{ isEnabled: () => false, isHidden: true, name: 'stub' }` |
| `src/utils/teleport.tsx` | Teleport 核心逻辑 | `teleportToRemote`, `teleportResumeCodeSession`, `teleportWithProgress`, `validateGitState`, `checkOutTeleportedSessionBranch` |
| `src/utils/teleport/api.ts` | Sessions API 客户端 | `fetchSession`, `fetchCodeSessionsFromSessionsAPI`, `sendEventToRemoteSession`, `updateSessionTitle`, `pollRemoteSessionEvents` |
| `src/utils/teleport/gitBundle.ts` | Git Bundle 创建上传 | `createAndUploadGitBundle` |
| `src/utils/teleport/environments.ts` | 环境管理 | `fetchEnvironments`, `createDefaultCloudEnvironment` |

### 4.2 UI 组件文件

| 文件路径 | 职责 |
|---------|------|
| `src/components/TeleportResumeWrapper.tsx` | 远程会话选择器包装器 |
| `src/components/TeleportProgress.tsx` | 恢复进度 UI |
| `src/components/TeleportStash.tsx` | Git stash 确认对话框 |
| `src/components/TeleportError.tsx` | Teleport 错误处理（登录、Git 状态） |
| `src/components/TeleportRepoMismatchDialog.tsx` | 仓库不匹配时选择本地路径 |

### 4.3 远程会话管理

| 文件路径 | 职责 |
|---------|------|
| `src/remote/RemoteSessionManager.ts` | WebSocket 连接管理、消息收发、权限桥接 |
| `src/hooks/useRemoteSession.ts` | React Hook 封装远程会话状态管理 |
| `src/hooks/useTeleportResume.tsx` | Teleport 恢复逻辑 Hook |

### 4.4 入口与调用

| 文件路径 | 职责 |
|---------|------|
| `src/main.tsx` | CLI 参数解析、`--teleport`/`--remote` 处理逻辑 (行 1254-3568) |
| `src/dialogLaunchers.tsx` | Teleport 相关对话框启动器 |
| `src/commands.ts` | 命令注册（`teleport` 命令列入 `INTERNAL_ONLY_COMMANDS`） |

### 4.5 关键代码流程图

```
main.tsx --teleport
    │
    ├─▶ teleport === true ──▶ launchTeleportResumeWrapper()
    │                              │
    │                              ▼
    │                    TeleportResumeWrapper.tsx
    │                              │
    │                              ▼
    │                    useTeleportResume() hook
    │                              │
    │                              ▼
    │                    teleportResumeCodeSession()
    │                              │
    │                    ┌─────────┴─────────┐
    │                    ▼                   ▼
    │         validateSessionRepository   fetchSession
    │                    │                   │
    │                    ▼                   ▼
    │         checkOutTeleportedSessionBranch  getSessionLogsViaOAuth/getTeleportEvents
    │                    │
    │                    ▼
    │         processMessagesForTeleportResume
    │
    └─▶ teleport === string ──▶ teleportWithProgress()
                                   │
                                   ▼
                         TeleportProgress.tsx (UI)
                                   │
                                   ▼
                         teleportResumeCodeSession()
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

```
┌─────────────────────────────────────────────────────────────────┐
│                      Teleport 依赖关系图                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐│
│  │  Git 工具   │  │  OAuth 认证 │  │  API 客户端 │  │  UI 组件 ││
│  │  git.ts     │  │  auth.ts    │  │  axios      │  │  Ink     ││
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬────┘│
│         │                │                │               │     │
│         ▼                ▼                ▼               ▼     │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    teleport.tsx                           │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.1.1 Git 相关依赖

| 依赖文件 | 用途 |
|---------|------|
| `src/utils/git.ts` | `getIsClean`, `findGitRoot`, `getDefaultBranch`, `stashToCleanState` |
| `src/utils/detectRepository.ts` | `detectCurrentRepositoryWithHost`, `parseGitRemote`, `parseGitHubRepository` |
| `src/utils/githubRepoPathMapping.ts` | `getKnownPathsForRepo`, `validateRepoAtPath` |

#### 5.1.2 认证依赖

| 依赖文件 | 用途 |
|---------|------|
| `src/utils/auth.ts` | `getClaudeAIOAuthTokens`, `checkAndRefreshOAuthTokenIfNeeded`, `isClaudeAISubscriber` |
| `src/services/oauth/client.ts` | `getOrganizationUUID` |

#### 5.1.3 状态管理依赖

| 依赖文件 | 用途 |
|---------|------|
| `src/bootstrap/state.ts` | `setTeleportedSessionInfo`, `getOriginalCwd`, `getSessionId` |
| `src/utils/conversationRecovery.ts` | `deserializeMessages`, `restoreSkillStateFromMessages` |

### 5.2 外部 API 依赖

#### 5.2.1 Claude.ai Sessions API

- **基础 URL**: `https://api.claude.ai` (通过 `getOauthConfig().BASE_API_URL` 配置)
- **认证**: OAuth 2.0 Bearer Token
- **主要端点**: `/v1/sessions`, `/v1/environment_providers`

#### 5.2.2 Files API

- **用途**: 上传 git bundle 文件
- **端点**: `/v1/files`
- **返回**: `file_id` 用于 `seed_bundle_file_id`

#### 5.2.3 Session Ingress API (旧版)

- **用途**: 获取会话日志（逐步迁移到 v2）
- **端点**: `/v1/session_ingress/session/{id}`

### 5.3 环境变量

| 变量名 | 用途 |
|--------|------|
| `CCR_FORCE_BUNDLE=1` | 强制使用 bundle 模式（跳过 GitHub 预检） |
| `CCR_ENABLE_BUNDLE=1` | 启用 bundle 功能（配合 GrowthBook 开关） |
| `CLAUDE_AFTER_LAST_COMPACT=1` | 只获取最后一次压缩后的日志 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 仓库匹配风险

**风险描述**：用户在错误的仓库中运行 `claude --teleport <sessionId>` 可能导致代码应用到错误的项目。

**现有防护**：
- `validateSessionRepository()` 函数比较本地仓库与远程会话关联的仓库
- 支持 GitHub Enterprise 主机名比较（避免跨实例混淆）
- `TeleportRepoMismatchDialog` 允许用户选择正确的本地路径

**代码位置**：`src/utils/teleport.tsx` (行 367-421)

#### 6.1.2 Bundle 大小限制

**风险描述**：大仓库（>100MB）无法通过 bundle 模式传输。

**现有处理**：
- 三级降级策略：`--all` → `HEAD` → `squashed-root`
- 超过限制时提示用户设置 GitHub 授权

**代码位置**：`src/utils/teleport/gitBundle.ts` (行 50-146)

#### 6.1.3 认证过期

**风险描述**：OAuth token 过期导致 teleport 失败。

**现有处理**：
- `checkAndRefreshOAuthTokenIfNeeded()` 自动刷新
- 清晰的错误提示引导用户重新登录

### 6.2 边界情况

#### 6.2.1 空仓库

```typescript
// gitBundle.ts 行 174-189
const refCheck = await execFileNoThrowWithCwd(
  gitExe(),
  ['for-each-ref', '--count=1', 'refs/'],
  { cwd: gitRoot },
)
if (refCheck.code === 0 && refCheck.stdout.trim() === '') {
  return {
    success: false,
    error: 'Repository has no commits yet',
    failReason: 'empty_repo',
  }
}
```

#### 6.2.2 仓库无 GitHub 远程

- 检测：`detectCurrentRepositoryWithHost()` 返回 `null`
- 处理：自动切换到 bundle 模式（如果启用）

#### 6.2.3 会话不存在

- API 返回 404
- 处理：抛出 `TeleportOperationError`，提示用户检查 session ID

### 6.3 改进建议

#### 6.3.1 代码组织改进

**当前问题**：
- `src/commands/teleport/index.js` 是存根，实际逻辑分散在多个文件
- 新开发者难以找到 teleport 的完整实现

**建议**：
1. 将 `src/commands/teleport/index.js` 重命名为实际实现入口
2. 或者添加注释文档指向实际实现位置
3. 考虑将 `src/utils/teleport.tsx` 拆分为更小的模块

#### 6.3.2 错误处理改进

**当前问题**：
- Bundle 失败时的错误消息分散在多个位置
- 用户难以区分 GitHub 授权问题和网络问题

**建议**：
1. 统一错误码和错误消息
2. 添加更详细的故障排除指南链接

#### 6.3.3 性能优化

**当前问题**：
- `fetchCodeSessionsFromSessionsAPI` 获取所有会话，可能数据量大
- Bundle 创建对大型仓库较慢

**建议**：
1. 添加分页或搜索功能
2. 考虑增量 bundle（只传输变更）

#### 6.3.4 测试覆盖

**当前状态**：未找到 teleport 相关的单元测试文件

**建议**：
1. 为 `validateSessionRepository` 添加单元测试
2. 为 Bundle 创建逻辑添加测试（使用临时 git 仓库）
3. 为 API 客户端添加 mock 测试

#### 6.3.5 文档改进

**建议**：
1. 在 `--teleport` 帮助中显示更多使用示例
2. 添加 teleport 工作流程的架构图到内部文档
3. 记录常见错误和解决方案

---

## 7. 附录

### 7.1 相关文件完整列表

```
src/
├── commands/
│   └── teleport/
│       └── index.js                    # 存根命令定义
├── components/
│   ├── TeleportError.tsx               # 错误处理组件
│   ├── TeleportProgress.tsx            # 进度 UI 组件
│   ├── TeleportRepoMismatchDialog.tsx  # 仓库不匹配对话框
│   ├── TeleportResumeWrapper.tsx       # 恢复流程包装器
│   └── TeleportStash.tsx               # Git stash 对话框
├── hooks/
│   ├── useRemoteSession.ts             # 远程会话 Hook
│   └── useTeleportResume.tsx           # Teleport 恢复 Hook
├── remote/
│   └── RemoteSessionManager.ts         # WebSocket 会话管理
├── services/api/
│   └── sessionIngress.ts               # Session Ingress API 客户端
└── utils/
    ├── teleport.tsx                    # 核心 teleport 逻辑
    ├── teleport/
    │   ├── api.ts                      # Sessions API 客户端
    │   ├── environments.ts             # 环境管理
    │   └── gitBundle.ts                # Git Bundle 创建
    ├── background/remote/
    │   └── preconditions.ts            # 前置条件检查
    └── conversationRecovery.ts         # 会话恢复工具
```

### 7.2 关键类型定义

```typescript
// Teleport 进度步骤
type TeleportProgressStep = 'validating' | 'fetching_logs' | 'fetching_branch' | 'checking_out' | 'done';

// 仓库验证结果
type RepoValidationResult = {
  status: 'match' | 'mismatch' | 'not_in_repo' | 'no_repo_required' | 'error';
  sessionRepo?: string;
  currentRepo?: string | null;
  sessionHost?: string;
  currentHost?: string;
  errorMessage?: string;
};

// Bundle 上传结果
type BundleUploadResult =
  | { success: true; fileId: string; bundleSizeBytes: number; scope: 'all' | 'head' | 'squashed'; hasWip: boolean }
  | { success: false; error: string; failReason?: 'git_error' | 'too_large' | 'empty_repo' };
```

---

*文档生成时间：2026-04-01*  
*研究范围：代码、配置、测试及必要实现上下文*  
*执行器：kimi (k2p5)*
