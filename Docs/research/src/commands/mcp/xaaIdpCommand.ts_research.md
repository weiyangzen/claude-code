# MCP XAA IdP Command 研究文档

## 场景与职责

`xaaIdpCommand.ts` 实现了 `claude mcp xaa` CLI 子命令，用于管理 XAA (Cross-App Access / SEP-990) 的 IdP (Identity Provider) 连接配置。XAA 是一种企业级单点登录机制，允许用户通过一次 IdP 登录，无缝访问多个 MCP 服务器。

核心职责：
1. **setup**: 配置 IdP 连接（一次性设置，所有 XAA 启用的服务器共享）
2. **login**: 获取并缓存 IdP 的 id_token（通过浏览器登录或 JWT 直接注入）
3. **show**: 显示当前 IdP 连接配置状态
4. **clear**: 清除 IdP 连接配置和缓存的令牌

## 功能点目的

### 1. IdP 连接配置 (`xaa setup`)
- 配置 IdP issuer URL（支持 OIDC Discovery）
- 设置 Claude Code 在 IdP 的 client_id
- 可选配置 client_secret（用于机密客户端）
- 可选配置固定回调端口（用于不支持 RFC 8252 端口通配的 IdP）

### 2. 身份验证 (`xaa login`)
- 标准流程：打开浏览器进行 OIDC 授权码 + PKCE 登录
- 直接注入：通过 `--id-token` 直接写入预获取的 JWT（用于测试）
- 令牌缓存：id_token 缓存在钥匙串中，按 issuer 键控

### 3. 状态查看 (`xaa show`)
- 显示配置的 issuer、client_id、callback_port
- 显示 client_secret 是否已存储
- 显示登录状态（id_token 是否缓存且有效）

### 4. 配置清除 (`xaa clear`)
- 清除 settings.xaaIdp 配置
- 清除关联的钥匙串条目（id_token 和 client_secret）

## 具体技术实现

### 命令注册结构
```typescript
export function registerMcpXaaIdpCommand(mcp: Command): void {
  const xaaIdp = mcp.command('xaa').description('Manage the XAA (SEP-990) IdP connection')

  xaaIdp
    .command('setup')
    .requiredOption('--issuer <url>', 'IdP issuer URL')
    .requiredOption('--client-id <id>', "Claude Code's client_id")
    .option('--client-secret', 'Read from MCP_XAA_IDP_CLIENT_SECRET env var')
    .option('--callback-port <port>', 'Fixed loopback callback port')
    .action(options => { ... })

  xaaIdp
    .command('login')
    .option('--force', 'Ignore cached token and re-login')
    .option('--id-token <jwt>', 'Write pre-obtained JWT directly')
    .action(async options => { ... })

  xaaIdp.command('show').action(() => { ... })
  xaaIdp.command('clear').action(() => { ... })
}
```

### 关键流程详解

#### Setup 命令 - 配置验证与写入

**验证阶段（先验证，后写入）:**
```typescript
// 1. URL 格式验证
let issuerUrl: URL
try {
  issuerUrl = new URL(options.issuer)
} catch {
  return cliError(`Error: --issuer must be a valid URL`)
}

// 2. 协议安全验证（仅允许 https，localhost 例外）
if (
  issuerUrl.protocol !== 'https:' &&
  !(issuerUrl.protocol === 'http:' && isLoopback(issuerUrl.hostname))
) {
  return cliError(`Error: --issuer must use https://`)
}

// 3. 回调端口验证
const callbackPort = options.callbackPort ? parseInt(options.callbackPort, 10) : undefined
if (callbackPort !== undefined && (!Number.isInteger(callbackPort) || callbackPort <= 0)) {
  return cliError('Error: --callback-port must be a positive integer')
}

// 4. Client secret 环境变量验证
const secret = options.clientSecret ? process.env.MCP_XAA_IDP_CLIENT_SECRET : undefined
if (options.clientSecret && !secret) {
  return cliError('Error: --client-secret requires MCP_XAA_IDP_CLIENT_SECRET env var')
}
```

**旧配置清理:**
```typescript
const old = getXaaIdpSettings()
const oldIssuer = old?.issuer
const oldClientId = old?.clientId

// 写入新配置
updateSettingsForSource('userSettings', { xaaIdp: { issuer, clientId, callbackPort } })

