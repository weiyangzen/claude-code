# betas.ts 深度研究文档

## 场景与职责

`betas.ts` 是 Claude Code CLI 中管理所有 Anthropic API Beta 功能开关的核心配置文件。它集中定义了各种实验性功能、新模型能力的 Beta Header 标识符，以及不同平台（Bedrock、Vertex AI）对 Beta 功能的支持策略。

### 核心使用场景
1. **API 请求构造**：在调用 Anthropic API 时附加必要的 Beta Headers 以启用特定功能
2. **多平台适配**：处理 Claude API/Foundry、Vertex AI、Bedrock 等不同平台对 Beta 功能的支持差异
3. **功能开关控制**：通过 GrowthBook feature flags 动态控制某些 Beta 功能的启用
4. **Token 估算**：在估算请求 token 数量时考虑 Beta 功能的影响

---

## 功能点目的

### 1. Beta Header 定义

| 常量 | Header 值 | 功能描述 | 控制方式 |
|------|-----------|---------|---------|
| `CLAUDE_CODE_20250219_BETA_HEADER` | `claude-code-20250219` | Claude Code 核心功能 | 固定 |
| `INTERLEAVED_THINKING_BETA_HEADER` | `interleaved-thinking-2025-05-14` | 交错思考模式 | 固定 |
| `CONTEXT_1M_BETA_HEADER` | `context-1m-2025-08-07` | 1M 上下文窗口 | 固定 |
| `CONTEXT_MANAGEMENT_BETA_HEADER` | `context-management-2025-06-27` | 上下文管理 | 固定 |
| `STRUCTURED_OUTPUTS_BETA_HEADER` | `structured-outputs-2025-12-15` | 结构化输出 | 固定 |
| `WEB_SEARCH_BETA_HEADER` | `web-search-2025-03-05` | 网络搜索 | 固定 |
| `TOOL_SEARCH_BETA_HEADER_1P` | `advanced-tool-use-2025-11-20` | 工具搜索（第一方） | 固定 |
| `TOOL_SEARCH_BETA_HEADER_3P` | `tool-search-tool-2025-10-19` | 工具搜索（第三方） | 固定 |
| `EFFORT_BETA_HEADER` | `effort-2025-11-24` | 努力程度控制 | 固定 |
| `TASK_BUDGETS_BETA_HEADER` | `task-budgets-2026-03-13` | 任务预算 | 固定 |
| `PROMPT_CACHING_SCOPE_BETA_HEADER` | `prompt-caching-scope-2026-01-05` | Prompt 缓存范围 | 固定 |
| `FAST_MODE_BETA_HEADER` | `fast-mode-2026-02-01` | 快速模式 | 固定 |
| `REDACT_THINKING_BETA_HEADER` | `redact-thinking-2026-02-12` | 思考内容脱敏 | 固定 |
| `TOKEN_EFFICIENT_TOOLS_BETA_HEADER` | `token-efficient-tools-2026-03-28` | Token 高效工具 | 固定 |
| `SUMMARIZE_CONNECTOR_TEXT_BETA_HEADER` | `summarize-connector-text-2026-03-13` | 连接器文本摘要 | GrowthBook |
| `AFK_MODE_BETA_HEADER` | `afk-mode-2026-01-31` | AFK 模式 | GrowthBook |
| `CLI_INTERNAL_BETA_HEADER` | `cli-internal-2026-02-09` | CLI 内部功能 | 仅 ant 用户 |
| `ADVISOR_BETA_HEADER` | `advisor-tool-2026-03-01` | Advisor 工具 | 固定 |

### 2. 平台特定处理

#### Bedrock 特殊处理
```typescript
export const BEDROCK_EXTRA_PARAMS_HEADERS = new Set([
  INTERLEAVED_THINKING_BETA_HEADER,
  CONTEXT_1M_BETA_HEADER,
  TOOL_SEARCH_BETA_HEADER_3P,
])
```

