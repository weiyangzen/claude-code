# wrapper.tsx 深度研究文档

## 1. 场景与职责

### 1.1 定位
`wrapper.tsx` 是 Claude Code CLI 中 **Computer Use MCP (代号 Chicago/CHICAGO_MCP)** 功能的核心适配层。它作为 MCP 工具调用与底层计算机控制实现之间的**薄适配器(thin adapter)**，负责将 `ToolUseContext` 绑定到 `@ant/computer-use-mcp` 包的 `bindSessionContext`。

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **工具调用拦截** | 为所有 `mcp__computer-use__*` 工具提供 `.call()` 覆盖 |
| **会话上下文绑定** | 构建并缓存 `ComputerUseSessionContext`，连接 AppState 与 CU 包 |
| **渲染覆盖** | 提供工具使用/结果消息的自定义 React 渲染 |
| **锁管理协调** | 与 `computerUseLock.ts` 协作，确保单会话独占 |
| **权限对话框** | 通过 `setToolJSX` 渲染 `ComputerUseApproval` 组件处理用户授权 |
| **生命周期管理** | 配合 `cleanup.ts` 在 turn 结束时释放资源 |

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        MCP Tool Layer                           │
│  (src/services/mcp/client.ts - fetchToolsForClient)            │
└──────────────────────┬──────────────────────────────────────────┘
                       │ .call() override
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    wrapper.tsx (本文件)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ getComputer │  │ buildSession│  │ runPermissionDialog     │  │
│  │ UseMCPTool  │  │ Context     │  │ (ComputerUseApproval)   │  │
│  │ Overrides   │  │             │  │                         │  │
│  └──────┬──────┘  └──────┬──────┘  └─────────────────────────┘  │
│         │                │                                       │
│         └────────────────┼───────────────────────────────────────┘
                          │
                          ▼ bindSessionContext
