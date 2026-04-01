# slackChannelSuggestions.ts 深度研究文档

## 场景与职责

`slackChannelSuggestions.ts` 是 Claude Code CLI 的 **Slack 频道自动补全模块**，通过与 Slack MCP (Model Context Protocol) 服务器集成，提供 Slack 频道名称的智能提示功能。该模块在用户输入 `#channel-name` 格式时提供频道补全建议：

1. **频道搜索**：通过 MCP 服务器搜索匹配的 Slack 频道
2. **智能缓存**：使用前缀匹配缓存策略减少 MCP 调用
3. **频道高亮**：在文本中定位已知的 Slack 频道名称用于 UI 高亮
4. **去抖动查询**：处理部分输入（如 `claude-code-te`）以优化 MCP 查询

该模块是 Claude Code 与 Slack 集成的关键组件，提升用户在提及 Slack 频道时的体验。

## 功能点目的

### 1. Slack 频道补全 (`getSlackChannelSuggestions`)
- **目的**：根据用户输入的搜索 token 返回匹配的 Slack 频道列表
- **智能查询处理**：
  - 去除尾部部分词（如 `claude-code-te` → `claude-code`）
  - 利用缓存前缀匹配避免重复查询
  - 合并进行中的查询（避免重复 MCP 调用）
- **缓存策略**：
  - 使用普通 Map（非 LRU）支持前缀迭代
  - 缓存大小限制 50 条目
  - 缓存复用：输入 `c` → `cl` → `cla` 可复用 `c` 的缓存结果

### 2. 频道获取 (`fetchChannels`)
- **目的**：通过 MCP 服务器调用 Slack 搜索工具
- **工具调用**：`slack_search_channels`
- **参数**：
  - `query`: 搜索关键词
  - `limit`: 20
  - `channel_types`: `public_channel,private_channel`
- **超时**：5 秒

### 3. 频道解析 (`parseChannels`)
- **目的**：从 Slack MCP 返回的 markdown 格式中提取频道名称
- **解析规则**：匹配 `Name: #channel-name` 格式的行
- **验证**：频道名必须符合 Slack 命名规范（小写字母、数字、下划线、连字符，1-80 字符）

### 4. 频道高亮 (`findSlackChannelPositions`)
- **目的**：在文本中定位已知的 Slack 频道用于语法高亮
- **匹配模式**：`#channel-name`（必须符合 Slack 命名规范）
- **过滤**：仅高亮已确认存在的频道（存在于 `knownChannels` 集合）

### 5. 已知频道管理
- **目的**：维护所有从 MCP 返回过的频道名称集合
- **用途**：
  - 高亮过滤（避免高亮不存在的频道）
  - 信号通知（频道列表变更时通知 UI 更新）

## 具体技术实现

### 关键数据结构

```typescript
// 缓存结构（普通 Map，支持前缀迭代）
const cache = new Map<string, string[]>()  // query -> channels[]

// 已知频道集合（用于高亮过滤）
const knownChannels = new Set<string>()
let knownChannelsVersion = 0
const knownChannelsChanged = createSignal()

// 进行中的查询（去重）
let inflightQuery: string | null = null
let inflightPromise: Promise<string[]> | null = null

// MCP 工具名
const SLACK_SEARCH_TOOL = 'slack_search_channels'

// 响应信封解析（Slack MCP 返回 JSON 包裹的 markdown）
const resultsEnvelopeSchema = z.object({ results: z.string() })
```

### 核心算法流程

#### 1. 频道建议获取流程
```
getSlackChannelSuggestions(clients, searchToken)
├── 空检查
│   └── searchToken 为空 → 返回 []
├── 查询预处理
│   └── mcpQuery = mcpQueryFor(searchToken)  // 去除尾部部分词
├── 缓存查找
│   ├── 精确匹配 cache.get(mcpQuery)
│   └── 前缀匹配 findReusableCacheEntry(mcpQuery, searchToken)
├── 缓存命中？
│   ├── 是 → 使用缓存结果
│   └── 否 → 发起 MCP 查询
│       ├── 检查进行中的相同查询
│       │   └── 复用 inflightPromise
│       └── 新查询
│           ├── 设置 inflightQuery / inflightPromise
│           ├── 调用 fetchChannels()
│           ├── 更新缓存
│           ├── 更新 knownChannels
│           ├── 触发信号通知
│           └── 清理 inflight 状态
└── 过滤和格式化结果
    ├── 按 searchToken 过滤（前缀匹配）
    ├── 排序
    ├── 限制 10 条
    └── 格式化为 SuggestionItem
```

