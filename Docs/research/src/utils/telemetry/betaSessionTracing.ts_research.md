# Beta Session Tracing 研究文档

## 场景与职责

`betaSessionTracing.ts` 是 Claude Code 的 Beta 级详细追踪模块，用于在调试和诊断场景下收集详细的会话追踪数据。该模块主要服务于以下场景：

1. **内部调试 (Ant 用户)**：Anthropic 内部员工使用，可获取完整的系统提示词、模型输出和思考输出
2. **外部用户调试**：SDK/Headless 模式或经 GrowthBook 门控允许的组织可启用详细追踪
3. **性能分析**：追踪 LLM 请求、工具执行、Hook 调用的完整生命周期

### 可见性规则

| 内容类型 | 外部用户 | Ant 用户 |
|---------|---------|---------|
| System prompts | ✅ | ✅ |
| Model output | ✅ | ✅ |
| Thinking output | ❌ | ✅ |
| Tools | ✅ | ✅ |
| new_context | ✅ | ✅ |

## 功能点目的

### 1. 哈希去重机制
- **目的**：系统提示词和工具模式在会话期间很少变化，通过哈希去重避免重复传输大内容
- **实现**：使用 `seenHashes` Set 跟踪已记录的内容哈希，每个唯一哈希只记录一次完整内容

### 2. 增量上下文追踪
- **目的**：调试时只查看每次交互新增的信息，而非完整的对话历史（可能非常大）
- **实现**：使用 `lastReportedMessageHash` Map 按 querySource（agent）跟踪最后报告的消息，计算并发送增量内容

### 3. 内容截断
- **目的**：遵守 Honeycomb 64KB 限制，避免数据被丢弃
- **实现**：`MAX_CONTENT_SIZE = 60KB`，超出时截断并标记 `truncated: true`

### 4. 系统提醒分离
- **目的**：将用户内容与系统提醒（如恶意软件警告）分开显示，便于调试
- **实现**：使用正则表达式 `<system-reminder>` 检测并提取系统提醒内容

## 具体技术实现

### 关键数据结构

```typescript
// 全局状态 - 会话级去重
const seenHashes = new Set<string>()
const lastReportedMessageHash = new Map<string, string>()

// LLM 请求新上下文配置
interface LLMRequestNewContext {
  systemPrompt?: string      // 系统提示词
  querySource?: string       // 查询来源（如 'repl_main_thread', 'agent:builtin'）
  tools?: string            // 工具模式 JSON
}

// 格式化后的消息结构
interface FormattedMessages {
  contextParts: string[]     // 常规用户内容
  systemReminders: string[]  // 系统提醒内容
}
```

### 关键流程

#### 1. Beta 追踪启用检查 (`isBetaTracingEnabled`)

```
检查流程：
1. 基础检查：ENABLE_BETA_TRACING_DETAILED=1 AND BETA_TRACING_ENDPOINT 存在
2. 外部用户检查：
   - SDK/Headless 模式（getIsNonInteractiveSession()）
   - 或组织在 GrowthBook 门控 tengu_trace_lantern 中被允许
3. Ant 用户：直接启用
```

#### 2. 系统提示词记录 (`addBetaLLMRequestAttributes`)

```
流程：
1. 生成系统提示词哈希：sp_<sha256前12位>
2. 设置 span 属性：hash, preview(前500字符), length
3. 如果哈希未记录过：
   - 截断内容（如需要）
   - 发送 OTel 事件 'system_prompt'
   - 标记哈希为已记录
```

#### 3. 工具模式记录

```
流程：
1. 解析 tools JSON 数组
2. 为每个工具生成哈希和名称/哈希对
3. 设置 span 属性：tools（名称/哈希数组）, tools_count
4. 对每个未记录过的工具：
   - 截断工具定义（如需要）
   - 发送 OTel 事件 'tool'
   - 标记工具哈希为已记录
```

#### 4. 增量上下文计算

```
流程：
1. 获取 querySource 对应的 lastHash
2. 在 messagesForAPI 中查找 lastHash 位置
3. 从 lastHash 之后开始切片，获取新消息
4. 过滤出 UserMessage（排除 AssistantMessage）
5. 格式化消息，分离系统提醒
6. 设置 span 属性：new_context, system_reminders
7. 更新 lastReportedMessageHash 为最新消息哈希
```

### 哈希生成算法

```typescript
// 短哈希（SHA-256 前12位十六进制）
function shortHash(content: string): string {
  return createHash('sha256').update(content).digest('hex').slice(0, 12)
}

// 系统提示词哈希前缀
function hashSystemPrompt(systemPrompt: string): string {
  return `sp_${shortHash(systemPrompt)}`
}

// 消息哈希前缀
function hashMessage(message: APIMessage): string {
  const content = jsonStringify(message.message.content)
  return `msg_${shortHash(content)}`
}
```

### 系统提醒检测

