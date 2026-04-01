# LSPServerInstance.ts 深度研究文档

## 场景与职责

`LSPServerInstance.ts` 是 LSP 子系统的中间层，负责管理单个 LSP 服务器的完整生命周期，包括启动、初始化、健康监控、请求转发和优雅关闭。

**核心职责：**
1. **生命周期管理**：管理 LSP 服务器的完整状态机（stopped → starting → running → stopping → stopped/error）
2. **初始化协调**：执行 LSP 协议标准的初始化握手（initialize → initialized）
3. **健康监控**：提供 `isHealthy()` 检查，确保服务器就绪后才发送请求
4. **请求转发**：将 LSP 请求转发到底层客户端，支持 transient 错误重试
5. **崩溃恢复**：检测服务器崩溃并支持有限次数的自动重启
6. **配置管理**：处理服务器特定的初始化选项和客户端 capabilities

**在系统中的位置：**
- 被 `LSPServerManager.ts` 调用，作为多个服务器实例的管理单元
- 调用 `LSPClient.ts` 进行实际的 JSON-RPC 通信
- 属于 LSP 架构的中层（LSPClient ← LSPServerInstance ← LSPServerManager）

---

## 功能点目的

### 1. LSPServerInstance 接口

定义单个 LSP 服务器实例的公共 API：
```typescript
export type LSPServerInstance = {
  readonly name: string                    // 服务器标识
  readonly config: ScopedLspServerConfig   // 配置
  readonly state: LspServerState           // 当前状态
  readonly startTime: Date | undefined     // 启动时间
  readonly lastError: Error | undefined    // 最后错误
  readonly restartCount: number            // 手动重启次数
  start(): Promise<void>                   // 启动并初始化
  stop(): Promise<void>                    // 优雅关闭
  restart(): Promise<void>                 // 手动重启
  isHealthy(): boolean                     // 健康检查
  sendRequest<T>(method, params): Promise<T>      // 发送请求
  sendNotification(method, params): Promise<void> // 发送通知
  onNotification(method, handler): void    // 注册通知处理器
  onRequest<TParams, TResult>(method, handler): void  // 注册请求处理器
}
```

### 2. 状态机

**状态定义：** `LspServerState = 'stopped' | 'starting' | 'running' | 'stopping' | 'error'`

**状态转换：**
```
stopped → starting → running
                    ↓
              stopping → stopped
                    ↓
                   error
                    ↓
              starting (重启)
```

### 3. 初始化流程

**`start()` 方法核心逻辑：**
1. 状态检查：已在运行则直接返回
2. 崩溃恢复限制：检查 `crashRecoveryCount` 是否超过 `maxRestarts`
3. 状态设置为 `starting`
4. 启动 LSPClient（进程 + 连接）
5. 构建 `InitializeParams`：
   - `processId`: 当前进程 PID
   - `workspaceFolders`: 工作区信息（LSP 3.16+）
   - `rootPath/rootUri`: 向后兼容字段
   - `capabilities`: 客户端能力声明
   - `initializationOptions`: 服务器特定选项
6. 发送 `initialize` 请求（支持超时）
7. 发送 `initialized` 通知
8. 状态更新为 `running`

### 4. Transient 错误重试

**`sendRequest()` 重试逻辑：**
```typescript
const LSP_ERROR_CONTENT_MODIFIED = -32801  // LSP 标准错误码
const MAX_RETRIES = 3
const RETRY_BASE_DELAY_MS = 500

for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
  try {
    return await client.sendRequest(method, params)
  } catch (error) {
    if (error.code === LSP_ERROR_CONTENT_MODIFIED && attempt < MAX_RETRIES) {
      await sleep(RETRY_BASE_DELAY_MS * Math.pow(2, attempt))  // 指数退避
      continue
    }
    throw error
  }
}
```

**适用场景：**
- rust-analyzer 等服务器在索引期间返回 `ContentModified`
- 符合 LSP 规范，客户端应静默重试

### 5. 崩溃检测与恢复

**崩溃检测：**
```typescript
const client = createLSPClient(name, error => {
  state = 'error'
  lastError = error
  crashRecoveryCount++
})
```

