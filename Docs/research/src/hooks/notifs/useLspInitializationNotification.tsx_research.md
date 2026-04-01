# useLspInitializationNotification.tsx 深度研究

## 场景与职责

`useLspInitializationNotification` 是一个 React Hook，用于监控 LSP（Language Server Protocol）初始化状态并在出现问题时显示通知。它定期轮询 LSP 管理器状态，检测初始化失败和服务器错误，并将这些错误同时显示为通知和记录到应用状态中供 `/doctor` 命令查看。

### 核心场景
1. **LSP 管理器初始化失败**：当 LSP 管理器无法初始化时通知用户
2. **LSP 服务器错误**：当任何 LSP 服务器进入错误状态时通知用户
3. **错误去重**：避免重复显示相同的错误通知
4. **状态持久化**：将错误记录到应用状态，供 `/doctor` 显示

## 功能点目的

### 1. LSP 状态轮询
- 每 5 秒（`LSP_POLL_INTERVAL_MS = 5000`）轮询一次 LSP 状态
- 仅在 `ENABLE_LSP_TOOL` 环境变量设置时启用
- 使用 `useInterval` Hook 实现定时轮询

### 2. 错误检测与通知
- 检测 LSP 管理器初始化失败（`status === "failed"`）
- 检测各个 LSP 服务器的错误状态（`server.state === "error"`）
- 显示带有超时（8秒）的通知

### 3. 错误去重机制
- 使用 `notifiedErrorsRef`（Set）追踪已通知的错误
- 使用错误键 `${source}:${errorMessage}` 唯一标识错误
- 避免同一错误重复显示

### 4. 应用状态同步
- 将错误记录到 `appState.plugins.errors`
- 供 `/doctor` 命令显示详细的插件错误信息
- 再次检查去重，避免状态中的重复错误

### 5. 远程模式和滚动排空保护
- 在远程模式下禁用轮询
- 在滚动排空期间暂停轮询（避免与 UI 渲染竞争）

## 具体技术实现

### 关键数据结构

```typescript
// LSP 初始化状态
interface InitializationStatus {
  status: 'pending' | 'not-started' | 'success' | 'failed'
  error?: { message: string }
}

// LSP 服务器状态
interface LspServer {
  state: 'running' | 'error' | 'stopped'
  lastError?: { message: string }
}

// 应用状态中的插件错误
interface PluginError {
  type: 'generic-error'
  source: string
  error: string
}

// 常量
const LSP_POLL_INTERVAL_MS = 5000
```

### 核心流程

#### 1. 初始化与配置
```
Hook 初始化
    ↓
创建 notifiedErrorsRef (Set)
    ↓
创建 addError 回调函数
    ↓
创建 poll 函数
    ↓
使用 useInterval 设置轮询
```

#### 2. 轮询逻辑
```
poll 函数执行
    ↓
检查远程模式 (getIsRemoteMode)
    ↓
检查滚动排空 (getIsScrollDraining)
    ↓
获取初始化状态 (getInitializationStatus)
    ↓
如果状态为 "failed"
    调用 addError("lsp-manager", status.error.message)
    停止轮询 (setShouldPoll(false))
    返回
    ↓
如果状态为 "pending" 或 "not-started"
    返回（等待下次轮询）
    ↓
获取 LSP 服务器管理器 (getLspServerManager)
    ↓
遍历所有服务器
    如果 server.state === "error" 且 server.lastError 存在
        调用 addError(serverName, server.lastError.message)
```

#### 3. 错误添加流程
```
addError(source, errorMessage)
    ↓
生成错误键: `${source}:${errorMessage}`
    ↓
检查 notifiedErrorsRef 中是否已存在
    如果存在，返回（去重）
    ↓
添加到 notifiedErrorsRef
    ↓
记录调试日志 (logForDebugging)
    ↓
更新应用状态 (setAppState)
    - 检查状态中是否已存在相同错误
    - 如果不存在，添加到 plugins.errors 数组
    ↓
提取显示名称（移除 "plugin:" 前缀）
    ↓
显示通知
    - key: `lsp-error-${source}`
    - JSX: "LSP for {displayName} failed · /plugin for details"
    - priority: "medium"
    - timeoutMs: 8000
```

