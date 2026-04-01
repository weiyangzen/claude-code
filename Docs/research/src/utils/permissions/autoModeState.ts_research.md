# autoModeState.ts 深度研究

## 场景与职责

`autoModeState.ts` 是 Claude Code 权限系统的状态管理模块，专门负责**自动模式 (Auto Mode) 的状态管理**。该模块设计为独立模块，使调用者可以在功能标志 `TRANSCRIPT_CLASSIFIER` 启用时通过条件 `require()` 加载它。

### 核心职责
1. **自动模式激活状态**: 跟踪自动模式是否处于激活状态
2. **CLI 标志状态**: 记录用户是否通过 CLI 传入了自动模式标志
3. **熔断器状态**: 实现熔断器模式，当 GrowthBook 配置禁用自动模式时阻止重新进入
4. **测试支持**: 提供重置函数用于测试隔离

### 设计背景
自动模式（YOLO 模式）是一种 AI 驱动的权限模式，使用分类器自动评估工具调用的安全性，而非提示用户。该模块的状态管理确保：
- 自动模式的进入和退出状态一致性
- 熔断器机制防止在配置禁用后自动重新进入
- CLI 标志意图能够传递到异步验证流程

---

## 功能点目的

### 1. 自动模式激活状态 (`autoModeActive`)
跟踪自动模式是否当前处于激活状态：
- `setAutoModeActive(active: boolean)` - 设置激活状态
- `isAutoModeActive()` - 查询激活状态

### 2. CLI 标志状态 (`autoModeFlagCli`)
记录用户是否通过 `--permission-mode=auto` CLI 参数请求自动模式：
- `setAutoModeFlagCli(passed: boolean)` - 设置 CLI 标志
- `getAutoModeFlagCli()` - 获取 CLI 标志

**用途**: 即使 GrowthBook 配置禁用了自动模式，CLI 标志仍然携带用户意图，用于在 `verifyAutoModeGateAccess` 中向用户显示通知。

### 3. 熔断器状态 (`autoModeCircuitBroken`)
当 `verifyAutoModeGateAccess` 检测到 `tengu_auto_mode_config.enabled === 'disabled'` 时设置：
- `setAutoModeCircuitBroken(broken: boolean)` - 设置熔断状态
- `isAutoModeGateEnabled()` - 检查熔断器是否断开

**用途**: 阻止 SDK/显式重新进入自动模式，防止在配置禁用后自动模式被意外激活。

### 4. 测试重置 (`_resetForTesting`)
将所有状态重置为初始值，用于测试隔离：
```typescript
export function _resetForTesting(): void {
  autoModeActive = false
  autoModeFlagCli = false
  autoModeCircuitBroken = false
}
```

---

## 具体技术实现

### 状态变量

```typescript
// 模块级私有状态变量
let autoModeActive = false
let autoModeFlagCli = false
let autoModeCircuitBroken = false
```

使用模块级变量实现**单例模式**，确保整个应用共享相同的状态。

### 状态转换图

```
                    ┌─────────────────┐
                    │   Initial State │
                    │ (all false)     │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ autoModeFlagCli │  │ autoModeActive  │  │ autoModeCircuit │
│    = true       │  │    = true       │  │    Broken       │
│                 │  │                 │  │    = true       │
│ (CLI requested) │  │ (Auto mode on)  │  │ (Gate disabled) │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                   │                   │
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ _resetForTesting│
                    │ (all reset)     │
                    └─────────────────┘
```

### 使用模式

#### 条件加载
```typescript
// 在 feature('TRANSCRIPT_CLASSIFIER') 保护下加载
const autoModeStateModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./autoModeState.js') as typeof import('./autoModeState.js')
  : null

// 使用
autoModeStateModule?.setAutoModeActive(true)
```

#### 状态检查
```typescript
// 检查自动模式是否激活
if (autoModeStateModule?.isAutoModeActive()) {
  // 自动模式特定的逻辑
}

// 检查熔断器
if (autoModeStateModule?.isAutoModeGateEnabled()) {
  // 可以进入自动模式
}
```

