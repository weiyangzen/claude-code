# ComputerUseApproval.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位

`ComputerUseApproval.tsx` 是 Claude Code CLI 中 **Computer Use (CU) 功能** 的核心权限审批 UI 组件。它负责在终端界面中渲染交互式对话框，让用户审批 Claude 对 macOS 应用程序的自动化控制权限。

### 1.2 使用场景

1. **首次使用 Computer Use 功能时**：当用户首次触发需要控制其他应用的操作（如点击、输入、截图等），系统会弹出权限审批对话框
2. **macOS 系统权限缺失时**：当 Accessibility 或 Screen Recording 权限未授予时，显示 TCC (Transparency, Consent, and Control) 权限引导面板
3. **应用权限审批**：当 Claude 请求控制特定应用程序时，显示应用列表供用户选择允许/拒绝

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                     Claude Code CLI (Terminal)                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────────────────────────┐ │
│  │  wrapper.tsx    │───▶│    ComputerUseApproval.tsx          │ │
│  │  (调用方/封装层) │    │    (本组件 - 权限审批UI)             │ │
│  └─────────────────┘    └─────────────────────────────────────┘ │
│           │                            │                        │
│           ▼                            ▼                        │
│  ┌─────────────────┐    ┌─────────────────────────────────────┐ │
│  │ @ant/computer-  │    │  Dialog.tsx / Select.tsx (UI组件)   │ │
│  │ use-mcp (MCP包) │    │  execFileNoThrow (系统调用)         │ │
│  └─────────────────┘    └─────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 功能点目的

### 2.1 双面板调度器 (Two-panel Dispatcher)

组件核心逻辑是根据 `request.tccState` 的存在与否，决定显示哪个面板：

| 条件 | 显示面板 | 目的 |
|------|----------|------|
| `request.tccState` 存在 | `ComputerUseTccPanel` | 处理 macOS 系统级权限缺失 |
| `request.tccState` 不存在 | `ComputerUseAppListPanel` | 处理应用级权限审批 |

### 2.2 TCC 权限面板 (`ComputerUseTccPanel`)

**目的**：引导用户授予 macOS 系统级权限

**功能**：
- 检测并显示 Accessibility 权限状态
- 检测并显示 Screen Recording 权限状态
- 提供快捷打开系统设置的功能
- 支持重试检测

**交互选项**：
- `open_accessibility`: 打开系统设置 → 辅助功能
- `open_screen_recording`: 打开系统设置 → 屏幕录制
- `retry`: 重新检测权限状态

### 2.3 应用列表面板 (`ComputerUseAppListPanel`)

**目的**：让用户选择允许 Claude 控制哪些应用程序

**功能**：
- 显示请求控制的应用列表
- 标记已安装/未安装的应用
- 标记已授权的应用
- 显示高风险应用警告（Sentinel 分类）
- 显示额外请求的权限标志（clipboardRead, clipboardWrite, systemKeyCombos）
- 显示将被隐藏的其他应用数量

**交互选项**：
- `allow_all`: 允许控制所有选中的应用（当前会话）
- `deny`: 拒绝并告诉 Claude 如何不同地操作

---

## 3. 具体技术实现

### 3.1 关键数据结构与类型

#### 3.1.1 组件 Props

```typescript
type ComputerUseApprovalProps = {
  request: CuPermissionRequest;    // 权限请求数据
  onDone: (response: CuPermissionResponse) => void;  // 完成回调
};
```

#### 3.1.2 权限请求数据结构 (`CuPermissionRequest`)

```typescript
interface CuPermissionRequest {
  // TCC 状态（可选）
  tccState?: {
    accessibility: boolean;      // 辅助功能权限状态
    screenRecording: boolean;    // 屏幕录制权限状态
  };
  
  // 应用列表
  apps: Array<{
    requestedName: string;       // 请求的应用名称
    resolved?: {                 // 解析后的应用信息
      bundleId: string;
      displayName: string;
    };
    alreadyGranted: boolean;     // 是否已授权
  }>;
  
  // 请求的权限标志
  requestedFlags: {
    clipboardRead: boolean;
    clipboardWrite: boolean;
    systemKeyCombos: boolean;
  };
  
  // 其他元数据
  reason?: string;               // 请求原因
  willHide?: Array<{...}>;       // 将被隐藏的其他应用
}
```

