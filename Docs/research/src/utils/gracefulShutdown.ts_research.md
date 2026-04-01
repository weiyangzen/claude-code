# gracefulShutdown.ts 深度研究

## 场景与职责

本模块是 Claude Code 的优雅关闭核心控制器，负责在进程退出前执行必要的清理工作，确保终端状态恢复、数据持久化、分析事件发送等关键操作完成。它是系统稳定性的最后一道防线。

**核心场景：**
1. **用户主动退出**：SIGINT (Ctrl+C)、SIGTERM、SIGHUP 信号处理
2. **终端断开检测**：macOS 终端关闭时的孤儿进程检测
3. **异常退出**：未捕获异常和未处理 Promise 拒绝的日志记录
4. **强制退出保障**：防止清理操作挂起导致进程无法退出

## 功能点目的

### 1. 终端状态恢复
- **目的**：退出时恢复终端到正常状态
- **内容**：
  - 退出 alt screen（备用屏幕缓冲区）
  - 禁用鼠标追踪、焦点事件、括号粘贴模式
  - 禁用 Kitty 键盘扩展、修改其他键
  - 显示光标
  - 清除 iTerm2 进度条、标签状态、终端标题

### 2. 会话恢复提示
- **目的**：告知用户如何恢复当前会话
- **条件**：交互式会话且启用了会话持久化
- **格式**：`claude --resume <session-id>`

### 3. 清理操作执行
- **目的**：执行注册的清理函数（如会话数据保存）
- **超时**：2 秒超时，防止阻塞

