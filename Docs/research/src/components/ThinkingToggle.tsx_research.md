# ThinkingToggle.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`ThinkingToggle` 是 Claude Code CLI 中用于**切换 Extended Thinking（深度思考）模式**的终端 UI 组件。它以模态浮层（modal-like overlay）的形式出现在 REPL 输入区上方，允许用户在不离开当前会话的前提下，快速开启或关闭 Claude 的扩展思考能力。

### 1.2 使用场景
- **热键触发切换**：用户在聊天界面按下 `meta+t`（默认快捷键，对应 `chat:thinkingToggle`）时，PromptInput 会渲染该组件
- **会话中动态调整**：用户可以在对话的任何阶段修改 thinking 设置，但系统会针对"对话中途修改"给出二次确认警告
- **启动时默认状态**：组件的 `currentValue` 由上层 `AppState.thinkingEnabled` 驱动，初始值通常为 `true`

### 1.3 调用方与挂载位置
该组件由 `src/components/PromptInput/PromptInput.tsx` 条件渲染：

```tsx
const [showThinkingToggle, setShowThinkingToggle] = useState(false);

// 热键 handler
const handleThinkingToggle = useCallback(() => {
  setShowThinkingToggle(prev => !prev);
}, []);

// 渲染片段
const thinkingToggleElement = useMemo(() => {
  if (!showThinkingToggle) return null;
  return (
    <Box flexDirection="column" marginTop={1}>
      <ThinkingToggle
        currentValue={thinkingEnabled ?? true}
        onSelect={handleThinkingSelect}
        onCancel={handleThinkingCancel}
        isMidConversation={messages.some(m => m.type === 'assistant')}
      />
    </Box>
  );
}, [showThinkingToggle, thinkingEnabled, handleThinkingSelect, handleThinkingCancel, messages.length]);
```

当 `showThinkingToggle` 为 `true` 时，组件挂载并自动注册为 overlay（通过内部 `Select` 组件的 `useRegisterOverlay`），从而：
- 禁用 PromptInput 的文本输入焦点
- 阻止 `CancelRequestHandler` 将 `Escape` 误识别为取消请求

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| 状态切换 | 在 Enabled / Disabled 两个选项间切换 thinking 模式 |
| 中途确认保护 | 当 `isMidConversation` 为 `true` 且用户改变当前值时，弹出二次确认，防止用户因不了解后果而降低对话质量 |
| 键盘导航 | 支持 `↑/↓` 选择选项，`Enter` 确认，`Esc` 退出/取消 |
| 可配置快捷键提示 | Footer 显示当前用户自定义的快捷键（而非写死的 `Esc`），兼容 keybindings.json 覆盖 |
| 退出手势兼容 | 集成 `useExitOnCtrlCDWithKeybindings`，支持 `Ctrl+D/Ctrl+C` 双按退出 |

### 2.2 Props 接口

```typescript
export type Props = {
  currentValue: boolean;           // 当前 thinking 状态
  onSelect: (enabled: boolean) => void;  // 用户确认选择后的回调
  onCancel?: () => void;           // 取消/关闭回调
  isMidConversation?: boolean;     // 是否处于对话中途（用于触发确认页）
};
```

### 2.3 选项定义

组件内部硬编码了两个选项：

```typescript
const options = [
  { value: 'true',  label: 'Enabled',  description: 'Claude will think before responding' },
  { value: 'false', label: 'Disabled', description: 'Claude will respond without extended thinking' },
];
```

注意 `value` 为字符串类型，`onChange` 时通过 `value === 'true'` 转回 `boolean`。

---

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 状态机：选择 vs 确认

组件内部仅维护一个状态 `confirmationPending`：

```typescript
const [confirmationPending, setConfirmationPending] = useState<boolean | null>(null);
```

状态流转如下：

