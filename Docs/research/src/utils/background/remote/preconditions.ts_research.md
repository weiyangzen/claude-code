# preconditions.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`src/utils/background/remote/preconditions.ts` 是 Claude Code CLI 中远程会话（Remote Session）的前置条件检查模块。它负责验证用户是否有权限和能力创建、管理远程 Claude.ai 会话。

### 1.2 核心职责
- **身份验证检查**: 验证用户是否已登录 Claude.ai 账户
- **Git 状态检查**: 验证当前工作目录的 Git 状态是否符合远程操作要求
- **远程环境检查**: 验证用户是否有可用的云端执行环境
- **GitHub 集成检查**: 验证 GitHub App 是否已安装或 GitHub Token 是否已同步
- **仓库访问权限**: 提供分层检查机制判断仓库是否可访问

### 1.3 使用场景
| 场景 | 说明 |
|------|------|
| Teleport 功能 | 用户通过 `--teleport` 恢复远程会话时检查前提条件 |
| 后台远程任务 | 创建后台远程 Agent 任务前的资格验证 |
| Schedule 功能 | `/schedule` 命令检查 GitHub 仓库访问权限 |
| 错误处理 | `TeleportError.tsx` 组件使用这些检查来显示相应的错误提示 |

---

## 2. 功能点目的

### 2.1 检查函数清单

| 函数名 | 目的 | 返回值 |
|--------|------|--------|
| `checkNeedsClaudeAiLogin()` | 检查用户是否需要登录 Claude.ai | `boolean` |
| `checkIsGitClean()` | 检查 Git 工作目录是否干净（无未提交更改） | `boolean` |
| `checkHasRemoteEnvironment()` | 检查用户是否有至少一个远程环境 | `boolean` |
| `checkIsInGitRepo()` | 检查当前目录是否在 Git 仓库中 | `boolean` |
| `checkHasGitRemote()` | 检查当前仓库是否配置了 Git 远程 | `boolean` |
| `checkGithubAppInstalled()` | 检查 GitHub App 是否安装在指定仓库 | `boolean` |
| `checkGithubTokenSynced()` | 检查用户是否通过 /web-setup 同步了 GitHub Token | `boolean` |
| `checkRepoForRemoteAccess()` | 分层检查仓库远程访问权限 | `{ hasAccess, method }` |

### 2.2 分层访问检查策略
`checkRepoForRemoteAccess()` 实现了三级回退策略：
1. **第一优先级**: GitHub App 已安装在仓库上
2. **第二优先级**: 通过 `/web-setup` 同步了 GitHub Token（需 `tengu_cobalt_lantern` feature flag）
3. **无访问权限**: 提示用户设置访问

---

## 3. 具体技术实现

### 3.1 关键流程

#### 3.1.1 checkNeedsClaudeAiLogin 流程
```
1. 检查 isClaudeAISubscriber() - 用户是否是 Claude.ai 订阅者
2. 如果不是订阅者，直接返回 false（不需要登录）
3. 如果是订阅者，调用 checkAndRefreshOAuthTokenIfNeeded()
4. 返回 token 是否需要刷新（true = 需要登录）
```

#### 3.1.2 checkGithubAppInstalled 流程
```
1. 获取 accessToken 和 orgUUID
2. 构造 API URL: /api/oauth/organizations/{orgUUID}/code/repos/{owner}/{repo}
3. 发送 GET 请求（15秒超时，支持 AbortSignal）
4. 解析响应：
   - status === 200 且 status.app_installed === true → 已安装
   - 4XX 错误 → 未安装或无法访问
   - 其他错误 → 记录日志，返回 false（安全失败）
```

#### 3.1.3 checkGithubTokenSynced 流程
```
1. 获取 accessToken 和 orgUUID
2. 构造 API URL: /api/oauth/organizations/{orgUUID}/sync/github/auth
3. 发送 GET 请求（15秒超时）
4. 检查响应：status === 200 且 is_authenticated === true
```

### 3.2 数据结构

#### API 响应类型 (checkGithubAppInstalled)
```typescript
{
  repo: {
    name: string
    owner: { login: string }
    default_branch: string
  }
  status: {
    app_installed: boolean
    relay_enabled: boolean
  } | null
}
```

