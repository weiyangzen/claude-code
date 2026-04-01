# EnterPlanModeTool.ts 研究文档

## 场景与职责

EnterPlanModeTool 是 Claude Code CLI 中负责**进入计划模式 (Plan Mode)** 的核心工具。计划模式是一种特殊的权限模式，允许 AI 在执行复杂任务前先进行代码库探索、架构设计和方案规划，而无需立即执行代码修改。

### 核心职责
1. **模式切换**：将当前会话从默认模式切换到计划模式
2. **权限管理**：协调权限系统的转换，包括自动模式分类器的激活/停用
3. **状态管理**：更新应用状态以反映计划模式的进入
4. **用户引导**：向模型提供计划模式下的工作指引

### 使用场景
- 新功能实现前的架构设计
- 多文件修改任务的规划
- 需要用户确认方案后再执行的复杂任务
- 需要探索代码库理解现有模式的场景

---

## 功能点目的

### 1. 计划模式进入控制
- **目的**：提供一个受控的入口点，让用户决定是否进入计划模式
- **机制**：通过工具的 `call` 方法执行实际的权限模式转换
- **限制**：禁止在 Agent 上下文中使用（`context.agentId` 检查）

### 2. 权限模式转换协调
- **目的**：确保进入计划模式时正确处理权限系统的状态
- **机制**：
  - 调用 `handlePlanModeTransition` 处理状态转换副作用
  - 调用 `prepareContextForPlanMode` 准备权限上下文
  - 应用权限更新将模式设置为 'plan'

### 3. KAIROS/Channels 兼容性保护
- **目的**：防止在 Channels 模式下进入计划模式后无法退出
- **机制**：当 `--channels` 激活时禁用此工具（因为 ExitPlanMode 的审批对话框需要终端）

### 4. 差异化提示策略
- **目的**：根据用户类型（内部/外部）提供不同的使用指导
- **机制**：通过 `getEnterPlanModeToolPrompt()` 获取不同的提示内容

---

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式定义（空对象，无需参数）
const inputSchema = lazySchema(() =>
  z.strictObject({
    // No parameters needed
  }),
)

// 输出模式定义
const outputSchema = lazySchema(() =>
  z.object({
    message: z.string().describe('Confirmation that plan mode was entered'),
  }),
)
```

### 核心调用流程

```
Tool Call
  ↓
检查 agentId（禁止 Agent 上下文使用）
  ↓
调用 handlePlanModeTransition(fromMode, 'plan')
  ↓
调用 prepareContextForPlanMode(prev.toolPermissionContext)
  ↓
应用权限更新 { type: 'setMode', mode: 'plan', destination: 'session' }
  ↓
