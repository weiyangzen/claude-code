# vscodeSdkMcp.ts 深度研究文档

## 场景与职责

`vscodeSdkMcp.ts` 是 Claude Code 与 VSCode 扩展之间双向通信的专用模块，实现：

1. **文件更新通知**：当 Claude Code 编辑或写入文件时，通知 VSCode 扩展
2. **实验性功能门控同步**：将 GrowthBook 实验门控状态同步给 VSCode
3. **分析事件接收**：接收 VSCode 扩展发送的分析事件并上报

该模块是 VSCode MCP 服务器（`claude-vscode`）的客户端集成点，仅在 `USER_TYPE === 'ant'`（内部用户）时启用。

## 功能点目的

### 1. 文件更新通知

**目的**：让 VSCode 扩展知道 Claude Code 修改了哪些文件，以便：
- 刷新文件树
- 显示文件变更指示器
- 触发语言服务重新分析

**通知内容**：
- `filePath`：被修改的文件路径
- `oldContent`：修改前的内容（可为 null）
- `newContent`：修改后的内容（可为 null）

**使用场景**：
- `BashTool` 执行后
- `FileEditTool` 执行后
- `FileWriteTool` 执行后
- `print.ts` 中的文件输出

### 2. 实验性功能门控同步

**目的**：让 VSCode 扩展了解当前启用的实验性功能，以便：
- 条件渲染 UI 元素
- 启用/禁用特定功能
- 保持与 Claude Code 一致的用户体验

**同步的门控**：
- `tengu_vscode_review_upsell`：Review 功能推广
- `tengu_vscode_onboarding`： onboarding 流程
- `tengu_quiet_fern`：浏览器支持
- `tengu_vscode_cc_auth`：内置 OAuth 认证
- `tengu_auto_mode_state`：自动模式状态（三态：enabled/disabled/opt-in）

### 3. 分析事件接收

**目的**：接收 VSCode 扩展的用户交互事件，统一上报到分析系统

**事件格式**：
```typescript
{
  method: 'log_event',
  params: {
    eventName: string,
    eventData: Record<string, unknown>
  }
}
```

**事件前缀**：`tengu_vscode_`

## 具体技术实现

### 关键数据结构

```typescript
// VSCode MCP 客户端引用
let vscodeMcpClient: ConnectedMCPServer | null = null

// 日志事件通知 Schema
export const LogEventNotificationSchema = lazySchema(() =>
  z.object({
    method: z.literal('log_event'),
    params: z.object({
      eventName: z.string(),
      eventData: z.object({}).passthrough(),
    }),
  }),
)

// 自动模式状态（镜像 permissionSetup.ts）
type AutoModeEnabledState = 'enabled' | 'disabled' | 'opt-in'
```

### 关键流程

#### notifyVscodeFileUpdated 实现

```typescript
export function notifyVscodeFileUpdated(
  filePath: string,
  oldContent: string | null,
  newContent: string | null,
): void {
  // 仅内部用户启用
  if (process.env.USER_TYPE !== 'ant' || !vscodeMcpClient) {
    return
  }

  // 异步发送通知，不阻塞主流程
  void vscodeMcpClient.client
    .notification({
      method: 'file_updated',
      params: { filePath, oldContent, newContent },
    })
    .catch((error: Error) => {
      // 静默处理错误，不影响主流程
      logForDebugging(
        `[VSCode] Failed to send file_updated notification: ${error.message}`,
      )
    })
}
```

#### setupVscodeSdkMcp 实现

```typescript
export function setupVscodeSdkMcp(sdkClients: MCPServerConnection[]): void {
  // 查找 claude-vscode 客户端
  const client = sdkClients.find(client => client.name === 'claude-vscode')

  if (client && client.type === 'connected') {
    // 存储客户端引用
    vscodeMcpClient = client

    // 1. 注册分析事件处理器
    client.client.setNotificationHandler(
      LogEventNotificationSchema(),
      async notification => {
        const { eventName, eventData } = notification.params
        logEvent(
          `tengu_vscode_${eventName}`,
          eventData as { [key: string]: boolean | number | undefined },
        )
      },
    )

    // 2. 发送实验门控状态
    const gates: Record<string, boolean | string> = {
      tengu_vscode_review_upsell: checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_vscode_review_upsell'),
      tengu_vscode_onboarding: checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_vscode_onboarding'),
      tengu_quiet_fern: getFeatureValue_CACHED_MAY_BE_STALE('tengu_quiet_fern', false),
      tengu_vscode_cc_auth: getFeatureValue_CACHED_MAY_BE_STALE('tengu_vscode_cc_auth', false),
    }
    
    // 添加自动模式状态（如果已知）
    const autoModeState = readAutoModeEnabledState()
    if (autoModeState !== undefined) {
      gates.tengu_auto_mode_state = autoModeState
    }
    
    // 异步发送，不等待响应
    void client.client.notification({
      method: 'experiment_gates',
      params: { gates },
    })
  }
}
```

