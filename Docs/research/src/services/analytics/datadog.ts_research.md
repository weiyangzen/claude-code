# datadog.ts 研究文档

## 场景与职责

`datadog.ts` 实现 Claude Code 的 Datadog 日志分析集成，负责将应用事件发送到 Datadog 日志平台用于监控和告警。该模块采用批量发送策略，优化网络性能并减少 API 调用次数。

核心职责：
- 事件过滤（仅允许白名单事件）
- 批量日志收集与定时刷新
- 用户分桶（用于影响范围估算而不暴露用户 ID）
- 元数据归一化（模型名、版本号、MCP 工具名等）

## 功能点目的

### 1. 事件白名单机制 (`DATADOG_ALLOWED_EVENTS`)

**目的**：严格控制可发送到 Datadog 的事件类型，防止敏感事件意外泄露。

**当前允许的事件**（64 个）：
- Chrome Bridge 相关：`chrome_bridge_connection_*`, `chrome_bridge_tool_call_*`
- API 事件：`tengu_api_error`, `tengu_api_success`
- 用户交互：`tengu_brief_mode_*`, `tengu_cancel`, `tengu_exit`
- OAuth 流程：`tengu_oauth_*`（包含详细的令牌刷新步骤）
- 工具使用：`tengu_tool_use_*`
- 团队内存同步：`tengu_team_mem_sync_*`

**设计原则**：事件名必须经过显式审查才能加入白名单。

### 2. 批量日志发送

**目的**：减少网络请求次数，提高性能。

**配置参数**：
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `DEFAULT_FLUSH_INTERVAL_MS` | 15000 | 日志刷新间隔 |
| `MAX_BATCH_SIZE` | 100 | 单批次最大日志数 |
| `NETWORK_TIMEOUT_MS` | 5000 | 网络请求超时 |

**刷新策略**：
1. 定时刷新：默认 15 秒间隔
2. 批量满时立即刷新：达到 100 条时
3. 优雅关闭时强制刷新

### 3. 用户分桶 (`getUserBucket`)

**目的**：估算受影响用户数量而不暴露具体用户 ID。

**实现**：
```typescript
const NUM_USER_BUCKETS = 30
const getUserBucket = memoize((): number => {
  const userId = getOrCreateUserID()
  const hash = createHash('sha256').update(userId).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

**原理**：
- 将用户 ID 哈希后映射到 0-29 的桶
- 告警时统计唯一桶数量即可估算受影响用户数
- 保护用户隐私（无法从桶号反推用户）

### 4. 数据归一化

**MCP 工具名归一化**：
```typescript
if (typeof allData.toolName === 'string' && allData.toolName.startsWith('mcp__')) {
  allData.toolName = 'mcp'
}
```
- 目的：降低基数（cardinality），避免高基数标签导致 Datadog 性能问题

**模型名归一化**（外部用户）：
```typescript
if (process.env.USER_TYPE !== 'ant' && typeof allData.model === 'string') {
  const shortName = getCanonicalName(allData.model.replace(/\[1m]$/i, ''))
  allData.model = shortName in MODEL_COSTS ? shortName : 'other'
}
```

**版本号截断**：
```typescript
allData.version = allData.version.replace(
  /^(\d+\.\d+\.\d+-dev\.\d{8})\.t\d+\.sha[a-f0-9]+$/,
  '$1',
)
// "2.0.53-dev.20251124.t173302.sha526cc6a" → "2.0.53-dev.20251124"
```

## 具体技术实现

### 数据结构

```typescript
type DatadogLog = {
  ddsource: string      // 'nodejs'
  ddtags: string        // 逗号分隔的标签，如 "event:tengu_init,platform:linux"
  message: string       // 事件名
  service: string       // 'claude-code'
  hostname: string      // 'claude-code'
  [key: string]: unknown // 动态属性（事件特定数据）
}
```

### 标签字段 (`TAG_FIELDS`)

预定义的可搜索标签字段：
```typescript
const TAG_FIELDS = [
  'arch', 'clientType', 'errorType', 'http_status_range', 'http_status',
  'kairosActive', 'model', 'platform', 'provider', 'skillMode',
  'subscriptionType', 'toolName', 'userBucket', 'userType', 'version', 'versionBase',
]
```

**注意**：`event:<name>` 标签被显式添加，因为 `message` 字段是 Datadog 保留字段，无法用于仪表板查询。

### 核心函数流程

#### `trackDatadogEvent(eventName, properties)`

```
1. 环境检查（仅 production + firstParty 提供商）
2. 初始化检查（跳过如果未初始化或事件不在白名单）
3. 获取事件元数据（异步）
4. 数据归一化：
   - MCP 工具名 → 'mcp'
   - 模型名（外部用户）
   - 版本号截断
   - status → http_status + http_status_range
