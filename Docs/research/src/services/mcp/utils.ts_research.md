# utils.ts 深度研究文档

## 场景与职责

`utils.ts` 是 MCP (Model Context Protocol) 服务层的通用工具函数集合，提供：

1. **工具/命令/资源过滤**：按 MCP 服务器名称筛选或排除相关数据
2. **配置哈希与变更检测**：检测 MCP 服务器配置变更，支持 `/reload-plugins` 后的重新连接决策
3. **配置作用域管理**：描述和验证配置来源（user/project/local/dynamic/enterprise/claudeai）
4. **项目级 MCP 服务器审批状态**：管理 `.mcp.json` 中定义的服务器的用户审批状态
5. **Agent MCP 服务器提取**：从 Agent 定义中提取内联 MCP 服务器配置
6. **安全日志 URL 处理**：提取安全的 MCP 服务器基础 URL（去除查询参数）

该文件是 MCP 系统的基础工具层，被 `config.ts`、`client.ts`、`useManageMCPConnections.ts` 等多个模块依赖。

## 功能点目的

### 1. 工具/命令/资源过滤系统

**目的**：支持按 MCP 服务器名称精确筛选或排除相关数据

**命名约定**：
- MCP 工具名：`mcp__<normalizedServerName>__<toolName>`
- MCP 提示词名：`mcp__<normalizedServerName>__<promptName>`
- MCP 技能名：`<serverName>:<skillName>`（与插件技能命名一致）

**函数列表**：
- `filterToolsByServer`：筛选属于指定服务器的工具
- `filterCommandsByServer`：筛选属于指定服务器的命令
- `filterMcpPromptsByServer`：仅筛选 MCP 提示词（排除技能）
- `filterResourcesByServer`：筛选属于指定服务器的资源
- `excludeToolsByServer`：排除指定服务器的工具
- `excludeCommandsByServer`：排除指定服务器的命令
- `excludeResourcesByServer`：排除指定服务器的资源

### 2. 配置变更检测

**目的**：检测 MCP 服务器配置变更，决定是否需要重新连接

**关键函数**：`hashMcpConfig`
- 排除 `scope` 字段（来源信息，不是内容）
- 按键名排序确保哈希稳定
- 使用 SHA-256 取前 16 位

**关键函数**：`excludeStalePluginClients`
- 识别过期客户端：scope 为 'dynamic' 且名称不在新配置中，或配置哈希变化
- 清理关联的工具/命令/资源
- 返回过期客户端列表供调用方断开连接

### 3. 项目级 MCP 服务器审批

**目的**：管理 `.mcp.json` 中定义的 MCP 服务器的用户审批流程

**状态**：`'approved' | 'rejected' | 'pending'`

**审批逻辑**：
1. 检查 `disabledMcpjsonServers`：如果被禁用，返回 'rejected'
2. 检查 `enabledMcpjsonServers` 或 `enableAllProjectMcpServers`：如果启用，返回 'approved'
3. 检查 `--dangerously-skip-permissions` 模式：如果启用且 `projectSettings` 启用，返回 'approved'
4. 检查非交互式会话（SDK/`-p`/管道输入）：如果是且 `projectSettings` 启用，返回 'approved'
5. 默认返回 'pending'

**安全考虑**：
- 仅检查 `userSettings/localSettings/flagSettings/policySettings` 的 `skipDangerousModePermissionPrompt`，不检查 `projectSettings`
- 防止仓库通过 `.claude/settings.json` 自动绕过权限检查

### 4. Agent MCP 服务器提取

**目的**：从 Agent 定义的 frontmatter 中提取内联 MCP 服务器配置

**处理逻辑**：
1. 遍历所有 Agent 定义
2. 提取 `mcpServers` 字段中的内联定义（排除字符串引用）
3. 按服务器名称分组，记录来源 Agent 列表
4. 根据配置类型创建 `AgentMcpServerInfo`

**支持类型**：stdio、sse、http、ws（排除 sdk、claudeai-proxy、sse-ide、ws-ide）

## 具体技术实现

### 关键数据结构

```typescript
// 配置作用域
export type ConfigScope = 'local' | 'user' | 'project' | 'dynamic' | 'enterprise' | 'claudeai' | 'managed'

// MCP 服务器配置联合类型
export type McpServerConfig = 
  | McpStdioServerConfig 
  | McpSSEServerConfig 
  | McpSSEIDEServerConfig 
  | McpWebSocketIDEServerConfig 
  | McpHTTPServerConfig 
  | McpWebSocketServerConfig 
  | McpSdkServerConfig 
  | McpClaudeAIProxyServerConfig

// 带作用域的配置
type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  pluginSource?: string  // 插件来源标识
}

// Agent MCP 服务器信息
export type AgentMcpServerInfo = {
  name: string
  sourceAgents: string[]
  transport: 'stdio' | 'sse' | 'http' | 'ws'
  needsAuth: boolean
  // ... 传输类型特定字段
}
```

### 关键流程

