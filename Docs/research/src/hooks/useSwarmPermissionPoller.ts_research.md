# useSwarmPermissionPoller.ts 深度研究文档

## 场景与职责

`useSwarmPermissionPoller` 是一个 React Hook，用于在 Agent Swarm 中作为工作节点（Worker）时轮询权限响应。当工作节点需要工具使用权限时，它会向团队领导发送请求，然后通过此 Hook 轮询响应。

### 核心职责

1. **权限响应轮询**: 定期轮询权限响应文件
2. **回调注册管理**: 管理待处理权限请求的回调注册表
3. **响应处理**: 处理收到的权限响应并调用相应回调
4. **沙盒权限支持**: 同时支持普通工具权限和沙盒网络权限
5. **邮箱集成**: 支持通过邮箱系统接收权限响应

### 使用场景

- **工作节点权限请求**: 工作节点需要执行受权限控制的工具
- **领导审批流程**: 等待团队领导审批或拒绝权限请求
- **沙盒网络访问**: 沙盒环境需要网络访问权限时
- **权限更新应用**: 接收并应用"始终允许"等权限更新

---

## 功能点目的

### 1. 权限响应轮询

每 500ms 轮询一次权限响应：
- 检查是否处于工作节点模式
- 防止并发轮询
- 遍历所有待处理的回调

### 2. 回调注册系统

模块级注册表管理待处理请求：
- `registerPermissionCallback`: 注册权限回调
- `unregisterPermissionCallback`: 取消注册
- `hasPermissionCallback`: 检查是否存在

### 3. 响应处理

处理收到的权限响应：
- 查找对应的回调
- 根据决策调用 `onAllow` 或 `onReject`
- 传递权限更新和修改后的输入

### 4. 沙盒权限支持

独立的沙盒权限回调注册表：
- `registerSandboxPermissionCallback`: 注册沙盒回调
- `processSandboxPermissionResponse`: 处理沙盒响应

### 5. 邮箱消息处理

支持通过邮箱系统接收权限响应：
- `processMailboxPermissionResponse`: 处理邮箱权限响应
- `processMailboxSandboxPermissionResponse`: 处理邮箱沙盒响应

---

## 具体技术实现

### 关键数据结构

```typescript
// 权限响应回调
interface PermissionResponseCallback {
  requestId: string
  toolUseId: string
  onAllow: (
    updatedInput: Record<string, unknown> | undefined,
    permissionUpdates: PermissionUpdate[],
    feedback?: string,
  ) => void
  onReject: (feedback?: string) => void
}

// 沙盒权限回调
interface SandboxPermissionResponseCallback {
  requestId: string
  host: string
  resolve: (allow: boolean) => void
}

// 权限响应
interface PermissionResponse {
  requestId: string
  decision: 'approved' | 'denied'
  timestamp: string
  feedback?: string
  updatedInput?: Record<string, unknown>
  permissionUpdates?: unknown[]
}

// 模块级注册表
const pendingCallbacks: Map<string, PermissionResponseCallback> = new Map()
const pendingSandboxCallbacks: Map<string, SandboxPermissionResponseCallback> = new Map()
```

### 核心流程

#### 1. 回调注册流程
```
调用 registerPermissionCallback(callback)
  ↓
pendingCallbacks.set(callback.requestId, callback)
  ↓
记录调试日志
```

#### 2. 轮询流程
```
useInterval 触发 (每 500ms)
  ↓
poll 函数执行
  ↓
检查 isSwarmWorker() → 否 则返回
  ↓
检查 isProcessingRef → 是 则返回（防止并发）
  ↓
检查 pendingCallbacks.size → 0 则返回
  ↓
设置 isProcessingRef = true
  ↓
获取 agentName 和 teamName
  ↓
遍历 pendingCallbacks:
  对每个 requestId:
    调用 pollForResponse(requestId, agentName, teamName)
      ↓
    如果响应存在:
      调用 processResponse(response)
        ↓
      如果处理成功:
        调用 removeWorkerResponse 清理响应文件
  ↓
设置 isProcessingRef = false
```

#### 3. 响应处理流程
```
processResponse(response)
  ↓
从 pendingCallbacks 查找 callback
  ↓
未找到 → 记录日志，返回 false
  ↓
从注册表删除 callback（防止重复处理）
  ↓
根据 decision:
  ├── 'approved' → 
  │   解析 permissionUpdates
  │   调用 callback.onAllow(updatedInput, permissionUpdates)
  └── 'denied' → 
      调用 callback.onReject(feedback)
  ↓
返回 true
```

### 关键代码路径

#### 权限更新解析（行 35-53）
```typescript
function parsePermissionUpdates(raw: unknown): PermissionUpdate[] {
  if (!Array.isArray(raw)) {
    return []
  }
  const schema = permissionUpdateSchema()
  const valid: PermissionUpdate[] = []
  for (const entry of raw) {
    const result = schema.safeParse(entry)
    if (result.success) {
      valid.push(result.data)
    } else {
      logForDebugging(
        `[SwarmPermissionPoller] Dropping malformed permissionUpdate entry: ${result.error.message}`,
        { level: 'warn' },
      )
    }
  }
  return valid
}
```