**恢复策略：**
- 下次请求时检查状态
- 如果状态为 `error` 且未超过 `maxRestarts`，自动重启
- 超过限制则抛出错误

---

## 具体技术实现

### 关键数据结构

**客户端 Capabilities（Lines 188-236）：**
```typescript
capabilities: {
  workspace: {
    configuration: false,      // 不支持 workspace/configuration
    workspaceFolders: false,   // 不支持工作区变更
  },
  textDocument: {
    synchronization: {
      didSave: true,           // 支持文件保存通知
    },
    publishDiagnostics: {
      relatedInformation: true,
      tagSupport: { valueSet: [1, 2] },  // Unnecessary, Deprecated
      codeDescriptionSupport: true,
    },
    hover: { contentFormat: ['markdown', 'plaintext'] },
    definition: { linkSupport: true },
    documentSymbol: { hierarchicalDocumentSymbolSupport: true },
  },
  general: { positionEncodings: ['utf-16'] },
}
```

### 懒加载优化

**LSPClient 懒加载（Lines 106-112）：**
```typescript
// vscode-jsonrpc (~129KB) 仅在需要时加载
const { createLSPClient } = require('./LSPClient.js') as {
  createLSPClient: typeof createLSPClientType
}
```

### 超时处理

**`withTimeout()` 辅助函数（Lines 499-511）：**
```typescript
function withTimeout<T>(promise: Promise<T>, ms: number, message: string): Promise<T> {
  let timer: ReturnType<typeof setTimeout>
  const timeoutPromise = new Promise<never>((_, reject) => {
    timer = setTimeout((rej, msg) => rej(new Error(msg)), ms, reject, message)
  })
  return Promise.race([promise, timeoutPromise]).finally(() => clearTimeout(timer!))
}
```

### 未实现配置验证