```
初始状态
   │
   ▼
┌─────────────────┐
│  Select 列表页   │ ← 显示 Enabled/Disabled 选项
│  (confirmationPending === null)
└─────────────────┘
   │ 用户改变选项且 isMidConversation === true
   ▼
┌─────────────────┐
│  确认警告页      │ ← 显示 latency/quality 警告
│  (confirmationPending === true/false)
└─────────────────┘
   │ Enter (confirm:yes)
   ▼
调用 onSelect(confirmationPending)
   │ Esc/n (confirm:no)
   ▼
返回 Select 列表页（setConfirmationPending(null)）
```

### 3.2 选择变更处理

```typescript
const handleSelectChange = (value: string) => {
  const selected = value === 'true';
  if (isMidConversation && selected !== currentValue) {
    setConfirmationPending(selected);
  } else {
    onSelect(selected);
  }
};
```

**关键逻辑**：
- 若不在对话中途（`!isMidConversation`），直接触发 `onSelect`
- 若在对话中途但选择的是**当前相同值**（`selected === currentValue`），也直接触发 `onSelect`（无副作用）
- 只有"对话中途 + 改变值"才会进入确认页

### 3.3 键盘绑定与上下文隔离

组件注册了两个 `Confirmation` 上下文的按键绑定：

```typescript
// 取消 / 返回
useKeybinding('confirm:no', handleCancel, { context: 'Confirmation' });

// 确认（仅在确认页激活）
useKeybinding('confirm:yes', handleConfirm, {
  context: 'Confirmation',
  isActive: confirmationPending !== null,
});
```

`handleCancel` 的行为：
- 如果在确认页 → 返回列表页（`setConfirmationPending(null)`）
- 如果在列表页 → 调用 `onCancel?.()` 关闭整个浮层

`handleConfirm` 的行为：
- 仅在 `confirmationPending !== null` 时生效，调用 `onSelect(confirmationPending)`

### 3.4 默认快捷键映射

来自 `src/keybindings/defaultBindings.ts` 的 `Confirmation` 上下文：

| 按键 | action | 在 ThinkingToggle 中的行为 |
|------|--------|---------------------------|
| `y` | `confirm:yes` | 确认修改（仅在确认页有效） |
| `Enter` | `confirm:yes` | 同上 |
| `n` | `confirm:no` | 取消/返回 |
| `Escape` | `confirm:no` | 取消/返回 |
| `↑/↓` | `confirm:previous/next` | 由 `Select` 组件内部消费，用于选项导航 |

### 3.5 UI 渲染结构

组件最终渲染结构（基于编译后代码还原）：

```tsx
<Pane color="permission">
  {/* 标题区 */}
  <Box marginBottom={1} flexDirection="column">
    <Text color="remember" bold>Toggle thinking mode</Text>
    <Text dimColor>Enable or disable thinking for this session.</Text>
  </Box>

  {/* 内容区：确认页 或 选择列表 */}
  <Box flexDirection="column">
    {confirmationPending !== null ? (
      <Box flexDirection="column" marginBottom={1} gap={1}>
        <Text color="warning">
          Changing thinking mode mid-conversation will increase latency and may reduce quality...
        </Text>
        <Text color="warning">Do you want to proceed?</Text>
      </Box>
    ) : (
      <Box flexDirection="column" marginBottom={1}>
        <Select
          defaultValue={currentValue ? 'true' : 'false'}
          defaultFocusValue={currentValue ? 'true' : 'false'}
          options={options}
          onChange={handleSelectChange}
          onCancel={onCancel ?? (() => {})}
          visibleOptionCount={2}
        />
      </Box>
    )}
  </Box>

  {/* Footer 快捷键提示 */}
  <Text dimColor italic>
    {exitState.pending ? (
      <>Press {exitState.keyName} again to exit</>
    ) : confirmationPending !== null ? (
      <Byline>
        <KeyboardShortcutHint shortcut="Enter" action="confirm" />
        <ConfigurableShortcutHint
          action="confirm:no"
          context="Confirmation"
          fallback="Esc"
          description="cancel"
        />
      </Byline>
    ) : (
      <Byline>
        <KeyboardShortcutHint shortcut="Enter" action="confirm" />
        <ConfigurableShortcutHint
          action="confirm:no"
          context="Confirmation"
          fallback="Esc"
          description="exit"
        />
      </Byline>
    )}
  </Text>
</Pane>
```

