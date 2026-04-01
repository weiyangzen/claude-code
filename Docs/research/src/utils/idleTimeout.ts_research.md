# idleTimeout.ts 研究文档

## 场景与职责

`idleTimeout.ts` 是 Claude Code CLI 的 **SDK 模式空闲超时管理器**，用于在 SDK（Software Development Kit）模式下自动管理进程生命周期。当系统处于空闲状态超过指定时间后，自动触发优雅关闭，释放资源并退出进程。

### 核心使用场景

1. **SDK 模式资源管理**：
   - 当 Claude Code 作为 SDK 被其他应用程序调用时
   - 长时间无操作后自动退出，避免资源浪费
   - 防止僵尸进程长期占用系统资源

2. **CI/CD 集成**：
   - 在自动化流程中，确保任务完成后及时清理
   - 避免因异常导致的进程挂起

3. **开发服务器场景**：
   - 临时启动的 Claude Code 实例
   - 空闲后自动关闭，减少开发环境负担

### 环境变量配置

通过 `CLAUDE_CODE_EXIT_AFTER_STOP_DELAY` 环境变量配置超时时间（毫秒）：
```bash
# 设置 5 分钟空闲超时
export CLAUDE_CODE_EXIT_AFTER_STOP_DELAY=300000
```

---

## 功能点目的

### 1. 空闲超时管理器 (`createIdleTimeoutManager`)

**设计目标**：提供一个可控制的空闲计时器，支持启动、停止和自动退出功能。

**核心特性**：
- **条件启动**：仅在配置了有效超时时间时才启动计时器
- **空闲状态检查**：通过传入的 `isIdle` 函数持续检查系统状态
- **连续空闲验证**：确保在计时器触发时，系统确实已连续空闲指定时长
- **优雅关闭**：使用 `gracefulShutdownSync` 确保资源正确释放

### 2. 计时器生命周期管理

**状态流转**：
```
[创建] --> [start()] --> [计时器运行]
              |                |
              |           [超时触发]
              |                |
              |           [检查 isIdle()]
              |                |
              |           [是] --> [gracefulShutdownSync()]
              |                |
              |           [否] --> [重置计时器]
              |                |
           [stop()] <-- [手动停止]
```

---

## 具体技术实现

### 核心实现

```typescript
export function createIdleTimeoutManager(isIdle: () => boolean): {
  start: () => void
  stop: () => void
} {
  // 解析环境变量
  const exitAfterStopDelay = process.env.CLAUDE_CODE_EXIT_AFTER_STOP_DELAY
  const delayMs = exitAfterStopDelay ? parseInt(exitAfterStopDelay, 10) : null
  const isValidDelay = delayMs && !isNaN(delayMs) && delayMs > 0

  let timer: NodeJS.Timeout | null = null
  let lastIdleTime = 0

  return {
    start() {
      // 清除现有计时器
      if (timer) {
        clearTimeout(timer)
        timer = null
      }

      // 仅在有有效延迟时启动
      if (isValidDelay) {
        lastIdleTime = Date.now()

        timer = setTimeout(() => {
          // 验证连续空闲时长
          const idleDuration = Date.now() - lastIdleTime
          if (isIdle() && idleDuration >= delayMs) {
            logForDebugging(`Exiting after ${delayMs}ms of idle time`)
            gracefulShutdownSync()
          }
        }, delayMs)
      }
    },

    stop() {
      if (timer) {
        clearTimeout(timer)
        timer = null
      }
    },
  }
}
```

### 关键设计点

#### 1. 连续空闲验证
```typescript
const idleDuration = Date.now() - lastIdleTime
if (isIdle() && idleDuration >= delayMs) {
  // 确认从 start() 调用到现在一直处于空闲状态
}
```
- 防止在计时器等待期间有活动但 `isIdle()` 未及时更新导致的误判

#### 2. 计时器重置策略
- 每次调用 `start()` 都会重置计时器
- 适合在活动开始时调用，空闲计数重新开始

#### 3. 无效配置处理
```typescript
const isValidDelay = delayMs && !isNaN(delayMs) && delayMs > 0
```
- 环境变量未设置 → 不启动计时器
- 非数字值 → 不启动计时器
- 负数或零 → 不启动计时器

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/idleTimeout.ts`
- **导出函数**：
  - `createIdleTimeoutManager(isIdle)` - 创建空闲超时管理器

### 调用方分布

| 文件路径 | 使用场景 |
|---------|---------|
| `src/cli/print.ts` | SDK 模式下的空闲超时管理 |

### 调用示例（print.ts）

```typescript
import { createIdleTimeoutManager } from '../utils/idleTimeout.js'

// 创建空闲管理器
const idleManager = createIdleTimeoutManager(() => {
  // 检查系统是否空闲
  return !isProcessing && !hasPendingRequests()
})

// 在任务完成时启动计时器
idleManager.start()

