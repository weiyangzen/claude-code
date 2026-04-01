# agenticSessionSearch.ts 深度研究文档

## 场景与职责

`agenticSessionSearch.ts` 是 Claude Code CLI 中用于**智能会话搜索**的模块。它利用 Claude AI 的语义理解能力，帮助用户基于自然语言查询找到相关的历史会话，而不仅仅是简单的关键词匹配。

### 核心场景

1. **Resume 命令智能搜索**：当用户使用 `/resume` 命令时，基于查询找到最相关的历史会话
2. **语义理解匹配**：理解查询的语义，匹配相关但不完全相同的概念（如 "testing" 匹配 "tests", "unit tests", "QA" 等）
3. **多字段综合评分**：综合考虑标签、标题、分支、摘要、对话内容等多个字段
4. **渐进式加载优化**：支持从精简日志（lite logs）加载完整内容以获取对话记录

### 设计原则

模块实现体现了以下设计原则：
- **包容性匹配**：宁可返回过多结果，也不要遗漏相关会话
- **标签优先**：用户明确设置的标签具有最高优先级
- **语义相关性**：不仅匹配关键词，还理解概念间的关联

---

## 功能点目的

### 1. 智能会话搜索

`agenticSessionSearch()` 是核心函数，实现以下功能：
- 基于用户查询和自然语言理解找到相关会话
- 返回按相关性排序的会话列表

### 2. 多字段搜索

搜索范围包括：
- 标签（最高优先级 - 用户明确分类）
- 标题（自定义标题或首条消息）
- 分支名称
- 摘要（AI 生成的会话摘要）
- 首条消息内容
- 对话记录摘录

### 3. 预过滤优化

在调用 AI 之前进行本地预过滤：
- 快速筛选包含查询词的会话
- 减少发送给 AI 的数据量
- 如果匹配不足，补充最近的不匹配会话作为上下文

### 4. 对话内容提取

从会话消息中提取可搜索的文本内容：
- 处理字符串和数组格式的消息内容
- 支持多种内容块类型（text 等）
- 限制提取的消息数量和字符数

---

## 具体技术实现

### 核心常量配置

```typescript
// 对话记录提取限制
const MAX_TRANSCRIPT_CHARS = 2000      // 每个会话最大字符数
const MAX_MESSAGES_TO_SCAN = 100       // 最多扫描的消息数
const MAX_SESSIONS_TO_SEARCH = 100     // 发送给 API 的最大会话数
```

### AI 系统提示词

```typescript
const SESSION_SEARCH_SYSTEM_PROMPT = `Your goal is to find relevant sessions based on a user's search query.

You will be given a list of sessions with their metadata and a search query. Identify which sessions are most relevant to the query.

Each session may include:
- Title (display name or custom title)
- Tag (user-assigned category, shown as [tag: name] - users tag sessions with /tag command to categorize them)
- Branch (git branch name, shown as [branch: name])
- Summary (AI-generated summary)
- First message (beginning of the conversation)
- Transcript (excerpt of conversation content)

IMPORTANT: Tags are user-assigned labels that indicate the session's topic or category. If the query matches a tag exactly or partially, those sessions should be highly prioritized.

For each session, consider (in order of priority):
1. Exact tag matches (highest priority - user explicitly categorized this session)
2. Partial tag matches or tag-related terms
3. Title matches (custom titles or first message content)
4. Branch name matches
5. Summary and transcript content matches
6. Semantic similarity and related concepts

CRITICAL: Be VERY inclusive in your matching. Include sessions that:
- Contain the query term anywhere in any field
- Are semantically related to the query (e.g., "testing" matches sessions about "tests", "unit tests", "QA", etc.)
- Discuss topics that could be related to the query
- Have transcripts that mention the concept even in passing

When in doubt, INCLUDE the session. It's better to return too many results than too few. The user can easily scan through results, but missing relevant sessions is frustrating.

Return sessions ordered by relevance (most relevant first). If truly no sessions have ANY connection to the query, return an empty array - but this should be rare.

Respond with ONLY the JSON object, no markdown formatting:
{"relevant_indices": [2, 5, 0]}`
```

### 核心数据结构

```typescript
type AgenticSearchResult = {
  relevant_indices: number[]  // 相关会话的索引数组
}

type LogOption = {
  date: string
  messages: SerializedMessage[]
  fullPath?: string
  // ... 其他字段
  customTitle?: string
  tag?: string
  gitBranch?: string
  summary?: string
  firstPrompt: string
  // ...
}

type SerializedMessage = Message & {
  cwd: string
  userType: string
  sessionId: string
  timestamp: string
  version: string
  // ...
}
```

### 关键流程

#### 1. 消息文本提取
```typescript
function extractMessageText(message: SerializedMessage): string {
  if (message.type !== 'user' && message.type !== 'assistant') {
    return ''
  }

  const content = 'message' in message ? message.message?.content : undefined
  if (!content) return ''

  if (typeof content === 'string') {
    return content
  }

  if (Array.isArray(content)) {
    return content
      .map(block => {
        if (typeof block === 'string') return block
        if ('text' in block && typeof block.text === 'string') return block.text
        return ''
      })
      .filter(Boolean)
      .join(' ')
  }

  return ''
}
```