#### RepoAccessMethod 类型
```typescript
type RepoAccessMethod = 'github-app' | 'token-sync' | 'none'
```

### 3.3 协议与 API

| API 端点 | 用途 | 认证方式 |
|----------|------|----------|
| `GET /api/oauth/organizations/{orgUUID}/code/repos/{owner}/{repo}` | 检查 GitHub App 安装状态 | OAuth Bearer Token + x-organization-uuid header |
| `GET /api/oauth/organizations/{orgUUID}/sync/github/auth` | 检查 GitHub Token 同步状态 | OAuth Bearer Token + x-organization-uuid header |

### 3.4 错误处理策略

所有检查函数都采用**安全失败（fail-safe）**策略：
- 网络错误或 API 失败时返回 `false`（假设条件不满足）
- 使用 `logForDebugging()` 记录详细错误信息
- 4XX 错误通常表示资源不存在或权限不足，不抛出异常

---

## 4. 关键代码路径与文件引用

### 4.1 导出函数
```typescript
// src/utils/background/remote/preconditions.ts
export async function checkNeedsClaudeAiLogin(): Promise<boolean>
export async function checkIsGitClean(): Promise<boolean>
export async function checkHasRemoteEnvironment(): Promise<boolean>
export function checkIsInGitRepo(): boolean
export async function checkHasGitRemote(): Promise<boolean>
export async function checkGithubAppInstalled(owner: string, repo: string, signal?: AbortSignal): Promise<boolean>
export async function checkGithubTokenSynced(): Promise<boolean>
export async function checkRepoForRemoteAccess(owner: string, repo: string): Promise<{ hasAccess: boolean; method: RepoAccessMethod }>
```

### 4.2 调用方文件

| 调用方 | 使用的函数 | 用途 |
|--------|-----------|------|
| `src/utils/background/remote/remoteSession.ts` | `checkNeedsClaudeAiLogin`, `checkHasRemoteEnvironment`, `checkIsInGitRepo`, `checkGithubAppInstalled` | 后台远程会话资格检查 |
| `src/components/TeleportError.tsx` | `checkNeedsClaudeAiLogin`, `checkIsGitClean` | Teleport 错误提示 |
| `src/utils/teleport.tsx` | `checkGithubAppInstalled` | 远程会话创建时的 GitHub 预检 |
| `src/skills/bundled/scheduleRemoteAgents.ts` | `checkRepoForRemoteAccess` | Schedule 功能的仓库访问检查 |

### 4.3 核心代码片段

#### GitHub App 安装检查
```typescript
// 行 78-158
export async function checkGithubAppInstalled(
  owner: string,
  repo: string,
  signal?: AbortSignal,
): Promise<boolean> {
  try {
    const accessToken = getClaudeAIOAuthTokens()?.accessToken
    if (!accessToken) return false

    const orgUUID = await getOrganizationUUID()
    if (!orgUUID) return false

    const url = `${getOauthConfig().BASE_API_URL}/api/oauth/organizations/${orgUUID}/code/repos/${owner}/${repo}`
    const headers = {
      ...getOAuthHeaders(accessToken),
      'x-organization-uuid': orgUUID,
    }

    const response = await axios.get<...>(url, { headers, timeout: 15000, signal })
    
    if (response.status === 200) {
      if (response.data.status) {
        return response.data.status.app_installed
      }
      return false
    }
    return false
  } catch (error) {
    // 4XX 错误通常表示 App 未安装
    if (axios.isAxiosError(error)) {
      const status = error.response?.status
      if (status && status >= 400 && status < 500) {
        return false
      }
    }
    return false
  }
}
```

