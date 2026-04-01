# sink.ts 深度研究文档

## 场景与职责

`sink.ts` 是 Claude Code 分析系统的核心路由模块，负责在应用启动时初始化分析后端，并将分析事件路由到两个主要目的地：

1. **Datadog**：用于一般访问的后端，接收脱敏后的事件数据
2. **1P 事件日志（First-Party Event Logging）**：用于内部分析，接收完整的事件数据

该模块实现了"sink"（接收器）模式，将事件生成与事件路由解耦，使得分析系统可以在应用启动后才附加具体的后端实现。

### 核心职责

1. **分析 Sink 初始化**：在应用启动时附加分析后端
2. **事件路由**：将事件分发到 Datadog 和 1P 事件日志
3. **采样控制**：根据配置决定是否采样特定事件
4. **PII 数据隔离**：在发送到 Datadog 前剥离 `_PROTO_*` 字段
5. **功能门控集成**：通过 GrowthBook 控制 Datadog 事件记录

---

## 功能点目的

### 1. Datadog 门控检查

```typescript
function shouldTrackDatadog(): boolean
```

**目的**：检查是否应该将事件发送到 Datadog。考虑因素：
- Sink killswitch 是否禁用了 Datadog
- 模块级门控状态 `isDatadogGateEnabled`
- 回退到缓存的 GrowthBook 值

### 2. 事件记录实现（同步）

```typescript
function logEventImpl(eventName: string, metadata: LogEventMetadata): void
```

**目的**：实际的事件记录逻辑：
1. 检查事件是否应该被采样
2. 如果采样结果为 0，丢弃事件
3. 如果 Datadog 门控启用，发送脱敏后的事件
4. 始终发送完整事件到 1P 事件日志

### 3. 事件记录实现（异步）

```typescript
function logEventAsyncImpl(eventName: string, metadata: LogEventMetadata): Promise<void>
```

**目的**：异步版本的事件记录。由于 Segment 已被移除，剩余的两个 sink（Datadog 和 1P）都是 fire-and-forget 模式，此函数仅包装同步实现以保持接口契约。

### 4. 分析门控初始化

```typescript
export function initializeAnalyticsGates(): void
```

**目的**：在应用启动时初始化分析门控值。从服务器更新门控值，早期事件使用上一会话的缓存值以避免初始化期间的数据丢失。

**调用位置**：`main.tsx` 的 `setupBackend()` 中

### 5. 分析 Sink 初始化

```typescript
export function initializeAnalyticsSink(): void
```

**目的**：附加分析后端。在应用启动期间调用，此前记录的事件会被排队并在 sink 附加后排出。

**特性**：
- 幂等性：多次调用安全（后续调用为无操作）
- 异步排出队列：使用 `queueMicrotask` 避免阻塞启动路径

---

## 具体技术实现

### 关键数据结构

```typescript
// 本地类型，匹配 logEvent 元数据签名
type LogEventMetadata = { [key: string]: boolean | number | undefined }

// 模块级门控状态
let isDatadogGateEnabled: boolean | undefined = undefined

// Datadog 功能门控名称
const DATADOG_GATE_NAME = 'tengu_log_datadog_events'
```

### 关键流程

#### 1. 事件记录流程

```
logEventImpl(eventName, metadata)
    │
    ├──> shouldSampleEvent(eventName)
    │       ├──> 返回 0 → 丢弃事件
    │       └──> 返回 sample_rate → 添加到 metadata
    │
    ├──> shouldTrackDatadog()
    │       ├──> isSinkKilled('datadog')? → false
    │       ├──> isDatadogGateEnabled? → 使用缓存值
    │       └──> checkStatsigFeatureGate_CACHED_MAY_BE_STALE(DATADOG_GATE_NAME)
    │
    ├──> shouldTrackDatadog() === true
    │       └──> trackDatadogEvent(eventName, stripProtoFields(metadataWithSampleRate))
    │
    └──> logEventTo1P(eventName, metadataWithSampleRate)
            └──> 接收完整 payload（包括 _PROTO_*）
```

#### 2. Sink 初始化流程

```
initializeAnalyticsSink()
    │
    └──> attachAnalyticsSink({ logEvent: logEventImpl, logEventAsync: logEventAsyncImpl })
            │
            ├──> sink !== null? → 幂等返回
            │
            ├──> 复制队列事件
            ├──> 清空事件队列
            │
            ├──> USER_TYPE === 'ant'? 
            │       └──> logEvent('analytics_sink_attached', { queued_event_count })
            │
            └──> queueMicrotask(() => {
                    for (event of queuedEvents) {
                        event.async ? sink.logEventAsync() : sink.logEvent()
                    }
                })
```

### PII 数据隔离机制

```typescript
// 在发送到 Datadog 前剥离 _PROTO_* 字段
void trackDatadogEvent(eventName, stripProtoFields(metadataWithSampleRate))

// 1P 接收完整 payload
logEventTo1P(eventName, metadataWithSampleRate)
```

