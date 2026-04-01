# sessionTracing.ts 深度研究文档

## 场景与职责

`sessionTracing.ts` 是 Claude Code 的 OpenTelemetry (OTel) 分布式追踪核心模块，负责：

1. **用户交互追踪**：将每个用户请求-响应周期封装为 `interaction` span（根 span）
2. **LLM 请求追踪**：追踪每个 API 调用（模型请求、预热请求、子代理请求等）
3. **工具执行追踪**：追踪工具调用生命周期（权限检查、执行、结果）
4. **Hook 执行追踪**：追踪 PreToolUse/PostToolUse 等 hook 的执行
5. **内存管理**：通过 WeakRef 和定期清理防止 orphaned span 累积
6. **双轨追踪**：同时支持 OTel 标准追踪和 Perfetto 可视化追踪

该模块是 Claude Code 可观测性基础设施的核心，为性能分析、故障排查和用量分析提供数据基础。

## 功能点目的

### 1. 增强型遥测开关 (`isEnhancedTelemetryEnabled`)
- **目的**：控制标准 OTel 追踪功能的启用
- **优先级**：环境变量覆盖 > ant 构建标记 > GrowthBook 特性开关
- **触发条件**：`feature('ENHANCED_TELEMETRY_BETA')` 为 true 且满足后续条件

### 2. Beta 详细追踪 (`isAnyTracingEnabled`)
- **目的**：支持更详细的调试追踪（系统提示词、工具输入输出等）
- **独立配置**：通过 `ENABLE_BETA_TRACING_DETAILED` 和 `BETA_TRACING_ENDPOINT` 启用
- **权限控制**：外部用户仅在 SDK/headless 模式或组织白名单中启用

### 3. Span 类型体系
| Span 类型 | 用途 | 存储方式 |
|-----------|------|----------|
| `interaction` | 用户交互周期 | AsyncLocalStorage |
| `llm_request` | LLM API 调用 | strongSpans Map |
| `tool` | 工具调用整体 | AsyncLocalStorage |
| `tool.blocked_on_user` | 等待用户权限 | strongSpans Map |
| `tool.execution` | 工具实际执行 | strongSpans Map |
| `hook` | Hook 执行 (Beta) | strongSpans Map |

### 4. 内存安全机制
- **WeakRef 策略**：存储在 ALS 中的 span 使用 WeakRef，允许 GC 回收
- **强引用兜底**：非 ALS span（LLM、blocked、execution、hook）使用 strongSpans 防止提前回收
- **TTL 清理**：30 分钟过期 span 自动清理（`SPAN_TTL_MS`）
- **定期清理器**：每分钟运行一次（`ensureCleanupInterval`）

### 5. Perfetto 集成
- **并行追踪**：无论 OTel 是否启用，Perfetto 追踪独立运行
- **环境控制**：通过 `CLAUDE_CODE_PERFETTO_TRACE` 启用
- **Ant 专用**：Perfetto 功能通过 `feature('PERFETTO_TRACING')` 在外部构建中剔除

## 具体技术实现

### 关键数据结构

```typescript
// Span 上下文 - 存储在 ALS 或 Map 中
interface SpanContext {
  span: Span                    // OTel Span 对象
  startTime: number            // 开始时间戳
  attributes: Record<string, string | number | boolean>
  ended?: boolean              // 是否已结束
  perfettoSpanId?: string      // 关联的 Perfetto span ID
}

// ALS 存储层级
const interactionContext = new AsyncLocalStorage<SpanContext | undefined>()
const toolContext = new AsyncLocalStorage<SpanContext | undefined>()

// Span 存储
const activeSpans = new Map<string, WeakRef<SpanContext>>()  // 主存储
const strongSpans = new Map<string, SpanContext>()            // 强引用兜底
```

### 关键流程

#### 1. 交互 Span 生命周期
```
startInteractionSpan(userPrompt)
  ├── ensureCleanupInterval()           // 启动清理器
  ├── startInteractionPerfettoSpan()    // 启动 Perfetto span
  ├── 创建 OTel span (如果启用)
  │     ├── tracer.startSpan('claude_code.interaction')
  │     ├── addBetaInteractionAttributes()  // Beta 属性
  │     └── 设置 user_prompt, interaction.sequence
  └── interactionContext.enterWith()    // 设置 ALS 上下文

endInteractionSpan()
  ├── 从 ALS 获取 spanContext
  ├── endInteractionPerfettoSpan()      // 结束 Perfetto span
  ├── 设置 duration_ms 属性
  ├── span.end()
  └── interactionContext.enterWith(undefined)  // 清除 ALS
```

**调用位置**：
- `startInteractionSpan`: `src/screens/REPL.tsx` - 用户提交输入时
- `endInteractionSpan`: `src/screens/REPL.tsx` - 响应完成或中断时；`src/utils/telemetry/instrumentation.ts` - 进程退出时

