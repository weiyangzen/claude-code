# unifiedSuggestions.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`unifiedSuggestions.ts` 是 Claude Code 的统一建议聚合器，负责将来自不同来源的建议（文件、MCP 资源、Agent）合并为一个统一的排序列表。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 用户输入 `@` | 显示文件 + MCP 资源 + Agent 的混合建议 |
| 模糊搜索 | 使用 Fuse.js 对非文件来源进行评分排序 |
| 空查询展示 | 无输入时显示所有可用建议的截断列表 |

### 1.3 调用方
- `src/hooks/useTypeahead.tsx` - 主要的建议系统集成点

---

## 2. 功能点目的

### 2.1 多源建议聚合
- **目的**：将异构建议源统一为一致的接口
- **来源**：
  - 文件建议（来自 `fileSuggestions.ts`，使用 Rust nucleo 评分）
  - MCP 资源（来自 MCP 服务器）
  - Agent 定义（来自 `loadAgentsDir.ts`）

### 2.2 统一评分排序
- **目的**：确保不同来源的建议可以公平比较
- **策略**：
  - 文件建议：使用 nucleo 评分（0-1，越低越好）
  - 其他来源：使用 Fuse.js 模糊搜索评分

### 2.3 结果限制
- **目的**：控制 UI 渲染性能和用户体验
- **实现**：`MAX_UNIFIED_SUGGESTIONS = 15`

---

## 3. 具体技术实现

### 3.1 数据结构

```typescript
// 文件建议源
type FileSuggestionSource = {
  type: 'file'
  displayText: string
  description?: string
  path: string
  filename: string
  score?: number  // nucleo 评分
}

// MCP 资源建议源
type McpResourceSuggestionSource = {
  type: 'mcp_resource'
  displayText: string
  description: string
  server: string
  uri: string
  name: string
}

// Agent 建议源
type AgentSuggestionSource = {
  type: 'agent'
  displayText: string
  description: string
  agentType: string
  color?: keyof Theme
}

type SuggestionSource = 
  | FileSuggestionSource 
  | McpResourceSuggestionSource 
  | AgentSuggestionSource
```

### 3.2 统一建议生成流程

```typescript
export async function generateUnifiedSuggestions(
  query: string,
  mcpResources: Record<string, ServerResource[]>,
  agents: AgentDefinition[],
  showOnEmpty = false,
): Promise<SuggestionItem[]> {
  // 1. 并行获取文件和 Agent 建议
  const [fileSuggestions, agentSources] = await Promise.all([
    generateFileSuggestions(query, showOnEmpty),
    Promise.resolve(generateAgentSuggestions(agents, query, showOnEmpty)),
  ])

  // 2. 转换文件建议
  const fileSources: FileSuggestionSource[] = fileSuggestions.map(suggestion => ({
    type: 'file' as const,
    displayText: suggestion.displayText,
    description: suggestion.description,
    path: suggestion.displayText,
    filename: basename(suggestion.displayText),
    score: (suggestion.metadata as { score?: number })?.score,
  }))

  // 3. 转换 MCP 资源
  const mcpSources: McpResourceSuggestionSource[] = Object.values(mcpResources)
    .flat()
    .map(resource => ({
      type: 'mcp_resource' as const,
      displayText: `${resource.server}:${resource.uri}`,
      description: truncateDescription(
        resource.description || resource.name || resource.uri,
      ),
      server: resource.server,
      uri: resource.uri,
      name: resource.name || resource.uri,
    }))

  // 4. 空查询：简单截断返回
  if (!query) {
    const allSources = [...fileSources, ...mcpSources, ...agentSources]
    return allSources
      .slice(0, MAX_UNIFIED_SUGGESTIONS)
      .map(createSuggestionFromSource)
  }

  // 5. 有查询：评分排序
  const nonFileSources: SuggestionSource[] = [...mcpSources, ...agentSources]
  const scoredResults: ScoredSource[] = []

  // 5.1 添加文件建议（已有 nucleo 评分）
  for (const fileSource of fileSources) {
    scoredResults.push({
      source: fileSource,
      score: fileSource.score ?? 0.5,  // 默认中等评分
    })
  }

  // 5.2 使用 Fuse.js 评分非文件来源
  if (nonFileSources.length > 0) {
    const fuse = new Fuse(nonFileSources, {
      includeScore: true,
      threshold: 0.6,
      keys: [
        { name: 'displayText', weight: 2 },
        { name: 'name', weight: 3 },
        { name: 'server', weight: 1 },
        { name: 'description', weight: 1 },
        { name: 'agentType', weight: 3 },
      ],
    })

    const fuseResults = fuse.search(query, { limit: MAX_UNIFIED_SUGGESTIONS })
    for (const result of fuseResults) {
      scoredResults.push({
        source: result.item,
        score: result.score ?? 0.5,
      })
    }
  }

  // 6. 排序并截断
  scoredResults.sort((a, b) => a.score - b.score)
  return scoredResults
    .slice(0, MAX_UNIFIED_SUGGESTIONS)
    .map(r => r.source)
    .map(createSuggestionFromSource)
}
```