---

## 关键代码路径与文件引用

### 调用方

| 调用方 | 路径 | 场景 |
|--------|------|------|
| `permissionSetup.ts` | `src/utils/permissions/permissionSetup.ts` | 模式转换、初始化 |
| `permissions.ts` | `src/utils/permissions/permissions.ts` | 权限检查、拒绝跟踪 |
| `bypassPermissionsKillswitch.ts` | `src/utils/permissions/bypassPermissionsKillswitch.ts` | 自动模式门控检查 |
| `REPL.tsx` | `src/screens/REPL.tsx` | UI 状态同步 |
| `useReplBridge.tsx` | `src/hooks/useReplBridge.tsx` | 桥接模式状态 |
| `claude.ts` | `src/services/api/claude.ts` | API 调用上下文 |
| `ExitPlanModeV2Tool.ts` | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | 退出计划模式 |
| `spawnMultiAgent.ts` | `src/tools/shared/spawnMultiAgent.ts` | 多代理权限 |
| `attachments.ts` | `src/utils/attachments.ts` | 附件处理 |
| `PromptInput.tsx` | `src/components/PromptInput/PromptInput.tsx` | 输入组件 |
| `ExitPlanModePermissionRequest.tsx` | `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 权限请求 UI |

### 相关模块

| 模块 | 路径 | 关系 |
|------|------|------|
| `permissionSetup.ts` | `src/utils/permissions/permissionSetup.ts` | 调用 `verifyAutoModeGateAccess` 设置熔断器 |
| `permissions.ts` | `src/utils/permissions/permissions.ts` | 检查自动模式状态进行权限决策 |
| `classifierDecision.ts` | `src/utils/permissions/classifierDecision.ts` | 提供 `isAutoModeAllowlistedTool` |
| `yoloClassifier.ts` | `src/utils/permissions/yoloClassifier.ts` | 自动模式分类器 |

---

## 依赖与外部交互

### 模块依赖

```
autoModeState.ts
    ← 被调用（无运行时依赖，纯状态模块）
