# 研究文档：src/services/mcp/claudeai.ts

## 场景与职责

`claudeai.ts` 负责将 **Claude.ai 云端配置的 MCP 服务器** 拉取并注入到 Claude Code 的本地 MCP 生态中。企业或团队管理员可以在 Claude.ai 网页控制台（如 `claude.ai/settings/connectors`）为整个组织配置 MCP 服务器，Claude Code 在启动时通过 OAuth 认证向 Anthropic API 请求这些服务器列表，并将其作为 `claudeai-proxy` 类型的配置合并到本地 MCP 配置中。

该文件的核心职责：
- 在会话生命周期内**一次性、有缓存地**拉取 Claude.ai 侧的 MCP 服务器清单；
- 对服务器名称做规范化与去重处理，防止与本地手动配置的服务器名称冲突；
- 提供缓存清理接口，在用户重新登录后刷新服务器列表；
- 记录哪些 claude.ai 配置的连接器曾经成功连接过，用于智能过滤启动时的“需要授权”通知。

---

## 功能点目的

### 1. 让组织级 MCP 配置自动下沉到 CLI
企业 IT 管理员在 Claude.ai 后台添加的 MCP 服务器（如内部知识库、Jira、GitHub Enterprise 等），无需每个员工手动编辑本地 `mcp.json`，登录后即可自动可用。

### 2. 与本地/项目级配置和平共存
Claude.ai 配置以 `scope: 'claudeai'` 标记，优先级低于用户手动配置的本地/项目级服务器。若 URL 重复，系统会主动去重，避免同一服务器出现两次。

### 3. 区分“从未连接过”与“曾经连接过现在失败”
通过 `markClaudeAiMcpConnected` / `hasClaudeAiMcpEverConnected` 记录连接历史，启动通知可以只提示“昨天还能用今天不行了”的状态变化，而忽略“组织配置了但员工一直没授权”的噪音。

---

## 具体技术实现

### 3.1 API 请求与响应类型

```ts
type ClaudeAIMcpServer = {
  type: 'mcp_server'
  id: string
  display_name: string
  url: string
  created_at: string
}

type ClaudeAIMcpServersResponse = {
  data: ClaudeAIMcpServer[]
  has_more: boolean
  next_page: string | null
}
```

请求细节：
- **Endpoint**：`GET {BASE_API_URL}/v1/mcp_servers?limit=1000`
- **Headers**：
  - `Authorization: Bearer {accessToken}`
  - `anthropic-beta: mcp-servers-2025-12-04`
  - `anthropic-version: 2023-06-01`
- **Timeout**：`FETCH_TIMEOUT_MS = 5000`

### 3.2  eligibility 检查（四层过滤）

`fetchClaudeAIMcpConfigsIfEligible` 在发起请求前执行以下检查，任一失败返回空对象 `{}`：

1. **Env 开关**：`ENABLE_CLAUDEAI_MCP_SERVERS` 若显式设为 `0/false/no/off`，直接禁用；
2. **OAuth Token**：`getClaudeAIOAuthTokens()?.accessToken` 不存在则返回；
3. **Scope 检查**：token 的 `scopes` 必须包含 `user:mcp_servers`。
   - 这里**直接检查 scope** 而非调用 `isClaudeAISubscriber()`，原因是非交互模式（print mode）下若设置了 `ANTHROPIC_API_KEY`，`preferThirdPartyAuthentication()` 会导致 `isAnthropicAuthEnabled()` 返回 false，从而误判为无权限。直接检查 scope 允许同时持有 API Key 和 OAuth Token 的用户在 print mode 下也能使用 claude.ai MCP。

### 3.3 名称规范化与碰撞处理

```ts
const baseName = `claude.ai ${server.display_name}`
```

- 先调用 `normalizeNameForMCP(baseName)` 得到规范名；
- 若规范名已被占用，则递增后缀 `(2)`、`(3)`…直到唯一；
- 使用 `usedNormalizedNames` Set 追踪已用名，处理极端边界（如 `"Example Server 2"` 与 `"Example Server! (2)"` 规范化后可能相同）。

`normalizeNameForMCP` 逻辑（位于 `src/services/mcp/normalization.ts`）：
- 将所有非 `[a-zA-Z0-9_-]` 字符替换为 `_`；
- 对于以 `claude.ai ` 开头的名称，还会压缩连续下划线并去除首尾下划线，防止干扰 MCP tool 名称中的 `__` 分隔符。

