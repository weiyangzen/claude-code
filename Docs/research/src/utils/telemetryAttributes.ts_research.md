# telemetryAttributes.ts 研究文档

## 场景与职责

`telemetryAttributes.ts` 是 Claude Code 遥测系统的**属性收集与配置模块**。它负责收集和组装用于 OpenTelemetry 事件、指标和追踪的上下文属性，包括用户身份、会话信息、OAuth 账户数据等。

### 核心职责

1. **遥测属性组装**：收集并返回标准化的 OpenTelemetry 属性对象
2. **可配置基数控制**：通过环境变量控制哪些高基数属性（如 session ID）包含在指标中
3. **用户身份解析**：整合用户 ID、OAuth 账户信息、组织 ID 等
4. **终端类型检测**：包含终端环境信息用于遥测分析

### 在遥测系统中的位置

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenTelemetry 遥测系统                     │
├─────────────────────────────────────────────────────────────┤
│  events.ts ──────┐                                          │
│  sessionTracing.ts ─────► telemetryAttributes.ts ◄────┐     │
│  perfettoTracing.ts ────┘                            │     │
│                                                      │     │
│  ┌───────────────────────────────────────────────────┘     │
│  ▼                                                         │
│  getTelemetryAttributes()                                   │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  user.id, session.id, organization.id, user.email   │   │
│  │  user.account_uuid, terminal.type, app.version      │   │
│  └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼                                                     │
│  OpenTelemetry Exporter (OTLP/Console/etc.)                │
└─────────────────────────────────────────────────────────────┘
```

## 功能点目的

### 1. 可配置基数控制

指标系统需要控制基数（cardinality）以避免存储和查询性能问题。本模块通过环境变量提供细粒度控制：

```typescript
const METRICS_CARDINALITY_DEFAULTS = {
  OTEL_METRICS_INCLUDE_SESSION_ID: true,    // 默认包含 session ID
  OTEL_METRICS_INCLUDE_VERSION: false,      // 默认不包含版本号
  OTEL_METRICS_INCLUDE_ACCOUNT_UUID: true,  // 默认包含账户 UUID
}
```

### 2. 属性收集

`getTelemetryAttributes()` 函数收集以下属性：

| 属性名 | 来源 | 条件 |
|--------|------|------|
| `user.id` | `getOrCreateUserID()` | 始终包含 |
| `session.id` | `getSessionId()` | `OTEL_METRICS_INCLUDE_SESSION_ID` |
| `app.version` | `MACRO.VERSION` | `OTEL_METRICS_INCLUDE_VERSION` |
| `organization.id` | OAuth account | OAuth 认证时 |
| `user.email` | OAuth account | OAuth 认证时 |
| `user.account_uuid` | OAuth account | OAuth 认证且 `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` |
| `user.account_id` | 环境变量或计算 | OAuth 认证时 |
| `terminal.type` | `envDynamic.terminal` | 终端类型可用时 |

### 3. Tagged ID 生成

支持将 UUID 转换为 API 兼容的 tagged ID 格式：

```typescript
attributes['user.account_id'] =
  process.env.CLAUDE_CODE_ACCOUNT_TAGGED_ID ||
  toTaggedId('user', accountUuid)
```

## 具体技术实现

### 基数控制逻辑

```typescript
function shouldIncludeAttribute(
  envVar: keyof typeof METRICS_CARDINALITY_DEFAULTS,
): boolean {
  const defaultValue = METRICS_CARDINALITY_DEFAULTS[envVar]
  const envValue = process.env[envVar]

  if (envValue === undefined) {
    return defaultValue
  }
  return isEnvTruthy(envValue)
}
```

### 主函数实现

```typescript
export function getTelemetryAttributes(): Attributes {
  const userId = getOrCreateUserID()
  const sessionId = getSessionId()

  const attributes: Attributes = {
    'user.id': userId,
  }

  // Session ID（可配置）
  if (shouldIncludeAttribute('OTEL_METRICS_INCLUDE_SESSION_ID')) {
    attributes['session.id'] = sessionId
  }

  // 应用版本（可配置）
  if (shouldIncludeAttribute('OTEL_METRICS_INCLUDE_VERSION')) {
    attributes['app.version'] = MACRO.VERSION
  }

  // OAuth 账户信息（仅在使用 OAuth 时）
  const oauthAccount = getOauthAccountInfo()
  if (oauthAccount) {
    const orgId = oauthAccount.organizationUuid
    const email = oauthAccount.emailAddress
    const accountUuid = oauthAccount.accountUuid

    if (orgId) attributes['organization.id'] = orgId
    if (email) attributes['user.email'] = email

    if (accountUuid && shouldIncludeAttribute('OTEL_METRICS_INCLUDE_ACCOUNT_UUID')) {
      attributes['user.account_uuid'] = accountUuid
      attributes['user.account_id'] =
        process.env.CLAUDE_CODE_ACCOUNT_TAGGED_ID ||
        toTaggedId('user', accountUuid)
    }
  }

  // 终端类型
  if (envDynamic.terminal) {
    attributes['terminal.type'] = envDynamic.terminal
  }

  return attributes
}
```

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `bootstrap/state.ts` | `getSessionId()` 获取会话 ID |
| `auth.ts` | `getOauthAccountInfo()` 获取 OAuth 账户信息 |
| `config.ts` | `getOrCreateUserID()` 获取或创建用户 ID |
| `envDynamic.ts` | `envDynamic.terminal` 终端类型 |
| `envUtils.ts` | `isEnvTruthy()` 环境变量真值判断 |
| `taggedId.ts` | `toTaggedId()` Tagged ID 生成 |

### 外部调用方

| 文件 | 调用目的 |
|------|----------|
| `events.ts` | `logOTelEvent()` 记录遥测事件时获取基础属性 |
| `sessionTracing.ts` | 创建 span 时获取基础属性 |
| `init.ts` | 初始化遥测时获取属性 |

### 调用链示例

```typescript
// events.ts
export async function logOTelEvent(eventName: string, metadata = {}) {
  const attributes: Attributes = {
    ...getTelemetryAttributes(),  // <-- 调用
    'event.name': eventName,
    'event.timestamp': new Date().toISOString(),
    'event.sequence': eventSequence++,
  }
  // ...
  eventLogger.emit({ body: `claude_code.${eventName}`, attributes })
}
```

## 依赖与外部交互

### 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OTEL_METRICS_INCLUDE_SESSION_ID` | `true` | 是否在指标中包含 session ID |
| `OTEL_METRICS_INCLUDE_VERSION` | `false` | 是否在指标中包含应用版本 |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | `true` | 是否在指标中包含账户 UUID |
| `CLAUDE_CODE_ACCOUNT_TAGGED_ID` | - | 预计算的账户 tagged ID |

