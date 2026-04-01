# firstPartyEventLoggingExporter.ts 研究文档

## 场景与职责

`firstPartyEventLoggingExporter.ts` 实现 OpenTelemetry `LogRecordExporter` 接口，负责将 1P 分析事件实际发送到 Anthropic 的事件收集 API（`/api/event_logging/batch`）。这是 1P 事件日志系统的核心导出组件，提供企业级的可靠性和容错能力。

核心职责：
- 事件批量导出到 `/api/event_logging/batch`
- 失败事件的磁盘持久化和重试
- 指数退避重试机制
- 认证回退（401 时尝试无认证发送）
- 进程重启后恢复未发送事件

## 功能点目的

### 1. 可靠的事件导出

**目的**：确保事件最终到达服务器，即使在网络不稳定时。

**可靠性机制**：
| 机制 | 实现 |
|------|------|
| 批量分块 | 大批次拆分为小批次（默认 200） |
| 批次间延迟 | 批次间 100ms 延迟，避免压垮服务器 |
| 短路机制 | 第一个批次失败时跳过剩余批次 |
| 磁盘持久化 | 失败事件写入本地文件 |
| 指数退避 | 二次退避：500ms × attempts² |
| 最大重试 | 默认 8 次后丢弃 |

### 2. 磁盘持久化 (`queueFailedEvents`)

**目的**：进程重启后仍能恢复未发送的事件。

**存储位置**：
```typescript
function getStorageDir(): string {
  return path.join(getClaudeConfigHomeDir(), 'telemetry')
}
// ~/.claude/telemetry/
```

**文件命名**：
```typescript
`${FILE_PREFIX}${getSessionId()}.${BATCH_UUID}.json`
// 1p_failed_events.<sessionId>.<batchUUID>.json
```

**格式**：JSON Lines（每行一个事件）

**恢复机制**：
- 启动时扫描 `retryPreviousBatches()`
- 排除当前 `BATCH_UUID` 的文件
- 后台异步重试，避免阻塞启动

### 3. 认证策略

**目的**：支持多种认证状态，优雅降级。

**认证检查流程**：
```
1. 检查 trust dialog 是否已接受
2. 检查是否非交互式会话（隐式信任）
3. 检查 OAuth token 是否过期
4. 检查是否有 profile scope
5. 尝试带认证头发送
6. 401 错误时回退到无认证发送
```

**无认证发送**：
- 事件仍被接受，但关联性降低
- 用于服务密钥会话或 token 过期场景

### 4. 数据转换 (`transformLogsToEvents`)

**目的**：将 OTel LogRecord 转换为 API 期望的 Protobuf JSON 格式。

**支持的两种事件类型**：

**GrowthbookExperimentEvent**：
```typescript
{
  event_type: 'GrowthbookExperimentEvent',
  event_data: GrowthbookExperimentEvent.toJSON({
    event_id, timestamp, experiment_id, variation_id,
    environment, user_attributes, experiment_metadata,
    device_id, session_id, auth
  })
}
```

**ClaudeCodeInternalEvent**：
```typescript
{
  event_type: 'ClaudeCodeInternalEvent',
  event_data: ClaudeCodeInternalEvent.toJSON({
    event_id, event_name, client_timestamp, device_id, email,
    auth, ...core, env, process,
    skill_name, plugin_name, marketplace_name,
    additional_metadata: base64(json(additional))
  })
}
```

**PII 处理**：
- `_PROTO_*` 字段被提取到顶层 proto 字段
- 剩余字段被 strip，防止意外泄露
- `additional_metadata` 使用 base64 编码

### 5. Killswitch 支持

**目的**：紧急情况下停止所有 1P 事件流量。

**实现**：
```typescript
private async sendBatchWithRetry(payload): Promise<void> {
  if (this.isKilled()) {
    throw new Error('firstParty sink killswitch active')
  }
  // ...
}
```

**行为**：
- 检查失败时抛出错误
- 调用方将事件短路到磁盘
- 退避计时器继续运行
- killswitch 清除后自动恢复

## 具体技术实现

### 类结构