┌─────────────────────────────────────────────────────────────────┐
│              @ant/computer-use-mcp (npm package)                │
│                   (bindSessionContext)                          │
└──────────────────────┬──────────────────────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│ hostAdapter  │ │ gates.ts │ │ toolRendering│
│ (executor)   │ │ (config) │ │ (UI)         │
└──────────────┘ └──────────┘ └──────────────┘
```

## 2. 功能点目的

### 2.1 模块级状态管理

```typescript
let binding: Binding | undefined;
let currentToolUseContext: ToolUseContext | undefined;
```

**设计意图**：
- `binding`: 进程生命周期单例，缓存 `bindSessionContext` 结果，避免重复初始化
- `currentToolUseContext`: 每次工具调用时更新，使 `ctx` 回调能访问当前调用的上下文

**例外说明**：代码注释明确说明这是 "deliberate exception to the no-module-scope-state rule"，因为 dispatcher closure 必须跨调用保持截图 blob 状态。

### 2.2 会话上下文构建 (buildSessionContext)

构建 `ComputerUseSessionContext` 对象，包含以下回调类别：

#### 2.2.1 读取状态回调
| 回调 | 用途 | 数据来源 |
|------|------|----------|
| `getAllowedApps` | 获取已授权应用列表 | `computerUseMcpState.allowedApps` |
| `getGrantFlags` | 获取权限标志 | `computerUseMcpState.grantFlags` |
| `getUserDeniedBundleIds` | 用户拒绝的应用 | 固定返回 `[]` (cc-2 无设置页) |
| `getSelectedDisplayId` | 当前选定显示器 | `computerUseMcpState.selectedDisplayId` |
| `getDisplayPinnedByModel` | 显示器是否被模型固定 | `computerUseMcpState.displayPinnedByModel` |
| `getDisplayResolvedForApps` | 显示器解析的缓存键 | `computerUseMcpState.displayResolvedForApps` |
| `getLastScreenshotDims` | 最后截图尺寸 | `computerUseMcpState.lastScreenshotDims` |

#### 2.2.2 写入状态回调
| 回调 | 触发场景 | 操作 |
|------|----------|------|
| `onPermissionRequest` | 需要用户授权时 | 显示 `ComputerUseApproval` 对话框 |
| `onAllowedAppsChanged` | 授权应用列表变更 | 更新 AppState 中的 `allowedApps` 和 `grantFlags` |
| `onAppsHidden` | 应用被隐藏时 | 累加到 `hiddenDuringTurn` Set |
| `onResolvedDisplayUpdated` | 显示器解析更新时 | 更新 `selectedDisplayId`，清除 pin 状态 |
| `onDisplayPinned` | 模型切换显示器时 | 设置/清除 pin 状态 |
| `onDisplayResolvedForApps` | 显示器解析完成时 | 缓存解析键 |
| `onScreenshotCaptured` | 截图完成时 | 更新截图尺寸信息 |

#### 2.2.3 锁管理回调
| 回调 | 用途 |
|------|------|
| `checkCuLock` | 检查文件锁状态 (free/held_by_self/blocked) |
| `acquireCuLock` | 获取独占锁，注册 ESC 热键，发送进入通知 |
| `formatLockHeldMessage` | 格式化锁被占用的错误消息 |

### 2.3 权限对话框 (runPermissionDialog)

```typescript
async function runPermissionDialog(req: CuPermissionRequest): Promise<CuPermissionResponse>
```

**流程**：
1. 检查 `setToolJSX` 是否存在（非交互式会话会被 gate 排除）
2. 创建 Promise，在 `ComputerUseApproval` 的 `onDone` 回调中 resolve
3. 监听 `abortController.signal`，用户按 Ctrl+C 时 reject
4. 使用 `setToolJSX` 渲染对话框，阻塞工具调用直到用户响应
5. finally 中清除对话框 (`setToolJSX(null)`)

**参考模式**：与 `spawnMultiAgent.ts:419-436` 的 `It2SetupPrompt` 模式一致。

### 2.4 工具调用覆盖 (getComputerUseMCPToolOverrides)

返回包含以下属性的对象：

```typescript
{
  ...getComputerUseMCPRenderingOverrides(toolName),  // 来自 toolRendering.tsx
  call: async (args, context) => { /* ... */ }       // 自定义调用逻辑
}
```

**调用流程**：
1. 更新 `currentToolUseContext` 为当前上下文
2. 获取或创建 `binding` (通过 `getOrBind()`)
3. 调用 `dispatch(toolName, args)` 执行实际工具逻辑
4. 转换 MCP content blocks 为 Anthropic API blocks:
   - `image` → `{type: 'image', source: {type: 'base64', media_type, data}}`
   - `text` → `{type: 'text', text}`
5. 记录 telemetry 错误信息到调试日志

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 Binding 类型
```typescript
type Binding = {
  ctx: ComputerUseSessionContext;           // 会话上下文
  dispatch: (name: string, args: unknown) => Promise<CuCallToolResult>;
};
```

#### 3.1.2 ComputerUseSessionContext (来自 @ant/computer-use-mcp)
```typescript
type ComputerUseSessionContext = {
  // 读取
  getAllowedApps: () => Array<{bundleId: string, displayName: string, grantedAt: number}>;
  getGrantFlags: () => CuGrantFlags;
  getUserDeniedBundleIds: () => string[];
  getSelectedDisplayId: () => number | undefined;
  getDisplayPinnedByModel: () => boolean;
  getDisplayResolvedForApps: () => string | undefined;
  getLastScreenshotDims: () => ScreenshotDims | undefined;
  
  // 写入
  onPermissionRequest: (req: CuPermissionRequest, dialogSignal: AbortSignal) => Promise<CuPermissionResponse>;
  onAllowedAppsChanged: (apps: AllowedApp[], flags: CuGrantFlags) => void;
  onAppsHidden: (bundleIds: string[]) => void;
  onResolvedDisplayUpdated: (id: number) => void;
  onDisplayPinned: (id?: number) => void;
  onDisplayResolvedForApps: (key: string) => void;
  onScreenshotCaptured: (dims: ScreenshotDims) => void;
  
  // 锁
  checkCuLock: () => Promise<{holder: string | undefined, isSelf: boolean}>;
  acquireCuLock: () => Promise<void>;
  formatLockHeldMessage: (holder: string) => string;
};
```

#### 3.1.3 AppState 中的 computerUseMcpState
```typescript
computerUseMcpState?: {
  allowedApps?: readonly {bundleId: string, displayName: string, grantedAt: number}[];
  grantFlags?: {clipboardRead: boolean, clipboardWrite: boolean, systemKeyCombos: boolean};
  lastScreenshotDims?: {width, height, displayWidth, displayHeight, displayId?, originX?, originY?};
  hiddenDuringTurn?: ReadonlySet<string>;
  selectedDisplayId?: number;
  displayPinnedByModel?: boolean;
  displayResolvedForApps?: string;
}
```

### 3.2 关键流程

#### 3.2.1 首次工具调用初始化流程

```
getComputerUseMCPToolOverrides(toolName).call(args, context)
    │
    ▼