### 3.4 返回的配置结构

每个成功解析的服务器生成如下 `ScopedMcpServerConfig`：

```ts
configs[finalName] = {
  type: 'claudeai-proxy',
  url: server.url,
  id: server.id,
  scope: 'claudeai',
}
```

- `type: 'claudeai-proxy'` 表示该服务器通过 Anthropic 的 MCP Proxy 连接（实际连接逻辑在 `client.ts` 中根据 `type` 做分支处理）；
- `scope: 'claudeai'` 用于配置合并时的优先级排序。

### 3.5 缓存与清理

```ts
export const fetchClaudeAIMcpConfigsIfEligible = memoize(async () => { ... })
```

- 使用 `lodash-es/memoize` 做**会话级缓存**（函数引用级 memoize）；
- `clearClaudeAIMcpConfigsCache()` 调用 `fetchClaudeAIMcpConfigsIfEligible.cache.clear?.()` 并同步调用 `clearMcpAuthCache()`，确保新登录后下一次 fetch 使用全新 token；
- 在 `useManageMCPConnections.ts` 中，当检测到 auth version 变化时自动调用清理。

### 3.6 连接历史记录

```ts
export function markClaudeAiMcpConnected(name: string): void
export function hasClaudeAiMcpEverConnected(name: string): boolean
```

- 数据持久化在 `globalConfig.claudeAiMcpEverConnected` 字符串数组中；
- `saveGlobalConfig` 以函数式更新方式追加，保证幂等性；
- 在 `client.ts` 中，当 `config.type === 'claudeai-proxy'` 且连接成功时调用 `markClaudeAiMcpConnected(name)`。

---

## 关键代码路径与文件引用

### 本文件
- `src/services/mcp/claudeai.ts`（6126 bytes）

### 直接调用方
- `src/services/mcp/config.ts`
  - `getAllMcpConfigs()` 在加载全部 MCP 配置时调用 `fetchClaudeAIMcpConfigsIfEligible()`；
  - 若存在 enterprise 配置（`doesEnterpriseMcpConfigExist()`），则**完全跳过** claude.ai 服务器加载（enterprise 拥有独占控制权）。
- `src/services/mcp/useManageMCPConnections.ts`
  - 在 auth version 变化 effect 中调用 `clearClaudeAIMcpConfigsCache()` 并重新发起 `fetchClaudeAIMcpConfigsIfEligible()`。
- `src/services/mcp/client.ts`
  - 在 `connectToServer` 和 `reconnectMcpServerImpl` 成功后，若 `config.type === 'claudeai-proxy'`，调用 `markClaudeAiMcpConnected(name)`。

### 依赖文件
- `src/constants/oauth.ts`：`getOauthConfig()`（提供 `BASE_API_URL`）
- `src/services/analytics/index.ts`：`logEvent`（事件 `tengu_claudeai_mcp_eligibility`）
- `src/utils/auth.ts`：`getClaudeAIOAuthTokens`
- `src/utils/config.ts`：`getGlobalConfig`、`saveGlobalConfig`
- `src/utils/debug.ts`：`logForDebugging`
- `src/utils/envUtils.ts`：`isEnvDefinedFalsy`
- `src/services/mcp/client.ts`：`clearMcpAuthCache`
- `src/services/mcp/normalization.ts`：`normalizeNameForMCP`
- `src/services/mcp/types.ts`：`ScopedMcpServerConfig`

---

## 依赖与外部交互

### 外部库
- `axios`：发起 HTTP GET 请求；
- `lodash-es/memoize.js`：实现函数级缓存。

### 外部服务
- **Anthropic API** (`api.anthropic.com/v1/mcp_servers`)：返回组织配置的 MCP 服务器列表；
- **OAuth Token Provider**：依赖 `getClaudeAIOAuthTokens` 提供有效的 Bearer Token；
- **Analytics Sink**：通过 `logEvent('tengu_claudeai_mcp_eligibility', { state: ... })` 上报 eligibility 状态（`disabled_env_var`、`no_oauth_token`、`missing_scope`、`eligible`、`fetch_failed` 等）。

### 数据流

