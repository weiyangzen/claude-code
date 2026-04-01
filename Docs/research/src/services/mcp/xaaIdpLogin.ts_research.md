# xaaIdpLogin.ts 深度研究文档

## 场景与职责

`xaaIdpLogin.ts` 实现 XAA（Cross-App Access）IdP 登录流程，负责：

1. **OIDC 登录流程**：通过标准 `authorization_code + PKCE` 流程从企业 IdP 获取 `id_token`
2. **Token 缓存管理**：将 `id_token` 安全存储在钥匙串中，按 IdP issuer 缓存
3. **IdP 客户端密钥管理**：存储和读取 IdP 的 `client_secret`
4. **OIDC 发现**：发现 IdP 的元数据端点

这是 XAA 流程的"一次浏览器登录"部分：用户只需在 IdP 登录一次，后续连接多个 MCP 服务器时可以复用缓存的 `id_token` 进行静默认证。

## 功能点目的

### 1. OIDC 登录流程

**目的**：获取用户的 `id_token`，用于后续的 Token Exchange

**流程**：
1. 检查缓存中是否有有效的 `id_token`
2. 如果没有，启动本地回调服务器
3. 打开浏览器访问 IdP 授权端点
4. 用户登录并授权
5. 接收授权码，交换 `id_token`
6. 缓存 `id_token` 及其过期时间

**安全特性**：
- 使用 PKCE（Proof Key for Code Exchange）防止授权码拦截攻击
- 生成随机 `state` 防止 CSRF 攻击
- 使用 XSS 过滤渲染错误页面

### 2. Token 缓存管理

**目的**：避免每次连接 MCP 服务器都弹出浏览器

**缓存键**：归一化的 IdP issuer URL（去除尾部斜杠，小写主机名）

**过期策略**：
- 使用 JWT 的 `exp` claim 作为过期时间
- 预留 60 秒缓冲期（`ID_TOKEN_EXPIRY_BUFFER_S`）
- 如果 `exp` 缺失，使用 `expires_in` 回退到 1 小时

### 3. IdP 客户端密钥管理

**目的**：支持需要 `client_secret` 的机密客户端

**存储位置**：钥匙串中的 `mcpXaaIdpConfig` 命名空间

**注意**：IdP `client_secret` 与 MCP 服务器的 AS `client_secret` 是分开存储的，因为它们属于不同的信任域。

### 4. OIDC 发现

**目的**：自动发现 IdP 的配置端点

**端点构造**：
- 标准：`{issuer}/.well-known/openid-configuration`
- 正确处理带路径的 issuer（如 Azure AD 的 `login.microsoftonline.com/{tenant}/v2.0`）

**安全验证**：
- 验证 `token_endpoint` 使用 HTTPS

## 具体技术实现

### 关键数据结构

```typescript
// IdP 设置
export type XaaIdpSettings = {
  issuer: string
  clientId: string
  callbackPort?: number
}

// 登录选项
export type IdpLoginOptions = {
  idpIssuer: string
  idpClientId: string
  idpClientSecret?: string
  callbackPort?: number
  onAuthorizationUrl?: (url: string) => void
  skipBrowserOpen?: boolean
  abortSignal?: AbortSignal
}

// Token 缓存条目
type IdTokenEntry = {
  idToken: string
  expiresAt: number
}
```

### 关键常量

```typescript
const IDP_LOGIN_TIMEOUT_MS = 5 * 60 * 1000      // 5 分钟登录超时
const IDP_REQUEST_TIMEOUT_MS = 30000            // 30 秒请求超时
const ID_TOKEN_EXPIRY_BUFFER_S = 60             // 60 秒过期缓冲
```

### 关键流程

#### issuerKey 实现

```typescript
export function issuerKey(issuer: string): string {
  try {
    const u = new URL(issuer)
    u.pathname = u.pathname.replace(/\/+$/, '')  // 去除尾部斜杠
    u.host = u.host.toLowerCase()                // 小写主机名
    return u.toString()
  } catch {
    return issuer.replace(/\/+$/, '')
  }
}
```

