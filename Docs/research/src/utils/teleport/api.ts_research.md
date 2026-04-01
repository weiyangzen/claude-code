# Research: src/utils/teleport/api.ts

## 场景与职责

`src/utils/teleport/api.ts` 是 Claude Code 与后端 **Sessions API**（即 CCR / Teleport / Remote Session 体系）交互的核心 HTTP 客户端层。它负责：

- 为所有 Sessions API 请求准备 OAuth 认证凭据（access token + organization UUID）。
- 提供带指数退避重试的 GET 请求能力，用于应对瞬态网络故障。
- 暴露类型定义，映射后端 API 的会话、上下文、来源（source）、产出（outcome）等数据结构。
- 封装对 `/v1/sessions` 及其子资源的具体 REST 调用：列表查询、单条获取、事件发送、标题更新。

该文件是远程会话功能（`--teleport`、Remote Agent、Bridge、Ultraplan 等）的底层协议入口。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `isTransientNetworkError` | 判断 axios 错误是否为可重试的瞬态故障（无响应或 5xx）。4xx 视为不可重试。 |
| `axiosGetWithRetry` | 对 GET 请求执行最多 4 次退避重试（延迟 2s / 4s / 8s / 16s）。 |
| `prepareApiRequest` | 统一校验 access token 与 org UUID 是否就绪；未登录时抛出带 `/login` 引导的错误。 |
| `fetchCodeSessionsFromSessionsAPI` | 获取用户所有远程会话列表，并将后端 `SessionResource[]` 转换为前端兼容的 `CodeSession[]`。 |
| `fetchSession` | 按 ID 获取单个会话详情，用于 resume、验证仓库匹配、轮询元数据。 |
| `getBranchFromSession` | 从会话的 git_repository outcome 中提取第一个分支名，供 teleport resume 时本地 checkout。 |
| `sendEventToRemoteSession` | 向远程会话发送用户消息事件（`user` 类型），支持传入已有 UUID 做去重。 |
| `updateSessionTitle` | 通过 PATCH 更新远程会话标题，用于 `/rename` 等命令的同步。 |
| `getOAuthHeaders` | 生成统一的 OAuth 请求头（Bearer + `anthropic-version: 2023-06-01`）。 |
| `CCR_BYOC_BETA` | Beta header 常量 `ccr-byoc-2025-07-29`，所有 Sessions API 调用必须携带。 |

## 具体技术实现

### 重试策略
- 常量 `TELEPORT_RETRY_DELAYS = [2000, 4000, 8000, 16000]`，对应最多 4 次重试（共 5 次尝试）。
- 仅 `axiosGetWithRetry` 实现了该策略；**POST / PATCH 方法（`sendEventToRemoteSession`、`updateSessionTitle`）无重试**，网络抖动时直接失败或返回 `false`。

### 认证与请求头
- 所有函数通过 `prepareApiRequest()` 获取 `accessToken` 和 `orgUUID`。
- 请求头模板：
  ```ts
  {
    Authorization: `Bearer ${accessToken}`,
    'Content-Type': 'application/json',
    'anthropic-version': '2023-06-01',
    'anthropic-beta': 'ccr-byoc-2025-07-29',
    'x-organization-uuid': orgUUID,
  }
  ```
- 基地址来自 `getOauthConfig().BASE_API_URL`（`src/constants/oauth.ts`）。

### 类型映射
- `SessionResource` / `SessionContext` / `GitSource` / `KnowledgeBaseSource` / `Outcome` 等类型直接对应后端 `api/schemas/sessions/sessions.py` 与 `api/schemas/sandbox.py` 的字段。
- `CodeSessionSchema` 使用 Zod 定义，用于向后兼容旧版 `CodeSession` 类型；`fetchCodeSessionsFromSessionsAPI` 将 `SessionResource` 手动映射为 `CodeSession`，其中 `session_status` 通过类型断言映射到 `CodeSession['status']`。

