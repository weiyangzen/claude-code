# MCP Headers 助手 (headersHelper.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`headersHelper.ts` 是 Claude Code MCP 系统的**动态 HTTP Headers 管理工具**，支持通过外部脚本动态获取 MCP 服务器的请求头：
- **动态认证**：支持需要动态生成认证头的场景（如 AWS SigV4、临时令牌）
- **安全执行**：在信任检查通过后执行外部脚本
- **Header 合并**：动态 headers 覆盖静态配置的 headers
- **错误容错**：执行失败不阻断连接，返回 null

### 1.2 业务场景
| 场景 | 说明 |
|------|------|
| AWS 签名 | 使用 `aws-sigv4` 动态生成签名头 |
| 临时令牌 | 从本地凭证管理器获取临时访问令牌 |
| 多租户 | 根据环境动态选择租户标识 |
| 凭证轮换 | 自动获取最新凭证，无需重启 |

### 1.3 设计原则
- **安全优先**：项目/本地配置的脚本需要信任确认
- **Git 凭证助手风格**：通过环境变量传递上下文，支持一个脚本服务多个服务器
- **容错设计**：脚本失败不阻断连接，降级到静态 headers
- **超时控制**：10 秒超时防止脚本挂起

---

## 2. 功能点目的

### 2.1 动态 Header 获取
**目的**：支持 MCP 服务器需要动态生成请求头的场景。

**典型用例**：
- AWS 服务的 SigV4 签名
- 短期访问令牌（STS、OAuth 设备流）
- 基于当前环境的动态路由

### 2.2 安全执行控制
**目的**：防止恶意项目通过 `headersHelper` 执行任意代码。

**安全措施**：
- 项目/本地配置的脚本需要工作区信任确认
- 非交互模式（CI/CD）跳过信任检查
- 执行超时限制（10 秒）

### 2.3 Header 合并策略
**目的**：支持静态和动态 headers 的组合使用。

**合并规则**：
- 动态 headers 覆盖同名的静态 headers
- 保留静态配置中动态未提供的 headers

### 2.4 脚本上下文传递
**目的**：支持一个脚本服务多个 MCP 服务器（Git 凭证助手风格）。

**环境变量**：
- `CLAUDE_CODE_MCP_SERVER_NAME`：当前服务器名称
- `CLAUDE_CODE_MCP_SERVER_URL`：当前服务器 URL

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 支持的远程服务器配置类型
type RemoteMcpConfig = 
  | McpSSEServerConfig      // { type: 'sse', url, headers?, headersHelper?, oauth? }
  | McpHTTPServerConfig     // { type: 'http', url, headers?, headersHelper?, oauth? }
  | McpWebSocketServerConfig // { type: 'ws', url, headers?, headersHelper? }
```

### 3.2 安全来源检查

```typescript
/**
 * Check if the MCP server config comes from project settings (projectSettings or localSettings)
 * This is important for security checks
 */
