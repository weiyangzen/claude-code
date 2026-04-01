# 研究文档：src/utils/uuid.ts

## 场景与职责

本模块提供两类与标识符相关的实用功能：

1. **UUID 格式校验**：验证一个未知值是否符合标准 UUID（8-4-4-4-12 十六进制）格式，用于命令行参数校验、会话 ID 校验、resume 操作等。
2. **Agent ID 生成**：为 Claude Code 的子代理（subagent）生成短格式、可读的标识符，格式为 `a{label-}{16 hex chars}`，与任务 ID 保持一致性。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `validateUuid(maybeUuid)` | 校验输入是否为标准 UUID 格式字符串，返回 `UUID` 类型或 `null`。 |
| `createAgentId(label?)` | 生成一个新的 Agent ID，可选带标签前缀。 |

## 具体技术实现

### 1. UUID 校验

```ts
const uuidRegex =
  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
```

- 正则严格匹配 8-4-4-4-12 的十六进制分组结构，不区分大小写。
- 先检查输入类型：`typeof maybeUuid !== 'string'` 直接返回 `null`。
- 校验通过后将字符串断言为 `UUID` 类型（来自 `node:crypto` 的类型别名）返回。

### 2. Agent ID 生成

```ts
export function createAgentId(label?: string): AgentId {
  const suffix = randomBytes(8).toString('hex') // 16 个十六进制字符
  return (label ? `a${label}-${suffix}` : `a${suffix}`) as AgentId
}
```

- 使用 `node:crypto` 的 `randomBytes(8)` 生成 64 位随机数，转为 16 位十六进制字符串。
- 前缀固定为 `a`（agent），若提供 `label` 则格式为 `a{label}-{suffix}`（如 `acompact-a3f2c1b4d5e6f7a8`）。
- `AgentId` 类型来自 `src/types/ids.js`，是 branded type，防止与普通字符串混淆。

## 关键代码路径与文件引用

- **主实现**：`src/utils/uuid.ts`（27 行）
- **类型定义**：`src/types/ids.ts`（`AgentId`、`SessionId` 等 branded types）
- **调用方（CLI 入口）**：`src/main.tsx`（`--session-id <uuid>` 参数校验）
- **调用方（resume 命令）**：`src/commands/resume/resume.tsx`（会话 ID 校验）
- **调用方（CLI print 模式）**：`src/cli/print.ts`
- **调用方（Agent 工具）**：`src/tools/AgentTool/AgentTool.tsx`、`src/tools/AgentTool/runAgent.ts`
- **调用方（Skill 工具）**：`src/tools/SkillTool/SkillTool.ts`
- **调用方（会话存储）**：`src/utils/sessionStorage.ts`
- **调用方（会话 URL）**：`src/utils/sessionUrl.ts`
- **调用方（slash 命令处理）**：`src/utils/processUserInput/processSlashCommand.tsx`
- **调用方（forked agent）**：`src/utils/forkedAgent.ts`

## 依赖与外部交互

- **`crypto`**（Node.js 内置）：`randomBytes`、`UUID` 类型。
- **`src/types/ids.js`**：`AgentId` 类型定义。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **UUID 校验不严格**：`validateUuid` 仅校验格式（8-4-4-4-12 十六进制），不校验 UUID 版本位（如版本 4 的 `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` 中第 13 位应为 `4`，第 17 位应在 `[89ab]` 范围内）。这意味着一个格式正确但版本位非法的字符串也会通过校验，可能导致下游系统（如服务端会话 ingress）拒绝接受。
2. **Agent ID 碰撞概率**：`randomBytes(8)` 提供 64 位熵，碰撞概率在常规会话量（单用户每天数十个 agent）下可忽略，但在大规模自动化测试或极端长会话中，理论上存在碰撞可能。当前无去重或冲突检测机制。
3. **`randomBytes` 的同步阻塞**：`createAgentId` 是同步函数，直接调用 `crypto.randomBytes(8)`。在 Linux 上，若 `/dev/urandom` 熵池耗尽（极端罕见），该调用可能短暂阻塞事件循环。对于高频批量生成 Agent ID 的场景（如自动化测试并发创建大量 subagent），这可能成为微瓶颈。

### 边界

- **仅支持标准 UUID**：模块不处理 UUID 的变体（如带 `urn:uuid:` 前缀的 URN、或带大括号的 Microsoft GUID 格式）。调用方需自行剥离前缀。
- **Agent ID 无时间戳/序列号**：与 UUID v1/v7 不同，Agent ID 是纯随机字符串，无法从 ID 本身推断创建顺序或时间。
- **正则大小写不敏感**：`validateUuid` 接受大写 UUID（如 `550E8400-E29B-41D4-A716-446655440000`），但项目内部通常统一使用小写。

### 改进建议

1. **增强 UUID 版本校验**：若项目主要使用 UUID v4，可将正则升级为：
   ```ts
   /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i
   ```
   这能过滤掉大量非法的“伪 UUID”。
2. **引入异步熵源**：对于高频生成场景，可提供 `createAgentIdAsync()` 变体，使用 `crypto.randomFill` 或 Web Crypto API 的异步接口，避免阻塞事件循环。
3. **增加前缀校验/规范化**：在 `validateUuid` 中增加对常见包装格式（如 `{uuid}`、`urn:uuid:`）的自动剥离，提升 CLI 参数的容错性。
4. **Agent ID 碰撞检测（可选）**：在创建 Agent ID 的调用方（如 `AgentTool.tsx`）维护一个会话级已用 ID 集合，若发生碰撞则重新生成。虽然概率极低，但在关键路径中可增加一层保险。
