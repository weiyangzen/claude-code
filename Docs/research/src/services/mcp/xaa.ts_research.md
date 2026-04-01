# xaa.ts 深度研究文档

## 场景与职责

`xaa.ts` 实现 Cross-App Access (XAA) / Enterprise Managed Authorization (SEP-990) 协议，提供无需浏览器授权屏幕的 MCP 服务器认证流程。核心能力：

1. **RFC 8693 Token Exchange**：在 IdP 处将 `id_token` 交换为 ID-JAG（Identity Assertion Authorization Grant）
2. **RFC 7523 JWT Bearer Grant**：在 AS（Authorization Server）处将 ID-JAG 交换为 `access_token`
3. **RFC 9728 PRM Discovery**：发现受保护资源的授权服务器元数据
4. **RFC 8414 AS Discovery**：发现授权服务器元数据

XAA 的价值主张是"一次浏览器登录，多次静默认证"：用户只需在 IdP 登录一次，后续连接多个 MCP 服务器时无需再次弹出浏览器。

## 功能点目的

### 1. 受保护资源元数据发现（PRM Discovery）

**目的**：根据 RFC 9728 发现 MCP 服务器的授权服务器

**流程**：
1. 向 MCP 服务器 `/.well-known/oauth-protected-resource` 发送请求
2. 验证返回的 `resource` 与请求 URL 匹配（防混肴攻击）
3. 提取 `authorization_servers` 列表

### 2. 授权服务器元数据发现（AS Discovery）

**目的**：根据 RFC 8414 发现授权服务器的端点信息

**流程**：
1. 向授权服务器 `/.well-known/oauth-authorization-server` 发送请求
2. 验证返回的 `issuer` 与请求 URL 匹配（防混肴攻击）
3. 验证 `token_endpoint` 使用 HTTPS
4. 提取支持的授权类型和认证方法

### 3. Token Exchange（RFC 8693）

**目的**：在 IdP 处将用户的 `id_token` 交换为 ID-JAG

**请求参数**：
- `grant_type`: `urn:ietf:params:oauth:grant-type:token-exchange`
- `requested_token_type`: `urn:ietf:params:oauth:token-type:id-jag`
- `audience`: AS 的 issuer URL
- `resource`: MCP 服务器 URL
- `subject_token`: 用户的 `id_token`
- `subject_token_type`: `urn:ietf:params:oauth:token-type:id_token`

**响应验证**：
- 检查 `issued_token_type` 是否为 ID-JAG
- 提取 `access_token`（即 ID-JAG）

### 4. JWT Bearer Grant（RFC 7523）

**目的**：在 AS 处将 ID-JAG 交换为 MCP 服务器的 `access_token`

**请求参数**：
- `grant_type`: `urn:ietf:params:oauth:grant-type:jwt-bearer`
- `assertion`: ID-JAG

**认证方法**：
- `client_secret_basic`（默认）：Base64 编码的 `client_id:client_secret` 放在 Authorization 头
- `client_secret_post`：`client_id` 和 `client_secret` 放在请求体

### 5. 完整 XAA 流程编排

**目的**：将上述四个 Layer-2 操作组合成完整的认证流程

**流程**：
1. PRM Discovery → 获取 AS 列表
2. 遍历 AS 列表，找到支持 `jwt-bearer` 的 AS
3. AS Discovery → 获取 token endpoint 和认证方法
4. Token Exchange → 获取 ID-JAG
5. JWT Bearer Grant → 获取 `access_token`

## 具体技术实现

### 关键数据结构

```typescript
// XAA 配置
export type XaaConfig = {
  clientId: string                    // AS 的 client_id
  clientSecret: string                // AS 的 client_secret
  idpClientId: string                 // IdP 的 client_id
  idpClientSecret?: string            // IdP 的 client_secret（可选）
  idpIdToken: string                  // 用户的 id_token
  idpTokenEndpoint: string            // IdP 的 token endpoint
}

// Token Exchange 结果
export type JwtAuthGrantResult = {
  jwtAuthGrant: string                // ID-JAG
  expiresIn?: number
  scope?: string
}

// JWT Bearer 结果
export type XaaTokenResult = {
  access_token: string
  token_type: string
  expires_in?: number
  scope?: string
  refresh_token?: string
}

// 完整 XAA 结果
export type XaaResult = XaaTokenResult & {
  authorizationServerUrl: string      // 用于后续刷新和撤销
}

// PRM 元数据
export type ProtectedResourceMetadata = {
  resource: string
  authorization_servers: string[]
}

// AS 元数据
export type AuthorizationServerMetadata = {
  issuer: string
  token_endpoint: string
  grant_types_supported?: string[]
  token_endpoint_auth_methods_supported?: string[]
}
```

