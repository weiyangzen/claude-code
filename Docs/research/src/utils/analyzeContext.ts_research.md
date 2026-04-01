# analyzeContext.ts 深度研究文档

## 场景与职责

`analyzeContext.ts` 是 Claude Code CLI 中用于**上下文窗口使用分析**的核心模块。它提供了详细的 Token 计数和可视化功能，帮助用户理解当前会话的上下文使用情况，包括系统提示词、工具定义、消息历史等各个组成部分的 Token 消耗。

### 核心场景

1. **`/context` 命令**：显示当前会话的上下文使用可视化界面
2. **自动压缩决策**：为自动压缩（autocompact）功能提供 Token 计数依据
3. **上下文建议**：为上下文优化建议提供数据支持
4. **工具搜索优化**：分析工具定义 Token 消耗，支持延迟加载决策
5. **调试和诊断**：帮助开发者和用户理解 Token 使用分布

### 设计原则

模块实现体现了以下设计原则：
- **准确性优先**：使用 Anthropic Token 计数 API 获取精确计数
- **降级容错**：API 失败时使用 Haiku 回退或本地估算
- **详细分解**：提供系统提示词、工具、消息等多维度的 Token 分解
- **可视化支持**：生成网格数据用于上下文可视化组件

---

## 功能点目的

### 1. Token 计数与分解

`analyzeContextUsage()` 是核心函数，提供：
- 系统提示词 Token 计数
- 内置工具定义 Token 计数
- MCP 工具 Token 计数
- 自定义代理定义 Token 计数
- 斜杠命令/技能 Token 计数
- 消息历史 Token 计数（详细分解）
- CLAUDE.md 内存文件 Token 计数

### 2. 工具延迟加载分析

- 区分"始终加载"和"延迟加载"的工具
- 计算已加载和延迟的工具 Token
- 支持工具搜索（Tool Search）功能的决策

### 3. 上下文可视化数据

生成用于 UI 展示的数据结构：
- 分类 Token 统计（系统提示词、工具、消息等）
- 网格可视化数据（颜色、填充状态、百分比）
- 预留缓冲区计算（自动压缩/手动压缩）

### 4. 消息分解

详细分析消息历史的 Token 分布：
- 工具调用 Token
- 工具结果 Token
- 附件 Token
- 助手消息 Token
- 用户消息 Token
- 按工具类型分解

---

## 具体技术实现

### 核心数据结构

```typescript
interface ContextData {
  readonly categories: ContextCategory[]
  readonly totalTokens: number
  readonly maxTokens: number
  readonly rawMaxTokens: number
  readonly percentage: number
  readonly gridRows: GridSquare[][]
  readonly model: string
  readonly memoryFiles: MemoryFile[]
  readonly mcpTools: McpTool[]
  readonly deferredBuiltinTools?: DeferredBuiltinTool[]  // Ant-only
  readonly systemTools?: SystemToolDetail[]              // Ant-only
  readonly systemPromptSections?: SystemPromptSectionDetail[]  // Ant-only
  readonly agents: Agent[]
  readonly slashCommands?: SlashCommandInfo
  readonly skills?: SkillInfo
  readonly autoCompactThreshold?: number
  readonly isAutoCompactEnabled: boolean
  messageBreakdown?: {
    toolCallTokens: number
    toolResultTokens: number
    attachmentTokens: number
    assistantMessageTokens: number
    userMessageTokens: number
    toolCallsByType: Array<{ name: string; callTokens: number; resultTokens: number }>
    attachmentsByType: Array<{ name: string; tokens: number }>
  }
  readonly apiUsage: {
    input_tokens: number
    output_tokens: number
    cache_creation_input_tokens: number
    cache_read_input_tokens: number
  } | null
}

interface ContextCategory {
  name: string
  tokens: number
  color: keyof Theme
  isDeferred?: boolean  // 延迟加载的 Token 不计入实际使用
}

interface GridSquare {
  color: keyof Theme
  isFilled: boolean
  categoryName: string
  tokens: number
  percentage: number
  squareFullness: number  // 0-1，表示方格的填充程度
}
```

