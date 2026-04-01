# useFastModeNotification.tsx 深度研究

## 场景与职责

`useFastModeNotification` 是一个 React Hook，用于管理 Fast Mode（快速模式）相关的各种状态通知。Fast Mode 是一种特殊的运行模式，可能受到组织策略、使用限制等多种因素影响。此 Hook 负责在 Fast Mode 状态变化时向用户显示相应的通知。

### 核心场景
1. **组织策略变更通知**：当组织启用或禁用 Fast Mode 时通知用户
2. **使用限制提示**：当 Fast Mode 因使用量达到限制而进入冷却期时通知用户
3. **冷却期管理**：显示冷却开始和冷却结束的通知
4. **超额拒绝处理**：当请求因超额被拒绝时通知用户并自动关闭 Fast Mode

## 功能点目的

### 1. 组织策略变更通知
- 监听组织级别的 Fast Mode 启用状态变化
- 当组织启用 Fast Mode 时提示用户可用（`/fast to turn on`）
- 当组织禁用 Fast Mode 时，如果用户当前正在使用，自动关闭并提示

### 2. 使用限制与冷却期
- 监听冷却期触发事件（`onCooldownTriggered`）
- 显示冷却期开始通知，包含重置时间
- 监听冷却期结束事件（`onCooldownExpired`）
- 显示冷却期结束通知，告知用户 Fast Mode 已恢复

### 3. 超额拒绝处理
- 监听 Fast Mode 超额拒绝事件（`onFastModeOverageRejection`）
- 自动关闭 Fast Mode
- 显示拒绝原因通知

### 4. 通知互斥管理
- 冷却开始通知会使冷却结束通知失效（`invalidates`）
- 冷却结束通知会使冷却开始通知失效
- 避免显示冲突的通知

## 具体技术实现

### 关键数据结构

```typescript
// 冷却原因类型
type CooldownReason = 'overloaded' | 'rate_limit'

// Fast Mode 状态
interface FastModeState {
  fastMode: boolean
}

// 通知键常量
const COOLDOWN_STARTED_KEY = 'fast-mode-cooldown-started'
const COOLDOWN_EXPIRED_KEY = 'fast-mode-cooldown-expired'
const ORG_CHANGED_KEY = 'fast-mode-org-changed'
const OVERAGE_REJECTED_KEY = 'fast-mode-overage-rejected'
```

### 核心流程

#### 1. 组织策略变更处理
```
onOrgFastModeChanged 回调
    ↓
如果 orgEnabled === true
    显示 "Fast mode is now available · /fast to turn on"
否则如果 orgEnabled === false 且当前正在使用 Fast Mode
    自动关闭 Fast Mode (setAppState)
    显示 "Fast mode has been disabled by your organization"
```

#### 2. 冷却期管理
```
onCooldownTriggered 回调
    ↓
格式化重置时间 (formatDuration)
    ↓
根据原因生成消息
    - 'overloaded': "Fast mode overloaded and is temporarily unavailable · resets in {time}"
    - 'rate_limit': "Fast limit reached and temporarily disabled · resets in {time}"
    ↓
显示通知（带 invalidates: [COOLDOWN_EXPIRED_KEY]）

onCooldownExpired 回调
    ↓
显示 "Fast limit reset · now using fast mode"
    （带 invalidates: [COOLDOWN_STARTED_KEY]）
```

#### 3. 超额拒绝处理
```
onFastModeOverageRejection 回调
    ↓
自动关闭 Fast Mode (setAppState)
    ↓
显示拒绝消息通知
```

### 关键代码路径