### 关键常量

```typescript
const XAA_REQUEST_TIMEOUT_MS = 30000
const TOKEN_EXCHANGE_GRANT = 'urn:ietf:params:oauth:grant-type:token-exchange'
const JWT_BEARER_GRANT = 'urn:ietf:params:oauth:grant-type:jwt-bearer'
const ID_JAG_TOKEN_TYPE = 'urn:ietf:params:oauth:token-type:id-jag'
const ID_TOKEN_TYPE = 'urn:ietf:params:oauth:token-type:id_token'
```

### 关键流程

#### makeXaaFetch 实现

```typescript
function makeXaaFetch(abortSignal?: AbortSignal): FetchLike {
  return (url, init) => {
    const timeout = AbortSignal.timeout(XAA_REQUEST_TIMEOUT_MS)
    const signal = abortSignal
      ? AbortSignal.any([timeout, abortSignal])
      : timeout
    return fetch(url, { ...init, signal })
  }
}
```

#### discoverProtectedResource 实现

```typescript
export async function discoverProtectedResource(
  serverUrl: string,
  opts?: { fetchFn?: FetchLike },
): Promise<ProtectedResourceMetadata> {
  // 使用 SDK 进行 PRM 发现
  const prm = await discoverOAuthProtectedResourceMetadata(
    serverUrl,
    undefined,
    opts?.fetchFn ?? defaultFetch,
  )
  
  // 验证必需字段
  if (!prm.resource || !prm.authorization_servers?.[0]) {
    throw new Error('XAA: PRM discovery failed: PRM missing resource or authorization_servers')
  }
  
  // RFC 9728 §3.3 资源匹配验证（防混肴）
  if (normalizeUrl(prm.resource) !== normalizeUrl(serverUrl)) {
    throw new Error(`XAA: PRM discovery failed: PRM resource mismatch`)
  }
  
  return {
    resource: prm.resource,
    authorization_servers: prm.authorization_servers,
  }
}
```

#### requestJwtAuthorizationGrant 实现

```typescript
export async function requestJwtAuthorizationGrant(opts: {
  tokenEndpoint: string
  audience: string
  resource: string
  idToken: string
  clientId: string
  clientSecret?: string
  scope?: string
  fetchFn?: FetchLike
}): Promise<JwtAuthGrantResult> {
  const params = new URLSearchParams({
    grant_type: TOKEN_EXCHANGE_GRANT,
    requested_token_type: ID_JAG_TOKEN_TYPE,
    audience: opts.audience,
    resource: opts.resource,
    subject_token: opts.idToken,
    subject_token_type: ID_TOKEN_TYPE,
    client_id: opts.clientId,
  })
  if (opts.clientSecret) params.set('client_secret', opts.clientSecret)
  if (opts.scope) params.set('scope', opts.scope)

  const res = await fetchFn(opts.tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: params,
  })
  
  if (!res.ok) {
    const body = redactTokens(await res.text()).slice(0, 200)
    const shouldClear = res.status < 500  // 4xx → 清除 id_token，5xx → 保留
    throw new XaaTokenExchangeError(`XAA: token exchange failed: HTTP ${res.status}: ${body}`, shouldClear)
  }
  
  const exchangeParsed = TokenExchangeResponseSchema().safeParse(await res.json())
  if (!exchangeParsed.success) {
    throw new XaaTokenExchangeError(`XAA: token exchange response did not match expected shape`, true)
  }
  
  const result = exchangeParsed.data
  if (result.issued_token_type !== ID_JAG_TOKEN_TYPE) {
    throw new XaaTokenExchangeError(`XAA: token exchange returned unexpected issued_token_type`, true)
  }
  
  return {
    jwtAuthGrant: result.access_token!,
    expiresIn: result.expires_in,
    scope: result.scope,
  }
}
```