#### 2. 智能 MCP 查询构建
```typescript
// Slack 搜索按连字符和下划线分词
// "claude-code-team-en" 搜索 0 结果（"en" 是部分词）
// 策略：去除尾部部分词
function mcpQueryFor(searchToken: string): string {
  const lastSep = Math.max(
    searchToken.lastIndexOf('-'),
    searchToken.lastIndexOf('_'),
  )
  return lastSep > 0 ? searchToken.slice(0, lastSep) : searchToken
}

// 示例：
// "claude-code-te" → "claude-code"
// "general" → "general"
```

#### 3. 缓存前缀匹配
```typescript
function findReusableCacheEntry(mcpQuery: string, searchToken: string): string[] | undefined {
  let best: string[] | undefined
  let bestLen = 0
  
  // 遍历所有缓存条目
  for (const [key, channels] of cache) {
    // 检查：mcpQuery 是否以 key 开头（key 是前缀）
    // 且 key 比当前最佳更长（更具体）
    // 且缓存结果包含匹配 searchToken 的频道
    if (
      mcpQuery.startsWith(key) &&
      key.length > bestLen &&
      channels.some(c => c.startsWith(searchToken))
    ) {
      best = channels
      bestLen = key.length
    }
  }
  return best
}

// 示例：
// 缓存: { "c": ["claude-code", "general"] }
// 输入: "cl" → mcpQuery = "cl"
// "cl".startsWith("c") = true，复用 "c" 的缓存
```

#### 4. 频道解析
```typescript
// Slack MCP 返回格式：
// {"results":"# Search Results...\nName: #claude-code\nName: #general\n..."}
// 或纯 markdown：
// "# Search Results...\nName: #claude-code\n..."

function parseChannels(text: string): string[] {
  const channels: string[] = []
  const seen = new Set<string>()
  
  // 正则：匹配 "Name: #channel-name" 格式
  // 频道名规范：小写字母开头，后跟字母数字下划线连字符，1-80 字符
  for (const line of text.split('\n')) {
    const m = line.match(/^Name:\s*#?([a-z0-9][a-z0-9_-]{0,79})\s*$/)
    if (m && !seen.has(m[1]!)) {
      seen.add(m[1]!)
      channels.push(m[1]!)
    }
  }
  return channels
}
```

### 关键代码路径

| 功能 | 函数 | 行号 |
|------|------|------|
| 频道建议获取 | `getSlackChannelSuggestions` | 159-201 |
| MCP 查询构建 | `mcpQueryFor` | 129-135 |
| 缓存前缀匹配 | `findReusableCacheEntry` | 140-157 |
| 频道获取 | `fetchChannels` | 29-65 |
| 响应解析 | `parseChannels` | 87-100 |
| 响应解包 | `unwrapResults` | 73-83 |
| 频道高亮 | `findSlackChannelPositions` | 110-122 |
| Slack 客户端检测 | `hasSlackMcpServer` | 102-104 |
| 缓存清除 | `clearSlackChannelCache` | 203-209 |

## 依赖与外部交互

### 直接依赖模块

```typescript
import { z } from 'zod'  // Schema 验证
import type { SuggestionItem } from '../../components/PromptInput/PromptInputFooterSuggestions.js'  // UI 类型
import type { MCPServerConnection } from '../../services/mcp/types.js'  // MCP 类型
import { logForDebugging } from '../debug.js'  // 调试日志
import { lazySchema } from '../lazySchema.js'  // 延迟 Schema 构造
import { createSignal } from '../signal.js'  // 信号通知
import { jsonParse } from '../slowOperations.js'  // JSON 解析
```

### 依赖详解

1. **zod** (外部库)
   - 用途：Schema 验证和类型安全
   - 用于解析 Slack MCP 返回的 JSON 信封