**原因**：Bedrock 仅支持有限的 Beta Headers，且只能通过 `extraBodyParams` 传递，不能放在 Headers 中。

#### Vertex AI 限制
```typescript
export const VERTEX_COUNT_TOKENS_ALLOWED_BETAS = new Set([
  CLAUDE_CODE_20250219_BETA_HEADER,
  INTERLEAVED_THINKING_BETA_HEADER,
  CONTEXT_MANAGEMENT_BETA_HEADER,
])
```

**原因**：Vertex 的 countTokens API 仅允许特定的 Beta Headers，其他会导致 400 错误。

### 3. 动态 Beta 控制

部分 Beta Header 通过 GrowthBook feature flags 动态控制：

```typescript
export const SUMMARIZE_CONNECTOR_TEXT_BETA_HEADER = feature('CONNECTOR_TEXT')
  ? 'summarize-connector-text-2026-03-13'
  : ''

export const AFK_MODE_BETA_HEADER = feature('TRANSCRIPT_CLASSIFIER')
  ? 'afk-mode-2026-01-31'
  : ''
```

以及 ant 用户专属：
```typescript
export const CLI_INTERNAL_BETA_HEADER =
  process.env.USER_TYPE === 'ant' ? 'cli-internal-2026-02-09' : ''
```

---

## 具体技术实现

### 数据结构

```typescript
// Beta Header 字符串常量
export const CLAUDE_CODE_20250219_BETA_HEADER = 'claude-code-20250219'
// ... 更多 header 定义

// Bedrock 特殊处理的 Header 集合
export const BEDROCK_EXTRA_PARAMS_HEADERS: Set<string>

// Vertex countTokens 允许的 Header 集合
export const VERTEX_COUNT_TOKENS_ALLOWED_BETAS: Set<string>
```

### 关键代码路径

#### 1. API 请求构造路径

```
发起 API 请求
    ↓
src/services/api/claude.ts
    ↓
根据当前平台和配置收集 Beta Headers
    ↓
检查 BEDROCK_EXTRA_PARAMS_HEADERS / VERTEX_COUNT_TOKENS_ALLOWED_BETAS
    ↓
构造最终请求（headers 或 extraBodyParams）
```

**关键文件引用**：
- `src/services/api/claude.ts`: 核心 API 请求构造，使用所有 Beta Headers
- `src/services/api/client.ts`: API 客户端配置
- `src/utils/betas.ts`: Beta 功能工具函数

#### 2. Token 估算路径

```
估算请求 token
    ↓
src/services/tokenEstimation.ts
    ↓
过滤 Beta Headers（仅保留 VERTEX_COUNT_TOKENS_ALLOWED_BETAS 中的）
    ↓
调用 countTokens API
```

**关键文件引用**：
- `src/services/tokenEstimation.ts`: Token 估算，处理 Vertex 限制

#### 3. Prompt 缓存检测路径

```
检测 prompt 缓存状态
    ↓
src/services/api/promptCacheBreakDetection.ts
    ↓
分析 Beta Headers 对缓存的影响
    ↓
报告缓存破坏情况
```

**关键文件引用**：
- `src/services/api/promptCacheBreakDetection.ts`: 缓存破坏检测

#### 4. 上下文管理路径

```
管理对话上下文
    ↓
src/utils/context.ts
    ↓
根据 CONTEXT_MANAGEMENT_BETA_HEADER 等调整上下文策略
    ↓
优化缓存命中率
```

**关键文件引用**：
- `src/utils/context.ts`: 上下文管理
- `src/utils/sideQuery.ts`: 侧边查询，使用 Beta Headers

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `bun:bundle` 的 `feature` | GrowthBook feature flag 检查 |

### 被依赖方

