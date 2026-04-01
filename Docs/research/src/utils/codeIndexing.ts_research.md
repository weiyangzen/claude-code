# src/utils/codeIndexing.ts 深度研究文档

## 场景与职责

`codeIndexing.ts` 是 Claude Code 的代码索引工具检测模块，用于识别用户是否在使用其他代码索引/搜索工具。这主要用于：
- 分析工具使用情况（遥测）
- 了解用户的开发工具栈
- 检测潜在的冲突或互补工具

检测的场景包括：
- Bash 命令中调用的代码索引 CLI 工具
- MCP 服务器名称匹配代码索引工具
- MCP 工具名称匹配代码索引工具

## 功能点目的

### 1. 代码索引工具识别
- 维护已知代码索引工具的标识符列表
- 包括代码搜索引擎、AI 编程助手、MCP 代码索引服务器等

### 2. CLI 命令检测
- 从 Bash 命令中提取工具名称
- 处理 `npx`/`bunx` 前缀的命令
- 匹配已知的 CLI 命令到工具标识符

### 3. MCP 服务器/工具检测
- 从 MCP 服务器名称识别代码索引工具
- 从 MCP 工具名称（格式：`mcp__serverName__toolName`）识别
- 使用正则表达式模式匹配（支持大小写不敏感）

## 具体技术实现

### 核心数据结构

```typescript
// 代码索引工具标识符
export type CodeIndexingTool =
  // 代码搜索引擎
  | 'sourcegraph'
  | 'hound'
  | 'seagoat'
  | 'bloop'
  | 'gitloop'
  // AI 编程助手
  | 'cody'
  | 'aider'
  | 'continue'
  | 'github-copilot'
  | 'cursor'
  | 'tabby'
  | 'codeium'
  | 'tabnine'
  | 'augment'
  | 'windsurf'
  | 'aide'
  | 'pieces'
  | 'qodo'
  | 'amazon-q'
  | 'gemini'
  // MCP 代码索引服务器
  | 'claude-context'
  | 'code-index-mcp'
  | 'local-code-search'
  | 'autodev-codebase'
  // 上下文提供者
  | 'openctx'
```

### CLI 命令映射

```typescript
const CLI_COMMAND_MAPPING: Record<string, CodeIndexingTool> = {
  // Sourcegraph 生态
  src: 'sourcegraph',
  cody: 'cody',
  // AI 编程助手
  aider: 'aider',
  tabby: 'tabby',
  tabnine: 'tabnine',
  augment: 'augment',
  pieces: 'pieces',
  qodo: 'qodo',
  aide: 'aide',
  // 代码搜索工具
  hound: 'hound',
  seagoat: 'seagoat',
  bloop: 'bloop',
  gitloop: 'gitloop',
  // 云服务商 AI 助手
  q: 'amazon-q',
  gemini: 'gemini',
}
```

### MCP 服务器模式

```typescript
const MCP_SERVER_PATTERNS: Array<{
  pattern: RegExp
  tool: CodeIndexingTool
}> = [
  // Sourcegraph 生态
  { pattern: /^sourcegraph$/i, tool: 'sourcegraph' },
  { pattern: /^cody$/i, tool: 'cody' },
  { pattern: /^openctx$/i, tool: 'openctx' },
  // AI 编程助手
  { pattern: /^aider$/i, tool: 'aider' },
  { pattern: /^continue$/i, tool: 'continue' },
  { pattern: /^github[-_]?copilot$/i, tool: 'github-copilot' },
  { pattern: /^copilot$/i, tool: 'github-copilot' },
  { pattern: /^cursor$/i, tool: 'cursor' },
  { pattern: /^augment[-_]?code$/i, tool: 'augment' },
  { pattern: /^windsurf$/i, tool: 'windsurf' },
  { pattern: /^codestory$/i, tool: 'aide' },
  // ... 更多模式
]
```

### 检测函数

#### CLI 命令检测
```typescript
export function detectCodeIndexingFromCommand(
  command: string,
): CodeIndexingTool | undefined {
  const trimmed = command.trim()
  const firstWord = trimmed.split(/\s+/)[0]?.toLowerCase()

  if (!firstWord) return undefined

  // 处理 npx/bunx 前缀
  if (firstWord === 'npx' || firstWord === 'bunx') {
    const secondWord = trimmed.split(/\s+/)[1]?.toLowerCase()
    if (secondWord && secondWord in CLI_COMMAND_MAPPING) {
      return CLI_COMMAND_MAPPING[secondWord]
    }
  }

  return CLI_COMMAND_MAPPING[firstWord]
}
```