### Token 计数策略

#### 1. 带降级的 Token 计数
```typescript
async function countTokensWithFallback(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
  tools: Anthropic.Beta.Messages.BetaToolUnion[],
): Promise<number | null> {
  try {
    // 1. 尝试使用官方 Token 计数 API
    const result = await countMessagesTokensWithAPI(messages, tools)
    if (result !== null) return result
  } catch (err) {
    logForDebugging(`countTokensWithFallback: API failed: ${errorMessage(err)}`)
    logError(err)
  }

  try {
    // 2. 降级到 Haiku 模型估算
    const fallbackResult = await countTokensViaHaikuFallback(messages, tools)
    return fallbackResult
  } catch (err) {
    logForDebugging(`countTokensWithFallback: haiku fallback failed`)
    return null
  }
}
```

#### 2. 工具 Token 计数开销处理
```typescript
export const TOOL_TOKEN_COUNT_OVERHEAD = 500
```
- API 在每次调用时添加约 500 Token 的工具提示前导
- 单独计数工具时需要减去此开销，避免重复计算

### 关键流程

#### 1. 系统提示词 Token 计数
```typescript
async function countSystemTokens(effectiveSystemPrompt: readonly string[]) {
  const systemContext = await getSystemContext()
  
  const namedEntries = [
    ...effectiveSystemPrompt
      .filter(content => content.length > 0 && content !== SYSTEM_PROMPT_DYNAMIC_BOUNDARY)
      .map(content => ({ name: extractSectionName(content), content })),
    ...Object.entries(systemContext)
      .filter(([, content]) => content.length > 0)
      .map(([name, content]) => ({ name, content })),
  ]

  const systemTokenCounts = await Promise.all(
    namedEntries.map(({ content }) => countTokensWithFallback([{ role: 'user', content }], []))
  )

  return {
    systemPromptTokens: systemTokenCounts.reduce((sum, tokens) => sum + (tokens || 0), 0),
    systemPromptSections: namedEntries.map((entry, i) => ({
      name: entry.name,
      tokens: systemTokenCounts[i] || 0,
    })),
  }
}
```

#### 2. 内置工具 Token 计数（支持延迟加载）
```typescript
async function countBuiltInToolTokens(tools, getToolPermissionContext, agentInfo, model, messages) {
  const builtInTools = tools.filter(tool => !tool.isMcp)
  
  // 检查工具搜索是否启用
  const { isToolSearchEnabled } = await import('./toolSearch.js')
  const { isDeferredTool } = await import('../tools/ToolSearchTool/prompt.js')
  const isDeferred = await isToolSearchEnabled(...)

  // 分离始终加载和延迟加载的工具
  const alwaysLoadedTools = builtInTools.filter(t => !isDeferredTool(t))
  const deferredBuiltinTools = builtInTools.filter(t => isDeferredTool(t))

  // 计算已加载的工具 Token
  const loadedToolNames = new Set<string>()
  if (messages) {
    // 从消息历史中找出已使用的延迟工具
    for (const msg of messages) {
      if (msg.type === 'assistant') {
        for (const block of msg.message.content) {
          if (block.type === 'tool_use' && deferredToolNameSet.has(block.name)) {
            loadedToolNames.add(block.name)
          }
        }
      }
    }
  }

  return {
    builtInToolTokens: alwaysLoadedTokens + loadedDeferredTokens,
    deferredBuiltinDetails,
    deferredBuiltinTokens: totalDeferredTokens - loadedDeferredTokens,
    systemToolDetails,
  }
}
```

