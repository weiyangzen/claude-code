# MCP OAuth 端口管理 (oauthPort.ts) 深度研究

## 1. 场景与职责

### 1.1 核心定位
`oauthPort.ts` 是 Claude Code MCP 系统的 **OAuth 回调端口管理工具**，负责：
- **端口查找**：在动态端口范围内查找可用端口
- **回退处理**：动态端口用尽时使用固定回退端口
- **URI 构建**：生成符合 RFC 8252 的本地回调 URI
- **平台适配**：Windows 使用不同的端口范围避免系统保留端口

### 1.2 业务场景
| 场景 | 说明 |
|------|------|
| OAuth 授权流程 | MCP 服务器 OAuth 认证需要本地回调端口 |
| 动态端口分配 | 避免固定端口冲突，提高可靠性 |
| 端口冲突处理 | 随机选择 + 回退机制确保可用性 |
| 企业环境 | 支持通过环境变量配置固定端口 |

### 1.3 设计原则
- **RFC 8252 兼容**：遵循 OAuth 2.0 for Native Apps 规范
- **平台适配**：Windows 使用非保留端口范围
- **随机选择**：提高安全性，避免可预测性
- **优雅降级**：动态端口用尽时使用固定回退端口

---

## 2. 功能点目的

### 2.1 动态端口分配
**目的**：在 IANA 建议的动态端口范围内查找可用端口。

**端口范围**：
- 非 Windows：`49152-65535`（IANA 动态/私有端口）
- Windows：`39152-49151`（避免系统保留端口）

### 2.2 回退机制
**目的**：当动态端口用尽时，使用固定回退端口确保功能可用。

**回退端口**：`3118`

### 2.3 回调 URI 构建
**目的**：生成符合 OAuth 2.0 规范的本地回调 URI。

**格式**：`http://localhost:{port}/callback`

**RFC 8252 说明**：
> loopback redirect URIs match any port as long as the path matches.

### 2.4 可配置端口
**目的**：支持企业环境通过环境变量配置固定端口。

**环境变量**：`MCP_OAUTH_CALLBACK_PORT`

---

## 3. 具体技术实现

### 3.1 核心常量

```typescript
// Windows dynamic port range 49152-65535 is reserved
const REDIRECT_PORT_RANGE =
  getPlatform() === 'windows'
    ? { min: 39152, max: 49151 }
    : { min: 49152, max: 65535 }

const REDIRECT_PORT_FALLBACK = 3118
```

### 3.2 端口配置获取

```typescript
function getMcpOAuthCallbackPort(): number | undefined {
  const port = parseInt(process.env.MCP_OAUTH_CALLBACK_PORT || '', 10)
  return port > 0 ? port : undefined
}
```

### 3.3 回调 URI 构建

```typescript
/**
 * Builds a redirect URI on localhost with the given port and a fixed `/callback` path.
 *
 * RFC 8252 Section 7.3 (OAuth for Native Apps): loopback redirect URIs match any
 * port as long as the path matches.
 */
export function buildRedirectUri(
  port: number = REDIRECT_PORT_FALLBACK
): string {
  return `http://localhost:${port}/callback`
}
```

### 3.4 可用端口查找

```typescript
/**
 * Finds an available port in the specified range for OAuth redirect
 * Uses random selection for better security
 */
