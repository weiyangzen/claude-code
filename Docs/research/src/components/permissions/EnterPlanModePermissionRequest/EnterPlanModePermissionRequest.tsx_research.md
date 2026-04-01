# EnterPlanModePermissionRequest 组件研究文档

## 1. 场景与职责

### 1.1 核心定位

`EnterPlanModePermissionRequest` 是 Claude Code CLI 中负责**计划模式进入权限请求**的 React 组件。它作为权限系统的一部分，当 Claude 需要进入计划模式（Plan Mode）时，向用户展示确认对话框，获取用户明确授权。

### 1.2 业务场景

| 场景 | 说明 |
|------|------|
| **工具触发** | 当 `EnterPlanModeTool` 被调用时，通过权限系统路由到此组件 |
| **模式切换** | 用户从默认模式切换到计划模式时的确认流程 |
| **安全 gate** | 防止 Claude 自动进入计划模式，确保用户知情同意 |

### 1.3 职责边界

- **UI 展示**: 渲染计划模式进入确认对话框
- **用户交互**: 处理用户选择（同意/拒绝）
- **状态协调**: 调用状态转换函数，记录分析事件
- **权限回调**: 调用 `toolUseConfirm` 的允许/拒绝回调

---

## 2. 功能点目的

### 2.1 功能概述

该组件实现了一个**受控的权限请求流程**：

1. **信息展示**: 向用户说明计划模式的用途和行为
2. **选择交互**: 提供"是，进入计划模式"和"否，立即开始实现"两个选项
3. **状态转换**: 根据用户选择执行相应的模式切换和回调
4. **分析追踪**: 记录用户决策用于产品分析

### 2.2 计划模式行为说明

组件向用户展示计划模式下的 Claude 行为：

- 彻底探索代码库
- 识别现有模式
- 设计实现策略
- 呈现计划供用户批准
- **关键承诺**: 在用户批准计划前不会进行代码更改

### 2.3 用户选择处理

| 用户选择 | 行为 |
|----------|------|
| **Yes** | 记录分析事件 → 处理模式转换 → 调用 `onAllow` → 设置模式为 'plan' |
| **No** | 调用 `onDone` → 调用 `onReject` → 调用 `toolUseConfirm.onReject` |

---

## 3. 具体技术实现

### 3.1 组件接口定义

```typescript
// 来自 PermissionRequest.tsx
export type PermissionRequestProps<Input extends AnyObject = AnyObject> = {
  toolUseConfirm: ToolUseConfirm<Input>
  toolUseContext: ToolUseContext
  onDone(): void
  onReject(): void
  verbose: boolean
  workerBadge: WorkerBadgeProps | undefined
}

// ToolUseConfirm 关键字段
export type ToolUseConfirm<Input extends AnyObject = AnyObject> = {
  // ... 其他字段
  onAllow(updatedInput, permissionUpdates, feedback?, contentBlocks?): void
  onReject(feedback?, contentBlocks?): void
}
```

### 3.2 核心实现逻辑

```typescript
export function EnterPlanModePermissionRequest({
  toolUseConfirm,
  onDone,
  onReject,
  workerBadge,
}: PermissionRequestProps): React.ReactNode {
  // 获取当前权限上下文模式
  const toolPermissionContextMode = useAppState(s => s.toolPermissionContext.mode)

  // 处理用户响应
  function handleResponse(value: "yes" | "no") {
    if (value === "yes") {
      // 1. 记录分析事件
      logEvent("tengu_plan_enter", {
        interviewPhaseEnabled: isPlanModeInterviewPhaseEnabled(),
        entryMethod: "tool"
      })
      
      // 2. 处理计划模式转换（清理状态、设置标志）
      handlePlanModeTransition(toolPermissionContextMode, "plan")
      
      // 3. 完成权限请求流程
      onDone()
      
      // 4. 调用允许回调，触发模式更新
      toolUseConfirm.onAllow({}, [{
        type: "setMode",
        mode: "plan",
        destination: "session"
      }])
    } else {
      // 用户拒绝
      onDone()
      onReject()
      toolUseConfirm.onReject()
    }
  }

  // 渲染权限对话框...
}
```

### 3.3 关键数据结构

#### 3.3.1 权限更新对象