#### 3. MCP 工具 Token 计数
```typescript
export async function countMcpToolTokens(tools, getToolPermissionContext, agentInfo, model, messages) {
  const mcpTools = tools.filter(tool => tool.isMcp)
  
  // 批量计数所有 MCP 工具（一次 API 调用）
  const totalTokensRaw = await countToolDefinitionTokens(mcpTools, ...)
  const totalTokens = Math.max(0, (totalTokensRaw || 0) - TOOL_TOKEN_COUNT_OVERHEAD)

  // 使用本地估算按比例分配 Token 到各个工具
  const estimates = await Promise.all(
    mcpTools.map(async t =>
      roughTokenCountEstimation(
        jsonStringify({
          name: t.name,
          description: await t.prompt(...),
          input_schema: t.inputJSONSchema ?? {},
        }),
      ),
    ),
  )
  
  // 分配 Token 并标记已加载状态
  const mcpToolDetails = mcpTools.map((tool, i) => ({
    name: tool.name,
    serverName: tool.name.split('__')[1] || 'unknown',
    tokens: Math.round((estimates[i] / estimateTotal) * totalTokens),
    isLoaded: loadedMcpToolNames.has(tool.name) || !isDeferredTool(tool),
  }))
}
```

#### 4. 消息 Token 分解
```typescript
async function approximateMessageTokens(messages: Message[]): Promise<MessageBreakdown> {
  // 1. 先进行微压缩，移除不必要的字段
  const microcompactResult = await microcompactMessages(messages)

  // 2. 初始化跟踪器
  const breakdown: MessageBreakdown = {
    totalTokens: 0,
    toolCallTokens: 0,
    toolResultTokens: 0,
    attachmentTokens: 0,
    assistantMessageTokens: 0,
    userMessageTokens: 0,
    toolCallsByType: new Map(),
    toolResultsByType: new Map(),
    attachmentsByType: new Map(),
  }

  // 3. 构建 tool_use_id 到 tool_name 的映射
  const toolUseIdToName = new Map<string, string>()
  for (const msg of microcompactResult.messages) {
    if (msg.type === 'assistant') {
      for (const block of msg.message.content) {
        if (block.type === 'tool_use') {
          toolUseIdToName.set(block.id, block.name)
        }
      }
    }
  }

  // 4. 处理每条消息
  for (const msg of microcompactResult.messages) {
    if (msg.type === 'assistant') {
      processAssistantMessage(msg, breakdown)
    } else if (msg.type === 'user') {
      processUserMessage(msg, breakdown, toolUseIdToName)
    } else if (msg.type === 'attachment') {
      processAttachment(msg, breakdown)
    }
  }

  // 5. 使用 API 获取准确的 Token 计数
  breakdown.totalTokens = await countTokensWithFallback(normalizedMessages, []) ?? 0
  return breakdown
}
```

#### 5. 网格可视化数据生成
```typescript
// 网格配置
const isNarrowScreen = terminalWidth && terminalWidth < 80
const GRID_WIDTH = contextWindow >= 1000000
  ? (isNarrowScreen ? 5 : 20)
  : (isNarrowScreen ? 5 : 10)
const GRID_HEIGHT = contextWindow >= 1000000 ? 10 : (isNarrowScreen ? 5 : 10)
const TOTAL_SQUARES = GRID_WIDTH * GRID_HEIGHT

// 计算每个分类的方格数
const categorySquares = nonDeferredCats.map(cat => ({
  ...cat,
  squares: cat.name === 'Free space'
    ? Math.round((cat.tokens / contextWindow) * TOTAL_SQUARES)
    : Math.max(1, Math.round((cat.tokens / contextWindow) * TOTAL_SQUARES)),
  percentageOfTotal: Math.round((cat.tokens / contextWindow) * 100),
}))

// 创建方格并分配颜色/元数据
function createCategorySquares(category): GridSquare[] {
  const squares: GridSquare[] = []
  const exactSquares = (category.tokens / contextWindow) * TOTAL_SQUARES
  const wholeSquares = Math.floor(exactSquares)
  const fractionalPart = exactSquares - wholeSquares

  for (let i = 0; i < category.squares; i++) {
    const squareFullness = (i === wholeSquares && fractionalPart > 0) 
      ? fractionalPart 
      : 1.0
    
    squares.push({
      color: category.color,
      isFilled: true,
      categoryName: category.name,
      tokens: category.tokens,
      percentage: category.percentageOfTotal,
      squareFullness,
    })
  }
  return squares
}
```

