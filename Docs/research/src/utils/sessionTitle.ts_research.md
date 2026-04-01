# sessionTitle.ts 深度研究

## 场景与职责

`sessionTitle.ts` 是 Claude Code CLI 中**会话标题生成的单一可信源**。它使用 Anthropic 的 Haiku 模型通过 API 调用生成简洁、描述性的会话标题。

**设计背景：**
- 之前存在多个独立的标题生成实现（`teleport.tsx`、`rename/generateSessionName.ts`）
- 为避免代码重复和依赖链膨胀（特别是避免引入 React/chalk/git 依赖），提取此独立模块
- 可被 SDK 控制请求处理器 (`print.ts`) 安全导入

**核心职责：**
1. 从消息历史中提取对话文本
2. 调用 Haiku 模型生成 3-7 词的句子式标题
3. 提供标准化的 JSON Schema 输出格式

---

## 功能点目的

### 1. 对话文本提取
```typescript
const MAX_CONVERSATION_TEXT = 1000
export function extractConversationText(messages: Message[]): string
```

**处理逻辑：**
- 仅提取 `type === 'user'` 或 `type === 'assistant'` 的消息
- 跳过 `isMeta` 标记的消息
- 跳过非人类来源的消息 (`origin.kind !== 'human'`)
- 支持字符串内容和数组内容块
- 尾部切片保留最近 1000 字符（优先近期上下文）

**应用场景：**
- 会话首次创建时根据首条消息生成标题
- `/rename` 命令根据对话内容建议新名称

### 2. 会话标题生成
```typescript
export async function generateSessionTitle(
  description: string,
  signal: AbortSignal
): Promise<string | null>
```

**系统提示词 (`SESSION_TITLE_PROMPT`)：**
- 要求生成 3-7 词的句子式标题
- 使用句子大小写（仅首词和专有名词大写）
- 提供正反示例引导模型输出

**输出格式：**
```json
{"title": "Fix login button on mobile"}
```

**错误处理：**
- 空输入返回 `null`
- API 错误或解析失败返回 `null`
- 记录 `tengu_session_title_generated` 分析事件

---

## 具体技术实现

### 文本提取算法
```typescript
for (const msg of messages) {
  if (msg.type !== 'user' && msg.type !== 'assistant') continue
  if ('isMeta' in msg && msg.isMeta) continue
  if ('origin' in msg && msg.origin && msg.origin.kind !== 'human') continue
  // 提取文本...
}
// 尾部切片
return text.length > MAX_CONVERSATION_TEXT
  ? text.slice(-MAX_CONVERSATION_TEXT)
  : text
```

### API 调用参数
```typescript
await queryHaiku({
  systemPrompt: asSystemPrompt([SESSION_TITLE_PROMPT]),
  userPrompt: trimmed,
  outputFormat: {
    type: 'json_schema',
    schema: {
      type: 'object',
      properties: { title: { type: 'string' } },
      required: ['title'],
      additionalProperties: false,
    },
  },
  signal,
  options: {
    querySource: 'generate_session_title',
    agents: [],
    isNonInteractiveSession: getIsNonInteractiveSession(),
    hasAppendSystemPrompt: false,
    mcpTools: [],
  },
})
```

### Schema 验证
```typescript
const titleSchema = lazySchema(() => z.object({ title: z.string() }))
const parsed = titleSchema().safeParse(safeParseJSON(text))
const title = parsed.success ? parsed.data.title.trim() || null : null
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `extractConversationText` | 从消息提取对话文本 |
| `generateSessionTitle` | 生成会话标题 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `zod/v4` | Schema 验证 |
| `../bootstrap/state.js` | `getIsNonInteractiveSession` |
| `../services/analytics/index.js` | `logEvent` |
| `../services/api/claude.js` | `queryHaiku` |
| `../types/message.js` | `Message` 类型 |
| `./debug.js` | `logForDebugging` |
| `./json.js` | `safeParseJSON` |
| `./lazySchema.js` | 延迟加载 Schema |
| `./messages.js` | `extractTextContent` |
| `./systemPromptType.js` | `asSystemPrompt` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/cli/print.ts` | SDK 控制请求处理 |
| `src/hooks/useRemoteSession.ts` | 远程会话标题生成 |
| `src/screens/REPL.tsx` | REPL 会话标题 |
| `src/utils/teleport.tsx` | CCR 标题和分支生成（向后兼容） |
| `src/commands/rename/generateSessionName.ts` | /rename 命令（向后兼容） |
| `src/bridge/initReplBridge.ts` | REPL Bridge 初始化 |

---

## 依赖与外部交互

### 外部 API
- **Anthropic API**：通过 `queryHaiku` 调用 Haiku 模型

### 分析事件
```typescript
logEvent('tengu_session_title_generated', { success: title !== null })
```

### 环境感知
- 检测交互式/非交互式会话模式
- 影响 API 调用的元数据报告

---

## 风险、边界与改进建议

### 已知风险

1. **API 依赖**
   - 标题生成依赖外部 API 可用性
   - 网络故障或限流导致标题生成失败（返回 null）

2. **模型输出不确定性**
   - 即使使用 JSON Schema，模型可能输出不符合格式的内容
   - Schema 验证失败时优雅降级为 null

3. **长对话截断**
   - 仅使用最近 1000 字符，可能丢失早期关键上下文
   - 对于多主题会话，标题可能仅反映最新主题

### 边界情况

| 场景 | 处理 |
|------|------|
| 空描述 | 提前返回 null |
| API 超时 | AbortSignal 取消，返回 null |
| JSON 解析失败 | `safeParseJSON` 返回 null，Schema 验证失败 |
| 空标题字符串 | 视为 null |
| 标题超长 | 依赖模型遵循提示词限制 |

### 改进建议

1. **缓存优化**
   - 添加本地缓存避免重复生成相同对话的标题
   - 缓存键：消息内容的哈希

2. **降级策略**
   - 实现基于规则的标题生成作为 API 失败时的回退
   - 使用首条消息的前 N 个字符作为应急标题

3. **多语言支持**
   - 当前提示词为英文，可能生成英文标题描述非英文对话
   - 考虑检测对话语言并调整提示词

4. **个性化**
   - 支持用户自定义标题风格偏好（正式/ casual/技术）
   - 通过设置调整标题长度限制

5. **性能**
   - 考虑客户端缓存已生成的标题
   - 延迟加载（仅在需要显示时生成）
