# Session Ingress 服务研究文档

## 文件信息
- **路径**: `src/services/api/sessionIngress.ts`
- **大小**: 17,055 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Session Ingress 服务负责管理 Claude Code 的 **会话日志持久化和 Teleport 功能**，是 CCR（Claude Code Remote）和会话同步的核心基础设施。该服务处理以下核心场景：

1. **会话日志追加**: 使用乐观并发控制（Last-Uuid 头）将日志条目追加到远程会话
2. **会话日志获取**: 获取完整会话日志用于 hydration 和 Teleport
3. **Teleport 事件获取**: 从新的 Sessions API (v2) 获取事件，支持分页
4. **OAuth 会话获取**: 用于 Teleport 的 OAuth 认证会话日志获取
5. **并发控制**: 每会话顺序执行，防止竞态条件

---

## 功能点目的

### 1. 日志追加 (`appendSessionLog`)
核心日志持久化功能：
- 使用 JWT token 认证（Bearer 或 Cookie）
- 乐观并发控制：通过 `Last-Uuid` 头维护追加链
- **每会话顺序执行**：使用 `sequential` 包装器确保单会话内串行处理
- 自动重试：最多 10 次，指数退避（最大 8 秒）

#### 409 冲突处理
```typescript
if (response.status === 409) {
  // 检查是否我们的条目已经是最后一条（之前成功但客户端未收到确认）
  if (serverLastUuid === entry.uuid) {
    // 恢复状态，视为成功
  }
  // 否则采用服务器的 lastUuid 并重试
}
```

### 2. 日志获取 (`getSessionLogs`)
使用 JWT token 获取会话日志：
- 用于 CCR 环境的会话 hydration
- 更新本地 `lastUuidMap` 以同步状态

### 3. OAuth 日志获取 (`getSessionLogsViaOAuth`)
使用 OAuth token 获取会话日志：
- 用于 Teleport 功能
- 端点: `/v1/session_ingress/session/{sessionId}`

### 4. Teleport 事件获取 (`getTeleportEvents`)
新的 v2 Sessions API 事件获取：
- 端点: `/v1/code/sessions/{sessionId}/teleport-events`
- **分页支持**：默认 500/页，最大 1000/页，最多 100 页
- 无限循环保护：超过 100 页自动停止
- 404 处理：区分"会话不存在"和"端点未部署"

### 5. 状态管理
- `lastUuidMap`: 每会话的最后 UUID 缓存
- `sequentialAppendBySession`: 每会话的顺序执行包装器
- `clearSession` / `clearAllSessions`: 清理状态

---

## 具体技术实现

### 关键数据结构

```typescript
// 会话入口错误响应
interface SessionIngressError {
  error?: {
    message?: string
    type?: string
  }
}

// Teleport 事件响应（v2 API）
type TeleportEventsResponse = {
  data: Array<{
    event_id: string
    event_type: string
    is_compaction: boolean
    payload: Entry | null  // 实际就是 TranscriptMessage
    created_at: string
  }>
  next_cursor?: string  // 分页游标，undefined 表示结束
}

// 模块级状态
const lastUuidMap: Map<string, UUID> = new Map()
const sequentialAppendBySession: Map<string, SequentialFunction> = new Map()

// 重试配置
const MAX_RETRIES = 10
const BASE_DELAY_MS = 500
```

### 关键流程

#### 日志追加流程（含重试和 409 处理）
```
appendSessionLog(sessionId, entry, url)
├── 获取 session token
├── 获取或创建 sequential 包装器
└── sequentialAppend(entry, url, headers)
    └── appendSessionLogImpl(sessionId, entry, url, headers)
        ├── 设置 Last-Uuid 头（如果有）
        ├── 发送 PUT 请求
        ├── 200/201 → 更新 lastUuidMap，返回 true
        ├── 409 → 处理冲突
        │   ├── 检查是否我们的条目已是最后一条 → 恢复状态
        │   ├── 采用服务器的 x-last-uuid → 重试
        │   └── 无 x-last-uuid → 重新获取日志链
        ├── 401 → 返回 false（非重试able）
        └── 其他/网络错误 → 指数退避重试
```

