# 研究文档：src/services/mcp/channelNotification.ts

## 场景与职责

`channelNotification.ts` 是 Claude Code 的 **MCP Channel（通道）通知入口层**，负责让外部 MCP 服务器（如 Discord、Slack、Telegram、iMessage 等）把用户消息“推送”进当前对话流。一个 MCP 服务器要成为 Channel，需要同时满足两个条件：

1. ** outbound**：暴露标准 MCP tool（例如 `send_message`），供 Claude Code 向该渠道发消息；
2. **inbound**：通过 MCP notification 向 Claude Code 发送 `notifications/claude/channel`，把用户在该渠道的新消息推入对话。

该文件的核心职责是：
- 定义 Channel 消息与权限通知的 JSON-RPC 协议 Schema；
- 对入站消息做 XML 包装（`<channel source="...">...</channel>`），使其能被模型识别来源；
- 提供多层安全闸门 `gateChannelServer`，决定是否为某个 MCP 客户端注册 inbound 通知处理器；
- 在会话级 `--channels` 列表中查找匹配的服务器条目。

该功能受 GrowthBook feature flag `tengu_harbor` 与编译期 `feature('KAIROS') || feature('KAIROS_CHANNELS')` 双重控制，且**仅对 claude.ai OAuth 用户开放**（API Key 用户被显式拦截）。

---

## 功能点目的

### 1. 把外部 IM/聊天渠道变成对话输入源
用户在 Telegram/Slack 里回复的消息，通过 MCP notification 进入 Claude Code 的命令队列，模型在下一轮能看到 `<channel>` 标签包裹的内容，并决定用哪个 tool 回复（可能是该 Channel 的 send_message，也可能是本地 `SendUserMessage`）。

### 2. 权限提示的跨渠道双向协议
除了普通聊天消息，Channel 还可以作为**权限审批表面**（permission surface）。当 Claude Code 遇到需要用户确认的工具调用时，可以把权限提示转发到 Channel，用户在手机上回复 "yes abcde" 即可完成审批。该文件定义了：
-  outbound 请求格式 `notifications/claude/channel/permission_request`
-  inbound 回复格式 `notifications/claude/channel/permission`

### 3. 组织级与用户级的双重安全控制
- **Org 级**：Team/Enterprise 客户必须在 managed settings 中显式设置 `channelsEnabled: true`，否则全部 Channel 功能被拦截；
- **用户/会话级**：服务器必须出现在当前会话的 `--channels` 列表中，防止已被信任的 MCP 服务器“突然”获得消息注入能力；
- **Allowlist 级**：对于 `plugin:` 类型的 Channel，还需要匹配 GrowthBook allowlist（`tengu_harbor_ledger`）或 org 自定义的 `allowedChannelPlugins`。

---

## 具体技术实现

### 3.1 协议 Schema（Zod 懒加载）

```ts
export const ChannelMessageNotificationSchema = lazySchema(() =>
  z.object({
    method: z.literal('notifications/claude/channel'),
    params: z.object({
      content: z.string(),
      meta: z.record(z.string(), z.string()).optional(),
    }),
  }),
)
```

- `meta` 是透传字段，会被渲染成 `<channel>` 标签的 XML 属性（如 `chat_id`、`thread_ts`、`message_id`）。
- `ChannelPermissionNotificationSchema` 用于结构化权限回复：`{ request_id: string, behavior: 'allow' | 'deny' }`。
- `CHANNEL_PERMISSION_REQUEST_METHOD = 'notifications/claude/channel/permission_request'` 是 CC **发出**的请求方法类型定义（非 Zod Schema，仅作文档化类型）。

### 3.2 XML 包装与属性安全

```ts
export function wrapChannelMessage(
  serverName: string,
  content: string,
  meta?: Record<string, string>,
): string
```

实现要点：
- 使用 `escapeXmlAttr` 对属性值做转义（`& < > " '`）。
- `SAFE_META_KEY = /^[a-zA-Z_][a-zA-Z0-9_]*$/` 过滤 meta key，防止恶意 key（如 `x=" injected="y`）破坏 XML 属性结构。
- 输出示例：
  ```xml
  <channel source="plugin:telegram:tg" chat_id="12345">
  Hello from Telegram
  </channel>
  ```

### 3.3 会话级 Channel 条目匹配

