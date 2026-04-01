# MCP 字符串工具 (mcpStringUtils.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`mcpStringUtils.ts` 是 Claude Code MCP 系统的**字符串处理工具库**，专注于 MCP 工具/服务器名称的解析和生成：
- **工具名解析**：从 `mcp__serverName__toolName` 格式提取服务器和工具名
- **工具名生成**：构建规范化的 MCP 工具名
- **权限检查**：生成用于权限规则匹配的工具名
- **显示名提取**：从完整工具名提取用户友好的显示名

### 1.2 业务场景
| 场景 | 说明 |
|------|------|
| 工具调用 | 解析工具名确定调用哪个 MCP 服务器的哪个工具 |
| 权限控制 | 基于 `mcp__server__tool` 格式进行权限规则匹配 |
| UI 展示 | 将技术工具名转换为用户友好的显示名 |
| 工具注册 | 生成规范的 MCP 工具名前缀 |

### 1.3 设计原则
- **纯函数**：无状态、无副作用，易于测试
- **轻量依赖**：仅依赖 `normalization.ts`，避免循环引用
- **容错设计**：解析失败返回 null，不抛出异常
- **命名空间隔离**：使用 `mcp__` 前缀避免与其他工具冲突

---

## 2. 功能点目的

### 2.1 工具名解析
**目的**：将 MCP 工具名分解为服务器名和工具名。

**格式**：`mcp__{serverName}__{toolName}`

**示例**：
- `mcp__github__create_issue` → `{ serverName: 'github', toolName: 'create_issue' }`
- `mcp__filesystem__read_file` → `{ serverName: 'filesystem', toolName: 'read_file' }`

### 2.2 工具名生成
**目的**：构建规范化的 MCP 工具名，确保一致性。

**规范化**：
- 服务器名和工具名都经过 `normalizeNameForMCP` 处理
- 替换非法字符为下划线
- 确保符合 API 模式 `^[a-zA-Z0-9_-]{1,64}$`

### 2.3 权限检查支持
**目的**：支持基于 MCP 工具名的权限规则。

**重要性**：
- 防止内置工具规则（如 `Write`）误匹配 MCP 工具
- 支持服务器级别和工具级别的权限控制

### 2.4 显示名提取
**目的**：将技术工具名转换为 UI 展示用的友好名称。

**处理**：
- 移除 `(MCP)` 后缀
- 提取服务器前缀后的描述部分

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 解析结果
{
  serverName: string      // MCP 服务器名称
  toolName: string | undefined  // 工具名称（可能为 undefined 表示服务器级别）
}
```

### 3.2 工具名解析

```typescript
/**
 * Extracts MCP server information from a tool name string
 * @param toolString The string to parse. Expected format: "mcp__serverName__toolName"
 * @returns An object containing server name and optional tool name, or null if not a valid MCP rule
 * 
 * Known limitation: If a server name contains "__", parsing will be incorrect.
 * For example, "mcp__my__server__tool" would parse as server="my" and tool="server__tool"
 * instead of server="my__server" and tool="tool". This is rare in practice since server
 * names typically don't contain double underscores.
 */
export function mcpInfoFromString(toolString: string): {
  serverName: string
  toolName: string | undefined
} | null {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  
  // 验证前缀和服务器名
  if (mcpPart !== 'mcp' || !serverName) {
    return null
  }
  
  // 合并工具名部分（保留工具名中的双下划线）
  const toolName =
    toolNameParts.length > 0 ? toolNameParts.join('__') : undefined
  
  return { serverName, toolName }
}
```

**解析逻辑**：
1. 按 `__` 分割字符串
2. 验证第一部分是 `mcp`
3. 第二部分是服务器名
4. 剩余部分合并为工具名

**边界情况**：
```typescript
// 有效解析
mcpInfoFromString('mcp__github__create_issue')
// → { serverName: 'github', toolName: 'create_issue' }

mcpInfoFromString('mcp__github')  // 服务器级别
// → { serverName: 'github', toolName: undefined }

// 无效解析
mcpInfoFromString('github__create_issue')  // 缺少 mcp 前缀
// → null

mcpInfoFromString('other__github__tool')  // 前缀错误
// → null
```

### 3.3 MCP 前缀生成

```typescript
/**
 * Generates the MCP tool/command name prefix for a given server
 * @param serverName Name of the MCP server
 * @returns The prefix string
 */
