# useFilePermissionDialog.ts 研究文档

## 场景与职责

`useFilePermissionDialog.ts` 是 Claude Code 中文件权限对话框的状态管理 Hook。它封装了文件权限对话框的所有状态逻辑，包括选项生成、用户交互处理、反馈收集和键盘快捷键支持。

### 核心职责
1. **状态管理**：管理反馈输入、焦点状态、输入模式等 UI 状态
2. **选项生成**：根据文件路径和上下文生成权限选项
3. **事件处理**：处理用户选择、反馈输入、模式切换等交互
4. **快捷键支持**：注册和处理键盘快捷键（如 `confirm:cycleMode`）
5. **分析日志**：记录用户交互事件用于分析

### 使用场景
- `FilePermissionDialog` 组件的状态管理
- `FilesystemPermissionRequest` 等组件的权限处理
- 任何需要文件操作权限确认的场景

---

## 功能点目的

### 1. 反馈状态管理
- **目的**：收集用户在允许或拒绝时的额外说明
- **状态**：
  - `acceptFeedback`: 允许时的反馈文本
  - `rejectFeedback`: 拒绝时的反馈文本
- **持久化标记**：
  - `yesFeedbackModeEntered`: 标记用户是否曾进入过允许反馈模式
  - `noFeedbackModeEntered`: 标记用户是否曾进入过拒绝反馈模式

### 2. 焦点与输入模式管理
- **目的**：支持键盘导航和 Tab 切换输入模式
- **状态**：
  - `focusedOption`: 当前聚焦的选项（'yes', 'no', 'yes-session' 等）
  - `yesInputMode`: 是否处于允许选项的输入模式
  - `noInputMode`: 是否处于拒绝选项的输入模式

### 3. 快捷键支持
- **目的**：提供快捷操作提升用户体验
- **支持快捷键**：
  - `confirm:cycleMode`: 快速选择会话级允许选项（默认 Shift+Tab）

### 4. 选项变更处理
- **目的**：统一处理用户选择并调用相应的权限处理器
- **处理流程**：
  1. 构建参数对象（`PermissionHandlerParams`）
  2. 重写 `toolUseConfirm.onAllow` 以传递解析后的输入
  3. 根据选项类型调用对应的处理器

---

## 具体技术实现

### 关键数据结构

```typescript
// 工具输入基础类型
export interface ToolInput {
  [key: string]: unknown;
}

// Hook Props
export type UseFilePermissionDialogProps<T extends ToolInput> = {
  filePath: string;
  completionType: CompletionType;
  languageName: string | Promise<string>;
  toolUseConfirm: ToolUseConfirm;
  onDone: () => void;
  onReject: () => void;
  parseInput: (input: unknown) => T;
  operationType?: FileOperationType;
};

// Hook 返回结果
export type UseFilePermissionDialogResult<T> = {
  options: PermissionOptionWithLabel[];
  onChange: (option: PermissionOption, input: T, feedback?: string) => void;
  acceptFeedback: string;
  rejectFeedback: string;
  focusedOption: string;
  setFocusedOption: (option: string) => void;
  handleInputModeToggle: (value: string) => void;
  yesInputMode: boolean;
  noInputMode: boolean;
};
```

### 核心算法