1. 启动时 `config.ts` → `getAllMcpConfigs()`；
2. 若非 enterprise 模式，并发启动 `fetchClaudeAIMcpConfigsIfEligible()`；
3. 检查 env / token / scope 三层 eligibility；
4. 通过 axios 向 Anthropic API 请求服务器列表；
5. 对每个服务器做名称规范化与碰撞处理；
6. 返回 `Record<string, ScopedMcpServerConfig>`；
7. `config.ts` 通过 `filterMcpServersByPolicy` 做策略过滤，再通过 `dedupClaudeAiMcpServers` 与本地配置去重；
8. 最终合并到 `getClaudeCodeMcpConfigs` 返回的服务器列表中（`claudeai` 优先级最低）。

---

## 风险、边界与改进建议

### 风险

1. **Enterprise 模式完全排除 Claude.ai MCP**
   - 当 `doesEnterpriseMcpConfigExist()` 为 true 时，`getAllMcpConfigs` 直接返回 `getClaudeCodeMcpConfigs()`，不拉取任何 claude.ai 服务器。这是设计上的有意隔离，但需确保企业客户明确知晓：他们在 Claude.ai 网页端配置的连接器**不会**出现在 CLI 中。

2. **API 超时与失败静默吞掉**
   - 当前 `catch` 块为空（仅 `logForDebugging` 和返回 `{}`），若 Anthropic API 500 或网络超时，用户不会收到任何提示，只会发现“组织配置的 MCP 服务器没出现”。在调试模式外难以排查。

3. **Memoize 缓存的隐式生命周期**
   - `lodash memoize` 绑定在函数对象上，若模块被热重载或测试环境多次 require，缓存行为可能不符合预期。虽然生产环境是单例模块，但测试代码需显式 `clearClaudeAIMcpConfigsCache()`。

4. **名称碰撞后缀可能产生歧义**
   - 当多个服务器规范化后同名时，后缀 `(2)`、`(3)` 对用户不够直观。例如两个 display_name 分别为 `"My Server"` 和 `"My_Server"` 的服务器，最终可能变成 `"claude.ai My Server"` 与 `"claude.ai My Server (2)"`，用户难以区分。

5. **Scope 检查绕过 `isClaudeAISubscriber` 的副作用**
   - 代码注释解释了直接检查 scope 的原因（兼容 print mode + API Key），但这意味着未来若 `isClaudeAISubscriber` 增加了其他 eligibility 逻辑（如订阅状态校验、封号检查），`claudeai.ts` 将不会同步受益。

### 边界

- **分页**：API 返回 `has_more` / `next_page`，但当前实现只请求第一页（`limit=1000`）。在组织配置超过 1000 个 MCP 服务器时会发生截断——这在现实中几乎不可能，但属于未处理的边界。
- **Beta Header 硬编码**：`MCP_SERVERS_BETA_HEADER = 'mcp-servers-2025-12-04'` 是写死的日期版本，未来 API GA 后需移除或更新。
- **连接历史无时间戳**：`claudeAiMcpEverConnected` 仅记录名称字符串，不记录首次/最近连接时间，无法做“30 天内未连接则重新提示”之类的策略。

### 改进建议

1. **为 fetch 失败提供用户可见的降级提示**
   - 在 `catch` 块中除了返回 `{}`，可考虑在 `getAllMcpConfigs` 层返回一个 `warnings` 数组，或在启动时打印一行 stderr 提示：`"Unable to fetch claude.ai MCP servers; using local config only."`

2. **支持 API 分页**
   - 若 `has_more` 为 true，应递归或循环拉取 `next_page`，直到全部加载完毕。虽然 1000 条上限很高，但防御性编程应覆盖该边界。

3. **为连接历史增加时间戳与过期机制**
   - 将 `claudeAiMcpEverConnected` 从 `string[]` 升级为 `Record<string, number>`（name → 最近连接时间戳），并支持 30 天或 90 天的过期清理，使“需要授权”通知更精准。

4. **统一 eligibility 接口**
   - 考虑在 `auth.ts` 或 `oauth` 模块中暴露一个 `hasMcpServersScope()` 公共函数，替代 `claudeai.ts` 中直接读取 `tokens.scopes?.includes('user:mcp_servers')`，避免其他未来调用点重复实现相同的绕过逻辑。

5. **增强 telemetry**
   - 当前仅上报 `tengu_claudeai_mcp_eligibility` 的 eligibility 状态，建议增加：
     - `tengu_claudeai_mcp_fetch_duration_ms`：记录 API 请求耗时；
     - `tengu_claudeai_mcp_server_count`：记录返回的服务器数量；
     - `tengu_claudeai_mcp_dedup_count`：记录被去重掉的服务器数量。
