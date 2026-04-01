# leaderPermissionBridge.ts 研究文档

## 场景与职责

`leaderPermissionBridge.ts` 是 Agent Swarm 系统中的权限桥接模块，它解决了 React 组件状态管理与非 React 代码之间的解耦问题。具体来说，它允许 REPL（Read-Eval-Print Loop）组件将其权限确认队列和权限上下文设置器注册为模块级变量，供 In-Process 队友运行器使用。

### 核心职责
1. **桥接 React 与非 React 代码**: 将 React 的 `setToolUseConfirmQueue` 和 `setToolPermissionContext` 暴露给非 React 模块
2. **支持权限委托**: 使 In-Process 队友能够使用 Leader 的标准权限对话框
3. **维护单向数据流**: Leader 注册设置器，队友通过桥接读取并使用

## 功能点目的

### 1. 权限确认队列桥接

In-Process 队友需要请求工具使用权限时，标准路径是使用 Leader 的 `ToolUseConfirm` 对话框。这个对话框需要操作 React 状态队列，而队友运行器（`inProcessRunner.ts`）是纯逻辑代码，不直接访问 React。

**解决方案**:
- Leader 组件（如 REPL）调用 `registerLeaderToolUseConfirmQueue(setter)` 注册设置器
- 队友运行器调用 `getLeaderToolUseConfirmQueue()` 获取设置器
- 队友通过设置器将权限请求加入 Leader 的队列

### 2. 权限上下文桥接

当用户在权限对话框中修改权限规则（如"始终允许此目录"），这些更新需要回写到 Leader 的共享权限上下文。

**解决方案**:
- Leader 注册 `setToolPermissionContext` 设置器
- 队友在权限批准时通过桥接更新 Leader 的上下文
- 支持 `preserveMode` 选项防止工作线程的模式泄漏回协调器

### 3. 生命周期管理

提供注册和注销函数，确保：
- Leader 启动时注册设置器
- Leader 关闭时注销设置器，防止悬空引用
- 队友在使用前检查设置器是否存在（回退到邮箱系统）

## 具体技术实现

### 类型定义

```typescript
// 设置 ToolUseConfirm 队列的函数类型
export type SetToolUseConfirmQueueFn = (
  updater: (prev: ToolUseConfirm[]) => ToolUseConfirm[],
) => void

// 设置权限上下文的函数类型
export type SetToolPermissionContextFn = (
  context: ToolPermissionContext,
  options?: { preserveMode?: boolean },
) => void
```

### 模块级状态

```typescript
// 模块级变量，保存注册设置器
let registeredSetter: SetToolUseConfirmQueueFn | null = null
let registeredPermissionContextSetter: SetToolPermissionContextFn | null = null
```

### API 设计

```typescript
// ToolUseConfirm 队列桥接
export function registerLeaderToolUseConfirmQueue(setter: SetToolUseConfirmQueueFn): void
export function getLeaderToolUseConfirmQueue(): SetToolUseConfirmQueueFn | null
export function unregisterLeaderToolUseConfirmQueue(): void

// 权限上下文桥接
export function registerLeaderSetToolPermissionContext(setter: SetToolPermissionContextFn): void
export function getLeaderSetToolPermissionContext(): SetToolPermissionContextFn | null
export function unregisterLeaderSetToolPermissionContext(): void
```

### 使用模式

#### Leader 侧（React 组件）
```typescript
// REPL.tsx 或类似组件
import { registerLeaderToolUseConfirmQueue, registerLeaderSetToolPermissionContext } from './leaderPermissionBridge.js'

function REPL() {
  const [toolUseConfirmQueue, setToolUseConfirmQueue] = useState<ToolUseConfirm[]>([])
  const [toolPermissionContext, setToolPermissionContext] = useState<ToolPermissionContext>(...)
  
  useEffect(() => {
    // 注册设置器到桥接
    registerLeaderToolUseConfirmQueue(setToolUseConfirmQueue)
    registerLeaderSetToolPermissionContext(setToolPermissionContext)
    
    return () => {
      // 清理时注销
      unregisterLeaderToolUseConfirmQueue()
      unregisterLeaderSetToolPermissionContext()
    }
  }, [])
  
  // ...
}
```

#### 队友侧（inProcessRunner.ts）
```typescript
// createInProcessCanUseTool 函数中
const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()

if (setToolUseConfirmQueue) {
  // 使用 Leader 的对话框
  return new Promise<PermissionDecision>(resolve => {
    setToolUseConfirmQueue(queue => [...queue, {
      // ... 权限请求详情
      onAllow(updatedInput, permissionUpdates) {
        // 回写权限更新
        const setToolPermissionContext = getLeaderSetToolPermissionContext()
        if (setToolPermissionContext && permissionUpdates.length > 0) {
          const updatedContext = applyPermissionUpdates(...)
          setToolPermissionContext(updatedContext, { preserveMode: true })
        }
        resolve({ behavior: 'allow', updatedInput, ... })
      },
      onReject(feedback) { ... },
    }])
  })
} else {
  // 回退到邮箱系统
  return sendPermissionRequestViaMailbox(...)
}
```

## 关键代码路径与文件引用

### 本文件导出
| 导出项 | 类型 | 说明 |
|-------|------|------|
| `SetToolUseConfirmQueueFn` | type | 队列设置器函数类型 |
| `SetToolPermissionContextFn` | type | 上下文设置器函数类型 |
| `registerLeaderToolUseConfirmQueue` | function | 注册队列设置器 |
| `getLeaderToolUseConfirmQueue` | function | 获取队列设置器 |
| `unregisterLeaderToolUseConfirmQueue` | function | 注销队列设置器 |
| `registerLeaderSetToolPermissionContext` | function | 注册上下文设置器 |
| `getLeaderSetToolPermissionContext` | function | 获取上下文设置器 |
| `unregisterLeaderSetToolPermissionContext` | function | 注销上下文设置器 |

