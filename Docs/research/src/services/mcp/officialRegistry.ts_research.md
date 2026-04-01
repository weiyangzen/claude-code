# MCP 官方注册表 (officialRegistry.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`officialRegistry.ts` 是 Claude Code MCP 系统的**官方注册表客户端**，负责：
- **预取官方 URL**：从 Anthropic MCP 注册表获取官方服务器 URL 列表
- **URL 验证**：判断给定 URL 是否为官方注册的 MCP 服务器
- **失败静默**：网络失败时不阻断功能，降级到空列表
- **非关键流量控制**：支持禁用非必要网络请求

### 1.2 业务场景
| 场景 | 说明 |
|------|------|
| 信任指示 | 在 UI 中标识官方 MCP 服务器 |
| 安全提示 | 对非官方服务器显示额外安全警告 |
| 分析统计 | 区分官方与非官方服务器使用 |
| 企业环境 | 离线环境可禁用注册表查询 |

### 1.3 设计原则
- **fire-and-forget**：预取是异步的，不阻塞启动
- **fail-closed**：获取失败时返回 false（保守策略）
- **可禁用**：通过环境变量禁用非必要网络请求
- **内存缓存**：获取结果缓存在内存中

---

## 2. 功能点目的

### 2.1 官方 URL 预取
**目的**：在应用启动时预加载官方 MCP 服务器 URL 列表。

**注册表 API**：
- URL：`https://api.anthropic.com/mcp-registry/v0/servers`
- 参数：`version=latest&visibility=commercial`
- 超时：5 秒

### 2.2 URL 验证
**目的**：判断用户配置的 MCP 服务器 URL 是否为官方注册。

**使用场景**：
- UI 中显示官方徽章
- 安全提示差异化
- 分析事件标记

### 2.3 离线支持
**目的**：支持离线或受限环境使用。

**禁用方式**：
```bash
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 注册表服务器条目
type RegistryServer = {
  server: {
    remotes?: Array<{ url: string }>
  }
}

// 注册表响应
type RegistryResponse = {
  servers: RegistryServer[]
}

// 内存缓存
let officialUrls: Set<string> | undefined = undefined
```

### 3.2 URL 规范化

```typescript
function normalizeUrl(url: string): string | undefined {
  try {
    const u = new URL(url)
    u.search = ''                    // 移除查询字符串
    return u.toString().replace(/\/$/, '')  // 移除尾部斜杠
  } catch {
    return undefined
  }
}
```

**规范化规则**：
- 移除查询字符串（可能包含敏感 token）
- 移除尾部斜杠（统一格式）
- 异常时返回 undefined

### 3.3 官方 URL 预取

```typescript
export async function prefetchOfficialMcpUrls(): Promise<void> {
  // 1. 检查是否禁用非必要流量
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) {
    return
  }

  try {
    const response = await axios.get<RegistryResponse>(
      'https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial',
      { timeout: 5000 }
    )

    const urls = new Set<string>()
    for (const entry of response.data.servers) {
      for (const remote of entry.server.remotes ?? []) {
        const normalized = normalizeUrl(remote.url)
        if (normalized) {
          urls.add(normalized)
        }
      }
    }
    officialUrls = urls
    logForDebugging(`[mcp-registry] Loaded ${urls.size} official MCP URLs`)
  } catch (error) {
    logForDebugging(`Failed to fetch MCP registry: ${errorMessage(error)}`, {
      level: 'error'
    })
  }
}
```

### 3.4 URL 验证

```typescript
export function isOfficialMcpUrl(normalizedUrl: string): boolean {
  return officialUrls?.has(normalizedUrl) ?? false
}
```

**注意**：
- 输入 URL 需要先通过 `getLoggingSafeMcpBaseUrl` 规范化
- 未获取注册表时返回 false（fail-closed）

### 3.5 测试重置

```typescript
export function resetOfficialMcpUrlsForTesting(): void {
  officialUrls = undefined
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `prefetchOfficialMcpUrls` | 33-59 | 预取官方 MCP URL 列表 |
| `isOfficialMcpUrl` | 66-68 | 检查 URL 是否为官方 |
| `resetOfficialMcpUrlsForTesting` | 70-72 | 测试重置函数 |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `axios` | HTTP 客户端 |
| `../../utils/debug.js` | logForDebugging |
| `../../utils/errors.js` | errorMessage |

### 4.3 调用方

| 文件 | 用途 |
|------|------|
| `client.ts` | 连接服务器时检查是否官方 |
| `../../main.tsx` | 启动时预取注册表 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
import axios from 'axios'
import { logForDebugging } from '../../utils/debug.js'
import { errorMessage } from '../../utils/errors.js'
```

### 5.2 外部 API

```
GET https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial

Response:
{
  "servers": [
    {
      "server": {
        "remotes": [
          { "url": "https://server1.example.com" },
          { "url": "https://server2.example.com" }
        ]
      }
    }
  ]
}
```

### 5.3 环境变量

```typescript
// 禁用非必要网络请求
process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC
```

### 5.4 与 utils.ts 的协作

