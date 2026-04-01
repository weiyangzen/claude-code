# prompt.ts 深度研究文档

## 场景与职责

`prompt.ts` 是 Claude Code 中 Agent 工具的**提示生成模块**，包含 287 行代码。它负责生成 Agent 工具的系统提示，包括工具使用说明、代理列表、使用示例等。该模块是 Agent 工具与用户交互的"说明书"，直接影响代理工具的使用效果。

该模块的核心职责：
1. **提示生成**：根据代理定义列表生成完整的 Agent 工具提示
2. **Fork 功能集成**：在启用 Fork 功能时添加 Fork 相关的指导说明
3. **代理列表格式化**：格式化代理列表，支持内联和附件两种模式
4. **使用示例生成**：提供不同场景下的使用示例
5. **环境适配**：根据环境（嵌入式工具、Coordinator 模式、队友模式）调整提示

## 功能点目的

### 1. 代理列表注入模式
支持两种代理列表注入方式：
- **内联模式**：直接将代理列表嵌入工具描述
- **附件模式**：将代理列表作为系统消息附件发送

附件模式是优化策略，用于解决缓存失效问题：
- MCP 异步连接、插件重载、权限模式变化会改变代理列表
- 代理列表变化会导致工具描述变化
- 工具描述变化会导致完整的工具模式缓存失效
- 附件模式保持工具描述静态，避免缓存失效

### 2. Fork 功能提示
当启用 Fork 功能时，添加专门的指导：
- **何时 Fork**：解释 Fork 的适用场景（研究、实现）
- **Fork 规则**：不要查看输出文件、不要竞争、如何写 Fork 提示
- **Fork 示例**：提供 Fork 使用的具体示例

### 3. 使用指导
提供全面的 Agent 工具使用指导：
- **何时不使用**：简单文件读取、搜索等应使用专用工具
- **使用说明**：描述参数、并发执行、后台任务、继续代理等
- **提示写作**：如何编写有效的代理提示

### 4. 环境适配
根据运行环境调整提示：
- **嵌入式工具**：使用 Bash 的 find/grep 替代专用工具
- **Coordinator 模式**：提供精简提示
- **队友模式**：禁用某些参数
- **订阅类型**：控制并发提示的显示

## 具体技术实现

### 关键函数

**1. 代理行格式化**

```typescript
export function formatAgentLine(agent: AgentDefinition): string {
  const toolsDescription = getToolsDescription(agent)
  return `- ${agent.agentType}: ${agent.whenToUse} (Tools: ${toolsDescription})`
}

function getToolsDescription(agent: AgentDefinition): string {
  const { tools, disallowedTools } = agent
  const hasAllowlist = tools && tools.length > 0
  const hasDenylist = disallowedTools && disallowedTools.length > 0

  if (hasAllowlist && hasDenylist) {
    // 两者都定义：过滤允许列表
    const denySet = new Set(disallowedTools)
    const effectiveTools = tools.filter(t => !denySet.has(t))
    return effectiveTools.length === 0 ? 'None' : effectiveTools.join(', ')
  } else if (hasAllowlist) {
    return tools.join(', ')
  } else if (hasDenylist) {
    return `All tools except ${disallowedTools.join(', ')}`
  }
  return 'All tools'
}
```

**2. 附件模式检测**

```typescript
export function shouldInjectAgentListInMessages(): boolean {
  // 环境变量强制启用
  if (isEnvTruthy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES)) return true
  // 环境变量强制禁用
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES))
    return false
  // 使用 GrowthBook 特性值
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false)
}
```

**3. 主提示生成函数**

```typescript
export async function getPrompt(
  agentDefinitions: AgentDefinition[],
  isCoordinator?: boolean,
  allowedAgentTypes?: string[],
): Promise<string> {
  // 1. 按允许的代理类型过滤
  const effectiveAgents = allowedAgentTypes
    ? agentDefinitions.filter(a => allowedAgentTypes.includes(a.agentType))
    : agentDefinitions

  // 2. 检查 Fork 功能
  const forkEnabled = isForkSubagentEnabled()

  // 3. 构建 Fork 指导部分
  const whenToForkSection = forkEnabled ? `...` : ''

  // 4. 构建提示写作指导
  const writingThePromptSection = `...`

  // 5. 选择示例（Fork 或普通）
  const forkExamples = `...`
  const currentExamples = `...`

  // 6. 确定代理列表注入方式
  const listViaAttachment = shouldInjectAgentListInMessages()
  const agentListSection = listViaAttachment
    ? `Available agent types are listed in <system-reminder> messages...`
    : `Available agent types and the tools they have access to:\n${effectiveAgents.map(formatAgentLine).join('\n')}`

  // 7. 构建共享核心提示
  const shared = `Launch a new agent to handle complex, multi-step tasks autonomously...

The ${AGENT_TOOL_NAME} tool launches specialized agents...

${agentListSection}

${forkEnabled 
  ? `When using the ${AGENT_TOOL_NAME} tool, specify a subagent_type to use a specialized agent, or omit it to fork yourself...`
  : `When using the ${AGENT_TOOL_NAME} tool, specify a subagent_type parameter...`}`

  // 8. Coordinator 模式返回精简提示
  if (isCoordinator) {
    return shared
  }

  // 9. 非 Coordinator 模式返回完整提示
  return `${shared}
