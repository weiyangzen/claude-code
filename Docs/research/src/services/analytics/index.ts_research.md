# index.ts 研究文档

## 场景与职责

`index.ts` 是 Claude Code 分析服务的公共 API 入口，提供简洁的事件日志接口。该模块采用**延迟初始化**设计，在 Sink 附加前将事件排队，避免启动时的循环依赖问题。

核心职责：
- 提供类型安全的分析元数据标记类型
- 实现事件队列机制（Sink 附加前缓冲）
- 定义 Sink 接口契约
- 提供同步和异步事件日志 API

## 功能点目的

### 1. 类型安全标记 (`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`)

**目的**：防止敏感数据（代码片段、文件路径）意外进入分析系统。

**实现**：
```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```

**使用方式**：
```typescript
// 需要显式类型断言才能传入字符串
logEvent('tengu_event', {
  filePath: '/home/user/secret' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
})
```

**设计原理**：
- `never` 类型意味着不能直接赋值
- 强制开发者显式审查每个字符串值
- 编译时检查，零运行时开销

### 2. PII 标记类型 (`AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`)

**目的**：允许标记为 PII 的数据进入特权列，同时防止泄露到通用后端。

**使用方式**：
```typescript
// 标记为 PII，将路由到 _PROTO_* 字段
logEvent('tengu_event', {
  _PROTO_skill_name: skillName as AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED
})
```

**处理流程**：
1. 1P Exporter：提取 `_PROTO_*` 到 proto 顶层字段
2. Sink/Datadog：`stripProtoFields` 移除所有 `_PROTO_*` 键

### 3. Proto 字段剥离 (`stripProtoFields`)

**目的**：确保 `_PROTO_*` 数据不会进入非特权存储。

**实现**：
```typescript
export function stripProtoFields<V>(metadata: Record<string, V>): Record<string, V> {
  let result: Record<string, V> | undefined
  for (const key in metadata) {
    if (key.startsWith('_PROTO_')) {
      if (result === undefined) {
        result = { ...metadata }
      }
      delete result[key]
    }
  }
  return result ?? metadata
}
```

**性能优化**：
- 无 `_PROTO_` 键时返回原对象（无拷贝）
- 仅在需要时创建新对象

**调用点**：
- `sink.ts`: 发送到 Datadog 前
- `firstPartyEventLoggingExporter.ts`: 防御性剥离（已提取后）

### 4. 事件队列机制

**目的**：解决启动时的循环依赖问题。

**问题场景**：
```
main.tsx → logEvent → sink.ts → growthbook.ts → getGlobalConfig → 
getConfig → logEvent → ...（无限循环）
```

**解决方案**：
```typescript
// 无依赖的轻量级模块
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

export function logEvent(eventName: string, metadata: LogEventMetadata): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

**队列排空**：
```typescript
export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // 幂等
  sink = newSink
  
  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0
    
    queueMicrotask(() => {
      for (const event of queuedEvents) {
        // 发送到 sink
      }
    })
  }
}
```

**设计特点**：
- Sink 附加前事件不丢失
- 异步排空避免阻塞启动
- 幂等附加（多次调用安全）

### 5. Sink 接口契约

**目的**：定义分析后端的统一接口。

```typescript
export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (eventName: string, metadata: LogEventMetadata) => Promise<void>
}
```

**当前实现**：`sink.ts` 路由到 Datadog 和 1P

**扩展性**：未来可添加新的 Sink（如本地文件、自定义后端）

## 具体技术实现

### 元数据类型

```typescript
// 允许的元数据值类型（ intentionally no strings ）
type LogEventMetadata = { [key: string]: boolean | number | undefined }
```

**设计决策**：
- 禁止字符串：防止意外传入文件路径或代码
- 布尔/数字：足够表达大多数分析维度
- `undefined`：允许条件属性（`...(condition && { key: value })`）

### 队列数据结构

```typescript
type QueuedEvent = {
  eventName: string
  metadata: LogEventMetadata
  async: boolean  // 区分同步/异步调用
}
```

**异步标记用途**：
- 同步调用：`sink.logEvent()`
- 异步调用：`await sink.logEventAsync()`

### 核心函数流程

#### `logEvent(eventName, metadata)`

```
1. 检查 sink 是否存在
2. 不存在：加入队列（async: false）
3. 存在：调用 sink.logEvent()
```

#### `logEventAsync(eventName, metadata)`

```
1. 检查 sink 是否存在
2. 不存在：加入队列（async: true），返回 resolved Promise
3. 存在：await sink.logEventAsync()
```

#### `attachAnalyticsSink(newSink)`

```
1. 检查是否已附加（幂等）
2. 记录 sink 引用
3. 如果有队列事件：
   a. 复制队列
   b. 清空原队列
   c. 记录队列大小事件（ant 用户）
   d. queueMicrotask 异步排空
