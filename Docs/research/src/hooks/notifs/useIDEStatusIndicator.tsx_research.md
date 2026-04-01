# useIDEStatusIndicator.tsx 深度研究

## 场景与职责

`useIDEStatusIndicator` 是一个 React Hook，用于管理 IDE（集成开发环境）连接状态的通知和提示。它处理 IDE 扩展安装、连接状态变化、JetBrains 特殊处理等多种场景，为用户提供及时的 IDE 集成状态反馈。

### 核心场景
1. **IDE 连接状态提示**：显示 IDE 连接/断开状态
2. **IDE 扩展安装提示**：提示用户安装 IDE 扩展以获得更好的体验
3. **JetBrains 特殊处理**：针对 JetBrains IDE 的特殊提示逻辑
4. **安装错误提示**：当 IDE 扩展安装失败时显示错误信息
5. **选择提示**：当 IDE 中有文本选择时显示相关信息

## 功能点目的

### 1. IDE 提示显示（Hint）
- 检测是否运行在支持的终端中（VS Code、JetBrains 等）
- 如果不在支持的终端中且 IDE 未连接，延迟 3 秒后显示 `/ide` 命令提示
- 限制显示次数（最多 5 次），避免过度打扰
- 使用 `ideHintShownCount` 在全局配置中持久化计数

### 2. IDE 连接状态指示
- 监听 IDE 连接状态变化（connected/disconnected/pending）
- 显示 IDE 断开连接的错误通知
- 在 IDE 重新连接或显示选择信息时清除断开通知

### 3. JetBrains IDE 特殊处理
- 检测是否为 JetBrains IDE
- 显示特殊的插件连接提示（"IDE plugin not connected · /status for info"）
- 处理 JetBrains 插件未安装的情况

### 4. 安装错误处理
- 检测 IDE 扩展安装错误
- 显示安装失败通知，引导用户查看 `/status` 获取详情

### 5. IDE 选择信息展示
- 当 IDE 中有文本选择时显示选择信息
- 在有选择信息时隐藏断开连接通知

## 具体技术实现

### 关键数据结构

```typescript
// IDE 连接状态
interface IdeConnectionResult {
  status: 'connected' | 'disconnected' | 'pending' | null
  ideName: string | null
}

// IDE 扩展安装状态
interface IDEExtensionInstallationStatus {
  installed: boolean
  error: string | null
  installedVersion: string | null
  ideType: IdeType | null
}

// IDE 选择信息
interface IDESelection {
  lineCount: number
  lineStart?: number
  text?: string
  filePath?: string
}

// Props 接口
interface Props {
  ideInstallationStatus: IDEExtensionInstallationStatus | null
  ideSelection: IDESelection | undefined
  mcpClients: MCPServerConnection[]
}

// 常量
const MAX_IDE_HINT_SHOW_COUNT = 5
```

### 核心流程

#### 1. 状态计算
```
输入: ideInstallationStatus, ideSelection, mcpClients
    ↓
获取 IDE 连接状态 (useIdeConnectionStatus)
    ↓
计算派生状态：
    - isJetBrains: 是否为 JetBrains IDE
    - showIDEInstallErrorOrJetBrainsInfo: 显示安装错误或 JetBrains 信息
    - shouldShowIdeSelection: 是否显示选择信息
    - shouldShowConnected: 是否显示已连接状态
    - showIDEInstallError: 是否显示安装错误
    - showJetBrainsInfo: 是否显示 JetBrains 信息
```

#### 2. IDE 提示效果（Effect 1）
```
触发条件: addNotification, removeNotification, ideStatus, showJetBrainsInfo 变化
    ↓
如果运行在支持的终端中，或 IDE 已连接，或需要显示 JetBrains 信息
    移除 "ide-status-hint" 通知
    返回
    ↓
如果已显示过提示或已达到最大次数
    返回
    ↓
延迟 3 秒后检测 IDE
    ↓
如果检测到 IDE 且未显示过提示
    显示 "/ide for {ideName}" 提示
    增加显示计数
```

#### 3. IDE 断开连接效果（Effect 2）
```
触发条件: addNotification, removeNotification, ideStatus, ideName, showIDEInstallError, showJetBrainsInfo 变化
    ↓
如果需要显示安装错误、JetBrains 信息，或 IDE 不是断开状态，或没有 IDE 名称
    移除 "ide-status-disconnected" 通知
    返回
    ↓
显示 "{ideName} disconnected" 错误通知
```