```

该模块是**纯状态模块**，不依赖其他模块，确保：
1. 可以安全地在条件 `require()` 中加载
2. 不会引入循环依赖
3. 测试时容易模拟

### 与权限设置的交互

```typescript
// permissionSetup.ts
export async function verifyAutoModeGateAccess(
  toolPermissionContext: ToolPermissionContext,
  fastMode?: boolean,
): Promise<{ updateContext: ContextUpdater; notification?: string }> {
  // ...
  if (config.enabled === 'disabled') {
    autoModeState.setAutoModeCircuitBroken(true)
    // ...
  }
  // ...
}
```

### 与权限检查的交互

```typescript
// permissions.ts
if (
  feature('TRANSCRIPT_CLASSIFIER') &&
  (appState.toolPermissionContext.mode === 'auto' ||
    (appState.toolPermissionContext.mode === 'plan' &&
      (autoModeStateModule?.isAutoModeActive() ?? false)))
) {
  // 自动模式权限处理逻辑
}
```

---

## 风险、边界与改进建议

### 当前风险

1. **模块级状态**: 
   - 使用模块级变量意味着状态在测试间可能泄漏
   - 虽然提供了 `_resetForTesting`，但依赖测试正确调用

2. **无持久化**:
   - 状态仅存在于内存中，页面刷新后丢失
   - 熔断器状态不会跨会话保持

3. **并发风险**:
   - 如果多个异步流程同时修改状态，可能导致竞态条件
   - 例如：`verifyAutoModeGateAccess` 和模式切换同时发生

### 边界情况

1. **重复设置**:
   ```typescript
   setAutoModeActive(true)
   setAutoModeActive(true) // 无操作，但无日志
   ```

2. **状态不一致**:
   ```typescript
   // 熔断器断开但自动模式仍激活
   setAutoModeCircuitBroken(true)
   isAutoModeActive() // 仍可能返回 true
   ```

3. **测试隔离**:
   ```typescript
   // 如果测试忘记调用 _resetForTesting
   // 状态可能从前一个测试泄漏
   ```

### 改进建议

1. **状态持久化**:
   ```typescript
   // 考虑将会话级状态持久化到 sessionStorage
   export function setAutoModeActive(active: boolean): void {
     autoModeActive = active
     if (typeof sessionStorage !== 'undefined') {
       sessionStorage.setItem('claude:autoModeActive', String(active))
     }
   }
   
   // 初始化时恢复
   function initFromStorage(): void {
     if (typeof sessionStorage !== 'undefined') {
       autoModeActive = sessionStorage.getItem('claude:autoModeActive') === 'true'
     }
   }
   ```

2. **状态一致性检查**:
   ```typescript
   export function setAutoModeCircuitBroken(broken: boolean): void {
     autoModeCircuitBroken = broken
     if (broken && autoModeActive) {
       // 熔断时自动退出自动模式
       autoModeActive = false
       logForDebugging('Auto mode deactivated due to circuit breaker', { level: 'warn' })
     }
   }
   ```

3. **事件通知**:
   ```typescript
   type AutoModeStateListener = (state: { active: boolean; circuitBroken: boolean }) => void
   const listeners: AutoModeStateListener[] = []
   
   export function onAutoModeStateChange(listener: AutoModeStateListener): () => void {
     listeners.push(listener)
     return () => {
       const index = listeners.indexOf(listener)
       if (index > -1) listeners.splice(index, 1)
     }
   }
   
   function notifyListeners(): void {
     const state = { active: autoModeActive, circuitBroken: autoModeCircuitBroken }
     listeners.forEach(l => l(state))
   }
   ```

4. **结构化日志**:
   ```typescript
   export function setAutoModeActive(active: boolean): void {
     if (autoModeActive !== active) {
       logForDebugging('Auto mode state changed', {
         from: autoModeActive,
         to: active,
         timestamp: Date.now(),
       })
       autoModeActive = active
     }
   }
   ```

5. **类型安全增强**:
   ```typescript
   // 使用 branded types 防止错误赋值
   type AutoModeActive = boolean & { __brand: 'AutoModeActive' }
   type AutoModeCircuitBroken = boolean & { __brand: 'AutoModeCircuitBroken' }
   
   let autoModeActive: AutoModeActive = false as AutoModeActive
   let autoModeCircuitBroken: AutoModeCircuitBroken = false as AutoModeCircuitBroken
   ```

6. **原子状态更新**:
   ```typescript
   export type AutoModeState = {
     active: boolean
     cliFlag: boolean
     circuitBroken: boolean
   }
   
   export function getAutoModeState(): AutoModeState {
     return {
       active: autoModeActive,
       cliFlag: autoModeFlagCli,
       circuitBroken: autoModeCircuitBroken,
     }
   }
   
   export function setAutoModeState(state: Partial<AutoModeState>): void {
     if (state.active !== undefined) autoModeActive = state.active
     if (state.cliFlag !== undefined) autoModeFlagCli = state.cliFlag
     if (state.circuitBroken !== undefined) autoModeCircuitBroken = state.circuitBroken
   }
   ```

### 测试建议

1. **单元测试**:
   ```typescript
   describe('autoModeState', () => {
     beforeEach(() => {
       _resetForTesting()
     })
     
     it('tracks active state', () => {
       expect(isAutoModeActive()).toBe(false)
       setAutoModeActive(true)
       expect(isAutoModeActive()).toBe(true)
     })
     
     it('tracks CLI flag', () => {
       setAutoModeFlagCli(true)
       expect(getAutoModeFlagCli()).toBe(true)
     })
     
     it('tracks circuit breaker', () => {
       setAutoModeCircuitBroken(true)
       expect(isAutoModeGateEnabled()).toBe(true)
     })
   })
   ```

2. **集成测试**:
   - 测试与 `permissionSetup.ts` 的集成
   - 测试与 `permissions.ts` 的集成
   - 测试模式转换时的状态一致性

3. **并发测试**:
   - 模拟多个异步流程同时修改状态
   - 验证最终状态一致性