### 与 config.ts 的交互

`getOrCreateUserID()` 从配置文件中读取或生成匿名用户 ID：

```typescript
// config.ts
export function getOrCreateUserID(): string {
  const config = getGlobalConfig()
  if (config.account?.userId) {
    return config.account.userId
  }
  // 生成新的用户 ID 并保存
  const userId = generateUserId()
  saveGlobalConfig({ ...config, account: { ...config.account, userId } })
  return userId
}
```

### 与 auth.ts 的交互

`getOauthAccountInfo()` 返回当前 OAuth 会话的账户信息：

```typescript
// auth.ts
export function getOauthAccountInfo(): OAuthAccountInfo | null {
  const tokens = getOAuthTokens()
  if (!tokens) return null
  return {
    accountUuid: tokens.accountUuid,
    organizationUuid: tokens.organizationUuid,
    emailAddress: tokens.emailAddress,
  }
}
```

### 与 taggedId.ts 的交互

Tagged ID 格式与 API 的 `tagged_id.py` 兼容：

```typescript
// taggedId.ts
export function toTaggedId(tag: string, uuid: string): string {
  const n = uuidToBigInt(uuid)
  return `${tag}_${VERSION}${base58Encode(n)}`
}
// 输出示例: "user_01PaGUP2rbg1XDh7Z9W1CEpd"
```

## 风险、边界与改进建议

### 已知风险

1. **隐私合规**：收集用户 email、account UUID 等 PII 数据需要符合隐私政策
2. **基数爆炸**：即使可配置，默认包含 session ID 仍可能导致某些场景下的高基数问题
3. **OAuth 依赖**：`getOauthAccountInfo()` 可能触发令牌刷新，增加延迟

### 边界情况

1. **非交互式会话**：`getSessionId()` 在非交互式会话中的行为
2. **OAuth 令牌过期**：`getOauthAccountInfo()` 可能返回 null，导致属性缺失
3. **终端类型检测失败**：`envDynamic.terminal` 可能为 undefined
4. **用户 ID 生成**：首次调用 `getOrCreateUserID()` 会同步写入配置文件

### 改进建议

1. **缓存机制**：
   - `getTelemetryAttributes()` 每次调用都重新收集所有属性，可以考虑缓存
   - 但需要注意 OAuth 令牌刷新等可能变化的值

2. **延迟加载**：
   - OAuth 账户信息可以延迟加载，避免在冷路径上阻塞
   - 使用 Promise 或异步 getter 模式

3. **属性验证**：
   - 添加 Zod schema 验证返回的属性对象
   - 确保所有属性值符合 OpenTelemetry 规范

4. **更细粒度的控制**：
   - 添加 `OTEL_METRICS_INCLUDE_USER_EMAIL` 等更细粒度的开关
   - 允许用户完全禁用某些属性类别

5. **文档完善**：
   - 添加每个属性的用途和隐私影响说明
   - 提供配置示例和最佳实践

6. **测试覆盖**：
   - 添加单元测试覆盖各种环境变量组合
   - 测试 OAuth 账户信息缺失时的降级行为

7. **性能优化**：
   - 考虑使用 `AsyncLocalStorage` 缓存用户 ID 等不变值
   - 避免在热路径上重复调用同步文件操作
