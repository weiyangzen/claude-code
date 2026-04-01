# subprocessEnv.ts 研究文档

## 场景与职责

`subprocessEnv.ts` 负责管理子进程的环境变量，特别是在 GitHub Actions 环境中防止敏感信息泄露。该模块实现了环境变量清理机制，确保在可能存在提示注入攻击风险的环境中，子进程无法访问敏感凭证。

## 功能点目的

### GitHub Actions 环境安全
- **威胁模型**: 在 GitHub Actions 中，不受信任的内容（如 PR 代码）可能通过 shell 扩展（如 `${ANTHROPIC_API_KEY}`）窃取密钥
- **解决方案**: 在子进程中清理敏感环境变量，同时保留父进程访问权限
- **范围**: 仅影响子进程（bash、shell snapshot、MCP stdio、LSP、hooks），父进程保持完整访问

### 代理环境注入
- **功能**: 在 CCR (Claude Code Remote) 环境中注入上游代理设置
- **机制**: 通过注册的回调函数获取代理环境变量
- **目的**: 确保子进程中的 curl/gh/python 等工具通过本地中继路由

## 具体技术实现

### 清理的环境变量列表

```typescript
const GHA_SUBPROCESS_SCRUB = [
  // Anthropic 认证
  'ANTHROPIC_API_KEY',
  'CLAUDE_CODE_OAUTH_TOKEN',
  'ANTHROPIC_AUTH_TOKEN',
  'ANTHROPIC_FOUNDRY_API_KEY',
  'ANTHROPIC_CUSTOM_HEADERS',

  // OTLP 导出器头（可能包含 Bearer Token）
  'OTEL_EXPORTER_OTLP_HEADERS',
  'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
  'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
  'OTEL_EXPORTER_OTLP_TRACES_HEADERS',

  // 云提供商凭证
  'AWS_SECRET_ACCESS_KEY',
  'AWS_SESSION_TOKEN',
  'AWS_BEARER_TOKEN_BEDROCK',
  'GOOGLE_APPLICATION_CREDENTIALS',
  'AZURE_CLIENT_SECRET',
  'AZURE_CLIENT_CERTIFICATE_PATH',

  // GitHub Actions OIDC
  'ACTIONS_ID_TOKEN_REQUEST_TOKEN',
  'ACTIONS_ID_TOKEN_REQUEST_URL',

  // GitHub Actions 运行时 Token
  'ACTIONS_RUNTIME_TOKEN',
  'ACTIONS_RUNTIME_URL',

  // Claude Code Action 特定变量
  'ALL_INPUTS',
  'OVERRIDE_GITHUB_TOKEN',
  'DEFAULT_WORKFLOW_TOKEN',
  'SSH_SIGNING_KEY',
] as const
```

### 核心函数

```typescript
export function subprocessEnv(): NodeJS.ProcessEnv {
  // 获取代理环境（CCR 环境）
  const proxyEnv = _getUpstreamProxyEnv?.() ?? {}

  // 检查是否需要清理
  if (!isEnvTruthy(process.env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB)) {
    return Object.keys(proxyEnv).length > 0
      ? { ...process.env, ...proxyEnv }
      : process.env
  }

  // 清理敏感变量
  const env = { ...process.env, ...proxyEnv }
  for (const k of GHA_SUBPROCESS_SCRUB) {
    delete env[k]
    delete env[`INPUT_${k}`]  // GitHub Actions 自动创建的 INPUT_ 前缀变量
  }
  return env
}
```

### 代理注册机制

```typescript
let _getUpstreamProxyEnv: (() => Record<string, string>) | undefined

export function registerUpstreamProxyEnvFn(
  fn: () => Record<string, string>,
): void {
  _getUpstreamProxyEnv = fn
}
```

## 关键代码路径与文件引用

### 本文件导出
- `subprocessEnv()`: 获取清理后的环境变量
- `registerUpstreamProxyEnvFn(fn)`: 注册代理环境获取函数