#### MCP 工具检测
```typescript
export function detectCodeIndexingFromMcpTool(
  toolName: string,
): CodeIndexingTool | undefined {
  // MCP 工具名格式：mcp__serverName__toolName
  if (!toolName.startsWith('mcp__')) return undefined

  const parts = toolName.split('__')
  if (parts.length < 3) return undefined

  const serverName = parts[1]
  if (!serverName) return undefined

  for (const { pattern, tool } of MCP_SERVER_PATTERNS) {
    if (pattern.test(serverName)) return tool
  }

  return undefined
}
```

#### MCP 服务器名称检测
```typescript
export function detectCodeIndexingFromMcpServerName(
  serverName: string,
): CodeIndexingTool | undefined {
  for (const { pattern, tool } of MCP_SERVER_PATTERNS) {
    if (pattern.test(serverName)) return tool
  }
  return undefined
}
```

## 依赖与外部交互

### 无外部依赖

此模块是纯数据映射，不依赖任何外部模块。

### 被依赖方

| 模块 | 用途 |
|------|------|
| `src/tools/BashTool/BashTool.tsx` | 检测 Bash 命令中的代码索引工具 |
| `src/services/mcp/client.ts` | 检测 MCP 服务器和工具 |

## 风险、边界与改进建议

### 已知风险

1. **误报/漏报**
   - 简单的字符串匹配可能导致误报（如 `src` 命令可能是 sourcegraph，也可能是普通的 `src` 目录操作）
   - 新工具或变体名称可能漏报

2. **维护负担**
   - 代码索引工具生态快速发展，需要持续更新列表
   - 正则表达式模式需要测试和维护

3. **命名冲突**
   - 某些工具名称可能与其他工具冲突（如 `q` 可能是 amazon-q，也可能是其他工具）

### 边界情况

1. **命令参数**
   - 只检查命令的第一个词，忽略参数
   - 例如 `src search "pattern"` 能识别，但 `cd src && something` 不会误识别

2. **大小写处理**
   - CLI 命令映射使用小写匹配
   - MCP 模式使用 `/i` 标志进行大小写不敏感匹配

3. **部分匹配**
   - MCP 模式使用 `^...$` 锚定，避免部分匹配
   - 例如 `my-sourcegraph` 不会匹配 `sourcegraph` 模式

4. **MCP 工具格式**
   - 严格检查 `mcp__` 前缀和 `__` 分隔符
   - 格式不符返回 undefined

### 改进建议

1. **减少误报**
   - 对 CLI 命令添加更多上下文检查（如检查命令路径）
   - 使用更具体的模式（如 `^src$` 而不是简单的词匹配）
   - 添加排除列表（已知的非索引工具使用相同名称）

2. **扩展覆盖**
   - 添加更多代码索引工具（如 `ripgrep` 虽然不是索引工具但用于代码搜索）
   - 支持工具别名（如 `sg` 作为 sourcegraph 的别名）
   - 添加版本检测（某些工具的不同版本行为不同）

3. **动态更新**
   - 考虑从远程配置加载工具列表（便于更新）
   - 添加用户自定义工具映射

4. **遥测增强**
   - 记录检测到的工具使用频率
   - 分析工具组合（哪些工具经常一起使用）

5. **代码质量**
   - 将工具列表和模式分离到配置文件
   - 添加单元测试验证模式匹配
   - 生成文档（工具列表和描述）

6. **代码示例**

```typescript
// 改进版本（带置信度分数）
export type DetectionResult = {
  tool: CodeIndexingTool
  confidence: 'high' | 'medium' | 'low'
  reason: string
}

export function detectCodeIndexingFromCommand(
  command: string,
  context?: { cwd: string; env: Record<string, string> }
): DetectionResult | undefined {
  const trimmed = command.trim()
  const firstWord = trimmed.split(/\s+/)[0]?.toLowerCase()
  
  if (!firstWord) return undefined
  
  // 高置信度：直接匹配已知命令
  if (firstWord in CLI_COMMAND_MAPPING) {
    // 额外检查：npx/bunx 前缀增加置信度
    const isNpx = firstWord === 'npx' || firstWord === 'bunx'
    const actualCommand = isNpx 
      ? trimmed.split(/\s+/)[1]?.toLowerCase()
      : firstWord
      
    if (actualCommand && actualCommand in CLI_COMMAND_MAPPING) {
      return {
        tool: CLI_COMMAND_MAPPING[actualCommand],
        confidence: isNpx ? 'high' : 'medium',
        reason: isNpx ? 'npx/bunx prefix' : 'direct command match'
      }
    }
  }
  
  // 低置信度：路径包含工具名
  if (context?.cwd.includes('sourcegraph')) {
    return { tool: 'sourcegraph', confidence: 'low', reason: 'path heuristic' }
  }
  
  return undefined
}
```
