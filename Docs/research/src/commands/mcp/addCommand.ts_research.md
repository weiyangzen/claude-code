# MCP Add Command 研究文档

## 场景与职责

`addCommand.ts` 实现了 `claude mcp add` CLI 子命令，用于向 Claude Code 添加 MCP (Model Context Protocol) 服务器配置。这是用户通过命令行界面配置 MCP 服务器的核心入口点，支持多种传输协议（stdio、SSE、HTTP）和高级认证机制（OAuth、XAA）。

该文件从 `main.tsx` 中提取出来，以便进行直接测试，体现了模块化和可测试性的设计原则。

## 功能点目的

### 1. 服务器添加功能
- **stdio 服务器**: 通过命令行启动的本地进程（如 `npx my-mcp-server`）
- **SSE 服务器**: Server-Sent Events 远程服务器
- **HTTP 服务器**: 基于 HTTP 的 MCP 服务器

### 2. 配置范围支持
支持三种配置存储范围：
- `local` (默认): 项目本地配置（存储在 `.claude/settings.json`）
- `user`: 用户全局配置（存储在 `~/.claude/settings.json`）
- `project`: 项目共享配置（存储在项目根目录的 `.mcp.json`）

### 3. 认证机制
- **OAuth 2.0**: 支持 `client-id`、`client-secret`、`callback-port` 配置
- **XAA (Cross-App Access / SEP-990)**: 企业级单点登录机制
  - 需要预先运行 `claude mcp xaa setup` 配置 IdP
  - 需要 `--client-id` 和 `--client-secret`
  - 通过环境变量 `CLAUDE_CODE_ENABLE_XAA=1` 启用

### 4. 环境变量与请求头
- `-e, --env`: 为 stdio 服务器设置环境变量（如 `-e API_KEY=xxx`）
- `-H, --header`: 为 HTTP/SSE 服务器设置请求头（如 `-H "Authorization: Bearer ..."`）

## 具体技术实现

### 命令注册与参数解析

```typescript
export function registerMcpAddCommand(mcp: Command): void {
  mcp
    .command('add <name> <commandOrUrl> [args...]')
    .description('Add an MCP server to Claude Code...')
    .option('-s, --scope <scope>', 'Configuration scope', 'local')
    .option('-t, --transport <transport>', 'Transport type')
    .option('-e, --env <env...>', 'Set environment variables')
    .option('-H, --header <header...>', 'Set WebSocket headers')
    .option('--client-id <clientId>', 'OAuth client ID')
    .option('--client-secret', 'Prompt for OAuth client secret')
    .option('--callback-port <port>', 'Fixed port for OAuth callback')
    .option('--xaa', 'Enable XAA (SEP-990)')
    .action(async (name, commandOrUrl, args, options) => { ... })
}
```

### 关键流程

#### 1. XAA 验证流程
```typescript
// XAA fail-fast: validate at add-time, not auth-time
if (options.xaa && !isXaaEnabled()) {
  cliError('Error: --xaa requires CLAUDE_CODE_ENABLE_XAA=1 in your environment')
}
if (xaa) {
  const missing: string[] = []
  if (!options.clientId) missing.push('--client-id')
  if (!options.clientSecret) missing.push('--client-secret')
  if (!getXaaIdpSettings()) {
    missing.push("'claude mcp xaa setup' (settings.xaaIdp not configured)")
  }
  if (missing.length) {
    cliError(`Error: --xaa requires: ${missing.join(', ')}`)
  }
}
```

#### 2. 传输类型检测与配置构建

**SSE 传输:**
```typescript
const serverConfig = {
  type: 'sse' as const,
  url: actualCommand,
  headers,
  oauth: options.clientId || callbackPort || xaa
    ? { clientId: options.clientId, callbackPort, xaa: true }
    : undefined,
}
await addMcpConfig(name, serverConfig, scope)
```

**HTTP 传输:**
与 SSE 类似，但 `type: 'http'`

**stdio 传输:**
```typescript
const env = parseEnvVars(options.env)
await addMcpConfig(
  name,
  { type: 'stdio', command: actualCommand, args: actualArgs, env },
  scope,
)
```

#### 3. URL 检测警告
当用户可能误将 URL 作为命令传递时，提供警告：
```typescript
const looksLikeUrl =
  actualCommand.startsWith('http://') ||
  actualCommand.startsWith('https://') ||
  actualCommand.startsWith('localhost') ||
  actualCommand.endsWith('/sse') ||
  actualCommand.endsWith('/mcp')

if (!transportExplicit && looksLikeUrl) {
  process.stderr.write(
    `\nWarning: The command "${actualCommand}" looks like a URL...`
  )
}
```

### 数据结构

