# combinedAbortSignal.ts 研究文档

> 文件路径：`src/utils/combinedAbortSignal.ts`  
> 行数：47 行  
> 研究日期：2026-04-01

---

## 场景与职责

在 Claude Code 的异步执行链路中（如 CLI 打印、Hook 执行、HTTP 请求、Agent 调用），经常需要把多个取消信号源组合在一起：用户主动取消（`Ctrl+C`）、父级超时、子级超时等。标准的 `AbortSignal.any()` 或 `AbortSignal.timeout()` 在 Bun 运行时存在已知的内存泄漏问题——Bun 的 `AbortSignal.timeout` 定时器会被延迟 finalize，在超时触发前持续占用原生内存（实测约 2.4KB/次）。

`combinedAbortSignal.ts` 提供了一个**显式清理**的组合取消信号实现：
- 支持组合最多两个输入信号 + 一个可选的 `setTimeout` 超时。
- 返回组合后的 `AbortSignal` 以及一个 `cleanup()` 函数，调用方可显式移除监听器、清除定时器，避免内存堆积。

---

## 功能点目的

| 导出项 | 签名 | 目的 |
|--------|------|------|
| `createCombinedAbortSignal` | `(signal, opts?) => { signal: AbortSignal; cleanup: () => void }` | 创建一个在任一输入信号 abort 或超时到达时自动 abort 的组合信号，并暴露清理入口 |

**使用约束（代码注释强调）：**
- 调用方应优先使用本函数的 `timeoutMs` 选项，而不是把 `AbortSignal.timeout(ms)` 作为外部 `signal` 传入。

---

## 具体技术实现

### 3.1 实现细节

```ts
export function createCombinedAbortSignal(
  signal: AbortSignal | undefined,
  opts?: { signalB?: AbortSignal; timeoutMs?: number },
): { signal: AbortSignal; cleanup: () => void } {
  const { signalB, timeoutMs } = opts ?? {}
  const combined = createAbortController()

  // 快速路径：任一输入已 abort，直接 abort 并返回空清理
  if (signal?.aborted || signalB?.aborted) {
    combined.abort()
    return { signal: combined.signal, cleanup: () => {} }
  }

  let timer: ReturnType<typeof setTimeout> | undefined
  const abortCombined = () => {
    if (timer !== undefined) clearTimeout(timer)
    combined.abort()
  }

  if (timeoutMs !== undefined) {
    timer = setTimeout(abortCombined, timeoutMs)
    timer.unref?.()          // 避免阻止进程退出（Node/Bun 兼容）
  }
  signal?.addEventListener('abort', abortCombined)
  signalB?.addEventListener('abort', abortCombined)

  const cleanup = () => {
    if (timer !== undefined) clearTimeout(timer)
    signal?.removeEventListener('abort', abortCombined)
    signalB?.removeEventListener('abort', abortCombined)
  }

  return { signal: combined.signal, cleanup }
}
```

### 3.2 设计要点

1. **快速路径（Fast Path）**
   若任一输入信号已经 `aborted`，立即 abort 组合信号并返回无操作清理函数，避免创建无意义的定时器和监听器。

2. **单一 abort 处理器**
   `abortCombined` 同时负责清除定时器和触发 `combined.abort()`，保证无论超时还是外部信号触发，资源都能被清理。

3. **`timer.unref?.()`**
   在 Node/Bun 中，`setTimeout` 默认会阻止进程退出。加上 `unref()` 后，如果事件循环中只剩该定时器，进程可以正常退出。这对于 CLI 工具尤为重要。

4. **显式 cleanup**
   与 `AbortSignal.any` 不同，本实现要求调用方在操作完成后手动调用 `cleanup()`。这虽然增加了调用方的责任，但彻底规避了 Bun 的内存问题。

### 3.3 与 `abortController.ts` 的关系

`createCombinedAbortSignal` 依赖同目录下的 `abortController.ts` 中的 `createAbortController()`。后者封装了 `new AbortController()` 并调用 `setMaxListeners(50)`，防止在高并发场景下出现 `MaxListenersExceededWarning`。

---

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 用途 |
|------|------|
| `src/utils/abortController.ts` | `createAbortController` — 创建带 50 个 max listeners 的 AbortController |

### 调用方
| 文件 | 调用场景 |
|------|----------|
| `src/utils/hooks.ts` | Hook 执行框架中组合用户取消与超时信号 |
| `src/utils/hooks/execPromptHook.ts` | Prompt hook 执行 |
| `src/utils/hooks/execHttpHook.ts` | HTTP hook 执行 |
| `src/utils/hooks/execAgentHook.ts` | Agent hook 执行 |
| `src/cli/print.ts` | CLI 打印输出时的取消控制 |

---

## 依赖与外部交互

- **无外部 I/O**：纯内存操作，仅依赖标准 Web API（`AbortController`、`AbortSignal`、`setTimeout`/`clearTimeout`）和 `events.setMaxListeners`。
- **Bun 运行时适配**：核心动机是绕过 Bun 的 `AbortSignal.timeout` 内存泄漏，属于运行时兼容性层。

---

## 风险、边界与改进建议

### 风险与边界

1. **cleanup 必须显式调用**
   如果调用方忘记 `cleanup()`，定时器和事件监听器会泄漏。虽然比 `AbortSignal.timeout` 的问题小（因为定时器最终还是会 fire），但在高频调用场景（如每轮 LLM 请求）仍可能累积。

2. **不支持超过两个信号**
   目前只支持 `signal` + `signalB` + `timeoutMs`。如果有第三个信号源（如 `signalC`），需要嵌套调用或扩展参数。

3. **unref 的兼容性**
   `timer.unref?.()` 在浏览器环境中不存在，但 Claude Code 是 Node/Bun CLI 应用，因此安全。若未来有浏览器端复用需求，需加环境判断。

4. **竞态条件**
   在 `abortCombined` 执行期间，如果 `signal` 和 `timeout` 同时触发，由于共享同一个处理器且内部有 `clearTimeout(timer)`，不会导致重复 abort 或异常。`AbortController.abort()` 是幂等的。

### 改进建议

1. **支持可变长信号数组**
   将签名改为 `createCombinedAbortSignal(signal?: AbortSignal, opts?: { signals?: AbortSignal[]; timeoutMs?: number })`，以支持任意数量的信号组合，避免嵌套。

2. **自动 cleanup 封装**
   可提供一个高阶包装函数 `withCombinedAbortSignal(signal, opts, async (combinedSignal) => { ... })`，在 Promise 完成后自动调用 `cleanup()`，减少调用方遗漏风险。

3. **增加调试日志**
   在 `abortCombined` 被触发时记录触发源（signal / signalB / timeout），便于排查“为什么被取消了”的问题。

4. **单元测试**
   建议覆盖：
   - 已 abort 输入的快速路径
   - 超时触发与 cleanup 后超时不再触发
   - 信号触发后定时器是否被清除
   - 两个信号同时触发的幂等性