#### readAutoModeEnabledState 实现

```typescript
function readAutoModeEnabledState(): AutoModeEnabledState | undefined {
  const v = getFeatureValue_CACHED_MAY_BE_STALE<{ enabled?: string }>(
    'tengu_auto_mode_config',
    {},
  )?.enabled
  return v === 'enabled' || v === 'disabled' || v === 'opt-in' ? v : undefined
}
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/services/mcp/types.ts` | `ConnectedMCPServer`, `MCPServerConnection` 类型 |
| `src/services/analytics/growthbook.ts` | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE`, `getFeatureValue_CACHED_MAY_BE_STALE` |
| `src/services/analytics/index.ts` | `logEvent` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/lazySchema.ts` | `lazySchema` |

### 调用方

| 文件 | 调用函数 |
|------|----------|
| `src/cli/print.ts` | `notifyVscodeFileUpdated` |
| `src/tools/BashTool/BashTool.tsx` | `notifyVscodeFileUpdated` |
| `src/tools/FileEditTool/FileEditTool.ts` | `notifyVscodeFileUpdated` |
| `src/tools/FileWriteTool/FileWriteTool.ts` | `notifyVscodeFileUpdated` |
| `src/utils/fileHistory.ts` | `notifyVscodeFileUpdated` |

### 被调用方

| 函数 | 被调用文件 |
|------|------------|
| `setupVscodeSdkMcp` | `print.ts`（通过 `setupSdkMcpClients` 调用） |
| `notifyVscodeFileUpdated` | 多个工具文件 |

## 依赖与外部交互

### 外部系统交互

1. **MCP SDK**：通过 `Client.notification()` 发送通知
2. **GrowthBook**：获取实验门控状态
3. **分析系统**：`logEvent` 上报 VSCode 事件

### 配置依赖

- `USER_TYPE`：必须为 `'ant'` 才启用
- 无其他环境变量依赖

## 风险、边界与改进建议

### 风险点

1. **单例状态管理**
   - `vscodeMcpClient` 是模块级变量，无并发保护
   - 如果 `setupVscodeSdkMcp` 被多次调用，可能覆盖之前的引用
   - 建议：添加初始化状态检查

2. **错误静默处理**
   - 通知发送失败仅记录调试日志，调用方无法感知
   - 在某些场景下可能需要重试机制

3. **功能门控缓存**
   - 使用 `_CACHED_MAY_BE_STALE` 后缀的 API，数据可能过期
   - 对于需要实时一致性的场景可能有问题

4. **硬编码服务器名称**
   - `'claude-vscode'` 名称硬编码，如果 VSCode 扩展改名会失效
   - 建议：提取为常量或配置

### 边界情况

1. **客户端未连接**
   - `notifyVscodeFileUpdated` 检查 `vscodeMcpClient` 是否存在
   - 如果 VSCode 扩展未启动或连接失败，通知会被静默丢弃

2. **通知发送失败**
   - 使用 `.catch()` 捕获错误，不影响主流程
   - 但可能导致 VSCode 扩展状态与 Claude Code 不一致

3. **自动模式状态缺失**
   - `readAutoModeEnabledState` 可能返回 `undefined`
   - 此时不发送 `tengu_auto_mode_state`，VSCode 应处理缺失情况

### 改进建议

1. **添加初始化状态检查**
   ```typescript
   let isInitialized = false
   
   export function setupVscodeSdkMcp(sdkClients: MCPServerConnection[]): void {
     if (isInitialized) return
     isInitialized = true
     // ... 原有逻辑
   }
   ```

2. **提取常量**
   ```typescript
   const VSCODE_MCP_SERVER_NAME = 'claude-vscode'
   ```

3. **添加重试机制**
   ```typescript
   async function notifyWithRetry(notification: unknown, maxRetries = 3): Promise<void> {
     for (let i = 0; i < maxRetries; i++) {
       try {
         await vscodeMcpClient!.client.notification(notification)
         return
       } catch (error) {
         if (i === maxRetries - 1) throw error
         await sleep(100 * Math.pow(2, i))
       }
     }
   }
   ```

4. **添加健康检查**
   ```typescript
   export function isVscodeMcpConnected(): boolean {
     return vscodeMcpClient !== null && vscodeMcpClient.type === 'connected'
   }
   ```

5. **支持动态门控更新**
   - 当前仅在 `setupVscodeSdkMcp` 时发送门控状态
   - 建议：监听门控变化，实时同步给 VSCode
