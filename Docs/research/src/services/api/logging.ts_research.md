# API Logging 服务研究文档

## 文件信息
- **路径**: `src/services/api/logging.ts`
- **大小**: 24,191 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

API Logging 服务是 Claude Code 的核心 **API 遥测和分析基础设施**，负责记录所有 Anthropic API 调用的生命周期事件。该服务处理以下核心场景：

1. **API 查询日志记录**: 在发送 API 请求前记录查询参数和上下文
2. **API 成功日志记录**: 记录成功的 API 响应，包括 token 使用量、成本、延迟等
3. **API 错误日志记录**: 记录失败的 API 调用，包括错误类型、重试次数、网关检测等
4. **网关检测**: 自动识别用户是否通过 AI 网关（LiteLLM、Helicone 等）访问 API
5. **Teleport 会话跟踪**: 记录 Teleport 会话的首消息成功/失败率

---

## 功能点目的

### 1. 查询日志 (`logAPIQuery`)
在发送 API 请求前记录查询上下文：
- 模型信息、消息长度、温度参数
- Beta 功能标志、权限模式、查询来源
- Thinking 类型、Effort 级别、Fast 模式状态
- 查询链跟踪（用于子 Agent 分析）

### 2. 成功日志 (`logAPISuccess` / `logAPISuccessAndDuration`)
记录成功的 API 响应，包含：
- **Token 使用**: input/output/cached/uncached tokens
- **成本分析**: costUSD、构建年龄（用于版本影响分析）
- **性能指标**: TTFT（首 token 时间）、总延迟、重试次数
- **内容分析**: 文本长度、thinking 长度、工具调用长度
- **会话上下文**: 是否非交互式、是否 post-compaction、Teleport 状态

### 3. 错误日志 (`logAPIError`)
记录失败的 API 调用：
- 错误分类（通过 `classifyAPIError`）
- 连接错误详情（SSL、DNS、超时等）
- 网关检测（从响应头识别）
- 重试上下文（尝试次数、client request ID）

### 4. 网关检测 (`detectGateway`)
识别用户是否通过第三方 AI 网关访问 API：
- **已知网关**: LiteLLM、Helicone、Portkey、Cloudflare AI Gateway、Kong、Braintrust、Databricks
- **检测方式**: 响应头前缀匹配 + 主机名后缀匹配

---

## 具体技术实现

### 关键数据结构

```typescript
// 全局缓存策略类型
type GlobalCacheStrategy = 'tool_based' | 'system_prompt' | 'none'

// 已知网关枚举
type KnownGateway = 
  | 'litellm' 
  | 'helicone' 
  | 'portkey' 
  | 'cloudflare-ai-gateway' 
  | 'kong' 
  | 'braintrust' 
  | 'databricks'

// 网关指纹配置
const GATEWAY_FINGERPRINTS: Partial<Record<KnownGateway, { prefixes: string[] }>> = {
  litellm: { prefixes: ['x-litellm-'] },
  helicone: { prefixes: ['helicone-'] },
  portkey: { prefixes: ['x-portkey-'] },
  'cloudflare-ai-gateway': { prefixes: ['cf-aig-'] },
  kong: { prefixes: ['x-kong-'] },
  braintrust: { prefixes: ['x-bt-'] },
}

// 主机名后缀匹配（用于无特征头的网关）
const GATEWAY_HOST_SUFFIXES: Partial<Record<KnownGateway, string[]>> = {
  databricks: ['.cloud.databricks.com', '.azuredatabricks.net', '.gcp.databricks.com']
}
```

### 关键流程

#### 网关检测流程
```
detectGateway({ headers, baseUrl })
├── 检查响应头
│   └── 遍历所有头名，匹配已知前缀
├── 检查 baseUrl 主机名
│   └── 匹配已知后缀（如 .cloud.databricks.com）
└── 返回识别的网关或 undefined
```

#### 成功日志记录流程
```
logAPISuccessAndDuration(params)
├── 计算 durationMs 和 durationMsIncludingRetries
├── 检测网关（从响应头）
├── 解析消息内容长度（文本/thinking/工具调用）
├── 调用 logAPISuccess（发送 analytics 事件）
├── 调用 logOTelEvent（发送 OTLP 事件）
├── 调用 endLLMRequestSpan（结束 tracing span）
└── Teleport 首消息跟踪（如适用）
```

### 内容长度解析