#### hashMcpConfig 实现

```typescript
export function hashMcpConfig(config: ScopedMcpServerConfig): string {
  // 排除 scope 字段
  const { scope: _scope, ...rest } = config
  
  // 按键名排序确保稳定哈希
  const stable = jsonStringify(rest, (_k, v: unknown) => {
    if (v && typeof v === 'object' && !Array.isArray(v)) {
      const obj = v as Record<string, unknown>
      const sorted: Record<string, unknown> = {}
      for (const k of Object.keys(obj).sort()) sorted[k] = obj[k]
      return sorted
    }
    return v
  })
  
  return createHash('sha256').update(stable).digest('hex').slice(0, 16)
}
```

#### excludeStalePluginClients 实现

```typescript
export function excludeStalePluginClients(
  mcp: { clients: MCPServerConnection[]; tools: Tool[]; commands: Command[]; resources: Record<string, ServerResource[]> },
  configs: Record<string, ScopedMcpServerConfig>
): { clients: MCPServerConnection[]; tools: Tool[]; commands: Command[]; resources: Record<string, ServerResource[]>; stale: MCPServerConnection[] } {
  // 识别过期客户端
  const stale = mcp.clients.filter(c => {
    const fresh = configs[c.name]
    if (!fresh) return c.config.scope === 'dynamic'  // 仅 dynamic 类型会被移除
    return hashMcpConfig(c.config) !== hashMcpConfig(fresh)  // 配置变更
  })
  
  if (stale.length === 0) return { ...mcp, stale: [] }
  
  // 清理过期客户端关联的数据
  let { tools, commands, resources } = mcp
  for (const s of stale) {
    tools = excludeToolsByServer(tools, s.name)
    commands = excludeCommandsByServer(commands, s.name)
    resources = excludeResourcesByServer(resources, s.name)
  }
  
  const staleNames = new Set(stale.map(c => c.name))
  return {
    clients: mcp.clients.filter(c => !staleNames.has(c.name)),
    tools,
    commands,
    resources,
    stale,
  }
}
```

#### getProjectMcpServerStatus 实现

```typescript
export function getProjectMcpServerStatus(serverName: string): 'approved' | 'rejected' | 'pending' {
  const settings = getSettings_DEPRECATED()
  const normalizedName = normalizeNameForMCP(serverName)
  
  // 1. 检查禁用列表
  if (settings?.disabledMcpjsonServers?.some(name => normalizeNameForMCP(name) === normalizedName)) {
    return 'rejected'
  }
  
  // 2. 检查启用列表
  if (settings?.enabledMcpjsonServers?.some(name => normalizeNameForMCP(name) === normalizedName) ||
      settings?.enableAllProjectMcpServers) {
    return 'approved'
  }
  
  // 3. 检查 --dangerously-skip-permissions 模式
  // 安全：仅检查 userSettings/localSettings/flagSettings/policySettings，不检查 projectSettings
  if (hasSkipDangerousModePermissionPrompt() && isSettingSourceEnabled('projectSettings')) {
    return 'approved'
  }
  
  // 4. 检查非交互式会话
  if (getIsNonInteractiveSession() && isSettingSourceEnabled('projectSettings')) {
    return 'approved'
  }
  
  return 'pending'
}
```

#### extractAgentMcpServers 实现

```typescript
export function extractAgentMcpServers(agents: AgentDefinition[]): AgentMcpServerInfo[] {
  const serverMap = new Map<string, { config: McpServerConfig & { name: string }; sourceAgents: string[] }>()
  
  for (const agent of agents) {
    if (!agent.mcpServers?.length) continue
    
    for (const spec of agent.mcpServers) {
      // 跳过字符串引用
      if (typeof spec === 'string') continue
      
      // 提取内联定义
      const entries = Object.entries(spec)
      if (entries.length !== 1) continue
      
      const [serverName, serverConfig] = entries[0]!
      const existing = serverMap.get(serverName)
      
      if (existing) {
        // 添加来源 Agent
        if (!existing.sourceAgents.includes(agent.agentType)) {
          existing.sourceAgents.push(agent.agentType)
        }
      } else {
        // 新建服务器条目
        serverMap.set(serverName, { config: { ...serverConfig, name: serverName }, sourceAgents: [agent.agentType] })
      }
    }
  }
  
  // 转换为 AgentMcpServerInfo 数组
  const result: AgentMcpServerInfo[] = []
  for (const [name, { config, sourceAgents }] of serverMap) {
    // 使用类型守卫确定传输类型
    if (isStdioConfig(config)) {
      result.push({ name, sourceAgents, transport: 'stdio', command: config.command, needsAuth: false })
    } else if (isSSEConfig(config)) {
      result.push({ name, sourceAgents, transport: 'sse', url: config.url, needsAuth: true })
    }
    // ... 其他类型
  }
  
  return result.sort((a, b) => a.name.localeCompare(b.name))
}
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/services/mcp/types.ts` | `ConfigScope`, `MCPServerConnection`, `McpServerConfig` 等类型 |
| `src/services/mcp/mcpStringUtils.ts` | `mcpInfoFromString`, `normalizeNameForMCP` |
| `src/services/mcp/config.ts` | `getMcpConfigByName`, `getEnterpriseMcpFilePath` |
| `src/utils/settings/settings.ts` | `getSettings_DEPRECATED`, `hasSkipDangerousModePermissionPrompt` |
| `src/utils/settings/constants.ts` | `isSettingSourceEnabled` |
| `src/bootstrap/state.ts` | `getIsNonInteractiveSession` |
| `src/utils/env.ts` | `getGlobalClaudeFile` |
| `src/utils/cwd.ts` | `getCwd` |
| `src/utils/slowOperations.ts` | `jsonStringify` |
| `src/tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition` 类型 |