#### 2. 对话记录提取
```typescript
function extractTranscript(messages: SerializedMessage[]): string {
  if (messages.length === 0) return ''

  // 取开头和结尾的消息以获取上下文
  const messagesToScan =
    messages.length <= MAX_MESSAGES_TO_SCAN
      ? messages
      : [
          ...messages.slice(0, MAX_MESSAGES_TO_SCAN / 2),
          ...messages.slice(-MAX_MESSAGES_TO_SCAN / 2),
        ]

  const text = messagesToScan
    .map(extractMessageText)
    .filter(Boolean)
    .join(' ')
    .replace(/\s+/g, ' ')
    .trim()

  return text.length > MAX_TRANSCRIPT_CHARS
    ? text.slice(0, MAX_TRANSCRIPT_CHARS) + '…'
    : text
}
```

#### 3. 本地预过滤
```typescript
function logContainsQuery(log: LogOption, queryLower: string): boolean {
  // 检查标题
  const title = getLogDisplayTitle(log).toLowerCase()
  if (title.includes(queryLower)) return true

  // 检查自定义标题
  if (log.customTitle?.toLowerCase().includes(queryLower)) return true

  // 检查标签
  if (log.tag?.toLowerCase().includes(queryLower)) return true

  // 检查分支
  if (log.gitBranch?.toLowerCase().includes(queryLower)) return true

  // 检查摘要
  if (log.summary?.toLowerCase().includes(queryLower)) return true

  // 检查首条消息
  if (log.firstPrompt?.toLowerCase().includes(queryLower)) return true

  // 检查对话记录（最耗时，最后检查）
  if (log.messages && log.messages.length > 0) {
    const transcript = extractTranscript(log.messages).toLowerCase()
    if (transcript.includes(queryLower)) return true
  }

  return false
}
```

#### 4. 主搜索流程
```typescript
export async function agenticSessionSearch(
  query: string,
  logs: LogOption[],
  signal?: AbortSignal,
): Promise<LogOption[]> {
  if (!query.trim() || logs.length === 0) {
    return []
  }

  const queryLower = query.toLowerCase()

  // 1. 预过滤：找到包含查询词的会话
  const matchingLogs = logs.filter(log => logContainsQuery(log, queryLower))

  // 2. 组合会话列表（匹配优先，不足时补充最近会话）
  let logsToSearch: LogOption[]
  if (matchingLogs.length >= MAX_SESSIONS_TO_SEARCH) {
    logsToSearch = matchingLogs.slice(0, MAX_SESSIONS_TO_SEARCH)
  } else {
    const nonMatchingLogs = logs.filter(log => !logContainsQuery(log, queryLower))
    const remainingSlots = MAX_SESSIONS_TO_SEARCH - matchingLogs.length
    logsToSearch = [
      ...matchingLogs,
      ...nonMatchingLogs.slice(0, remainingSlots),
    ]
  }

  // 3. 为精简日志加载完整内容
  const logsWithTranscripts = await Promise.all(
    logsToSearch.map(async log => {
      if (isLiteLog(log)) {
        try {
          return await loadFullLog(log)
        } catch (error) {
          logError(error as Error)
          return log
        }
      }
      return log
    })
  )

  // 4. 构建提示词并调用 AI
  const sessionList = buildSessionList(logsWithTranscripts)
  const response = await sideQuery({
    model: getSmallFastModel(),
    system: SESSION_SEARCH_SYSTEM_PROMPT,
    messages: [{ role: 'user', content: sessionList }],
    signal,
    querySource: 'session_search',
  })

  // 5. 解析响应并返回相关会话
  const result = parseResponse(response)
  return result.relevant_indices
    .filter(index => index >= 0 && index < logsWithTranscripts.length)
    .map(index => logsWithTranscripts[index]!)
}
```

---

## 关键代码路径与文件引用

### 调用方（Consumers）

| 文件 | 用途 |
|------|------|
| `src/screens/ResumeConversation.tsx` | 恢复会话界面的搜索功能 |
| `src/commands/resume/resume.tsx` | `/resume` 命令的搜索实现 |

### 依赖模块

| 模块 | 用途 |
|------|------|
| `src/types/logs.ts` | `LogOption`, `SerializedMessage` 类型 |
| `src/utils/array.ts` | `count` 函数 |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/log.ts` | `getLogDisplayTitle`, `logError` |
| `src/utils/model/model.ts` | `getSmallFastModel` |
| `src/utils/sessionStorage.ts` | `isLiteLog`, `loadFullLog` |
| `src/utils/sideQuery.ts` | `sideQuery` - AI 查询封装 |
| `src/utils/slowOperations.ts` | `jsonParse` |

---

## 依赖与外部交互

### 直接依赖

```typescript
import type { LogOption, SerializedMessage } from '../types/logs.js'
import { count } from './array.js'
import { logForDebugging } from './debug.js'
import { getLogDisplayTitle, logError } from './log.js'
import { getSmallFastModel } from './model/model.js'
import { isLiteLog, loadFullLog } from './sessionStorage.js'
import { sideQuery } from './sideQuery.js'
import { jsonParse } from './slowOperations.js'
```

### AI 服务交互

- **使用模型**：`getSmallFastModel()`（通常是轻量级快速模型，如 Haiku）
- **查询封装**：`sideQuery()` 提供统一的 AI 查询接口
- **查询来源**：`'session_search'` 用于遥测追踪

### 数据流

```
用户查询
    ↓