```typescript
// utils.ts
export function getLoggingSafeMcpBaseUrl(config: McpServerConfig): string | undefined {
  if (!('url' in config)) return undefined
  try {
    const url = new URL(config.url)
    url.search = ''
    return url.toString().replace(/\/$/, '')
  } catch {
    return undefined
  }
}

// 使用示例
const baseUrl = getLoggingSafeMcpBaseUrl(config)
if (baseUrl && isOfficialMcpUrl(baseUrl)) {
  // 标记为官方服务器
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 安全风险
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| DNS 劫持 | 注册表 API 被劫持返回恶意 URL | HTTPS + 证书验证 |
| 缓存投毒 | 内存中的 officialUrls 被篡改 | 仅内部使用，无外部输入 |
| 信息泄露 | 注册表请求暴露用户行为 | 无用户特定信息 |

#### 6.1.2 功能边界
| 边界 | 说明 |
|------|------|
| 内存缓存 | 应用重启后需重新获取 |
| 无持久化 | 不缓存到磁盘 |
| 单次获取 | 无自动刷新机制 |
| 仅商业版 | visibility=commercial 过滤 |

### 6.2 潜在问题

#### 6.2.1 启动时网络请求
```typescript
// 问题：启动时增加网络请求，可能延迟启动
// 缓解：fire-and-forget，不阻塞启动流程
```

#### 6.2.2 缓存过期
```typescript
// 问题：长时间运行的进程可能使用过期的注册表数据
// 当前：无刷新机制
// 缓解：数据相对稳定，服务器增减不频繁
```

#### 6.2.3 错误静默
```typescript
// 问题：获取失败仅记录 debug 日志，用户无感知
catch (error) {
  logForDebugging(`Failed to fetch MCP registry: ${errorMessage(error)}`, {
    level: 'error'
  })
}
```

### 6.3 改进建议

#### 6.3.1 功能增强
1. **定期刷新**：
   ```typescript
   const REGISTRY_REFRESH_INTERVAL = 24 * 60 * 60 * 1000  // 24 小时
   
   export async function prefetchOfficialMcpUrls(): Promise<void> {
     // 检查是否需要刷新
     if (officialUrls && Date.now() - lastFetchTime < REGISTRY_REFRESH_INTERVAL) {
       return
     }
     // ... 获取逻辑
   }
   ```

2. **持久化缓存**：
   ```typescript
   // 缓存到磁盘，启动时优先读取
   const CACHE_FILE = join(getClaudeConfigHomeDir(), 'mcp-registry-cache.json')
   
   async function loadCachedRegistry(): Promise<Set<string> | undefined> {
     try {
       const data = await readFile(CACHE_FILE, 'utf-8')
       const { urls, timestamp } = JSON.parse(data)
       if (Date.now() - timestamp < CACHE_TTL) {
         return new Set(urls)
       }
     } catch {
       return undefined
     }
   }
   ```

3. **增量更新**：
   ```typescript
   // 支持 ETag 或 Last-Modified 实现增量更新
   const response = await axios.get(REGISTRY_URL, {
     headers: { 'If-None-Match': cachedEtag },
     timeout: 5000
   })
   if (response.status === 304) {
     // 数据未变更，使用缓存
     return
   }
   ```

#### 6.3.2 可靠性增强
1. **重试机制**：
   ```typescript
   import axiosRetry from 'axios-retry'
   
   axiosRetry(axios, {
     retries: 3,
     retryDelay: axiosRetry.exponentialDelay,
     retryCondition: (error) => {
       return axiosRetry.isNetworkOrIdempotentRequestError(error)
     }
   })
   ```

2. **离线模式检测**：
   ```typescript
   function isOffline(): boolean {
     return !navigator.onLine  // 浏览器环境
     // 或检查网络接口状态
   }
   
   if (isOffline()) {
     logForDebugging('Offline mode, skipping MCP registry fetch')
     return
   }
   ```

3. **降级策略**：
   ```typescriptn   // 硬编码主要官方 URL 作为降级
   const FALLBACK_OFFICIAL_URLS = [
     'https://slack.mcp.anthropic.com',
     'https://github.mcp.anthropic.com'
   ]
   ```

#### 6.3.3 可观测性
1. **详细分析**：
   ```typescript
   logEvent('tengu_mcp_registry_fetch', {
     success: true,
     urlCount: urls.size,
     durationMs: Date.now() - startTime
   })
   ```

2. **缓存命中率**：
   ```typescript
   let cacheHits = 0
   let cacheMisses = 0
   
   export function isOfficialMcpUrl(normalizedUrl: string): boolean {
     const result = officialUrls?.has(normalizedUrl) ?? false
     if (officialUrls) {
       result ? cacheHits++ : cacheMisses++
     }
     return result
   }
   ```

#### 6.3.4 代码质量
1. **类型安全**：
   ```typescript
   // 使用 branded type
   type NormalizedUrl = string & { __brand: 'NormalizedUrl' }
   
   export function isOfficialMcpUrl(url: NormalizedUrl): boolean {
     return officialUrls?.has(url) ?? false
   }
   ```

2. **单元测试**：
   ```typescript
   describe('officialRegistry', () => {
     beforeEach(() => {
       resetOfficialMcpUrlsForTesting()
     })
     
     it('returns false when registry not fetched', () => {
       expect(isOfficialMcpUrl('https://example.com')).toBe(false)
     })
     
     it('returns true for official URL', async () => {
       // mock axios response
       await prefetchOfficialMcpUrls()
       expect(isOfficialMcpUrl('https://official.example.com')).toBe(true)
     })
     
     it('respects DISABLE_NONESSENTIAL_TRAFFIC', async () => {
       process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC = '1'
       await prefetchOfficialMcpUrls()
       // 验证未发起请求
       delete process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC
     })
   })
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| 正常获取和验证 | 高 |
| 网络失败处理 | 高 |
| 禁用标志检查 | 高 |
| URL 规范化 | 中 |
| 缓存行为 | 中 |
| 并发获取 | 低 |