**防御性编程（Lines 94-104）：**
```typescript
if (config.restartOnCrash !== undefined) {
  throw new Error(`LSP server '${name}': restartOnCrash is not yet implemented.`)
}
if (config.shutdownTimeout !== undefined) {
  throw new Error(`LSP server '${name}': shutdownTimeout is not yet implemented.`)
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `path` (Node.js) | 路径处理 |
| `url.pathToFileURL` | 文件路径转 URI |
| `../../utils/cwd.js` | `getCwd()` - 获取工作目录 |
| `../../utils/debug.js` | `logForDebugging()` - 调试日志 |
| `../../utils/errors.js` | `errorMessage()` - 错误处理 |
| `../../utils/log.js` | `logError()` - 错误日志 |
| `../../utils/sleep.js` | `sleep()` - 重试延迟 |
| `./LSPClient.js` | `createLSPClient` - 底层客户端 |
| `./types.js` | `LspServerState`, `ScopedLspServerConfig` |

### 被调用方

| 文件 | 调用方式 |
|------|----------|
| `LSPServerManager.ts` | `createLSPServerInstance(name, config)` - 创建实例 |

### 关键代码行

| 行号 | 功能 |
|------|------|
| 17 | `LSP_ERROR_CONTENT_MODIFIED` 错误码定义 |
| 22-28 | 重试配置常量 |
| 33-65 | `LSPServerInstance` 接口定义 |
| 90-104 | 工厂函数与未实现配置验证 |
| 106-112 | LSPClient 懒加载 |
| 113-118 | 私有状态定义 |
| 121-125 | 崩溃回调设置 |
| 135-263 | `start()` 完整实现 |
| 167-236 | InitializeParams 构建 |
| 239-248 | 初始化超时处理 |
| 274-290 | `stop()` 实现 |
| 300-331 | `restart()` 实现 |
| 338-340 | `isHealthy()` 实现 |
| 355-410 | `sendRequest()` 含重试逻辑 |
| 416-437 | `sendNotification()` 实现 |
| 445-466 | Handler 注册方法 |
| 499-511 | `withTimeout()` 辅助函数 |

---

## 依赖与外部交互

### 外部依赖

**Node.js 内置：**
- `path` - 路径操作
- `url` - URL 转换

**第三方库：**
- `vscode-languageserver-protocol` - LSP 类型定义
  - `InitializeParams`
  - `ServerCapabilities`

### LSP 协议交互

**标准 LSP 方法：**
| 方法 | 方向 | 用途 |
|------|------|------|
| `initialize` | Client → Server | 初始化握手 |
| `initialized` | Client → Server | 初始化完成通知 |
| `shutdown` | Client → Server | 关闭请求 |
| `exit` | Client → Server | 退出通知 |
| `workspace/configuration` | Server → Client | 配置请求（返回 null）|

**错误码：**
- `-32801` (`ContentModified`) - 内容已修改，应重试

### 配置接口

**ScopedLspServerConfig（来自 types.js）：**
```typescript
interface ScopedLspServerConfig {
  command: string              // 启动命令
  args?: string[]              // 参数
  extensionToLanguage: Record<string, string>  // 扩展名映射
  env?: Record<string, string> // 环境变量
  workspaceFolder?: string     // 工作区目录
  initializationOptions?: unknown  // 初始化选项
  startupTimeout?: number      // 启动超时
  maxRestarts?: number         // 最大重启次数
  scope: 'dynamic' | 'static'  // 作用域
  source: string               // 来源插件
}
```

---

## 风险、边界与改进建议

### 已知风险

**1. 状态机竞态**
- `start()` 和 `stop()` 可能被并发调用
- 当前依赖简单的状态检查，无锁机制
- 可能状态不一致（如正在启动时收到停止请求）

**2. 重试风暴**
- 如果服务器持续返回 `ContentModified`，会进行 3 次重试
- 指数退避（500ms, 1000ms, 2000ms）可缓解，但仍可能累积延迟

**3. 内存泄漏风险**
- `restartCount` 和 `crashRecoveryCount` 只增不减
- 长期运行的会话可能溢出（实际影响很小）

### 边界情况

**1. 初始化超时**
```typescript
if (config.startupTimeout !== undefined) {
  await withTimeout(initPromise, config.startupTimeout, ...)
}
```
- 超时后调用 `client.stop().catch(() => {})` 清理
- 使用 `initPromise?.catch(() => {})` 防止未处理 rejection

**2. 健康检查严格性**
```typescript
function isHealthy(): boolean {
  return state === 'running' && client.isInitialized
}
```
- 需要同时满足状态为 running 且客户端已初始化
- 防止在初始化过程中发送请求

**3. 通知失败处理**
- `sendNotification` 失败会抛出错误（不同于 LSPClient 的静默处理）
- 调用方需要处理通知失败

### 改进建议

**1. 并发控制**
```typescript
// 建议：添加启动/停止锁
let lifecycleLock: Promise<void> | undefined

async function start(): Promise<void> {
  if (lifecycleLock) await lifecycleLock
  lifecycleLock = startInternal()
  await lifecycleLock
  lifecycleLock = undefined
}
```

**2. 健康检查增强**
```typescript
// 建议：添加心跳检测
async function healthCheck(): Promise<boolean> {
  try {
    await client.sendRequest('$/ping', {})
    return true
  } catch {
    return false
  }
}
```

**3. 更细粒度的错误分类**
```typescript
// 建议：区分可重试和不可重试错误
class LSPRetryableError extends Error {}
class LSPPermanentError extends Error {}
```

**4. 配置热更新**
- 当前配置在创建时固定
- 建议支持运行时更新（如 `workspace/didChangeConfiguration`）

**5. 性能指标收集**
```typescript
// 建议：添加指标
interface LSPMetrics {
  requestCount: number
  averageResponseTime: number
  errorRate: number
  restartCount: number
}
```

**6. 实现未完成的配置选项**
- `restartOnCrash` - 自动崩溃恢复
- `shutdownTimeout` - 关闭超时控制

### 测试建议

**关键测试场景：**
1. 初始化超时处理
2. `ContentModified` 错误重试（3 次后失败）
3. 崩溃恢复次数限制
4. 并发 start/stop 调用
5. 健康检查边界（初始化中、已停止、错误状态）
6. 不同服务器的初始化选项（vue-language-server, Pyright, gopls）
7. 长时间运行后的内存使用

### 相关 Issue

- 注释提到 Issue #15521：插件刷新后 LSP 需要重新初始化
- 当前通过 `reinitializeLspServerManager()` 在 `manager.ts` 中处理