注释说明：验证来自外部来源（邮箱 IPC、磁盘轮询）的权限更新，过滤掉格式错误的条目。

#### 邮箱响应处理（行 124-156）
```typescript
export function processMailboxPermissionResponse(params: {
  requestId: string
  decision: 'approved' | 'rejected'
  feedback?: string
  updatedInput?: Record<string, unknown>
  permissionUpdates?: unknown
}): boolean {
  const callback = pendingCallbacks.get(params.requestId)

  if (!callback) {
    logForDebugging(
      `[SwarmPermissionPoller] No callback registered for mailbox response ${params.requestId}`,
    )
    return false
  }

  // Remove from registry before invoking callback
  pendingCallbacks.delete(params.requestId)

  if (params.decision === 'approved') {
    const permissionUpdates = parsePermissionUpdates(params.permissionUpdates)
    const updatedInput = params.updatedInput
    callback.onAllow(updatedInput, permissionUpdates)
  } else {
    callback.onReject(params.feedback)
  }

  return true
}
```

#### Hook 实现（行 268-330）
```typescript
export function useSwarmPermissionPoller(): void {
  const isProcessingRef = useRef(false)

  const poll = useCallback(async () => {
    if (!isSwarmWorker()) return
    if (isProcessingRef.current) return
    if (pendingCallbacks.size === 0) return

    isProcessingRef.current = true

    try {
      const agentName = getAgentName()
      const teamName = getTeamName()

      if (!agentName || !teamName) return

      for (const [requestId, _callback] of pendingCallbacks) {
        const response = await pollForResponse(requestId, agentName, teamName)

        if (response) {
          const processed = processResponse(response)
          if (processed) {
            await removeWorkerResponse(requestId, agentName, teamName)
          }
        }
      }
    } catch (error) {
      logForDebugging(
        `[SwarmPermissionPoller] Error during poll: ${errorMessage(error)}`,
      )
    } finally {
      isProcessingRef.current = false
    }
  }, [])

  const shouldPoll = isSwarmWorker()
  useInterval(() => void poll(), shouldPoll ? POLL_INTERVAL_MS : null)

  useEffect(() => {
    if (isSwarmWorker()) {
      void poll()
    }
  }, [poll])
}
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `usehooks-ts` | `useInterval` Hook |
| `../utils/debug.js` | `logForDebugging` |
| `../utils/errors.js` | `errorMessage` |
| `../utils/permissions/PermissionUpdateSchema.js` | `PermissionUpdate`, `permissionUpdateSchema` |
| `../utils/swarm/permissionSync.js` | `isSwarmWorker`, `PermissionResponse`, `pollForResponse`, `removeWorkerResponse` |
| `../utils/teammate.js` | `getAgentName`, `getTeamName` |

### 外部交互

1. **权限同步系统**: 
   - `pollForResponse()`: 轮询权限响应
   - `removeWorkerResponse()`: 清理已处理的响应
   - `isSwarmWorker()`: 检查是否处于工作节点模式

2. **队友系统**: 
   - `getAgentName()`: 获取当前代理名称
   - `getTeamName()`: 获取团队名称

3. **权限更新验证**: 
   - `permissionUpdateSchema()`: 验证权限更新格式

4. **邮箱系统**: 
   - `processMailboxPermissionResponse()`: 处理邮箱权限响应
   - `processSandboxPermissionResponse()`: 处理邮箱沙盒响应

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**: `isProcessingRef` 防止并发，但快速连续调用仍可能有问题
2. **响应丢失**: 如果回调在处理前被取消注册，响应可能丢失
3. **僵尸回调**: 未正确清理的回调可能永远留在注册表中

### 边界情况

1. **非工作节点模式**: Hook 在非工作节点模式下几乎不执行任何操作
2. **空注册表**: 没有待处理回调时跳过轮询
3. **agentName/teamName 缺失**: 缺少身份标识时无法轮询
4. **响应处理失败**: 处理响应时出错不会阻止其他响应的处理

### 改进建议

1. **超时机制**: 为权限请求添加超时，自动清理过期回调
2. **批处理**: 批量处理多个响应减少文件系统操作
3. **退避策略**: 轮询失败时添加指数退避
4. **持久化**: 考虑将待处理请求持久化，支持进程重启恢复
5. **健康检查**: 定期检查注册表健康状态，清理异常条目
6. **性能优化**: 使用文件系统事件监听替代轮询

### 测试关注点

1. 权限响应的正确处理和回调调用
2. 并发轮询的防止
3. 邮箱响应处理
4. 沙盒权限处理
5. 非工作节点模式下的行为
6. 回调注册和取消注册的正确性
7. 错误处理和日志记录
