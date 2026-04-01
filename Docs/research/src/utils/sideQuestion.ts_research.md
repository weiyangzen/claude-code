# sideQuestion.ts 深度研究

## 场景与职责

`sideQuestion.ts` 实现 **"/btw"（By The Way）功能**，允许用户在主对话进行中快速提问，而不中断主 Agent 的上下文。

**核心职责：**
1. 检测和处理 `/btw` 命令
2. 使用 forked agent 执行旁路查询
3. 隔离旁路响应与主对话
4. 处理各种响应和错误情况

**应用场景：**
- 用户想询问与当前任务相关但不打断主流程的问题
- 快速澄清概念或语法
- 询问实现建议而不影响主 Agent 的执行状态

---

## 功能点目的

### 1. /btw 触发检测
```typescript
const BTW_PATTERN = /^\/btw\b/gi

export function findBtwTriggerPositions(text: string): Array<{
  word: string
  start: number
  end: number
}>
```

**特点：**
- 大小写不敏感（`gi` 标志）
- 词边界匹配（`\b`）
- 仅匹配行首（`^`）
- 返回匹配位置用于 UI 高亮

### 2. 旁路问题执行
```typescript
export type SideQuestionResult = {
  response: string | null
  usage: NonNullableUsage
}

export async function runSideQuestion({
  question,
  cacheSafeParams,
}: {
  question: string
  cacheSafeParams: CacheSafeParams
}): Promise<SideQuestionResult>
```

**核心约束：**
- 无工具可用（`canUseTool` 始终拒绝）
- 单轮响应（`maxTurns: 1`）
- 跳过缓存写入（`skipCacheWrite: true`）
- 继承主线程的思考配置（保持缓存键一致）

### 3. 系统提示包装
```typescript
const wrappedQuestion = `<system-reminder>This is a side question from the user...

IMPORTANT CONTEXT:
- You are a separate, lightweight agent spawned to answer this one question
- The main agent is NOT interrupted...

CRITICAL CONSTRAINTS:
- You have NO tools available...
- This is a one-off response...

Simply answer the question with the information you have.</system-reminder>

${question}`
```

**设计意图：**
- 明确告知模型这是一个旁路问题
- 强调主 Agent 未被中断
- 禁止工具使用和后续行动承诺

### 4. 响应提取
```typescript
function extractSideQuestionResponse(messages: Message[]): string | null
```

**处理逻辑：**
1. 展平所有 Assistant 消息的内容块
2. 提取并连接所有文本块
3. 检测工具使用尝试（模型违规）
4. 检测 API 错误（`api_error` 系统消息）
5. 返回适当的错误提示或 null

---

## 具体技术实现

### Forked Agent 调用
```typescript
const agentResult = await runForkedAgent({
  promptMessages: [createUserMessage({ content: wrappedQuestion })],
  cacheSafeParams,
  canUseTool: async () => ({
    behavior: 'deny' as const,
    message: 'Side questions cannot use tools',
    decisionReason: { type: 'other' as const, reason: 'side_question' },
  }),
  querySource: 'side_question',
  forkLabel: 'side_question',
  maxTurns: 1,
  skipCacheWrite: true,
})
```

**关键决策：**
- 不覆盖 `thinkingConfig`：思考配置是 API 缓存键的一部分，覆盖会破坏提示缓存
- 使用 `runForkedAgent`：共享父上下文的提示缓存

### 响应处理细节
```typescript
// 展平所有 Assistant 内容块
const assistantBlocks = messages.flatMap(m =>
  m.type === 'assistant' ? m.message.content : [],
)

// 处理思考块 + 文本块的情况
const text = extractTextContent(assistantBlocks, '\n\n').trim()

// 检测工具使用违规
const toolUse = assistantBlocks.find(b => b.type === 'tool_use')
if (toolUse) {
  return `(The model tried to call ${toolName} instead of answering directly...)`
}

// 检测 API 错误
const apiErr = messages.find(
  (m): m is SystemAPIErrorMessage =>
    m.type === 'system' && 'subtype' in m && m.subtype === 'api_error',
)
if (apiErr) {
  return `(API error: ${formatAPIError(apiErr.error)})`
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `BTW_PATTERN` | /btw 匹配正则 |
| `findBtwTriggerPositions` | 查找触发位置 |
| `SideQuestionResult` | 结果类型 |
| `runSideQuestion` | 执行旁路问题 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `../services/api/errorUtils.js` | API 错误格式化 |
| `../services/api/logging.js` | `NonNullableUsage` 类型 |
| `../types/message.js` | 消息类型 |
| `./forkedAgent.js` | `runForkedAgent`, `CacheSafeParams` |
| `./messages.js` | `createUserMessage`, `extractTextContent` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/cli/print.ts` | SDK 打印处理 |
| `src/commands/btw/btw.tsx` | /btw 命令 |
| `src/utils/queryContext.ts` | 查询上下文 |
| `src/components/PromptInput/PromptInput.tsx` | 输入处理 |

---

## 依赖与外部交互

### 外部依赖
- **Anthropic API**：通过 `runForkedAgent` 间接调用

### 内部依赖
| 模块 | 用途 |
|------|------|
| `forkedAgent.js` | Forked agent 执行 |
| `messages.js` | 消息创建和内容提取 |

---

## 风险、边界与改进建议

### 已知风险

1. **模型违规**
   - 模型可能尝试使用工具（尽管系统提示禁止）
   - 缓解：检测工具使用并返回友好错误提示

2. **响应质量**
   - 无工具访问限制了解答能力
   - 模型可能无法回答需要文件/网络查询的问题

3. **缓存污染**
   - 旁路问题内容进入 forked agent 上下文
   - 不影响主对话，但可能影响后续旁路问题

### 边界情况

| 场景 | 处理 |
|------|------|
| 空问题 | 由调用方处理 |
| 模型返回思考块 | 正确提取后续文本块 |
| API 错误耗尽重试 | 返回格式化错误信息 |
| 模型尝试工具使用 | 返回违规提示 |
| 无 Assistant 消息 | 返回 null |

### 改进建议

1. **增强约束**
   - 添加 Token 预算限制
   - 实现响应长度限制

2. **上下文管理**
   - 允许选择性包含主对话片段
   - 支持引用特定消息

3. **UI 集成**
   - 更好的响应展示（与主对话区分）
   - 支持响应折叠/展开

4. **历史记录**
   - 保存旁路问题历史
   - 支持回顾和重新提问

5. **智能检测**
   - 自动检测适合旁路的问题类型
   - 建议用户使用 /btw
