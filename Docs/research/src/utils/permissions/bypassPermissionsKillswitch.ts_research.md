# bypassPermissionsKillswitch.ts 深度研究

## 场景与职责

`bypassPermissionsKillswitch.ts` 是 Claude Code 权限系统的**安全门控模块**，负责在运行时动态检查和禁用 `bypassPermissions` 模式和 `auto` 模式。该模块实现了两种安全机制：

1. **Bypass Permissions Killswitch**: 通过 Statsig gate 检查是否应该禁用 bypass 权限模式
2. **Auto Mode Gate**: 通过 GrowthBook 检查自动模式是否可用

### 核心职责
1. **运行时安全检查**: 在首次查询前执行一次性的安全检查
2. **状态转换**: 根据检查结果更新应用状态（禁用 bypass 或 auto 模式）
3. **React 集成**: 提供自定义 Hook 用于在组件中触发检查
4. **重置机制**: 支持在 `/login` 后重新运行检查（新组织可能不同策略）

### 设计背景
这些检查必须在**首次查询前**完成，以确保使用最新的 gate 值。同时，检查只运行一次（run-once），避免重复的网络请求。

---

## 功能点目的

### 1. Bypass Permissions 检查

#### `checkAndDisableBypassPermissionsIfNeeded`
核心检查函数：
```typescript
export async function checkAndDisableBypassPermissionsIfNeeded(
  toolPermissionContext: ToolPermissionContext,
  setAppState: (f: (prev: AppState) => AppState) => void,
): Promise<void>
```

流程：
1. 检查是否已运行过（`bypassPermissionsCheckRan`）
2. 检查 bypass 模式是否可用
3. 异步检查 Statsig gate (`shouldDisableBypassPermissions`)
4. 如需要禁用，更新应用状态为禁用 bypass 的上下文

#### `resetBypassPermissionsCheck`
重置运行标志，用于 `/login` 后：
```typescript
export function resetBypassPermissionsCheck(): void
```

#### `useKickOffCheckAndDisableBypassPermissionsIfNeeded`
React Hook，在组件挂载时触发检查：
```typescript
export function useKickOffCheckAndDisableBypassPermissionsIfNeeded(): void
```

### 2. Auto Mode 检查

#### `checkAndDisableAutoModeIfNeeded`
自动模式门控检查：
```typescript
export async function checkAndDisableAutoModeIfNeeded(
  toolPermissionContext: ToolPermissionContext,
  setAppState: (f: (prev: AppState) => AppState) => void,
  fastMode?: boolean,
): Promise<void>
```

流程：
1. 检查 `TRANSCRIPT_CLASSIFIER` 功能标志
2. 调用 `verifyAutoModeGateAccess` 进行完整验证
3. 更新应用状态和通知队列
4. 处理异步 GrowthBook 等待期间的并发状态变更

#### `resetAutoModeGateCheck`
重置自动模式检查标志：
```typescript
export function resetAutoModeGateCheck(): void
```

#### `useKickOffCheckAndDisableAutoModeIfNeeded`
React Hook，监听模式变化：
```typescript
export function useKickOffCheckAndDisableAutoModeIfNeeded(): void
```

监听：
- `mainLoopModel` - 主循环模型变化
- `mainLoopModelForSession` - 会话模型变化
- `fastMode` - 快速模式变化

---

## 具体技术实现

### 运行一次模式 (Run-Once Pattern)

```typescript
let bypassPermissionsCheckRan = false

export async function checkAndDisableBypassPermissionsIfNeeded(...): Promise<void> {
  if (bypassPermissionsCheckRan) {
    return  // 已运行过，直接返回
  }
  bypassPermissionsCheckRan = true
  // ... 执行检查
}
```

### 状态更新模式

```typescript
setAppState(prev => {
  return {
    ...prev,
    toolPermissionContext: createDisabledBypassPermissionsContext(
      prev.toolPermissionContext,
    ),
  }
})
```

### 异步检查与并发处理

