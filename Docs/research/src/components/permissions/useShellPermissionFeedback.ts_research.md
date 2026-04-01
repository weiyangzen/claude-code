# useShellPermissionFeedback.ts 研究文档

## 场景与职责

`useShellPermissionFeedback.ts` 是 Claude Code CLI 权限系统的 **Shell 权限反馈状态管理 Hook**。它封装了 Shell 工具（Bash、PowerShell）权限对话框的反馈模式状态管理和事件处理逻辑。

该模块解决了 Shell 权限对话框的**交互复杂性**：
- 管理 Yes/No 选项的输入模式切换（Tab 键切换）
- 跟踪反馈文本状态
- 处理拒绝操作和焦点变化
- 记录用户交互分析事件

## 功能点目的

### 1. 输入模式管理
- **Yes 输入模式**：用户可以在选择"Yes"时提供额外反馈
- **No 输入模式**：用户可以在选择"No"时提供拒绝原因
- 通过 Tab 键在普通模式和输入模式之间切换

### 2. 反馈状态跟踪
- 跟踪接受反馈文本（`acceptFeedback`）
- 跟踪拒绝反馈文本（`rejectFeedback`）
- 记录用户是否曾经进入过反馈模式（用于 UI 提示）

### 3. 焦点管理
- 跟踪当前聚焦的选项（`focusedOption`）
- 处理焦点变化时的状态清理
- 当离开选项且没有输入文本时，自动退出输入模式

### 4. 拒绝处理
- 处理拒绝操作（带或不带反馈）
- 记录 ESC 键退出事件
- 更新归因统计（escape 计数）
- 记录一元权限事件

### 5. 分析日志
- 记录输入模式进入/折叠事件
- 记录权限请求 ESC 退出事件
- 包含工具名称和 MCP 状态

## 具体技术实现

### 核心数据结构

```typescript
// Hook 参数
interface UseShellPermissionFeedbackParams {
  toolUseConfirm: ToolUseConfirm;
  onDone: () => void;
  onReject: () => void;
  explainerVisible: boolean;
}

// Hook 返回值
interface UseShellPermissionFeedbackReturn {
  yesInputMode: boolean;           // Yes 选项是否处于输入模式
  noInputMode: boolean;            // No 选项是否处于输入模式
  yesFeedbackModeEntered: boolean; // 是否曾经进入过 Yes 反馈模式
  noFeedbackModeEntered: boolean;  // 是否曾经进入过 No 反馈模式
  acceptFeedback: string;          // Yes 反馈文本
  rejectFeedback: string;          // No 反馈文本
  setAcceptFeedback: (v: string) => void;
  setRejectFeedback: (v: string) => void;
  focusedOption: string;           // 当前聚焦的选项值
  handleInputModeToggle: (option: string) => void;  // Tab 键处理
  handleReject: (feedback?: string) => void;        // 拒绝处理
  handleFocus: (value: string) => void;             // 焦点变化处理
}
```

### 关键流程

1. **输入模式切换** (`handleInputModeToggle`):
   ```typescript
   function handleInputModeToggle(option: string) {
     toolUseConfirm.onUserInteraction();  // 通知用户交互
     
     if (option === 'yes') {
       if (yesInputMode) {
         setYesInputMode(false);
         logEvent('tengu_accept_feedback_mode_collapsed', analyticsProps);
       } else {
         setYesInputMode(true);
         setYesFeedbackModeEntered(true);
         logEvent('tengu_accept_feedback_mode_entered', analyticsProps);
       }
     } else if (option === 'no') {
       // 类似逻辑...
     }
   }
   ```

2. **拒绝处理** (`handleReject`):
   ```typescript
   function handleReject(feedback?: string) {
     const trimmedFeedback = feedback?.trim();
     const hasFeedback = !!trimmedFeedback;
     
     // 无反馈时记录 ESC 事件
     if (!hasFeedback) {
       logEvent('tengu_permission_request_escape', { explainer_visible: explainerVisible });
       setAppState(prev => ({
         ...prev,
         attribution: { ...prev.attribution, escapeCount: prev.attribution.escapeCount + 1 }
       }));
     }
     
     // 记录一元事件
     logUnaryPermissionEvent('tool_use_single', toolUseConfirm, 'reject', hasFeedback);
     
     // 调用回调
     if (trimmedFeedback) {
       toolUseConfirm.onReject(trimmedFeedback);
     } else {
       toolUseConfirm.onReject();
     }
     
     onReject();
     onDone();
   }
   ```

3. **焦点处理** (`handleFocus`):
   ```typescript
   function handleFocus(value: string) {
     // 焦点变化时通知交互
     if (value !== focusedOption) {
       toolUseConfirm.onUserInteraction();
     }
     
     // 离开选项且无文本时退出输入模式
     if (value !== 'yes' && yesInputMode && !acceptFeedback.trim()) {
       setYesInputMode(false);
     }
     if (value !== 'no' && noInputMode && !rejectFeedback.trim()) {
       setNoInputMode(false);
     }
     
     setFocusedOption(value);
   }
   ```

### 状态管理