#### 2. LLM 请求 Span 生命周期
```
startLLMRequestSpan(model, newContext, messagesForAPI, fastMode)
  ├── startLLMRequestPerfettoSpan()     // 启动 Perfetto span
  ├── 创建 OTel span (如果启用)
  │     ├── tracer.startSpan('claude_code.llm_request')
  │     ├── 设置 model, speed, llm_request.context
  │     ├── addBetaLLMRequestAttributes()  // 系统提示词、工具、new_context
  │     └── 继承 interaction span 作为 parent
  └── 存储到 activeSpans + strongSpans

endLLMRequestSpan(span?, metadata)
  ├── 通过 span ID 或查找获取 spanContext
  ├── endLLMRequestPerfettoSpan()       // 结束 Perfetto span（含丰富元数据）
  ├── 设置 duration_ms, input_tokens, output_tokens, cache_* 等
  ├── addBetaLLMResponseAttributes()    // model_output, thinking_output
  └── 从存储中删除
```

**调用位置**：
- `src/services/api/claude.ts` - 每次 API 调用前后

**关键设计**：支持并行请求通过传递 `span` 参数精确匹配，避免回退到"最近 span"导致的错误关联。

#### 3. 工具 Span 生命周期
```
startToolSpan(toolName, toolAttributes, toolInput)
  ├── startToolPerfettoSpan()
  ├── 创建 OTel span
  │     ├── tracer.startSpan('claude_code.tool')
  │     ├── addBetaToolInputAttributes()  // 工具输入 (Beta)
  │     └── 继承 interaction span 作为 parent
  └── toolContext.enterWith()

startToolBlockedOnUserSpan()
  └── 创建等待用户决策的子 span

endToolBlockedOnUserSpan(decision, source)
  └── 记录用户决策（允许/拒绝）和来源

startToolExecutionSpan()
  └── 创建实际执行子 span

endToolExecutionSpan({success, error})
  └── 记录执行结果

endToolSpan(toolResult, resultTokens)
  ├── endToolPerfettoSpan()
  ├── addBetaToolResultAttributes()     // 工具结果 (Beta)
  └── 设置 result_tokens, duration_ms
```

**调用位置**：
- `src/services/tools/toolExecution.ts` - 工具执行全流程

#### 4. Hook Span 生命周期 (Beta)
```
startHookSpan(hookEvent, hookName, numHooks, hookDefinitions)
  └── 仅在 isBetaTracingEnabled() 时创建 span

endHookSpan(span, {numSuccess, numBlocking, numNonBlockingError, numCancelled})
  └── 记录 hook 执行统计
```

### 属性与元数据

#### 标准属性（所有 span）
- `user.id`, `session.id` - 用户/会话标识
- `organization.id`, `user.email` - OAuth 信息
- `span.type` - span 类型
- `interaction.sequence` - 交互序号

#### Interaction Span 属性
- `user_prompt` / `user_prompt_length` - 用户输入（可脱敏）
- `interaction.duration_ms` - 交互耗时

#### LLM Request Span 属性
- `model` - 模型名称
- `speed` - fast/normal
- `llm_request.context` - interaction/standalone
- `query_source` - 请求来源（agent 标识）
- `input_tokens`, `output_tokens` - Token 用量
- `cache_read_tokens`, `cache_creation_tokens` - 缓存统计
- `ttft_ms` - Time To First Token
- `success`, `error`, `status_code` - 结果状态
- `attempt` - 重试次数
- `response.has_tool_call` - 是否包含工具调用

#### Tool Span 属性
- `tool_name` - 工具名称
- `duration_ms` - 执行耗时
- `result_tokens` - 结果 Token 数
- `success`, `error` - 执行结果

#### Beta 专属属性
- `new_context` - 增量上下文（用户消息、工具结果）
- `system_prompt_hash`, `system_prompt_preview` - 系统提示词
- `tools` - 工具 schema 哈希列表
- `tool_input` - 工具输入内容
- `response.model_output`, `response.thinking_output` - 模型输出

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `src/utils/telemetry/sessionTracing.ts` | 本文件，Span 生命周期管理 |
| `src/utils/telemetry/betaSessionTracing.ts` | Beta 追踪属性计算（系统提示词、new_context） |
| `src/utils/telemetry/perfettoTracing.ts` | Perfetto 可视化追踪实现 |
| `src/utils/telemetry/instrumentation.ts` | OTel SDK 初始化、导出器配置 |
| `src/utils/telemetry/events.ts` | OTel 事件日志记录 |
| `src/utils/telemetryAttributes.ts` | 通用遥测属性获取 |

### 调用方文件
| 文件 | 调用内容 |
|------|----------|
| `src/screens/REPL.tsx` | `startInteractionSpan`, `endInteractionSpan` |
| `src/services/api/claude.ts` | `startLLMRequestSpan`, `endLLMRequestSpan` |
| `src/services/tools/toolExecution.ts` | `startToolSpan`, `endToolSpan`, `startToolBlockedOnUserSpan`, `endToolBlockedOnUserSpan`, `startToolExecutionSpan`, `endToolExecutionSpan` |
| `src/utils/processUserInput/processTextPrompt.ts` | `executeInSpan` |