// 在有新活动时停止计时器
idleManager.stop()
```

### 依赖导入

```typescript
import { logForDebugging } from './debug.js'              // 调试日志
import { gracefulShutdownSync } from './gracefulShutdown.js'  // 优雅关闭
```

---

## 依赖与外部交互

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `utils/debug.js` | `logForDebugging` | 记录退出原因（调试用） |
| `utils/gracefulShutdown.js` | `gracefulShutdownSync` | 执行进程优雅关闭 |

### 环境变量

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `CLAUDE_CODE_EXIT_AFTER_STOP_DELAY` | `string` | 空闲超时时间（毫秒） |

### 被依赖关系

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `cli/print.ts` | `createIdleTimeoutManager` | SDK 模式空闲管理 |

---

## 风险、边界与改进建议

### 已知风险

1. **`isIdle` 函数可靠性**
   - 空闲状态判断完全依赖传入的 `isIdle` 函数
   - 如果 `isIdle` 实现不正确，可能导致意外退出或永不退出
   - **缓解**：调用方需确保 `isIdle` 准确反映系统状态

2. **进程强制终止风险**
   - `gracefulShutdownSync` 可能因资源清理耗时过长而被强制终止
   - 数据丢失风险
   - **缓解**：`gracefulShutdownSync` 实现了同步清理逻辑，但无法完全避免 SIGKILL

3. **多实例冲突**
   - 如果同一进程创建多个 `idleTimeoutManager` 实例
   - 可能导致重复退出或竞态条件
   - **缓解**：模块设计为单例模式使用，调用方应确保唯一实例

4. **时间精度问题**
   - `setTimeout` 在 Node.js 中的精度约为 4ms（受事件循环影响）
   - 极端情况下可能略有延迟
   - **缓解**：通过 `lastIdleTime` 记录和验证，确保最小空闲时长

### 边界情况

| 场景 | 处理 |
|------|------|
| 环境变量未设置 | 不启动计时器，`start()` 无操作 |
| 环境变量为非数字 | `parseInt` 返回 `NaN`，`isValidDelay` 为 `false`，不启动 |
| 环境变量为负数 | `isValidDelay` 为 `false`，不启动 |
| 环境变量为零 | `isValidDelay` 为 `false`，不启动 |
| 连续调用 `start()` | 清除旧计时器，创建新计时器，重置 `lastIdleTime` |
| 调用 `stop()` 时无运行计时器 | 安全处理，无操作 |
| `isIdle()` 在超时时不为 `true` | 不退出，但计时器已触发（不再重复检查） |

### 改进建议

1. **添加重复检查机制**
   ```typescript
   timer = setTimeout(async () => {
     const idleDuration = Date.now() - lastIdleTime
     if (isIdle() && idleDuration >= delayMs) {
       // 双重检查，等待一小段时间再次确认
       await sleep(100)
       if (isIdle()) {
         logForDebugging(`Exiting after ${delayMs}ms of idle time`)
         gracefulShutdownSync()
       } else {
         // 重新启动计时器
         this.start()
       }
     }
   }, delayMs)
   ```

2. **支持动态配置更新**
   ```typescript
   export function createIdleTimeoutManager(isIdle: () => boolean): {
     start: (delayMs?: number) => void  // 允许运行时指定/覆盖延迟
     stop: () => void
     updateDelay: (delayMs: number) => void  // 热更新延迟
   }
   ```

3. **添加事件通知**
   ```typescript
   export type IdleTimeoutEvents = {
     onTimeoutWarning?: (remainingMs: number) => void  // 超时前警告
     onIdleStart?: () => void  // 空闲开始
     onIdleEnd?: () => void    // 空闲结束（有活动）
   }
   
   export function createIdleTimeoutManager(
     isIdle: () => boolean,
     events?: IdleTimeoutEvents
   ) { /* ... */ }
   ```

4. **支持异步空闲检查**
   ```typescript
   export function createIdleTimeoutManagerAsync(
     isIdle: () => Promise<boolean>
   ): { start: () => void; stop: () => void } {
     // 支持异步空闲检查，适用于复杂状态查询
   }
   ```

5. **添加健康检查**
   ```typescript
   export function createIdleTimeoutManager(isIdle: () => boolean): {
     start: () => void
     stop: () => void
     getStatus: () => { isRunning: boolean; remainingMs: number | null }
   }
   ```

6. **单元测试覆盖**
   - 各种环境变量值的处理
   - `start()`/`stop()` 生命周期
   - 超时触发和空闲验证逻辑
   - 边界时间条件（刚好在阈值上）

7. **文档增强**
   ```typescript
   /**
    * Creates an idle timeout manager for SDK mode.
    * 
    * @param isIdle - Function that returns true if the system is currently idle.
    *                 Should be lightweight as it may be called frequently.
    * @returns Object with start/stop methods to control the idle timer.
    * 
    * @example
    * ```typescript
    * const manager = createIdleTimeoutManager(() => requestQueue.length === 0)
    * manager.start()  // Start monitoring for idle timeout
    * // ... later ...
    * manager.stop()   // Stop monitoring (e.g., when new request arrives)
    * ```
    */
   ```