```typescript
const [rejectFeedback, setRejectFeedback] = useState('');
const [acceptFeedback, setAcceptFeedback] = useState('');
const [yesInputMode, setYesInputMode] = useState(false);
const [noInputMode, setNoInputMode] = useState(false);
const [focusedOption, setFocusedOption] = useState('yes');
const [yesFeedbackModeEntered, setYesFeedbackModeEntered] = useState(false);
const [noFeedbackModeEntered, setNoFeedbackModeEntered] = useState(false);
```

### 关键代码路径

```typescript
// 分析服务
import {
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  logEvent,
} from '../../services/analytics/index.js';
import { sanitizeToolNameForAnalytics } from '../../services/analytics/metadata.js';

// 状态管理
import { useSetAppState } from '../../state/AppState.js';

// 权限请求类型
import type { ToolUseConfirm } from './PermissionRequest.js';

// 一元日志工具
import { logUnaryPermissionEvent } from './utils.js';
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `useState` | `react` | React 状态管理 |
| `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | `../../services/analytics/index.js` | 分析元数据类型 |
| `logEvent` | `../../services/analytics/index.js` | 分析日志 |
| `sanitizeToolNameForAnalytics` | `../../services/analytics/metadata.js` | 工具名称清理 |
| `useSetAppState` | `../../state/AppState.js` | 状态更新 |
| `ToolUseConfirm` | `./PermissionRequest.js` | 工具使用确认类型 |
| `logUnaryPermissionEvent` | `./utils.js` | 一元日志辅助函数 |

### 被调用方

该 Hook 被以下组件使用：

1. **BashPermissionRequest.tsx** - Bash 权限请求
   ```typescript
   const {
     yesInputMode,
     noInputMode,
     yesFeedbackModeEntered,
     noFeedbackModeEntered,
     acceptFeedback,
     rejectFeedback,
     setAcceptFeedback,
     setRejectFeedback,
     focusedOption,
     handleInputModeToggle,
     handleReject,
     handleFocus,
   } = useShellPermissionFeedback({
     toolUseConfirm,
     onDone,
     onReject,
     explainerVisible,
   });
   ```

2. **PowerShellPermissionRequest.tsx** - PowerShell 权限请求
   - 使用方式与 BashPermissionRequest 相同

### 数据流

```
BashPermissionRequest / PowerShellPermissionRequest 挂载
  ↓
useShellPermissionFeedback 初始化状态
  ↓
用户按 Tab 键
  ↓
handleInputModeToggle('yes' | 'no')
  ↓
更新输入模式状态 + 记录分析事件
  ↓
用户输入反馈文本
  ↓
setAcceptFeedback / setRejectFeedback
  ↓
用户选择 No 或按 ESC
  ↓
handleReject(feedback?)
  ↓
记录分析事件 + 更新归因 + 调用 onReject
```

## 风险、边界与改进建议

### 当前风险

1. **状态耦合**：
   - 多个相关状态（yesInputMode、acceptFeedback、yesFeedbackModeEntered）
   - 可能导致状态不一致

2. **分析事件重复**：
   - 快速按 Tab 键可能产生大量事件
   - 没有防抖机制

3. **ESC 检测的局限性**：
   - 通过 `!hasFeedback` 推断 ESC 键
   - 如果用户输入后删除，也会被视为 ESC

4. **硬编码的选项值**：
   - `'yes'` 和 `'no'` 是硬编码的
   - 如果选项值改变，需要同步更新

### 边界情况

1. **空反馈文本**：
   - `feedback?.trim()` 处理 undefined 和空白字符
   - 空文本被视为无反馈

2. **焦点快速切换**：
   - 快速切换焦点可能导致输入模式意外退出
   - 如果用户在输入时切换焦点

3. **组件卸载**：
   - 如果组件在输入模式下卸载
   - 状态会丢失，但不会影响父组件

### 改进建议

1. **防抖分析事件**：
   ```typescript
   import { useCallback } from 'react';
   import { debounce } from 'lodash-es';
   
   const debouncedLogEvent = useCallback(
     debounce((event, props) => logEvent(event, props), 300),
     []
   );
   ```

2. **使用 Reducer 管理复杂状态**：
   ```typescript
   type State = {
     yes: { inputMode: boolean; feedback: string; entered: boolean };
     no: { inputMode: boolean; feedback: string; entered: boolean };
     focusedOption: string;
   };
   
   const [state, dispatch] = useReducer(feedbackReducer, initialState);
   ```

3. **明确的 ESC 检测**：
   - 从键盘事件处理器传递明确的 ESC 信号
   - 而不是通过反馈文本推断

4. **配置化选项值**：
   ```typescript
   const OPTION_VALUES = {
     YES: 'yes',
     NO: 'no',
   } as const;
   ```

5. **反馈草稿保存**：
   - 在组件卸载前保存反馈文本
   - 重新挂载时恢复（可选功能）

6. **字符计数**：
   - 显示反馈文本的字符计数
   - 帮助用户控制反馈长度

7. **快捷输入**：
   - 提供常用拒绝原因的快捷选择
   - 如 "Too risky", "Not what I asked", "Need more info"

8. **焦点恢复**：
   - 记录最后聚焦的选项
   - 对话框重新打开时恢复焦点