### 关键代码路径

```typescript
export function useLspInitializationNotification() {
  const { addNotification } = useNotifications()
  const setAppState = useSetAppState()
  const [shouldPoll, setShouldPoll] = useState(() => isEnvTruthy("true"))
  const notifiedErrorsRef = useRef<Set<string>>(new Set())

  // addError 回调
  const addError = useCallback((source: string, errorMessage: string) => {
    const errorKey = `${source}:${errorMessage}`
    if (notifiedErrorsRef.current.has(errorKey)) {
      return
    }
    notifiedErrorsRef.current.add(errorKey)
    logForDebugging(`LSP error: ${source} - ${errorMessage}`)
    
    // 更新应用状态
    setAppState(prev => {
      const existingKeys = new Set(prev.plugins.errors.map(e => 
        e.type === "generic-error" ? `generic-error:${e.source}:${e.error}` : `${e.type}:${e.source}`
      ))
      const stateErrorKey = `generic-error:${source}:${errorMessage}`
      if (existingKeys.has(stateErrorKey)) {
        return prev
      }
      return {
        ...prev,
        plugins: {
          ...prev.plugins,
          errors: [...prev.plugins.errors, {
            type: "generic-error" as const,
            source,
            error: errorMessage
          }]
        }
      }
    })
    
    // 显示通知
    const displayName = source.startsWith("plugin:") 
      ? source.split(":")[1] ?? source 
      : source
    addNotification({
      key: `lsp-error-${source}`,
      jsx: <>
        <Text color="error">LSP for {displayName} failed</Text>
        <Text dimColor={true}> · /plugin for details</Text>
      </>,
      priority: "medium",
      timeoutMs: 8000
    })
  }, [addNotification, setAppState])

  // 轮询函数
  const poll = useCallback(() => {
    if (getIsRemoteMode()) return
    if (getIsScrollDraining()) return
    
    const status = getInitializationStatus()
    if (status.status === "failed") {
      addError("lsp-manager", status.error.message)
      setShouldPoll(false)
      return
    }
    if (status.status === "pending" || status.status === "not-started") {
      return
    }
    
    const manager = getLspServerManager()
    if (manager) {
      const servers = manager.getAllServers()
      for (const [serverName, server] of servers) {
        if (server.state === "error" && server.lastError) {
          addError(serverName, server.lastError.message)
        }
      }
    }
  }, [addError])

  // 设置轮询
  useInterval(poll, shouldPoll ? LSP_POLL_INTERVAL_MS : null)
  
  // 立即执行一次（如果应该轮询）
  useEffect(() => {
    if (getIsRemoteMode() || !shouldPoll) return
    poll()
  }, [poll, shouldPoll])
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `React`, `useState`, `useRef`, `useCallback` | `react` | React Hook API |
| `useInterval` | `usehooks-ts` | 定时轮询 |
| `getIsRemoteMode`, `getIsScrollDraining` | `src/bootstrap/state.js` | 远程模式和滚动排空检测 |
| `useNotifications` | `src/context/notifications.js` | 通知系统 |
| `Text` | `src/ink.js` | Ink 文本组件 |
| `getInitializationStatus`, `getLspServerManager` | `src/services/lsp/manager.js` | LSP 管理器 API |
| `useSetAppState` | `src/state/AppState.js` | 应用状态更新 |
| `logForDebugging` | `src/utils/debug.js` | 调试日志 |
| `isEnvTruthy` | `src/utils/envUtils.js` | 环境变量检查 |

### 依赖模块详解

#### 1. LSP 管理器 (src/services/lsp/manager.js)
提供 LSP 相关功能：
- `getInitializationStatus()`: 获取 LSP 管理器初始化状态
- `getLspServerManager()`: 获取 LSP 服务器管理器实例
- `manager.getAllServers()`: 获取所有注册的 LSP 服务器

#### 2. useInterval (usehooks-ts)
提供可靠的定时器 Hook，支持动态间隔和清理：
```typescript
useInterval(callback, delay | null)
```

#### 3. 应用状态 (src/state/AppState.ts)
存储插件错误信息：
```typescript
interface AppState {
  plugins: {
    errors: PluginError[]
  }
}
```

## 风险、边界与改进建议

### 潜在风险

1. **内存泄漏**
   - `notifiedErrorsRef` 使用 Set 存储错误键
   - 长时间运行的会话中可能积累大量错误
   - 建议：限制 Set 大小或定期清理

2. **轮询开销**
   - 每 5 秒轮询一次，即使 LSP 状态稳定后也在继续
   - 可以考虑在成功初始化后降低轮询频率

3. **错误键冲突**
   - 使用 `${source}:${errorMessage}` 作为键
   - 如果错误消息包含动态内容（如时间戳），可能导致重复通知

4. **状态更新竞态**
   - `addError` 可能在短时间内被多次调用
   - 使用函数式状态更新是正确的，但仍需注意性能

### 边界情况

1. **ENABLE_LSP_TOOL 未设置**
   - `shouldPoll` 默认为 `false`（`isEnvTruthy("true")` 在环境变量未设置时返回 false）
   - 实际上代码中使用了 `isEnvTruthy("true")` 这看起来是个 bug，应该是检查实际的环境变量

2. **管理器初始化失败后恢复**
   - 一旦检测到失败，停止轮询
   - 如果问题被修复，需要重启应用才能恢复检测

3. **服务器错误持续存在**
   - 同一错误不会重复通知（通过 Set 去重）
   - 但新用户可能看不到历史错误

4. **远程模式切换**
   - 如果在会话中切换到远程模式，轮询会停止
   - 但已显示的通知不会自动清除

### 改进建议

1. **修复环境变量检查**
   ```typescript
   // 当前代码可能有 bug
   const [shouldPoll, setShouldPoll] = useState(() => isEnvTruthy(process.env.ENABLE_LSP_TOOL))
   
   // 或者使用正确的环境变量名
   const [shouldPoll, setShouldPoll] = useState(() => isEnvTruthy("ENABLE_LSP_TOOL"))
   ```

2. **限制错误追踪集合大小**
   ```typescript
   const MAX_TRACKED_ERRORS = 100
   if (notifiedErrorsRef.current.size >= MAX_TRACKED_ERRORS) {
     // 清理最旧的错误或使用 LRU 策略
     const [first] = notifiedErrorsRef.current
     notifiedErrorsRef.current.delete(first)
   }
   ```

3. **动态轮询频率**
   ```typescript
   const [pollInterval, setPollInterval] = useState(LSP_POLL_INTERVAL_MS)
   
   // 成功初始化后降低频率
   if (status.status === "success") {
     setPollInterval(30000) // 30秒
   }
   ```

4. **添加分析事件**
   ```typescript
   logEvent('tengu_lsp_error', { source, error: errorMessage.substring(0, 100) })
   ```

5. **错误恢复检测**
   ```typescript
   // 如果之前失败的错误已解决，可以清除相关通知
   if (server.state === "running" && wasPreviouslyInError) {
     removeNotification(`lsp-error-${serverName}`)
   }
   ```

6. **改进错误键生成**
   ```typescript
   // 使用更稳定的键，去除可能的动态内容
   const errorKey = `${source}:${errorMessage.replace(/\d+/g, 'N')}`
   ```

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useLspInitializationNotification.tsx`
- **启动通知基类**: `src/hooks/notifs/useStartupNotification.ts`
- **LSP 管理器**: `src/services/lsp/manager.js`
- **通知系统**: `src/context/notifications.tsx`
- **应用状态**: `src/state/AppState.ts`
- **调试工具**: `src/utils/debug.ts`
- **环境工具**: `src/utils/envUtils.ts`
- **Ink 组件**: `src/ink.js`
