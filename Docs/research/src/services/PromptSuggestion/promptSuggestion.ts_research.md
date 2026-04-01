# PromptSuggestion Service 研究文档

## 文件信息
- **文件路径**: `src/services/PromptSuggestion/promptSuggestion.ts`
- **文件大小**: 17,065 bytes
- **研究日期**: 2026-04-01

---

## 一、场景与职责

### 1.1 核心场景
PromptSuggestion 服务是 Claude Code CLI 的**智能提示建议系统**，旨在：
1. **预测用户下一步输入**: 基于对话上下文，预测用户可能想要输入的下一个指令
2. **提升交互效率**: 在用户思考时预生成可能的输入选项，减少输入时间
3. **支持 SDK 推送路径**: 为 SDK 集成提供提示建议的生成和追踪能力

### 1.2 主要职责
- **提示建议生成**: 通过 forked agent 调用 LLM 生成上下文相关的输入建议
- **建议过滤与验证**: 多维度过滤不合适的建议（长度、内容、格式等）
- **状态管理**: 管理提示建议的生成、展示、接受/拒绝状态
- **遥测追踪**: 记录提示建议的接受率、忽略率等关键指标
- **与 Speculation 集成**: 当用户接受建议时，触发 Speculation 预执行

---

## 二、功能点目的

### 2.1 提示建议生成 (`generateSuggestion`)
**目的**: 使用 forked agent 异步生成用户可能输入的下一个指令

**关键设计决策**:
- 使用 `SUGGESTION_PROMPT` 系统提示词指导模型生成符合用户风格的建议
- 通过 `runForkedAgent` 复用主线程的 prompt cache 以降低成本
- 不传递 `tools: []`（这会破坏 cache），而是通过 `canUseTool` 回调拒绝所有工具调用

### 2.2 建议过滤 (`shouldFilterSuggestion`)
**目的**: 确保生成的建议质量，过滤掉不合适的内容