本地预过滤（关键词匹配）
    ↓
加载精简日志的完整内容
    ↓
构建会话列表提示词
    ↓
sideQuery() → Claude API
    ↓
解析 JSON 响应
    ↓
返回相关会话索引
    ↓
映射回原始日志对象
```

---

## 风险、边界与改进建议

### 已知风险

1. **API 成本**
   - 每次搜索都需要调用 Claude API
   - 如果用户频繁搜索，可能产生较高成本
   - 缓解：使用轻量级模型（Haiku）和限制会话数量

2. **响应解析失败**
   - AI 可能返回非预期的格式
   - 代码已处理 JSON 提取失败的情况，返回空数组
   - 风险：搜索失败时用户体验下降

3. **隐私问题**
   - 会话内容（包括对话记录）被发送到 AI API
   - 虽然使用内部 API，但仍需考虑数据隐私

4. **性能问题**
   - 加载大量精简日志的完整内容可能耗时
   - 如果会话文件很大，`loadFullLog` 可能成为瓶颈

### 边界情况

1. **空查询**
   ```typescript
   if (!query.trim() || logs.length === 0) {
     return []
   }
   ```

2. **无匹配会话**
   - 预过滤后如果没有匹配，会补充最近的不匹配会话
   - AI 仍然有机会基于语义找到相关会话

3. **精简日志加载失败**
   ```typescript
   if (isLiteLog(log)) {
     try {
       return await loadFullLog(log)
     } catch (error) {
       logError(error as Error)
       return log  // 失败时使用精简日志（无对话记录）
     }
   }
   ```

4. **AI 响应解析失败**
   ```typescript
   const jsonMatch = textContent.text.match(/\{[\s\S]*\}/)
   if (!jsonMatch) {
     logForDebugging('Could not find JSON in agentic search response')
     return []
   }
   ```

5. **索引越界**
   ```typescript
   const relevantLogs = relevantIndices
     .filter(index => index >= 0 && index < logsWithTranscripts.length)
     .map(index => logsWithTranscripts[index]!)
   ```

### 改进建议

1. **添加缓存机制**
   ```typescript
   // 缓存搜索结果，避免重复查询
   const searchCache = new Map<string, LogOption[]>()
   
   export async function agenticSessionSearch(
     query: string,
     logs: LogOption[],
     signal?: AbortSignal,
   ): Promise<LogOption[]> {
     const cacheKey = `${query}:${logs.map(l => l.date).join(',')}`
     if (searchCache.has(cacheKey)) {
       return searchCache.get(cacheKey)!
     }
     // ... 执行搜索 ...
     searchCache.set(cacheKey, results)
     return results
   }
   ```

2. **添加搜索超时**
   ```typescript
   export async function agenticSessionSearch(
     query: string,
     logs: LogOption[],
     signal?: AbortSignal,
   ): Promise<LogOption[]> {
     const timeoutSignal = AbortSignal.timeout(10000) // 10秒超时
     const combinedSignal = signal 
       ? AbortSignal.any([signal, timeoutSignal])
       : timeoutSignal
     
     // ... 使用 combinedSignal ...
   }
   ```

3. **改进错误恢复**
   ```typescript
   try {
     // ... AI 查询 ...
   } catch (error) {
     // 降级到本地关键词搜索
     logForDebugging('AI search failed, falling back to keyword search')
     return logs.filter(log => logContainsQuery(log, queryLower))
   }
   ```

4. **添加相关性分数**
   ```typescript
   type SearchResult = {
     log: LogOption
     relevanceScore: number  // 0-1
     matchReasons: string[]   // 为什么匹配
   }
   ```

5. **支持高级查询语法**
   - 标签过滤：`tag:bugfix`
   - 分支过滤：`branch:main`
   - 日期范围：`after:2024-01-01`

6. **批量加载优化**
   ```typescript
   // 并行加载但有并发限制
   const CONCURRENCY_LIMIT = 5
   const logsWithTranscripts = await pMap(
     logsToSearch,
     async log => { /* ... */ },
     { concurrency: CONCURRENCY_LIMIT }
   )
   ```

7. **添加遥测**
   ```typescript
   logEvent('tengu_session_search', {
     queryLength: query.length,
     totalLogs: logs.length,
     matchingLogs: matchingLogs.length,
     resultsCount: relevantLogs.length,
     durationMs: Date.now() - startTime,
   })
   ```