### 依赖导入

| 导入路径 | 用途 |
|---------|------|
| `../../components/permissions/PermissionRequest.js` | `ToolUseConfirm` 类型 |
| `../../Tool.js` | `ToolPermissionContext` 类型 |

### 被引用情况

通过代码分析，本模块被以下文件引用：

#### `src/utils/swarm/inProcessRunner.ts`
```typescript
import {
  getLeaderSetToolPermissionContext,
  getLeaderToolUseConfirmQueue,
} from './leaderPermissionBridge.js'
```

在 `createInProcessCanUseTool` 函数中使用：
1. 获取 Leader 的权限队列以显示 `ToolUseConfirm` 对话框
2. 获取 Leader 的权限上下文设置器以回写权限更新

#### `src/screens/REPL.tsx`（推测）
虽然未在提供的代码片段中直接显示，但根据设计意图，REPL 组件应该：
1. 导入并调用注册函数
2. 在组件卸载时调用注销函数

## 依赖与外部交互

### 与 React 的关系
- 本模块本身不依赖 React
- 但设计目的是桥接 React 状态管理
- 使用函数引用来避免直接依赖 React 类型

### 与权限系统的关系
- 依赖 `PermissionRequest.js` 中的 `ToolUseConfirm` 类型
- 依赖 `Tool.js` 中的 `ToolPermissionContext` 类型
- 是权限系统的中继层，而非决策层

### 数据流

```
┌─────────────────┐     register      ┌─────────────────────┐
│   REPL.tsx      │ ─────────────────>│ leaderPermissionBridge.ts │
│  (React 组件)    │                   │    (模块级存储)       │
│                 │ <─────────────────│                     │
└─────────────────┘     get/set       └─────────────────────┘
                                              │
                                              │ get
                                              ▼
                                     ┌─────────────────────┐
                                     │ inProcessRunner.ts  │
                                     │  (队友运行器)        │
                                     └─────────────────────┘
```

## 风险、边界与改进建议

### 风险点

1. **悬空引用风险**
   - 如果 Leader 组件卸载但未调用注销函数，模块级变量仍保留已卸载组件的设置器
   - 可能导致内存泄漏或尝试更新已卸载组件的状态
   - **缓解**: REPL 组件的 `useEffect` 返回清理函数

2. **竞态条件**
   - 多个 Leader 实例（理论上不应发生）可能互相覆盖注册
   - 队友可能使用错误的设置器
   - **缓解**: Claude Code 是单实例应用，通常只有一个 REPL

3. **类型安全**
   - 使用模块级变量丢失了 TypeScript 的严格类型检查
   - `getLeaderToolUseConfirmQueue()` 返回可能为 `null`
   - **缓解**: 调用方必须检查返回值

4. **测试复杂性**
   - 模块级状态在测试间共享，需要清理
   - 并行测试可能互相干扰
   - **缓解**: 每个测试后调用注销函数

### 边界情况

1. **设置器为 null**
   - 队友启动时 Leader 可能尚未注册
   - 或 Leader 已注销但队友仍在运行
   - 队友代码已处理：回退到邮箱系统

2. **并发权限请求**
   - 多个队友同时请求权限
   - 队列设置器需要正确处理并发更新
   - React 的 `setState` 函数形式（接收 updater）天然支持

3. **权限更新冲突**
   - 多个队友同时修改权限上下文
   - 最后一个写入者获胜（Last Write Wins）
   - 当前设计接受此行为

4. **preserveMode 选项**
   - 防止工作线程转换后的 'acceptEdits' 上下文泄漏回协调器
   - 这是特定于内部实现的细节

### 改进建议

1. **添加验证机制**
   ```typescript
   export function registerLeaderToolUseConfirmQueue(
     setter: SetToolUseConfirmQueueFn,
     validator?: () => boolean  // 验证设置器是否仍然有效
   ): void
   ```

2. **支持多个消费者**
   - 当前设计假设只有一个 Leader
   - 可考虑支持多个设置器（如主窗口 + 浮动窗口）
   ```typescript
   const registeredSetters: SetToolUseConfirmQueueFn[] = []
   ```

3. **添加调试信息**
   ```typescript
   export function getBridgeDebugInfo() {
     return {
       hasQueueSetter: registeredSetter !== null,
       hasContextSetter: registeredPermissionContextSetter !== null,
       // 可添加注册时间戳等
     }
   }
   ```

4. **类型强化**
   - 使用 branded types 避免错误赋值
   ```typescript
   type LeaderSetterToken = { readonly __brand: 'leader-setter' }
   export type SetToolUseConfirmQueueFn = (
     updater: (prev: ToolUseConfirm[]) => ToolUseConfirm[]
   ) => void & LeaderSetterToken
   ```

5. **事件通知**
   - 添加注册/注销事件监听
   - 便于调试和监控
   ```typescript
   type BridgeEvent = 
     | { type: 'registered'; kind: 'queue' | 'context' }
     | { type: 'unregistered'; kind: 'queue' | 'context' }
   
   export function onBridgeEvent(callback: (event: BridgeEvent) => void): () => void
   ```

6. **超时保护**
   - 队友等待权限响应时添加超时
   - 防止无限等待导致资源占用
   ```typescript
   const PERMISSION_TIMEOUT_MS = 5 * 60 * 1000  // 5分钟
   ```

7. **文档化契约**
   - 明确 Leader 和队友的调用约定
   - 添加 JSDoc 说明调用时机和线程安全要求