function isMcpServerFromProjectOrLocalSettings(
  config: ScopedMcpServerConfig
): boolean {
  return config.scope === 'project' || config.scope === 'local'
}
```

### 3.3 动态 Headers 获取

```typescript
export async function getMcpHeadersFromHelper(
  serverName: string,
  config: McpSSEServerConfig | McpHTTPServerConfig | McpWebSocketServerConfig
): Promise<Record<string, string> | null> {
  // 1. 检查是否配置了 headersHelper
  if (!config.headersHelper) {
    return null
  }

  // 2. 安全检查（项目/本地配置需要信任确认）
  if (
    'scope' in config &&
    isMcpServerFromProjectOrLocalSettings(config as ScopedMcpServerConfig) &&
    !getIsNonInteractiveSession()
  ) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust) {
      const error = new Error(
        `Security: headersHelper for MCP server '${serverName}' executed before workspace trust is confirmed.`
      )
      logAntError('MCP headersHelper invoked before trust check', error)
      logEvent('tengu_mcp_headersHelper_missing_trust', {})
      return null
    }
  }

  try {
    // 3. 执行脚本
    logMCPDebug(serverName, 'Executing headersHelper to get dynamic headers')
    const execResult = await execFileNoThrowWithCwd(config.headersHelper, [], {
      shell: true,           // 允许 shell 脚本
      timeout: 10000,        // 10 秒超时
      env: {
        ...process.env,
        CLAUDE_CODE_MCP_SERVER_NAME: serverName,  // 传递服务器名称
        CLAUDE_CODE_MCP_SERVER_URL: config.url,   // 传递服务器 URL
      },
    })

    // 4. 检查执行结果
    if (execResult.code !== 0 || !execResult.stdout) {
      throw new Error(
        `headersHelper for MCP server '${serverName}' did not return a valid value`
      )
    }
    const result = execResult.stdout.trim()

    // 5. 解析 JSON 输出
    const headers = jsonParse(result)
    if (
      typeof headers !== 'object' ||
      headers === null ||
      Array.isArray(headers)
    ) {
      throw new Error(
        `headersHelper for MCP server '${serverName}' must return a JSON object with string key-value pairs`
      )
    }

    // 6. 验证所有值为字符串
    for (const [key, value] of Object.entries(headers)) {
      if (typeof value !== 'string') {
        throw new Error(
          `headersHelper for MCP server '${serverName}' returned non-string value for key "${key}": ${typeof value}`
        )
      }
    }

    logMCPDebug(
      serverName,
      `Successfully retrieved ${Object.keys(headers).length} headers from headersHelper`
    )
    return headers as Record<string, string>
  } catch (error) {
    // 7. 错误处理：记录但不阻断
    logMCPError(
      serverName,
      `Error getting headers from headersHelper: ${errorMessage(error)}`
    )
    logError(
      new Error(
        `Error getting MCP headers from headersHelper for server '${serverName}': ${errorMessage(error)}`
      )
    )
    return null
  }
}
```

### 3.4 Header 合并

```typescript
export async function getMcpServerHeaders(
  serverName: string,
  config: McpSSEServerConfig | McpHTTPServerConfig | McpWebSocketServerConfig
): Promise<Record<string, string>> {
  const staticHeaders = config.headers || {}
  const dynamicHeaders =
    (await getMcpHeadersFromHelper(serverName, config)) || {}

  // Dynamic headers override static headers if both are present
  return {
    ...staticHeaders,
    ...dynamicHeaders,
  }
}
```

### 3.5 使用示例

#### 3.5.1 配置示例
```json
{
  "mcpServers": {
    "aws-service": {
      "type": "http",
      "url": "https://api.aws.example.com",
      "headers": {
        "X-Custom-Header": "static-value"
      },
      "headersHelper": "/path/to/aws-sigv4-helper.sh"
    }
  }
}
```

#### 3.5.2 Helper 脚本示例
```bash
#!/bin/bash
# aws-sigv4-helper.sh
# 环境变量：CLAUDE_CODE_MCP_SERVER_NAME, CLAUDE_CODE_MCP_SERVER_URL

# 生成 AWS SigV4 签名
SIGNATURE=$(aws sigv4 sign --url "$CLAUDE_CODE_MCP_SERVER_URL")

# 输出 JSON 格式的 headers
echo '{
  "Authorization": "'"$SIGNATURE"'",
  "X-Amz-Date": "'"$(date -u +%Y%m%dT%H%M%SZ)"'"
}'
```

#### 3.5.3 Node.js Helper 示例
```javascript
#!/usr/bin/env node
// token-helper.js

const serverName = process.env.CLAUDE_CODE_MCP_SERVER_NAME;
const serverUrl = process.env.CLAUDE_CODE_MCP_SERVER_URL;

// 从本地凭证存储获取令牌
const token = getTokenFromStore(serverName);