// 清理旧钥匙串条目（仅在新配置写入成功后）
if (oldIssuer) {
  if (issuerKey(oldIssuer) !== issuerKey(options.issuer)) {
    // Issuer 改变，清理旧令牌和密钥
    clearIdpIdToken(oldIssuer)
    clearIdpClientSecret(oldIssuer)
  } else if (oldClientId !== options.clientId) {
    // 同一 issuer 但 client_id 改变，令牌和密钥失效
    clearIdpIdToken(oldIssuer)
    clearIdpClientSecret(oldIssuer)
  }
}
```

#### Login 命令 - 令牌获取

**直接注入路径（测试用）:**
```typescript
if (options.idToken) {
  const expiresAt = saveIdpIdTokenFromJwt(idp.issuer, options.idToken)
  return cliOk(`id_token cached for ${idp.issuer} (expires ${new Date(expiresAt).toISOString()})`)
}
```

**标准 OIDC 流程:**
```typescript
// 强制重新登录
if (options.force) {
  clearIdpIdToken(idp.issuer)
}

// 检查缓存
const wasCached = getCachedIdpIdToken(idp.issuer) !== undefined
if (wasCached) {
  return cliOk(`Already logged in... Use --force to re-login.`)
}

// 执行 OIDC 登录
await acquireIdpIdToken({
  idpIssuer: idp.issuer,
  idpClientId: idp.clientId,
  idpClientSecret: getIdpClientSecret(idp.issuer),
  callbackPort: idp.callbackPort,
  onAuthorizationUrl: url => { /* 显示授权 URL */ },
})
```

#### Clear 命令 - 安全清理
```typescript
// 1. 先读取 issuer（用于后续钥匙串清理）
const idp = getXaaIdpSettings()

// 2. 清除配置（使用 undefined 触发 mergeWith 删除）
updateSettingsForSource('userSettings', { xaaIdp: undefined })

// 3. 仅配置清除成功后，清理钥匙串
if (idp) {
  clearIdpIdToken(idp.issuer)
  clearIdpClientSecret(idp.issuer)
}
```

### 数据结构

**XaaIdpSettings（配置存储）:**
```typescript
type XaaIdpSettings = {
  issuer: string
  clientId: string
  callbackPort?: number
}
```

**钥匙串存储结构:**
```typescript
// id_token 缓存（按 issuer 键控）
mcpXaaIdp: {
  [issuerKey: string]: { idToken: string, expiresAt: number }
}

// client_secret 存储（按 issuer 键控）
mcpXaaIdpConfig: {
  [issuerKey: string]: { clientSecret: string }
}
```

**Issuer Key 规范化:**
```typescript
export function issuerKey(issuer: string): string {
  try {
    const u = new URL(issuer)
    u.pathname = u.pathname.replace(/\/+$/, '')  // 去除尾部斜杠
    u.host = u.host.toLowerCase()               // 小写主机名
    return u.toString()
  } catch {
    return issuer.replace(/\/+$/, '')
  }
}
```

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/cli/exit.ts` | `cliError`, `cliOk` - CLI 退出处理 |
| `src/services/mcp/xaaIdpLogin.ts` | XAA IdP 登录核心功能 |
| `src/utils/errors.ts` | `errorMessage` - 错误处理 |
| `src/utils/settings/settings.ts` | `updateSettingsForSource` - 设置持久化 |

### xaaIdpLogin.ts 核心函数
```typescript
// 功能开关
isXaaEnabled(): boolean  // 检查 CLAUDE_CODE_ENABLE_XAA

// 配置读写
getXaaIdpSettings(): XaaIdpSettings | undefined

// 令牌管理
getCachedIdpIdToken(issuer): string | undefined
saveIdpIdTokenFromJwt(issuer, jwt): number  // 返回 expiresAt
clearIdpIdToken(issuer): void

// Client Secret 管理
saveIdpClientSecret(issuer, secret): { success, warning }
getIdpClientSecret(issuer): string | undefined
clearIdpClientSecret(issuer): void

// OIDC 流程
acquireIdpIdToken(options): Promise<string>  // 浏览器登录
discoverOidc(issuer): Promise<OpenIdProviderDiscoveryMetadata>
issuerKey(issuer): string  // 规范化 issuer 作为键
```

### 配置存储流程
```
xaaIdpCommand.ts
  → updateSettingsForSource('userSettings', { xaaIdp: {...} })
    [src/utils/settings/settings.ts]
    → 读取现有设置
    → 使用 mergeWith 深度合并
    → 验证 SettingsSchema
    → 写入 ~/.claude/settings.json
```

