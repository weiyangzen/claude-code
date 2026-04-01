# ExitPlanModePermissionRequest.tsx 深度研究文档

## 1. 场景与职责

### 1.1 组件定位
`ExitPlanModePermissionRequest` 是 Claude Code CLI 中 Plan Mode（计划模式）的核心 UI 组件，负责处理用户退出计划模式时的权限请求和交互流程。当 Claude 完成计划编写并准备进入执行阶段时，此组件渲染计划审批对话框，允许用户选择不同的执行模式。

### 1.2 核心职责
- **计划展示**: 渲染计划内容供用户审阅
- **模式选择**: 提供多种退出计划模式的选项（默认模式、自动接受编辑、绕过权限、自动模式等）
- **权限更新**: 根据用户选择构建并应用权限更新
- **外部编辑集成**: 支持 Ctrl+G 快捷键在外部编辑器中编辑计划
- **Ultraplan 集成**: 支持将计划发送到远程 Claude Code 实例进行优化
- **反馈收集**: 允许用户拒绝计划时提供反馈或附加图片

### 1.3 调用场景
| 场景 | 触发方式 |
|------|----------|
| 正常计划审批 | Claude 调用 `ExitPlanModeV2Tool` 请求退出计划模式 |
| 空计划处理 | 计划内容为空时的简化审批流程 |
| 远程队友审批 | Agent Swarm 场景下队友计划的领导审批 |

---

## 2. 功能点目的

### 2.1 计划审批选项构建 (`buildPlanApprovalOptions`)

根据当前权限上下文动态构建审批选项列表：

```typescript
export function buildPlanApprovalOptions({
  showClearContext,      // 是否显示"清除上下文"选项
  showUltraplan,         // 是否显示 Ultraplan 选项
  usedPercent,           // 上下文使用百分比
  isAutoModeAvailable,   // 自动模式是否可用
  isBypassPermissionsModeAvailable,  // 绕过权限模式是否可用
  onFeedbackChange       // 反馈输入变化回调
}): OptionWithDescription<ResponseValue>[]
```

**选项优先级逻辑**:
1. 清除上下文 + 自动模式 (TRANSCRIPT_CLASSIFIER feature 且可用)
2. 清除上下文 + 绕过权限 (如果可用)
3. 清除上下文 + 自动接受编辑 (默认)
4. 保留上下文 + 自动模式/绕过权限/自动接受编辑
5. 保留上下文 + 手动审批
6. Ultraplan 优化选项
7. 拒绝并继续规划（带反馈输入）

### 2.2 权限更新构建 (`buildPermissionUpdates`)

根据选择的模式和允许的提示构建权限更新：

```typescript
export function buildPermissionUpdates(
  mode: PermissionMode, 
  allowedPrompts?: AllowedPrompt[]
): PermissionUpdate[]
```

**功能**:
- 设置权限模式 (`setMode`)
- 添加基于提示的权限规则 (`addRules`) - Ant-only 特性
- 仅当分类器权限启用且存在允许的提示时才添加规则

### 2.3 自动会话命名 (`autoNameSessionFromPlan`)

当用户接受计划时，基于计划内容自动生成会话名称：

- 使用 `generateSessionName` 从计划内容生成 kebab-case 名称
- 更新提示边框徽章显示
- 仅在未手动重命名且未禁用会话持久化时执行
- 支持 clear-context 场景下的新会话命名

### 2.4 响应处理 (`handleResponse`)

处理用户的各种响应选项：

| 响应值 | 行为 |
|--------|------|
| `ultraplan` | 启动 Ultraplan 远程优化流程 |
| `yes-bypass-permissions` | 退出到绕过权限模式 |
| `yes-accept-edits` | 退出到自动接受编辑模式（清除上下文） |
| `yes-accept-edits-keep-context` | 保留上下文，自动接受编辑 |
| `yes-default-keep-context` | 保留上下文，手动审批 |
| `yes-resume-auto-mode` | 恢复自动模式 |
| `yes-auto-clear-context` | 清除上下文并启用自动模式 |
| `no` | 拒绝退出，可选择提供反馈 |

---

## 3. 具体技术实现

