# compact.ts 深度研究文档

## 场景与职责

`compact.ts` 是 Claude Code 上下文压缩功能的核心实现模块，负责执行完整的对话摘要和上下文压缩流程。它是整个压缩体系中最复杂的模块，处理从消息预处理、摘要生成到后压缩状态恢复的全流程。

该模块支持两种压缩模式：
1. **完整压缩** (`compactConversation`): 压缩整个对话历史，保留最近上下文
2. **部分压缩** (`partialCompactConversation`): 基于消息选择器的定向压缩，支持 `from` 和 `up_to` 两种方向

## 功能点目的

### 1. 消息预处理
- **图片剥离** (`stripImagesFromMessages`): 移除用户消息中的图片/文档块，替换为文本标记，避免摘要 API 因媒体内容触发 prompt-too-long
- **附件过滤** (`stripReinjectedAttachments`): 过滤 skill_discovery/skill_listing 附件，避免向摘要器提供冗余信息

### 2. 摘要生成与容错
- **双路径摘要**: 优先使用 Forked Agent 路径（缓存共享），失败时回退到流式路径
- **Prompt-Too-Long 重试** (`truncateHeadForPTLRetry`): 当摘要请求本身触发 PTL 时，自动丢弃最旧的消息组并重试
- **工具使用限制** (`createCompactCanUseTool`): 禁止压缩 Agent 使用任何工具

### 3. 后压缩状态恢复
- **文件附件恢复** (`createPostCompactFileAttachments`): 恢复最近访问的文件内容
- **计划附件** (`createPlanAttachmentIfNeeded`): 保留计划文件引用
- **技能附件** (`createSkillAttachmentIfNeeded`): 保留已调用技能的指令
- **异步 Agent 附件** (`createAsyncAgentAttachmentsIfNeeded`): 保留后台 Agent 状态
- **计划模式附件** (`createPlanModeAttachmentIfNeeded`): 维持计划模式上下文

### 4. 部分压缩支持
- **方向控制**: `from`（保留前面，压缩后面）vs `up_to`（保留后面，压缩前面）
- **缓存优化**: `up_to` 方向保留前缀缓存命中，`from` 方向牺牲缓存换取灵活性

## 具体技术实现

### 关键常量

```typescript
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
const MAX_COMPACT_STREAMING_RETRIES = 2
const MAX_PTL_RETRIES = 3
```

### 核心数据结构

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage           // 压缩边界标记
  summaryMessages: UserMessage[]          // 摘要消息列表
  attachments: AttachmentMessage[]        // 附件消息
  hookResults: HookResultMessage[]        // SessionStart hooks 结果
  messagesToKeep?: Message[]              // 部分压缩保留的消息
  userDisplayMessage?: string             // 向用户显示的附加信息
  preCompactTokenCount?: number           // 压缩前 token 数
  postCompactTokenCount?: number          // 压缩 API 调用总 token（历史字段）
  truePostCompactTokenCount?: number      // 实际压缩后上下文 token 估算
  compactionUsage?: ReturnType<typeof getTokenUsage>  // API 使用统计
}

export type RecompactionInfo = {
  isRecompactionInChain: boolean
  turnsSincePreviousCompact: number
  previousCompactTurnId?: string
  autoCompactThreshold: number
  querySource?: QuerySource
}
```

### 核心流程

#### 1. 完整压缩流程 (`compactConversation`)

```
compactConversation(messages, context, cacheSafeParams, suppressFollowUpQuestions, customInstructions, isAutoCompact, recompactionInfo)
  │
  ├─> 前置检查
  │   └─ messages.length === 0 → 抛出 ERROR_MESSAGE_NOT_ENOUGH_MESSAGES
  │
  ├─> 执行 PreCompact Hooks
  │   └─ executePreCompactHooks → 合并 customInstructions
  │
  ├─> 构建摘要请求
  │   ├─ getCompactPrompt(customInstructions)
  │   └─ createUserMessage({ content: compactPrompt })
  │
  ├─> 摘要生成（带 PTL 重试）
  │   ├─ streamCompactSummary()
  │   │   ├─ Forked Agent 路径（缓存共享）
  │   │   │   └─ runForkedAgent({ skipCacheWrite: true })
  │   │   │
  │   │   └─ 流式回退路径
  │   │       └─ queryModelWithStreaming()
  │   │
  │   └─ PTL 处理
  │       ├─ summary.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)
  │       ├─ truncateHeadForPTLRetry() → 丢弃最旧消息组
  │       └─ 最多重试 MAX_PTL_RETRIES 次
  │
  ├─> 状态清理
  │   ├─ context.readFileState.clear()
  │   └─ context.loadedNestedMemoryPaths?.clear()
  │
  ├─> 并行生成附件
  │   ├─ createPostCompactFileAttachments()
  │   ├─ createAsyncAgentAttachmentsIfNeeded()
  │   ├─ createPlanAttachmentIfNeeded()
  │   ├─ createPlanModeAttachmentIfNeeded()
  │   ├─ createSkillAttachmentIfNeeded()
  │   └─ 重新注入工具/MCP/Agent 列表附件
  │
  ├─> 执行 SessionStart Hooks
  │   └─ processSessionStartHooks('compact')
  │
  ├─> 构建结果消息
  │   ├─ createCompactBoundaryMessage()
  │   ├─ createUserMessage({ isCompactSummary: true })
  │   └─ buildPostCompactMessages()
  │
  ├─> 埋点上报
  │   └─ logEvent('tengu_compact', { ... })
  │
  ├─> 缓存断点检测通知
  │   └─ notifyCompaction()
  │
  ├─> 执行 PostCompact Hooks
  │   └─ executePostCompactHooks()
  │
  └─> 返回 CompactionResult