```typescript
export class FirstPartyEventLoggingExporter implements LogRecordExporter {
  private readonly endpoint: string
  private readonly timeout: number
  private readonly maxBatchSize: number
  private readonly skipAuth: boolean
  private readonly maxAttempts: number
  private readonly isKilled: () => boolean
  private pendingExports: Promise<void>[] = []
  private isShutdown = false
  private attempts = 0
  private isRetrying = false
  // ...
}
```

### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `timeout` | 10000 | HTTP 超时（毫秒） |
| `maxBatchSize` | 200 | 单批次最大事件数 |
| `skipAuth` | false | 是否完全跳过认证 |
| `batchDelayMs` | 100 | 批次间延迟 |
| `baseBackoffDelayMs` | 500 | 退避基础延迟 |
| `maxBackoffDelayMs` | 30000 | 最大退避延迟 |
| `maxAttempts` | 8 | 最大重试次数 |

### 核心方法流程

#### `export(logs, resultCallback)`

```
1. 检查 shutdown 状态
2. 过滤事件（仅处理 instrumentationScope === 'com.anthropic.claude_code.events'）
3. 转换日志为事件
4. 检查重试次数限制
5. 调用 sendEventsInBatches
6. 处理失败事件：
   - 写入磁盘
   - 调度退避重试
7. 调用 resultCallback 返回结果
```

#### `sendEventsInBatches(events)`

```
1. 将事件分块（chunk size = maxBatchSize）
2. 逐个发送批次
3. 第一个失败时短路（跳过剩余批次）
4. 批次间 sleep(batchDelayMs)
5. 返回失败的事件列表
```

#### `sendBatchWithRetry(payload)`

```
1. 检查 killswitch
2. 确定认证策略（有/无认证）
3. 尝试带认证发送
4. 401 错误时回退到无认证
5. 其他错误抛出
```

#### `retryFailedEvents()`

```
1. 循环直到队列为空或 shutdown
2. 检查重试次数限制
3. 从磁盘加载事件
4. 删除磁盘文件（乐观删除）
5. 发送事件
6. 失败时写回磁盘并调度重试
7. 成功时重置退避并继续循环
```

### 错误处理

**Axios 错误上下文提取** (`getAxiosErrorContext`)：
```typescript
function getAxiosErrorContext(error: unknown): string {
  // 提取：request-id, status, code, message
  // 格式："request-id=xxx, status=401, code=ECONNRESET, message..."
}
```

**错误分类**：
- 网络错误（ECONNRESET, ETIMEDOUT）：可重试
- 401 未授权：尝试无认证回退
- 4xx 客户端错误：不重试（数据问题）
- 5xx 服务器错误：可重试

## 关键代码路径与文件引用

### 被调用方

| 文件 | 调用 | 说明 |
|------|------|------|
| `src/services/analytics/firstPartyEventLogger.ts` | `new FirstPartyEventLoggingExporter()` | 初始化时创建 |

### 依赖文件

| 文件 | 提供功能 |
|------|----------|
| `src/bootstrap/state.ts` | `getSessionId()`, `getIsNonInteractiveSession()` |
| `src/types/generated/events_mono/...` | Protobuf 类型定义 |
| `src/utils/auth.ts` | OAuth token 检查 |
| `src/utils/config.ts` | `checkHasTrustDialogAccepted()` |
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir()` |
| `src/utils/errors.ts` | `errorMessage()`, `isFsInaccessible()` |
| `src/utils/http.ts` | `getAuthHeaders()` |
| `src/utils/json.ts` | `readJSONLFile()` |
| `src/utils/log.ts` | `logError()` |
| `src/utils/sleep.ts` | `sleep()` |
| `src/utils/slowOperations.ts` | `jsonStringify()` |
| `src/utils/userAgent.ts` | `getClaudeCodeUserAgent()` |
| `src/services/oauth/client.ts` | `isOAuthTokenExpired()` |
| `src/services/analytics/index.ts` | `stripProtoFields()` |
| `src/services/analytics/metadata.ts` | `to1PEventFormat()` |

### 存储文件

| 路径 | 用途 |
|------|------|
| `~/.claude/telemetry/1p_failed_events.<session>.<uuid>.json` | 失败事件队列 |

## 依赖与外部交互

### 外部服务

**Anthropic Event Logging API**:
- 端点：`POST /api/event_logging/batch`
- 基础 URL：`https://api.anthropic.com` 或 `https://api-staging.anthropic.com`
- 请求头：
  - `Content-Type: application/json`
  - `User-Agent: <claude-code user agent>`
  - `x-service-name: claude-code`
  - `Authorization: Bearer <token>`（可选）