### 3.1 数据结构

#### 3.1.1 响应值类型
```typescript
type ResponseValue = 
  | 'yes-bypass-permissions' 
  | 'yes-accept-edits' 
  | 'yes-accept-edits-keep-context' 
  | 'yes-default-keep-context' 
  | 'yes-resume-auto-mode' 
  | 'yes-auto-clear-context' 
  | 'ultraplan' 
  | 'no';
```

#### 3.1.2 组件 Props
```typescript
interface PermissionRequestProps {
  toolUseConfirm: ToolUseConfirm<Input>;  // 工具使用确认对象
  onDone(): void;                          // 完成回调
  onReject(): void;                        // 拒绝回调
  workerBadge: WorkerBadgeProps | undefined;  // 工作线程徽章
  setStickyFooter?: (jsx: React.ReactNode | null) => void;  // 粘性页脚设置
}
```

### 3.2 关键流程

#### 3.2.1 V1/V2 计划检测流程
```typescript
// 使用工具名称检测 V2，而非检查 input.plan
const isV2 = toolUseConfirm.tool.name === EXIT_PLAN_MODE_V2_TOOL_NAME;
const inputPlan = isV2 ? undefined : toolUseConfirm.input.plan as string | undefined;
const planFilePath = isV2 ? getPlanFilePath() : undefined;
```

**背景**: PR #10394 为 hooks/SDK 注入 plan 内容到 input.plan，导致旧的检测逻辑失效（见 issue #10878）。

#### 3.2.2 自动模式状态处理流程
```typescript
if (feature('TRANSCRIPT_CLASSIFIER')) {
  const goingToAuto = (value === 'yes-resume-auto-mode' || value === 'yes-auto-clear-context') 
    && isAutoModeGateEnabled();
  const autoWasUsedDuringPlan = autoModeStateModule?.isAutoModeActive() ?? false;
  
  if (value !== 'no' && !goingToAuto && autoWasUsedDuringPlan) {
    // 1. 停用自动模式
    autoModeStateModule?.setAutoModeActive(false);
    // 2. 设置需要自动模式退出附件
    setNeedsAutoModeExitAttachment(true);
    // 3. 恢复危险权限
    setAppState(prev => ({
      ...prev,
      toolPermissionContext: {
        ...restoreDangerousPermissions(prev.toolPermissionContext),
        prePlanMode: undefined
      }
    }));
  }
}
```

#### 3.2.3 清除上下文流程
```typescript
if (value !== 'no' && !isKeepContextOption) {
  // 1. 自动命名会话
  autoNameSessionFromPlan(currentPlan, setAppState, !isKeepContextOption);
  
  // 2. 设置初始消息和清除上下文标志
  setAppState(prev => ({
    ...prev,
    initialMessage: {
      message: createUserMessage({
        content: `Implement the following plan:\n\n${currentPlan}...`
      }),
      planContent: currentPlan
    },
    clearContext: true,
    mode,
    allowedPrompts
  }));
  
  // 3. 设置已退出计划模式标志
  setHasExitedPlanMode(true);
  
  // 4. 触发完成和拒绝回调
  onDone();
  onReject();
  toolUseConfirm.onReject();
}
```

### 3.3 键盘快捷键处理

```typescript
const handleKeyDown = (e: KeyboardEvent): void => {
  // Ctrl+G: 在外部编辑器中编辑计划
  if (e.ctrl && e.key === 'g') {
    e.preventDefault();
    logEvent('tengu_plan_external_editor_used', {});
    // 调用 editFileInEditor 或 editPromptInEditor
  }
  
  // Shift+Tab: 快速选择"自动接受编辑"
  if (e.shift && e.key === 'tab') {
    e.preventDefault();
    void handleResponse(showClearContext ? 'yes-accept-edits' : 'yes-accept-edits-keep-context');
  }
};
```

### 3.4 粘性页脚实现

用于全屏模式，确保长计划的响应选项始终可见：

