# Research: src/utils/teleport/environments.ts

## 场景与职责

`src/utils/teleport/environments.ts` 是 Claude Code 与后端 **Environment Providers API**（`/v1/environment_providers`）的直接客户端。它负责：

- 查询用户可用的远程计算环境（`anthropic_cloud`、`byoc`、`bridge`）。
- 为首次使用且无环境的新用户自动创建一个默认的 `anthropic_cloud` 环境。

该文件是远程会话创建的前置条件：在 `teleportToRemote`、`createBridgeSession`、以及 `/remote-env` 对话框中都会先调用 `fetchEnvironments()` 来获取目标环境列表。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `EnvironmentKind` / `EnvironmentState` | 类型别名：`anthropic_cloud` \| `byoc` \| `bridge`；`active`。 |
| `EnvironmentResource` | 单个环境的资源模型：kind、environment_id、name、created_at、state。 |
| `EnvironmentListResponse` | 后端列表接口的原始响应结构（含 `has_more`、`first_id`、`last_id` 分页字段）。 |
| `fetchEnvironments` | 获取可用环境数组；认证失败或网络错误时抛出带引导文案的 Error。 |
| `createDefaultCloudEnvironment` | 为无环境用户创建默认云端环境，固定配置 Python 3.11 + Node 20。 |

## 具体技术实现

### `fetchEnvironments`
- 请求：`GET ${BASE_API_URL}/v1/environment_providers`
- 认证头：
  ```ts
  {
    ...getOAuthHeaders(accessToken),
    'x-organization-uuid': orgUUID,
  }
  ```
- 注意：**未携带 `anthropic-beta: ccr-byoc-2025-07-29`**（与 Sessions API 其他端点不一致）。
- 超时：15 秒。
- 错误处理：非 200 状态码抛出 `Failed to fetch environments: ...`；网络错误会经过 `toError` + `logError` 后重新抛出。
- 返回：直接取 `response.data.environments`，**忽略 `has_more` 分页字段**。

### `createDefaultCloudEnvironment`
- 请求：`POST ${BASE_API_URL}/v1/environment_providers/cloud/create`
- 请求体硬编码：
  ```ts
  {
    name,                 // 调用方传入
    kind: 'anthropic_cloud',
    description: '',
    config: {
      environment_type: 'anthropic',
      cwd: '/home/user',
      init_script: null,
      environment: {},
      languages: [
        { name: 'python', version: '3.11' },
        { name: 'node', version: '20' },
      ],
      network_config: {
        allowed_hosts: [],
        allow_default_hosts: true,
      },
    },
  }
  ```
- 认证头：除标准 OAuth 头外，还额外携带了 `anthropic-beta: ccr-byoc-2025-07-29`。
- 超时：15 秒。

## 关键代码路径与文件引用

- **本文件**：`src/utils/teleport/environments.ts`（120 行）
- **直接调用方**：
  - `src/utils/teleport/environmentSelection.ts` — 为 UI 提供环境选择信息。
  - `src/utils/teleport.tsx` — `teleportToRemote` 在创建会话前拉取并选择环境。
  - `src/bridge/createSession.ts` — bridge 会话创建。
  - `src/utils/background/remote/preconditions.ts` — `checkHasRemoteEnvironment` 判断用户是否有可用环境。
  - `src/utils/ultraplan/ccrSession.ts` — ultraplan 流程。
  - `src/components/RemoteEnvironmentDialog.tsx` — 通过 `environmentSelection.ts` 间接调用。

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `axios` | HTTP 客户端。 |
| `src/constants/oauth.ts` | `getOauthConfig` 获取 API 基地址。 |
| `src/services/oauth/client.ts` | `getOrganizationUUID` 获取组织 UUID。 |
| `src/utils/auth.ts` | `getClaudeAIOAuthTokens` 获取 access token。 |
| `src/utils/errors.ts` | `toError` 统一错误转换。 |
| `src/utils/log.ts` | `logError` 记录错误日志。 |
| `src/utils/teleport/api.ts` | `getOAuthHeaders` 生成认证头。 |

外部 API 端点：
- `GET  /v1/environment_providers`
- `POST /v1/environment_providers/cloud/create`

## 风险、边界与改进建议

### 风险
1. **无重试机制**：`fetchEnvironments` 在遭遇瞬态 5xx 或网络抖动时直接抛出，导致上层会话创建流程中断。与 `api.ts` 中的 `axiosGetWithRetry` 形成能力落差。
2. **分页被忽略**：`EnvironmentListResponse` 包含 `has_more`、`first_id`、`last_id`，但 `fetchEnvironments` 只返回第一页。若用户环境数量超过后端单页限制（通常 100），部分环境不可见，可能导致选择错误或失败。
3. **默认环境配置硬编码**：`createDefaultCloudEnvironment` 中 Python 3.11 和 Node 20 是写死的。若后端更新默认镜像版本，CLI 创建的默认环境将长期停留在旧版本。
4. **Beta header 不一致**：`fetchEnvironments` 未带 `ccr-byoc-2025-07-29`，而 `createDefaultCloudEnvironment` 带了。若后端未来要求该 header，可能出现 404 或行为差异。

### 边界
- 两个函数都要求用户已登录（OAuth access token 存在），否则抛出明确错误文案，引导用户执行 `/login`。
- `createDefaultCloudEnvironment` 不会检查用户是否已有环境；调用方需自行判断，否则可能重复创建。
- `bridge` 环境在列表中正常返回，但上层 `teleportToRemote` 和 `getEnvironmentSelectionInfo` 会将其降级，避免误选。

### 改进建议
- **增加重试**：为 `fetchEnvironments` 引入与 `api.ts` 一致的指数退避重试（或复用 `axiosGetWithRetry`）。
- **处理分页**：当 `has_more === true` 时，基于 `last_id` 继续翻页，直到获取全部环境。
- **动态默认配置**：与后端协商是否可以通过 `GET /v1/environment_providers/defaults` 获取推荐配置，避免在 CLI 硬编码语言版本。
- **统一 Beta header**：确认 `fetchEnvironments` 是否需要 `ccr-byoc-2025-07-29`，若需要则补齐，保持与 Sessions API 的一致性。