**过滤维度**:
| 过滤器 | 规则 | 目的 |
|--------|------|------|
| `done` | 完全匹配 "done" | 避免无意义建议 |
| `meta_text` | "nothing found", "no suggestion", "silence" 等 | 过滤模型拒绝输出 |
| `meta_wrapped` | 被括号/方括号包裹的内容 | 过滤元推理输出 |
| `error_message` | API 错误前缀 | 过滤错误信息 |
| `prefixed_label` | 带标签前缀 (如 "Suggestion:") | 过滤格式化输出 |
| `too_few_words` | 少于 2 个词（除白名单外） | 确保建议完整性 |
| `too_many_words` | 超过 12 个词 | 限制建议长度 |
| `too_long` | 超过 100 字符 | 限制建议长度 |
| `multiple_sentences` | 多句话 | 确保简洁性 |
| `has_formatting` | 包含换行或 markdown | 确保纯文本 |
| `evaluative` | 评价性词汇 (thanks, looks good 等) | 避免评价性语言 |
| `claude_voice` | Claude 口吻 (Let me, I'll 等) | 确保用户口吻 |

**单字白名单**: `yes`, `yeah`, `yep`, `ok`, `okay`, `push`, `commit`, `deploy`, `stop`, `continue`, `check`, `exit`, `quit`, `no` 等

### 2.3 启用控制 (`shouldEnablePromptSuggestion`)
**目的**: 根据多种条件决定是否启用提示建议功能

**决策层级** (按优先级):
1. **环境变量覆盖**: `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` (true/false)
2. **GrowthBook 开关**: `tengu_chomp_inflection` feature flag
3. **交互模式检查**: 非交互模式（print mode, piped input, SDK）禁用
4. **Swarm 检查**: Agent Swarms 中的 teammate 禁用（仅 leader 显示建议）
5. **用户设置**: `settings.promptSuggestionEnabled` (默认 true)

### 2.4 抑制原因检查 (`getSuggestionSuppressReason`)
**目的**: 在生成前检查是否应该跳过建议生成

**抑制条件**:
- `disabled`: 功能被禁用
- `pending_permission`: 有待处理的权限请求
- `elicitation_active`: 有活跃的引导请求
- `plan_mode`: 处于计划模式
- `rate_limit`: 外部用户且处于限流状态

### 2.5 遥测追踪
**事件类型**:
- `tengu_prompt_suggestion_init`: 初始化时记录启用状态及来源
- `tengu_prompt_suggestion`: 记录建议的接受、忽略、抑制事件
  - `outcome`: `accepted` | `ignored` | `suppressed`
  - `source`: `cli` | `sdk`
  - `prompt_id`: `user_intent` | `stated_intent`
  - `similarity`: 用户输入与建议的相似度比率
  - `timeToAcceptMs` / `timeToIgnoreMs`: 响应时间

---

## 三、具体技术实现

### 3.1 关键流程

#### 3.1.1 建议生成流程 (`executePromptSuggestion`)
```
1. 检查 querySource 为 'repl_main_thread' (仅 CLI 主线程)
2. 创建 AbortController 用于取消
3. 调用 tryGenerateSuggestion 尝试生成
   a. 检查 abort 状态
   b. 检查 assistant 轮数 >= 2
   c. 检查最后一条消息不是 API 错误
   d. 检查 cache 冷启动状态 (MAX_PARENT_UNCACHED_TOKENS = 10,000)
   e. 检查 AppState 抑制条件
   f. 调用 generateSuggestion 生成
   g. 再次检查 abort 状态
   h. 过滤建议
4. 更新 AppState.promptSuggestion
5. 如果 Speculation 启用，启动 speculation
```

#### 3.1.2 Cache 冷启动检查 (`getParentCacheSuppressReason`)
```typescript
// 计算未缓存的 token 数量
const uncachedTokens = inputTokens + cacheWriteTokens + outputTokens
// 如果超过 10,000 tokens，返回 'cache_cold' 抑制原因
return uncachedTokens > MAX_PARENT_UNCACHED_TOKENS ? 'cache_cold' : null
```

### 3.2 数据结构

#### 3.2.1 PromptVariant 类型
```typescript
export type PromptVariant = 'user_intent' | 'stated_intent'
// 当前仅使用 'user_intent'
```

#### 3.2.2 建议结果类型
```typescript
{
  suggestion: string           // 建议文本
  promptId: PromptVariant     // 提示变体
  generationRequestId: string | null  // 用于 RL 数据集关联
}
```

#### 3.2.3 AppState 中的提示建议状态
```typescript
promptSuggestion: {
  text: string | null                    // 建议文本
  promptId: 'user_intent' | 'stated_intent' | null
  shownAt: number                        // 展示时间戳
  acceptedAt: number                     // 接受时间戳
  generationRequestId: string | null     // 生成请求 ID
}
```

### 3.3 关键常量

#### 3.3.1 SUGGESTION_PROMPT (系统提示词)
```
[SUGGESTION MODE: Suggest what the user might naturally type next into Claude Code.]

核心指导原则:
1. 预测用户会输入什么，而不是你认为他们应该做什么
2. "他们会想'我正要输入这个'吗？" 测试
3. 具体优于笼统: "run the tests" 优于 "continue"
4. 禁止内容: 评价性语言、问题、Claude 口吻、新想法、多句话
5. 格式: 2-12 个词，匹配用户风格，或为空
```

#### 3.3.2 MAX_PARENT_UNCACHED_TOKENS
```typescript
const MAX_PARENT_UNCACHED_TOKENS = 10_000
// 当父请求的未缓存 token 超过此阈值时，跳过建议生成
// 避免在 cache 冷启动时增加额外成本
```

### 3.4 协议与接口

#### 3.4.1 REPLHookContext 依赖
```typescript
type REPLHookContext = {
  messages: Message[]           // 完整消息历史
  systemPrompt: SystemPrompt    // 系统提示
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  querySource?: QuerySource
}
```

#### 3.4.2 CacheSafeParams 传递
通过 `createCacheSafeParams(context)` 创建，确保 forked agent 与父请求共享 prompt cache:
- `systemPrompt`
- `userContext`
- `systemContext`
- `toolUseContext`
- `forkContextMessages`

---

## 四、关键代码路径与文件引用

### 4.1 核心函数调用链

```
executePromptSuggestion (REPL hook 入口)
├── tryGenerateSuggestion
│   ├── count(messages, m => m.type === 'assistant') >= 2
│   ├── getLastAssistantMessage(messages)
│   ├── getParentCacheSuppressReason(lastAssistantMessage)
│   ├── getSuggestionSuppressReason(appState)
│   ├── getPromptVariant()
│   └── generateSuggestion
│       └── runForkedAgent (from utils/forkedAgent.ts)
│           ├── createSubagentContext
│           └── query (主查询循环)
├── shouldFilterSuggestion
└── startSpeculation (from ./speculation.ts, 条件触发)
```

### 4.2 文件依赖关系

```
promptSuggestion.ts
├── 依赖
│   ├── ../../bootstrap/state.js (getIsNonInteractiveSession)
│   ├── ../../state/AppState.js (AppState type)
│   ├── ../../types/message.js (Message type)
│   ├── ../../utils/agentSwarmsEnabled.js (isAgentSwarmsEnabled)
│   ├── ../../utils/array.js (count)
│   ├── ../../utils/envUtils.js (isEnvDefinedFalsy, isEnvTruthy)
│   ├── ../../utils/errors.js (toError)
│   ├── ../../utils/forkedAgent.js (runForkedAgent, createCacheSafeParams)
│   ├── ../../utils/hooks/postSamplingHooks.js (REPLHookContext)
│   ├── ../../utils/log.js (logError)
│   ├── ../../utils/messages.js (createUserMessage, getLastAssistantMessage)
│   ├── ../../utils/settings/settings.js (getInitialSettings)
│   ├── ../../utils/teammate.js (isTeammate)
│   ├── ../analytics/growthbook.js (getFeatureValue_CACHED_MAY_BE_STALE)
│   ├── ../analytics/index.js (logEvent)
│   ├── ../claudeAiLimits.js (currentLimits)
│   └── ./speculation.js (isSpeculationEnabled, startSpeculation)
└── 被依赖
    ├── ../../state/AppStateStore.ts (shouldEnablePromptSuggestion import)
    ├── ./speculation.ts (generateSuggestion import)
    └── REPL hooks 注册
```

### 4.3 关键代码位置

| 功能 | 行号 | 说明 |
|------|------|------|
| `shouldEnablePromptSuggestion` | 37-94 | 启用检查主函数 |
| `executePromptSuggestion` | 184-237 | REPL hook 入口 |
| `tryGenerateSuggestion` | 125-182 | 建议生成协调器 |
| `generateSuggestion` | 294-352 | 核心生成逻辑 |
| `shouldFilterSuggestion` | 354-456 | 建议过滤器 |
| `getSuggestionSuppressReason` | 107-119 | 抑制原因检查 |
| `getParentCacheSuppressReason` | 241-256 | Cache 冷启动检查 |
| `SUGGESTION_PROMPT` | 258-287 | 系统提示词定义 |
| `logSuggestionOutcome` | 462-497 | 结果追踪 |
| `logSuggestionSuppressed` | 499-523 | 抑制事件追踪 |

---

## 五、依赖与外部交互

### 5.1 外部服务依赖

| 服务 | 用途 | 关键交互 |
|------|------|----------|
| **Anthropic API** | 生成提示建议 | 通过 `runForkedAgent` -> `query` 调用 |
| **GrowthBook** | Feature flag 控制 | `getFeatureValue_CACHED_MAY_BE_STALE('tengu_chomp_inflection')` |
| **Analytics** | 遥测追踪 | `logEvent('tengu_prompt_suggestion', ...)` |

### 5.2 内部模块依赖

| 模块 | 依赖内容 | 用途 |
|------|----------|------|
| `forkedAgent.ts` | `runForkedAgent`, `createCacheSafeParams` | 创建隔离的 agent 执行建议生成 |
| `AppStateStore.ts` | `AppState`, `SpeculationState` | 状态类型定义和访问 |
| `postSamplingHooks.ts` | `REPLHookContext` | Hook 上下文类型 |
| `settings.ts` | `getInitialSettings` | 读取用户设置 |
| `speculation.ts` | `isSpeculationEnabled`, `startSpeculation` | 启动预执行 |
| `growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` | Feature flag |

### 5.3 环境变量依赖

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | 强制启用/禁用提示建议 |
| `USER_TYPE` | 区分 ant/external 用户，影响遥测和限制检查 |

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 Cache 冷启动成本
- **风险**: 当父请求未缓存 token 超过 10,000 时，建议生成会增加额外 API 成本
- **缓解**: `getParentCacheSuppressReason` 检查会跳过此类情况
- **边界**: 阈值固定为 10,000，可能过于保守或激进

#### 6.1.2 Prompt Cache 破坏风险
- **风险**: 如果 forked agent 的 API 参数与父请求不同，会破坏 prompt cache
- **缓解**: 注释明确警告不要覆盖 `effortValue` 或 `maxOutputTokens`
- **历史**: PR #18143 曾因设置 `effort:'low'` 导致 cache 命中率从 92.7% 降至 61%

#### 6.1.3 建议质量风险
- **风险**: 模型可能生成不相关或不恰当的建议
- **缓解**: 12 层过滤器 (`shouldFilterSuggestion`) 拦截低质量建议
- **边界**: 过滤器基于启发式规则，可能误杀或漏过

#### 6.1.4 竞态条件
- **风险**: 用户快速输入时，建议生成可能仍在进行
- **缓解**: `abortPromptSuggestion` 和 AbortController 用于取消进行中的请求
- **边界**: 取消可能有延迟，导致过时建议短暂显示

### 6.2 边界条件

| 边界 | 行为 |
|------|------|
| Assistant 轮数 < 2 | 不生成建议 (上下文不足) |
| 最后消息是 API 错误 | 不生成建议 |
| 非交互模式 | 功能禁用 |
| Teammate 模式 | 功能禁用 (仅 leader 显示) |
| 计划模式 | 不生成建议 |
| 限流状态 | 外部用户不生成建议 |

### 6.3 改进建议

#### 6.3.1 动态 Cache 阈值
```typescript
// 建议: 基于会话历史动态调整阈值
const dynamicThreshold = calculateDynamicThreshold(sessionHistory)
```

#### 6.3.2 个性化建议模型
- 当前: 使用通用提示词生成建议
- 建议: 基于用户历史输入风格微调模型或提示词

#### 6.3.3 A/B 测试框架
- 当前: 仅支持简单的 feature flag
- 建议: 完整的 A/B 测试框架，测试不同提示词变体效果

#### 6.3.4 离线建议缓存
- 当前: 每次都需要 API 调用
- 建议: 缓存常见场景的建议模板，减少 API 调用

#### 6.3.5 多语言支持
- 当前: 提示词为英文，可能不适用于非英语用户
- 建议: 基于用户设置的语言动态调整提示词

#### 6.3.6 建议质量反馈循环
- 当前: 仅记录接受/忽略
- 建议: 收集用户编辑建议后的最终输入，用于改进模型

### 6.4 测试建议

| 测试类型 | 覆盖点 |
|----------|--------|
| 单元测试 | 每个过滤器的边界条件 |
| 集成测试 | 与 forkedAgent 的 cache 共享 |
| 端到端测试 | 完整用户交互流程 |
| 性能测试 | 高频率建议生成的资源消耗 |
| 稳定性测试 | 竞态条件下的行为一致性 |