### 被调用方/依赖
| 文件 | 用途 |
|------|------|
| `@opentelemetry/api` | OTel API（trace, context, Span） |
| `async_hooks` | AsyncLocalStorage 用于上下文传播 |
| `src/services/analytics/growthbook.ts` | 特性开关读取 |
| `src/utils/telemetryAttributes.ts` | 基础遥测属性 |
| `src/utils/envUtils.ts` | 环境变量工具 |

## 依赖与外部交互

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` / `ENABLE_ENHANCED_TELEMETRY_BETA` | 增强遥测开关 |
| `OTEL_TRACES_EXPORTER` | Traces 导出器（console, otlp） |
| `OTEL_LOG_USER_PROMPTS` | 是否记录用户提示词原文 |
| `OTEL_LOG_TOOL_CONTENT` | 是否记录工具内容 |
| `ENABLE_BETA_TRACING_DETAILED` | Beta 详细追踪开关 |
| `BETA_TRACING_ENDPOINT` | Beta 追踪端点 |
| `CLAUDE_CODE_PERFETTO_TRACE` | Perfetto 追踪开关/路径 |
| `USER_TYPE` | 用户类型（ant/external） |

### OpenTelemetry 集成
- **Tracer**: `com.anthropic.claude_code.tracing`, version `1.0.0`
- **Span 名称**: `claude_code.interaction`, `claude_code.llm_request`, `claude_code.tool`, etc.
- **导出器**: OTLP (HTTP/gRPC/Protobuf)、Console
- **处理器**: BatchSpanProcessor

### Perfetto 集成
- 通过 `perfettoTracing.ts` 生成 Chrome Trace Event 格式
- 支持在 ui.perfetto.dev 可视化
- 包含 Agent 层级、API 调用、工具执行、用户等待时间

## 风险、边界与改进建议

### 风险点

1. **并行请求 Span 匹配风险**
   - `endLLMRequestSpan` 的 legacy fallback 通过查找"最近的 llm_request span"匹配
   - 当多个 LLM 请求并行时（如预热请求 + 主请求），可能导致响应关联到错误的 span
   - **缓解**：调用方应始终传递 `span` 参数（`claude.ts` 已实现）

2. **内存泄漏风险**
   - `strongSpans` 使用强引用，如果 `end*` 函数未被调用，span 无法被 GC
   - **缓解**：TTL 清理器（30 分钟）会强制结束并删除过期 span

3. **AsyncLocalStorage 上下文丢失**
   - 某些异步模式（如 setTimeout、非 Promise 回调）可能丢失 ALS 上下文
   - **缓解**：使用 `interactionContext.enterWith()` 而非 `run()`，上下文持续到显式清除

4. **敏感数据泄露风险**
   - Beta 追踪可能记录系统提示词、工具输入输出
   - **缓解**：
     - 外部用户默认脱敏（`<REDACTED>`）
     - `thinking_output` 仅对 ant 用户可见
     - 内容截断（60KB 限制）
     - 哈希去重（系统提示词、工具 schema 只记录一次）

### 边界条件

1. **Span 嵌套边界**
   - Interaction span 是根 span，包含所有其他 span
   - Tool span 可以包含 blocked_on_user 和 execution 子 span
   - LLM request span 是 interaction 的子 span

2. **启用状态边界**
   - Perfetto 追踪独立于 OTel，可单独启用
   - Beta 追踪需要同时满足 `ENABLE_BETA_TRACING_DETAILED=1` 和 `BETA_TRACING_ENDPOINT` 设置
   - 标准增强遥测需要 `feature('ENHANCED_TELEMETRY_BETA')` 为 true

3. **进程生命周期边界**
   - `beforeExit` 和 `exit` 事件处理器确保追踪数据在进程退出前写入
   - 同步写入作为最后兜底（`writePerfettoTraceSync`）

### 改进建议

1. **Span 匹配优化**
   - 考虑使用 OpenTelemetry 的上下文传播机制替代手动 span 存储
   - 为并行请求场景添加 span 栈机制

2. **性能优化**
   - `activeSpans` 遍历在大量 span 时可能成为瓶颈
   - 考虑按类型分桶存储（interactionSpans, llmSpans, toolSpans）

3. **可观测性增强**
   - 添加 span 计数器指标（active spans, orphaned spans）
   - 导出清理器运行统计

4. **错误处理强化**
   - 在 `executeInSpan` 中捕获异常时记录更多上下文
   - 添加 span 创建失败的降级处理

5. **配置统一**
   - 当前遥测开关分散在多个环境变量和特性标志中
   - 考虑统一的遥测配置接口