currentToolUseContext = context  // 更新当前上下文
    │
    ▼
getOrBind()
    │
    ├─ binding 已存在? ──→ 直接返回
    │
    └─ binding 不存在?
        │
        ▼
    buildSessionContext()  // 构建完整上下文
        │
        ▼
    bindSessionContext(hostAdapter, coordinateMode, ctx)
        │
        ▼
    返回 {ctx, dispatch} 并缓存到 binding
```

#### 3.2.2 权限请求流程

```
@ant/computer-use-mcp 内部触发权限检查
    │
    ▼
ctx.onPermissionRequest(req, dialogSignal)
    │
    ▼
runPermissionDialog(req)
    │
    ▼
检查 abortController.signal.aborted (已中止?)
    │
    ▼
setToolJSX({
  jsx: <ComputerUseApproval request={req} onDone={resolve} />,
  shouldHidePromptInput: true
})
    │
    ▼
等待用户交互 → onDone(resp) → resolve(resp)
    │
    ▼
finally: setToolJSX(null)  // 清除对话框
```

#### 3.2.3 锁获取流程 (acquireCuLock)

```
tryAcquireComputerUseLock()
    │
    ├─ kind: 'blocked' → throw Error(formatLockHeld(by))
    │
    └─ kind: 'acquired'
        │
        ├─ fresh: true (首次获取)
        │   │
        │   ▼
        │ registerEscHotkey(() => abortController.abort())
        │   │
        │   ▼
        │ sendOSNotification({
        │   message: 'Claude is using your computer · press Esc to stop',
        │   notificationType: 'computer_use_enter'
        │ })
        │
        └─ fresh: false (重入) → 无额外操作