```typescript
// PermissionUpdateSchema.ts
export type PermissionUpdate = 
  | { type: 'setMode'; mode: PermissionMode; destination: PermissionUpdateDestination }
  | { type: 'addRules'; rules: PermissionRuleValue[]; behavior: RuleBehavior; destination: PermissionUpdateDestination }
  | // ... 其他类型

// 本组件使用的 setMode 更新
{
  type: "setMode",
  mode: "plan",
  destination: "session"  // 仅会话级别，不持久化
}
```

#### 3.3.2 分析事件元数据

```typescript
// 事件名称: tengu_plan_enter
{
  interviewPhaseEnabled: boolean,  // 是否启用访谈阶段（Plan Mode V2）
  entryMethod: "tool" | "shortcut" | "command"  // 进入方式
}
```

### 3.4 关键流程

#### 3.4.1 用户同意流程

```
用户选择 "Yes"
    ↓
logEvent("tengu_plan_enter", {...})  // 记录分析
    ↓
handlePlanModeTransition(fromMode, "plan")  // 状态转换准备
    ↓
onDone()  // 关闭权限对话框
    ↓
toolUseConfirm.onAllow({}, [PermissionUpdate])  // 应用权限更新
    ↓
AppState.toolPermissionContext.mode = "plan"  // 最终状态变更
```

#### 3.4.2 用户拒绝流程

```
用户选择 "No"
    ↓
onDone()  // 关闭对话框
    ↓
onReject()  // 通知权限系统拒绝
    ↓
toolUseConfirm.onReject()  // 调用工具拒绝回调
    ↓
工具执行被中止，返回拒绝结果给模型
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 职责 |
|----------|------|
| `src/components/permissions/EnterPlanModePermissionRequest/EnterPlanModePermissionRequest.tsx` | 本组件实现 |
| `src/components/permissions/PermissionRequest.tsx` | 权限请求路由和类型定义 |
| `src/components/permissions/PermissionDialog.tsx` | 权限对话框容器组件 |

### 4.2 依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/bootstrap/state.ts` | `handlePlanModeTransition` 全局状态管理 |
| `src/state/AppState.tsx` | `useAppState` Hook 定义 |
| `src/utils/planModeV2.ts` | `isPlanModeInterviewPhaseEnabled` 功能开关 |
| `src/services/analytics/index.ts` | `logEvent` 分析日志 |
| `src/components/CustomSelect/index.ts` | `Select` 选择组件 |
| `src/ink.js` | `Box`, `Text` UI 组件 |

### 4.3 调用方文件

| 文件路径 | 调用方式 |
|----------|----------|
| `src/components/permissions/PermissionRequest.tsx` | `permissionComponentForTool()` 路由映射 |
| `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` | 工具定义，触发权限请求 |

### 4.4 代码引用关系图