### 3.6 退出手势集成

```typescript
const exitState = useExitOnCtrlCDWithKeybindings();
```

该 hook 将 `useExitOnCtrlCD` 与 `useKeybindings` 桥接，提供：
- `exitState.pending`：用户已按一次 `Ctrl+D`（或 `Ctrl+C`），正在等待第二次按键
- `exitState.keyName`：显示在提示中的按键名称（如 `"Ctrl+D"`）

当 `exitState.pending` 为 `true` 时，Footer 会覆盖常规快捷键提示，显示 `"Press Ctrl+D again to exit"`。

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置
```
src/components/ThinkingToggle.tsx
```

### 4.2 依赖关系图

```
ThinkingToggle.tsx
├── react (React 核心 / useState)
├── src/hooks/useExitOnCtrlCDWithKeybindings.ts
│   └── useExitOnCtrlCD.ts (双按退出逻辑)
├── ../ink.js
│   └── Box, Text (Ink TUI 组件)
├── ../keybindings/useKeybinding.ts
│   └── useKeybinding / useKeybindings (按键绑定注册)
├── ./ConfigurableShortcutHint.tsx
│   └── useShortcutDisplay (读取用户自定义快捷键)
├── ./CustomSelect/index.ts
│   └── Select (列表选择器)
│       └── useRegisterOverlay('select', !!onCancel) ← 注册 overlay
├── ./design-system/Byline.tsx
├── ./design-system/KeyboardShortcutHint.tsx
└── ./design-system/Pane.tsx
```

### 4.3 调用链（从热键到回调）

```
用户按下 meta+t
    │
    ▼
PromptInput.tsx: handleThinkingToggle()
    │
    ▼
setShowThinkingToggle(true)
    │
    ▼
ThinkingToggle 挂载
    │
    ▼
Select 组件内部调用 useRegisterOverlay('select', true)
    │
    ▼
AppState.activeOverlays 加入 'select'
    │
    ▼
useIsModalOverlayActive() 返回 true
    │
    ▼
PromptInput 的 chatHandlers 被禁用（isActive: !isModalOverlayActive）
CancelRequestHandler 的 Escape 处理被禁用（isOverlayActive === true）
    │
    ▼
用户在 Select 中按 Enter / ↑ / ↓ / Esc
    │
    ▼
ThinkingToggle 的 onSelect / onCancel 被触发
    │
    ▼
PromptInput 更新 AppState.thinkingEnabled 并关闭浮层
```

### 4.4 上层状态更新代码

