# 研究文档：src/services/mcp/channelPermissions.ts

## 场景与职责

`channelPermissions.ts` 实现 Claude Code 的 **跨渠道权限审批（Channel Permission Relay）** 机制。当用户在本地终端遇到工具调用权限对话框时，系统会同时把该权限提示转发到所有已激活的 Channel（如 Telegram、iMessage、Discord 等），并与本地 UI、CCR Bridge、Hooks、Bash Classifier **竞速（race）**——第一个返回的审批结果获胜。

该文件的核心职责包括：
- 提供独立的 GrowthBook 运行时开关 `tengu_harbor_permissions`，控制权限转发功能是否启用；
- 把较长的 `toolUseID` 压缩成适合手机输入的 5 字母短 ID；
- 对 tool input 做手机友好的 JSON 截断预览；
- 过滤出具备权限转发能力的 MCP 客户端；
- 维护一个基于闭包 `Map` 的 pending 权限请求池，供 inbound 结构化事件匹配与解析。

该模块的设计哲学是：**Channel 服务器负责解析用户回复文本，Claude Code 只接收结构化事件**。这意味着 CC 侧不再对 Channel 消息做正则匹配，从而避免“普通聊天内容意外触发审批”的安全隐患。

---

## 功能点目的

### 1. 让用户在手机上审批终端权限请求
用户在 Telegram 收到类似下面的消息：
```
Claude wants to use BashTool
Preview: {"command":"ls -la"}
Reply: yes tbxkq
```
用户在手机上回复后，Telegram MCP Server 解析该回复，并向 Claude Code 发送结构化通知 `notifications/claude/channel/permission`，CC 即可立即执行或拒绝该工具调用。

### 2. 与本地/Bridge/Classifier 竞速
`interactiveHandler.ts` 使用 `createResolveOnce` 保证多个审批来源（本地 Enter 键、Bridge 网页端、Channel 手机端、自动 Classifier）中**只有一个能生效**。Channel 权限回调通过 `claim()` 参与该竞速。

### 3. 结构化事件替代文本正则
早期实现可能直接在 CC 侧对 Channel 消息内容做正则匹配（如 `^\s*(y|yes|n|no)\s+([a-km-z]{5})\s*$`）。当前设计把该正则**下沉到 Channel Server**，CC 只认 `{request_id, behavior}` 结构化 payload。这样：
- 普通聊天文本永远不会误触发审批；
- 只有明确支持权限功能的 Channel Server 才能参与审批流程。

---

## 具体技术实现

### 3.1 GrowthBook 运行时开关

```ts
export function isChannelPermissionRelayEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_harbor_permissions', false)
}
```

- 与 `tengu_harbor`（Channel 总开关）分离，确保 Channel 消息功能可以独立上线，而权限转发功能可单独灰度或回滚。
- 在 `useManageMCPConnections.ts` mount 时检查一次；会话中 flag 变更需重启生效。

### 3.2 短 ID 生成 `shortRequestId`

输入为 `toolUseID`（形如 `toolu_01AbCdEfGhIjKlMnOpQrStUv`），输出为 **5 位小写字母**（不含 `l`，避免与 `1/I` 混淆）。

算法步骤：
1. **FNV-1a 32-bit 哈希**：
   ```ts
   let h = 0x811c9dc5
   for (let i = 0; i < input.length; i++) {
     h ^= input.charCodeAt(i)
     h = Math.imul(h, 0x01000193)
   }
   h = h >>> 0
   ```
2. **Base-25 编码**：取 `h % 25` 映射到字母表 `abcdefghijkmnopqrstuvwxyz`（25 个字符，去掉 `l`），循环 5 次得到 5 字母 ID。
3. **敏感词过滤**：若生成的 ID 包含 `ID_AVOID_SUBSTRINGS` 中的子串（如 `fuck`、`shit`、`nazi` 等 24 个词），则加盐重哈希（`toolUseID:${salt}`），最多重试 10 次。

数学边界：
- 空间大小：25^5 ≈ 9,765,625；
- 阻塞 ID 数量估算：约 13,877 个（~1/700 命中率）；
- 50% 生日碰撞阈值：约 3,700 个同时 pending 的请求，单会话几乎不可能达到。

### 3.3 手机预览截断 `truncateForPreview`

```ts
export function truncateForPreview(input: unknown): string
```

- 使用 `jsonStringify` 序列化 tool input；
- 超过 200 字符时截断并追加 `…`；
- 序列化失败时返回 `(unserializable)`。