```
EnterPlanModePermissionRequest.tsx
├── PermissionRequest.tsx (导入 PermissionRequestProps 类型)
├── PermissionDialog.tsx (UI 容器)
├── Select.tsx (交互组件)
├── state.ts ─────────────┐
│   └── handlePlanModeTransition  │
├── AppState.tsx          │
│   └── useAppState       │
├── planModeV2.ts         │
│   └── isPlanModeInterviewPhaseEnabled
├── analytics/index.ts    │
│   └── logEvent          │
└── ink.js (Box, Text)    │
                          │
EnterPlanModeTool.ts ◄────┘
    └── call() 触发权限请求
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

#### 5.1.1 React Compiler Runtime

```typescript
import { c as _c } from "react/compiler-runtime"
```

组件使用 React Compiler 进行自动记忆化优化，通过 `_c(n)` 创建记忆化缓存数组。

#### 5.1.2 Ink (终端 UI 库)

```typescript
import { Box, Text } from '../../../ink.js'
```

使用 Ink 组件在终端中渲染 React UI。

### 5.2 状态依赖

#### 5.2.1 AppState 订阅

```typescript
const toolPermissionContextMode = useAppState(s => s.toolPermissionContext.mode)
```

订阅权限上下文模式，用于传递给 `handlePlanModeTransition`。

### 5.3 分析服务交互

```typescript
logEvent("tengu_plan_enter", {
  interviewPhaseEnabled: isPlanModeInterviewPhaseEnabled(),
  entryMethod: "tool" as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
})
```

- **事件名**: `tengu_plan_enter`
- **元数据**: 访谈阶段启用状态、进入方式
- **类型安全**: 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 确保不包含敏感信息

### 5.4 权限系统交互

#### 5.4.1 模式转换函数

```typescript
// src/bootstrap/state.ts
export function handlePlanModeTransition(fromMode: string, toMode: string): void {
  // 切换到计划模式时，清除待处理的退出附件
  if (toMode === 'plan' && fromMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = false
  }
  // 离开计划模式时，触发退出附件
  if (fromMode === 'plan' && toMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = true
  }
}
```

#### 5.4.2 权限更新应用

```typescript
// src/utils/permissions/PermissionUpdate.ts
export function applyPermissionUpdate(
  context: ToolPermissionContext,
  update: PermissionUpdate,
): ToolPermissionContext {
  switch (update.type) {
    case 'setMode':
      return { ...context, mode: update.mode }
    // ...
  }
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 状态不一致风险

**风险**: `handlePlanModeTransition` 在 `onAllow` 之前调用，如果 `onAllow` 失败，状态可能已变更但权限未更新。

**缓解**: `onAllow` 通常是同步的，失败概率低；`onDone` 在状态变更前调用确保 UI 关闭。

#### 6.1.2 竞态条件

**风险**: 用户快速切换选择可能导致回调被多次调用。

**缓解**: `Select` 组件内部有防抖处理，`onDone` 调用后组件即将卸载。

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| **重复进入** | 已在计划模式时，工具会阻止再次进入（`EnterPlanModeTool` 内部检查） |
| **子代理上下文** | 工具会检查 `context.agentId`，在子代理中禁止进入计划模式 |
| **Channels 激活时** | `--channels` 激活时禁用计划模式（防止无法退出） |
| **取消操作** | 用户按 Escape 会触发 `onCancel`，等同于选择 "No" |

### 6.3 改进建议

#### 6.3.1 代码结构改进

```typescript
// 建议: 将 handleResponse 逻辑提取为可测试的纯函数
export function createPlanModeResponseHandler({
  toolPermissionContextMode,
  onDone,
  onReject,
  toolUseConfirm,
}: HandlerDeps) {
  return function handleResponse(value: "yes" | "no") {
    // 实现...
  }
}
```

#### 6.3.2 错误处理增强

当前实现假设 `onAllow`/`onReject` 不会抛出。建议添加错误边界：

```typescript
function handleResponse(value: "yes" | "no") {
  try {
    if (value === "yes") {
      // ... 同意流程
    } else {
      // ... 拒绝流程
    }
  } catch (error) {
    logEvent("tengu_plan_enter_error", { error: error.message })
    onDone()
  }
}
```

#### 6.3.3 可访问性改进

当前 Select 组件依赖键盘导航，建议：
- 添加更明确的焦点指示器
- 支持屏幕阅读器的 aria-live 区域用于描述计划模式行为

#### 6.3.4 分析事件丰富

当前 `tengu_plan_enter` 事件仅包含基础信息，建议添加：
- 当前模式（从哪个模式进入）
- 会话时长
- 项目类型检测

#### 6.3.5 测试覆盖

建议添加以下测试场景：
1. 用户选择 "Yes" 的完整流程验证
2. 用户选择 "No" 的拒绝流程验证
3. 快速多次选择的竞态条件测试
4. `handlePlanModeTransition` 调用时序验证

### 6.4 相关配置项

| 配置/环境变量 | 影响 |
|--------------|------|
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | 控制 `isPlanModeInterviewPhaseEnabled()` 返回值 |
| `USER_TYPE=ant` | 内部用户始终启用访谈阶段 |
| `--channels` | 禁用计划模式进入 |
| `feature('KAIROS')` | 影响计划模式可用性 |

---

## 7. 附录

### 7.1 相关工具对比

| 组件 | 用途 | 触发工具 |
|------|------|----------|
| `EnterPlanModePermissionRequest` | 进入计划模式确认 | `EnterPlanModeTool` |
| `ExitPlanModePermissionRequest` | 退出计划模式确认 | `ExitPlanModeV2Tool` |
| `BashPermissionRequest` | Bash 命令确认 | `BashTool` |
| `FileEditPermissionRequest` | 文件编辑确认 | `FileEditTool` |

### 7.2 计划模式 V2 功能

当 `isPlanModeInterviewPhaseEnabled()` 返回 `true` 时：
- 启用 5 阶段计划工作流
- 分析事件标记 `interviewPhaseEnabled: true`
- 工具提示中省略"What Happens"部分（由附件提供详细说明）