### 调用方

| 文件 | 调用函数 |
|------|----------|
| `src/services/mcp/useManageMCPConnections.ts` | `commandBelongsToServer`, `excludeStalePluginClients` |
| `src/services/mcp/client.ts` | `getLoggingSafeMcpBaseUrl`, `filterToolsByServer` 等 |
| `src/services/mcp/config.ts` | `hashMcpConfig`, `describeMcpConfigFilePath`, `getScopeLabel` |
| `src/services/mcp/auth.ts` | `getServerKey`, `getLoggingSafeMcpBaseUrl` |
| `src/services/mcp/McpServerMenu.tsx` | `getProjectMcpServerStatus` |
| `src/components/mcp/MCPServerDetails.tsx` | `extractAgentMcpServers` |

### 被调用方

| 函数 | 被调用文件 |
|------|------------|
| `hashMcpConfig` | `useManageMCPConnections.ts`, `config.ts` |
| `excludeStalePluginClients` | `useManageMCPConnections.ts` |
| `getProjectMcpServerStatus` | `McpServerMenu.tsx` |
| `extractAgentMcpServers` | `MCPServerDetails.tsx` |

## 依赖与外部交互

### 外部系统交互

1. **文件系统**：通过 `getGlobalClaudeFile`、`getCwd`、`getEnterpriseMcpFilePath` 获取配置路径
2. **设置系统**：通过 `getSettings_DEPRECATED` 读取用户设置
3. **加密哈希**：使用 Node.js `crypto` 模块的 `createHash`

### 配置依赖

- 无直接环境变量依赖
- 依赖设置系统的 `disabledMcpjsonServers`、`enabledMcpjsonServers`、`enableAllProjectMcpServers`

## 风险、边界与改进建议

### 风险点

1. **哈希碰撞风险**
   - `hashMcpConfig` 使用 16 位十六进制哈希（64 位）
   - 对于大量 MCP 服务器，碰撞概率虽低但存在
   - 建议：增加哈希长度或使用完整哈希比较

2. **配置排序稳定性**
   - `jsonStringify` 的排序逻辑依赖 `Object.keys().sort()`
   - 不同 JavaScript 引擎的排序实现可能略有差异
   - 建议：使用稳定的排序算法

3. **设置系统依赖**
   - `getSettings_DEPRECATED` 标记为废弃，但仍在使用
   - 需要迁移到新的设置 API

4. **类型守卫完整性**
   - `isStdioConfig`、`isSSEConfig` 等类型守卫依赖 `type` 字段
   - 如果配置对象结构变化，类型守卫可能失效

### 边界情况

1. **空配置处理**
   - `hashMcpConfig` 对空对象仍能生成哈希
   - `excludeStalePluginClients` 对空数组返回空结果

2. **配置字段缺失**
   - `getProjectMcpServerStatus` 使用可选链操作符处理可能缺失的设置
   - 注释中提到 `?.` 是修复 e2e 测试的必要条件

3. **Agent MCP 服务器重复**
   - `extractAgentMcpServers` 使用 Map 去重同名服务器
   - 多个 Agent 引用同一服务器时，合并来源列表

### 改进建议

1. **哈希算法优化**
   ```typescript
   // 建议使用更长的哈希或完整比较
   return createHash('sha256').update(stable).digest('hex')  // 完整 64 位
   ```

2. **设置系统迁移**
   - 从 `getSettings_DEPRECATED` 迁移到新的设置 API
   - 统一设置读取接口

3. **类型守卫增强**
   ```typescript
   // 建议添加运行时验证
   function isStdioConfig(config: unknown): config is McpStdioServerConfig {
     return typeof config === 'object' && 
            config !== null && 
            (config as McpStdioServerConfig).type === 'stdio' &&
            typeof (config as McpStdioServerConfig).command === 'string'
   }
   ```

4. **添加单元测试**
   - `hashMcpConfig` 的排序稳定性
   - `excludeStalePluginClients` 的各种边界情况
   - `getProjectMcpServerStatus` 的审批逻辑

5. **性能优化**
   - `extractAgentMcpServers` 可以缓存结果，避免重复计算
   - `getMcpServerScopeFromToolName` 可以添加缓存