设计意图：完整 input 仍在本地终端对话框展示，Channel 仅收到摘要，防止大文件写入请求淹没手机短信。

### 3.4 客户端过滤 `filterPermissionRelayClients`

```ts
export function filterPermissionRelayClients<T>(
  clients: readonly T[],
  isInAllowlist: (name: string) => boolean,
): (T & { type: 'connected' })[]
```

三个必要条件（AND）：
1. `client.type === 'connected'`
2. `isInAllowlist(c.name) === true`（即该服务器在当前会话 `--channels` 列表中）
3. 同时声明两个 experimental capability：
   - `claude/channel`
   - `claude/channel/permission`

第三个条件是 Channel Server 的**显式 opt-in**，防止仅做消息转发的普通 Channel 意外变成权限表面。

### 3.5 权限回调工厂 `createChannelPermissionCallbacks`

```ts
export function createChannelPermissionCallbacks(): ChannelPermissionCallbacks
```

返回对象：
- `onResponse(requestId, handler)`：将 handler 注册到闭包 `pending` Map 中，返回 unsubscribe 函数；
- `resolve(requestId, behavior, fromServer)`：从 Map 中查找并调用 handler，返回是否匹配成功。

实现细节：
- `requestId` 统一转小写存储，避免大小写不一致导致永远无法匹配；
- `resolve` 在调用 handler **之前**先 `pending.delete(key)`，防止重复消费或 resolver 抛出时留下脏数据；
- 生命周期：在 `useManageMCPConnections.ts` 中**每会话构造一次**，稳定引用存入 `AppState.channelPermissionCallbacks`。

### 3.6 回复格式规范（供 Channel Server 实现参考）

```ts
export const PERMISSION_REPLY_RE = /^\s*(y|yes|n|no)\s+([a-km-z]{5})\s*$/i
```

- 支持 `y/yes/n/no` + 5 位小写字母；
- 不区分大小写（兼容手机自动大写首字母）；
- 不允许纯 `yes/no`（防止日常对话误触）；
- 不允许前后缀闲聊（必须严格匹配）。

---

## 关键代码路径与文件引用

### 本文件
- `src/services/mcp/channelPermissions.ts`（8981 bytes）

### 直接调用方
- `src/hooks/toolPermission/handlers/interactiveHandler.ts`
  - 调用 `shortRequestId` 生成 Channel 权限请求 ID；
  - 调用 `filterPermissionRelayClients` 获取可转发的客户端列表；
  - 调用 `truncateForPreview` 构造 `input_preview`；
  - 通过 `channelCallbacks.onResponse` 订阅回复，与本地/Bridge/Classifier 竞速。
- `src/services/mcp/useManageMCPConnections.ts`
  - 在 hook mount 时调用 `createChannelPermissionCallbacks()` 构造回调对象；
  - 若 `isChannelPermissionRelayEnabled()` 返回 true，将回调存入 `AppState`；
  - 为声明了 `claude/channel/permission` capability 的客户端注册 inbound notification handler，handler 内部调用 `channelPermCallbacksRef.current?.resolve(...)`。

### 依赖文件
- `src/utils/slowOperations.js`：`jsonStringify`
- `src/services/analytics/growthbook.js`：`getFeatureValue_CACHED_MAY_BE_STALE`
- `src/services/mcp/channelNotification.ts`：
  - `CHANNEL_PERMISSION_METHOD`（`notifications/claude/channel/permission`）
  - `ChannelPermissionNotificationSchema`
  - `CHANNEL_PERMISSION_REQUEST_METHOD`
  - `ChannelPermissionRequestParams` 类型
- `src/state/AppStateStore.ts`：`AppState.channelPermissionCallbacks` 字段定义
- `src/utils/messageQueueManager.ts`：`enqueue`（由调用方在收到普通 Channel 消息时使用）

---

## 依赖与外部交互

### 外部库
- 无直接第三方运行时依赖（仅依赖项目内部模块）。

### 外部服务
- **GrowthBook**：`tengu_harbor_permissions` flag 控制权限转发功能是否可用；
- **MCP SDK**：Channel Server 需基于 `@modelcontextprotocol/sdk` 实现客户端，支持 `notification` 发送与接收。

### 数据流（权限审批完整链路）