export async function findAvailablePort(): Promise<number> {
  // 1. 首先尝试配置的固定端口
  const configuredPort = getMcpOAuthCallbackPort()
  if (configuredPort) {
    return configuredPort
  }

  const { min, max } = REDIRECT_PORT_RANGE
  const range = max - min + 1
  const maxAttempts = Math.min(range, 100) // 最多尝试 100 次

  // 2. 随机选择端口尝试
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const port = min + Math.floor(Math.random() * range)

    try {
      await new Promise<void>((resolve, reject) => {
        const testServer = createServer()
        testServer.once('error', reject)
        testServer.listen(port, () => {
          testServer.close(() => resolve())
        })
      })
      return port
    } catch {
      // Port in use, try another random port
      continue
    }
  }

  // 3. 随机选择失败，尝试回退端口
  try {
    await new Promise<void>((resolve, reject) => {
      const testServer = createServer()
      testServer.once('error', reject)
      testServer.listen(REDIRECT_PORT_FALLBACK, () => {
        testServer.close(() => resolve())
      })
    })
    return REDIRECT_PORT_FALLBACK
  } catch {
    throw new Error(`No available ports for OAuth redirect`)
  }
}
```

### 3.5 实现细节分析

#### 3.5.1 端口测试机制
```typescript
await new Promise<void>((resolve, reject) => {
  const testServer = createServer()
  testServer.once('error', reject)  // 端口占用时触发 error
  testServer.listen(port, () => {
    testServer.close(() => resolve())  // 成功监听后关闭
  })
})
```

**流程**：
1. 创建临时 HTTP 服务器
2. 尝试监听指定端口
3. 成功：关闭服务器并返回端口
4. 失败（EADDRINUSE）：reject，继续尝试下一个

#### 3.5.2 随机选择策略
```typescript
const port = min + Math.floor(Math.random() * range)
```

**安全考虑**：
- 避免顺序扫描的可预测性
- 分散端口使用，减少冲突概率
- 符合 OAuth 安全最佳实践

#### 3.5.3 尝试次数限制
```typescript
const maxAttempts = Math.min(range, 100)
```

**平衡**：
- 范围大小：16484 个端口（非 Windows）
- 最多尝试 100 次
- 在性能和成功率之间取得平衡

---

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `buildRedirectUri` | 21-25 | 构建 OAuth 回调 URI |
| `findAvailablePort` | 36-78 | 查找可用端口 |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `http` | Node.js HTTP 模块，createServer |
| `../../utils/platform.js` | getPlatform 平台检测 |

### 4.3 调用方

| 文件 | 用途 |
|------|------|
| `auth.ts` | OAuth 流程中获取回调端口 |
| `xaaIdpLogin.ts` | XAA IdP 登录流程 |

---

## 5. 依赖与外部交互

### 5.1 外部依赖

```typescript
import { createServer } from 'http'
import { getPlatform } from '../../utils/platform.js'
```

### 5.2 平台检测

```typescript
// 来自 ../../utils/platform.js
export function getPlatform(): 'windows' | 'macos' | 'linux' {
  // 基于 process.platform 返回平台标识
}
```

### 5.3 环境变量

```typescript
// 可选配置
process.env.MCP_OAUTH_CALLBACK_PORT

// 使用示例
MCP_OAUTH_CALLBACK_PORT=8080 claude mcp add ...
```

### 5.4 端口范围参考

| 平台 | 范围 | 说明 |
|------|------|------|
| 非 Windows | 49152-65535 | IANA 动态/私有端口 |
| Windows | 39152-49151 | 避开系统保留端口 |
| 回退 | 3118 | 固定回退端口 |

**Windows 特殊处理原因**：
- Windows 动态端口范围 `49152-65535` 被系统保留
- 使用 `39152-49151` 避免与系统服务冲突

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 安全风险
| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 端口预测 | 固定回退端口可能被攻击者预测 | 优先使用随机端口 |
| 端口劫持 | 恶意程序占用 OAuth 回调端口 | 随机选择降低概率 |
| 中间人攻击 | 不安全的回调 URI 可能被拦截 | 仅使用 localhost |

#### 6.1.2 功能边界
| 边界 | 说明 |
|------|------|
| 仅支持 localhost | 不支持自定义主机名 |
| 固定路径 | 仅支持 `/callback` 路径 |
| 无 IPv6 | 仅支持 IPv4 localhost |
| 单端口 | 每个 OAuth 流程使用单一端口 |

### 6.2 潜在问题

#### 6.2.1 端口竞争条件
```typescript
// 问题：测试端口可用后到实际使用之间，端口可能被占用
// 时间窗口：testServer.close() 到实际 OAuth 服务器启动

// 缓解：时间窗口很短，且随机端口降低冲突概率
```

#### 6.2.2 防火墙问题
```typescript
// 问题：某些防火墙可能阻止非标准端口的 localhost 连接
// 特别是企业环境中的主机防火墙

