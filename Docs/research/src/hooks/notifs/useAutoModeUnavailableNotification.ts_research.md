# useAutoModeUnavailableNotification.ts 深度研究

## 场景与职责

`useAutoModeUnavailableNotification` 是一个 React Hook，用于在 Auto Mode（自动模式）不可用时向用户显示一次性通知。当用户通过 Shift+Tab 在权限模式轮播中切换时，如果经过了 Auto Mode 位置但 Auto Mode 当前不可用，此 Hook 会触发通知提示用户。

### 核心场景
1. **模式轮播提示**：用户在 Shift+Tab 切换模式（如 default → auto → acceptEdits）时，如果 auto 模式被禁用或不可用，需要告知用户原因
2. **启动时静默降级处理**：与 `verifyAutoModeGateAccess` 和 `checkAndDisableAutoModeIfNeeded` 配合，处理启动时的默认模式降级情况
3. **多原因覆盖**：涵盖设置禁用、熔断器触发、组织白名单限制等所有导致 Auto Mode 不可用的原因

## 功能点目的

### 1. 一次性通知机制
- 使用 `shownRef` 确保每个会话只显示一次通知，避免重复打扰用户
- 在远程模式 (`getIsRemoteMode()`) 下禁用通知

### 2. 智能触发条件
通知仅在以下所有条件满足时触发：
- 当前模式为 `'default'`
- 上一个模式不是 `'default'` 且不是 `'auto'`（表示用户从其他模式切换回来，经过了 auto 位置）
- Auto Mode 当前不可用 (`!isAutoModeAvailable`)
- 用户已选择加入 Auto Mode (`hasAutoModeOptIn()`)
- 功能开关 `TRANSCRIPT_CLASSIFIER` 已启用

### 3. 原因获取与展示
通过 `getAutoModeUnavailableReason()` 获取具体原因，并生成对应的通知文本。

## 具体技术实现

### 关键数据结构

```typescript
// 权限模式类型
interface PermissionMode {
  mode: 'default' | 'auto' | 'acceptEdits' | 'plan' | 'dontAsk' | 'bypassPermissions'
}

// 工具权限上下文
interface ToolPermissionContext {
  mode: PermissionMode
  isAutoModeAvailable: boolean
}
```

### 核心流程

```
useEffect 触发
    ↓
检查 TRANSCRIPT_CLASSIFIER 功能开关
    ↓
检查是否为远程模式 (getIsRemoteMode)
    ↓
检查是否已显示过通知 (shownRef.current)
    ↓
判断模式切换条件：
    - 当前模式 === 'default'
    - 上一模式 !== 'default' && 上一模式 !== 'auto'
    - !isAutoModeAvailable
    - hasAutoModeOptIn()
    ↓
获取不可用原因 (getAutoModeUnavailableReason)
    ↓
添加通知 (addNotification)
```

### 关键代码路径

```typescript
// 模式切换检测逻辑
const wrappedPastAutoSlot =
  mode === 'default' &&
  prevMode !== 'default' &&
  prevMode !== 'auto' &&
  !isAutoModeAvailable &&
  hasAutoModeOptIn()

// 通知添加
addNotification({
  key: 'auto-mode-unavailable',
  text: getAutoModeUnavailableNotification(reason),
  color: 'warning',
  priority: 'medium',
})
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `feature` | `bun:bundle` | 功能开关检查 |
| `useEffect`, `useRef` | `react` | React Hook API |
| `useNotifications` | `src/context/notifications.js` | 通知系统 |
| `getIsRemoteMode` | `src/bootstrap/state.js` | 远程模式检测 |
| `useAppState` | `src/state/AppState.js` | 应用状态访问 |
| `PermissionMode` | `src/utils/permissions/PermissionMode.js` | 权限模式类型 |
| `getAutoModeUnavailableNotification`, `getAutoModeUnavailableReason` | `src/utils/permissions/permissionSetup.js` | 原因获取与通知文本生成 |
| `hasAutoModeOptIn` | `src/utils/settings/settings.js` | 检查用户是否选择加入 Auto Mode |

### 依赖模块详解

#### 1. useNotifications (src/context/notifications.tsx)
提供通知系统的核心功能：
- `addNotification`: 添加通知到队列
- `removeNotification`: 移除通知
- 支持优先级队列（immediate > high > medium > low）
- 支持通知超时和自动清除

#### 2. useAppState (src/state/AppState.ts)
全局应用状态管理：
- `toolPermissionContext.mode`: 当前权限模式
- `toolPermissionContext.isAutoModeAvailable`: Auto Mode 可用性状态

#### 3. hasAutoModeOptIn (src/utils/settings/settings.ts)
检查用户是否已选择加入 Auto Mode：
```typescript
export function hasAutoModeOptIn(): boolean {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const user = getSettingsForSource('userSettings')?.skipAutoPermissionPrompt
    const local = getSettingsForSource('localSettings')?.skipAutoPermissionPrompt
    const flag = getSettingsForSource('flagSettings')?.skipAutoPermissionPrompt
    const policy = getSettingsForSource('policySettings')?.skipAutoPermissionPrompt
    return !!(user || local || flag || policy)
  }
  return false
}
```

## 风险、边界与改进建议

### 潜在风险

1. **竞态条件**
   - `shownRef` 在组件生命周期内保持，如果组件被卸载后重新挂载，可能再次显示通知
   - 建议：考虑将 shown 状态持久化到全局状态或 sessionStorage

2. **模式切换误判**
   - 当前逻辑假设模式切换是按顺序进行的，如果用户快速切换或存在异常状态，可能误判
   - 建议：增加更精确的模式切换追踪

3. **依赖功能开关**
   - 整个 Hook 的行为完全依赖 `TRANSCRIPT_CLASSIFIER` 功能开关
   - 如果开关配置错误，用户可能看不到重要的可用性提示

### 边界情况

1. **远程模式**
   - 在远程模式下完全禁用通知，这是预期行为

2. **快速模式切换**
   - 如果用户在极短时间内多次切换模式，`prevModeRef` 可能无法准确追踪

3. **原因获取失败**
   - 如果 `getAutoModeUnavailableReason()` 返回 null，通知不会显示

### 改进建议

1. **增加日志记录**
   ```typescript
   // 建议添加调试日志
   logForDebugging(`[AutoModeUnavailable] wrappedPastAutoSlot=${wrappedPastAutoSlot}, reason=${reason}`)
   ```

2. **考虑使用更持久的状态**
   - 对于跨会话的提示，考虑使用 `getGlobalConfig()` 存储已显示状态

3. **增强原因展示**
   - 当前只显示文本，可以考虑添加可操作的建议（如"/settings 修改配置"）

4. **单元测试覆盖**
   - 建议添加测试覆盖：
     - 正常触发场景
     - 远程模式禁用
     - 已显示后不再显示
     - 不同原因的通知文本

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useAutoModeUnavailableNotification.ts`
- **通知系统**: `src/context/notifications.tsx`
- **权限模式**: `src/utils/permissions/PermissionMode.ts`
- **设置管理**: `src/utils/settings/settings.ts`
- **应用状态**: `src/state/AppState.ts`
- **启动通知基类**: `src/hooks/notifs/useStartupNotification.ts`
