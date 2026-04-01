# escHotkey.ts 研究文档

## 场景与职责

本文件实现 **全局 Escape 键中止功能**，作为 Computer Use 会话的安全机制。这是 Prompt Injection（PI）防御的关键组件——防止恶意提示注入的操作通过 Escape 键关闭对话框。

核心职责：
1. **全局 Escape 监听**：通过 CGEventTap 监听系统范围的 Escape 键
2. **生命周期管理**：在锁获取时注册，锁释放时注销
3. **预期 Escape 通知**：为模型合成的 Escape 键提供"打孔"机制

## 功能点目的

### 1. 全局 Escape → 中止
- 用户按 Escape 时中止当前回合
- 消费 Escape 事件（PI 防御）
- 与 Cowork 的 `escAbort.ts` 功能对等，但无 Electron 依赖

### 2. CGEventTap 集成
- 使用 `@ant/computer-use-swift` 的 `hotkey.registerEscape`
- 在 `CFRunLoopGetMain()` 的 `.defaultMode` 中运行
- 需要 `drainRunLoop` 泵支持（与 `@MainActor` 方法共享）

### 3. 预期 Escape 通知
- `notifyExpectedEscape()` 为模型合成的 Escape 打孔
- 在 `key("escape")` 调用前通知 Swift 层
- Swift 调度 100ms 衰减期，防止误消费后续用户 Escape

## 具体技术实现

### 核心变量

```typescript
let registered = false
```

### 关键流程

#### `registerEscHotkey(onEscape)` - 注册热键

```
输入: onEscape () => void - Escape 回调
输出: boolean - 是否成功注册

流程:
1. 如果已注册，返回 true
2. 获取 Swift 模块
3. 调用 hotkey.registerEscape(onEscape)
4. 如果失败（通常缺少 Accessibility 权限）：
   - 记录警告日志
   - 返回 false（CU 仍工作，只是无 ESC 中止）
5. 成功：
   - retainPump() 保持泵运行
   - registered = true
   - 记录调试日志
   - 返回 true
```

#### `unregisterEscHotkey()` - 注销热键

```
流程:
1. 如果未注册，直接返回
2. try:
   - 调用 hotkey.unregister()
3. finally:
   - releasePump() 释放泵保留
   - registered = false
   - 记录调试日志
```

#### `notifyExpectedEscape()` - 预期 Escape 通知

```
流程:
1. 如果未注册，直接返回
2. 调用 hotkey.notifyExpectedEscape()
```

### 生命周期时序

```
回合开始（首次获取锁）
    ↓
registerEscHotkey(() => abortController.abort())
    ↓
[CU 操作期间...]
    ↓
回合结束 / 中止
    ↓
unregisterEscHotkey()
```

## 关键代码路径与文件引用

### 本文件导出
- `registerEscHotkey(onEscape)` - 注册 Escape 热键
- `unregisterEscHotkey()` - 注销热键
- `notifyExpectedEscape()` - 通知预期 Escape

### 调用方
- `src/utils/computerUse/wrapper.tsx:217` - `registerEscHotkey` 在获取锁时注册
- `src/utils/computerUse/cleanup.ts:73` - `unregisterEscHotkey` 在清理时注销
- `src/utils/computerUse/executor.ts:468` - `notifyExpectedEscape` 在发送 Escape 前通知
- `src/utils/computerUse/executor.ts:496` - `notifyExpectedEscape` 在 holdKey 中通知

### 依赖文件
- `src/utils/debug.ts` - `logForDebugging`
- `src/utils/computerUse/drainRunLoop.ts` - `retainPump`, `releasePump`
- `src/utils/computerUse/swiftLoader.ts` - `requireComputerUseSwift`

## 依赖与外部交互

### 外部包依赖
- `@ant/computer-use-swift` - CGEventTap 封装

### 系统权限
- 需要 **Accessibility 权限**（系统偏好设置 → 安全性与隐私 → 辅助功能）
- 权限缺失时 `registerEscape` 返回 false，CU 仍可用但无 ESC 中止

### CFRunLoop 集成
- CGEventTap 的 `CFRunLoopSource` 在 `CFRunLoopGetMain()` 的 `.defaultMode` 中
- 需要 `drainRunLoop` 泵来处理事件
- `retainPump()` 确保泵在注册期间持续运行

## 风险、边界与改进建议

### 已知风险

1. **Accessibility 权限依赖**：
   - 用户可能未授予权限
   - 缓解：优雅降级，CU 仍可用，只是无 ESC 中止
   - 通知消息根据注册结果显示不同提示

2. **系统范围监听**：
   - Escape 键被全局消费
   - 100ms 衰减期可能不够/过多
   - 可能影响其他应用的 Escape 行为

3. **CGEventTap 失败**：
   - 某些系统配置可能阻止 EventTap 创建
   - 需要监控失败率

### 边界情况

1. **重复注册**：
   - 检查 `registered` 标志防止重复
   - 幂等操作：重复调用返回 true

2. **未注册时注销**：
   - 早期返回，无副作用
   - 用于 cleanup.ts 的防御性调用

3. **注销时抛出**：
   - try/finally 确保 `releasePump` 被调用
   - 防止泵泄漏

4. **快速 Escape 序列**：
   - 模型 Escape → 用户 Escape 在 100ms 内
   - 用户的 Escape 可能被误消费

### 改进建议

1. **权限引导**：
   - 检测权限缺失时显示引导用户授权的 UI
   - 提供直接打开系统偏好设置的按钮

2. **衰减期配置**：
   - 允许通过配置调整 100ms 衰减期
   - 根据用户反馈优化默认值

3. **备用中止机制**：
   - 如果 ESC 不可用，考虑其他组合键（如 Ctrl+Shift+C）
   - 提供多种中止选项

4. **可观测性**：
   - 记录 ESC 按下事件（用于调试）
   - 监控注册/注销的成功率

5. **用户体验**：
   - 在 CU 会话期间显示 ESC 可中止的提示
   - 考虑添加视觉指示器（如状态栏图标）

6. **测试覆盖**：
   - 模拟 Accessibility 权限缺失的场景
   - 测试快速 Escape 序列的处理
   - 验证泵的正确 retain/release