// 缓解：回退端口 3118 相对常见，可能已被允许
```

#### 6.2.3 并发 OAuth 流程
```typescript
// 问题：同时连接多个需要 OAuth 的 MCP 服务器
// 每个都需要独立的回调端口

// 当前实现：每个流程独立调用 findAvailablePort()
// 潜在冲突：两个流程可能同时选中同一端口

// 缓解：随机选择 + 快速使用降低冲突概率
```

### 6.3 改进建议

#### 6.3.1 功能增强
1. **端口池管理**：
   ```typescript
   class PortPool {
     private available: Set<number>
     private inUse: Set<number>
     
     async acquire(): Promise<number> {
       // 从可用池分配，避免并发冲突
     }
     
     release(port: number): void {
       // 归还到可用池
     }
   }
   ```

2. **IPv6 支持**：
   ```typescript
   export function buildRedirectUri(
     port: number = REDIRECT_PORT_FALLBACK,
     useIPv6 = false
   ): string {
     const host = useIPv6 ? '[::1]' : 'localhost'
     return `http://${host}:${port}/callback`
   }
   ```

3. **自定义路径**：
   ```typescript
   export function buildRedirectUri(
     port: number = REDIRECT_PORT_FALLBACK,
     path = '/callback'
   ): string {
     return `http://localhost:${port}${path}`
   }
   ```

#### 6.3.2 可靠性增强
1. **端口预留**：
   ```typescript
   // 找到端口后立即占用，直到 OAuth 服务器启动
   export async function reservePort(): Promise<{ port: number; release: () => void }> {
     const server = createServer()
     // ... 查找端口
     return {
       port,
       release: () => server.close()
     }
   }
   ```

2. **重试机制**：
   ```typescript
   export async function findAvailablePort(
     maxRetries = 3
   ): Promise<number> {
     for (let i = 0; i < maxRetries; i++) {
       try {
         return await attemptFindPort()
       } catch (error) {
         if (i === maxRetries - 1) throw error
         await sleep(100)  // 短暂等待后重试
       }
     }
   }
   ```

3. **端口健康检查**：
   ```typescript
   async function isPortHealthy(port: number): Promise<boolean> {
     try {
       const response = await fetch(`http://localhost:${port}/health`)
       return response.ok
     } catch {
       return false
     }
   }
   ```

#### 6.3.3 可观测性
1. **端口使用统计**：
   ```typescript
   logEvent('tengu_mcp_oauth_port_selected', {
     port,
     isFallback: port === REDIRECT_PORT_FALLBACK,
     attempts
   })
   ```

2. **冲突检测**：
   ```typescript
   if (attempts > 10) {
     logWarning('High port contention detected', { attempts, port })
   }
   ```

#### 6.3.4 代码质量
1. **单元测试**：
   ```typescript
   describe('findAvailablePort', () => {
     it('returns configured port when set', async () => {
       process.env.MCP_OAUTH_CALLBACK_PORT = '8888'
       const port = await findAvailablePort()
       expect(port).toBe(8888)
       delete process.env.MCP_OAUTH_CALLBACK_PORT
     })
     
     it('returns port in valid range', async () => {
       const port = await findAvailablePort()
       expect(port).toBeGreaterThanOrEqual(REDIRECT_PORT_RANGE.min)
       expect(port).toBeLessThanOrEqual(REDIRECT_PORT_RANGE.max)
     })
     
     it('falls back when all ports in use', async () => {
       // 模拟所有端口被占用
       // ...
       const port = await findAvailablePort()
       expect(port).toBe(REDIRECT_PORT_FALLBACK)
     })
   })
   ```

2. **类型安全**：
   ```typescript
   type Port = number & { __brand: 'ValidPort' }
   
   function validatePort(port: number): Port {
     if (port < 1 || port > 65535) {
       throw new Error('Invalid port number')
     }
     return port as Port
   }
   ```

### 6.4 测试建议

| 测试场景 | 优先级 |
|----------|--------|
| 配置端口优先 | 高 |
| 随机端口选择 | 高 |
| 回退端口机制 | 高 |
| 端口占用检测 | 高 |
| 平台特定范围 | 中 |
| 并发端口分配 | 中 |
| 超时处理 | 低 |