`_PROTO_*` 字段包含标记为 PII 的数据，应该只发送到受控访问的 1P 后端，而不应该发送到一般访问的 Datadog。

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `./datadog.js` | `trackDatadogEvent` - Datadog 事件跟踪 |
| `./firstPartyEventLogger.js` | `logEventTo1P`, `shouldSampleEvent` - 1P 事件日志 |
| `./growthbook.js` | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` - 功能门控 |
| `./index.js` | `attachAnalyticsSink`, `stripProtoFields` - Sink 接口和 PII 剥离 |
| `./sinkKillswitch.js` | `isSinkKilled` - Sink killswitch |

### 调用方

| 函数 | 调用方 |
|------|--------|
| `initializeAnalyticsSink` | `src/utils/sinks.ts`, `src/utils/claudeInChrome/mcpServer.ts`, `src/utils/computerUse/mcpServer.ts` |
| `initializeAnalyticsGates` | `src/main.tsx` (在 `setupBackend()` 中) |

### 依赖关系图

```
sink.ts
    ├──> datadog.ts (trackDatadogEvent)
    │       └──> metadata.ts (getEventMetadata)
    │
    ├──> firstPartyEventLogger.ts (logEventTo1P, shouldSampleEvent)
    │       ├──> metadata.ts (getEventMetadata)
    │       ├──> sinkKillswitch.ts (isSinkKilled)
    │       └──> growthbook.ts (getDynamicConfig_CACHED_MAY_BE_STALE)
    │
    ├──> growthbook.ts (checkStatsigFeatureGate_CACHED_MAY_BE_STALE)
    │
    ├──> index.ts (attachAnalyticsSink, stripProtoFields)
    │       └── (无依赖，避免循环)
    │
    └──> sinkKillswitch.ts (isSinkKilled)
            └──> growthbook.ts (getDynamicConfig_CACHED_MAY_BE_STALE)
```

---

## 依赖与外部交互

### 外部系统交互

1. **GrowthBook**：
   - 检查 `tengu_log_datadog_events` 功能门控
   - 使用缓存值避免阻塞启动

2. **Datadog**：
   - 通过 `trackDatadogEvent` 发送事件
   - 仅发送脱敏后的事件（剥离 `_PROTO_*`）

3. **1P 事件日志 API**：
   - 通过 `logEventTo1P` 发送事件
   - 接收完整事件数据（包括 PII 标记字段）

---

## 风险、边界与改进建议

### 风险点

1. **启动顺序依赖**
   - `initializeAnalyticsGates` 必须在 `initializeAnalyticsSink` 之前调用
   - 如果顺序错误，早期事件可能使用错误的门控值

2. **事件丢失风险**
   - 如果 sink 在附加前崩溃，队列中的事件会丢失
   - 队列是内存中的，不会持久化到磁盘

3. **采样逻辑重复**
   - `shouldSampleEvent` 在 `firstPartyEventLogger.ts` 中实现
   - 采样决策在 sink 层做出，但采样配置在另一个模块

4. **异步排出竞争条件**
   - 队列使用 `queueMicrotask` 异步排出
   - 如果在排出过程中有新事件到达，可能产生竞态条件

### 边界情况

1. **Sink 未初始化**：事件被排队，直到 sink 附加
2. **Datadog 门控未初始化**：回退到缓存值
3. **采样率为 0**：事件被完全丢弃
4. **Killswitch 激活**：特定 sink 被禁用，但其他 sink 继续工作

### 改进建议

1. **启动顺序保护**
   ```typescript
   // 添加运行时检查确保正确顺序
   let gatesInitialized = false
   export function initializeAnalyticsGates(): void {
       gatesInitialized = true
       // ...
   }
   export function initializeAnalyticsSink(): void {
       if (!gatesInitialized) {
           console.warn('initializeAnalyticsGates should be called before initializeAnalyticsSink')
       }
       // ...
   }
   ```

2. **队列持久化**
   - 考虑将队列中的事件持久化到磁盘，防止启动期间崩溃导致事件丢失
   - 或者使用预分配固定大小的循环缓冲区

3. **同步排出选项**
   - 添加选项允许同步排出队列，用于测试或需要强一致性的场景

4. **指标和监控**
   - 添加指标跟踪队列大小、排出时间、事件丢失数量
   - 当前只有 ant 用户能看到 `analytics_sink_attached` 事件

5. **错误处理增强**
   ```typescript
   // 当前实现中，如果 sink.logEvent 抛出异常，后续事件会丢失
   queueMicrotask(() => {
       for (const event of queuedEvents) {
           try {
               if (event.async) {
                   void sink!.logEventAsync(event.eventName, event.metadata)
               } else {
                   sink!.logEvent(event.eventName, event.metadata)
               }
           } catch (e) {
               // 应该记录错误但继续处理其他事件
               logError(e)
           }
       }
   })
   ```

6. **测试覆盖**
   - 当前没有专门的测试文件
   - 建议添加单元测试验证：
     - 幂等初始化
     - 队列排出逻辑
     - PII 剥离
     - 采样集成