```typescript
// 从 AssistantMessage 提取内容统计
for (const msg of newMessages) {
  for (const block of msg.message.content) {
    if (block.type === 'text') {
      textLen += block.text.length
    } else if (block.type === 'thinking') {
      thinkingLen += block.thinking.length
    } else if (block.type === 'tool_use') {
      toolLengths[sanitizedName] = jsonStringify(block.input).length
    }
  }
}
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/logging.ts` - 本文件，所有 API 日志逻辑
- `src/services/api/emptyUsage.ts` - 导出 `EMPTY_USAGE` 常量

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/QueryEngine.ts` | `EMPTY_USAGE`, `NonNullableUsage` | 查询引擎初始化 |
| `src/cli/print.ts` | `EMPTY_USAGE` | 非交互式模式 |
| `src/utils/forkedAgent.ts` | `EMPTY_USAGE`, `NonNullableUsage` | Fork Agent 使用跟踪 |
| `src/utils/sideQuestion.ts` | `NonNullableUsage` | Side question 类型定义 |
| `src/services/api/claude.ts` | `logAPIQuery`, `logAPIError`, `logAPISuccessAndDuration` | API 调用包装器 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/services/analytics/index.ts` | `logEvent` - 发送 analytics 事件 |
| `src/services/analytics/metadata.ts` | `sanitizeToolNameForAnalytics` |
| `src/utils/telemetry/events.ts` | `logOTelEvent` - OTLP 遥测 |
| `src/utils/telemetry/sessionTracing.ts` | `endLLMRequestSpan`, `isBetaTracingEnabled` |
| `src/utils/model/providers.ts` | `getAPIProviderForStatsig` |
| `src/utils/agentContext.ts` | `consumeInvokingRequestId` |
| `src/bootstrap/state.ts` | 会话状态管理 |

---

## 依赖与外部交互

### Analytics 事件

| 事件名 | 触发时机 | 关键字段 |
|--------|---------|---------|
| `tengu_api_query` | API 请求发送前 | model, messagesLength, temperature, betas |
| `tengu_api_success` | API 响应成功 | inputTokens, outputTokens, costUSD, durationMs |
| `tengu_api_error` | API 调用失败 | error, errorType, status, attempt |
| `tengu_teleport_first_message_success` | Teleport 首消息成功 | session_id |
| `tengu_teleport_first_message_error` | Teleport 首消息失败 | session_id, error_type |

### OTLP 事件

| 事件名 | 用途 |
|--------|------|
| `api_request` | 成功的 API 请求（用于 Perfetto 追踪） |
| `api_error` | 失败的 API 请求 |

### 环境变量读取

```typescript
// Anthropic 环境配置（用于分析用户自定义配置）
process.env.ANTHROPIC_BASE_URL   // 自定义 API 端点
process.env.ANTHROPIC_MODEL      // 环境指定的模型
process.env.ANTHROPIC_SMALL_FAST_MODEL // Fast 模式模型

// 构建时宏
MACRO.BUILD_TIME  // 用于计算构建年龄
MACRO.VERSION     // 用户代理版本
```

---

## 风险、边界与改进建议

### 已知风险

1. **类型安全**
   - `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型断言广泛使用
   - 需要人工验证确保不包含敏感路径

2. **性能影响**
   - 消息内容解析遍历所有 content blocks
   - 大型会话可能影响日志记录性能

3. **内存使用**
   - `toolUseContentLengths` 记录每个工具调用的输入长度
   - 大量工具调用时对象可能很大

### 边界情况

| 场景 | 行为 |
|------|------|
| 网关检测失败 | 不记录 gateway 字段，不影响核心功能 |
| `newMessages` 为空 | 内容长度字段为 undefined，不发送 |
| `llmSpan` 未提供 | Tracing 结束时不匹配请求/响应 |
| `cache_deleted_input_tokens` | 仅在 `CACHED_MICROCOMPACT` feature 开启时记录 |
| Teleport 非首消息 | 不触发 teleport 专用事件 |

### 改进建议

1. **网关检测扩展**
   ```typescript
   // 建议添加更多网关支持
   const GATEWAY_FINGERPRINTS = {
     // ... existing
     'azure-openai': { prefixes: ['x-ms-'] },
     'aws-bedrock': { prefixes: ['x-amzn-'] },
   }
   ```

2. **错误分类细化**
   - 当前 `classifyAPIError` 在其他文件中
   - 建议统一错误分类标准，支持更多错误类型

3. **采样策略**
   - 高频率 API 调用可能导致 analytics 事件过多
   - 考虑添加客户端采样逻辑

4. **敏感数据过滤**
   - `sanitizeToolNameForAnalytics` 已处理 MCP 工具名
   - 建议添加工具输入参数的自动 PII 检测

5. **测试覆盖**
   - 添加网关检测的单元测试（模拟各种响应头）
   - 测试内容长度计算的准确性

### 相关模式
- 与 `promptCacheBreakDetection.ts` 共享 `GlobalCacheStrategy` 类型
- 与 `errors.ts` 共享错误分类逻辑
- 与 `telemetry/` 目录紧密集成