```tsx
// PromptInput.tsx
const handleThinkingSelect = useCallback((enabled: boolean) => {
  setAppState(prev => ({ ...prev, thinkingEnabled: enabled }));
  setShowThinkingToggle(false);
  logEvent('tengu_thinking_toggled_hotkey', { enabled });
  addNotification({
    key: 'thinking-toggled-hotkey',
    jsx: <Text color={enabled ? 'suggestion' : undefined} dimColor={!enabled}>
          Thinking {enabled ? 'on' : 'off'}
        </Text>,
    priority: 'immediate',
    timeoutMs: 3000
  });
}, [setAppState, addNotification]);
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖清单

| 依赖 | 路径 | 用途 |
|------|------|------|
| `React` / `useState` | `'react'` | 组件运行时、本地状态管理 |
| `useExitOnCtrlCDWithKeybindings` | `src/hooks/useExitOnCtrlCDWithKeybindings.ts` | 双按 `Ctrl+D/C` 退出提示 |
| `Box`, `Text` | `../ink.js` | Ink 终端 UI 布局与文本渲染 |
| `useKeybinding` | `../keybindings/useKeybinding.js` | 注册 `confirm:yes` / `confirm:no` 按键处理器 |
| `ConfigurableShortcutHint` | `./ConfigurableShortcutHint.js` | 显示用户自定义的取消快捷键 |
| `Select` | `./CustomSelect/index.js` | 选项列表渲染与键盘导航 |
| `Byline` | `./design-system/Byline.js` | Footer 快捷键提示的中点分隔排版 |
| `KeyboardShortcutHint` | `./design-system/KeyboardShortcutHint.js` | 渲染 `"Enter to confirm"` 等提示 |
| `Pane` | `./design-system/Pane.js` | 带顶部彩色分割线的容器 |

### 5.2 与全局按键系统的交互

`useKeybinding` 在 `Confirmation` 上下文中注册处理器。解析优先级（来自 `useKeybinding.ts`）：

```typescript
const contextsToCheck: KeybindingContextName[] = [
  ...keybindingContext.activeContexts, // 已激活的上下文（如 Select 内部可能额外注册）
  context,                             // 当前传入的 'Confirmation'
  'Global',                            // 全局兜底
];
```

这意味着在 ThinkingToggle 浮层打开时：
- `Confirmation` 上下文的 `Escape` 会优先于 `Chat` 上下文的 `chat:cancel` 被解析
- `Select` 组件内部也可能注册 `Select` 上下文，用于 `↑/↓/Enter` 导航

### 5.3 与 Overlay 系统的交互

ThinkingToggle **本身不直接调用** `useRegisterOverlay`，而是通过内部嵌套的 `Select` 组件间接注册：

```typescript
// CustomSelect/use-select-input.ts（推断）
useRegisterOverlay('select', !!state.onCancel);
```

这带来一个边界情况：当 `onCancel` 为 `undefined` 时，`Select` 不会注册 overlay。但在实际调用中，PromptInput 始终传入了 `handleThinkingCancel`，因此 overlay 注册是可靠的。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 严重程度 |
|------|------|----------|
| 中途切换质量下降 | 用户可能在未阅读警告的情况下确认切换，导致后续回答质量降低 | 中 |
| 快捷键冲突 | 若用户将 `confirm:no` 绑定到非 `Esc` 键，可能与 Select 导航键冲突 | 低 |
| Overlay 泄漏 | 若 `Select` 的 `onCancel` 为 `undefined`，不会注册 overlay，可能导致 Escape 被下层误消费 | 低 |
| 编译后代码可读性 | 文件经过 React Compiler 编译，包含大量 `_c` memoization 代码，直接阅读困难 | 低（开发影响） |

### 6.2 边界情况

1. **`isMidConversation` 判断逻辑**
   - 上层通过 `messages.some(m => m.type === 'assistant')` 判断
   - 这意味着只要历史记录中有一条 assistant 消息，就算"对话中途"
   - 边界：如果用户清空了屏幕但消息仍在状态中，仍会触发确认

2. **选择相同值**
   - `handleSelectChange` 中 `selected !== currentValue` 的判断确保了选择相同值时不会进入确认页
   - 这符合直觉：重新选择当前状态不需要确认

3. **`confirmationPending` 的布尔语义**
   - `null` = 列表页
   - `true/false` = 确认页，同时保存了用户最终想要切换到的目标值
   - 该设计用一个状态同时表达了"页面模式"和"目标值"

4. **`visibleOptionCount={2}`**
   - 由于只有两项，该设置确保不需要滚动条，所有选项始终可见

5. **`_temp` 空函数**
   - 编译产物中为 `onCancel` 提供了默认空函数 `_temp() {}`
   - 这是 React Compiler 的降级处理，确保 `onCancel` 在 JSX 中始终有值

### 6.3 改进建议

#### 6.3.1 增强中途切换的上下文感知

当前 `isMidConversation` 仅基于是否存在 assistant 消息，可以更精确：

```typescript
// PromptInput.tsx 中
const isMidConversation = messages.some(
  m => m.type === 'assistant' && m.role !== 'system'
);