```typescript
setAppState(prev => {
  // 关键：使用当前状态（prev）而非传入的 stale snapshot
  const nextCtx = updateContext(prev.toolPermissionContext)
  
  // 如果上下文未变化，保持原状态（优化）
  const newState =
    nextCtx === prev.toolPermissionContext
      ? prev
      : { ...prev, toolPermissionContext: nextCtx }
  
  // 添加通知（如果有）
  if (!notification) return newState
  return {
    ...newState,
    notifications: {
      ...newState.notifications,
      queue: [...newState.notifications.queue, { /* ... */ }],
    },
  }
})
```

### React Hook 实现

```typescript
export function useKickOffCheckAndDisableAutoModeIfNeeded(): void {
  const mainLoopModel = useAppState(s => s.mainLoopModel)
  const mainLoopModelForSession = useAppState(s => s.mainLoopModelForSession)
  const fastMode = useAppState(s => s.fastMode)
  const setAppState = useSetAppState()
  const store = useAppStateStore()
  const isFirstRunRef = useRef(true)

  useEffect(() => {
    if (getIsRemoteMode()) return  // 远程模式跳过
    
    if (isFirstRunRef.current) {
      isFirstRunRef.current = false
    } else {
      resetAutoModeGateCheck()  // 非首次运行，重置检查
    }
    
    void checkAndDisableAutoModeIfNeeded(
      store.getState().toolPermissionContext,
      setAppState,
      fastMode,
    )
  }, [mainLoopModel, mainLoopModelForSession, fastMode])
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `AppState`, `useAppState`, `useSetAppState`, `useAppStateStore` | `src/state/AppState.js` | 应用状态管理 |
| `ToolPermissionContext` | `src/Tool.js` | 权限上下文类型 |
| `getIsRemoteMode` | `src/bootstrap/state.js` | 检查远程模式 |
| `createDisabledBypassPermissionsContext`, `shouldDisableBypassPermissions`, `verifyAutoModeGateAccess` | `src/utils/permissions/permissionSetup.ts` | 权限设置函数 |

### 调用方

| 调用方 | 路径 | 场景 |
|--------|------|------|
| REPL.tsx | `src/screens/REPL.tsx` | 主 REPL 界面挂载时触发检查 |
| 其他组件 | 权限相关组件 | 需要时触发检查 |

### 相关模块

| 模块 | 路径 | 关系 |
|------|------|------|
| `permissionSetup.ts` | `src/utils/permissions/permissionSetup.ts` | 提供底层检查逻辑 |
| `autoModeState.ts` | `src/utils/permissions/autoModeState.ts` | 自动模式状态管理 |

---

## 依赖与外部交互

### 与 Statsig 的交互

```typescript
const shouldDisable = await shouldDisableBypassPermissions()
// 内部调用 Statsig gate: 'tengu_disable_bypass_permissions_mode'
```

### 与 GrowthBook 的交互

```typescript
const { updateContext, notification } = await verifyAutoModeGateAccess(
  toolPermissionContext,
  fastMode,
)
// 内部检查 GrowthBook config: 'tengu_auto_mode_config'
```

### 与应用状态的交互

```typescript
// 更新权限上下文
setAppState(prev => ({
  ...prev,
  toolPermissionContext: createDisabledBypassPermissionsContext(prev.toolPermissionContext),
}))

