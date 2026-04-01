# src/utils/cleanupRegistry.ts 深度研究文档

## 场景与职责

`cleanupRegistry.ts` 是一个极简的全局清理函数注册表，用于管理在应用优雅关闭时需要执行的清理函数。它是 graceful shutdown 系统的核心组件之一，与 `gracefulShutdown.ts` 配合使用。

设计目标：
- 提供简单的注册/注销机制
- 避免循环依赖（从 `gracefulShutdown.ts` 分离出来）
- 支持异步清理函数
- 确保在进程退出前执行所有清理

## 功能点目的

### 1. 全局清理函数注册
- 允许模块注册在应用关闭时需要执行的清理逻辑
- 返回注销函数，支持动态移除清理处理器

### 2. 批量执行清理
- 在 graceful shutdown 时并行执行所有注册的清理函数
- 使用 `Promise.all` 确保所有清理完成

## 具体技术实现

### 核心数据结构

```typescript
// 全局清理函数集合
const cleanupFunctions = new Set<() => Promise<void>>()
```

### API 设计

```typescript
/**
 * 注册清理函数
 * @param cleanupFn - 清理函数（可以是同步或异步）
 * @returns 注销函数，调用后移除该清理处理器
 */
export function registerCleanup(cleanupFn: () => Promise<void>): () => void

/**
 * 执行所有注册的清理函数
 * 由 gracefulShutdown 内部调用
 */
export async function runCleanupFunctions(): Promise<void>
```

### 使用模式

#### 注册清理函数
```typescript
import { registerCleanup } from './cleanupRegistry.js'

// 注册资源清理
const unregister = registerCleanup(async () => {
  await closeConnections()
  await flushBuffers()
})

// 如果需要，可以注销
// unregister()
```

#### 执行清理（内部使用）
```typescript
// 在 gracefulShutdown.ts 中
import { runCleanupFunctions } from './cleanupRegistry.js'

async function gracefulShutdown(exitCode: number): Promise<void> {
  // ... 其他清理 ...
  
  // 执行所有注册的清理函数
  await runCleanupFunctions()
  
  // ... 退出 ...
}
```

## 依赖与外部交互

### 无外部依赖

此模块是底层基础设施，不依赖任何其他模块。

### 被依赖方

| 模块 | 用途 |
|------|------|
| `src/utils/gracefulShutdown.ts` | 调用 `runCleanupFunctions` |
| `src/utils/config.ts` | 注册配置缓存统计报告 |
| `src/utils/settings/changeDetector.ts` | 注册文件监视器清理 |
| `src/utils/telemetry/perfettoTracing.ts` | 注册 tracing 清理 |
| `src/utils/nativeInstaller/installer.ts` | 注册安装器清理 |
| `src/utils/hooks/fileChangedWatcher.ts` | 注册文件变更监视器清理 |
| `src/utils/skills/skillChangeDetector.ts` | 注册技能变更检测器清理 |
| `src/utils/git/gitFilesystem.ts` | 注册 git 文件系统清理 |
| `src/utils/sessionActivity.ts` | 注册会话活动追踪清理 |
| `src/utils/concurrentSessions.ts` | 注册并发会话检测清理 |
| `src/utils/cleanup.ts` | 注册清理统计报告 |
| `src/utils/sessionStorage.ts` | 注册会话存储清理 |
| `src/utils/tmuxSocket.ts` | 注册 tmux socket 清理 |
| `src/utils/plugins/headlessPluginInstall.ts` | 注册插件安装清理 |
| `src/utils/cronTasksLock.ts` | 注册 cron 任务锁清理 |
| `src/utils/errorLogSink.ts` | 注册错误日志接收器清理 |
| `src/utils/streamJsonStdoutGuard.ts` | 注册 stdout guard 清理 |
| `src/utils/swarm/backends/PaneBackendExecutor.ts` | 注册 pane backend 清理 |

## 风险、边界与改进建议

### 已知风险

1. **清理函数异常**
   - 当前实现中，如果某个清理函数抛出异常，会影响 `Promise.all`
   - 但 gracefulShutdown 中的调用有 try/catch 包装

2. **清理函数超时**
   - 没有内置超时机制
   - 如果清理函数挂起，会阻塞 graceful shutdown
   - gracefulShutdown 有总体 failsafe 计时器（5s + hook budget）

3. **重复注册**
   - 使用 Set 存储函数引用，同一函数多次注册只会执行一次
   - 但匿名函数每次都会被视为不同函数

### 边界情况

1. **空注册表**
   - `runCleanupFunctions` 对空 Set 调用 `Promise.all([])`，立即 resolve

2. **同步函数**
   - 类型要求 `Promise<void>`，但同步函数也会工作（返回 undefined 会被包装为 resolved promise）

3. **清理函数中注册新清理函数**
   - 在 `runCleanupFunctions` 执行期间注册新函数不会在本次执行中被调用
   - 因为 Set 在遍历前已被转换为 Array

### 改进建议

1. **增强功能**
   - 添加清理函数优先级/顺序支持（某些清理需要在其他之前执行）
   - 添加命名清理函数（便于调试和日志记录）
   - 添加清理函数执行超时控制

2. **可观测性**
   - 记录每个清理函数的执行时间
   - 添加清理函数注册/执行日志
   - 报告未执行的清理函数（如果 graceful shutdown 被中断）

3. **健壮性**
   - 包装每个清理函数，确保单个失败不影响其他
   - 添加清理函数执行时间限制
   - 支持清理函数的依赖关系（DAG 执行顺序）

4. **代码示例**

```typescript
// 改进版本示例（带超时和错误隔离）
export async function runCleanupFunctions(): Promise<void> {
  const results = await Promise.allSettled(
    Array.from(cleanupFunctions).map(async (fn, index) => {
      const timeout = setTimeout(() => {
        console.warn(`Cleanup function ${index} timed out`)
      }, 5000)
      
      try {
        await fn()
      } finally {
        clearTimeout(timeout)
      }
    })
  )
  
  // 报告失败的清理
  results.forEach((result, index) => {
    if (result.status === 'rejected') {
      console.error(`Cleanup function ${index} failed:`, result.reason)
    }
  })
}
```

5. **架构考虑**
   - 当前是全局单例，考虑支持作用域注册表（如 per-session 清理）
   - 与依赖注入框架集成（如果未来引入）