```

### 3.3 状态更新优化

所有 `on*` 回调都实现了**引用相等优化**：

```typescript
// 示例: onAllowedAppsChanged
onAllowedAppsChanged: (apps, flags) => tuc().setAppState(prev => {
  const cu = prev.computerUseMcpState;
  const prevApps = cu?.allowedApps;
  const prevFlags = cu?.grantFlags;
  
  // 检查是否真的变化
  const sameApps = prevApps?.length === apps.length && 
                   apps.every((a, i) => prevApps[i]?.bundleId === a.bundleId);
  const sameFlags = /* ... */;
  
  // 无变化返回原对象，避免 React 重渲染
  return sameApps && sameFlags ? prev : {
    ...prev,
    computerUseMcpState: { ...cu, allowedApps: [...apps], grantFlags: flags }
  };
})
```

## 4. 关键代码路径与文件引用

### 4.1 直接依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `@ant/computer-use-mcp` | `bindSessionContext`, `ComputerUseSessionContext`, `CuCallToolResult`, `CuPermissionRequest`, `CuPermissionResponse`, `DEFAULT_GRANT_FLAGS`, `ScreenshotDims` | 核心 CU 包接口 |
| `react` | `React` | 创建 `ComputerUseApproval` 元素 |
| `../../bootstrap/state.js` | `getSessionId` | 锁标识 |
| `../../components/permissions/ComputerUseApproval/ComputerUseApproval.js` | `ComputerUseApproval` | 权限对话框组件 |
| `../../Tool.js` | `Tool`, `ToolUseContext` | 类型定义 |
| `../debug.js` | `logForDebugging` | 调试日志 |
| `./computerUseLock.js` | `checkComputerUseLock`, `tryAcquireComputerUseLock` | 文件锁 |
| `./escHotkey.ts` | `registerEscHotkey` | ESC 热键注册 |
| `./gates.ts` | `getChicagoCoordinateMode` | 坐标模式配置 |
| `./hostAdapter.js` | `getComputerUseHostAdapter` | 主机适配器 |
| `./toolRendering.tsx` | `getComputerUseMCPRenderingOverrides` | 渲染覆盖 |

### 4.2 调用方文件

| 文件 | 调用方式 | 场景 |
|------|----------|------|
| `src/services/mcp/client.ts:1986` | `computerUseWrapper!().getComputerUseMCPToolOverrides(tool.name)` | MCP 工具获取时注入覆盖 |

### 4.3 相关支撑文件

| 文件 | 职责 |
|------|------|
| `src/utils/computerUse/common.ts` | 常量定义 (`COMPUTER_USE_MCP_SERVER_NAME`, `CLI_HOST_BUNDLE_ID`, `CLI_CU_CAPABILITIES`) |
| `src/utils/computerUse/setup.ts` | MCP 配置初始化，构建工具列表 |
| `src/utils/computerUse/mcpServer.ts` | 创建 in-process MCP 服务器 |
| `src/utils/computerUse/hostAdapter.ts` | 实现 `ComputerUseHostAdapter` 接口 |
| `src/utils/computerUse/executor.ts` | CLI `ComputerExecutor` 实现，包装原生模块 |
| `src/utils/computerUse/gates.ts` | GrowthBook 配置读取 (`tengu_malort_pedway`) |
| `src/utils/computerUse/cleanup.ts` | Turn 结束清理 (unhide apps, release lock) |
| `src/utils/computerUse/computerUseLock.ts` | 文件锁实现 (O_EXCL 原子创建) |
| `src/utils/computerUse/escHotkey.ts` | CGEventTap ESC 热键 |
| `src/utils/computerUse/drainRunLoop.ts` | CFRunLoop 泵，解决 libuv 下主队列阻塞 |
| `src/utils/computerUse/swiftLoader.ts` | 懒加载 `@ant/computer-use-swift` |
| `src/utils/computerUse/inputLoader.ts` | 懒加载 `@ant/computer-use-input` |
| `src/utils/computerUse/toolRendering.tsx` | 工具消息渲染覆盖 |
| `src/utils/computerUse/appNames.ts` | 应用名称过滤和净化 |

### 4.4 关键代码片段

#### 4.4.1 工具调用转换逻辑
```typescript
// wrapper.tsx:268-278
const data = Array.isArray(result.content) ? result.content.map(item => 
  item.type === 'image' ? {
    type: 'image' as const,
    source: {
      type: 'base64' as const,
      media_type: item.mimeType ?? 'image/jpeg',
      data: item.data
    }
  } : {
    type: 'text' as const,
    text: item.type === 'text' ? item.text : ''
  }
) : result.content;
```

#### 4.4.2 锁状态检查
```typescript
// wrapper.tsx:181-200
checkCuLock: async () => {
  const c = await checkComputerUseLock();
  switch (c.kind) {
    case 'free': return { holder: undefined, isSelf: false };
    case 'held_by_self': return { holder: getSessionId(), isSelf: true };
    case 'blocked': return { holder: c.by, isSelf: false };
  }
}
```

#### 4.4.3 获取或绑定缓存
```typescript
// wrapper.tsx:230-238
function getOrBind(): Binding {
  if (binding) return binding;
  const ctx = buildSessionContext();
  binding = {
    ctx,
    dispatch: bindSessionContext(getComputerUseHostAdapter(), getChicagoCoordinateMode(), ctx)
  };
  return binding;
}
```

## 5. 依赖与外部交互

### 5.1 外部 npm 包

| 包名 | 用途 |
|------|------|
| `@ant/computer-use-mcp` | 核心 Computer Use MCP 实现，提供 `bindSessionContext` 和工具定义 |
| `@ant/computer-use-swift` | macOS Swift 原生模块，截图、应用管理、TCC 检查 |
| `@ant/computer-use-input` | Rust/enigo 原生模块，鼠标键盘控制 |
| `react` | UI 渲染 |

### 5.2 原生模块交互

通过 `hostAdapter.ts` 和 `executor.ts` 间接使用：

```
wrapper.tsx
    │
    ▼ bindSessionContext
hostAdapter.ts (getComputerUseHostAdapter)
    │
    ├─ executor: createCliExecutor()
    │       │
    │       ▼
    │   executor.ts
    │       │
    │       ├─ @ant/computer-use-swift (截图、应用、TCC)
    │       └─ @ant/computer-use-input (鼠标、键盘)
    │
    └─ ensureOsPermissions() → swift.tcc.checkAccessibility/ScreenRecording()
```

### 5.3 AppState 交互

```
wrapper.tsx 回调
    │
    ▼ setAppState / getAppState
AppStateStore (src/state/AppStateStore.ts)
    │
    ▼ computerUseMcpState 字段
    ├─ allowedApps: 授权应用列表
    ├─ grantFlags: 权限标志
    ├─ lastScreenshotDims: 截图尺寸
    ├─ hiddenDuringTurn: 本轮隐藏的应用
    ├─ selectedDisplayId: 选定显示器
    ├─ displayPinnedByModel: 显示器是否被固定
    └─ displayResolvedForApps: 显示器解析缓存