返回成功消息
```

### 关键代码路径

**1. 工具定义与构建**
```typescript
export const EnterPlanModeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_PLAN_MODE_TOOL_NAME,  // 'EnterPlanMode'
  searchHint: 'switch to plan mode to design an approach before coding',
  maxResultSizeChars: 100_000,
  shouldDefer: true,  // 延迟加载工具
  isEnabled() { /* KAIROS/Channels 检查 */ },
  isConcurrencySafe() { return true },
  isReadOnly() { return true },
  // ... 渲染和调用方法
})
```

**2. 调用实现**
```typescript
async call(_input, context) {
  // 1. 禁止 Agent 上下文
  if (context.agentId) {
    throw new Error('EnterPlanMode tool cannot be used in agent contexts')
  }

  // 2. 获取应用状态
  const appState = context.getAppState()
  
  // 3. 处理计划模式转换副作用
  handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

  // 4. 更新权限上下文
  context.setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prepareContextForPlanMode(prev.toolPermissionContext),
      { type: 'setMode', mode: 'plan', destination: 'session' },
    ),
  }))

  return {
    data: {
      message: 'Entered plan mode. You should now focus on exploring...',
    },
  }
}
```

**3. 结果映射到 ToolResultBlockParam**
```typescript
mapToolResultToToolResultBlockParam({ message }, toolUseID) {
  const instructions = isPlanModeInterviewPhaseEnabled()
    ? `${message}\n\nDO NOT write or edit any files except the plan file...`
    : `${message}\n\nIn plan mode, you should:\n1. Thoroughly explore...\n2. Identify patterns...`
  
  return {
    type: 'tool_result',
    content: instructions,
    tool_use_id: toolUseID,
  }
}
```

### 依赖模块与交互

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `bun:bundle` | `feature` | 功能开关检查（KAIROS, KAIROS_CHANNELS） |
| `zod/v4` | `z` | Schema 验证 |
| `../../bootstrap/state.js` | `getAllowedChannels`, `handlePlanModeTransition` | 全局状态管理 |
| `../../Tool.js` | `Tool`, `buildTool`, `ToolDef` | 工具基础设施 |
| `../../utils/lazySchema.js` | `lazySchema` | 延迟 Schema 构建 |
| `../../utils/permissions/PermissionUpdate.js` | `applyPermissionUpdate` | 权限更新应用 |
| `../../utils/permissions/permissionSetup.js` | `prepareContextForPlanMode` | 计划模式上下文准备 |
| `../../utils/planModeV2.js` | `isPlanModeInterviewPhaseEnabled` | 计划模式 V2 功能检查 |
| `./constants.js` | `ENTER_PLAN_MODE_TOOL_NAME` | 工具名称常量 |
| `./prompt.js` | `getEnterPlanModeToolPrompt` | 提示内容获取 |
| `./UI.js` | `renderToolResultMessage`, `renderToolUseMessage`, `renderToolUseRejectedMessage` | UI 渲染 |

---

## 依赖与外部交互

### 与权限系统的交互

**1. prepareContextForPlanMode**
- 位置：`src/utils/permissions/permissionSetup.ts`
- 功能：准备权限上下文以进入计划模式
- 关键行为：
  - 保存当前模式到 `prePlanMode`（用于退出时恢复）
  - 如果用户默认模式是 'auto'，激活分类器
  - 处理自动模式的权限剥离/恢复

**2. applyPermissionUpdate**
- 位置：`src/utils/permissions/PermissionUpdate.js`
- 功能：应用权限更新到上下文
- 更新类型：`{ type: 'setMode', mode: 'plan', destination: 'session' }`

**3. handlePlanModeTransition**
- 位置：`src/bootstrap/state.ts`
- 功能：处理计划模式状态转换的副作用
- 行为：
  - 进入计划模式时清除待处理的退出附件标志
  - 退出计划模式时设置 `needsPlanModeExitAttachment`

### 与状态管理的交互

```typescript
// 全局状态中的相关字段（State 类型）
{
  hasExitedPlanMode: boolean           // 追踪会话中是否退出过计划模式
  needsPlanModeExitAttachment: boolean // 是否需要显示计划模式退出附件
  needsAutoModeExitAttachment: boolean // 是否需要显示自动模式退出附件
}
```

### 与 ExitPlanModeTool 的配对

EnterPlanModeTool 与 ExitPlanModeV2Tool 形成完整的计划模式生命周期：
- **EnterPlanModeTool**：进入计划模式，设置 `prePlanMode` 保存原模式
- **ExitPlanModeV2Tool**：退出计划模式，从 `prePlanMode` 恢复原模式

---

## 风险、边界与改进建议

### 已知风险

**1. Agent 上下文限制**
- 风险：Agent 子代理无法使用此工具
- 原因：子代理的 `setAppState` 是 no-op，无法实际改变权限模式
- 缓解：调用时抛出明确错误

**2. Channels 模式陷阱**
- 风险：在 Channels 模式下进入计划模式后可能无法退出（审批对话框需要终端）
- 缓解：`isEnabled()` 检查 `getAllowedChannels().length > 0` 时返回 false

**3. 自动模式交互复杂性**
- 风险：从 auto 模式进入 plan 模式时，权限剥离和恢复逻辑复杂
- 相关代码：`prepareContextForPlanMode` 和 `transitionPermissionMode`

### 边界条件

| 场景 | 行为 |
|------|------|
| 已在计划模式 | 允许重复调用，会重新应用权限更新 |
| Agent 上下文 | 抛出错误 "EnterPlanMode tool cannot be used in agent contexts" |
| Channels 激活 | 工具被禁用（`isEnabled` 返回 false）|
| Interview Phase 启用 | 返回简化版指引（不列出详细步骤）|

### 改进建议

**1. 重复进入检测**
- 当前：允许在已在计划模式时重复调用
- 建议：添加早期返回或警告，避免不必要的权限更新

**2. 错误处理增强**
- 当前：Agent 上下文检查抛出普通 Error
- 建议：使用更具体的错误类型，便于上游处理

**3. 状态一致性验证**
- 当前：依赖 `setAppState` 的原子更新
- 建议：添加后置条件验证，确保模式确实已切换

**4. 测试覆盖**
- 需要测试场景：
  - 从各权限模式（default/auto/acceptEdits）进入计划模式
  - 与自动模式分类器的交互
  - Channels 模式下的禁用行为
  - Agent 上下文的拒绝行为

### 相关文件引用

```
src/tools/EnterPlanModeTool/
├── EnterPlanModeTool.ts    # 本文件 - 工具主逻辑
├── UI.tsx                  # UI 渲染组件
├── constants.ts            # 工具名称常量
└── prompt.ts               # 提示内容定义

src/utils/permissions/
├── permissionSetup.ts      # prepareContextForPlanMode 实现
├── PermissionUpdate.js     # applyPermissionUpdate 实现
└── PermissionMode.ts       # 权限模式类型定义

src/bootstrap/state.ts      # handlePlanModeTransition 和全局状态
src/utils/planModeV2.js     # isPlanModeInterviewPhaseEnabled
src/Tool.js                 # 工具基础设施
```
