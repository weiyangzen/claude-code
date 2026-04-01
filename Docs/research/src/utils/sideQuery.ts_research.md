# sideQuery.ts 深度研究

## 场景与职责

`sideQuery.ts` 是一个**轻量级 API 查询封装器**，用于在主对话循环之外执行 Anthropic API 调用。它确保所有"旁路查询"（side queries）都遵循统一的 OAuth 验证、归因和日志记录标准。

**核心职责：**
1. 提供标准化的 API 调用接口
2. 自动处理指纹计算和归因头注入
3. 管理模型 Beta 功能和结构化输出
4. 统一分析和日志记录

**应用场景：**
- 权限解释器（permission explainer）
- 会话搜索（session search）
- 模型验证
- 任何需要调用 API 但不在主对话流中的功能

---

## 功能点目的

### 1. SideQuery 选项定义
```typescript
export type SideQueryOptions = {
  model: string
  system?: string | TextBlockParam[]
  messages: MessageParam[]
  tools?: Tool[] | BetaToolUnion[]
  tool_choice?: ToolChoice
  output_format?: BetaJSONOutputFormat
  max_tokens?: number
  maxRetries?: number
  signal?: AbortSignal
  skipSystemPromptPrefix?: boolean
  temperature?: number
  thinking?: number | false
  stop_sequences?: string[]
  querySource: QuerySource
}
```

**关键选项说明：**
- `skipSystemPromptPrefix`：跳过 CLI 系统提示前缀（用于内部分类器）
- `thinking`：思考预算或禁用思考
- `querySource`：分析追踪标识符

### 2. 主查询函数
```typescript
export async function sideQuery(opts: SideQueryOptions): Promise<BetaMessage>
```

**处理流程：**
1. 获取 Anthropic 客户端（带重试配置）
2. 计算指纹用于 OAuth 归因
3. 构建系统提示块数组（归因头 + CLI 前缀 + 自定义系统提示）
4. 处理思考配置
5. 规范化模型字符串
6. 执行 API 调用
7. 记录分析事件 `tengu_api_success`

### 3. 指纹计算
```typescript
function extractFirstUserMessageText(messages: MessageParam[]): string
const fingerprint = computeFingerprint(messageText, MACRO.VERSION)
const attributionHeader = getAttributionHeader(fingerprint)
```

- 从首条用户消息提取文本
- 结合版本信息计算指纹
- 生成归因头用于 OAuth 验证

---

## 具体技术实现

### 系统提示构建
```typescript
const systemBlocks: TextBlockParam[] = [
  attributionHeader ? { type: 'text', text: attributionHeader } : null,
  ...(skipSystemPromptPrefix
    ? []
    : [{ type: 'text' as const, text: getCLISyspromptPrefix(...) }]),
  ...(Array.isArray(system)
    ? system
    : system
      ? [{ type: 'text' as const, text: system }]
      : []),
].filter((block): block is TextBlockParam => block !== null)
```

**设计要点：**
- 归因头始终独立成块，确保服务器正确解析 `cc_entrypoint`
- CLI 前缀可跳过（内部分类器使用自己的提示词）

### Beta 功能处理
```typescript
const betas = [...getModelBetas(model)]
if (
  output_format &&
  modelSupportsStructuredOutputs(model) &&
  !betas.includes(STRUCTURED_OUTPUTS_BETA_HEADER)
) {
  betas.push(STRUCTURED_OUTPUTS_BETA_HEADER)
}
```

### 分析日志
```typescript
logEvent('tengu_api_success', {
  requestId,
  querySource: opts.querySource,
  model: normalizedModel,
  inputTokens: response.usage.input_tokens,
  outputTokens: response.usage.output_tokens,
  cachedInputTokens: response.usage.cache_read_input_tokens ?? 0,
  uncachedInputTokens: response.usage.cache_creation_input_tokens ?? 0,
  durationMsIncludingRetries: now - start,
  timeSinceLastApiCallMs: lastCompletion !== null ? now - lastCompletion : undefined,
})
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `SideQueryOptions` | 查询选项类型 |
| `sideQuery` | 主查询函数 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `@anthropic-ai/sdk` | Anthropic API 客户端类型 |
| `../bootstrap/state.js` | API 完成时间戳管理 |
| `../constants/betas.js` | Beta 功能常量 |
| `../constants/querySource.js` | 查询源类型 |
| `../constants/system.js` | 归因头和系统前缀 |
| `../services/analytics/index.js` | 分析日志 |
| `../services/api/claude.js` | API 元数据 |
| `../services/api/client.js` | Anthropic 客户端 |
| `./betas.js` | 模型 Beta 功能 |
| `./fingerprint.js` | 指纹计算 |
| `./model/model.js` | 模型字符串规范化 |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/cli/handlers/autoMode.ts` | 自动模式 |
| `src/services/tokenEstimation.ts` | Token 估算 |
| `src/query.ts` | 主查询 |
| `src/utils/agenticSessionSearch.ts` | 会话搜索 |
| `src/memdir/memoryScan.ts` | 内存扫描 |
| `src/memdir/findRelevantMemories.ts` | 记忆查找 |
| `src/utils/claudeInChrome/mcpServer.ts` | Chrome MCP 服务器 |
| `src/utils/attachments.ts` | 附件处理 |
| `src/utils/permissions/yoloClassifier.ts` | YOLO 分类器 |
| `src/utils/model/validateModel.ts` | 模型验证 |
| `src/utils/permissions/permissionExplainer.ts` | 权限解释 |
| `src/utils/permissions/permissions.ts` | 权限管理 |

---

## 依赖与外部交互

### 外部 API
- **Anthropic API**：`client.beta.messages.create`

### 分析事件
- `tengu_api_success`：API 调用成功事件

### 环境交互
- 使用 `performance.now()` 计时
- 更新全局 API 完成时间戳

---

## 风险、边界与改进建议

### 已知风险

1. **API 依赖**
   - 所有功能依赖外部 API 可用性
   - 网络故障导致功能不可用

2. **缓存失效**
   - 每次调用独立，不共享主对话的提示缓存
   - 可能增加 Token 成本

3. **超时处理**
   - AbortSignal 由调用方提供
   - 未设置默认超时

### 边界情况

| 场景 | 处理 |
|------|------|
| 空系统提示 | 仅包含归因头和 CLI 前缀 |
| 已中止信号 | 由 API 客户端处理 |
| 模型不支持结构化输出 | 跳过添加 Beta 头 |
| thinking=false | 发送 `{ type: 'disabled' }` |

### 改进建议

1. **缓存优化**
   - 支持提示缓存复用
   - 添加缓存命中率追踪

2. **重试策略**
   - 可配置重试策略
   - 区分可重试和不可重试错误

3. **超时管理**
   - 添加默认超时
   - 支持按查询类型设置不同超时

4. **成本控制**
   - 添加 Token 预算限制
   - 超限自动降级或拒绝

5. **可观测性**
   - 添加更详细的调用链追踪
   - 记录请求/响应采样