### 关键流程
1. **列表会话**：`GET /v1/sessions` → `axiosGetWithRetry` → 解析 `ListSessionsResponse` → 遍历 `session_context.sources` 找到 `git_repository` 类型 → 用 `parseGitHubRepository` 提取 `owner/name` → 组装 `CodeSession[]`。
2. **获取单条**：`GET /v1/sessions/${sessionId}` → 15s 超时 → 对 404/401 做特定错误文案转换。
3. **发送事件**：`POST /v1/sessions/${sessionId}/events` → 30s 超时（考虑冷启动容器）→ 构造 `user` 类型事件体 → 仅接受 200/201 为成功。
4. **更新标题**：`PATCH /v1/sessions/${sessionId}` → 无超时显式配置。

## 关键代码路径与文件引用

- **本文件**：`src/utils/teleport/api.ts`（466 行）
- **核心调用方**：
  - `src/utils/teleport.tsx` — 最主要的 teleport 业务 orchestrator（`teleportToRemote`、`teleportResumeCodeSession`、`pollRemoteSessionEvents` 等）。
  - `src/bridge/createSession.ts` — bridge 会话创建与归档。
  - `src/remote/RemoteSessionManager.ts` — Remote Agent 的会话状态管理。
  - `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` — Remote Agent 任务 UI。
  - `src/hooks/useRemoteSession.ts`、`useTeleportResume.tsx` — React 侧 resume 逻辑。
  - `src/utils/ultraplan/ccrSession.ts` — Ultraplan 轮询。
- **依赖文件**：
  - `src/constants/oauth.ts` — `getOauthConfig`
  - `src/services/oauth/client.ts` — `getOrganizationUUID`
  - `src/utils/auth.ts` — `getClaudeAIOAuthTokens`
  - `src/utils/detectRepository.ts` — `parseGitHubRepository`
  - `src/utils/errors.ts`、`src/utils/debug.ts`、`src/utils/log.ts`、`src/utils/sleep.ts`、`src/utils/slowOperations.ts`

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `axios` | HTTP 客户端；直接使用原生实例，未经过统一 API 中间件。 |
| `crypto` (`randomUUID`) | 为 `sendEventToRemoteSession` 生成事件 UUID。 |
| `zod/v4` | `CodeSessionSchema` 的运行时校验。 |
| `src/constants/oauth.ts` | 获取 `BASE_API_URL` 及环境配置。 |
| `src/services/oauth/client.ts` | 获取 organization UUID。 |
| `src/utils/auth.ts` | 获取当前 OAuth access token。 |
| `src/utils/detectRepository.ts` | 将会话中的 git URL 解析为 `owner/name`。 |

外部 API 端点：
- `GET  /v1/sessions`
- `GET  /v1/sessions/{id}`
- `POST /v1/sessions/{id}/events`
- `PATCH /v1/sessions/{id}`

## 风险、边界与改进建议

### 风险
1. **写操作无重试**：`sendEventToRemoteSession` 和 `updateSessionTitle` 在弱网环境下可能因单次请求失败而丢失用户事件或标题同步。虽然函数返回 `false` 让调用方感知，但调用方通常没有进一步重试。
2. **类型断言脆弱性**：`fetchCodeSessionsFromSessionsAPI` 中将 `session_status` 直接 `as CodeSession['status']`；若后端新增或重命名状态，前端编译不会报错，运行时可能产生不匹配。
3. **超时硬编码**：`fetchSession` 15s、`sendEventToRemoteSession` 30s，均无可配置入口；对高延迟网络或超大型会话不够灵活。
4. **缺少分布式追踪**：请求未携带 `x-request-id` 或 trace header，排查后端 5xx 时难以定位。

### 边界
- `isTransientNetworkError` 明确**不**重试 4xx，这意味着后端限流（如 429）也不会被重试。当前实现把 429 当成不可恢复错误处理。
- `fetchCodeSessionsFromSessionsAPI` 假设列表中最多只有一个 `git_repository` source；若后端支持多仓库会话，映射逻辑只取第一个。

### 改进建议
- 为 `sendEventToRemoteSession` 增加有限重试（如 2 次），或对 5xx / 网络错误启用退避。
- 将 `fetchCodeSessionsFromSessionsAPI` 的类型断言替换为 Zod 解析，确保后端 schema 变更时前端能及时发现。
- 暴露可选的 `timeout` 参数给调用方，或根据环境动态调整。
- 在请求头中注入客户端生成的 `x-request-id`，与后端日志串联。