console.log(JSON.stringify({
  'Authorization': `Bearer ${token}`,
  'X-Request-URL': serverUrl
}));
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `getMcpHeadersFromHelper` | 32-117 | 从 helper 脚本获取动态 headers |
| `getMcpServerHeaders` | 125-138 | 获取合并后的 headers |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `types.ts` | McpSSEServerConfig、McpHTTPServerConfig、McpWebSocketServerConfig、ScopedMcpServerConfig |
| `../../bootstrap/state.js` | getIsNonInteractiveSession |
| `../../utils/config.js` | checkHasTrustDialogAccepted |
| `../../utils/execFileNoThrow.js` | execFileNoThrowWithCwd |
| `../../utils/log.js` | logMCPDebug、logMCPError、logError |
| `../../utils/slowOperations.js` | jsonParse |
| `../analytics/index.js` | logEvent |

### 4.3 调用方

| 文件 | 用途 |
|------|------|
| `client.ts` | 连接 SSE/HTTP/WebSocket 服务器时获取 headers |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
import { getIsNonInteractiveSession } from '../../bootstrap/state.js'
import { checkHasTrustDialogAccepted } from '../../utils/config.js'
import { logAntError } from '../../utils/debug.js'
import { errorMessage } from '../../utils/errors.js'
import { execFileNoThrowWithCwd } from '../../utils/execFileNoThrow.js'
import { logError, logMCPDebug, logMCPError } from '../../utils/log.js'
import { jsonParse } from '../../utils/slowOperations.js'
import { logEvent } from '../analytics/index.js'
```

### 5.2 执行环境

```typescript
execFileNoThrowWithCwd(config.headersHelper, [], {
  shell: true,           // 使用 shell 执行
  timeout: 10000,        // 10 秒超时
  env: {
    ...process.env,      // 继承当前环境
    CLAUDE_CODE_MCP_SERVER_NAME: serverName,
    CLAUDE_CODE_MCP_SERVER_URL: config.url,
  },
})
```

### 5.3 信任检查流程

```
getMcpHeadersFromHelper()
├── 检查 headersHelper 是否存在
├── 检查配置来源
│   ├── project/local 作用域？
│   │   ├── 是 → 检查信任状态
│   │   │   ├── 已信任 → 继续执行
│   │   │   └── 未信任 → 返回 null，记录事件
│   │   └── 否 → 继续执行
│   └── 非交互模式？
│       ├── 是 → 跳过信任检查
│       └── 否 → 执行信任检查
├── 执行脚本（10秒超时）
├── 解析 JSON 输出
└── 返回 headers 或 null
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 安全风险
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 任意代码执行 | `headersHelper` 可以执行任意命令 | 项目/本地配置需要信任确认 |
| 凭证泄露 | Helper 脚本可能泄露敏感信息 | 脚本输出仅用于 headers，不记录 |
| 命令注入 | 服务器名称/URL 可能包含恶意字符 | 通过环境变量传递，非命令行参数 |
| 超时绕过 | 长时间运行的脚本可能消耗资源 | 10 秒超时限制 |

#### 6.1.2 功能边界
| 边界 | 说明 |
|------|------|
| 仅支持远程服务器 | stdio 服务器不支持（无 HTTP headers） |
| 仅支持字符串值 | Header 值必须是字符串 |
| 单次执行 | 每个连接只执行一次，不自动刷新 |
| 同步输出 | 依赖 stdout 输出，不支持异步回调 |

### 6.2 潜在问题

#### 6.2.1 Shell 注入风险
```typescript
// 当前实现使用 shell: true
execFileNoThrowWithCwd(config.headersHelper, [], { shell: true })

// 风险：如果 headersHelper 包含 shell 元字符
// "headersHelper": "; rm -rf / #"

// 缓解：信任检查，但建议额外验证
```