```typescript
const SYSTEM_REMINDER_REGEX =
  /^<system-reminder>\n?([\s\S]*?)\n?<\/system-reminder>$/

function extractSystemReminderContent(text: string): string | null {
  const match = text.trim().match(SYSTEM_REMINDER_REGEX)
  return match && match[1] ? match[1].trim() : null
}
```

## 关键代码路径与文件引用

### 导出函数

| 函数名 | 用途 | 调用位置 |
|-------|------|---------|
| `isBetaTracingEnabled()` | 检查 Beta 追踪是否启用 | sessionTracing.ts, instrumentation.ts |
| `clearBetaTracingState()` | 压缩后清除追踪状态 | postCompactCleanup.ts |
| `truncateContent()` | 内容截断工具 | sessionTracing.ts（内部使用） |
| `addBetaInteractionAttributes()` | 添加交互 span 属性 | sessionTracing.ts:startInteractionSpan |
| `addBetaLLMRequestAttributes()` | 添加 LLM 请求属性 | sessionTracing.ts:startLLMRequestSpan |
| `addBetaLLMResponseAttributes()` | 添加 LLM 响应属性 | sessionTracing.ts:endLLMRequestSpan |
| `addBetaToolInputAttributes()` | 添加工具输入属性 | sessionTracing.ts:startToolSpan |
| `addBetaToolResultAttributes()` | 添加工具结果属性 | sessionTracing.ts:endToolSpan |

### 依赖文件

- `src/utils/telemetry/events.ts` - `logOTelEvent()` 用于发送系统提示词和工具事件
- `src/services/analytics/metadata.ts` - `sanitizeToolNameForAnalytics()` 工具名称清理
- `src/bootstrap/state.ts` - `getIsNonInteractiveSession()` 检查非交互模式
- `src/services/analytics/growthbook.ts` - `getFeatureValue_CACHED_MAY_BE_STALE()` 功能门控
- `src/types/message.ts` - `UserMessage`, `AssistantMessage` 类型定义

### 被调用方

- `src/utils/telemetry/sessionTracing.ts` - 主要调用方，集成到 OTel span 生命周期
- `src/services/compact/postCompactCleanup.ts` - 压缩后清除状态

## 依赖与外部交互

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `ENABLE_BETA_TRACING_DETAILED` | 启用 Beta 详细追踪（需为 truthy 值） |
| `BETA_TRACING_ENDPOINT` | Beta 追踪 OTLP 端点 URL |
| `USER_TYPE` | 用户类型（'ant' 为内部员工） |
| `OTEL_LOG_USER_PROMPTS` | 是否记录用户提示词（控制是否显示 `<REDACTED>`） |

### GrowthBook 门控

- `tengu_trace_lantern`：控制外部用户是否可访问详细追踪

### 外部服务

- OTLP HTTP 端点（通过 `BETA_TRACING_ENDPOINT` 配置）
- Honeycomb（通过 OTLP 接收追踪数据，有 64KB 限制）

## 风险、边界与改进建议

### 风险

1. **数据隐私风险**
   - Thinking output 仅对 Ant 用户可见，但代码中需要确保不会泄露给外部用户
   - 系统提示词和工具模式可能包含敏感信息

2. **内存增长**
   - `seenHashes` 和 `lastReportedMessageHash` 会持续增长，虽然去重但 Set/Map 本身会增长
   - 长会话中哈希数量可能变得很大

3. **Honeycomb 限制**
   - 60KB 截断限制可能导致重要信息丢失
   - 截断后无法恢复原始内容

4. **并发问题**
   - 全局状态 `seenHashes` 和 `lastReportedMessageHash` 在并行请求时可能产生竞态条件

### 边界情况

1. **压缩后状态**
   - `clearBetaTracingState()` 在压缩后被调用，重置所有追踪状态
   - 这可能导致压缩后首次请求发送重复内容（因为哈希被清除）

2. **消息哈希查找失败**
   - 如果 `lastHash` 在消息数组中找不到，会回退到发送所有消息（`startIndex = 0`）

3. **工具解析失败**
   - 工具 JSON 解析失败时会静默捕获错误，仅设置 `tools_parse_error: true`

### 改进建议

1. **内存优化**
   - 为 `seenHashes` 添加 LRU 淘汰机制，限制最大条目数
   - 定期清理长时间未使用的 querySource 条目

2. **可观测性**
   - 添加指标统计去重命中率（`seenHashes` 命中 vs 未命中）
   - 记录截断事件以便分析内容大小分布

3. **并发安全**
   - 考虑使用 AsyncLocalStorage 替代全局 Map 来隔离不同请求的状态

4. **配置灵活性**
   - 允许通过环境变量配置 `MAX_CONTENT_SIZE`
   - 支持按内容类型配置不同的截断限制

5. **错误处理**
   - 工具解析失败时记录更多上下文信息以便调试
   - 添加警告日志当消息哈希查找失败时