### 依赖模块

| 模块 | 用途 |
|------|------|
| `./envUtils.js` | `isEnvTruthy` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/utils/Shell.ts` | 执行 Bash 命令时使用 |
| `src/services/mcp/client.ts` | MCP 服务器连接时使用 |
| `src/services/lsp/LSPClient.ts` | LSP 服务器连接时使用 |
| `src/upstreamproxy/upstreamproxy.ts` | 注册代理环境函数 |
| `src/utils/bash/ShellSnapshot.ts` | Shell 快照功能 |
| `src/utils/hooks.ts` | 执行 hooks 时使用 |

## 依赖与外部交互

### 与 GitHub Actions 的集成
- 通过 `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` 环境变量启用
- `claude-code-action` 在配置 `allowed_non_write_users` 时自动设置
- 处理 GitHub Actions 自动创建的 `INPUT_*` 变量

### 与上游代理的集成
- 通过 `registerUpstreamProxyEnvFn` 延迟加载代理配置
- 避免在非 CCR 启动中引入上游代理模块依赖
- 在 `init.ts` 中动态导入后注册

### 与认证系统的集成
- 清理的变量包括所有 Anthropic 认证方式
- 父进程保留访问权限用于 API 调用和延迟凭证读取
- 明确保留 `GITHUB_TOKEN` / `GH_TOKEN`（工作流范围，需要用于 gh.sh 脚本）

## 风险、边界与改进建议

### 潜在风险

1. **遗漏的敏感变量**: 新添加的敏感环境变量可能未加入清理列表
2. **误清理**: 某些合法用途可能需要被清理的变量
3. **子进程绕过**: 子进程可能通过其他方式获取敏感信息（如文件系统）

### 边界情况

1. **大小写敏感**: 环境变量名是大小写敏感的，清理列表必须精确匹配
2. **空值变量**: 未设置值的变量在 `process.env` 中不存在，无需清理
3. **动态变量**: 运行时创建的环境变量不在清理范围内

### 改进建议

1. **动态敏感变量检测**: 使用正则或模式匹配检测可能的敏感变量
```typescript
const SENSITIVE_PATTERNS = [
  /API_KEY/i,
  /SECRET/i,
  /TOKEN/i,
  /PASSWORD/i,
  /PRIVATE_KEY/i,
]

function isSensitiveVar(name: string): boolean {
  return SENSITIVE_PATTERNS.some(p => p.test(name))
}
```

2. **可配置清理列表**: 允许用户或配置扩展清理列表
```typescript
export function subprocessEnv(customScrub?: string[]): NodeJS.ProcessEnv {
  const toScrub = [...GHA_SUBPROCESS_SCRUB, ...(customScrub || [])]
  // ...
}
```

3. **审计日志**: 记录哪些变量被清理（调试用）
```typescript
if (isDebugMode()) {
  logForDebugging(`Scrubbed env vars: ${scrubbedVars.join(', ')}`)
}
```

4. **文档化**: 在清理列表中添加每个变量的用途说明
```typescript
interface ScrubbedVar {
  name: string
  reason: string
  source: 'anthropic' | 'cloud' | 'github' | 'otel'
}

const GHA_SUBPROCESS_SCRUB: ScrubbedVar[] = [
  { name: 'ANTHROPIC_API_KEY', reason: 'API authentication', source: 'anthropic' },
  // ...
]
```

5. **测试覆盖**: 添加测试验证清理逻辑
```typescript
test('subprocessEnv scrubs sensitive vars when CLAUDE_CODE_SUBPROCESS_ENV_SCRUB is set', () => {
  process.env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB = '1'
  process.env.ANTHROPIC_API_KEY = 'secret'
  const env = subprocessEnv()
  expect(env.ANTHROPIC_API_KEY).toBeUndefined()
})
```

6. **与 secret scanning 集成**: 与 `secretScanner.ts` 共享敏感变量模式