2. **MCP Types** (`services/mcp/types.ts`)
   - `MCPServerConnection`: MCP 服务器连接类型
   - `ConnectedMCPServer`: 已连接的服务器结构
   - `client.callTool()`: 调用 MCP 工具

3. **signal.ts** (内部模块)
   - `createSignal()`: 创建事件信号
   - 用于通知 UI 已知频道列表变更

4. **lazySchema.ts** (内部模块)
   - `lazySchema()`: 延迟 Schema 构造
   - 避免模块加载时的性能开销

5. **slowOperations.ts** (内部模块)
   - `jsonParse()`: 带性能监控的 JSON 解析

6. **debug.ts** (内部模块)
   - `logForDebugging()`: 调试日志记录

### 调用方

- `useTypeahead.tsx`: 类型提示钩子，检测 `#` 前缀并调用补全
- `PromptInput.tsx`: 输入处理，频道高亮显示
- 其他需要检测 Slack 频道的组件

### MCP 交互

```typescript
// MCP 工具调用
const result = await slackClient.client.callTool(
  {
    name: SLACK_SEARCH_TOOL,
    arguments: {
      query,
      limit: 20,
      channel_types: 'public_channel,private_channel',
    },
  },
  undefined,
  { timeout: 5000 },
)

// 响应结构
interface ToolResult {
  content: Array<{
    type: 'text'
    text: string
  }>
}
```

## 风险、边界与改进建议

### 已知风险

1. **MCP 依赖**
   - 功能完全依赖 Slack MCP 服务器可用
   - **风险**：MCP 服务器断开或超时时无补全
   - **当前处理**：try-catch 捕获，返回空数组

2. **缓存膨胀**
   - 缓存使用普通 Map，无自动过期
   - **当前处理**：限制 50 条目，FIFO 淘汰
   - **风险**：长期运行可能积累过时数据

3. **频道重命名**
   - Slack 频道重命名后，缓存和 knownChannels 中的旧名称不会自动更新
   - **影响**：可能高亮或建议不存在的频道

4. **权限问题**
   - 私有频道需要适当的 Slack 权限
   - **风险**：搜索结果可能不完整

5. **网络延迟**
   - MCP 调用涉及网络请求（5 秒超时）
   - **影响**：补全可能有明显延迟

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 无 Slack MCP 服务器 | `hasSlackMcpServer` 返回 false，不启用功能 |
| MCP 调用超时 | 捕获错误，返回空数组 |
| MCP 返回非 JSON | `unwrapResults` 回退到原始文本 |
| 空搜索 token | 返回空数组 |
| 缓存满（50 条目） | 删除最早条目（FIFO） |
| 频道名不符合规范 | 正则过滤，不添加到结果 |
| 重复频道名 | Set 去重 |
| 并发查询 | 复用 `inflightPromise` |

### 改进建议

1. **缓存优化**
   - 添加 TTL 过期机制
   - 实现 LRU 淘汰（当前是 FIFO）
   - 添加缓存统计和监控

2. **错误处理**
   - 区分网络错误和权限错误
   - 添加重试机制（指数退避）
   - 用户友好的错误提示

3. **功能增强**
   - 支持私有频道的特殊标识
   - 添加频道成员数量显示
   - 支持频道描述搜索
   - 最近使用的频道优先

4. **性能优化**
   - 预加载常用频道
   - 客户端缓存持久化（localStorage）
   - 批量查询多个前缀

5. **可配置性**
   - 可配置的缓存大小和 TTL
   - 可配置的搜索限制
   - 排除特定频道的选项

6. **安全考虑**
   - 敏感频道名称的访问控制
   - 频道列表的加密存储
   - 审计日志记录

### 测试要点

- MCP 服务器可用/不可用场景
- 网络超时和错误处理
- 缓存命中/未命中/过期行为
- 前缀匹配的正确性
- 频道名解析的边界情况
- 并发查询的去重
- 频道高亮的准确性
- 内存泄漏检查（Map 增长）

### 相关代码参考

- `services/mcp/`: MCP 客户端实现
- `hooks/useTypeahead.tsx`: 类型提示集成
- `components/PromptInput/PromptInput.tsx`: 输入处理和高亮
- `services/mcp/types.ts`: MCP 类型定义