```ts
export function findChannelEntry(
  serverName: string,
  channels: readonly ChannelEntry[],
): ChannelEntry | undefined
```

- `server-kind`：裸名精确匹配（如 `slack`）。
- `plugin-kind`：匹配 `plugin:X:Y` 中的 `X` 部分（即插件名称）。
- 返回的 `ChannelEntry` 带有 `kind: 'plugin' | 'server'` 以及可选的 `dev` 标志。

### 3.4 多层闸门 `gateChannelServer`

闸门按**严格顺序**执行，任一环节失败即返回 `skip`，成功则返回 `register`：

| 顺序 | 检查项 | 失败原因示例 |
|------|--------|--------------|
| 1 | Capability | 服务器未声明 `capabilities.experimental['claude/channel']` |
| 2 | Runtime gate | `isChannelsEnabled()`（GrowthBook `tengu_harbor`）为 false |
| 3 | Auth | 无 claude.ai OAuth accessToken（API Key 用户被拦截） |
| 4 | Org Policy | Team/Enterprise 且 `policy.channelsEnabled !== true` |
| 5 | Session opt-in | 服务器不在 `--channels` 列表中 |
| 6 | Marketplace 校验 | `plugin:name@marketplace` 与实际安装的插件来源不符 |
| 7 | Allowlist | 插件不在 `tengu_harbor_ledger` 或 org 自定义 allowlist 中；`dev` 标志可绕过 |

对于 `server-kind` 条目，allowlist Schema 只包含 `{marketplace, plugin}`，因此 server-kind 非 `dev` 时**必然失败**，这防止了 `--channels server:plugin:foo:bar` 这种畸形输入绕过插件校验。

### 3.5 有效 Allowlist 的合并逻辑

```ts
export function getEffectiveChannelAllowlist(sub, orgList)
```

- 若用户订阅类型为 `team` 或 `enterprise`，且 org 配置了 `allowedChannelPlugins`，则**完全替换** GrowthBook ledger；
- 否则回退到 GrowthBook `tengu_harbor_ledger`；
- 非托管用户始终使用 ledger。

---

## 关键代码路径与文件引用

### 本文件
- `src/services/mcp/channelNotification.ts`（12540 bytes）

### 直接调用方
- `src/services/mcp/useManageMCPConnections.ts`
  - 在 MCP 客户端连接成功后调用 `gateChannelServer` 决定是否注册通知处理器；
  - 注册 `ChannelMessageNotificationSchema` 与 `ChannelPermissionNotificationSchema` 的 handler；
  - handler 内部调用 `wrapChannelMessage` 并把结果 `enqueue` 进命令队列。
- `src/cli/print.ts`
  - `handleChannelEnable`：处理 `channel_enable` control request，动态把插件加入 `--channels` 并注册 handler；
  - `reregisterChannelHandlerAfterReconnect`：在 `mcp_reconnect` / `mcp_toggle` 后重新绑定 handler（因为旧 client 对象已失效）。
- `src/hooks/toolPermission/handlers/interactiveHandler.ts`
  - 导入 `CHANNEL_PERMISSION_REQUEST_METHOD`、`findChannelEntry` 等，用于 outbound 权限请求。

### 依赖文件
- `src/services/mcp/channelAllowlist.ts`：`getChannelAllowlist`、`isChannelsEnabled`、`ChannelAllowlistEntry`
- `src/bootstrap/state.ts`：`ChannelEntry`、`getAllowedChannels`
- `src/constants/xml.ts`：`CHANNEL_TAG`（值为 `'channel'`）
- `src/utils/auth.ts`：`getClaudeAIOAuthTokens`、`getSubscriptionType`
- `src/utils/settings/settings.ts`：`getSettingsForSource`
- `src/utils/plugins/pluginIdentifier.ts`：`parsePluginIdentifier`
- `src/utils/xml.ts`：`escapeXmlAttr`
- `src/utils/lazySchema.ts`：`lazySchema`
- `src/utils/messageQueueManager.ts`：`enqueue`（由调用方使用）

---

## 依赖与外部交互

### 外部库
- `@modelcontextprotocol/sdk/types.js`：引入 `ServerCapabilities` 类型；
- `zod/v4`：用于运行时 Schema 校验（懒加载模式）。