5. 构建标签（ddtags）
6. 构建日志对象
7. 加入批次队列
8. 触发刷新逻辑（定时或立即）
```

#### `flushLogs()`

```
1. 检查批次是否为空
2. 清空批次（避免并发问题）
3. 发送 HTTP POST 到 Datadog
4. 错误时调用 logError（不抛异常）
```

### API 端点

```typescript
const DATADOG_LOGS_ENDPOINT = 'https://http-intake.logs.us5.datadoghq.com/api/v2/logs'
const DATADOG_CLIENT_TOKEN = 'pubbbf48e6d78dae54bceaa4acf463299bf'
```

**注意**：Client Token 是公开的（pub 前缀），仅用于日志摄入，无安全风险。

## 关键代码路径与文件引用

### 被调用方

| 文件 | 调用 | 说明 |
|------|------|------|
| `src/services/analytics/sink.ts` | `trackDatadogEvent()` | 分析 Sink 的路由调用 |

### 依赖文件

| 文件 | 提供功能 |
|------|----------|
| `src/services/analytics/config.ts` | `isAnalyticsDisabled()` |
| `src/services/analytics/metadata.ts` | `getEventMetadata()` |
| `src/utils/config.ts` | `getOrCreateUserID()` |
| `src/utils/log.ts` | `logError()` |
| `src/utils/model/model.ts` | `getCanonicalName()` |
| `src/utils/model/providers.ts` | `getAPIProvider()` |
| `src/utils/modelCost.ts` | `MODEL_COSTS` |

### 调用链

```
main.tsx → initializeAnalyticsSink() → attachAnalyticsSink()
  ↓
业务代码调用 logEvent()
  ↓
sink.ts logEventImpl()
  ↓
trackDatadogEvent()（如果 Gate 启用）
  ↓
axios.post() → Datadog
```

## 依赖与外部交互

### 外部服务

**Datadog Logs API**:
- 端点：`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- 认证：DD-API-KEY 头（Client Token）
- 格式：JSON 数组
- 超时：5 秒

### 环境变量

| 变量 | 用途 |
|------|------|
| `NODE_ENV` | 仅 production 环境发送 |
| `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS` | 测试时覆盖刷新间隔 |
| `USER_TYPE` | 区分 ant 和外部用户的数据处理 |

### 第三方库

- `axios`: HTTP 请求
- `lodash-es/memoize`: 函数缓存
- `crypto`: SHA256 哈希

## 风险、边界与改进建议

### 风险点

1. **Client Token 硬编码**
   - 风险：Token 泄露可能导致日志被污染
   - 缓解：Token 是只读的（pub 前缀），只能写入不能读取
   - 建议：考虑从配置文件或构建时注入

2. **批次丢失风险**
   - 场景：进程崩溃时未刷新的日志会丢失
   - 缓解：15 秒间隔较短，优雅关闭时会强制刷新
   - 改进：考虑磁盘持久化关键事件

3. **网络阻塞**
   - 场景：flushLogs 是异步的，但大量事件可能导致内存增长
   - 缓解：MAX_BATCH_SIZE 限制和超时机制

4. **基数爆炸**
   - 风险：动态标签值过多会导致 Datadog 计费问题
   - 缓解：严格的归一化和白名单机制

### 边界情况

1. **初始化失败**
   - `initializeDatadog` 捕获所有错误，失败时 `datadogInitialized = false`
   - 后续调用会快速返回，不影响业务

2. **网络超时**
   - 5 秒超时后错误被记录但不抛异常
   - 事件丢失，无重试机制（设计选择，避免阻塞）

3. **并发刷新**
   - `scheduleFlush` 检查 `flushTimer` 避免重复调度
   - `flushLogs` 先复制批次再清空，避免竞态

### 改进建议

1. **可观测性增强**
   ```typescript
   // 添加指标：批次大小、刷新延迟、失败率
   interface DatadogMetrics {
     batchSize: number
     flushLatencyMs: number
     failedEvents: number
   }
   ```

2. **自适应批处理**
   - 根据网络状况动态调整批量大小
   - 高延迟网络使用更小批次

3. **本地队列持久化**
   - 关键事件（如崩溃）写入磁盘
   - 下次启动时重试发送

4. **测试支持**
   ```typescript
   // 添加测试钩子
   export function _getPendingBatch(): DatadogLog[] {
     return [...logBatch]
   }
   ```

5. **配置外部化**
   ```typescript
   // 将硬编码配置移至 GrowthBook
   const config = getDynamicConfig_CACHED_MAY_BE_STALE('datadog_config', {
     flushIntervalMs: 15000,
     maxBatchSize: 100,
   })
   ```