#### exchangeJwtAuthGrant 实现

```typescript
export async function exchangeJwtAuthGrant(opts: {
  tokenEndpoint: string
  assertion: string
  clientId: string
  clientSecret: string
  authMethod?: 'client_secret_basic' | 'client_secret_post'
  scope?: string
  fetchFn?: FetchLike
}): Promise<XaaTokenResult> {
  const authMethod = opts.authMethod ?? 'client_secret_basic'
  
  const params = new URLSearchParams({
    grant_type: JWT_BEARER_GRANT,
    assertion: opts.assertion,
  })
  if (opts.scope) params.set('scope', opts.scope)

  const headers: Record<string, string> = {
    'Content-Type': 'application/x-www-form-urlencoded',
  }
  
  if (authMethod === 'client_secret_basic') {
    const basicAuth = Buffer.from(
      `${encodeURIComponent(opts.clientId)}:${encodeURIComponent(opts.clientSecret)}`,
    ).toString('base64')
    headers.Authorization = `Basic ${basicAuth}`
  } else {
    params.set('client_id', opts.clientId)
    params.set('client_secret', opts.clientSecret)
  }

  const res = await fetchFn(opts.tokenEndpoint, {
    method: 'POST',
    headers,
    body: params,
  })
  
  if (!res.ok) {
    const body = redactTokens(await res.text()).slice(0, 200)
    throw new Error(`XAA: jwt-bearer grant failed: HTTP ${res.status}: ${body}`)
  }
  
  const tokensParsed = JwtBearerResponseSchema().safeParse(await res.json())
  if (!tokensParsed.success) {
    throw new Error(`XAA: jwt-bearer response did not match expected shape`)
  }
  return tokensParsed.data
}
```

#### performCrossAppAccess 实现

```typescript
export async function performCrossAppAccess(
  serverUrl: string,
  config: XaaConfig,
  serverName = 'xaa',
  abortSignal?: AbortSignal,
): Promise<XaaResult> {
  const fetchFn = makeXaaFetch(abortSignal)

  // 1. PRM Discovery
  const prm = await discoverProtectedResource(serverUrl, { fetchFn })

  // 2. 遍历 AS 列表，找到支持 jwt-bearer 的 AS
  let asMeta: AuthorizationServerMetadata | undefined
  const asErrors: string[] = []
  for (const asUrl of prm.authorization_servers) {
    try {
      const candidate = await discoverAuthorizationServer(asUrl, { fetchFn })
      if (candidate.grant_types_supported &&
          !candidate.grant_types_supported.includes(JWT_BEARER_GRANT)) {
        asErrors.push(`${asUrl}: does not advertise jwt-bearer grant`)
        continue
      }
      asMeta = candidate
      break
    } catch (e) {
      asErrors.push(`${asUrl}: ${e instanceof Error ? e.message : String(e)}`)
    }
  }
  if (!asMeta) {
    throw new Error(`XAA: no authorization server supports jwt-bearer. Tried: ${asErrors.join('; ')}`)
  }

  // 3. 选择认证方法
  const authMethods = asMeta.token_endpoint_auth_methods_supported
  const authMethod: 'client_secret_basic' | 'client_secret_post' =
    authMethods && !authMethods.includes('client_secret_basic') && authMethods.includes('client_secret_post')
      ? 'client_secret_post'
      : 'client_secret_basic'

  // 4. Token Exchange
  const jag = await requestJwtAuthorizationGrant({
    tokenEndpoint: config.idpTokenEndpoint,
    audience: asMeta.issuer,
    resource: prm.resource,
    idToken: config.idpIdToken,
    clientId: config.idpClientId,
    clientSecret: config.idpClientSecret,
    fetchFn,
  })

  // 5. JWT Bearer Grant
  const tokens = await exchangeJwtAuthGrant({
    tokenEndpoint: asMeta.token_endpoint,
    assertion: jag.jwtAuthGrant,
    clientId: config.clientId,
    clientSecret: config.clientSecret,
    authMethod,
    fetchFn,
  })

  return { ...tokens, authorizationServerUrl: asMeta.issuer }
}
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk/client/auth.js` | `discoverAuthorizationServerMetadata`, `discoverOAuthProtectedResourceMetadata` |
| `@modelcontextprotocol/sdk/shared/transport.js` | `FetchLike` 类型 |
| `src/utils/lazySchema.ts` | `lazySchema` |
| `src/utils/log.ts` | `logMCPDebug` |
| `src/utils/slowOperations.ts` | `jsonStringify` |