### 外部服务
- **GrowthBook**：通过 `tengu_harbor`（总开关）、`tengu_harbor_ledger`（插件 allowlist）控制运行时行为；
- **Claude.ai OAuth**：闸门要求有效的 OAuth accessToken，且 token scopes 需包含 `user:mcp_servers`（该检查在 `claudeai.ts` 中，但 Channel 功能本身也依赖 OAuth 身份）。

### 数据流
1. 外部 Channel MCP Server → 发送 `notifications/claude/channel` JSON-RPC notification；
2. `useManageMCPConnections.ts` 中的 handler 接收 → 调用 `wrapChannelMessage`；
3. `enqueue({ mode: 'prompt', value: <channel>...</channel>, priority: 'next', ... })`；
4. `SleepTool` 检测到队列非空 → 唤醒主循环；
5. 模型读取 `<channel>` 标签内容 → 生成回复 → 可能调用该 Channel 的 tool 发回消息。

---

## 风险、边界与改进建议

### 风险

1. **OAuth-only 限制导致 API Key 用户无法使用**
   - 当前 `gateChannelServer` 显式检查 `getClaudeAIOAuthTokens()?.accessToken`，API Key 用户直接被拒。注释说明“console 还没有 channelsEnabled 管理界面”，未来需要解除该限制。

2. ** compromised Channel 服务器可伪造权限审批**
   - 虽然权限回复采用结构化事件（而非正则匹配文本），但一个已被允许且已连接的 Channel 服务器**可以在没有真实用户回复的情况下**直接 emit `notifications/claude/channel/permission` 并携带正确的 `request_id`。
   - 缓解：该风险在 `channelPermissions.ts` 的注释中被明确记录为“accepted risk”—— compromised Channel 已经拥有无限的消息注入轮次，伪造审批只是更快，而非更强。

3. **XML 属性注入**
   - `meta` key 若未经过滤，可能成为属性注入向量。当前 `SAFE_META_KEY` 正则足够严格，但未来若放宽正则（比如允许 `-` 或 `:`），需同步审计 XML 转义逻辑。

4. **Mid-session 降级（logout / policy 变更）**
   - `gateChannelServer` 在 `useManageMCPConnections.ts` 的 effect 重跑时会被再次调用。若结果从 `register` 变为 `skip`，代码会主动 `removeNotificationHandler`，防止旧 handler 继续注入消息。该机制已覆盖，但需确保所有调用路径（如 `print.ts` 的 reconnect）都执行相同的 teardown。

### 边界

- **并发权限提示数量**：`shortRequestId` 在 `channelPermissions.ts` 中生成 5 字母 ID，25^5 ≈ 9.8M 空间，单会话同时 pending 的权限提示达到 ~3K 时才有 50% 碰撞概率，实际不可能触及。
- **消息长度**：`truncateForPreview` 对权限请求的 tool input 做 200 字符截断，但普通 Channel 消息 `content` 本身**没有长度限制**，由下游 `enqueue` 和模型上下文窗口共同约束。
- **插件来源校验**：`pluginSource` 来自服务器配置对象，若配置层被篡改（如本地 `mcp.json` 手动构造），marketplace 校验可能失效。

### 改进建议

1. **统一 gate 与 handler 注册逻辑**
   - 当前 `useManageMCPConnections.ts` 和 `print.ts` 各自维护了一套几乎相同的 handler 注册代码。建议抽象为 `registerChannelHandlers(client, serverName, entry)` 公共函数，减少复制粘贴带来的漂移风险。

2. **为 API Key / Console 用户提供 Org Policy 入口**
   - 一旦 console 侧支持 `channelsEnabled` 管理开关，应移除 `gateChannelServer` 中的 OAuth 硬拦截，改为统一的 policy 层检查。

3. **增强权限审批的人机绑定**
   - 当前 `request_id` 仅基于 `toolUseID` 做确定性哈希。若未来需要更高安全级别，可考虑在 `permission_request` 中加入一次性 nonce，并要求回复事件回显该 nonce，增加 compromised server 的伪造难度。

4. **Telemetry 完善**
   - `gateChannelServer` 的 `skip` 原因已记录到 `tengu_mcp_channel_gate` 事件，但 `channel_enable` 和 `reregisterChannelHandlerAfterReconnect` 路径的 gate 结果只在 `print.ts` 中局部记录。建议统一 telemetry 接口。