---

## 关键代码路径与文件引用

### 调用方（Consumers）

| 文件 | 用途 |
|------|------|
| `src/commands/context/context.tsx` | `/context` 命令的交互式显示 |
| `src/commands/context/context-noninteractive.ts` | `/context` 命令的非交互式输出 |
| `src/utils/doctorContextWarnings.ts` | 上下文警告诊断 |
| `src/utils/contextSuggestions.ts` | 上下文优化建议 |
| `src/utils/toolSearch.ts` | 工具搜索功能 |
| `src/components/ContextVisualization.tsx` | 上下文可视化 React 组件 |

### 依赖模块

| 模块 | 用途 |
|------|------|
| `src/constants/prompts.ts` | 系统提示词获取 |
| `src/services/compact/microCompact.ts` | 消息微压缩 |
| `src/services/compact/autoCompact.ts` | 自动压缩配置 |
| `src/services/tokenEstimation.ts` | Token 计数 API |
| `src/Tool.ts` | 工具类型定义和工具查找 |
| `src/tools/AgentTool/loadAgentsDir.ts` | 代理定义加载 |
| `src/tools/SkillTool/constants.ts` | 技能工具常量 |
| `src/tools/SkillTool/prompt.ts` | 技能工具提示词 |
| `src/types/message.ts` | 消息类型定义 |
| `src/utils/api.ts` | 工具 API Schema 转换 |
| `src/utils/claudemd.ts` | CLAUDE.md 文件处理 |
| `src/utils/context.ts` | 上下文窗口大小获取 |
| `src/utils/messages.ts` | 消息规范化 |
| `src/utils/model/model.ts` | 运行时模型获取 |
| `src/utils/systemPrompt.ts` | 系统提示词构建 |
| `src/utils/tokens.ts` | Token 使用获取 |

---

## 依赖与外部交互

### Anthropic API 交互

- **Token 计数 API**：`countMessagesTokensWithAPI()` - 官方 Token 计数端点
- **Haiku 回退**：`countTokensViaHaikuFallback()` - 使用 Haiku 模型估算
- **API 使用数据**：从消息历史中提取实际的 API 使用统计

### 工具系统集成

- **工具定义转换**：`toolToAPISchema()` 将内部工具格式转换为 API 格式
- **延迟加载检查**：动态导入 `toolSearch.js` 和 `ToolSearchTool/prompt.js`
- **MCP 工具识别**：通过 `tool.isMcp` 属性区分内置和 MCP 工具

### 压缩系统集成

- **微压缩**：`microcompactMessages()` 移除消息中的不必要字段
- **自动压缩阈值**：`getEffectiveContextWindowSize()` 和 `AUTOCOMPACT_BUFFER_TOKENS`
- **上下文折叠**：检查 `tengu_cobalt_raccoon` 和 `marble_origami` 功能标志

---

## 风险、边界与改进建议

### 已知风险

1. **Token 计数 API 失败**
   - 依赖外部 API 进行 Token 计数
   - 失败时降级到 Haiku 估算，可能不够准确
   - 极端情况下可能返回 `null`，导致显示为 0

2. **性能问题**
   - 大量工具或长消息历史时，Token 计数可能耗时
   - 并行使用 `Promise.all` 缓解，但仍可能阻塞
   - MCP 工具描述需要异步获取，增加延迟

3. **工具开销计算复杂性**
   - `TOOL_TOKEN_COUNT_OVERHEAD = 500` 是经验值，可能随 API 变化
   - 批量计数和单独计数的开销处理容易出错

4. **Ant-only 功能泄露风险**
   - `deferredBuiltinTools`、`systemTools`、`systemPromptSections` 仅在 Ant 构建中返回
   - 使用 `process.env.USER_TYPE === 'ant'` 检查，但代码路径仍存在于外部构建

### 边界情况