```

#### 2. 流式摘要生成 (`streamCompactSummary`)

```typescript
async function streamCompactSummary({
  messages,
  summaryRequest,
  appState,
  context,
  preCompactTokenCount,
  cacheSafeParams,
}): Promise<AssistantMessage>
```

**双路径策略**:

1. **Forked Agent 路径**（优先）:
   - 使用 `runForkedAgent` 创建子进程
   - 通过 `skipCacheWrite: true` 复用主对话的 prompt cache
   - 不设置 `maxOutputTokens`（避免 thinking 配置不匹配导致缓存失效）
   - 成功指标：`tengu_compact_cache_sharing_success`
   - 失败回退：`tengu_compact_cache_sharing_fallback`

2. **流式回退路径**:
   - 使用 `queryModelWithStreaming` 直接调用 API
   - 工具集限制：仅 `FileReadTool` 和 `ToolSearchTool`（如启用）
   - 消息预处理：`stripImagesFromMessages` + `stripReinjectedAttachments`
   - 支持重试（`MAX_COMPACT_STREAMING_RETRIES`）

#### 3. PTL 重试机制 (`truncateHeadForPTLRetry`)

```
ttruncateHeadForPTLRetry(messages, ptlResponse)
  │
  ├─> 剥离之前的重试标记（避免无限循环）
  │
  ├─> 按 API 轮次分组 (groupMessagesByApiRound)
  │
  ├─> 计算需要丢弃的组数
  │   ├─ 能从 PTL 响应解析 tokenGap → 累加直到满足 gap
  │   └─ 无法解析 → 丢弃 20% 的组
  │
  ├─> 确保至少保留一组（有内容可摘要）
  │
  └─> 处理边界条件
      └─ 若结果以 assistant 消息开头 → 前置合成用户标记
```

#### 4. 文件附件恢复 (`createPostCompactFileAttachments`)

```
createPostCompactFileAttachments(readFileState, toolUseContext, maxFiles, preservedMessages)
  │
  ├─> 收集保留消息中已存在的 Read 工具文件路径
  │   └─ 避免重复注入（dedup）
  │
  ├─> 筛选候选文件
  │   ├─ 排除计划文件和 claude.md 文件
  │   ├─ 排除已在保留消息中的文件
  │   └─ 按时间戳排序，取最近 maxFiles 个
  │
  ├─> 并行读取文件
  │   └─ Promise.all(files.map(generateFileAttachment))
  │
  └─> Token 预算过滤
      └─ 累计 token 不超过 POST_COMPACT_TOKEN_BUDGET