#### Teleport 事件分页获取流程
```
getTeleportEvents(sessionId, accessToken, orgUUID)
├── 构建请求头
├── 循环获取（最多 100 页）
│   ├── 发送 GET 请求（带 cursor）
│   ├── 404（第一页）→ 返回 null（调用方回退到 session-ingress）
│   ├── 404（后续页）→ 返回已获取数据
│   ├── 401 → 抛出登录过期错误
│   ├── 200 → 解析 payload，过滤 null
│   └── next_cursor 为 null → 结束循环
└── 返回 Entry[]
```

### 顺序执行保证

```typescript
function getOrCreateSequentialAppend(sessionId: string) {
  let sequentialAppend = sequentialAppendBySession.get(sessionId)
  if (!sequentialAppend) {
    sequentialAppend = sequential(
      async (entry, url, headers) => 
        await appendSessionLogImpl(sessionId, entry, url, headers)
    )
    sequentialAppendBySession.set(sessionId, sequentialAppend)
  }
  return sequentialAppend
}
```

使用 `src/utils/sequential.ts` 的 `sequential` 包装器，确保同一 session 的日志追加串行执行。

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/sessionIngress.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/utils/sessionStorage.ts` | `appendSessionLog`, `getSessionLogs` | 会话存储集成 |
| `src/utils/teleport.tsx` | `getSessionLogsViaOAuth`, `getTeleportEvents` | Teleport 功能 |
| `src/commands/clear/caches.ts` | `clearAllSessions` | /clear 命令清理 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/types/logs.ts` | `Entry`, `TranscriptMessage` 类型 |
| `src/utils/sessionIngressAuth.ts` | `getSessionIngressAuthToken` |
| `src/utils/teleport/api.ts` | `getOAuthHeaders`, `prepareApiRequest` |
| `src/utils/sequential.ts` | `sequential` 包装器 |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/diagLogs.ts` | `logForDiagnosticsNoPII` |
| `src/utils/log.ts` | `logError` |
| `src/utils/sleep.ts` | `sleep` |
| `src/utils/slowOperations.ts` | `jsonStringify` |
| `src/utils/envUtils.ts` | `isEnvTruthy` |
| `src/constants/oauth.ts` | `getOauthConfig` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 认证 | 超时 | 用途 |
|------|------|------|------|------|
| `/v1/session_ingress/session/{sessionId}` | PUT | JWT Bearer/Cookie | - | 追加日志条目 |
| `/v1/session_ingress/session/{sessionId}` | GET | JWT Bearer/Cookie | 20s | 获取会话日志 |
| `/v1/session_ingress/session/{sessionId}` | GET | OAuth Bearer | 20s | OAuth 方式获取日志 |
| `/v1/code/sessions/{sessionId}/teleport-events` | GET | OAuth + x-organization-uuid | 20s | v2 Teleport 事件 |

### 认证方式

#### JWT Token（CCR 环境）
```typescript
// 从环境变量或文件描述符获取
const sessionToken = getSessionIngressAuthToken()
const headers = {
  Authorization: `Bearer ${sessionToken}`,
  'Content-Type': 'application/json',
}
```

#### OAuth Token（Teleport）
```typescript
const headers = {
  ...getOAuthHeaders(accessToken),
  'x-organization-uuid': orgUUID,
}
```

### 乐观并发控制

使用 `Last-Uuid` 头维护追加链的一致性：
```typescript
const lastUuid = lastUuidMap.get(sessionId)
if (lastUuid) {
  requestHeaders['Last-Uuid'] = lastUuid
}
```

服务器返回 `x-last-uuid` 头用于 409 冲突恢复。

### 诊断日志

| 事件 | 级别 | 用途 |
|------|------|------|
| `session_persist_recovered_from_409` | info | 409 冲突恢复 |
| `session_persist_409_adopt_server_uuid` | info | 采用服务器 UUID |
| `session_persist_fail_concurrent_modification` | error | 无法解决的冲突 |
| `session_persist_fail_bad_token` | error | 认证失败 |
| `session_persist_fail_status` | error | 其他状态码失败 |
| `session_persist_error_retries_exhausted` | error | 重试耗尽 |
| `teleport_events_fetch_fail` | error | Teleport 获取失败 |
| `teleport_events_not_found` | warn | 会话不存在 |

---

## 风险、边界与改进建议

### 已知风险

1. **内存泄漏**
   ```typescript
   const lastUuidMap: Map<string, UUID> = new Map()
   const sequentialAppendBySession: Map<string, SequentialFunction> = new Map()
   ```
   - 长期运行的 CLI 会话可能积累大量 subagent 条目
   - `clearAllSessions()` 可清理，但依赖 `/clear` 命令调用

2. **409 恢复复杂性**
   - 需要处理服务器返回/不返回 `x-last-uuid` 的两种情况
   - 重新获取日志链可能很昂贵（最多 50k 条目）

3. **分页上限**
   ```typescript
   const maxPages = 100  // 1000/page × 100 = 100k 事件
   ```
   - 超大会话可能触发上限，导致截断

4. **JWT Token 过期**
   - 无自动刷新机制
   - 401 错误直接返回 false 或抛出

### 边界情况

| 场景 | 行为 |
|------|------|
| 无 session token | 记录错误，返回 false/null |
| 409 + 我们的条目已是最后一条 | 视为成功，恢复状态 |
| 409 + 无 x-last-uuid | 重新获取完整日志链 |
| 409 + 无法确定服务器状态 | 返回 false（放弃）|
| Teleport 404（第一页） | 返回 null，调用方回退到 session-ingress |
| Teleport 404（后续页） | 返回已获取数据（会话可能已被删除）|
| 超过 maxPages | 返回已获取数据，记录警告 |
| `CLAUDE_AFTER_LAST_COMPACT` | 添加 `after_last_compact` 查询参数 |

### 改进建议

1. **自动 Token 刷新**
   ```typescript
   // 当前
   if (response.status === 401) {
     return false // 或抛出
   }
   
   // 建议：尝试刷新 session token
   if (response.status === 401) {
     const newToken = await refreshSessionToken()
     if (newToken) {
       updateSessionIngressAuthToken(newToken)
       continue // 重试
     }
   }
   ```

2. **内存管理优化**
   - 添加 LRU 淘汰策略
   - 或定期清理不活跃会话的状态

3. **409 恢复优化**
   - 添加专门的 409 恢复端点（只返回最后 UUID）
   - 避免完整日志链重新获取

4. **重试策略配置**
   ```typescript
   // 当前硬编码
   const MAX_RETRIES = 10
   const BASE_DELAY_MS = 500
   
   // 建议可配置
   const MAX_RETRIES = getConfig('sessionIngress.maxRetries', 10)
   ```

5. **批量追加支持**
   - 当前每次调用追加单个条目
   - 考虑支持批量追加以减少 API 调用

6. **Analytics 事件**
   ```typescript
   // 建议添加事件
   logEvent('tengu_session_persist_latency', {
     sessionId,
     durationMs,
     attempts,
     success
   })
   ```

7. **测试覆盖**
   - 添加 409 冲突恢复的单元测试
   - 测试分页获取的各种边界
   - 测试顺序执行保证

### 相关模式
- 与 `sessionIngressAuth.ts` 紧密集成（token 获取）
- 与 `teleport/api.ts` 共享 OAuth 头构建
- 与 `sequential.ts` 配合实现顺序执行
- 与 `sessionStorage.ts` 集成（主要调用方）

### 迁移说明
Teleport 事件 API (`getTeleportEvents`) 是 session-ingress 的替代方案：
- session-ingress: 一次性返回最多 50k 条目
- Teleport 事件: 分页，默认 500/页，支持到 100k+
- 当前实现支持 404 回退，兼容迁移期间