1. **空工具列表**
   ```typescript
   if (builtInTools.length < 1) {
     return {
       builtInToolTokens: 0,
       deferredBuiltinDetails: [],
       deferredBuiltinTokens: 0,
       systemToolDetails: [],
     }
   }
   ```

2. **API 使用数据不可用**
   ```typescript
   readonly apiUsage: {
     input_tokens: number
     output_tokens: number
     cache_creation_input_tokens: number
     cache_read_input_tokens: number
   } | null  // 可能为 null
   ```

3. **延迟分类 Token 处理**
   ```typescript
   const actualUsage = cats.reduce(
     (sum, cat) => sum + (cat.isDeferred ? 0 : cat.tokens),
     0,
   )
   ```

4. **窄屏适配**
   ```typescript
   const isNarrowScreen = terminalWidth && terminalWidth < 80
   const GRID_WIDTH = contextWindow >= 1000000
     ? (isNarrowScreen ? 5 : 20)
     : (isNarrowScreen ? 5 : 10)
   ```

5. **Simple 模式**
   ```typescript
   if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
     return { memoryFileDetails: [], claudeMdTokens: 0 }
   }
   ```

### 改进建议

1. **添加缓存机制**
   ```typescript
   // 缓存 Token 计数结果，避免重复计算
   const tokenCountCache = new Map<string, number>()
   
   async function countTokensWithCache(key: string, fn: () => Promise<number>): Promise<number> {
     if (tokenCountCache.has(key)) {
       return tokenCountCache.get(key)!
     }
     const result = await fn()
     tokenCountCache.set(key, result)
     return result
   }
   ```

2. **增量更新支持**
   ```typescript
   export async function analyzeContextUsageIncremental(
     previousAnalysis: ContextData,
     changes: ContextChanges,
   ): Promise<ContextData> {
     // 只重新计算变化的部分，而非全部
   }
   ```

3. **添加警告阈值**
   ```typescript
   interface ContextHealth {
     status: 'healthy' | 'warning' | 'critical'
     utilizationPercent: number
     recommendations: string[]
   }
   
   export function getContextHealth(contextData: ContextData): ContextHealth {
     // 基于使用率和分布提供健康评估
   }
   ```

4. **改进错误处理**
   ```typescript
   // 为每个计数操作添加独立的错误处理
   const [systemResult, toolResult] = await Promise.allSettled([
     countSystemTokens(...),
     countBuiltInToolTokens(...),
   ])
   
   const systemPromptTokens = systemResult.status === 'fulfilled' 
     ? systemResult.value 
     : { systemPromptTokens: 0, systemPromptSections: [] }
   ```

5. **支持更多模型提供商**
   ```typescript
   // 抽象 Token 计数接口，支持不同提供商
   interface TokenCounter {
     count(messages: Message[], tools: Tool[]): Promise<number>
   }
   
   class AnthropicTokenCounter implements TokenCounter { }
   class BedrockTokenCounter implements TokenCounter { }
   class VertexTokenCounter implements TokenCounter { }
   ```

6. **添加历史趋势**
   ```typescript
   interface ContextHistoryEntry {
     timestamp: number
     totalTokens: number
     categoryBreakdown: Record<string, number>
   }
   
   // 追踪 Token 使用趋势，帮助用户理解增长模式
   export function recordContextSnapshot(data: ContextData): void
   ```

7. **优化 MCP 工具计数**
   ```typescript
   // 缓存 MCP 工具描述，避免重复异步获取
   const mcpDescriptionCache = new Map<string, string>()
   
   async function getMcpToolDescription(tool: Tool): Promise<string> {
     if (mcpDescriptionCache.has(tool.name)) {
       return mcpDescriptionCache.get(tool.name)!
     }
     const desc = await tool.prompt(...)
     mcpDescriptionCache.set(tool.name, desc)
     return desc
   }
   ```

8. **添加单元测试覆盖**
   - 测试各种 Token 计数场景
   - 测试降级逻辑
   - 测试网格生成算法
   - 测试边界条件（空列表、超大消息等）
