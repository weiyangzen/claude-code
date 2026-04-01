# firstPartyEventLogger.ts 研究文档

## 场景与职责

`firstPartyEventLogger.ts` 实现 Claude Code 的第一方（1P）内部事件日志系统，用于将分析事件发送到 Anthropic 自己的事件收集服务（`/api/event_logging/batch`）。这是与 Datadog 并行的另一条分析管道，用于内部数据分析和产品改进。

核心职责：
- OpenTelemetry 日志 SDK 集成
- 事件采样（基于 GrowthBook 动态配置）
- 事件批处理和导出
- GrowthBook 实验曝光日志
- 配置热更新支持（GrowthBook 刷新时重建管道）

## 功能点目的

### 1. 事件采样 (`shouldSampleEvent`)

**目的**：通过配置动态控制事件采样率，在高流量时减少数据量。

**配置格式**：
```typescript
type EventSamplingConfig = {
  [eventName: string]: {
    sample_rate: number  // 0-1 之间
  }
}
```

**采样逻辑**：
- 未配置事件：100% 采样（返回 null）
- `sample_rate >= 1`：100% 采样
- `sample_rate <= 0`：完全丢弃（返回 0）
- `0 < sample_rate < 1`：随机采样，返回实际使用的 rate

**配置来源**：GrowthBook 动态配置 `tengu_event_sampling_config`

### 2. 异步事件日志 (`logEventTo1P` / `logEventTo1PAsync`)

**目的**：提供非阻塞的事件日志接口。

**设计特点**：
- 同步接口 `logEventTo1P`：立即返回，后台异步处理
- 异步接口 `logEventTo1PAsync`：返回 Promise，但通常 fire-and-forget
- 元数据自动丰富：模型、会话、环境上下文等

**数据结构**：
```typescript
const attributes = {
  event_name: eventName,
  event_id: randomUUID(),
  core_metadata: coreMetadata,      // 事件元数据
  user_metadata: getCoreUserData(true),
  event_metadata: metadata,         // 调用方传入的数据
  user_id: userId,                  // 可选
}
```

### 3. GrowthBook 实验曝光日志 (`logGrowthBookExperimentTo1P`)

**目的**：记录用户被分配到哪个实验组，用于实验效果分析。

**数据格式**：
```typescript
{
  event_type: 'GrowthbookExperimentEvent',
  event_id: randomUUID(),
  experiment_id: data.experimentId,
  variation_id: data.variationId,
  device_id: userId,
  account_uuid: accountUuid,
  organization_uuid: organizationUuid,
  session_id: sessionId,
  user_attributes: jsonStringify(data.userAttributes),
  experiment_metadata: jsonStringify(data.experimentMetadata),
  environment: 'production',
}
```

**注意**：仅当 1P 事件日志启用时记录。

### 4. 批处理配置 (`getBatchConfig`)

**目的**：从 GrowthBook 动态获取批处理参数。

**可配置项**：
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `scheduledDelayMillis` | 10000 | 调度延迟（毫秒） |
| `maxExportBatchSize` | 200 | 最大导出批次大小 |
| `maxQueueSize` | 8192 | 最大队列大小 |
| `skipAuth` | false | 是否跳过认证 |
| `maxAttempts` | 8 | 最大重试次数 |
| `path` | '/api/event_logging/batch' | API 路径 |
| `baseUrl` | - | 基础 URL（默认 anthropic API） |

### 5. 管道热更新 (`reinitialize1PEventLoggingIfConfigChanged`)

**目的**：当 GrowthBook 配置变化时，重建事件日志管道以应用新配置。

**安全机制**：
1. 先置空 logger，新事件在交换期间被丢弃
2. `forceFlush()` 排空旧处理器缓冲区
3. 失败时回退到旧 provider
4. 后台关闭旧 provider

**事件丢失防护**：
- 缓冲区数据通过 `forceFlush` 发送到 exporter
- 导出失败的数据已写入磁盘（由 exporter 处理）
- 新 exporter 会读取磁盘重试文件

## 具体技术实现

### OpenTelemetry 集成

```typescript
import { LoggerProvider, BatchLogRecordProcessor } from '@opentelemetry/sdk-logs'
import { resourceFromAttributes } from '@opentelemetry/resources'
```

**资源属性**：
```typescript
const attributes: Record<string, string> = {
  [ATTR_SERVICE_NAME]: 'claude-code',
  [ATTR_SERVICE_VERSION]: MACRO.VERSION,
  'wsl.version': wslVersion,  // 仅在 WSL 上
}
```

**关键设计**：使用独立的 `LoggerProvider`，与客户 OTLP 遥测完全隔离。

### 模块状态

```typescript
let firstPartyEventLogger: ReturnType<typeof logs.getLogger> | null = null
let firstPartyEventLoggerProvider: LoggerProvider | null = null
let lastBatchConfig: BatchConfig | null = null
```

**注意**：状态不暴露给全局，防止被其他模块误用。

### 核心流程

#### 初始化流程 (`initialize1PEventLogging`)

```
1. 检查是否启用（is1PEventLoggingEnabled）
2. 从 GrowthBook 获取批处理配置
3. 构建 OpenTelemetry Resource
4. 创建 FirstPartyEventLoggingExporter
5. 创建 LoggerProvider 和 BatchLogRecordProcessor
6. 获取 Logger 实例
```

#### 事件日志流程 (`logEventTo1P`)

```
1. 检查启用状态和 killswitch
2. 检查 logger 是否初始化
3. 调用 logEventTo1PAsync（fire-and-forget）
4. 异步 enrich 元数据
5. 构建 OTel LogRecord
6. logger.emit()
```