#### 3.1.3 权限响应数据结构 (`CuPermissionResponse`)

```typescript
interface CuPermissionResponse {
  granted: Array<{
    bundleId: string;
    displayName: string;
    grantedAt: number;           // 授权时间戳
  }>;
  denied: Array<{
    bundleId: string;
    reason: 'user_denied' | 'not_installed';
  }>;
  flags: GrantFlags;             // 实际授予的权限标志
}
```

#### 3.1.4 Sentinel 警告映射

```typescript
const SENTINEL_WARNING: Record<SentinelCategory, string> = {
  shell: 'equivalent to shell access',           // 终端类应用
  filesystem: 'can read/write any file',         // 文件系统访问
  system_settings: 'can change system settings', // 系统设置
};
```

### 3.2 关键流程

#### 3.2.1 TCC 面板处理流程

```
用户选择选项
    │
    ├──▶ "Open System Settings → Accessibility"
    │       └── execFileNoThrow("open", ["x-apple.systempreferences:...Privacy_Accessibility"])
    │
    ├──▶ "Open System Settings → Screen Recording"
    │       └── execFileNoThrow("open", ["x-apple.systempreferences:...Privacy_ScreenCapture"])
    │
    └──▶ "Try again"
            └── onDone(DENY_ALL_RESPONSE)  // 返回空授权，让外层重试
```

#### 3.2.2 应用列表面板处理流程

```
初始化
    │
    ├──▶ 创建 checked Set（默认选中所有未授权但已安装的应用）
    │
    ├──▶ 过滤出请求的权限标志键
    │
    └──▶ 构建选项列表（allow_all, deny）
    
用户交互
    │
    ├──▶ 切换应用选中状态（通过复选框）
    │
    └──▶ 选择选项
            │
            ├──▶ "Allow for this session (N apps)"
            │       └── respond(true)
            │           ├── 构建 granted 列表（选中的已安装应用）
            │           ├── 构建 denied 列表（未选中或未安装的）
            │           ├── 合并权限标志
            │           └── onDone({granted, denied, flags})
            │
            └──▶ "Deny, and tell Claude what to do differently (esc)"
                    └── respond(false)
                        └── onDone(DENY_ALL_RESPONSE)
```

### 3.3 渲染逻辑详解

#### 3.3.1 应用列表渲染

对于每个应用，根据状态渲染不同 UI：

```typescript
// 未安装的应用
<Text dimColor>○ {requestedName} (not installed)</Text>

// 已授权的应用
<Text dimColor>✓ {displayName} (already granted)</Text>

// 待审批的应用
<Box>
  <Text>{isChecked ? ◉ : ○} {displayName}</Text>
  {sentinel && <Text bold>⚠ {SENTINEL_WARNING[sentinel]}</Text>}
</Box>
```

#### 3.3.2 权限标志显示

```typescript
const ALL_FLAG_KEYS = ["clipboardRead", "clipboardWrite", "systemKeyCombos"];
const requestedFlagKeys = ALL_FLAG_KEYS.filter(k => request.requestedFlags[k]);

// 渲染为：
// Also requested:
//   · clipboardRead
//   · systemKeyCombos
```

### 3.4 系统调用

#### 3.4.1 打开系统设置

```typescript
// 辅助功能设置
execFileNoThrow("open", [
  "x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility"
], { useCwd: false });

// 屏幕录制设置
execFileNoThrow("open", [
  "x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture"
], { useCwd: false });
```

使用 `useCwd: false` 避免循环依赖问题（`getCwd()` 初始化期间调用）。

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 职责 |
|----------|------|
| `src/components/permissions/ComputerUseApproval/ComputerUseApproval.tsx` | **本组件** - 权限审批 UI |
| `src/utils/computerUse/wrapper.tsx` | 调用方 - 封装 MCP 调用，管理权限对话框生命周期 |
| `src/utils/computerUse/hostAdapter.ts` | 主机适配器 - 提供系统权限检查 |
| `src/utils/computerUse/executor.ts` | 执行器 - 实际执行 Computer Use 操作 |

