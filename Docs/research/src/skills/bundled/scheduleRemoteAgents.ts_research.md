# scheduleRemoteAgents.ts 研究文档

## 场景与职责

`scheduleRemoteAgents.ts` 实现了 `/schedule` 内置技能，用于帮助用户在 claude.ai 云端创建、更新、列出或立即运行**远程定时触发器（triggers）**。这些触发器不同于本地 `/loop` 的 session-only cron，而是在 Anthropic 云端基础设施（CCR, Claude Code Remote）中运行的完全隔离的远程会话。

## 功能点目的

1. **远程触发器生命周期管理**：通过 `RemoteTrigger` 工具与 claude.ai API 交互，支持 list/get/create/update/run 五种动作。
2. **环境自动发现与创建**：拉取用户的可用 CCR 环境列表；若用户没有任何环境，自动创建一个默认的 `anthropic_cloud` 环境。
3. **MCP 连接器匹配**：读取用户当前已连接的 claude.ai MCP 连接器，帮助用户为远程 agent 绑定所需的外部服务（如 Slack、Datadog）。
4. **GitHub 访问预检**：检测当前仓库是否为 GitHub 仓库，并检查 GitHub App 安装状态或 token 同步状态，提前在 prompt 中给出设置提醒。
5. **时区感知**：使用 `Intl.DateTimeFormat().resolvedOptions().timeZone` 获取用户本地时区，指导模型将用户描述的本地时间转换为 UTC cron 表达式。

## 具体技术实现

### 关键流程

- `registerScheduleRemoteAgentsSkill()` → `registerBundledSkill({ name: 'schedule', ... })`
- `getPromptForCommand(args, context)` 入口：
  1. **OAuth 校验**：`getClaudeAIOAuthTokens()?.accessToken` 不存在 → 提示先 `/login`
  2. **拉取环境**：`fetchEnvironments()`
  3. **自动创建环境**：若环境列表为空，调用 `createDefaultCloudEnvironment('claude-code-default')`
  4. **收集 setup notes**：
     - 检测当前 git 仓库及远程（`detectCurrentRepositoryWithHost`、`getRemoteUrl`）
     - 对 github.com 仓库检查远程访问权限（`checkRepoForRemoteAccess`）
     - 读取 MCP 连接器（`getConnectedClaudeAIConnectors`）
  5. **组装 prompt**：`buildPrompt({ userTimezone, connectorsInfo, gitRepoUrl, environmentsInfo, createdEnvironment, setupNotes, needsGitHubAccessReminder, userArgs: args })`

### 数据结构

```ts
type ConnectorInfo = {
  uuid: string
  name: string
  url: string
}
```

- `taggedIdToUUID(taggedId: string): string | null`：将 claude.ai 返回的 `mcpsrv_01...` Base58 标签 ID 解码为 UUID。
- `sanitizeConnectorName(name: string): string`：去除 `claude.ai` 前缀并将非法字符替换为 `-`，确保符合 `mcp_connections` 的 `name` 字段规范（仅允许 `[a-zA-Z0-9_-]`）。

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'schedule'` |
| `allowedTools` | `['RemoteTrigger', 'AskUserQuestion']` |
| `userInvocable` | `true` |
| `isEnabled` | GrowthBook `tengu_surreal_dali` && policy `allow_remote_sessions` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/scheduleRemoteAgents.ts`
- 注册入口：`src/skills/bundled/index.ts`（条件注册：`feature('AGENT_TRIGGERS_REMOTE')`）
- 核心注册器：`src/skills/bundledSkills.ts`
- OAuth 工具：`src/utils/auth.ts`（`getClaudeAIOAuthTokens`）
- 环境 API：`src/utils/teleport/environments.ts`（`fetchEnvironments`、`createDefaultCloudEnvironment`）
- 仓库检测：`src/utils/detectRepository.ts`（`detectCurrentRepositoryWithHost`、`parseGitRemote`）
- Git 工具：`src/utils/git.ts`（`getRemoteUrl`）
- 远程访问预检：`src/utils/background/remote/preconditions.ts`（`checkRepoForRemoteAccess`、`checkGithubAppInstalled`、`checkGithubTokenSynced`）
- MCP 类型：`src/services/mcp/types.ts`（`MCPServerConnection`、`McpClaudeAIProxyServerConfig`）
- 工具常量：`src/tools/RemoteTriggerTool/prompt.ts`（`REMOTE_TRIGGER_TOOL_NAME`）
- 提问工具常量：`src/tools/AskUserQuestionTool/prompt.ts`（`ASK_USER_QUESTION_TOOL_NAME`）
- GrowthBook：`src/services/analytics/growthbook.ts`（`getFeatureValue_CACHED_MAY_BE_STALE`）
- Policy Limits：`src/services/policyLimits/index.ts`（`isPolicyAllowed`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `getClaudeAIOAuthTokens` | 校验用户是否已登录 claude.ai |
| `fetchEnvironments` / `createDefaultCloudEnvironment` | 调用 claude.ai Environment API |
| `checkRepoForRemoteAccess` | 调用 claude.ai OAuth API 检查 GitHub App/token 状态 |
| `context.options.mcpClients` | 从 `ToolUseContext` 读取当前 MCP 客户端连接列表 |
| `REMOTE_TRIGGER_TOOL_NAME` | 在 prompt 中引用正确的远程触发器工具 |

- **外部 API 调用**：
  - `BASE_API_URL/v1/environment_providers`（GET）
  - `BASE_API_URL/v1/environment_providers/cloud/create`（POST）
  - `BASE_API_URL/api/oauth/organizations/{orgUUID}/code/repos/{owner}/{repo}`（GET）
  - `BASE_API_URL/api/oauth/organizations/{orgUUID}/sync/github/auth`（GET）
- **网络超时**：环境 API 调用使用 15 秒 axios timeout。

## 风险、边界与改进建议

1. **边界：Base58 解码的维护负担**：`taggedIdToUUID` 是客户端 workaround，注释明确说明 "TODO(public-ship): 服务端应直接返回原始 UUID"。该内部格式可能随时变更，导致 MCP 连接器匹配失败。
2. **边界：非 GitHub 仓库静默跳过**：对于 GHE/GitLab 等非 github.com 主机，`setupNotes` 不会给出任何远程访问提示，用户可能误以为远程 agent 可以自动克隆这些仓库。
3. **风险：环境创建失败后的降级**：若 `createDefaultCloudEnvironment` 失败，返回的 prompt 仅提示用户访问 `https://claude.ai/code`，没有更具体的错误原因（如 org 配额已满、网络超时）。
4. **风险：setup notes 的静默丢弃**：当用户直接带参数调用 `/schedule` 时，旧版本存在 "setup notes 被计算后静默丢弃" 的 bug，当前版本已通过 `userArgs && setupNotes.length > 0` 的判断在 prompt body 中显式补回。
5. **改进建议**：
   - 推动服务端在 `/v1/mcp_servers` 中返回原始 UUID，移除客户端 `taggedIdToUUID` 解码逻辑。
   - 对 GHE 仓库给出明确的配置指引（如需要手动提供 SSH key 或 PAT）。
   - 在环境创建失败时，将 axios 错误分类（如 403 配额、401 未授权、timeout）并给出针对性提示。
   - 增加对 `mcp_connections` 中 `name` 字段长度/格式的运行时校验，避免提交到 API 后被拒绝。