```

### 5.4 生命周期钩子

| 钩子点 | 文件 | 操作 |
|--------|------|------|
| Turn 自然结束 | `src/query/stopHooks.ts:164-173` | `cleanupComputerUseAfterTurn()` |
| 流式中止 | `src/query.ts:1033-1038` | `cleanupComputerUseAfterTurn()` |
| 工具执行中止 | `src/query.ts:1489-1494` | `cleanupComputerUseAfterTurn()` |
| 进程退出 | `src/utils/computerUse/computerUseLock.ts:94-99` | `registerCleanup(releaseComputerUseLock)` |

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 模块级状态风险
```typescript
// wrapper.tsx:49-50
let binding: Binding | undefined;
let currentToolUseContext: ToolUseContext | undefined;
```

**风险**：模块级状态违反 "no-module-scope-state" 规则，测试需要串行运行或注入缓存。

**缓解**：代码注释已说明这是 deliberate exception，因为 dispatcher closure 必须跨调用保持截图 blob。

#### 6.1.2 竞态条件
- **锁获取竞态**：`checkCuLock` 和 `acquireCuLock` 之间可能有其他进程获取锁
  - 缓解：`tryAcquireComputerUseLock` 使用 O_EXCL 原子创建，失败时抛出

- **截图 blob 跨调用依赖**：blob 存储在 dispatcher closure 中，依赖进程级缓存
  - 风险：进程重启后 blob 丢失，但模型侧可能有缓存

#### 6.1.3 非交互式会话
```typescript
// wrapper.tsx:299-305
if (!setToolJSX) {
  // Shouldn't happen — main.tsx gate excludes non-interactive
  return { granted: [], denied: [], flags: DEFAULT_GRANT_FLAGS };
}
```

**风险**：如果 gate 失效，非交互式会话会静默拒绝所有权限请求。

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 多显示器，目标显示器断开 | `onResolvedDisplayUpdated` 清除 pin，回退到主显示器 |
| 用户按 ESC | `registerEscHotkey` 回调触发 `abortController.abort()` |
| 用户按 Ctrl+C | `abortController.signal` 触发，权限对话框 reject |
| 锁被其他会话持有 | 返回格式化错误消息，提示等待或运行 /exit |
|  stale lock (PID 不存在) | `checkComputerUseLock` 自动 unlink 并返回 free |
| 重入获取锁 | `fresh: false`，不重复发送通知或注册热键 |

### 6.3 改进建议

#### 6.3.1 测试性改进
```typescript
// 建议：添加测试注入点
export function __resetBindingForTest(): void {
  binding = undefined;
  currentToolUseContext = undefined;
}
```

#### 6.3.2 错误处理增强
当前 `runPermissionDialog` 在 `setToolJSX` 不存在时静默返回拒绝响应：

```typescript
// 建议：添加警告日志
if (!setToolJSX) {
  logForDebugging('[ComputerUse] setToolJSX missing in permission dialog', { level: 'warn' });
  return DENY_ALL_RESPONSE;
}
```

#### 6.3.3 状态更新合并
多个 `on*` 回调可能在同一 tick 触发多次 `setAppState`，考虑使用队列合并：

```typescript
// 当前：每个回调独立调用 setAppState
// 建议：使用 pendingUpdates + requestAnimationFrame 合并
```

#### 6.3.4 锁超时处理
当前锁没有自动超时机制，如果进程崩溃且 cleanup 未运行，锁可能残留：

```typescript
// 已在 computerUseLock.ts 中处理：
// - 启动时检查 PID 是否存在 (isProcessRunning)
// - stale lock 自动清理
```

#### 6.3.5 类型安全
`computerUseMcpState` 在 AppState 中 inlined 定义，与 `@ant/computer-use-mcp` 包类型保持结构兼容：

```typescript
// AppStateStore.ts:257-258
// Types inlined (not imported from @ant/computer-use-mcp/types) 
// so external typecheck passes without the ant-scoped dep resolved.
```

建议：考虑添加结构兼容性测试，防止包升级导致类型漂移。

### 6.4 监控与调试

| 日志点 | 级别 | 内容 |
|--------|------|------|
| `[cu-esc] registered` | debug | ESC 热键注册成功 |
| `[cu-esc] user escape, aborting turn` | debug | 用户按 ESC |
| `[Computer Use MCP] ${toolName} error_kind=${...}` | debug | 工具执行错误 |
| `Released computer-use lock` | debug | 锁释放 |
| `Recovering stale computer-use lock...` | debug | 清理过期锁 |

---

**文档版本**: 基于代码提交 2026-04-01  
**作者**: Kimi Code CLI  
**范围**: src/utils/computerUse/wrapper.tsx 及其直接依赖上下文
