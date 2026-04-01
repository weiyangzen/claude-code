# agentDisplay.ts 深度研究文档

## 场景与职责

`agentDisplay.ts` 是 Claude Code 中负责**代理信息显示**的共享工具模块。它为 CLI 的 `claude agents` 命令处理器和交互式的 `/agents` 命令提供统一的代理信息展示逻辑，确保在不同界面中代理列表的显示格式和排序保持一致。

该模块的核心职责：
1. **代理源分组管理**：定义代理来源的优先级和显示顺序
2. **代理覆盖检测**：识别被更高优先级源覆盖的代理
3. **代理信息格式化**：提供统一的代理信息显示格式
4. **去重处理**：处理 git worktree 场景下的重复代理问题

## 功能点目的

### 1. 代理源分组（AGENT_SOURCE_GROUPS）
定义了 7 种代理来源，按优先级排序：
- User agents（用户设置）
- Project agents（项目设置）
- Local agents（本地设置）
- Managed agents（策略设置）
- Plugin agents（插件）
- CLI arg agents（命令行参数）
- Built-in agents（内置代理）

### 2. 代理覆盖解析（resolveAgentOverrides）
检测代理是否被更高优先级的源覆盖：
- 比较所有代理与活跃代理列表
- 识别相同 agentType 但不同 source 的覆盖关系
- 按 `(agentType, source)` 去重，处理 git worktree 重复问题

### 3. 代理模型显示（resolveAgentModelDisplay）
解析并格式化代理的模型显示字符串：
- 返回模型别名或 'inherit'
- 处理默认子代理模型的回退逻辑

### 4. 代理排序（compareAgentsByName）
提供按名称字母顺序排序的比较函数，支持国际化字符。

## 具体技术实现

### 关键数据类型

```typescript
// 代理源类型
 type AgentSource = SettingSource | 'built-in' | 'plugin'

// 代理源分组定义
export type AgentSourceGroup = {
  label: string
  source: AgentSource
}

// 解析后的代理（包含覆盖信息）
export type ResolvedAgent = AgentDefinition & {
  overriddenBy?: AgentSource
}
```

### 核心算法：代理覆盖解析

```typescript
export function resolveAgentOverrides(
  allAgents: AgentDefinition[],
  activeAgents: AgentDefinition[],
): ResolvedAgent[] {
  // 1. 构建活跃代理映射表（agentType -> agent）
  const activeMap = new Map<string, AgentDefinition>()
  for (const agent of activeAgents) {
    activeMap.set(agent.agentType, agent)
  }

  // 2. 使用 Set 去重（处理 git worktree 场景）
  const seen = new Set<string>()
  const resolved: ResolvedAgent[] = []

  // 3. 遍历所有代理，标注覆盖信息
  for (const agent of allAgents) {
    const key = `${agent.agentType}:${agent.source}`
    if (seen.has(key)) continue
    seen.add(key)

    const active = activeMap.get(agent.agentType)
    const overriddenBy =
      active && active.source !== agent.source ? active.source : undefined
    resolved.push({ ...agent, overriddenBy })
  }

  return resolved
}
```

### 代理源分组定义

```typescript
export const AGENT_SOURCE_GROUPS: AgentSourceGroup[] = [
  { label: 'User agents', source: 'userSettings' },
  { label: 'Project agents', source: 'projectSettings' },
  { label: 'Local agents', source: 'localSettings' },
  { label: 'Managed agents', source: 'policySettings' },
  { label: 'Plugin agents', source: 'plugin' },
  { label: 'CLI arg agents', source: 'flagSettings' },
  { label: 'Built-in agents', source: 'built-in' },
]
```

## 依赖与外部交互

### 依赖模块

| 模块路径 | 用途 |
|---------|------|
| `../../utils/model/agent.js` | 获取默认子代理模型 (`getDefaultSubagentModel`) |
| `../../utils/settings/constants.js` | 获取源显示名称 (`getSourceDisplayName`, `SettingSource`) |
| `./loadAgentsDir.js` | 代理定义类型 (`AgentDefinition`) |

### 被调用方

通过 Grep 搜索，该模块被以下文件引用：
- `src/cli/handlers/agents.ts` - CLI agents 命令处理
- `src/components/agents/AgentsList.tsx` - 代理列表 UI 组件
- `src/components/agents/AgentsMenu.tsx` - 代理菜单组件

### 使用示例

```typescript
// CLI 处理器中使用
const resolved = resolveAgentOverrides(allAgents, activeAgents)
for (const group of AGENT_SOURCE_GROUPS) {
  const groupAgents = resolved.filter(a => a.source === group.source)
  if (groupAgents.length > 0) {
    console.log(`\n${group.label}:`)
    for (const agent of groupAgents.sort(compareAgentsByName)) {
      const model = resolveAgentModelDisplay(agent)
      console.log(`  ${agent.agentType}${model ? ` (${model})` : ''}`)
    }
  }
}
```

## 风险、边界与改进建议

### 已知风险

1. **源优先级硬编码**：`AGENT_SOURCE_GROUPS` 的顺序是固定的，修改可能影响覆盖检测逻辑
2. **worktree 去重假设**：假设重复代理的 `(agentType, source)` 组合相同，这在某些边缘场景可能不成立
3. **无缓存**：每次显示代理列表都会重新计算覆盖关系

### 边界情况

1. **同名不同源代理**：相同 agentType 但不同 source 的代理会被正确识别为覆盖关系
2. **空列表处理**：函数对空数组有良好处理，不会抛出异常
3. **大小写敏感**：代理名称比较使用 `localeCompare` 的 `'base'` 敏感度，忽略大小写差异

### 改进建议

1. **缓存机制**：对 `resolveAgentOverrides` 的结果添加缓存，避免重复计算
2. **配置化优先级**：将源优先级改为可配置，而非硬编码
3. **覆盖原因说明**：不仅显示被哪个源覆盖，还显示为什么（如版本号、时间戳等）
4. **过滤功能**：添加按源过滤代理的功能
5. **搜索功能**：支持按名称或描述搜索代理

### 代码质量建议

1. 添加单元测试覆盖各种覆盖场景
2. 考虑使用 immer 等不可变库简化代理对象扩展
3. 添加 JSDoc 注释说明函数的副作用和性能特征
