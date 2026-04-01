# manager.ts 深度研究文档

## 场景与职责

`manager.ts` 是 LSP 子系统的单例管理器，负责管理全局唯一的 `LSPServerManager` 实例生命周期，提供初始化状态追踪、并发控制和重新初始化支持。

**核心职责：**
1. **单例管理**：维护全局唯一的 LSP Server Manager 实例
2. **初始化状态追踪**：管理 'not-started' | 'pending' | 'success' | 'failed' 状态机
3. **并发安全**：使用 generation counter 防止过时的初始化 Promise 更新状态
4. **重新初始化支持**：支持插件刷新后的强制重新初始化
5. **生命周期钩子**：提供初始化和关闭的入口点

**在系统中的位置：**
- 被 `main.tsx` 调用进行初始化
- 被 `init.ts` 注册关闭钩子
- 被 `LSPTool.ts` 调用获取管理器实例
- 被 `useManagePlugins.ts` 和 `refresh.ts` 调用进行重新初始化
- 是 LSP 架构的最顶层（manager.ts → LSPServerManager → LSPServerInstance → LSPClient）

---

## 功能点目的

### 1. 单例状态管理

**模块级状态变量：**
```typescript
let lspManagerInstance: LSPServerManager | undefined  // 实例
let initializationState: InitializationState = 'not-started'  // 状态
let initializationError: Error | undefined             // 错误
let initializationGeneration = 0                       // 代际计数器
let initializationPromise: Promise<void> | undefined   // 初始化 Promise
```

**状态流转：**
```
not-started → pending → success
                    ↘ failed

success/failed → not-started → pending → success  (重新初始化)
```

### 2. 初始化流程

**`initializeLspServerManager()` 核心逻辑（Lines 145-208）：**

1. **bare 模式检查**：
   ```typescript
   if (isBareMode()) return  // --bare / SIMPLE 模式无 LSP
   ```

2. **幂等性检查**：
   ```typescript
   if (lspManagerInstance !== undefined && initializationState !== 'failed') {
     return  // 已初始化或正在初始化，跳过
   }
   ```

3. **失败状态重置**：
   ```typescript
   if (initializationState === 'failed') {
     lspManagerInstance = undefined
     initializationError = undefined
   }
   ```

4. **创建实例并标记 pending**：
   ```typescript
   lspManagerInstance = createLSPServerManager()
   initializationState = 'pending'
   ```

5. **代际计数器递增**：
   ```typescript
   const currentGeneration = ++initializationGeneration
   ```

6. **异步初始化**：
   ```typescript
   initializationPromise = lspManagerInstance.initialize()
     .then(() => {
       if (currentGeneration === initializationGeneration) {
         initializationState = 'success'
         registerLSPNotificationHandlers(lspManagerInstance)  // 注册诊断处理器
       }
     })
     .catch((error) => {
       if (currentGeneration === initializationGeneration) {
         initializationState = 'failed'
         initializationError = error as Error
         lspManagerInstance = undefined
       }
     })
   ```

### 3. 代际计数器模式

**目的：** 防止过时的初始化 Promise 更新当前状态

**场景：**
- 调用 A 开始初始化（generation = 1）
- 调用 B 强制重新初始化（generation = 2）
- A 的 Promise resolve 时，检查 `currentGeneration === initializationGeneration`（1 === 2？否）
- A 的结果被忽略，不会错误地将状态设为 success

### 4. 重新初始化

**`reinitializeLspServerManager()` 核心逻辑（Lines 226-253）：**

**触发场景：**
- 插件刷新后（`refreshActivePlugins()`）
- Issue #15521：修复插件缓存导致 LSP 服务器未加载的问题

**实现步骤：**
1. 检查状态，如果从未初始化则跳过
2. 尝试关闭旧实例（fire-and-forget，不等待）
3. 重置所有状态变量
4. 递增 generation
5. 调用 `initializeLspServerManager()`

### 5. 关闭流程

**`shutdownLspServerManager()` 核心逻辑（Lines 267-289）：**

**设计决策：**
- 错误被吞掉（swallowed），不传播给调用方
- 状态总是重置，即使关闭失败
- 适用于应用退出场景，恢复不可能

---

## 具体技术实现

### 状态查询 API