#### 6.2.2 环境变量继承
```typescript
env: {
  ...process.env,  // 继承所有环境变量
  CLAUDE_CODE_MCP_SERVER_NAME: serverName,
  CLAUDE_CODE_MCP_SERVER_URL: config.url,
}

// 风险：Helper 脚本可以访问所有环境变量，包括敏感信息
// 缓解：这是设计行为，Helper 需要这些信息
```

#### 6.2.3 错误静默处理
```typescript
catch (error) {
  // 记录但不抛出，返回 null
  return null
}

// 风险：用户可能不知道 headers 获取失败
// 缓解：有日志记录，但 UI 无明确提示
```

### 6.3 改进建议

#### 6.3.1 安全增强
1. **路径验证**：
   ```typescript
   import { resolve } from 'path'
   
   function validateHelperPath(helperPath: string): boolean {
     const resolved = resolve(helperPath)
     // 限制在特定目录内
     return resolved.startsWith(ALLOWED_HELPERS_DIR)
   }
   ```

2. **沙箱执行**：
   ```typescript
   // 使用受限的 shell 环境
   execFileNoThrowWithCwd('/bin/sh', ['-c', config.headersHelper], {
     shell: false,  // 更安全
     // ...
   })
   ```

3. **校验和验证**：
   ```typescript
   // 配置中添加 expectedChecksum
   if (config.headersHelperChecksum) {
     const actual = computeChecksum(config.headersHelper)
     if (actual !== config.headersHelperChecksum) {
       throw new Error('Helper script checksum mismatch')
     }
   }
   ```

#### 6.3.2 功能扩展
1. **缓存机制**：
   ```typescript
   const headersCache = new Map<string, { headers: Record<string, string>, expiresAt: number }>()
   
   export async function getMcpHeadersFromHelper(
     serverName: string,
     config: RemoteMcpConfig,
     cacheTtlMs = 60000
   ): Promise<Record<string, string> | null> {
     const cacheKey = `${serverName}:${config.headersHelper}`
     const cached = headersCache.get(cacheKey)
     if (cached && cached.expiresAt > Date.now()) {
       return cached.headers
     }
     // ... 获取新 headers
     headersCache.set(cacheKey, { headers, expiresAt: Date.now() + cacheTtlMs })
   }
   ```

2. **异步刷新**：
   ```typescript
   // 支持后台刷新即将过期的 headers
   // 避免同步等待导致的延迟
   ```

3. **Header 模板**：
   ```typescript
   // 支持在配置中定义模板，减少脚本需求
   "headers": {
     "Authorization": "Bearer ${DYNAMIC_TOKEN}"
   },
   "headersHelper": {
     "type": "env",
     "vars": ["DYNAMIC_TOKEN"]
   }
   ```

#### 6.3.3 可观测性
1. **性能指标**：
   ```typescript
   const startTime = Date.now()
   const result = await execFileNoThrowWithCwd(...)
   logEvent('tengu_mcp_headersHelper_duration', {
     serverName,
     durationMs: Date.now() - startTime,
     success: result.code === 0
   })
   ```

2. **详细日志**：
   ```typescript
   logMCPDebug(serverName, `headersHelper stdout: ${result.stdout}`)
   logMCPDebug(serverName, `headersHelper stderr: ${result.stderr}`)
   ```

#### 6.3.4 代码质量
1. **类型安全**：
   ```typescript
   // 使用更精确的返回类型
type HeadersResult = 
     | { success: true; headers: Record<string, string> }
     | { success: false; error: string }
   ```

2. **单元测试**：
   ```typescript
   describe('getMcpHeadersFromHelper', () => {
     it('returns null when no helper configured', async () => { ... })
     it('requires trust for project scope', async () => { ... })
     it('parses valid JSON output', async () => { ... })
     it('validates string values', async () => { ... })
     it('handles timeout', async () => { ... })
   })
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| 信任检查逻辑 | 高 |
| JSON 解析和验证 | 高 |
| Header 合并 | 高 |
| 超时处理 | 中 |
| 错误恢复 | 中 |
| 并发执行 | 低 |