#### getCachedIdpIdToken 实现

```typescript
export function getCachedIdpIdToken(idpIssuer: string): string | undefined {
  const storage = getSecureStorage()
  const data = storage.read()
  const entry = data?.mcpXaaIdp?.[issuerKey(idpIssuer)]
  if (!entry) return undefined
  
  // 检查是否在缓冲期内过期
  const remainingMs = entry.expiresAt - Date.now()
  if (remainingMs <= ID_TOKEN_EXPIRY_BUFFER_S * 1000) return undefined
  
  return entry.idToken
}
```

#### discoverOidc 实现

```typescript
export async function discoverOidc(idpIssuer: string): Promise<OpenIdProviderDiscoveryMetadata> {
  // 正确处理带路径的 issuer
  const base = idpIssuer.endsWith('/') ? idpIssuer : idpIssuer + '/'
  const url = new URL('.well-known/openid-configuration', base)
  
  const res = await fetch(url, {
    headers: { Accept: 'application/json' },
    signal: AbortSignal.timeout(IDP_REQUEST_TIMEOUT_MS),
  })
  
  if (!res.ok) {
    throw new Error(`XAA IdP: OIDC discovery failed: HTTP ${res.status} at ${url}`)
  }
  
  // 处理 captive portal 返回 HTML 的情况
  let body: unknown
  try {
    body = await res.json()
  } catch {
    throw new Error(`XAA IdP: OIDC discovery returned non-JSON at ${url} (captive portal or proxy?)`)
  }
  
  const parsed = OpenIdProviderDiscoveryMetadataSchema.safeParse(body)
  if (!parsed.success) {
    throw new Error(`XAA IdP: invalid OIDC metadata: ${parsed.error.message}`)
  }
  
  // 强制 HTTPS
  if (new URL(parsed.data.token_endpoint).protocol !== 'https:') {
    throw new Error(`XAA IdP: refusing non-HTTPS token endpoint: ${parsed.data.token_endpoint}`)
  }
  
  return parsed.data
}
```

#### waitForCallback 实现

```typescript
function waitForCallback(
  port: number,
  expectedState: string,
  abortSignal: AbortSignal | undefined,
  onListening: () => void,
): Promise<string> {
  return new Promise<string>((resolve, reject) => {
    const server = createServer((req, res) => {
      const parsed = parse(req.url || '', true)
      if (parsed.pathname !== '/callback') {
        res.writeHead(404)
        res.end()
        return
      }
      
      const code = parsed.query.code as string | undefined
      const state = parsed.query.state as string | undefined
      const err = parsed.query.error as string | undefined
      
      // 处理错误
      if (err) {
        const desc = parsed.query.error_description as string | undefined
        const safeErr = xss(err)
        const safeDesc = desc ? xss(desc) : ''
        res.writeHead(400, { 'Content-Type': 'text/html' })
        res.end(`<html><body><h3>IdP login failed</h3><p>${safeErr}</p><p>${safeDesc}</p></body></html>`)
        reject(new Error(`XAA IdP: ${err}${desc ? ` — ${desc}` : ''}`))
        return
      }
      
      // 验证 state（CSRF 防护）
      if (state !== expectedState) {
        res.writeHead(400, { 'Content-Type': 'text/html' })
        res.end('<html><body><h3>State mismatch</h3></body></html>')
        reject(new Error('XAA IdP: state mismatch (possible CSRF)'))
        return
      }
      
      // 验证 code
      if (!code) {
        res.writeHead(400, { 'Content-Type': 'text/html' })
        res.end('<html><body><h3>Missing code</h3></body></html>')
        reject(new Error('XAA IdP: callback missing code'))
        return
      }
      
      // 成功响应
      res.writeHead(200, { 'Content-Type': 'text/html' })
      res.end('<html><body><h3>IdP login complete — you can close this window.</h3></body></html>')
      resolve(code)
    })
    
    server.listen(port, '127.0.0.1', () => {
      onListening()  // 通知调用方服务器已就绪
    })
    server.unref()
    
    // 超时处理
    const timeoutId = setTimeout(() => {
      reject(new Error('XAA IdP: login timed out'))
    }, IDP_LOGIN_TIMEOUT_MS)
    timeoutId.unref()
  })
}
```