### 调用方

| 文件 | 调用函数 |
|------|----------|
| `src/services/mcp/auth.ts` | `performCrossAppAccess`, `XaaTokenExchangeError` |
| `src/services/mcp/xaaIdpLogin.ts` | `discoverOidc`（类似模式） |

### 被调用方

| 函数 | 被调用文件 |
|------|------------|
| `performCrossAppAccess` | `auth.ts`（`performMCPXaaAuth`, `ClaudeAuthProvider.xaaRefresh`） |

## 依赖与外部交互

### 外部系统交互

1. **MCP SDK**：使用 SDK 的发现功能
2. **IdP**：发送 RFC 8693 Token Exchange 请求
3. **AS**：发送 RFC 7523 JWT Bearer Grant 请求
4. **MCP 服务器**：获取 PRM 元数据

### 配置依赖

- `CLAUDE_CODE_ENABLE_XAA`：XAA 功能开关（在 `xaaIdpLogin.ts` 中检查）

## 风险、边界与改进建议

### 风险点

1. **Token 泄露风险**
   - 错误响应中可能包含敏感 token
   - 使用 `redactTokens` 函数进行脱敏处理
   - 建议：定期审计脱敏规则是否完整

2. **AS 选择逻辑**
   - 仅检查 `grant_types_supported` 是否包含 `jwt-bearer`
   - 如果 AS 广告支持但实际上不支持，会在后续步骤失败
   - 建议：添加 AS 健康检查或优先级排序

3. **超时处理**
   - 30 秒超时可能过长或过长
   - 建议：根据网络环境动态调整

4. **认证方法选择**
   - 默认使用 `client_secret_basic`，但某些 AS 可能只支持 `client_secret_post`
   - 当前逻辑：如果 AS 明确不支持 basic 且支持 post，则使用 post
   - 建议：添加配置覆盖选项

### 边界情况

1. **多个 AS 的情况**
   - 遍历 `authorization_servers` 列表，使用第一个支持的 AS
   - 如果多个 AS 都支持，没有优先级排序

2. **Token Exchange 失败**
   - `XaaTokenExchangeError` 携带 `shouldClearIdToken` 标志
   - 4xx 错误清除 id_token（token 无效）
   - 5xx 错误保留 id_token（IdP 故障）

3. **空响应处理**
   - `expires_in` 使用 `z.coerce.number()` 容忍字符串类型
   - `token_type` 默认为 'Bearer'

### 改进建议

1. **添加重试机制**
   ```typescript
   async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
     for (let i = 0; i < maxRetries; i++) {
       try {
         return await fn()
       } catch (e) {
         if (i === maxRetries - 1) throw e
         if (e instanceof XaaTokenExchangeError && e.shouldClearIdToken) throw e
         await sleep(1000 * Math.pow(2, i))
       }
     }
     throw new Error('Unreachable')
   }
   ```

2. **AS 优先级排序**
   ```typescript
   // 根据响应时间或历史成功率排序 AS
   const sortedAsUrls = await sortByHealth(prm.authorization_servers)
   ```

3. **更详细的错误分类**
   ```typescript
   export type XaaErrorCode = 
     | 'prm_discovery_failed'
     | 'as_discovery_failed'
     | 'token_exchange_4xx'
     | 'token_exchange_5xx'
     | 'jwt_bearer_failed'
   ```

4. **缓存发现结果**
   - PRM 和 AS 元数据可以缓存（TTL 建议 1 小时）
   - 减少重复请求，提高性能

5. **支持更多认证方法**
   - `private_key_jwt`：使用私钥签名 JWT
   - `tls_client_auth`：使用 mTLS
