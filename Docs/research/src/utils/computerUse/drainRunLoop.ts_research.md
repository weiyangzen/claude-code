# drainRunLoop.ts 研究文档

## 场景与职责

本文件解决 **Node.js/libuv 环境下 Swift `@MainActor` 异步方法挂起** 的核心问题。在 Electron 中，CFRunLoop 持续运行，但在 Node.js/bun（libuv）环境下，DispatchQueue.main 永远不会排空，导致 Swift 异步调用永久挂起。

核心职责：
1. **共享 CFRunLoop 泵**：通过 `setInterval` 定期调用 `_drainMainRunLoop`
2. **引用计数管理**：多个并发调用共享单个泵，安全嵌套
3. **超时保护**：30 秒超时防止永久挂起

## 功能点目的

### 1. RunLoop 泵机制
- 每 1ms 调用 `_drainMainRunLoop`（`RunLoop.main.run`）
- 使用引用计数管理泵的生命周期
- 多个 `drainRunLoop()` 调用共享单个 `setInterval`

### 2. 引用计数 API
- `retainPump()` / `releasePump()` - 用于长期注册（如 CGEventTap）
- `drainRunLoop(fn)` - 用于单次异步调用，自动管理引用

### 3. 超时保护
- 30 秒超时（`TIMEOUT_MS = 30_000`）
- 超时后拒绝，防止永久阻塞
- 孤儿 Promise 处理：附加 no-op catch 防止 unhandledRejection

## 具体技术实现

### 核心变量

```typescript
let pump: ReturnType<typeof setInterval> | undefined
let pending = 0

const TIMEOUT_MS = 30_000
```

### 引用计数逻辑

```typescript
function retain(): void {
  pending++
  if (pump === undefined) {
    pump = setInterval(drainTick, 1, requireComputerUseSwift())
    logForDebugging('[drainRunLoop] pump started', { level: 'verbose' })
  }
}

function release(): void {
  pending--
  if (pending <= 0 && pump !== undefined) {
    clearInterval(pump)
    pump = undefined
    logForDebugging('[drainRunLoop] pump stopped', { level: 'verbose' })
    pending = 0
  }
}
```

### 关键流程

#### `drainRunLoop<T>(fn)` - 主函数

```
输入: fn () => Promise<T>
输出: Promise<T>

流程:
1. retain() 增加引用计数
2. 设置 30 秒超时定时器
3. 执行 fn()，附加 no-op catch 处理孤儿拒绝
4. Promise.race 与超时竞争
5. finally:
   - 清除超时定时器
   - release() 减少引用计数
```

#### 超时处理

```typescript
const work = fn()
work.catch(() => {})  // 吞孤儿拒绝

const timeout = withResolvers<never>()
timer = setTimeout(timeoutReject, TIMEOUT_MS, timeout.reject)

return await Promise.race([work, timeout.promise])
```

**关键设计**：
- 如果超时获胜，fn() 的 Promise 成为孤儿
- 附加 no-op catch 防止后续的拒绝成为 unhandledRejection
- 超时错误是用户看到的错误

## 关键代码路径与文件引用

### 本文件导出
- `drainRunLoop<T>(fn)` - 包装异步调用
- `retainPump()` - 长期保留泵（用于 ESC 热键）
- `releasePump()` - 释放泵保留

### 调用方
- `src/utils/computerUse/executor.ts:319` - `prepareForAction`
- `src/utils/computerUse/executor.ts:380` - `resolvePrepareCapture`
- `src/utils/computerUse/executor.ts:409` - `screenshot`
- `src/utils/computerUse/executor.ts:432` - `zoom`
- `src/utils/computerUse/executor.ts:462` - `key`
- `src/utils/computerUse/executor.ts:490` - `holdKey` (press phase)
- `src/utils/computerUse/executor.ts:505` - `holdKey` (release phase)
- `src/utils/computerUse/executor.ts:513` - `type` (via clipboard)
- `src/utils/computerUse/executor.ts:630` - `listInstalledApps`
- `src/utils/computerUse/escHotkey.ts:34` - `retainPump`
- `src/utils/computerUse/escHotkey.ts:45` - `releasePump`

### 依赖文件
- `src/utils/debug.ts` - `logForDebugging`
- `src/utils/withResolvers.ts` - `withResolvers`
- `src/utils/computerUse/swiftLoader.ts` - `requireComputerUseSwift`

## 依赖与外部交互

### 外部包依赖
- Node.js 定时器 API (`setInterval`, `setTimeout`, `clearTimeout`)

### Swift 运行时交互
- 调用 `@ant/computer-use-swift` 的 `_drainMainRunLoop()`
- 该方法内部执行 `RunLoop.main.run(mode: .default, before: .distantPast)`

### 需要泵的 Swift 方法
根据注释，四个 `@MainActor` 方法需要泵：
1. `captureExcluding` - 截图（排除应用）
2. `captureRegion` - 区域截图
3. `apps.listInstalled` - 列出已安装应用
4. `resolvePrepareCapture` - 解析准备捕获

以及 `@ant/computer-use-input` 的 `key()` / `keys()`

## 风险、边界与改进建议

### 已知风险

1. **孤儿 Promise**：
   - 超时后，原始 Promise 继续运行在后台
   - 如果后续拒绝，需要 no-op catch 防止崩溃
   - 资源泄漏：Swift 操作可能继续执行直到完成

2. **1ms 轮询开销**：
   - 高频轮询可能增加 CPU 使用
   - 当前设置是平衡延迟和开销的折中

3. **嵌套调用**：
   - 虽然引用计数保证安全，但深层嵌套可能难以调试
   - 超时在嵌套调用中累积可能有问题

### 边界情况

1. **同步抛出**：
   - fn() 可能同步抛出（如 NAPI 参数验证失败）
   - 使用 try/finally 确保 release() 被调用

2. **快速完成**：
   - 如果 fn() 在 1ms 内完成，泵可能刚刚启动就停止
   - 这是预期行为，无性能问题

3. **超时后完成**：
   - 超时后 fn() 仍可能成功完成
   - 结果被丢弃，但操作在 Swift 侧已执行

### 改进建议

1. **自适应轮询频率**：
   - 空闲时降低频率（如 10ms）
   - 检测到活动后提高到 1ms
   - 减少空闲 CPU 使用

2. **超时配置**：
   - 允许按操作类型配置不同超时
   - 截图可能需要更长超时（如 60 秒）

3. **取消支持**：
   - 探索向 Swift 传递取消信号的方法
   - 超时后真正终止长时间运行的操作

4. **性能监控**：
   - 记录泵运行时间和频率
   - 监控孤儿 Promise 的数量

5. **替代方案研究**：
   - 评估使用 `uv_poll_t` 或类似机制集成 CFRunLoop
   - 可能消除轮询开销

6. **调试增强**：
   - 添加详细日志记录每次 retain/release
   - 在调试模式下显示当前 pending 计数