```

#### 5. 部分压缩流程 (`partialCompactConversation`)

```
partialCompactConversation(allMessages, pivotIndex, context, cacheSafeParams, userFeedback, direction)
  │
  ├─> 消息分割
  │   ├─ direction='up_to': messagesToSummarize = slice(0, pivotIndex)
  │   │                    messagesToKeep = slice(pivotIndex) + 过滤
  │   └─ direction='from': messagesToSummarize = slice(pivotIndex)
  │                       messagesToKeep = slice(0, pivotIndex) + 过滤
  │
  ├─> 缓存策略
  │   ├─ 'up_to': 发送 messagesToSummarize（前缀缓存命中）
  │   └─ 'from': 发送 allMessages（牺牲缓存）
  │
  ├─> 摘要生成（同完整压缩）
  │
  ├─> 附件生成（差异：diff against messagesToKeep）
  │
  ├─> 边界标记锚点选择
  │   ├─ 'up_to': anchor = 最后一条摘要消息
  │   └─ 'from': anchor = 边界标记本身
  │
  └─> 返回 CompactionResult（包含 messagesToKeep）
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/utils/messages.js` | 消息创建、规范化、边界检测 |
| `src/utils/forkedAgent.js` | `runForkedAgent` |
| `src/services/api/claude.js` | `queryModelWithStreaming` |
| `src/utils/attachments.js` | 附件生成工具 |
| `src/utils/hooks.js` | Pre/Post Compact Hooks |
| `src/utils/sessionStart.js` | `processSessionStartHooks` |
| `src/utils/tokens.js` | Token 计数 |
| `src/utils/contextAnalysis.js` | `analyzeContext`（埋点） |
| `src/services/compact/grouping.js` | `groupMessagesByApiRound` |
| `src/services/compact/prompt.js` | 摘要提示模板 |

### 外部调用方

| 调用方 | 路径 | 用途 |
|-------|------|------|
| `autoCompact.ts` | `src/services/compact/autoCompact.ts` | 自动压缩调用 |
| `compact.ts` (command) | `src/commands/compact/compact.ts` | 手动 /compact 命令 |
| `inProcessRunner.ts` | `src/utils/swarm/inProcessRunner.ts` | Swarm 压缩 |
| `context.tsx` | `src/commands/context/context.tsx` | 上下文分析 |

### 关键调用链

```
compactConversation
  ├─> streamCompactSummary
  │   ├─> runForkedAgent (优先)
  │   │   ├─> src/utils/forkedAgent.ts:runForkedAgent
  │   │   └─> src/services/api/claude.ts:query (内部)
  │   │
  │   └─> queryModelWithStreaming (回退)
  │       └─> src/services/api/claude.ts:queryModelWithStreaming
  │
  ├─> createPostCompactFileAttachments
  │   └─> src/utils/attachments.ts:generateFileAttachment
  │
  ├─> createSkillAttachmentIfNeeded
  │   └─> src/bootstrap/state.ts:getInvokedSkillsForAgent
  │
  ├─> executePreCompactHooks / executePostCompactHooks
  │   └─> src/utils/hooks.ts
  │
  └─> processSessionStartHooks
      └─> src/utils/sessionStart.ts
```

## 依赖与外部交互

### Feature Flags

| Flag | 用途 |
|------|------|
| `KAIROS` | 会话转录功能 |
| `PROMPT_CACHE_BREAK_DETECTION` | 缓存断点检测通知 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索附件过滤 |
| `PROACTIVE` / `KAIROS` | 主动模式摘要后处理 |

### GrowthBook Flags

| Flag | 用途 |
|------|------|
| `tengu_compact_cache_prefix` | 控制缓存共享路径启用（默认 true） |
| `tengu_compact_streaming_retry` | 控制流式路径重试 |

### 环境变量

| 变量 | 说明 |
|------|------|
| `USER_TYPE` | 影响埋点详细程度 |

## 风险、边界与改进建议

### 已知风险

1. **Forked Agent 失败率高**:
   - Sonnet 4.6+ adaptive-thinking 模型在 Forked Agent 路径中工具调用率达 2.79%
   - 已通过 `NO_TOOLS_PREAMBLE` 缓解，但仍存在回退开销

2. **PTL 重试的精度问题**:
   - `roughTokenCountEstimationForMessages` 是粗略估算
   - 可能低估或高估实际需要丢弃的 token 数

3. **图片剥离的副作用**:
   - 摘要中 `[image]` / `[document]` 标记丢失原始媒体上下文
   - 可能影响摘要质量（如对话涉及图像内容分析）

4. **SessionStart Hooks 重复执行**:
   - 压缩后执行 SessionStart Hooks 可能导致某些初始化逻辑重复运行

### 边界条件

| 场景 | 处理 |
|------|------|
| 摘要为空 | 抛出 "no_summary" 错误，记录埋点 |
| 摘要以 API 错误开头 | 抛出错误，记录 "api_error" 埋点 |
| PTL 重试耗尽 | 抛出 ERROR_MESSAGE_PROMPT_TOO_LONG |
| 无消息可压缩 | 抛出 ERROR_MESSAGE_NOT_ENOUGH_MESSAGES |
| 用户中断 (ESC) | 抛出 ERROR_MESSAGE_USER_ABORT |
| 流式响应不完整 | 抛出 ERROR_MESSAGE_INCOMPLETE_RESPONSE |
| 部分压缩无内容可摘要 | 根据方向抛出对应错误消息 |

### 改进建议

1. **摘要质量评估**:
   - 添加摘要质量评分机制
   - 低质量摘要触发警告或重新生成

2. **PTL 重试优化**:
   - 使用更精确的 token 估算
   - 考虑模型特定的 token 计算差异

3. **Forked Agent 路径优化**:
   - 针对 adaptive-thinking 模型优化提示
   - 考虑完全禁用 thinking 配置以消除不匹配

4. **附件预算动态分配**:
   - 当前固定预算可能导致某些场景附件不足
   - 建议基于压缩后剩余空间动态调整

5. **部分压缩缓存优化**:
   - `from` 方向当前完全牺牲缓存
   - 可考虑智能选择分割点以最大化缓存命中

6. **错误恢复增强**:
   - 区分可恢复错误和永久失败
   - 添加更细粒度的重试策略