### 4.2 依赖的 UI 组件

| 文件路径 | 用途 |
|----------|------|
| `src/components/design-system/Dialog.tsx` | 对话框容器组件 |
| `src/components/CustomSelect/select.tsx` | 选择器组件（Select） |
| `src/ink.js` | Ink 渲染库（Box, Text） |

### 4.3 工具函数

| 文件路径 | 用途 |
|----------|------|
| `src/utils/execFileNoThrow.ts` | 安全执行系统命令 |
| `src/utils/stringUtils.ts` | `plural()` 单复数处理 |

### 4.4 外部包依赖

| 包名 | 用途 |
|------|------|
| `@ant/computer-use-mcp` | MCP 协议类型定义和核心逻辑 |
| `@ant/computer-use-mcp/sentinelApps` | `getSentinelCategory()` 高风险应用分类 |
| `figures` | 终端符号（✓, ✗, ○, ◉, ⚠） |
| `react` | React 运行时 |

### 4.5 调用链

```
用户触发 Computer Use 工具
    │
    ▼
@ant/computer-use-mcp 内部逻辑
    │
    ├──▶ 检查 TCC 权限 ──▶ hostAdapter.ensureOsPermissions()
    │
    └──▶ 需要用户审批 ──▶ sessionContext.onPermissionRequest()
                              │
                              ▼
                    wrapper.tsx: runPermissionDialog()
                              │
                              ├──▶ setToolJSX() 渲染 ComputerUseApproval
                              │           │
                              │           ▼
                              │   ComputerUseApproval.tsx
                              │   (本组件渲染对话框)
                              │           │
                              │           ▼
                              │   用户交互完成
                              │           │
                              │           ▼
                              │   onDone(response)
                              │           │
                              ▼           │
                    Promise.resolve(response)
                              │
                              ▼
                    @ant/computer-use-mcp 继续执行
```

---

## 5. 依赖与外部交互

### 5.1 与 MCP 包的交互

组件通过 `CuPermissionRequest` 和 `CuPermissionResponse` 与 `@ant/computer-use-mcp` 包进行数据交换：

**输入（来自 MCP 包）**：
- 请求控制的应用列表
- TCC 权限状态
- 请求的权限标志
- 操作原因

**输出（返回 MCP 包）**：
- 用户授权的应用列表
- 被拒绝的应用及原因
- 实际授予的权限标志

### 5.2 与系统设置的交互

通过 `execFileNoThrow` 调用 `open` 命令打开系统偏好设置：

```
x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility
x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture
```

### 5.3 与 Ink 渲染库的交互

使用 Ink（React for terminals）进行终端 UI 渲染：

- `Box`: 布局容器（flexDirection, padding, gap）
- `Text`: 文本渲染（dimColor, bold, color）
- `Select`: 交互式选择器
- `Dialog`: 对话框容器

### 5.4 与 AppState 的交互

通过 `wrapper.tsx` 中的 `onAllowedAppsChanged` 回调，将用户选择持久化到应用状态：