```typescript
export function useFastModeNotification() {
  const { addNotification } = useNotifications()
  const isFastMode = useAppState(s => s.fastMode)
  const setAppState = useSetAppState()

  // Effect 1: 组织策略变更
  useEffect(() => {
    if (getIsRemoteMode() || !isFastModeEnabled()) return
    
    return onOrgFastModeChanged(orgEnabled => {
      if (orgEnabled) {
        addNotification({
          key: ORG_CHANGED_KEY,
          color: "fastMode",
          priority: "immediate",
          text: "Fast mode is now available · /fast to turn on"
        })
      } else if (isFastMode) {
        setAppState(prev => ({ ...prev, fastMode: false }))
        addNotification({
          key: ORG_CHANGED_KEY,
          color: "warning",
          priority: "immediate",
          text: "Fast mode has been disabled by your organization"
        })
      }
    })
  }, [addNotification, isFastMode, setAppState])

  // Effect 2: 超额拒绝
  useEffect(() => {
    if (getIsRemoteMode() || !isFastModeEnabled()) return
    
    return onFastModeOverageRejection(message => {
      setAppState(prev => ({ ...prev, fastMode: false }))
      addNotification({
        key: OVERAGE_REJECTED_KEY,
        color: "warning",
        priority: "immediate",
        text: message
      })
    })
  }, [addNotification, setAppState])

  // Effect 3: 冷却期管理
  useEffect(() => {
    if (getIsRemoteMode() || !isFastMode) return
    
    const unsubTriggered = onCooldownTriggered((resetAt, reason) => {
      const resetIn = formatDuration(resetAt - Date.now(), { hideTrailingZeros: true })
      addNotification({
        key: COOLDOWN_STARTED_KEY,
        invalidates: [COOLDOWN_EXPIRED_KEY],
        text: getCooldownMessage(reason, resetIn),
        color: "warning",
        priority: "immediate"
      })
    })
    
    const unsubExpired = onCooldownExpired(() => {
      addNotification({
        key: COOLDOWN_EXPIRED_KEY,
        invalidates: [COOLDOWN_STARTED_KEY],
        color: "fastMode",
        text: "Fast limit reset · now using fast mode",
        priority: "immediate"
      })
    })
    
    return () => {
      unsubTriggered()
      unsubExpired()
    }
  }, [addNotification, isFastMode])
}

function getCooldownMessage(reason: CooldownReason, resetIn: string): string {
  switch (reason) {
    case 'overloaded':
      return `Fast mode overloaded and is temporarily unavailable · resets in ${resetIn}`
    case 'rate_limit':
      return `Fast limit reached and temporarily disabled · resets in ${resetIn}`
  }
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `useEffect` | `react` | React Hook API |
| `useNotifications` | `src/context/notifications.js` | 通知系统 |
| `useAppState`, `useSetAppState` | `src/state/AppState.js` | 应用状态访问和修改 |
| `CooldownReason`, `isFastModeEnabled`, `onCooldownExpired`, `onCooldownTriggered`, `onFastModeOverageRejection`, `onOrgFastModeChanged` | `src/utils/fastMode.js` | Fast Mode 工具函数和事件监听 |
| `formatDuration` | `src/utils/format.js` | 时间格式化 |
| `getIsRemoteMode` | `src/bootstrap/state.js` | 远程模式检测 |

### 依赖模块详解

#### 1. Fast Mode 工具 (src/utils/fastMode.js)
提供 Fast Mode 相关的核心功能：
- `isFastModeEnabled()`: 检查 Fast Mode 是否启用
- `onOrgFastModeChanged(callback)`: 监听组织策略变更
- `onFastModeOverageRejection(callback)`: 监听超额拒绝事件
- `onCooldownTriggered(callback)`: 监听冷却期触发
- `onCooldownExpired(callback)`: 监听冷却期结束

#### 2. formatDuration (src/utils/format.ts)
格式化时间持续时间为人类可读的字符串：
```typescript
export function formatDuration(
  ms: number,
  options?: { hideTrailingZeros?: boolean; mostSignificantOnly?: boolean }
): string
```

#### 3. useAppState / useSetAppState (src/state/AppState.ts)
全局状态管理：
- `useAppState`: 读取状态（如 `s.fastMode`）
- `useSetAppState`: 更新状态（如关闭 Fast Mode）

## 风险、边界与改进建议

### 潜在风险

1. **事件监听泄漏**
   - 使用多个 `useEffect`，每个都返回清理函数
   - 如果组件频繁挂载/卸载，可能导致事件重复订阅
   - 当前实现已正确处理清理，但需要确保回调函数稳定性

2. **状态更新竞态**
   - `onOrgFastModeChanged` 和 `onFastModeOverageRejection` 都可能修改 `fastMode` 状态
   - 如果两个事件同时触发，可能导致状态不一致
   - 建议：使用函数式状态更新确保一致性

3. **时间格式化精度**
   - `formatDuration` 使用 `Date.now()`，可能与服务器时间有偏差
   - 如果客户端时钟不准确，显示的重置时间可能有误

### 边界情况

1. **远程模式**
   - 所有效果都在远程模式下禁用
   - 这是预期行为，因为远程模式通常由管理端控制

2. **Fast Mode 未启用**
   - 如果 `isFastModeEnabled()` 返回 false，不监听组织变更和超额拒绝
   - 但冷却期监听仍然依赖于 `isFastMode` 状态

3. **冷却期重叠**
   - 如果新的冷却期在旧冷却期结束前开始，使用 `invalidates` 确保通知更新

### 改进建议

1. **增加防抖处理**
   ```typescript
   // 对于可能频繁触发的事件，考虑添加防抖
   const debouncedAddNotification = useMemo(
     () => debounce(addNotification, 100),
     [addNotification]
   )
   ```

2. **添加分析事件**
   ```typescript
   // 记录 Fast Mode 状态变化
   logEvent('tengu_fast_mode_cooldown_triggered', { reason })
   logEvent('tengu_fast_mode_org_changed', { enabled: orgEnabled })
   ```

3. **优化时间显示**
   ```typescript
   // 考虑添加倒计时更新
   const [timeLeft, setTimeLeft] = useState(resetAt - Date.now())
   useInterval(() => setTimeLeft(resetAt - Date.now()), 60000) // 每分钟更新
   ```

4. **增加错误处理**
   ```typescript
   try {
     const resetIn = formatDuration(resetAt - Date.now(), { hideTrailingZeros: true })
   } catch (error) {
     logError(error)
     return 'soon' // 降级显示
   }
   ```

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useFastModeNotification.tsx`
- **通知系统**: `src/context/notifications.tsx`
- **Fast Mode 工具**: `src/utils/fastMode.js`
- **时间格式化**: `src/utils/format.ts`
- **应用状态**: `src/state/AppState.ts`
- **启动状态**: `src/bootstrap/state.ts`