#### acquireIdpIdToken 实现

```typescript
export async function acquireIdpIdToken(opts: IdpLoginOptions): Promise<string> {
  const { idpIssuer, idpClientId } = opts

  // 1. 检查缓存
  const cached = getCachedIdpIdToken(idpIssuer)
  if (cached) {
    logMCPDebug('xaa', `Using cached id_token for ${idpIssuer}`)
    return cached
  }

  // 2. OIDC 发现
  const metadata = await discoverOidc(idpIssuer)
  
  // 3. 准备回调服务器
  const port = opts.callbackPort ?? (await findAvailablePort())
  const redirectUri = buildRedirectUri(port)
  const state = randomBytes(32).toString('base64url')
  
  // 4. 启动授权流程
  const { authorizationUrl, codeVerifier } = await startAuthorization(idpIssuer, {
    metadata,
    clientInformation: { client_id: idpClientId, ...(opts.idpClientSecret ? { client_secret: opts.idpClientSecret } : {}) },
    redirectUrl: redirectUri,
    scope: 'openid',
    state,
  })

  // 5. 等待回调
  const authorizationCode = await waitForCallback(port, state, opts.abortSignal, () => {
    if (opts.onAuthorizationUrl) opts.onAuthorizationUrl(authorizationUrl.toString())
    if (!opts.skipBrowserOpen) {
      logMCPDebug('xaa', `Opening browser to IdP authorization endpoint`)
      void openBrowser(authorizationUrl.toString())
    }
  })

  // 6. 交换 token
  const tokens = await exchangeAuthorization(idpIssuer, {
    metadata,
    clientInformation: { client_id: idpClientId },
    authorizationCode,
    codeVerifier,
    redirectUri,
    fetchFn: (url, init) => fetch(url, { ...init, signal: AbortSignal.timeout(IDP_REQUEST_TIMEOUT_MS) }),
  })
  
  if (!tokens.id_token) {
    throw new Error('XAA IdP: token response missing id_token (check scope=openid)')
  }

  // 7. 缓存 token
  const expFromJwt = jwtExp(tokens.id_token)
  const expiresAt = expFromJwt ? expFromJwt * 1000 : Date.now() + (tokens.expires_in ?? 3600) * 1000
  saveIdpIdToken(idpIssuer, tokens.id_token, expiresAt)

  return tokens.id_token
}
```

#### jwtExp 实现

```typescript
function jwtExp(jwt: string): number | undefined {
  const parts = jwt.split('.')
  if (parts.length !== 3) return undefined
  try {
    const payload = jsonParse(Buffer.from(parts[1]!, 'base64url').toString('utf-8')) as { exp?: number }
    return typeof payload.exp === 'number' ? payload.exp : undefined
  } catch {
    return undefined
  }
}
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk/client/auth.js` | `startAuthorization`, `exchangeAuthorization` |
| `@modelcontextprotocol/sdk/shared/auth.js` | `OpenIdProviderDiscoveryMetadataSchema` |
| `src/utils/browser.ts` | `openBrowser` |
| `src/utils/envUtils.ts` | `isEnvTruthy` |
| `src/utils/errors.ts` | `toError` |
| `src/utils/log.ts` | `logMCPDebug` |
| `src/utils/platform.ts` | `getPlatform` |
| `src/utils/secureStorage/index.ts` | `getSecureStorage` |
| `src/utils/settings/settings.ts` | `getInitialSettings` |
| `src/utils/slowOperations.ts` | `jsonParse` |
| `src/services/mcp/oauthPort.ts` | `buildRedirectUri`, `findAvailablePort` |

### 调用方

| 文件 | 调用函数 |
|------|----------|
| `src/services/mcp/auth.ts` | `acquireIdpIdToken`, `getCachedIdpIdToken`, `clearIdpIdToken`, `discoverOidc`, `getIdpClientSecret`, `getXaaIdpSettings`, `isXaaEnabled` |