#### 1. Hook 实现
```typescript
export function useFilePermissionDialog<T extends ToolInput>({
  filePath,
  completionType,
  languageName,
  toolUseConfirm,
  onDone,
  onReject,
  parseInput,
  operationType = 'write',
}: UseFilePermissionDialogProps<T>): UseFilePermissionDialogResult<T> {
  // 获取权限上下文
  const toolPermissionContext = useAppState(s => s.toolPermissionContext);
  
  // 反馈状态
  const [acceptFeedback, setAcceptFeedback] = useState('');
  const [rejectFeedback, setRejectFeedback] = useState('');
  
  // 焦点状态
  const [focusedOption, setFocusedOption] = useState('yes');
  
  // 输入模式状态
  const [yesInputMode, setYesInputMode] = useState(false);
  const [noInputMode, setNoInputMode] = useState(false);
  
  // 反馈模式进入标记（用于分析）
  const [yesFeedbackModeEntered, setYesFeedbackModeEntered] = useState(false);
  const [noFeedbackModeEntered, setNoFeedbackModeEntered] = useState(false);

  // 生成选项（memoized）
  const options = useMemo(
    () =>
      getFilePermissionOptions({
        filePath,
        toolPermissionContext,
        operationType,
        onRejectFeedbackChange: setRejectFeedback,
        onAcceptFeedbackChange: setAcceptFeedback,
        yesInputMode,
        noInputMode,
      }),
    [filePath, toolPermissionContext, operationType, yesInputMode, noInputMode],
  );

  // 处理选项变更
  const onChange = useCallback(
    (option: PermissionOption, input: T, feedback?: string) => {
      const params: PermissionHandlerParams = {
        messageId: toolUseConfirm.assistantMessage.message.id,
        path: filePath,
        toolUseConfirm,
        toolPermissionContext,
        onDone,
        onReject,
        completionType,
        languageName,
        operationType,
      };

      // 重写 onAllow 以传递解析后的输入
      const originalOnAllow = toolUseConfirm.onAllow;
      toolUseConfirm.onAllow = (
        _input: unknown,
        permissionUpdates: PermissionUpdate[],
        feedback?: string,
      ) => {
        originalOnAllow(input, permissionUpdates, feedback);
      };

      // 调用对应的处理器
      const handler = PERMISSION_HANDLERS[option.type];
      handler(params, {
        feedback,
        hasFeedback: !!feedback,
        enteredFeedbackMode:
          option.type === 'accept-once'
            ? yesFeedbackModeEntered
            : noFeedbackModeEntered,
        scope: option.type === 'accept-session' ? option.scope : undefined,
      });
    },
    [/* 依赖数组 */],
  );

  // 快捷键：选择 accept-session 选项
  const handleCycleMode = useCallback(() => {
    const sessionOption = options.find(o => o.option.type === 'accept-session');
    if (sessionOption) {
      const parsedInput = parseInput(toolUseConfirm.input);
      onChange(sessionOption.option, parsedInput);
    }
  }, [options, parseInput, toolUseConfirm.input, onChange]);

  useKeybindings(
    { 'confirm:cycleMode': handleCycleMode },
    { context: 'Confirmation' },
  );

  // 焦点变更处理（清理输入模式）
  const handleFocusedOptionChange = useCallback(
    (value: string) => {
      // 离开选项时，如果没有输入文本，退出输入模式
      if (value !== 'yes' && yesInputMode && !acceptFeedback.trim()) {
        setYesInputMode(false);
      }
      if (value !== 'no' && noInputMode && !rejectFeedback.trim()) {
        setNoInputMode(false);
      }
      setFocusedOption(value);
    },
    [yesInputMode, noInputMode, acceptFeedback, rejectFeedback],
  );

  // 输入模式切换
  const handleInputModeToggle = useCallback(
    (value: string) => {
      const analyticsProps = {
        toolName: sanitizeToolNameForAnalytics(toolUseConfirm.tool.name),
        isMcp: toolUseConfirm.tool.isMcp ?? false,
      };

      if (value === 'yes') {
        if (yesInputMode) {
          setYesInputMode(false);
          logEvent('tengu_accept_feedback_mode_collapsed', analyticsProps);
        } else {
          setYesInputMode(true);
          setYesFeedbackModeEntered(true);
          logEvent('tengu_accept_feedback_mode_entered', analyticsProps);
        }
      } else if (value === 'no') {
        if (noInputMode) {
          setNoInputMode(false);
          logEvent('tengu_reject_feedback_mode_collapsed', analyticsProps);
        } else {
          setNoInputMode(true);
          setNoFeedbackModeEntered(true);
          logEvent('tengu_reject_feedback_mode_entered', analyticsProps);
        }
      }
    },
    [yesInputMode, noInputMode, toolUseConfirm],
  );

  return {
    options,
    onChange,
    acceptFeedback,
    rejectFeedback,
    focusedOption,
    setFocusedOption: handleFocusedOptionChange,
    handleInputModeToggle,
    yesInputMode,
    noInputMode,
  };
}
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `react` (useCallback, useMemo, useState) | React Hooks |
| `src/state/AppState.js` | 应用状态管理（useAppState） |
| `../../../keybindings/useKeybinding.js` | 快捷键绑定 |
| `../../../services/analytics/index.js` | 分析日志（logEvent） |
| `../../../services/analytics/metadata.js` | 工具名清理（sanitizeToolNameForAnalytics） |
| `../../../utils/permissions/PermissionUpdateSchema.js` | PermissionUpdate 类型 |
| `../../../utils/unaryLogging.js` | CompletionType 类型 |
| `../PermissionRequest.js` | ToolUseConfirm 类型 |
| `./permissionOptions.js` | 选项生成（getFilePermissionOptions） |
| `./usePermissionHandler.js` | 权限处理器（PERMISSION_HANDLERS） |

### 被引用文件

| 文件路径 | 用途 |
|---------|------|
| `./FilePermissionDialog.tsx` | 使用 useFilePermissionDialog |
| `../FilesystemPermissionRequest/FilesystemPermissionRequest.tsx` | 可能使用 |

---

## 依赖与外部交互

### 状态管理依赖

```typescript
// 来自 src/state/AppState.js
const toolPermissionContext = useAppState(s => s.toolPermissionContext);
```

`toolPermissionContext` 提供：
- 当前权限规则
- 允许的工作目录
- 工具特定的权限配置

### 权限处理器依赖

```typescript
// 来自 ./usePermissionHandler.js
import { PERMISSION_HANDLERS } from './usePermissionHandler.js';
```

处理器映射：
- `'accept-once'` → `handleAcceptOnce`
- `'accept-session'` → `handleAcceptSession`
- `'reject'` → `handleReject`

### 分析日志依赖

```typescript
// 分析事件
logEvent('tengu_accept_feedback_mode_entered', analyticsProps);
logEvent('tengu_accept_feedback_mode_collapsed', analyticsProps);
logEvent('tengu_reject_feedback_mode_entered', analyticsProps);
logEvent('tengu_reject_feedback_mode_collapsed', analyticsProps);
```

### 快捷键系统依赖

```typescript
// 来自 ../../../keybindings/useKeybinding.js
useKeybindings(
  { 'confirm:cycleMode': handleCycleMode },
  { context: 'Confirmation' },
);
```

---

## 风险、边界与改进建议

### 潜在风险

#### 1. 函数重写风险
- **风险**：`toolUseConfirm.onAllow` 被重写可能导致意外行为
- **代码**：
  ```typescript
  const originalOnAllow = toolUseConfirm.onAllow;
  toolUseConfirm.onAllow = (...) => { ... };
  ```
- **潜在问题**：
  - 如果组件重新渲染，可能多次重写
  - 并发操作可能导致状态混乱
- **建议**：考虑使用 ref 或更函数式的方式处理

#### 2. 依赖数组复杂性
- **风险**：`onChange` 回调的依赖数组非常长（14 个依赖项）
- **影响**：可能导致不必要的重新创建回调
- **建议**：
  - 考虑使用 `useRef` 存储部分依赖
  - 或使用状态机模式简化逻辑

#### 3. 输入模式状态不一致
- **风险**：`yesInputMode` 和 `acceptFeedback` 可能不同步
- **现有防护**：`handleFocusedOptionChange` 会清理空输入的模式
- **建议**：考虑使用 reducer 模式统一管理相关状态

### 边界情况

#### 1. 选项查找失败
- `handleCycleMode` 中查找 `accept-session` 选项可能失败
- 当前实现：静默处理（if 检查）
- 建议：添加日志或错误处理

#### 2. 反馈文本处理
- 反馈文本的 `trim()` 处理在调用方进行
- 空字符串反馈会被转换为 `undefined`
- 建议：统一在 Hook 内部处理

#### 3. 快捷键冲突
- `confirm:cycleMode` 可能与系统快捷键冲突
- 建议：提供可配置快捷键或冲突检测

### 改进建议

#### 1. 状态管理优化
```typescript
// 建议：使用 reducer 模式
interface DialogState {
  acceptFeedback: string;
  rejectFeedback: string;
  focusedOption: string;
  yesInputMode: boolean;
  noInputMode: boolean;
  yesFeedbackModeEntered: boolean;
  noFeedbackModeEntered: boolean;
}

