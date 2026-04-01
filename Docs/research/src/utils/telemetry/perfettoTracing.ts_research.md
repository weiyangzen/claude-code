# Perfetto Tracing 研究文档

## 场景与职责

`perfettoTracing.ts` 是 Claude Code 的高性能追踪模块，生成 Chrome Trace Event 格式的追踪文件，可在 [ui.perfetto.dev](https://ui.perfetto.dev) 或 Chrome 的 `chrome://tracing` 中可视化。该模块主要服务于以下场景：

1. **性能分析**：分析 API 请求延迟、TTFT（Time To First Token）、TTLT（Time To Last Token）
2. **工具执行追踪**：监控工具调用耗时和结果
3. **Agent 层级可视化**：展示主线程与子 Agent 的层级关系
4. **用户交互分析**：追踪用户输入等待时间

### 核心特点

- **Ant 专用**：通过 `feature('PERFETTO_TRACING')` 控制，外部构建中完全剔除
- **Chrome 标准格式**：兼容 Perfetto UI 和 Chrome Tracing
- **内存受限**：最大 100,000 事件限制，自动淘汰旧事件
- **Stale Span 清理**：30 分钟 TTL，防止内存泄漏
- **多写入策略**：支持退出时写入、定期写入、同步回退写入

## 功能点目的

### 1. Agent 层级追踪
- **目的**：可视化主线程与子 Agent 的调用关系
- **实现**：使用 PID（Process ID）和 TID（Thread ID）区分不同 Agent

### 2. API 请求性能指标
- **目的**：分析模型调用性能（TTFT、TTLT、Token 处理速度）
- **实现**：记录 promptTokens、outputTokens、cacheReadTokens、ITPS、OTPS

### 3. 内存管理
- **目的**：防止长时间运行会话（如 Cron 驱动）内存无限增长
- **实现**：
  - 100,000 事件上限，超出时淘汰最旧的一半
  - 30 分钟 stale span TTL，自动清理未结束 span

### 4. 可靠写入
- **目的**：确保追踪数据不丢失，即使在异常退出时
- **实现**：
  - 正常退出：`beforeExit` 异步写入
  - 异常退出：`exit` 同步写入回退
  - 定期写入：可配置间隔（`CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S`）

## 具体技术实现

### 关键数据结构

```typescript
// Chrome Trace Event 格式
export type TraceEvent = {
  name: string           // 事件名称
  cat: string            // 类别（逗号分隔）
  ph: TraceEventPhase    // 事件阶段
  ts: number             // 时间戳（微秒，相对追踪开始）
  pid: number            // 进程 ID（Agent 标识）
  tid: number            // 线程 ID（Agent 名称哈希）
  dur?: number           // 持续时间（微秒，X 事件）
  args?: Record<string, unknown>  // 附加参数
  id?: string            // 异步事件 ID
  scope?: string         // 异步事件作用域
}

// 事件阶段类型
type TraceEventPhase =
  | 'B'   // Begin duration event
  | 'E'   // End duration event
  | 'X'   // Complete event (with duration)
  | 'i'   // Instant event
  | 'C'   // Counter event
  | 'b'   // Async begin
  | 'n'   // Async instant
  | 'e'   // Async end
  | 'M'   // Metadata event

// Agent 信息
type AgentInfo = {
  agentId: string        // Agent 唯一 ID
  agentName: string      // Agent 显示名称
  parentAgentId?: string // 父 Agent ID
  processId: number      // 分配的 PID
  threadId: number       // 分配的 TID
}

// 待处理 Span
type PendingSpan = {
  name: string
  category: string
  startTime: number      // 微秒时间戳
  agentInfo: AgentInfo
  args: Record<string, unknown>
}

// 全局状态
let isEnabled = false
let tracePath: string | null = null
const metadataEvents: TraceEvent[] = []     // 元数据事件（不过期）
const events: TraceEvent[] = []             // 普通事件（有上限）
const MAX_EVENTS = 100_000                  // 事件上限
const pendingSpans = new Map<string, PendingSpan>()
const agentRegistry = new Map<string, AgentInfo>()
```

### 关键流程

#### 1. 初始化流程 (`initializePerfettoTracing`)

```
initializePerfettoTracing():
1. 检查环境变量 CLAUDE_CODE_PERFETTO_TRACE
2. 使用 feature('PERFETTO_TRACING') 进行死代码消除检查
3. 确定追踪文件路径：
   - 如果值为 truthy：~/.claude/traces/trace-{sessionId}.json
   - 否则：使用提供的自定义路径
4. 启动定期写入（如配置了 CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S）
5. 启动 stale span 清理定时器（60s 间隔）
6. 注册清理回调：
   - registerCleanup: 异步写入
   - beforeExit: 异步写入
   - exit: 同步写入（最终回退）
7. 为主 Agent 生成元数据事件
```

#### 2. API 请求 Span 生命周期

```
startLLMRequestPerfettoSpan(args):
1. 生成 span ID
2. 获取当前 Agent 信息
3. 创建 PendingSpan 记录开始时间和参数
4. 发出 'B'（Begin）事件

endLLMRequestPerfettoSpan(spanId, metadata):
1. 查找 PendingSpan
2. 计算派生指标：
   - ITPS = promptTokens / (ttftMs / 1000)  [Input Tokens Per Second]
   - OTPS = outputTokens / (samplingMs / 1000)  [Output Tokens Per Second]
   - cacheHitRate = cacheReadTokens / promptTokens
3. 如果有 requestSetupMs：
   - 发出 'Request Setup' 子 span
   - 如果有重试，发出每个重试尝试的子 span
4. 如果 ttftMs 存在：
   - 发出 'First Token' 子 span（B/E 对）
   - 发出 'Sampling' 子 span（B/E 对）
5. 发出 API Call 的 'E'（End）事件
6. 从 pendingSpans 删除
```

#### 3. 事件淘汰机制

```
evictOldestEvents():
1. 检查 events.length >= MAX_EVENTS
2. 删除前一半事件：events.splice(0, MAX_EVENTS / 2)
3. 插入标记事件：
   {
     name: 'trace_truncated',
     cat: '__metadata',
     ph: 'i',
     ts: 被删除最后一个事件的时间戳,
     args: { dropped_events: 删除数量 }
   }
4. 记录调试日志
```

#### 4. Stale Span 清理

```
evictStaleSpans():
1. 计算截止时间：now - STALE_SPAN_TTL_MS (30分钟)
2. 遍历 pendingSpans：
   - 如果 span.startTime < 截止时间：
     * 发出 'E' 事件，标记 evicted: true
     * 记录实际持续时间
     * 从 pendingSpans 删除
```

#### 5. 追踪文件写入

```
writePerfettoTrace() (异步):
1. 检查 isEnabled, tracePath, traceWritten
2. 停止定时器
3. 关闭所有未结束 span（closeOpenSpans）
4. 构建追踪文档：
   {
     traceEvents: [...metadataEvents, ...events],
     metadata: {
       session_id,
       trace_start_time,
       agent_count,
       total_event_count
     }
   }
5. 异步写入文件
6. 标记 traceWritten = true

writePerfettoTraceSync() (同步):
- 同上，但使用 mkdirSync/writeFileSync
- 仅在 process.on('exit') 时调用
```

### Agent 注册与管理

```typescript
// 注册新 Agent（子 Agent 创建时调用）
registerAgent(agentId, agentName, parentAgentId):
1. 创建 AgentInfo
2. 分配 PID（递增计数器）
3. 计算 TID（agentName 的哈希）
4. 添加到 agentRegistry
5. 发出元数据事件（process_name, thread_name, parent_agent）

// 注销 Agent（Agent 完成时调用）
unregisterAgent(agentId):
1. 从 agentRegistry 删除
2. 从 agentIdToProcessId 删除
```

## 关键代码路径与文件引用

### 导出函数

| 函数名 | 用途 | 调用位置 |
|-------|------|---------|
| `initializePerfettoTracing()` | 初始化追踪 | instrumentation.ts |
| `isPerfettoTracingEnabled()` | 检查是否启用 | sessionTracing.ts |
| `registerAgent()` | 注册子 Agent | spawnInProcess.ts, inProcessRunner.ts |
| `unregisterAgent()` | 注销 Agent | Agent 完成时 |
| `startLLMRequestPerfettoSpan()` | 开始 API 请求追踪 | sessionTracing.ts:startLLMRequestSpan |
| `endLLMRequestPerfettoSpan()` | 结束 API 请求追踪 | sessionTracing.ts:endLLMRequestSpan |
| `startToolPerfettoSpan()` | 开始工具追踪 | sessionTracing.ts:startToolSpan |
| `endToolPerfettoSpan()` | 结束工具追踪 | sessionTracing.ts:endToolSpan |
| `startUserInputPerfettoSpan()` | 开始用户输入追踪 | sessionTracing.ts:startToolBlockedOnUserSpan |
| `endUserInputPerfettoSpan()` | 结束用户输入追踪 | sessionTracing.ts:endToolBlockedOnUserSpan |
| `startInteractionPerfettoSpan()` | 开始交互追踪 | sessionTracing.ts:startInteractionSpan |
| `endInteractionPerfettoSpan()` | 结束交互追踪 | sessionTracing.ts:endInteractionSpan |
| `emitPerfettoInstant()` | 发送即时事件 | 工具执行关键点 |
| `emitPerfettoCounter()` | 发送计数器事件 | 资源监控 |

### 依赖文件

| 文件 | 用途 |
|-----|------|
| `src/bootstrap/state.js` | `getSessionId()` 获取会话 ID |
| `src/utils/cleanupRegistry.js` | `registerCleanup()` 注册退出清理 |
| `src/utils/debug.js` | `logForDebugging()` 调试日志 |
| `src/utils/envUtils.js` | `getClaudeConfigHomeDir()`, `isEnvDefinedFalsy()`, `isEnvTruthy()` |
| `src/utils/errors.js` | `errorMessage()` 错误处理 |
| `src/utils/hash.js` | `djb2Hash()` 字符串哈希 |
| `src/utils/slowOperations.js` | `jsonStringify()` JSON 序列化 |
| `src/utils/teammate.js` | `getAgentId()`, `getAgentName()`, `getParentSessionId()` |

### 被调用方

- `src/utils/telemetry/sessionTracing.ts` - 主要调用方，集成到 OTel span 生命周期
- `src/utils/swarm/spawnInProcess.ts` - 注册子 Agent
- `src/utils/swarm/inProcessRunner.ts` - 注册子 Agent

### 特殊依赖

- `bun:bundle` - `feature()` 函数用于死代码消除

## 依赖与外部交互

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `CLAUDE_CODE_PERFETTO_TRACE` | 启用追踪（1/true 或自定义路径） |
| `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S` | 定期写入间隔（秒） |

### 文件系统

- **默认路径**：`~/.claude/traces/trace-{sessionId}.json`
- **写入模式**：异步（正常）、同步（exit 回退）
- **格式**：Chrome Trace Event JSON

### 可视化工具

- [ui.perfetto.dev](https://ui.perfetto.dev) - Google 的性能分析平台
- Chrome `chrome://tracing` - 浏览器内置追踪查看器

## 风险、边界与改进建议

### 风险

1. **内存泄漏**
   - 长时间运行会话可能积累大量 pendingSpans
   - Stale span 清理 60s 间隔可能不够频繁
   - Agent 未正确注销时，registry 持续增长

2. **数据丢失**
   - 进程崩溃时可能丢失未写入的数据
   - 同步写入回退仅在 `exit` 事件触发
   - 某些信号可能导致写入完全跳过

3. **性能影响**
   - 每次事件创建对象，GC 压力
   - 文件写入 I/O 可能阻塞事件循环
   - JSON 序列化大数据量时耗时

4. **并发问题**
   - 全局状态（events, pendingSpans）无锁保护
   - 多 Agent 同时操作可能产生竞态条件

### 边界情况

1. **事件上限**
   - 达到 100,000 事件时淘汰一半
   - 可能丢失重要历史数据
   - 淘汰标记事件帮助识别数据缺口

2. **Stale Span**
   - 30 分钟 TTL 可能误伤长时间运行的操作
   - 被驱逐的 span 标记为 `evicted: true`

3. **文件写入失败**
   - 磁盘满、权限问题导致写入失败
   - 错误被捕获记录，但不重试

4. **Agent ID 冲突**
   - 如果 agentId 重复，registry 中旧条目被覆盖
   - 可能导致 PID 分配不一致

5. **时间戳精度**
   - 使用 `Date.now() - startTimeMs` 计算相对时间
   - 系统时间调整可能导致异常值

### 改进建议

1. **可靠性增强**
   - 实现写前日志（WAL），崩溃后恢复
   - 添加写入确认和重试机制
   - 支持多个追踪文件轮转

2. **性能优化**
   - 使用流式 JSON 序列化（非一次性 stringify）
   - 批量写入减少 I/O
   - 添加采样率配置（高负载时）

3. **内存优化**
   - 配置化事件上限
   - 更频繁的 stale span 检查（可配置）
   - 使用环形缓冲区替代数组

4. **并发安全**
   - 添加适当的锁机制或使用 Atomics
   - 考虑使用 Worker 线程处理 I/O

5. **功能扩展**
   - 支持自定义事件属性
   - 添加内存/CPU 计数器事件
   - 支持追踪文件压缩

6. **可观测性**
   - 追踪系统自身的指标（事件率、写入延迟）
   - 内存使用监控
   - 数据丢失告警

7. **配置灵活性**
   - 支持运行时启用/禁用
   - 动态调整事件上限
   - 自定义淘汰策略

8. **调试支持**
   - 添加追踪文件验证工具
   - 支持实时流式输出（用于远程调试）
   - 追踪数据摘要统计
