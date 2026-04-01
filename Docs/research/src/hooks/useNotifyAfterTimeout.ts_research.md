# useNotifyAfterTimeout.ts 深度研究文档

## 场景与职责

`useNotifyAfterTimeout` 是一个用于在超时后发送桌面通知的 React 钩子。它检测用户是否处于空闲状态（无交互），并在空闲超过阈值后发送通知，提醒用户任务完成或有新消息。

### 核心场景

1. **任务完成通知**：长时间运行的任务完成后通知用户
2. **空闲检测**：基于最后交互时间判断用户是否空闲
3. **避免打扰**：用户活跃时不发送通知

### 与其他组件的关系

- 被各种长时间运行的组件使用
- 与 `useTerminalNotification` 配合发送终端通知
- 与 `bootstrap/state` 的交互时间跟踪配合

---

## 功能点目的

### 1. 交互时间跟踪

- 使用 `getLastInteractionTime()` 获取最后交互时间
- 使用 `updateLastInteractionTime()` 更新交互时间
- 交互包括键盘输入、鼠标操作等

### 2. 空闲检测

默认阈值：6 秒（`DEFAULT_INTERACTION_THRESHOLD_MS = 6000`）

空闲判断：
```typescript
function hasRecentInteraction(threshold: number): boolean {
  return getTimeSinceLastInteraction() < threshold
}
```

### 3. 通知触发

两种触发情况：
1. **立即触发**：如果用户已经空闲超过阈值
2. **延迟触发**：设置定时器，在阈值后检查并发送

### 4. 重置机制

- 钩子挂载时立即重置交互时间
- 防止长时间请求完成后立即发送通知（用户可能一直在等待）

---

## 具体技术实现

### 关键数据结构

```typescript
// 默认交互阈值
export const DEFAULT_INTERACTION_THRESHOLD_MS = 6000  // 6 秒

// 判断是否应该发送通知
function shouldNotify(threshold: number): boolean {
  return process.env.NODE_ENV !== 'test' && !hasRecentInteraction(threshold)
}
```

### 核心实现

```typescript
export function useNotifyAfterTimeout(
  message: string,
  notificationType: string,
): void {
  const terminal = useTerminalNotification()
  
  // 挂载时重置交互时间
  useEffect(() => {
    updateLastInteractionTime(true)  // true = immediate
  }, [])
  
  useEffect(() => {
    let hasNotified = false
    
    const timer = setInterval(() => {
      if (shouldNotify(DEFAULT_INTERACTION_THRESHOLD_MS) && !hasNotified) {
        hasNotified = true
        clearInterval(timer)
        void sendNotification({ message, notificationType }, terminal)
      }
    }, DEFAULT_INTERACTION_THRESHOLD_MS)
    
    return () => clearInterval(timer)
  }, [message, notificationType, terminal])
}
```

### 辅助函数

```typescript
function getTimeSinceLastInteraction(): number {
  return Date.now() - getLastInteractionTime()
}

function hasRecentInteraction(threshold: number): boolean {
  return getTimeSinceLastInteraction() < threshold
}
```

### 交互时间更新

交互时间更新现在由 `App.tsx` 的 `processKeysInBatch` 处理：
```typescript
// 避免单独的 stdin 监听器竞争
processKeysInBatch() {
  // 处理输入...
  updateLastInteractionTime()
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useNotifyAfterTimeout.ts
├── DEFAULT_INTERACTION_THRESHOLD_MS # 行 9: 6 秒阈值
├── getTimeSinceLastInteraction()    # 行 11-13
├── hasRecentInteraction()           # 行 15-17
├── shouldNotify()                   # 行 19-21
├── useNotifyAfterTimeout()          # 行 38-65: 主钩子
│   ├── useTerminalNotification()    # 行 42
│   ├── useEffect - 重置交互时间     # 行 49-51
│   └── useEffect - 通知逻辑         # 行 53-64
```

### 依赖文件

```
src/bootstrap/state.ts
├── getLastInteractionTime()         # 获取最后交互时间
├── updateLastInteractionTime()      # 更新交互时间
└── getIsRemoteMode()                # 远程模式检查

src/ink/useTerminalNotification.ts
└── useTerminalNotification()        # 终端通知 hook

src/services/notifier.ts
└── sendNotification()               # 发送通知
```

---

## 依赖与外部交互

### React Hooks 使用

- `useTerminalNotification`: 获取终端通知能力
- `useEffect`: 重置交互时间和设置通知定时器

### 与状态管理的交互

```typescript
// 重置交互时间
updateLastInteractionTime(true)  // immediate = true

// 检查是否应该通知
if (shouldNotify(DEFAULT_INTERACTION_THRESHOLD_MS)) {
  sendNotification({ message, notificationType }, terminal)
}
```

### 通知系统

```typescript
void sendNotification(
  { message, notificationType },
  terminal
)
```

---

## 风险、边界与改进建议

### 已知风险

1. **测试环境跳过**
   - `NODE_ENV === 'test'` 时不发送通知
   - 可能导致测试覆盖不足

2. **定时器累积**
   - 使用 `setInterval` 可能累积多个定时器
   - 缓解：清理函数和 `hasNotified` 标志

3. **交互检测局限**
   - 只检测键盘输入，不检测鼠标移动
   - 用户可能在阅读但无键盘输入

### 边界情况

| 场景 | 行为 |
|-----|------|
| 测试环境 | 不发送通知 |
| 用户活跃 | 不发送通知，定时器继续检查 |
| 组件卸载 | 清理定时器 |
| 消息变化 | 重新设置定时器 |
| 已经通知 | 不再重复通知 |

### 改进建议

1. **鼠标/触摸检测**
   - 添加鼠标移动和触摸事件监听
   - 更准确地检测用户活跃

2. **智能阈值**
   - 根据任务类型动态调整阈值
   - 例如：编译任务 10 秒，查询任务 5 秒

3. **通知去重**
   - 相同类型通知合并
   - 避免通知轰炸

4. **用户偏好**
   - 允许用户配置通知阈值
   - 支持完全禁用通知

5. **焦点检测**
   - 窗口在前台时不发送通知
   - 使用 Page Visibility API

### 测试建议

1. **单元测试**：
   - 空闲检测逻辑
   - 通知触发条件
   - 定时器清理

2. **集成测试**：
   - 与通知系统集成
   - 长时间运行任务场景

3. **手动测试**：
   - 实际空闲场景
   - 不同终端环境
