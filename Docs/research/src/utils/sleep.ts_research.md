# sleep.ts 深度研究

## 场景与职责

`sleep.ts` 提供**可中断的延迟和超时工具**，支持 AbortSignal 以实现优雅的取消和关闭。

**核心职责：**
1. 提供可中断的 sleep 函数
2. 提供 Promise 超时竞争功能
3. 支持自定义中止错误

**应用场景：**
- 重试循环中的延迟
- API 调用超时
- 优雅关闭时的定时器清理

---

## 功能点目的

### 1. 可中断 Sleep
```typescript
export function sleep(
  ms: number,
  signal?: AbortSignal,
  opts?: { 
    throwOnAbort?: boolean
    abortError?: () => Error
    unref?: boolean 
  }
): Promise<void>
```

**行为：**
| 信号状态 | throwOnAbort | 结果 |
|---------|--------------|------|
| 已中止 | false/undefined | 立即 resolve |
| 已中止 | true | 立即 reject（使用 abortError 或默认错误）|
| 未中止 | - | 等待 ms 毫秒或信号中止 |

**unref 选项：**
- 设置 `unref: true` 时，定时器不阻止进程退出
- 适用于后台任务的延迟

### 2. 超时竞争
```typescript
export function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
  message: string
): Promise<T>
```

**特点：**
- 与 Promise 竞争超时
- 超时后清除定时器（无悬挂定时器）
- 定时器 `unref` 避免阻塞进程退出
- **注意**：不取消底层操作

---

## 具体技术实现

### Sleep 实现细节
```typescript
return new Promise((resolve, reject) => {
  // 1. 先检查已中止状态（避免时序竞态）
  if (signal?.aborted) {
    if (opts?.throwOnAbort || opts?.abortError) {
      void reject(opts.abortError?.() ?? new Error('aborted'))
    } else {
      void resolve()
    }
    return
  }

  // 2. 设置定时器
  const timer = setTimeout(
    (signal, onAbort, resolve) => {
      signal?.removeEventListener('abort', onAbort)
      void resolve()
    },
    ms,
    signal,
    onAbort,
    resolve
  )

  // 3. 设置中止监听器
  function onAbort(): void {
    clearTimeout(timer)
    if (opts?.throwOnAbort || opts?.abortError) {
      void reject(opts.abortError?.() ?? new Error('aborted'))
    } else {
      void resolve()
    }
  }
  signal?.addEventListener('abort', onAbort, { once: true })

  // 4. 可选 unref
  if (opts?.unref) {
    timer.unref()
  }
})
```

**关键设计：**
- 先检查 `signal.aborted` 再设置监听器，避免竞态
- 定时器回调中移除事件监听器，避免内存泄漏
- 使用 `void` 操作符明确标记异步操作

### Timeout 实现
```typescript
export function withTimeout<T>(promise: Promise<T>, ms: number, message: string): Promise<T> {
  let timer: ReturnType<typeof setTimeout> | undefined
  const timeoutPromise = new Promise<never>((_, reject) => {
    timer = setTimeout(rejectWithTimeout, ms, reject, message)
    if (typeof timer === 'object') timer.unref?.()
  })
  return Promise.race([promise, timeoutPromise]).finally(() => {
    if (timer !== undefined) clearTimeout(timer)
  })
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `sleep` | 可中断延迟 |
| `withTimeout` | Promise 超时竞争 |

### 调用方（广泛使用的模块）
| 文件 | 用途 |
|------|------|
| `src/services/settingsSync/index.ts` | 设置同步 |
| `src/services/remoteManagedSettings/index.ts` | 远程设置 |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | 会话内存 |
| `src/services/policyLimits/index.ts` | 策略限制 |
| `src/services/compact/compact.ts` | 压缩 |
| `src/services/api/filesApi.ts` | 文件 API |
| `src/services/teamMemorySync/index.ts` | 团队内存 |
| `src/history.ts` | 历史记录 |
| `src/cli/transports/ccrClient.ts` | CCR 客户端 |
| `src/cli/print.ts` | 打印处理 |
| `src/services/api/withRetry.ts` | 重试逻辑 |
| `src/bridge/replBridge.ts` | REPL Bridge |
| `src/services/mcp/client.ts` | MCP 客户端 |
| `src/utils/gracefulShutdown.ts` | 优雅关闭 |
| `src/utils/worktree.ts` | Worktree 操作 |
| `src/hooks/useVoice.ts` | 语音功能 |

---

## 依赖与外部交互

### 外部依赖
- 标准 Promise 和定时器 API
- AbortSignal / AbortController

### 内部依赖
- 无内部依赖

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**
   - 信号在检查后和监听器设置前中止可能导致不一致
   - 缓解：先检查状态，使用 `{ once: true }`

2. **内存泄漏**
   - 未正确清理的事件监听器
   - 缓解：定时器触发时移除监听器

3. **withTimeout 不取消底层操作**
   - 仅返回控制权，不中止进行中的操作
   - 调用方需自行处理资源清理

### 边界情况

| 场景 | 处理 |
|------|------|
| ms = 0 | 立即触发定时器 |
| ms < 0 | 行为取决于 JavaScript 引擎 |
| signal 为 undefined | 作为普通 sleep |
| 重复中止 | `{ once: true }` 确保只触发一次 |
| withTimeout 已 settle | `finally` 确保清理定时器 |

### 改进建议

1. **精度控制**
   - 添加高精度定时器支持（`process.hrtime`）
   - 支持微秒级延迟

2. **统计信息**
   - 记录实际延迟与请求延迟的差异
   - 监控定时器堆积

3. **批量操作**
   - 添加 `sleepSequence` 支持多个延迟
   - 实现指数退避工具

4. **类型增强**
   - 区分已中止和正常完成的返回类型
   - 支持类型安全的错误传递

5. **测试工具**
   - 提供虚拟定时器用于测试
   - 支持时间快进