#### 服务器配置类型（来自 types.ts）
```typescript
type McpSSEServerConfig = {
  type: 'sse'
  url: string
  headers?: Record<string, string>
  oauth?: {
    clientId?: string
    callbackPort?: number
    xaa?: boolean
  }
}

type McpStdioServerConfig = {
  type: 'stdio'
  command: string
  args: string[]
  env?: Record<string, string>
}
```

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/cli/exit.ts` | `cliError`, `cliOk` - CLI 退出处理 |
| `src/services/analytics/index.ts` | `logEvent` - 分析事件上报 |
| `src/services/mcp/auth.ts` | `readClientSecret`, `saveMcpClientSecret` - OAuth 密钥管理 |
| `src/services/mcp/config.ts` | `addMcpConfig` - 配置持久化 |
| `src/services/mcp/utils.ts` | `describeMcpConfigFilePath`, `ensureConfigScope`, `ensureTransport`, `parseHeaders` - 工具函数 |
| `src/services/mcp/xaaIdpLogin.ts` | `getXaaIdpSettings`, `isXaaEnabled` - XAA 功能开关 |
| `src/utils/envUtils.ts` | `parseEnvVars` - 环境变量解析 |
| `src/utils/slowOperations.ts` | `jsonStringify` - JSON 序列化 |

### 配置持久化流程
```
addCommand.ts
  → addMcpConfig(name, serverConfig, scope) [src/services/mcp/config.ts]
    → validate config with McpServerConfigSchema()
    → check policy (denylist/allowlist)
    → write to appropriate scope:
      - 'project': writeMcpjsonFile() → .mcp.json
      - 'user': saveGlobalConfig() → ~/.claude/settings.json
      - 'local': saveCurrentProjectConfig() → .claude/settings.json
```

### 密钥存储流程
```
addCommand.ts
  → readClientSecret() [src/services/mcp/auth.ts]
    → prompts for secret if --client-secret flag is set
  → saveMcpClientSecret(name, serverConfig, clientSecret)
    → stores in secure storage (keychain) keyed by serverKey
```

## 依赖与外部交互

### 外部依赖
- `@commander-js/extra-typings`: Commander.js 类型安全命令行解析

### 内部服务交互
1. **Analytics**: 上报 `tengu_mcp_add` 事件，包含传输类型、范围、是否显式指定传输等元数据
2. **Secure Storage**: 通过 `auth.ts` 读写 OAuth 客户端密钥
3. **Settings**: 验证 XAA IdP 配置是否已设置
4. **Config**: 将服务器配置写入适当的配置文件

### 环境变量
- `MCP_CLIENT_SECRET`: 用于传递 OAuth 客户端密钥（替代交互式输入）
- `CLAUDE_CODE_ENABLE_XAA`: 启用 XAA 功能

## 风险、边界与改进建议

### 风险点

1. **XAA 功能标志依赖**
   - `--xaa` 选项仅在 `CLAUDE_CODE_ENABLE_XAA=1` 时显示（`hideHelp(!isXaaEnabled())`）
   - 如果用户未设置环境变量，帮助文本中不会显示该选项
   - **建议**: 考虑在帮助文本中添加注释，说明某些高级选项需要特定环境变量

2. **密钥输入安全**
   - `--client-secret` 标志会触发交互式密码输入
   - 但环境变量 `MCP_CLIENT_SECRET` 可能在 shell 历史中暴露
   - **建议**: 考虑支持从文件或 stdin 读取密钥

3. **URL 误用检测的局限性**
   - `looksLikeUrl` 检测使用简单的字符串匹配
   - 可能产生误报（如命令恰好以 `/mcp` 结尾）
   - **建议**: 考虑使用更严格的 URL 解析验证

4. **配置验证时机**
   - XAA 配置在添加时验证，但 OAuth 配置的部分验证延迟到连接时
   - **建议**: 统一验证策略，尽可能在配置添加时发现问题

### 边界情况

1. **传输类型推断**
   - 默认传输类型为 `stdio`
   - 如果 URL 看起来像 HTTP 但未指定传输类型，会发出警告但仍继续

2. **配置范围限制**
   - 企业配置 (`enterprise`) 和动态配置 (`dynamic`) 不能通过此命令添加
   - 尝试添加会抛出错误

3. **stdio 服务器的 OAuth 选项**
   - stdio 服务器不支持 `--client-id`、`--client-secret`、`--callback-port`、`--xaa`
   - 如果用户指定了这些选项，会打印警告并忽略它们

### 改进建议

1. **增强帮助文本**
   ```typescript
   // 当前: --xaa 选项被隐藏
   // 建议: 始终显示，但标记为需要特定环境变量
   .option('--xaa', 'Enable XAA (requires CLAUDE_CODE_ENABLE_XAA=1)')
   ```

2. **支持配置文件批量导入**
   - 当前一次只能添加一个服务器
   - 考虑支持 `claude mcp add --from-file servers.json`

3. **配置验证增强**
   - 添加对 URL 可访问性的预检（可选）
   - 验证 OAuth 配置完整性（如指定了 client-id 但没有 client-secret 时警告）

4. **密钥管理改进**
   - 支持密钥轮换（添加时检测现有密钥并提示更新）
   - 支持从密钥管理服务（如 AWS Secrets Manager）读取

5. **测试覆盖**
   - 文件注释提到 "Extracted from main.tsx to enable direct testing"
   - 确保有针对 XAA 验证、传输类型检测、配置范围处理的单元测试