```typescript
const useStickyFooter = !isEmpty && !!setStickyFooter;
useLayoutEffect(() => {
  if (!useStickyFooter) return;
  setStickyFooter(<Box flexDirection="column" ...>
    <Text dimColor>Would you like to proceed?</Text>
    <Select options={options} onChange={...} ... />
    {editorName && <Text>ctrl-g to edit in {editorName}</Text>}
  </Box>);
  return () => setStickyFooter(null);
}, [useStickyFooter, setStickyFooter, options, ...]);
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 职责 |
|----------|------|
| `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 主组件实现 |
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | ExitPlanMode 工具定义 |
| `src/tools/ExitPlanModeTool/constants.ts` | 工具名称常量 |
| `src/components/permissions/PermissionRequest.tsx` | 权限请求分发器 |

### 4.2 依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/utils/permissions/PermissionMode.ts` | 权限模式类型和转换 |
| `src/utils/permissions/permissionSetup.ts` | 权限设置、危险权限剥离/恢复 |
| `src/utils/permissions/autoModeState.ts` | 自动模式状态管理 |
| `src/utils/permissions/PermissionUpdateSchema.ts` | 权限更新 Schema |
| `src/utils/plans.ts` | 计划文件读写管理 |
| `src/bootstrap/state.ts` | 全局状态管理（hasExitedPlanMode 等） |
| `src/commands/ultraplan.tsx` | Ultraplan 启动逻辑 |
| `src/components/CustomSelect/select.tsx` | Select 组件 |
| `src/components/permissions/PermissionDialog.tsx` | 权限对话框容器 |

### 4.3 调用链

```
ExitPlanModeV2Tool.call() 
  -> checkPermissions() 返回 'ask'
  -> PermissionRequest 组件渲染
    -> permissionComponentForTool() 映射到 ExitPlanModePermissionRequest
      -> ExitPlanModePermissionRequest 渲染对话框
        -> 用户选择 -> handleResponse()
          -> onAllow() / onReject() 回调
            -> 工具执行完成
```

---

## 5. 依赖与外部交互

### 5.1 React Hooks 使用

| Hook | 用途 |
|------|------|
| `useAppState` | 获取工具权限上下文、设置 |
| `useSetAppState` | 更新应用状态 |
| `useAppStateStore` | 获取状态存储（用于 Ultraplan） |
| `useNotifications` | 添加通知（外部编辑器错误等） |
| `useState` | 本地状态（计划内容、反馈、粘贴内容等） |
| `useRef` | 回调引用（handleResponseRef、handleCancelRef） |
| `useEffect` | 自动隐藏保存消息 |
| `useLayoutEffect` | 粘性页脚设置 |
| `useMemo` | 选项列表缓存 |
| `useCallback` | 回调函数缓存（onRemoveImage） |

### 5.2 外部功能标志

| 功能标志 | 用途 |
|----------|------|
| `TRANSCRIPT_CLASSIFIER` | 启用自动模式相关功能 |
| `ULTRAPLAN` | 启用 Ultraplan 功能 |
| `KAIROS` / `KAIROS_CHANNELS` | 禁用计划模式（通道模式不支持） |

### 5.3 状态交互

```typescript
// 读取的状态
const toolPermissionContext = useAppState(s => s.toolPermissionContext);
const showClearContext = useAppState(s => s.settings.showClearContextOnPlanAccept);
const ultraplanSessionUrl = useAppState(s => s.ultraplanSessionUrl);
const ultraplanLaunching = useAppState(s => s.ultraplanLaunching);

// 更新的状态
setAppState(prev => ({ ...prev, initialMessage: {...}, toolPermissionContext: {...} }));
setHasExitedPlanMode(true);
setNeedsPlanModeExitAttachment(true);
setNeedsAutoModeExitAttachment(true);
```

### 5.4 工具回调