// 或者引入消息时间戳/轮数判断
const conversationTurns = messages.filter(m => m.type === 'assistant').length;
const isMidConversation = conversationTurns >= 2; // 至少两轮后再警告
```

#### 6.3.2 将确认逻辑提取为可复用 Hook

```typescript
function useMidConversationConfirmation<T>(
  currentValue: T,
  isMidConversation: boolean,
  onConfirm: (value: T) => void
) {
  const [pending, setPending] = useState<T | null>(null);

  const handleChange = useCallback((value: T) => {
    if (isMidConversation && value !== currentValue) {
      setPending(value);
    } else {
      onConfirm(value);
    }
  }, [isMidConversation, currentValue, onConfirm]);

  const confirm = useCallback(() => {
    if (pending !== null) {
      onConfirm(pending);
    }
  }, [pending, onConfirm]);

  const cancel = useCallback(() => {
    if (pending !== null) {
      setPending(null);
      return true; // 表示已消费（停留在组件内）
    }
    return false; // 表示应关闭浮层
  }, [pending]);

  return { pending, handleChange, confirm, cancel };
}
```

#### 6.3.3 显式注册 Overlay

虽然 `Select` 内部会注册 overlay，但 `ThinkingToggle` 作为浮层容器，自身也应显式注册，以防未来替换 `Select` 为其他输入控件时丢失 overlay 保护：

```typescript
import { useRegisterOverlay } from '../context/overlayContext.js';

export function ThinkingToggle(props: Props) {
  useRegisterOverlay('thinking-toggle', true);
  // ...
}
```

#### 6.3.4 分析事件补充

当前上层 `PromptInput` 在 `onSelect` 时记录了 `tengu_thinking_toggled_hotkey`，但缺少：
- 用户看到确认页后取消的事件（可用于评估警告有效性）
- 用户通过确认页最终确认切换的事件（与直接切换区分）

建议新增：

```typescript
// 在 handleConfirm 中
logEvent('tengu_thinking_toggled_confirmed', {
  enabled: confirmationPending,
  midConversation: isMidConversation,
});

// 在 handleCancel 返回列表页时
logEvent('tengu_thinking_toggle_cancelled', {
  stage: confirmationPending !== null ? 'confirmation' : 'selection',
});
```

#### 6.3.5 国际化支持

当前标题、描述、警告文本均为硬编码英文：

```typescript
const options = [
  { value: 'true', label: 'Enabled', description: 'Claude will think before responding' },
  { value: 'false', label: 'Disabled', description: 'Claude will respond without extended thinking' },
];
```

建议引入轻量级 i18n 或至少将文案提取为常量，便于后续本地化：

```typescript
const THINKING_TOGGLE_COPY = {
  title: 'Toggle thinking mode',
  subtitle: 'Enable or disable thinking for this session.',
  enabled: { label: 'Enabled', description: 'Claude will think before responding' },
  disabled: { label: 'Disabled', description: 'Claude will respond without extended thinking' },
  warning: 'Changing thinking mode mid-conversation will increase latency and may reduce quality...',
  proceedQuestion: 'Do you want to proceed?',
} as const;
```

#### 6.3.6 测试覆盖建议

| 测试场景 | 期望行为 |
|----------|----------|
| 非对话中途选择 Enabled | 直接调用 `onSelect(true)` |
| 非对话中途选择 Disabled | 直接调用 `onSelect(false)` |
| 对话中途选择相同值 | 直接调用 `onSelect(currentValue)`，不进入确认页 |
| 对话中途改变值 | 进入确认页，`confirmationPending` 为目标值 |
| 确认页按 Enter | 调用 `onSelect(pendingValue)` |
| 确认页按 Esc | 返回列表页，`confirmationPending` 重置为 `null` |
| 列表页按 Esc | 调用 `onCancel()` |
| 打开浮层时 | `Select` 自动聚焦当前值对应的选项 |

### 6.4 相关配置与标志

- **默认热键**：`meta+t` → `chat:thinkingToggle`（`src/keybindings/defaultBindings.ts`）
- **分析事件前缀**：`tengu_thinking_toggled_*`
- **状态字段**：`AppState.thinkingEnabled`（布尔值，默认 `true`）
- **颜色主题**：`Pane` 使用 `color="permission"`，警告文本使用 `color="warning"`