### 被调用方

| 函数 | 被调用文件 |
|------|------------|
| `acquireIdpIdToken` | `auth.ts`（`performMCPXaaAuth`） |
| `getCachedIdpIdToken` | `auth.ts`（`ClaudeAuthProvider.tokens`, `ClaudeAuthProvider.xaaRefresh`） |
| `saveIdpIdTokenFromJwt` | 调试/测试脚本 |

## 依赖与外部交互

### 外部系统交互

1. **MCP SDK**：使用 SDK 的 OAuth 功能
2. **IdP**：发送 OIDC 授权请求和 token 交换请求
3. **浏览器**：打开 IdP 授权页面
4. **钥匙串**：安全存储 `id_token` 和 `client_secret`

### 配置依赖

- `CLAUDE_CODE_ENABLE_XAA`：XAA 功能开关
- `settings.xaaIdp`：IdP 配置（issuer, clientId, callbackPort）

## 风险、边界与改进建议

### 风险点

1. **回调服务器安全**
   - 监听 `127.0.0.1`，仅接受本地连接
   - 验证 `state` 防止 CSRF
   - 使用 XSS 过滤渲染错误页面
   - 建议：添加更多安全头（CSP）

2. **Token 缓存安全**
   - `id_token` 存储在钥匙串中，依赖系统安全
   - JWT 签名不验证（由 IdP 在 Token Exchange 时验证）
   - 建议：文档中明确说明安全假设

3. **端口占用**
   - 如果指定了 `callbackPort` 但已被占用，会报错
   - 建议：自动端口扫描和回退

4. **超时处理**
   - 5 分钟登录超时可能过长或过短
   - 建议：可配置或根据用户行为动态调整

### 边界情况

1. **IdP 返回非标准响应**
   - `expires_in` 可能是字符串（PHP 后端常见）
   - 使用 `z.coerce.number()` 容忍

2. **JWT 解析失败**
   - `jwtExp` 函数捕获所有异常，返回 `undefined`
   - 回退到 `expires_in` 或默认 1 小时

3. **多 IdP 场景**
   - 缓存按键名是归一化的 issuer URL
   - 不同 issuer 的 token 分开存储

4. **并发登录**
   - 如果用户快速多次触发登录，会启动多个回调服务器
   - 建议：添加登录状态锁

### 改进建议

1. **添加 CSP 头**
   ```typescript
   res.writeHead(200, { 
     'Content-Type': 'text/html',
     'Content-Security-Policy': "default-src 'none'; style-src 'unsafe-inline'"
   })
   ```

2. **自动端口回退**
   ```typescript
   async function findAvailablePort(preferred?: number): Promise<number> {
     if (preferred) {
       try {
         return await tryPort(preferred)
       } catch {
         // 回退到随机端口
       }
     }
     return findRandomPort()
   }
   ```

3. **登录状态锁**
   ```typescript
   const loginInProgress = new Map<string, Promise<string>>()
   
   export async function acquireIdpIdToken(opts: IdpLoginOptions): Promise<string> {
     const key = issuerKey(opts.idpIssuer)
     if (loginInProgress.has(key)) {
       return loginInProgress.get(key)!
     }
     
     const promise = doAcquireIdpIdToken(opts).finally(() => {
       loginInProgress.delete(key)
     })
     loginInProgress.set(key, promise)
     return promise
   }
   ```

4. **支持刷新令牌**
   - 某些 IdP 可能返回 `refresh_token`
   - 可以缓存并用于刷新 `id_token`

5. **更好的错误分类**
   ```typescript
   export type IdpLoginErrorCode =
     | 'discovery_failed'
     | 'port_unavailable'
     | 'timeout'
     | 'cancelled'
     | 'token_exchange_failed'
   ```

6. **支持设备码流程**
   - 对于无浏览器环境，支持 Device Authorization Grant
   - 适用于 SSH 远程环境