```typescript
// 允许计划执行
toolUseConfirm.onAllow(updatedInput, permissionUpdates, acceptFeedback);

// 拒绝计划执行
toolUseConfirm.onReject(feedback, imageBlocks);
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 自动模式状态不一致风险
**问题**: `isAutoModeActive()` 是权威信号，但 `prePlanMode`/`strippedDangerousRules` 在 `transitionPlanAutoMode` 停用后可能过时。

**缓解**: 代码中已使用 `isAutoModeActive()` 作为主要检查，而非依赖 `prePlanMode`。

#### 6.1.2 电路断路器绕过风险
**问题**: 如果 `prePlanMode` 是自动模式但电路断路器已触发，`ExitPlanMode` 可能直接调用 `setAutoModeActive(true)` 绕过断路器。

**缓解**: `ExitPlanModeV2Tool.call()` 和本组件都实现了门控回退逻辑：
```typescript
if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
  restoreMode = 'default';
}
```

#### 6.1.3 竞态条件风险
**问题**: Ultraplan 启动和本地拒绝之间存在竞态，可能导致会话状态不一致。

**缓解**: `showUltraplan` 在会话激活或启动时隐藏按钮，且 `launchUltraplan` 检查现有会话。

### 6.2 边界情况

| 边界情况 | 处理逻辑 |
|----------|----------|
| 空计划 | 简化 UI，仅显示"Yes/No"选项 |
| 计划文件不存在 | 显示"No plan found"占位文本 |
| 外部编辑器不可用 | 不显示 Ctrl+G 提示 |
| 自动模式门控关闭 | 回退到默认模式并通知用户 |
| 图片粘贴 | 转换为 ImageBlockParam 并随拒绝反馈发送 |
| 会话持久化禁用 | 跳过自动命名 |

### 6.3 改进建议

#### 6.3.1 代码组织
- **建议**: 将 `buildPlanApprovalOptions` 和 `buildPermissionUpdates` 移到单独的 utils 文件，便于测试和复用。
- **建议**: 将 V1/V2 检测逻辑提取为共享 hook，避免硬编码工具名称检查。

#### 6.3.2 类型安全
- **建议**: `ResponseValue` 类型可以进一步细化为清除上下文/保留上下文的联合类型，减少运行时检查。

#### 6.3.3 性能优化
- **建议**: `getContextUsedPercent` 在每次渲染时重新计算，可以缓存结果直到 usage 变化。
- **建议**: 图片粘贴后的 resize 操作可以移到 Web Worker 避免阻塞主线程。

#### 6.3.4 用户体验
- **建议**: 添加计划内容长度警告（当前仅在 analytics 中记录）。
- **建议**: 支持计划内容的语法高亮（如果计划包含代码块）。
- **建议**: 添加计划版本历史，支持查看和恢复之前的计划版本。

#### 6.3.5 可访问性
- **建议**: 为快捷键（Ctrl+G、Shift+Tab）添加屏幕阅读器提示。
- **建议**: 确保粘性页脚在高对比度主题下正确显示。

#### 6.3.6 测试覆盖
- **建议**: 添加针对自动模式门控回退的集成测试。
- **建议**: 添加 Ultraplan 流程的 E2E 测试（使用 mock 远程服务）。
- **建议**: 添加图片粘贴和 resize 的单元测试。

---

## 7. 附录

### 7.1 相关常量

```typescript
// src/tools/ExitPlanModeTool/constants.ts
export const EXIT_PLAN_MODE_TOOL_NAME = 'ExitPlanMode'
export const EXIT_PLAN_MODE_V2_TOOL_NAME = 'ExitPlanMode'
```

### 7.2 权限模式映射

| 内部模式 | 外部模式 | 说明 |
|----------|----------|------|
| `default` | `default` | 手动审批 |
| `acceptEdits` | `acceptEdits` | 自动接受编辑 |
| `bypassPermissions` | `bypassPermissions` | 绕过权限 |
| `auto` | `default` | 自动模式（Ant-only） |
| `plan` | `plan` | 计划模式 |

### 7.3 Analytics 事件

| 事件名 | 触发时机 |
|--------|----------|
| `tengu_plan_exit` | 计划退出时（包含 outcome、planLengthChars 等元数据） |
| `tengu_plan_external_editor_used` | 用户使用 Ctrl+G 打开外部编辑器 |
| `tengu_ultraplan_awaiting_input` | Ultraplan 等待用户输入 |
| `tengu_ultraplan_approved` | Ultraplan 计划被批准 |
| `tengu_ultraplan_failed` | Ultraplan 失败 |