#### 分层仓库访问检查
```typescript
// 行 221-235
export async function checkRepoForRemoteAccess(
  owner: string,
  repo: string,
): Promise<{ hasAccess: boolean; method: RepoAccessMethod }> {
  if (await checkGithubAppInstalled(owner, repo)) {
    return { hasAccess: true, method: 'github-app' }
  }
  if (
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_lantern', false) &&
    (await checkGithubTokenSynced())
  ) {
    return { hasAccess: true, method: 'token-sync' }
  }
  return { hasAccess: false, method: 'none' }
}
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

| 依赖模块 | 用途 |
|----------|------|
| `src/constants/oauth.js` | 获取 OAuth 配置（BASE_API_URL） |
| `src/services/oauth/client.js` | 获取 organization UUID |
| `src/services/analytics/growthbook.js` | Feature flag 检查（`tengu_cobalt_lantern`） |
| `src/utils/auth.js` | OAuth token 获取与刷新 |
| `src/utils/cwd.js` | 获取当前工作目录 |
| `src/utils/debug.js` | 调试日志记录 |
| `src/utils/detectRepository.js` | 仓库检测 |
| `src/utils/errors.js` | 错误消息格式化 |
| `src/utils/git.js` | Git 状态检查（`getIsClean`, `findGitRoot`） |
| `src/utils/teleport/api.js` | OAuth headers 构造 |
| `src/utils/teleport/environments.js` | 远程环境获取 |

### 5.2 外部依赖

| 包名 | 用途 |
|------|------|
| `axios` | HTTP 请求 |

### 5.3 依赖关系图

```
preconditions.ts
├── auth.js (getClaudeAIOAuthTokens, checkAndRefreshOAuthTokenIfNeeded, isClaudeAISubscriber)
├── oauth/client.js (getOrganizationUUID)
├── oauth.js (getOauthConfig)
├── growthbook.js (getFeatureValue_CACHED_MAY_BE_STALE)
├── git.js (getIsClean, findGitRoot)
├── cwd.js (getCwd)
├── detectRepository.js (detectCurrentRepository)
├── teleport/api.js (getOAuthHeaders)
├── teleport/environments.js (fetchEnvironments)
├── debug.js (logForDebugging)
└── errors.js (errorMessage)
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 影响 |
|------|------|------|
| **API 超时** | 所有 API 调用都有 15 秒超时，慢网络下可能误报 | 用户可能被误判为无权限 |
| **缓存陈旧** | `getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期值 | Feature flag 状态可能不准确 |
| **安全失败** | 错误时返回 false，可能掩盖真实问题 | 调试困难 |
| **竞态条件** | AbortSignal 处理可能不完全 | 取消请求时可能有未处理的异常 |

### 6.2 边界情况

1. **无网络连接**: 所有检查返回 `false`，用户被引导到离线模式
2. **Token 过期**: `checkNeedsClaudeAiLogin()` 会触发刷新流程
3. **非 Git 目录**: `checkIsInGitRepo()` 返回 `false`，阻止远程操作
4. **GHE (GitHub Enterprise)**: `checkGithubAppInstalled` 仅支持 github.com，GHE 仓库会返回 false
5. **本地仓库**: 无远程配置的仓库 `checkHasGitRemote()` 返回 `false`

### 6.3 改进建议

#### 短期改进
1. **增加重试机制**: 对网络错误添加指数退避重试
2. **更细粒度的错误类型**: 区分网络错误、权限错误、配置错误
3. **缓存 GitHub App 状态**: 减少重复 API 调用

#### 中期改进
1. **支持 GHE**: 扩展 `checkGithubAppInstalled` 支持 GitHub Enterprise
2. **并行检查优化**: `checkRepoForRemoteAccess` 中的两个检查可以并行执行
3. **用户反馈优化**: 提供更具体的错误信息和修复指引

#### 长期改进
1. **统一权限模型**: 与后端协商统一的权限检查 API
2. **本地预检缓存**: 缓存用户权限状态，减少启动延迟
3. **离线模式支持**: 更好的离线检测和用户引导

### 6.4 测试建议

| 测试场景 | 验证点 |
|----------|--------|
| 网络超时 | 确保超时后返回 false 不崩溃 |
| Token 刷新 | 验证过期 token 触发刷新流程 |
| 取消信号 | 验证 AbortSignal 正确传播 |
| 4XX 错误 | 验证 404/403 返回 false 不抛出 |
| 并发调用 | 验证多个检查同时执行的安全性 |

---

## 7. 相关文档链接

- [remoteSession.ts 研究文档](./remoteSession.ts_research.md)
- `src/utils/teleport.tsx` - Teleport 主逻辑
- `src/components/TeleportError.tsx` - Teleport 错误 UI
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` - 远程 Agent 任务实现