${whenNotToUseSection}

Usage notes:
- Always include a short description...
${concurrencyNote}
- When the agent is done...
${forkEnabled ? '' : backgroundTasksSection}
- To continue a previously spawned agent...
${isolationSection}
${whenToForkSection}${writingThePromptSection}

${forkEnabled ? forkExamples : currentExamples}`
}
```

### 提示结构

**共享核心部分（所有模式）**：
```
1. 工具用途说明
2. 代理列表（内联或附件引用）
3. 使用方式说明（Fork 或普通）
```

**非 Coordinator 模式附加部分**：
```
4. 何时不使用该工具
5. 使用说明
   - 描述参数
   - 并发执行提示
   - 前后台任务选择
   - 继续代理方法
   - 隔离模式说明
6. Fork 指导（如果启用）
7. 提示写作指导
8. 使用示例
```

### 环境适配

**嵌入式工具适配**：
```typescript
const embedded = hasEmbeddedSearchTools()
const fileSearchHint = embedded
  ? '`find` via the Bash tool'
  : `the ${GLOB_TOOL_NAME} tool`
const contentSearchHint = embedded
  ? '`grep` via the Bash tool'
  : `the ${GLOB_TOOL_NAME} tool`
```

**队友模式适配**：
```typescript
${isInProcessTeammate()
  ? `- The run_in_background, name, team_name, and mode parameters are not available...`
  : isTeammate()
    ? `- The name, team_name, and mode parameters are not available...`
    : ''}
```

## 依赖与外部交互

### 依赖模块

| 模块路径 | 用途 |
|---------|------|
| `../../services/analytics/growthbook.js` | 获取特性值 (`getFeatureValue_CACHED_MAY_BE_STALE`) |
| `../../utils/auth.js` | 获取订阅类型 (`getSubscriptionType`) |
| `../../utils/embeddedTools.js` | 检查嵌入式工具 (`hasEmbeddedSearchTools`) |
| `../../utils/envUtils.js` | 环境变量检查 (`isEnvDefinedFalsy`, `isEnvTruthy`) |
| `../../utils/teammate.js` | 队友检查 (`isTeammate`) |
| `../../utils/teammateContext.js` | 进程内队友检查 (`isInProcessTeammate`) |
| `../FileReadTool/prompt.js` | 文件读取工具名称 |
| `../FileWriteTool/prompt.js` | 文件写入工具名称 |
| `../GlobTool/prompt.js` | Glob 工具名称 |
| `../SendMessageTool/constants.js` | 发送消息工具名称 |
| `./constants.js` | Agent 工具名称常量 |
| `./forkSubagent.js` | Fork 功能检查 (`isForkSubagentEnabled`) |
| `./loadAgentsDir.js` | 代理定义类型 (`AgentDefinition`) |

### 被调用方

通过 Grep 搜索，该模块被以下文件引用：
- `src/tools.ts` - 组装工具池时获取 Agent 工具提示
- `src/utils/attachments.ts` - 生成代理列表附件
- `src/utils/toolSearch.ts` - 工具搜索

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES` | 强制启用/禁用附件模式 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务说明 |
| `USER_TYPE` | 控制 remote 隔离模式的说明 |

### GrowthBook 特性

| 特性名 | 默认值 | 用途 |
|-------|-------|------|
| `tengu_agent_list_attach` | `false` | 控制附件模式 |

## 风险、边界与改进建议

### 已知风险

1. **提示长度**：完整的提示可能非常长，增加 token 消耗
2. **缓存失效**：虽然附件模式缓解了工具描述缓存失效，但提示内容变化仍会导致缓存失效
3. **特性标志依赖**：过多依赖 GrowthBook 特性可能导致提示不一致
4. **国际化**：提示目前只有英文版本

### 边界情况

1. **空代理列表**：如果没有任何代理定义，代理列表部分会显示为空
2. **全部禁用**：如果所有特性都禁用，提示会缺少某些部分
3. **并发提示**：非 Pro 订阅会显示并发执行提示
4. **示例适用性**：示例可能不适用于所有使用场景

### 改进建议

1. **提示优化**：
   - 实现提示压缩，减少 token 消耗
   - 根据用户历史行为个性化提示
   - 添加提示效果 A/B 测试

2. **国际化**：
   - 支持多语言提示
   - 根据用户区域设置自动选择语言

3. **动态示例**：
   - 根据可用代理动态生成示例
   - 提供交互式示例选择

4. **可配置性**：
   - 允许用户自定义提示部分
   - 提供提示模板系统

5. **可观测性**：
   - 跟踪提示长度和 token 消耗
   - 分析提示对代理使用效果的影响
   - 收集用户对提示的反馈

### 代码质量建议

1. 将长字符串提取为模板文件
2. 添加单元测试验证提示格式
3. 使用类型安全的模板系统
4. 实现提示版本控制

### 架构建议

1. **提示即服务**：将提示生成独立为微服务
2. **提示组合**：使用组合模式构建提示，提高灵活性
3. **提示缓存**：实现提示预生成和缓存
4. **提示分析**：添加提示质量和效果分析工具