type DialogAction =
  | { type: 'SET_FEEDBACK'; payload: { option: 'yes' | 'no'; value: string } }
  | { type: 'TOGGLE_INPUT_MODE'; payload: { option: 'yes' | 'no' } }
  | { type: 'SET_FOCUSED_OPTION'; payload: string }
  | { type: 'RESET' };
```

#### 2. 性能优化
```typescript
// 建议：使用 useRef 避免依赖循环
const toolUseConfirmRef = useRef(toolUseConfirm);
useEffect(() => {
  toolUseConfirmRef.current = toolUseConfirm;
}, [toolUseConfirm]);

// 然后在 onChange 中使用 toolUseConfirmRef.current
```

#### 3. 错误边界
```typescript
// 建议：添加错误处理
const onChange = useCallback((option, input, feedback) => {
  try {
    // 现有逻辑
  } catch (error) {
    logError(error);
    onReject();
  }
}, [/* ... */]);
```

#### 4. 测试友好性
```typescript
// 建议：导出纯函数版本便于测试
export function createPermissionHandlerParams(/* ... */): PermissionHandlerParams {
  // 纯函数逻辑
}
```

#### 5. TypeScript 严格性
```typescript
// 建议：更严格的选项值类型
export type FocusedOptionValue = 'yes' | 'yes-session' | 'yes-claude-folder' | 'no';

const [focusedOption, setFocusedOption] = useState<FocusedOptionValue>('yes');
```

### 代码质量建议

1. **常量提取**：
   ```typescript
   const DEFAULT_FOCUSED_OPTION = 'yes';
   const INPUT_MODE_YES = 'yes';
   const INPUT_MODE_NO = 'no';
   ```

2. **日志增强**：
   - 在关键状态变更处添加调试日志
   - 记录选项生成结果

3. **文档完善**：
   - 添加 JSDoc 说明各返回值的用途
   - 解释 `onAllow` 重写的设计决策

4. **内存泄漏防护**：
   - 确保事件监听器正确清理
   - 考虑组件卸载时的状态清理