#### 关闭流程 (`shutdown1PEventLogging`)

```
1. 检查 provider 是否存在
2. 调用 provider.shutdown()
3. 等待所有导出完成
4. 错误被吞掉（优雅关闭）
```

## 关键代码路径与文件引用

### 被调用方

| 文件 | 调用函数 | 说明 |
|------|----------|------|
| `src/services/analytics/sink.ts` | `logEventTo1P()` | 分析 Sink 路由 |
| `src/services/analytics/growthbook.ts` | `logGrowthBookExperimentTo1P()` | 实验曝光日志 |

### 依赖文件

| 文件 | 提供功能 |
|------|----------|
| `src/services/analytics/config.ts` | `isAnalyticsDisabled()` |
| `src/services/analytics/firstPartyEventLoggingExporter.ts` | `FirstPartyEventLoggingExporter` |
| `src/services/analytics/growthbook.ts` | `getDynamicConfig_CACHED_MAY_BE_STALE()` |
| `src/services/analytics/metadata.ts` | `getEventMetadata()` |
| `src/services/analytics/sinkKillswitch.ts` | `isSinkKilled()` |
| `src/utils/config.ts` | `getOrCreateUserID()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/log.ts` | `logError()` |
| `src/utils/platform.ts` | `getPlatform()`, `getWslVersion()` |
| `src/utils/slowOperations.ts` | `jsonStringify()` |
| `src/utils/startupProfiler.ts` | `profileCheckpoint()` |
| `src/utils/user.ts` | `getCoreUserData()` |

### 调用链

```
业务代码 → logEvent() → sink.ts
  ↓
logEventImpl() → logEventTo1P()
  ↓
logEventTo1PAsync() → getEventMetadata()
  ↓
firstPartyEventLogger.emit()
  ↓
BatchLogRecordProcessor → FirstPartyEventLoggingExporter
  ↓
POST /api/event_logging/batch
```

## 依赖与外部交互

### 外部服务

**Anthropic Event Logging API**:
- 端点：`/api/event_logging/batch`
- 基础 URL：`https://api.anthropic.com` 或 staging
- 认证：可选（OAuth 或匿名）
- 格式：Protobuf JSON 编码的事件数组

### OpenTelemetry SDK

**关键组件**：
- `LoggerProvider`: 日志提供者
- `BatchLogRecordProcessor`: 批量处理器
- `LogRecordExporter`: 自定义导出器（`FirstPartyEventLoggingExporter`）

### 环境变量

| 变量 | 用途 |
|------|------|
| `USER_TYPE` | ant 用户启用调试日志 |
| `NODE_ENV` | development 时抛出错误而非吞掉 |
| `OTEL_LOGS_EXPORT_INTERVAL` | 覆盖导出间隔 |
| `ANTHROPIC_BASE_URL` | 确定 staging/prod |

### 第三方库

- `@opentelemetry/api-logs`: OTel Logs API
- `@opentelemetry/sdk-logs`: OTel Logs SDK
- `@opentelemetry/resources`: 资源定义
- `@opentelemetry/semantic-conventions`: 标准属性名

## 风险、边界与改进建议

### 风险点

1. **事件丢失**
   - 场景：进程崩溃时未导出的内存中事件
   - 缓解：10 秒默认间隔较短，优雅关闭时强制刷新
   - 现状：OTel SDK 内存队列无持久化

2. **配置变化时的事件丢失**
   - 场景：`reinitialize1PEventLoggingIfConfigChanged` 期间
   - 缓解：先 flush 再交换，但仍有短暂窗口
   - 改进：使用双缓冲或队列暂存

3. **GrowthBook 依赖循环**
   - 风险：`is1PEventLoggingEnabled()` 被 `growthbook.ts` 调用
   - 现状：`isSinkKilled` 注释明确警告不要在此调用 GrowthBook
   - 缓解：killswitch 检查在 dispatch 点而非启用检查点

4. **内存泄漏**
   - 场景：长时间运行的会话，队列持续增长
   - 缓解：maxQueueSize 限制（默认 8192）
   - 超限时 OTel SDK 会丢弃旧事件

### 边界情况

1. **初始化失败**
   - `initialize1PEventLogging` 失败时静默返回
   - 后续 `logEventTo1P` 调用因 `!firstPartyEventLogger` 快速返回

2. **元数据获取失败**
   - `logEventTo1PAsync` 捕获所有错误
   - development 环境抛出，其他环境吞掉
   - ant 用户记录错误详情

3. **UUID 生成**
   - 每个事件生成新的 UUID
   - 用于事件去重和追踪

### 改进建议

1. **持久化队列**
   ```typescript
   // 添加磁盘持久化支持
   interface PersistentQueue {
     enqueue(event: LogRecord): void
     dequeue(): LogRecord | undefined
     sync(): Promise<void>
   }
   ```

2. **健康检查指标**
   ```typescript
   // 导出内部状态用于监控
   export function getEventLoggerHealth(): {
     queueSize: number
     lastFlushTime: number
     droppedEvents: number
   }
   ```

3. **批量配置热更新优化**
   - 当前重建整个管道开销较大
   - 考虑动态调整 BatchLogRecordProcessor 参数

4. **事件采样前置**
   - 当前在 sink 层采样，1P 层仍处理所有事件
   - 考虑将采样提前到 `logEventTo1P` 入口

5. **测试支持**
   ```typescript
   // 添加测试钩子
   export function _getPendingEventsForTesting(): LogRecord[]
   export function _resetForTesting(): void
   ```