### 钥匙串存储流程
```
xaaIdpCommand.ts
  → saveIdpClientSecret(issuer, secret)
    [src/services/mcp/xaaIdpLogin.ts]
    → getSecureStorage()
      [src/utils/secureStorage/index.ts]
      → 平台特定实现（macOS Keychain / Windows Credential / Linux Secret Service）
```

## 依赖与外部交互

### 外部依赖
- `@commander-js/extra-typings`: Commander.js 类型安全命令行解析

### 内部服务交互
1. **Settings**: 读写用户级设置（`userSettings`）
2. **Secure Storage**: 安全存储 id_token 和 client_secret
3. **Analytics**: 通过 `xaaIdpLogin.ts` 中的 `logEvent` 上报事件
4. **OIDC/OAuth**: 通过 MCP SDK 执行标准 OIDC 流程

### 环境变量
- `CLAUDE_CODE_ENABLE_XAA`: 启用 XAA 功能（在 `addCommand.ts` 中检查）
- `MCP_XAA_IDP_CLIENT_SECRET`: 传递 IdP client_secret

## 风险、边界与改进建议

### 风险点

1. **配置写入与钥匙串操作的原子性**
   - 代码通过"先验证、再写配置、最后清钥匙串"的顺序降低风险
   - 但仍存在配置写入成功但钥匙串写入失败的可能
   - **缓解**: `saveIdpClientSecret` 返回结果，调用方可以检测失败并提示重试

2. **Settings Schema 验证失败**
   - 注释提到：非 URL 格式的 issuer 会毒害整个设置源
   - `updateSettingsForSource` 在写入时不做 schema 检查
   - **缓解**: 在写入前进行严格的 URL 和端口验证

3. **JWT 直接注入的安全风险**
   - `--id-token` 参数可能在 shell 历史中暴露
   - 注释提到 TODO: 改为从 stdin 读取
   - **建议**: 尽快实现 stdin 读取，避免命令行参数暴露敏感信息

4. **Issuer Key 规范化边缘情况**
   - `issuerKey` 函数在 URL 解析失败时回退到简单字符串处理
   - 可能导致不同格式的同一 issuer 产生不同的键
   - **建议**: 添加警告日志，提示 URL 解析失败

### 边界情况

1. **HTTP 协议限制**
   - 仅允许 `localhost`、`127.0.0.1`、`[::1]` 使用 HTTP
   - 其他主机必须使用 HTTPS
   - 这是为了防止 client_secret 和授权码在明文传输中泄露

2. **Callback Port 处理**
   - 未指定时使用随机端口（RFC 8252 推荐）
   - 指定时验证为正整数
   - 某些 IdP 不支持端口通配，需要预注册固定端口

3. **Client Secret 可选性**
   - 支持 PKCE-only 的公共客户端
   - 也支持带 client_secret 的机密客户端
   - 由 IdP 元数据决定实际使用的认证方法

4. **配置变更的级联清理**
   - Issuer 改变：清理旧 issuer 的所有令牌和密钥
   - Client ID 改变：清理同一 issuer 下的令牌和密钥（因为 aud 声明不匹配）
   - 仅 callback port 改变：保留令牌和密钥

### 改进建议

1. **实现 stdin 读取 JWT**
   ```typescript
   .option('--stdin', 'Read id_token from stdin')
   .action(async options => {
     if (options.stdin) {
       const jwt = await readStdin()
       // ...
     }
   })
   ```

2. **添加配置导入/导出**
   ```typescript
   xaaIdp.command('export').action(() => {
     // 导出配置（不含密钥）用于备份或共享
   })
   xaaIdp.command('import').action(() => {
     // 导入配置
   })
   ```

3. **增强诊断信息**
   ```typescript
   xaaIdp.command('doctor').action(async () => {
     // 检查 IdP 可访问性
     // 验证 OIDC 发现端点
     // 测试令牌交换
   })
   ```

4. **支持多个 IdP 配置**
   - 当前设计是单 IdP
   - 企业用户可能需要为不同服务器使用不同 IdP
   - 考虑将 `xaaIdp` 改为数组或支持命名配置

5. **改进错误消息**
   - 当前某些错误消息较通用
   - 添加更具体的故障排除建议
   - 例如：IdP 发现失败时，提示检查网络和 URL

6. **令牌刷新提醒**
   ```typescript
   xaaIdp.command('status').action(() => {
     // 显示所有缓存令牌及其过期时间
     // 提醒即将过期的令牌
   })
   ```