### 4. SessionEnd Hooks 执行
- **目的**：执行用户配置的会话结束钩子
- **预算**：可配置的 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`（默认 1.5s）

### 5. 分析数据刷新
- **目的**：发送未完成的分析事件到 Datadog 和 1P 日志
- **超时**：500ms 上限，避免阻塞退出

### 6. 孤儿进程检测（macOS）
- **目的**：检测终端关闭但未收到 SIGHUP 的情况
- **机制**：每 30 秒检查 stdin/stdout 的可读写状态

## 具体技术实现

### 核心数据结构
```typescript
// 模块级状态
let shutdownInProgress = false
let failsafeTimer: ReturnType<typeof setTimeout> | undefined
let orphanCheckInterval: ReturnType<typeof setInterval> | undefined
let pendingShutdown: Promise<void> | undefined
let resumeHintPrinted = false
let currentTurnNumber = -1
```

### 关键流程

#### setupGracefulShutdown() - 初始化
```typescript
export const setupGracefulShutdown = memoize(() => {
  // 1. 固定 signal-exit v4，防止 Bun bug 导致处理器被移除
  onExit(() => {})
  
  // 2. 注册信号处理器
  process.on('SIGINT', () => { /* ... */ })
  process.on('SIGTERM', () => { /* ... */ })
  process.on('SIGHUP', () => { /* ... */ })
  
  // 3. 注册异常处理器
  process.on('uncaughtException', error => { /* ... */ })
  process.on('unhandledRejection', reason => { /* ... */ })
  
  // 4. 启动孤儿检测（macOS 非 Windows）
  if (process.platform !== 'win32' && process.stdin.isTTY) {
    orphanCheckInterval = setInterval(() => { /* ... */ }, 30_000)
  }
})
```

#### gracefulShutdown() - 异步关闭
```
1. 检查是否已在关闭中
2. 解析 SessionEnd hook 超时预算
3. 设置 failsafe 定时器（max(5s, hook预算 + 3.5s)）
4. 设置 exit code
5. 清理终端模式 + 打印恢复提示（优先执行）
6. 执行清理函数（2s 超时）
7. 执行 SessionEnd hooks
8. 记录启动性能报告
9. 发送缓存驱逐提示事件
10. 刷新分析数据（500ms 超时）
11. 打印最终消息（如有）
12. 强制退出
```

#### cleanupTerminalModes() - 终端恢复
**关键设计决策：**
- 使用 `writeSync` 确保在进程退出前完成写入
- 无条件发送所有禁用序列（不同终端支持不同）
- 优先禁用鼠标追踪（给终端处理时间）
- 调用 Ink 的 `unmount()` 而非直接写 `EXIT_ALT_SCREEN`

#### forceExit() - 强制退出
```typescript
function forceExit(exitCode: number): never {
  // 1. 清除 failsafe 定时器
  // 2. 最后 drain stdin
  // 3. 尝试 process.exit()
  // 4. 如失败（EIO），使用 SIGKILL
}
```

### failsafe 机制
```typescript
failsafeTimer = setTimeout(
  code => {
    cleanupTerminalModes()
    printResumeHint()
    forceExit(code)
  },
  Math.max(5000, sessionEndTimeoutMs + 3500),
  exitCode,
)
```

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `setupGracefulShutdown` | 函数 | 初始化关闭处理器（幂等） |
| `gracefulShutdown` | 函数 | 异步关闭入口 |
| `gracefulShutdownSync` | 函数 | 同步触发关闭（设置 exitCode） |
| `isShuttingDown` | 函数 | 检查关闭状态 |
| `resetShutdownState` | 函数 | 测试用：重置状态 |
| `getPendingShutdownForTesting` | 函数 | 测试用：获取 pending promise |

### 调用方分布
1. **main.tsx**: 启动时调用 `setupGracefulShutdown()`
2. **REPL.tsx**: 退出命令、错误处理
3. **print.ts**: 打印模式完成时
4. **exit.tsx**: `/exit` 命令
5. **logout.tsx**: `/logout` 命令
6. **autoUpdater.ts**: 更新前关闭
7. **idleTimeout.ts**: 空闲超时
8. **useQueueProcessor.ts**: 队列处理器

### 依赖模块
```typescript
import chalk from 'chalk'
import { writeSync } from 'fs'
import memoize from 'lodash-es/memoize.js'
import { onExit } from 'signal-exit'
import { getIsInteractive, getIsScrollDraining, getSessionId, isSessionPersistenceDisabled } from '../bootstrap/state.js'
import instances from '../ink/instances.js'
import { DISABLE_KITTY_KEYBOARD, DISABLE_MODIFY_OTHER_KEYS } from '../ink/termio/csi.js'
import { DBP, DFE, DISABLE_MOUSE_TRACKING, EXIT_ALT_SCREEN, SHOW_CURSOR } from '../ink/termio/dec.js'
import { CLEAR_ITERM2_PROGRESS, CLEAR_TAB_STATUS, CLEAR_TERMINAL_TITLE, supportsTabStatus, wrapForMultiplexer } from '../ink/termio/osc.js'
import { shutdownDatadog } from '../services/analytics/datadog.js'
import { shutdown1PEventLogging } from '../services/analytics/firstPartyEventLogger.js'
import { logEvent } from '../services/analytics/index.js'
import type { AppState } from '../state/AppState.js'
import { runCleanupFunctions } from './cleanupRegistry.js'
import { logForDebugging } from './debug.js'
import { logForDiagnosticsNoPII } from './diagLogs.js'
import { isEnvTruthy } from './envUtils.js'
import { getCurrentSessionTitle, sessionIdExists } from './sessionStorage.js'
import { sleep } from './sleep.js'
import { profileReport } from './startupProfiler.js'
```

## 依赖与外部交互

### 上游依赖

1. **signal-exit**: 跨平台信号处理库
   - 处理 Bun bug：防止 `removeListener` 重置内核信号处理器
   - 固定 v4 加载状态

2. **ink/instances.ts**: Ink 渲染实例管理
   - `instances.get(process.stdout)`: 获取当前 Ink 实例
   - `unmount()`: 卸载 React 组件树
   - `drainStdin()`: 清空输入缓冲区
   - `detachForShutdown()`: 标记实例为已卸载

3. **ink/termio/**: 终端控制序列
   - `csi.js`: Kitty 键盘、修改其他键禁用
   - `dec.js`: 鼠标追踪、焦点事件、alt screen、光标
   - `osc.js`: iTerm2 进度、标签状态、终端标题

4. **services/analytics/**: 分析服务
   - `shutdownDatadog()`: 刷新 Datadog 事件
   - `shutdown1PEventLogging()`: 刷新 1P 日志

5. **cleanupRegistry.ts**: 清理函数注册表
   - `runCleanupFunctions()`: 执行所有注册的清理函数

### 下游影响

1. **终端状态**：确保用户终端不会处于损坏状态
2. **会话恢复**：提示信息帮助用户恢复工作
3. **数据持久化**：确保会话数据、分析数据不丢失
4. **Hook 执行**：用户自定义清理逻辑

## 风险、边界与改进建议

### 已知风险

1. **Bun process.exit() EIO 错误**
   - 风险：终端已关闭时 `process.exit()` 抛出 EIO
   - 缓解：`forceExit` 捕获并回退到 `SIGKILL`

2. **清理函数挂起**
   - 风险：注册的清理函数可能死锁或长时间阻塞
   - 缓解：2 秒超时 + failsafe 定时器

3. **信号竞争**
   - 风险：SIGINT 和 SIGTERM 同时到达
   - 缓解：`shutdownInProgress` 标志防止重复执行

4. **孤儿检测误报**
   - 风险：滚动时 stdin 检查可能误判
   - 缓解：`getIsScrollDraining()` 跳过检查

5. **测试环境特殊性**
   - 风险：`process.exit` 在测试中可能被 mock 为返回而非退出
   - 缓解：测试模式特殊处理

### 边界情况

1. **非 TTY 环境**：跳过终端恢复操作
2. **非交互式会话**：跳过恢复提示
3. **无会话文件**：跳过恢复提示（如 `claude update`）
4. **SessionEnd hook 超时**：`AbortSignal.timeout` 自动中止
5. **分析刷新超时**：500ms 后放弃，优先保证退出

### 改进建议

1. **分级关闭策略**
   - 建议：区分"快速退出"和"完整清理"
   - 场景：SIGTERM 时快速退出，SIGINT 时完整清理

2. **清理函数优先级**
   - 建议：允许注册时指定优先级
   - 收益：关键清理（如数据保存）优先执行

3. **关闭进度反馈**
   - 建议：长时间清理时显示进度
   - 场景：大量 SessionEnd hooks 时用户可见

4. **优雅关闭超时配置**
   - 建议：用户可配置最大等待时间
   - 实现：环境变量或配置选项

5. **关闭原因追踪**
   - 建议：记录关闭触发原因（信号、异常、正常退出）
   - 收益：更好的故障分析

6. **跨平台孤儿检测**
   - 建议：Windows 支持（如可行）
   - 挑战：Windows 信号模型差异

### 测试要点

1. 各信号触发关闭的正确性
2. 并发关闭请求的幂等性
3. 清理超时的正确处理
4. 终端序列发送顺序
5. 非 TTY 环境的行为
6. 孤儿检测的准确性