```typescript
// 获取管理器实例（可能为 undefined）
export function getLspServerManager(): LSPServerManager | undefined

// 获取初始化状态
export function getInitializationStatus():
  | { status: 'not-started' }
  | { status: 'pending' }
  | { status: 'success' }
  | { status: 'failed'; error: Error }

// 检查是否有健康的服务器连接
export function isLspConnected(): boolean

// 等待初始化完成
export async function waitForInitialization(): Promise<void>
```

### isLspConnected 实现

```typescript
export function isLspConnected(): boolean {
  if (initializationState === 'failed') return false
  const manager = getLspServerManager()
  if (!manager) return false
  const servers = manager.getAllServers()
  if (servers.size === 0) return false
  for (const server of servers.values()) {
    if (server.state !== 'error') return true  // 至少一个非错误状态
  }
  return false
}
```

**用途：**
- `LSPTool.ts` 的 `isEnabled()` 方法使用
- 控制 LSP 工具是否可用

### 测试辅助函数

```typescript
export function _resetLspManagerForTesting(): void {
  initializationState = 'not-started'
  initializationError = undefined
  initializationPromise = undefined
  initializationGeneration++
}
```

**说明：**
- 仅用于测试
- 不清除实际连接（`shutdownLspServerManager` 才是正式关闭）
- 使 `reinitializeLspServerManager` 在测试中能从 'not-started' 开始

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `../../utils/debug.js` | `logForDebugging()` - 调试日志 |
| `../../utils/envUtils.js` | `isBareMode()` - 检查 bare 模式 |
| `../../utils/errors.js` | `errorMessage()` - 错误处理 |
| `../../utils/log.js` | `logError()` - 错误日志 |
| `./LSPServerManager.js` | `createLSPServerManager()` - 创建管理器 |
| `./passiveFeedback.js` | `registerLSPNotificationHandlers()` - 注册诊断处理器 |

### 被调用方

| 文件 | 调用函数 |
|------|----------|
| `main.tsx` | `initializeLspServerManager()` - 应用启动时初始化 |
| `entrypoints/init.ts` | `shutdownLspServerManager()` - 注册关闭钩子 |
| `tools/LSPTool/LSPTool.ts` | `getLspServerManager()`, `waitForInitialization()`, `getInitializationStatus()` |
| `hooks/useManagePlugins.ts` | `reinitializeLspServerManager()` - 插件管理后刷新 |
| `utils/plugins/refresh.ts` | `reinitializeLspServerManager()` - 插件刷新 |

### 关键代码行

| 行号 | 功能 |
|------|------|
| 14 | `InitializationState` 类型定义 |
| 20-35 | 模块级状态变量 |
| 48-53 | `_resetLspManagerForTesting()` 测试辅助 |
| 63-69 | `getLspServerManager()` 实现 |
| 76-94 | `getInitializationStatus()` 实现 |
| 100-110 | `isLspConnected()` 实现 |
| 121-133 | `waitForInitialization()` 实现 |
| 145-208 | `initializeLspServerManager()` 主初始化 |
| 173 | generation 递增 |
| 182-191 | 初始化成功处理 |
| 194-207 | 初始化失败处理 |
| 226-253 | `reinitializeLspServerManager()` 重新初始化 |
| 238-243 | 旧实例关闭（fire-and-forget）|
| 267-289 | `shutdownLspServerManager()` 关闭 |

---

## 依赖与外部交互

### 外部依赖

**环境检测：**
- `isBareMode()` - 检测 `--bare` 或 `SIMPLE` 模式，跳过 LSP 初始化

**插件系统：**
- `refresh.ts` - 插件刷新后触发重新初始化

### 初始化时序

```
main.tsx 启动
    ↓
initializeLspServerManager() 同步返回
    ↓
（异步）LSPServerManager.initialize()
    ↓
加载插件配置 → 创建服务器实例
    ↓
registerLSPNotificationHandlers() 注册诊断处理器
    ↓
状态变为 'success'
```

### 关闭时序

```
应用退出
    ↓
init.ts 注册的 cleanup 钩子
    ↓
shutdownLspServerManager()
    ↓
LSPServerManager.shutdown() → 关闭所有服务器
    ↓
状态重置为 'not-started'
```

---

## 风险、边界与改进建议

### 已知风险