export function getMcpPrefix(serverName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__`
}
```

**示例**：
```typescript
getMcpPrefix('github')        // → 'mcp__github__'
getMcpPrefix('my-server')     // → 'mcp__my-server__'
getMcpPrefix('server.name')   // → 'mcp__server_name__'（规范化后）
```

### 3.4 完整工具名构建

```typescript
/**
 * Builds a fully qualified MCP tool name from server and tool names.
 * Inverse of mcpInfoFromString().
 * @param serverName Name of the MCP server (unnormalized)
 * @param toolName Name of the tool (unnormalized)
 * @returns The fully qualified name, e.g., "mcp__server__tool"
 */
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}
```

**示例**：
```typescript
buildMcpToolName('github', 'create_issue')      // → 'mcp__github__create_issue'
buildMcpToolName('my server', 'my tool')        // → 'mcp__my_server__my_tool'
```

### 3.5 权限检查工具名

```typescript
/**
 * Returns the name to use for permission rule matching.
 * For MCP tools, uses the fully qualified mcp__server__tool name so that
 * deny rules targeting builtins (e.g., "Write") don't match unprefixed MCP
 * replacements that share the same display name. Falls back to `tool.name`.
 */
export function getToolNameForPermissionCheck(tool: {
  name: string
  mcpInfo?: { serverName: string; toolName: string }
}): string {
  return tool.mcpInfo
    ? buildMcpToolName(tool.mcpInfo.serverName, tool.mcpInfo.toolName)
    : tool.name
}
```

**使用场景**：
```typescript
// 内置工具
const builtinTool = { name: 'Write' }
getToolNameForPermissionCheck(builtinTool)  // → 'Write'

// MCP 工具
const mcpTool = { 
  name: 'mcp__github__create_issue',
  mcpInfo: { serverName: 'github', toolName: 'create_issue' }
}
getToolNameForPermissionCheck(mcpTool)  // → 'mcp__github__create_issue'
```

### 3.6 显示名提取

```typescript
/**
 * Extracts the display name from an MCP tool/command name
 * @param fullName The full MCP tool/command name (e.g., "mcp__server_name__tool_name")
 * @param serverName The server name to remove from the prefix
 * @returns The display name without the MCP prefix
 */
export function getMcpDisplayName(
  fullName: string,
  serverName: string
): string {
  const prefix = `mcp__${normalizeNameForMCP(serverName)}__`
  return fullName.replace(prefix, '')
}
```

**示例**：
```typescript
getMcpDisplayName('mcp__github__create_issue', 'github')
// → 'create_issue'
```

### 3.7 用户友好显示名提取

```typescript
/**
 * Extracts just the tool/command display name from a userFacingName
 * @param userFacingName The full user-facing name (e.g., "github - Add comment to issue (MCP)")
 * @returns The display name without server prefix and (MCP) suffix
 */
export function extractMcpToolDisplayName(userFacingName: string): string {
  // This is really ugly but our current Tool type doesn't make it easy to have different display names for different purposes.

  // First, remove the (MCP) suffix if present
  let withoutSuffix = userFacingName.replace(/\s*\(MCP\)\s*$/, '')

  // Trim the result
  withoutSuffix = withoutSuffix.trim()

  // Then, remove the server prefix (everything before " - ")
  const dashIndex = withoutSuffix.indexOf(' - ')
  if (dashIndex !== -1) {
    const displayName = withoutSuffix.substring(dashIndex + 3).trim()
    return displayName
  }

  // If no dash found, return the string without (MCP)
  return withoutSuffix
}
```

**示例**：
```typescript
extractMcpToolDisplayName('github - Add comment to issue (MCP)')
// → 'Add comment to issue'

extractMcpToolDisplayName('filesystem - Read file (MCP)')
// → 'Read file'
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `mcpInfoFromString` | 19-32 | 从工具名解析服务器和工具信息 |
| `getMcpPrefix` | 39-41 | 生成 MCP 工具名前缀 |
| `buildMcpToolName` | 50-52 | 构建完整 MCP 工具名 |
| `getToolNameForPermissionCheck` | 60-67 | 获取权限检查用的工具名 |
| `getMcpDisplayName` | 75-81 | 提取显示名（从完整工具名） |
| `extractMcpToolDisplayName` | 88-106 | 提取用户友好显示名 |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `normalization.ts` | `normalizeNameForMCP` 函数 |

### 4.3 调用方

| 文件 | 用途 |
|------|------|
| `utils.ts` | `filterToolsByServer`、`isToolFromMcpServer` 等 |
| `../../utils/settings/permissionValidation.ts` | 权限规则验证 |
| `../../utils/permissions/permissions.ts` | 权限检查 |
| `../../Tool.ts` | 工具类型定义 |
| `../../tools/MCPTool/MCPTool.ts` | MCP 工具实现 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
import { normalizeNameForMCP } from './normalization.js'
```

### 5.2 命名规范

```
完整工具名格式：mcp__{normalizedServerName}__{normalizedToolName}

示例：
- 原始服务器名："My Server"
- 规范化后："My_Server"
- 原始工具名："create-issue"
- 规范化后："create-issue"（已是合法字符）
- 完整工具名："mcp__My_Server__create-issue"
```

### 5.3 规范化规则

```typescript
// 来自 normalization.ts
export function normalizeNameForMCP(name: string): string {
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }
  return normalized
}
```

**规则**：
- 非法字符（非 `a-zA-Z0-9_-`）替换为 `_`
- Claude.ai 服务器特殊处理：合并连续下划线，去除首尾下划线

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 解析边界
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 双下划线歧义 | 服务器名含 `__` 时解析错误 | 文档说明，实际中罕见 |
| 大小写敏感 | 服务器名大小写影响规范化 | 规范化函数统一处理 |
| 空工具名 | `mcp__server__` 解析为空工具名 | 业务逻辑处理 |

#### 6.1.2 已知限制
```typescript
/*
 * Known limitation: If a server name contains "__", parsing will be incorrect.
 * For example, "mcp__my__server__tool" would parse as server="my" and tool="server__tool"
 * instead of server="my__server" and tool="tool". This is rare in practice since server
 * names typically don't contain double underscores.
 */
```

### 6.2 潜在问题

#### 6.2.1 正则表达式性能
```typescript
userFacingName.replace(/\s*\(MCP\)\s*$/, '')
// 简单正则，性能影响可忽略
```

#### 6.2.2 字符串操作
```typescript
const parts = toolString.split('__')
// 每次调用创建新数组，对高频调用可能有影响
```

### 6.3 改进建议

#### 6.3.1 功能增强
1. **严格解析模式**：
   ```typescript
   export function mcpInfoFromString(
     toolString: string,
     strict = false
   ): { serverName: string; toolName?: string } | null {
     const parts = toolString.split('__')
     if (strict && parts.length !== 3) {
       return null  // 严格模式要求恰好 3 部分
     }
     // ...
   }
   ```

2. **反向验证**：
   ```typescript
   export function isValidMcpToolName(toolString: string): boolean {
     return mcpInfoFromString(toolString) !== null
   }
   ```

3. **服务器级别检测**：
   ```typescript
   export function isServerLevelTool(toolString: string): boolean {
     const info = mcpInfoFromString(toolString)
     return info !== null && info.toolName === undefined
   }
   ```

#### 6.3.2 性能优化
1. **缓存常用前缀**：
   ```typescript
   const prefixCache = new Map<string, string>()
   
   export function getMcpPrefix(serverName: string): string {
     const cached = prefixCache.get(serverName)
     if (cached) return cached
     
     const prefix = `mcp__${normalizeNameForMCP(serverName)}__`
     prefixCache.set(serverName, prefix)
     return prefix
   }
   ```

2. **正则预编译**：
   ```typescript
   const MCP_SUFFIX_REGEX = /\s*\(MCP\)\s*$/g
   
   export function extractMcpToolDisplayName(userFacingName: string): string {
     let withoutSuffix = userFacingName.replace(MCP_SUFFIX_REGEX, '')
     // ...
   }
   ```

#### 6.3.3 代码质量
1. **单元测试**：
   ```typescript
   describe('mcpInfoFromString', () => {
     it('parses valid tool name', () => {
       expect(mcpInfoFromString('mcp__github__create_issue'))
         .toEqual({ serverName: 'github', toolName: 'create_issue' })
     })
     
     it('handles server-level name', () => {
       expect(mcpInfoFromString('mcp__github'))
         .toEqual({ serverName: 'github', toolName: undefined })
     })
     
     it('returns null for non-mcp prefix', () => {
       expect(mcpInfoFromString('other__github__tool')).toBeNull()
     })
     
     it('handles double underscore in tool name', () => {
       expect(mcpInfoFromString('mcp__github__my__tool'))
         .toEqual({ serverName: 'github', toolName: 'my__tool' })
     })
   })
   ```

2. **类型安全**：
   ```typescript
   // 使用 branded type
   type McpToolName = string & { __brand: 'McpToolName' }
   type ServerName = string & { __brand: 'ServerName' }
   
   export function buildMcpToolName(
     serverName: ServerName,
     toolName: string
   ): McpToolName {
     return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}` as McpToolName
   }
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| 标准工具名解析 | 高 |
| 服务器级别解析 | 高 |
| 工具名构建 | 高 |
| 权限检查工具名 | 高 |
| 显示名提取 | 中 |
| 边界情况（空字符串、特殊字符） | 中 |
| 性能测试（高频调用） | 低 |