| 文件 | 使用的常量 | 用途 |
|------|-----------|------|
| `src/services/api/claude.ts` | 所有 Beta Headers | API 请求构造 |
| `src/services/api/client.ts` | 多个 Headers | API 客户端配置 |
| `src/services/tokenEstimation.ts` | `VERTEX_COUNT_TOKENS_ALLOWED_BETAS` | Token 估算过滤 |
| `src/services/api/promptCacheBreakDetection.ts` | 多个 Headers | 缓存破坏检测 |
| `src/utils/context.ts` | `CONTEXT_1M_BETA_HEADER` 等 | 上下文管理 |
| `src/utils/betas.ts` | 多个 Headers | Beta 功能工具函数 |
| `src/utils/sideQuery.ts` | 多个 Headers | 侧边查询 |
| `src/bootstrap/state.ts` | 多个 Headers | 启动状态管理 |
| `src/services/api/errors.ts` | 多个 Headers | 错误处理 |

### 外部平台交互

| 平台 | 特殊处理 | 说明 |
|------|---------|------|
| Claude API / Foundry | 标准 Headers | 支持所有 Beta Headers |
| Bedrock | `extraBodyParams` | 仅支持 `BEDROCK_EXTRA_PARAMS_HEADERS` 中的 |
| Vertex AI | Header 过滤 | countTokens 仅支持 `VERTEX_COUNT_TOKENS_ALLOWED_BETAS` |

---

## 风险、边界与改进建议

### 当前风险

1. **平台差异复杂性**
   - 不同平台对 Beta Headers 的支持差异导致代码分支复杂
   - 容易遗漏某个平台的特殊处理

2. **Beta Header 过期**
   - 实验性功能转正后，对应的 Beta Header 可能不再需要
   - 需要定期清理已正式发布的功能的 Beta Header

3. **GrowthBook 依赖**
   - `SUMMARIZE_CONNECTOR_TEXT_BETA_HEADER` 和 `AFK_MODE_BETA_HEADER` 依赖 feature flags
   - 如果 GrowthBook 不可用，这些功能会被静默禁用

4. **空字符串 Beta Header**
   - 当 feature flag 关闭时，对应的常量为空字符串
   - 需要调用方过滤空值，否则可能传递无效 header

### 边界情况

| 场景 | 处理方式 |
|------|---------|
| Bedrock + 不支持的 Beta | 通过 `extraBodyParams` 传递，可能被服务器忽略 |
| Vertex countTokens + 不允许的 Beta | 客户端过滤，不发送到 API |
| 空字符串 Beta Header | 调用方应过滤，不加入 headers 数组 |
| ant 用户使用 CLI_INTERNAL | 正常启用，外部用户为空字符串 |

### 改进建议

1. **Beta Header 生命周期管理**
   ```typescript
   // 建议添加过期标记
   export const CONTEXT_1M_BETA_HEADER = {
     value: 'context-1m-2025-08-07',
     deprecated: false,  // 或 true，配合预计转正日期
     estimatedGA: '2026-06-01'
   }
   ```

2. **统一的平台适配层**
   ```typescript
   // 建议封装平台适配逻辑
   export function getBetaHeadersForPlatform(
     platform: 'claude-api' | 'bedrock' | 'vertex',
     requestedFeatures: string[]
   ): string[]
   ```

3. **类型安全增强**
   ```typescript
   // 建议使用联合类型替代字符串
   export type BetaHeader = 
     | 'claude-code-20250219'
     | 'interleaved-thinking-2025-05-14'
     | // ...
   ```

4. **自动化测试覆盖**
   - 每个 Beta Header 应在各平台有对应的测试用例
   - 验证 feature flag 开关状态变化时的行为

5. **文档化平台限制**
   - 在代码中更详细地注释各平台的限制原因
   - 链接到相关的外部文档或工单

### 相关配置关联

与 `src/utils/betas.ts` 的关系：
- `constants/betas.ts`: 定义原始 Beta Header 字符串
- `utils/betas.ts`: 提供使用这些 Header 的工具函数和逻辑

两者应保持一致，修改时需同步更新。