#### 4. JetBrains 信息效果（Effect 3）
```
触发条件: addNotification, removeNotification, showJetBrainsInfo 变化
    ↓
如果不需要显示 JetBrains 信息
    移除 "ide-status-jetbrains-disconnected" 通知
    返回
    ↓
显示 "IDE plugin not connected · /status for info" 通知
```

#### 5. 安装错误效果（Effect 4）
```
触发条件: addNotification, removeNotification, showIDEInstallError 变化
    ↓
如果不需要显示安装错误
    移除 "ide-status-install-error" 通知
    返回
    ↓
显示 "IDE extension install failed (see /status for info)" 错误通知
```

### 关键代码路径

```typescript
export function useIDEStatusIndicator({
  ideSelection,
  mcpClients,
  ideInstallationStatus,
}: Props) {
  const { addNotification, removeNotification } = useNotifications()
  const { status: ideStatus, ideName } = useIdeConnectionStatus(mcpClients)
  const hasShownHintRef = useRef(false)

  // 派生状态计算
  const isJetBrains = ideInstallationStatus 
    ? isJetBrainsIde(ideInstallationStatus?.ideType) 
    : false
  const showIDEInstallErrorOrJetBrainsInfo = ideInstallationStatus?.error || isJetBrains
  const shouldShowIdeSelection = ideStatus === "connected" && 
    (ideSelection?.filePath || ideSelection?.text && ideSelection.lineCount > 0)
  const shouldShowConnected = ideStatus === "connected" && !shouldShowIdeSelection
  const showIDEInstallError = showIDEInstallErrorOrJetBrainsInfo && !isJetBrains && 
    !shouldShowConnected && !shouldShowIdeSelection
  const showJetBrainsInfo = showIDEInstallErrorOrJetBrainsInfo && isJetBrains && 
    !shouldShowConnected && !shouldShowIdeSelection

  // Effect 1: IDE 提示
  useEffect(() => {
    if (getIsRemoteMode()) return
    if (isSupportedTerminal() || ideStatus !== null || showJetBrainsInfo) {
      removeNotification("ide-status-hint")
      return
    }
    if (hasShownHintRef.current || 
        (getGlobalConfig().ideHintShownCount ?? 0) >= MAX_IDE_HINT_SHOW_COUNT) {
      return
    }
    const timeoutId = setTimeout(() => {
      detectIDEs(true).then(infos => {
        const ideName = infos[0]?.name
        if (ideName && !hasShownHintRef.current) {
          hasShownHintRef.current = true
          saveGlobalConfig(current => ({
            ...current,
            ideHintShownCount: (current.ideHintShownCount ?? 0) + 1
          }))
          addNotification({
            key: "ide-status-hint",
            jsx: <Text dimColor={true}>/ide for <Text color="ide">{ideName}</Text></Text>,
            priority: "low"
          })
        }
      })
    }, 3000)
    return () => clearTimeout(timeoutId)
  }, [addNotification, removeNotification, ideStatus, showJetBrainsInfo])

  // Effect 2: 断开连接通知
  useEffect(() => {
    if (getIsRemoteMode()) return
    if (showIDEInstallError || showJetBrainsInfo || ideStatus !== "disconnected" || !ideName) {
      removeNotification("ide-status-disconnected")
      return
    }
    addNotification({
      key: "ide-status-disconnected",
      text: `${ideName} disconnected`,
      color: "error",
      priority: "medium"
    })
  }, [addNotification, removeNotification, ideStatus, ideName, showIDEInstallError, showJetBrainsInfo])

  // Effect 3: JetBrains 信息
  useEffect(() => {
    if (getIsRemoteMode()) return
    if (!showJetBrainsInfo) {
      removeNotification("ide-status-jetbrains-disconnected")
      return
    }
    addNotification({
      key: "ide-status-jetbrains-disconnected",
      text: "IDE plugin not connected · /status for info",
      priority: "medium"
    })
  }, [addNotification, removeNotification, showJetBrainsInfo])

  // Effect 4: 安装错误
  useEffect(() => {
    if (getIsRemoteMode()) return
    if (!showIDEInstallError) {
      removeNotification("ide-status-install-error")
      return
    }
    addNotification({
      key: "ide-status-install-error",
      text: "IDE extension install failed (see /status for info)",
      color: "error",
      priority: "medium"
    })
  }, [addNotification, removeNotification, showIDEInstallError])
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `React`, `useEffect`, `useRef` | `react` | React Hook API |
| `useNotifications` | `src/context/notifications.js` | 通知系统 |
| `Text` | `src/ink.js` | Ink 文本组件 |
| `MCPServerConnection` | `src/services/mcp/types.js` | MCP 服务器连接类型 |
| `getGlobalConfig`, `saveGlobalConfig` | `src/utils/config.js` | 全局配置读写 |
| `detectIDEs`, `IDEExtensionInstallationStatus`, `isJetBrainsIde`, `isSupportedTerminal` | `src/utils/ide.js` | IDE 检测和工具函数 |
| `getIsRemoteMode` | `src/bootstrap/state.js` | 远程模式检测 |
| `useIdeConnectionStatus` | `src/hooks/useIdeConnectionStatus.js` | IDE 连接状态 Hook |
| `IDESelection` | `src/hooks/useIdeSelection.js` | IDE 选择类型 |

### 依赖模块详解

#### 1. useIdeConnectionStatus (src/hooks/useIdeConnectionStatus.ts)
```typescript
export function useIdeConnectionStatus(
  mcpClients?: MCPServerConnection[]
): IdeConnectionResult {
  return useMemo(() => {
    const ideClient = mcpClients?.find(client => client.name === 'ide')
    if (!ideClient) {
      return { status: null, ideName: null }
    }
    const config = ideClient.config
    const ideName =
      config.type === 'sse-ide' || config.type === 'ws-ide'
        ? config.ideName
        : null
    if (ideClient.type === 'connected') {
      return { status: 'connected', ideName }
    }
    if (ideClient.type === 'pending') {
      return { status: 'pending', ideName }
    }
    return { status: 'disconnected', ideName }
  }, [mcpClients])
}
```

#### 2. IDE 工具函数 (src/utils/ide.ts)
- `detectIDEs(includeInvalid)`: 检测可用的 IDE
- `isJetBrainsIde(ideType)`: 检查是否为 JetBrains IDE
- `isSupportedTerminal()`: 检查是否运行在支持的终端中

#### 3. useIdeSelection (src/hooks/useIdeSelection.ts)
处理 IDE 文本选择信息的 Hook，通过 MCP 客户端通知处理器注册选择变更事件。

## 风险、边界与改进建议

### 潜在风险

1. **竞态条件**
   - 多个 effect 可能同时操作通知系统
   - `hasShownHintRef` 是组件级别的，重新挂载后可能重复显示

2. **性能问题**
   - `detectIDEs(true)` 涉及文件系统操作和进程检测
   - 虽然延迟 3 秒执行，但在某些系统上可能仍然耗时

3. **状态复杂度**
   - 派生状态逻辑复杂，多个条件相互影响
   - 新增状态时需要仔细考虑对其他通知的影响

### 边界情况

1. **多个 IDE**
   - 如果检测到多个 IDE，只显示第一个
   - 用户可能需要手动选择

2. **IDE 快速切换**
   - 如果 IDE 快速连接/断开，可能导致通知闪烁
   - 当前没有防抖处理

3. **远程模式**
   - 所有效果在远程模式下禁用
   - 这是预期行为

4. **计数器重置**
   - `ideHintShownCount` 存储在全局配置中
   - 新设备或清除配置后会重置

### 改进建议

1. **增加防抖处理**
   ```typescript
   // 对于 IDE 状态变化，添加防抖
   const debouncedIdeStatus = useDebounce(ideStatus, 500)
   ```

2. **优化 detectIDEs 调用**
   ```typescript
   // 考虑缓存检测结果
   const cachedIDEs = useRef<DetectedIDEInfo[] | null>(null)
   ```

3. **添加分析事件**
   ```typescript
   logEvent('tengu_ide_hint_shown', { ideName })
   logEvent('tengu_ide_disconnected', { ideName })
   ```

4. **改进错误处理**
   ```typescript
   detectIDEs(true).catch(error => {
     logForDebugging(`[IDEStatus] detectIDEs failed: ${error}`)
   })
   ```

5. **考虑使用状态机**
   - 当前使用多个布尔值表示状态
   - 可以考虑使用更明确的状态机（如 'idle' | 'hint-shown' | 'connected' | 'disconnected'）

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useIDEStatusIndicator.tsx`
- **通知系统**: `src/context/notifications.tsx`
- **IDE 连接状态**: `src/hooks/useIdeConnectionStatus.ts`
- **IDE 选择**: `src/hooks/useIdeSelection.ts`
- **IDE 工具**: `src/utils/ide.ts`
- **全局配置**: `src/utils/config.ts`
- **MCP 类型**: `src/services/mcp/types.js`
- **Ink 组件**: `src/ink.js`