```typescript
onAllowedAppsChanged: (apps, flags) => tuc().setAppState(prev => {
  // 更新 computerUseMcpState.allowedApps
  // 更新 computerUseMcpState.grantFlags
})
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 权限提升风险

**风险**：Sentinel 分类的应用（如 Terminal、Finder）具有高风险权限：
- `shell`: 等同于 shell 访问权限
- `filesystem`: 可读写任意文件
- `system_settings`: 可更改系统设置

**缓解措施**：
- 在 UI 中显示警告图标和描述性文字
- 用户必须显式选择允许

#### 6.1.2 竞态条件风险

**风险**：`execFileNoThrow` 打开系统设置是异步的，用户可能在权限实际授予前点击"Try again"。

**当前处理**：
- 返回 `DENY_ALL_RESPONSE` 让外层逻辑重试
- 用户需要手动确认权限已授予后再重试

#### 6.1.3 终端检测限制

**风险**：`getTerminalBundleId()` 依赖 `__CFBundleIdentifier` 环境变量，在 SSH 或某些终端配置下可能检测失败。

**影响**：
- 终端可能出现在截图中
- 终端窗口可能被意外隐藏

### 6.2 边界情况

#### 6.2.1 未安装应用

当请求的应用未安装时：
- 显示 `(not installed)` 标记
- 自动归类到 `denied` 列表，原因标记为 `"not_installed"`

#### 6.2.2 已授权应用

当应用已在当前会话中授权：
- 显示 `(already granted)` 标记
- 从默认选中集合中排除

#### 6.2.3 空应用列表

当 `request.apps` 为空数组时：
- `checked.size` 为 0
- 显示 "Allow for this session (0 apps)"
- 用户仍可选择允许（授予权限标志但无应用）

#### 6.2.4 中断处理

当用户按 Ctrl+C 或 Esc 取消：
- `Select` 组件触发 `onCancel`
- 调用 `respond(false)`
- 返回 `DENY_ALL_RESPONSE`

### 6.3 改进建议

#### 6.3.1 自动权限检测

**建议**：在 TCC 面板中添加轮询机制，自动检测权限是否已授予。

```typescript
// 伪代码
useEffect(() => {
  const interval = setInterval(async () => {
    const status = await checkPermissions();
    if (status.accessibility && status.screenRecording) {
      onDone(DENY_ALL_RESPONSE); // 触发重试
    }
  }, 1000);
  return () => clearInterval(interval);
}, []);
```

#### 6.3.2 应用搜索/过滤

**建议**：当应用列表很长时，添加搜索或过滤功能。

#### 6.3.3 权限持久化

**建议**：考虑将用户选择持久化到磁盘，跨会话记住用户偏好。

**当前限制**：权限仅在当前会话有效，每次重启 Claude Code 需要重新授权。

#### 6.3.4 更细粒度的权限控制

**建议**：允许用户为每个应用单独设置权限标志（如仅允许截图但不允许点击）。

#### 6.3.5 批量操作优化

**建议**：添加"全选"/"全不选"功能，方便用户批量管理大量应用。

#### 6.3.6 测试覆盖

**当前状态**：组件缺乏单元测试。

**建议**：
- 添加对 `ComputerUseApproval` 组件的单元测试
- 测试各种边界情况（空列表、未安装应用、已授权应用）
- 测试用户交互流程（选择、取消、确认）

### 6.4 技术债务

#### 6.4.1 React Compiler 缓存

代码中使用了 React Compiler 的自动缓存机制（`$[n]` 模式），这增加了代码复杂度：

```typescript
const $ = _c(48);  // 48 个缓存槽位
// ...
if ($[0] !== request.apps) {
  t1 = () => new Set(request.apps.flatMap(_temp));
  $[0] = request.apps;
  $[1] = t1;
}
```

**建议**：考虑使用 `useMemo` 和 `useCallback` 显式优化，提高代码可读性。

#### 6.4.2 硬编码字符串

部分用户可见字符串硬编码在组件中，不利于国际化：

```typescript
const SENTINEL_WARNING = {
  shell: 'equivalent to shell access',
  filesystem: 'can read/write any file',
  system_settings: 'can change system settings'
};
```

**建议**：引入国际化框架或至少将字符串提取到配置文件中。

---

## 7. 附录

### 7.1 相关配置

GrowthBook 功能开关（`src/utils/computerUse/gates.ts`）：
- `tengu_malort_pedway`: Computer Use 功能总开关
- 子开关：`pixelValidation`, `mouseAnimation`, `hideBeforeAction`, `clipboardGuard` 等

### 7.2 调试日志

相关调试日志标签：
- `[computer-use]`: 执行器日志
- `[cu-esc]`: ESC 热键日志
- `[drainRunLoop]`: CFRunLoop 泵日志

### 7.3 文件锁

Computer Use 使用文件锁（`computer-use.lock`）防止多会话冲突：
- 位置：`~/.config/claude/computer-use.lock`
- 实现：`src/utils/computerUse/computerUseLock.ts`

---

*文档生成时间：2026-04-01*
*研究范围：src/components/permissions/ComputerUseApproval/ComputerUseApproval.tsx 及其直接依赖*
