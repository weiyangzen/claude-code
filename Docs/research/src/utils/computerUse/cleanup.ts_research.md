# cleanup.ts 研究文档

## 场景与职责

本文件负责 **Chicago MCP 会话的回合结束清理工作**，是 Computer Use 功能生命周期管理的关键组件。主要任务包括：

1. **自动取消隐藏应用**：恢复 `prepareForAction` 隐藏的窗口
2. **释放文件锁**：解除进程级别的 Computer Use 锁
3. **注销 ESC 热键**：清理全局 Escape 键监听

清理操作在三个场景被调用：
- 自然回合结束（`stopHooks.ts`）
- 流式响应中止（`query.ts` aborted_streaming）
- 工具执行中止（`query.ts` aborted_tools）

## 功能点目的

### 1. 应用取消隐藏 (`cu.apps.unhide`)
- 恢复回合期间被隐藏的非目标应用窗口
- 使用 5 秒超时保护：防止在 abort 路径上挂起
- 注释说明：`cu.apps.unhide` 不是四个被 `drainRunLoop` 30 秒保护的方法之一

### 2. 文件锁释放 (`releaseComputerUseLock`)
- 零系统调用预检查：`isLockHeldLocally()` 避免非 CU 回合触碰磁盘
- 幂等操作：重复调用返回 false
- 成功释放后发送系统通知 "Claude is done using your computer"

### 3. ESC 热键注销 (`unregisterEscHotkey`)
- 在锁释放前注销，确保 pump-retain 在 CU 会话结束时立即下降
- 幂等操作：注册失败时无操作
- 吞异常：防止 NAPI 注销错误阻止锁释放

## 具体技术实现

### 核心常量

```typescript
// 取消隐藏操作的超时时间
const UNHIDE_TIMEOUT_MS = 5000
```

### 关键流程

#### `cleanupComputerUseAfterTurn()` - 主清理函数

```
输入: ToolUseContext (getAppState, setAppState, sendOSNotification)
输出: Promise<void>

流程:
1. 获取 AppState
2. 检查并恢复隐藏的应用
   - 如果有 hiddenDuringTurn 集合
   - 动态导入 executor.js 获取 unhideComputerUseApps
   - 使用 Promise.race 与 5 秒超时竞争
   - 清除 AppState 中的 hiddenDuringTurn

3. 检查并释放锁
   - 零系统调用预检查: isLockHeldLocally()
   - 注销 ESC 热键（吞异常）
   - 释放文件锁
   - 如果成功释放，发送退出通知
```

### 超时保护机制

```typescript
const unhide = unhideComputerUseApps([...hidden]).catch(err =>
  logForDebugging(`[Computer Use MCP] auto-unhide failed: ${errorMessage(err)}`),
)
const timeout = withResolvers<void>()
const timer = setTimeout(timeout.resolve, UNHIDE_TIMEOUT_MS)
await Promise.race([unhide, timeout.promise]).finally(() => clearTimeout(timer))
```

## 关键代码路径与文件引用

### 本文件导出
- `cleanupComputerUseAfterTurn(ctx)` - 主清理函数

### 调用方
- `src/query/stopHooks.ts:166` - 回合结束清理（主线程）
- `src/query.ts` - 流式响应中止和工具执行中止（动态导入）

### 依赖文件
- `src/utils/computerUse/computerUseLock.ts` - `isLockHeldLocally`, `releaseComputerUseLock`
- `src/utils/computerUse/escHotkey.ts` - `unregisterEscHotkey`
- `src/utils/computerUse/executor.ts` - `unhideComputerUseApps` (动态导入)
- `src/utils/debug.ts` - `logForDebugging`
- `src/utils/errors.ts` - `errorMessage`
- `src/utils/withResolvers.ts` - `withResolvers`

### 依赖类型
- `ToolUseContext` - 来自 `src/Tool.js`

## 依赖与外部交互

### 外部包依赖
- 无直接外部包依赖（动态导入 executor.js）

### 与系统交互
- 文件系统：通过 `computerUseLock.ts` 删除锁文件
- 系统通知：通过 `sendOSNotification` 发送退出通知
- Swift 运行时：通过 `executor.ts` 调用 `cu.apps.unhide`

### 调用时序
```
回合结束 / 中止
    ↓
cleanupComputerUseAfterTurn()
    ↓
├─→ 取消隐藏应用 (5s 超时)
├─→ 注销 ESC 热键
└─→ 释放文件锁 → 发送通知
```

## 风险、边界与改进建议

### 已知风险

1. **Abort 路径挂起风险**：
   - 用户按 Ctrl+C 中止时，如果 unhide 挂起，会阻塞 abort
   - 缓解：5 秒超时确保即使 unhide 失败也能继续

2. **锁释放失败**：
   - 如果 ESC 注销抛出异常，可能阻止锁释放
   - 缓解：try/catch 包裹注销操作，吞异常

3. **并发清理**：
   - 子代理不应调用清理（主线程持有锁）
   - 缓解：`stopHooks.ts` 检查 `!toolUseContext.agentId`

### 边界情况

1. **非 CU 回合**：
   - `isLockHeldLocally()` 返回 false，零开销快速返回
   - 不触碰磁盘，不加载原生模块

2. **重复调用**：
   - 锁释放是幂等的，重复调用返回 false
   - 不会重复发送通知

3. **部分失败**：
   - unhide 失败记录调试日志，不阻止锁释放
   - ESC 注销失败记录警告，不阻止锁释放

### 改进建议

1. **可观测性增强**：
   - 添加清理成功/失败的指标
   - 记录每个步骤的耗时（unhide 时间、锁释放时间）

2. **超时调整**：
   - 考虑根据应用数量动态调整 unhide 超时
   - 添加环境变量覆盖默认 5 秒

3. **优雅降级**：
   - 如果 unhide 持续失败，考虑添加重试机制
   - 在极端情况下，考虑强制终止相关进程

4. **资源泄漏检测**：
   - 添加调试模式检测未释放的 pump-retain
   - 在进程退出时验证所有资源已清理

5. **测试覆盖**：
   - 添加单元测试模拟各种失败场景
   - 测试超时路径和异常处理路径