- 请求体：`{ events: FirstPartyEventLoggingEvent[] }`
- 超时：10 秒

### Protobuf 类型

**生成文件**：
- `src/types/generated/events_mono/claude_code/v1/claude_code_internal_event.js`
- `src/types/generated/events_mono/growthbook/v1/growthbook_experiment_event.js`

**使用方式**：
```typescript
ClaudeCodeInternalEvent.toJSON({ ... })  // 序列化
```

### 文件系统操作

**使用 API**：
- `fs/promises`: `appendFile`, `mkdir`, `readdir`, `unlink`, `writeFile`
- `path`: 路径拼接

**并发安全**：
- 追加写入使用 `appendFile`（在大多数文件系统上是原子的）
- 每个进程有唯一的 `BATCH_UUID`，避免文件冲突

## 风险、边界与改进建议

### 风险点

1. **磁盘空间耗尽**
   - 场景：长期离线使用，失败事件持续累积
   - 缓解：无自动清理机制
   - 建议：添加最大文件年龄/大小限制

2. **会话 ID 变化**
   - 场景：会话恢复后 sessionId 变化
   - 影响：旧会话的失败事件文件不会被清理
   - 建议：定期扫描并清理过期文件

3. **并发重试**
   - 场景：`retryPreviousBatches` 和 `retryFailedEvents` 同时运行
   - 缓解：`isRetrying` 标志防止并发
   - 风险：标志非原子操作

4. **认证信息泄露**
   - 场景：失败事件文件包含认证头
   - 现状：事件数据不包含认证头
   - 缓解：认证在发送时动态添加

### 边界情况

1. **空批次处理**
   - `export` 快速返回 SUCCESS
   - `sendEventsInBatches` 返回空数组

2. **全部事件失败**
   - 整个批次写入磁盘
   - 调度重试
   - 达到最大重试后丢弃

3. **部分事件失败**
   - 仅失败的事件写回磁盘
   - 成功的事件确认

4. **进程崩溃恢复**
   - 下次启动时 `retryPreviousBatches`
   - 按文件逐个重试
   - 完全失败后保留文件

5. **Shutdown 期间**
   - 新 `export` 调用返回 FAILED
   - 正在进行的导出继续完成
   - `forceFlush` 等待所有 pending

### 改进建议

1. **事件压缩**
   ```typescript
   // 使用 gzip 压缩磁盘存储
   const compressed = await gzip(jsonStringify(events))
   await writeFile(filePath, compressed)
   ```

2. **智能重试策略**
   ```typescript
   // 根据错误类型调整重试策略
   interface RetryStrategy {
     maxAttempts: number
     backoffMultiplier: number
     retryableStatuses: number[]
   }
   ```

3. **监控指标**
   ```typescript
   // 导出关键指标
   export interface ExporterMetrics {
     eventsExported: number
     eventsFailed: number
     eventsQueued: number
     retryAttempts: number
     diskUsageBytes: number
   }
   ```

4. **批量删除优化**
   ```typescript
   // 重试成功后批量删除旧文件
   async function cleanupOldFiles(maxAgeMs: number): Promise<void>
   ```

5. **网络状态感知**
   ```typescript
   // 检测网络可用性，避免无效重试
   if (!navigator.onLine) {
     this.scheduleRetry(LONGER_DELAY)
   }
   ```

6. **事件去重**
   ```typescript
   // 使用 event_id 去重
   const seenEventIds = new Set<string>()
   events = events.filter(e => !seenEventIds.has(e.event_id))
   ```