**1. 重新初始化竞态**
```typescript
// 旧实例关闭是 fire-and-forget
void lspManagerInstance.shutdown().catch(err => {...})

// 立即开始新初始化
initializeLspServerManager()
```
- 旧实例可能仍在关闭过程中
- 可能导致端口冲突或资源竞争

**2. 无超时控制**
- `waitForInitialization()` 可能永远等待（如果初始化卡住）
- 建议添加可选的超时参数

**3. 失败状态处理**
- 初始化失败后，需要显式调用重新初始化才能重试
- 没有自动重试机制

### 边界情况

**1. bare 模式**
```typescript
if (isBareMode()) {
  return  // 完全跳过 LSP
}
```
- `--bare` 或 `SIMPLE` 模式不启动 LSP
- 适用于脚本化调用，无需编辑器集成

**2. 并发初始化调用**
```typescript
if (lspManagerInstance !== undefined && initializationState !== 'failed') {
  return  // 幂等：已在初始化中
}
```
- 多次调用不会创建多个实例
- 但重新初始化可以覆盖进行中的初始化

**3. 初始化期间查询**
```typescript
// LSPTool.ts 中的处理
const status = getInitializationStatus()
if (status.status === 'pending') {
  await waitForInitialization()
}
```
- 工具调用可能等待初始化完成
- 避免返回 "未初始化" 错误

### 改进建议

**1. 重新初始化等待**
```typescript
// 建议：等待旧实例完全关闭
export async function reinitializeLspServerManager(): Promise<void> {
  if (lspManagerInstance) {
    await lspManagerInstance.shutdown()  // 等待而非 fire-and-forget
  }
  // ...
}
```

**2. 初始化超时**
```typescript
// 建议：添加超时参数
export async function waitForInitialization(timeoutMs?: number): Promise<void> {
  if (!timeoutMs) {
    await initializationPromise
    return
  }
  await Promise.race([
    initializationPromise,
    sleep(timeoutMs).then(() => { throw new Error('Initialization timeout') })
  ])
}
```

**3. 自动重试**
```typescript
// 建议：失败后的指数退避重试
let retryCount = 0
const MAX_RETRIES = 3

export function initializeLspServerManager(): void {
  if (initializationState === 'failed' && retryCount >= MAX_RETRIES) {
    return  // 超过重试次数
  }
  // ...
  initializationPromise = lspManagerInstance.initialize()
    .catch((error) => {
      retryCount++
      if (retryCount < MAX_RETRIES) {
        setTimeout(initializeLspServerManager, 1000 * Math.pow(2, retryCount))
      }
    })
}
```

**4. 健康检查定时器**
```typescript
// 建议：定期检查服务器健康
let healthCheckInterval: ReturnType<typeof setInterval>

function startHealthChecks(): void {
  healthCheckInterval = setInterval(() => {
    const manager = getLspServerManager()
    if (!manager) return
    for (const [name, server] of manager.getAllServers()) {
      if (!server.isHealthy()) {
        logForDebugging(`LSP server ${name} unhealthy`)
        // 可选：自动重启
      }
    }
  }, 30000)  // 30秒
}
```

**5. 初始化进度报告**
```typescript
// 建议：支持进度回调
interface InitializationProgress {
  phase: 'loading-plugins' | 'creating-instances' | 'initializing-servers'
  current: number
  total: number
}

export function initializeLspServerManager(
  onProgress?: (progress: InitializationProgress) => void
): void
```

### 测试建议

**关键测试场景：**
1. 多次调用 `initializeLspServerManager` 的幂等性
2. 初始化期间的 `waitForInitialization` 行为
3. 重新初始化时的代际计数器效果
4. 初始化失败后的状态恢复
5. bare 模式下的跳过行为
6. `isLspConnected` 的各种状态组合
7. 关闭期间的并发调用

### 相关 Issue

**Issue #15521：**
- `loadAllPlugins()` 被 memoize，可能在市场协调前被调用
- 缓存空插件列表，导致 LSP 初始化时服务器为 0
- 修复：插件刷新后调用 `reinitializeLspServerManager()`

```typescript
/**
 * Fixes https://github.com/anthropics/claude-code/issues/15521:
 * loadAllPlugins() is memoized and can be called very early in startup
 * (via getCommands prefetch in setup.ts) before marketplaces are reconciled,
 * caching an empty plugin list.
 */
```