1. **触发**：`interactiveHandler.ts` 中遇到 `behavior: 'ask'` 的权限请求；
2. **生成 ID**：`shortRequestId(ctx.toolUseID)` → 5 字母短 ID；
3. **过滤客户端**：`filterPermissionRelayClients(...)` 找出具备权限能力的活跃 Channel；
4. **发送请求**：对每个 Channel Client 调用 `client.notification({ method: CHANNEL_PERMISSION_REQUEST_METHOD, params: { request_id, tool_name, description, input_preview } })`；
5. **订阅回复**：`channelCallbacks.onResponse(channelRequestId, handler)` 把 handler 加入 pending Map；
6. **用户回复**：用户在手机上输入 "yes tbxkq" → Channel Server 解析并 emit `notifications/claude/channel/permission`；
7. **解析匹配**：`useManageMCPConnections.ts` 中的 handler 调用 `resolve(request_id, behavior, client.name)`；
8. **竞速决胜**：`interactiveHandler.ts` 中的 handler 通过 `claim()` 获胜后，取消其他 racer（包括本地对话框、Bridge、Classifier），并执行 `allow` 或 `deny` 逻辑。

---

## 风险、边界与改进建议

### 风险

1. ** compromised Channel Server 可伪造审批**
   - 由于 `request_id` 是确定性哈希（`shortRequestId(toolUseID)`），且 CC 侧没有要求回显额外 nonce 或签名，一个已被允许连接的恶意 Channel Server 可以在未向真实用户展示提示的情况下，直接构造并发送 `notifications/claude/channel/permission`。
   - 缓解：该风险在代码注释中被明确标记为 **accepted risk**。理由是 compromised Channel 已经拥有无限的消息注入能力（可长期社会工程学攻击或等待 `acceptEdits` 等机会），伪造审批只是加速而非扩大攻击面。

2. **FNV-1a 不是密码学安全哈希**
   - `shortRequestId` 使用 FNV-1a 做确定性压缩，攻击者若知道 `toolUseID` 的生成规律，可以预测 `request_id`。但由于 `toolUseID` 本身由 Anthropic API 生成（`toolu_` + base64-ish 随机串），外部难以预测。

3. **GrowthBook 缓存可能过时**
   - `isChannelPermissionRelayEnabled` 使用 `getFeatureValue_CACHED_MAY_BE_STALE`，意味着 mid-session flag 翻转不会立即生效（需重启）。这是设计上的有意选择，但可能导致用户困惑。

4. **pending Map 的内存泄漏**
   - 若用户始终不回复 Channel 权限请求，且本地/Bridge/Classifier 也未触发 `claim()`，则 pending Map 中的条目将一直保留到会话结束。虽然单条目内存占用极小，但极端情况下可能累积。

### 边界

- **ID 空间**：25^5 ≈ 980 万，碰撞概率极低；
- **预览长度**：200 字符是经验值，约等于窄屏手机的 3 行；
- **盐值重试上限**：10 次，(1/700)^10 的阻塞概率可忽略；
- **并发 Channel 数量**：无硬性上限，取决于 `filterPermissionRelayClients` 返回的数组长度。

### 改进建议

1. **引入一次性 nonce 机制**
   - 在 `CHANNEL_PERMISSION_REQUEST_METHOD` 的 params 中增加 `nonce` 字段，并要求 `ChannelPermissionNotificationSchema` 回显该 nonce。这样即使 compromised server 知道 `request_id`，也无法伪造有效回复。

2. **为 pending Map 增加 TTL**
   - 为每个 pending 条目设置过期时间（如 10 分钟），过期后自动清理并调用 handler 返回 `deny` 或静默丢弃，防止长期内存占用。

3. **统一 Channel 权限与 Bridge 权限的抽象**
   - 当前 `BridgePermissionCallbacks` 与 `ChannelPermissionCallbacks` 接口不同，导致 `interactiveHandler.ts` 中需要分别处理两套逻辑。可考虑抽象为统一的 `RemotePermissionSurface` 接口，降低未来增加新远程审批渠道（如邮件、推送通知）的复杂度。

4. **增强 telemetry**
   - 当前仅记录了 `tengu_mcp_channel_message` 和 `tengu_mcp_channel_gate`，建议增加：
     - `tengu_mcp_channel_permission_sent`：记录发送的权限请求数量与目标 Channel；
     - `tengu_mcp_channel_permission_resolved`：记录回复来源（Channel vs local vs Bridge）与耗时；
     - `tengu_mcp_channel_permission_timeout`：记录未收到回复的超时情况。