```

## 关键代码路径与文件引用

### 被调用方（业务代码）

**高频调用点**（grep 结果摘要）：

| 文件 | 事件示例 | 说明 |
|------|----------|------|
| `src/main.tsx` | `tengu_init`, `tengu_startup_*` | 启动事件 |
| `src/services/api/*.ts` | `tengu_api_*` | API 调用 |
| `src/tools/*/*.ts` | `tengu_tool_use_*` | 工具使用 |
| `src/hooks/*.ts` | 各种交互事件 | 用户交互 |
| `src/commands/*/*.ts` | 命令执行事件 | CLI 命令 |

**典型调用模式**：
```typescript
import { logEvent } from '../services/analytics/index.js'

// 简单事件
logEvent('tengu_event_name', { success: true })

// 带条件属性
logEvent('tengu_complex_event', {
  count: items.length,
  hasFlag: someCondition,
  ...(extraCondition && { extra: 1 })
})
```

### 依赖方

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/services/analytics/sink.ts` | `attachAnalyticsSink`, `stripProtoFields` | Sink 实现 |
| `src/services/analytics/firstPartyEventLoggingExporter.ts` | `stripProtoFields` | PII 剥离 |
| `src/utils/config.ts` | `logEvent` | 配置相关事件 |
| `src/utils/sinks.ts` | `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 类型标记 |

### 模块依赖图

```
index.ts (本模块)
  ├── 被调用：业务代码各处
  ├── 调用：sink.ts (通过 attachAnalyticsSink 注册)
  └── 无其他依赖（设计目标）

sink.ts
  ├── 调用：datadog.ts, firstPartyEventLogger.ts
  └── 依赖：growthbook.ts (gate 检查)
```

## 依赖与外部交互

### 设计原则：零依赖

**目标**：避免导入循环

**允许**：
- 类型导入（`type` 关键字）
- 纯类型定义

**禁止**：
- 运行时值导入
- 函数调用
- 副作用

### 运行时依赖

**无**：本模块不导入任何运行时模块。

### 类型依赖

**无显式导入**：所有类型内联定义或从其他模块导入类型（非值）。

## 风险、边界与改进建议

### 风险点

1. **队列无限增长**
   - 场景：Sink 长时间未附加（启动失败）
   - 风险：内存持续增长
   - 缓解：实际场景 Sink 很快附加
   - 建议：添加最大队列大小限制

2. **事件顺序**
   - 场景：Sink 附加前后的事件时间戳
   - 现状：队列事件在微任务中发送，可能晚于新事件
   - 风险：时序分析偏差
   - 建议：添加队列事件标记或时间戳

3. **异步调用丢失**
   - 场景：`logEventAsync` 在 Sink 附加前调用
   - 现状：返回 resolved Promise，调用方继续
   - 风险：调用方可能期望事件已发送
   - 缓解：实际场景中 Sink 很快附加

4. **类型绕过**
   - 场景：开发者使用 `as any` 绕过标记类型
   - 缓解：代码审查，ESLint 规则
   - 建议：添加 lint 规则禁止 `as any` 在 logEvent 调用中

### 边界情况

1. **重复附加 Sink**
   - 处理：幂等检查，忽略后续调用
   - 场景：`setup()` 和 `preAction` 都可能调用

2. **测试重置**
   - 提供：`_resetForTesting()`
   - 用途：测试间隔离状态
   - 警告：仅用于测试

3. **空元数据**
   - 允许：`logEvent('name', {})`
   - 处理：正常发送空对象

4. **undefined 值**
   - 允许：`{ key: undefined }`
   - 处理：JSON 序列化时省略

### 改进建议

1. **队列大小限制**
   ```typescript
   const MAX_QUEUE_SIZE = 1000
   if (eventQueue.length >= MAX_QUEUE_SIZE) {
     // 丢弃最旧的事件或记录警告
     eventQueue.shift()
   }
   ```

2. **队列事件标记**
   ```typescript
   // 添加标记便于分析时识别
   sink.logEvent(eventName, {
     ...metadata,
     _queued: true,
     _queuedAt: timestamp
   })
   ```

3. **批量 Sink 接口**
   ```typescript
   export type AnalyticsSink = {
     logEvent: (eventName: string, metadata: LogEventMetadata) => void
     logEventAsync: (eventName: string, metadata: LogEventMetadata) => Promise<void>
     logEvents?: (events: QueuedEvent[]) => void  // 批量接口
   }
   ```

4. **元数据验证**
   ```typescript
   // 开发环境验证元数据
   function validateMetadata(metadata: LogEventMetadata): void {
     if (process.env.NODE_ENV === 'development') {
       for (const [key, value] of Object.entries(metadata)) {
         if (typeof value === 'string') {
           throw new Error(`String value not allowed: ${key}`)
         }
       }
     }
   }
   ```

5. **事件 Schema 定义**
   ```typescript
   // 为常用事件定义类型
   export interface EventSchemas {
     'tengu_api_success': { status: number; durationMs: number }
     'tengu_tool_use_success': { toolName: string }
   }
   
   export function logEventTyped<K extends keyof EventSchemas>(
     eventName: K,
     metadata: EventSchemas[K]
   ): void
   ```

6. **性能监控**
   ```typescript
   // 导出队列统计
   export function getQueueStats(): {
     pendingCount: number
     totalQueued: number
     totalDrained: number
   }
   ```