### 3.3 Agent 建议生成

```typescript
function generateAgentSuggestions(
  agents: AgentDefinition[],
  query: string,
  showOnEmpty = false,
): AgentSuggestionSource[] {
  if (!query && !showOnEmpty) {
    return []
  }

  const agentSources: AgentSuggestionSource[] = agents.map(agent => ({
    type: 'agent' as const,
    displayText: `${agent.agentType} (agent)`,
    description: truncateDescription(agent.whenToUse),
    agentType: agent.agentType,
    color: getAgentColor(agent.agentType),
  }))

  if (!query) {
    return agentSources
  }

  // 简单字符串匹配（Fuse.js 会在上层进行完整评分）
  const queryLower = query.toLowerCase()
  return agentSources.filter(
    agent =>
      agent.agentType.toLowerCase().includes(queryLower) ||
      agent.displayText.toLowerCase().includes(queryLower),
  )
}
```

### 3.4 建议项创建

```typescript
function createSuggestionFromSource(source: SuggestionSource): SuggestionItem {
  switch (source.type) {
    case 'file':
      return {
        id: `file-${source.path}`,
        displayText: source.displayText,
        description: source.description,
      }
    case 'mcp_resource':
      return {
        id: `mcp-resource-${source.server}__${source.uri}`,
        displayText: source.displayText,
        description: source.description,
      }
    case 'agent':
      return {
        id: `agent-${source.agentType}`,
        displayText: source.displayText,
        description: source.description,
        color: source.color,
      }
  }
}
```

### 3.5 描述截断

```typescript
const DESCRIPTION_MAX_LENGTH = 60

function truncateDescription(description: string): string {
  return truncateToWidth(description, DESCRIPTION_MAX_LENGTH)
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
unifiedSuggestions.ts
├── fileSuggestions.ts          (generateFileSuggestions)
├── components/PromptInput/PromptInputFooterSuggestions.js  (SuggestionItem type)
├── services/mcp/types.js       (ServerResource type)
├── tools/AgentTool/agentColorManager.js  (getAgentColor)
├── tools/AgentTool/loadAgentsDir.js      (AgentDefinition type)
├── utils/format.js             (truncateToWidth)
├── utils/theme.js              (Theme type)
└── fuse.js (npm package)       (模糊搜索)
```

### 4.2 调用链

```
useTypeahead.tsx
  └── generateUnifiedSuggestions()
      ├── generateFileSuggestions()     // 异步
      ├── generateAgentSuggestions()    // 同步
      └── Fuse.js search()              // 评分
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `fuse.js` | 模糊搜索评分 | npm 包 |
| `path` (basename) | 提取文件名 | Node.js 内置 |

### 5.2 输入数据源

| 来源 | 提供者 | 更新频率 |
|------|--------|----------|
| MCP 资源 | `useAppState(s => s.mcp.resources)` | 实时 |
| Agent 定义 | `loadAgentsDir()` | 会话启动 |
| 文件建议 | `generateFileSuggestions()` | 用户输入 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 评分不一致 | nucleo 和 Fuse.js 评分尺度不同 | 文件建议优先，其他兜底 0.5 |
| MCP 资源膨胀 | 大量资源导致性能下降 | 限制 15 条结果 |
| 空查询性能 | 无查询时仍处理所有来源 | 简单截断，跳过 Fuse.js |

### 6.2 边界条件

1. **空 agents 数组**：返回空数组
2. **空 MCP 资源**：不影响文件建议
3. **超长描述**：自动截断到 60 字符宽度
4. **特殊字符**：ID 中使用 `__` 分隔符避免冲突

### 6.3 改进建议

1. **评分归一化**：统一 nucleo 和 Fuse.js 的评分尺度
2. **来源优先级**：允许用户配置不同来源的优先级权重
3. **缓存**：缓存 Agent 建议（不随每次输入变化）
4. **增量更新**：MCP 资源变更时增量更新而非全量重建
5. **分类显示**：按来源分组显示建议，而非完全混合

### 6.4 代码质量

- **优点**：
  - 清晰的类型定义和区分
  - 并行获取优化性能
  - 统一的输出接口
  
- **潜在改进**：
  - 评分逻辑可提取为策略模式
  - 硬编码的权重可配置化
  - 添加更多单元测试覆盖边界情况