// 添加通知
setAppState(prev => ({
  ...prev,
  notifications: {
    ...prev.notifications,
    queue: [
      ...prev.notifications.queue,
      {
        key: 'auto-mode-gate-notification',
        text: notification,
        color: 'warning',
        priority: 'high',
      },
    ],
  },
}))
```

---

## 风险、边界与改进建议

### 安全风险

1. **竞态条件**:
   - `verifyAutoModeGateAccess` 是异步的
   - 如果用户在检查期间切换模式，可能导致状态不一致
   - 代码通过使用 `prev` 状态而非 stale snapshot 来缓解

2. **检查绕过**:
   - 如果 `checkAndDisableBypassPermissionsIfNeeded` 未被调用，bypass 模式可能保持启用
   - 需要确保所有入口点都调用检查

3. **网络依赖**:
   - 检查依赖 Statsig/GrowthBook 网络请求
   - 如果网络失败，可能使用缓存值（可能过期）

### 边界情况

1. **重复调用**:
   ```typescript
   // 第一次调用执行检查
   await checkAndDisableBypassPermissionsIfNeeded(...)
   // 第二次调用立即返回（已运行过）
   await checkAndDisableBypassPermissionsIfNeeded(...)
   ```

2. **并发调用**:
   ```typescript
   // 两个并发调用
   Promise.all([
     checkAndDisableBypassPermissionsIfNeeded(...),
     checkAndDisableBypassPermissionsIfNeeded(...),
   ])
   // 可能都通过检查，导致重复执行
   ```

3. **重置时机**:
   ```typescript
   // 在检查进行中重置
   checkAndDisableBypassPermissionsIfNeeded(...) // 开始检查
   resetBypassPermissionsCheck()                  // 重置标志
   checkAndDisableBypassPermissionsIfNeeded(...) // 再次执行检查
   ```

4. **远程模式**:
   ```typescript
   if (getIsRemoteMode()) return  // 远程模式跳过检查
   // 远程模式下 bypass 模式的行为?
   ```

### 改进建议

1. **原子性保证**:
   ```typescript
   let bypassPermissionsCheckPromise: Promise<void> | null = null
   
   export async function checkAndDisableBypassPermissionsIfNeeded(...): Promise<void> {
     if (bypassPermissionsCheckRan) return
     
     // 确保只有一个检查在执行
     if (bypassPermissionsCheckPromise) {
       return bypassPermissionsCheckPromise
     }
     
     bypassPermissionsCheckPromise = (async () => {
       try {
         bypassPermissionsCheckRan = true
         // ... 执行检查
       } finally {
         bypassPermissionsCheckPromise = null
       }
     })()
     
     return bypassPermissionsCheckPromise
   }
   ```

2. **超时处理**:
   ```typescript
   export async function checkAndDisableBypassPermissionsIfNeeded(...): Promise<void> {
     // ...
     const timeout = new Promise<void>((_, reject) => 
       setTimeout(() => reject(new Error('Check timeout')), 5000)
     )
     
     try {
       await Promise.race([shouldDisableBypassPermissions(), timeout])
     } catch (error) {
       logForDebugging('Bypass permissions check timed out', { level: 'warn' })
       // 使用安全默认值
     }
   }
   ```

3. **缓存策略**:
   ```typescript
   const CHECK_CACHE_DURATION = 5 * 60 * 1000 // 5分钟
   let lastCheckTime = 0
   
   export async function checkAndDisableBypassPermissionsIfNeeded(...): Promise<void> {
     const now = Date.now()
     if (now - lastCheckTime < CHECK_CACHE_DURATION) {
       return  // 使用缓存结果
     }
     lastCheckTime = now
     // ... 执行检查
   }
   ```

4. **结构化日志**:
   ```typescript
   logEvent('tengu_permission_killswitch_check', {
     type: 'bypass_permissions',
     result: shouldDisable ? 'disabled' : 'enabled',
     durationMs,
     timestamp: Date.now(),
   })
   ```

5. **错误边界**:
   ```typescript
   export async function checkAndDisableBypassPermissionsIfNeeded(...): Promise<void> {
     try {
       // ... 检查逻辑
     } catch (error) {
       logError(error)
       // 失败安全：默认禁用 bypass 模式
       setAppState(prev => ({
         ...prev,
         toolPermissionContext: createDisabledBypassPermissionsContext(prev.toolPermissionContext),
       }))
     }
   }
   ```

6. **测试钩子**:
   ```typescript
   export function _resetForTesting(): void {
     bypassPermissionsCheckRan = false
     autoModeCheckRan = false
   }
   ```

### 测试建议

1. **单元测试**:
   ```typescript
   describe('bypassPermissionsKillswitch', () => {
     beforeEach(() => {
       _resetForTesting()
     })
     
     it('runs check only once', async () => {
       const mock = jest.fn().mockResolvedValue(true)
       // ... 测试单次执行
     })
     
     it('handles concurrent calls', async () => {
       // 测试并发调用行为
     })
     
     it('resets after login', async () => {
       // 测试重置机制
     })
   })
   ```

2. **集成测试**:
   - 测试与 Statsig/GrowthBook 的集成
   - 测试状态更新正确性
   - 测试通知添加

3. **E2E 测试**:
   - 模拟 gate 变化，验证 UI 响应
   - 测试模式切换场景
