# Dialog.tsx 研究文档

## 场景与职责

Dialog 是一个通用的确认/取消对话框组件，用于在终端 UI 中显示模态对话框。它是 Claude Code 中各种权限请求、确认对话框的基础组件。

核心职责：
- 提供带边框的模态对话框容器
- 内置取消快捷键（Esc/n）和退出快捷键（Ctrl+C/D）处理
- 显示标题、副标题和自定义内容
- 提供默认的输入指南（Enter 确认、Esc 取消、Ctrl+C/D 退出）
- 支持自定义输入指南

## 功能点目的

### 1. 模态对话框容器
- 使用 Pane 组件提供带颜色的顶部边框
- 支持 `hideBorder` 选项，用于嵌套在 Pane 内部时避免双边框
- 支持自定义主题颜色

### 2. 快捷键处理
- **确认取消**: 注册 `confirm:no` 快捷键（默认 Esc/n）调用 `onCancel`
- **退出处理**: 通过 `useExitOnCtrlCDWithKeybindings` 处理 Ctrl+C/D 双按退出
- **可禁用**: `isCancelActive` 属性可在嵌入文本输入时禁用内置快捷键，让按键传递给输入框

### 3. 输入指南显示
- 默认显示 "Enter to confirm · Esc to cancel" 样式的提示
- 支持 Ctrl+C/D 待处理状态的自定义显示（"Press Ctrl+C again to exit"）
- 支持完全自定义输入指南内容
- 支持隐藏输入指南

### 4. 布局结构
```
┌─────────────────────────────────┐  ← Pane 边框（可选）
│  Title                          │  ← 粗体标题（主题色）
│  Subtitle                       │  ← 副标题（暗淡色，可选）
│                                 │
│  [自定义内容区域]                │  ← children
│                                 │
│  Enter to confirm · Esc to cancel │ ← 输入指南（可选）
└─────────────────────────────────┘
```

## 具体技术实现

### 关键数据结构

```typescript
type DialogProps = {
  title: React.ReactNode;
  subtitle?: React.ReactNode;
  children: React.ReactNode;
  onCancel: () => void;
  color?: keyof Theme;           // 默认 "permission"
  hideInputGuide?: boolean;
  hideBorder?: boolean;
  inputGuide?: (exitState: ExitState) => React.ReactNode;
  isCancelActive?: boolean;      // 默认 true
};

type ExitState = {
  pending: boolean;    // 是否处于待确认退出状态
  keyName: string;     // "Ctrl+C" 或 "Ctrl+D"
};
```

### 核心实现逻辑

1. **退出状态管理**
   ```tsx
   const exitState = useExitOnCtrlCDWithKeybindings(undefined, undefined, isCancelActive);
   ```
   - 使用 `useExitOnCtrlCDWithKeybindings` hook 处理双按退出逻辑
   - 当 `isCancelActive` 为 false 时，禁用退出快捷键

2. **快捷键注册**
   ```tsx
   useKeybinding("confirm:no", onCancel, {
     context: "Confirmation",
     isActive: isCancelActive
   });
   ```
   - 在 "Confirmation" 上下文中注册取消快捷键
   - 可通过 `isCancelActive` 动态启用/禁用

3. **默认输入指南生成**
   ```tsx
   const defaultInputGuide = exitState.pending 
     ? <Text>Press {exitState.keyName} again to exit</Text>
     : <Byline>
         <KeyboardShortcutHint shortcut="Enter" action="confirm" />
         <ConfigurableShortcutHint action="confirm:no" context="Confirmation" fallback="Esc" description="cancel" />
       </Byline>;
   ```
   - 根据退出状态显示不同内容
   - 使用 ConfigurableShortcutHint 支持用户自定义快捷键显示

4. **布局渲染**
   - 使用 React Compiler 缓存优化渲染性能
   - 条件渲染副标题、输入指南
   - 根据 `hideBorder` 决定是否包裹 Pane

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/design-system/Dialog.tsx`

### 依赖文件

| 文件 | 用途 |
|------|------|
| `../../hooks/useExitOnCtrlCDWithKeybindings.js` | 双按退出逻辑 |
| `../../ink.js` (Box, Text) | 布局容器和文本 |
| `../../keybindings/useKeybinding.js` | 快捷键注册 |
| `../../utils/theme.js` (Theme) | 主题类型 |
| `../ConfigurableShortcutHint.js` | 可配置快捷键提示 |
| `./Byline.js` | 分隔符连接 |
| `./KeyboardShortcutHint.js` | 键盘快捷键提示 |
| `./Pane.js` | 带边框的面板容器 |

### 调用方（部分重要文件）

| 文件 | 用途 |
|------|------|
| `src/components/TrustDialog/TrustDialog.tsx` | 信任确认对话框 |
| `src/components/permissions/*` | 各类权限请求对话框 |
| `src/components/MCPServerApprovalDialog.tsx` | MCP 服务器审批 |
| `src/components/ExitFlow.tsx` | 退出流程对话框 |
| `src/components/CostThresholdDialog.tsx` | 成本阈值提示 |
| `src/components/diff/DiffDialog.tsx` | Diff 查看对话框 |
| `src/components/Feedback.tsx` | 反馈对话框 |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导对话框布局 |

## 依赖与外部交互

### 快捷键系统交互

Dialog 与快捷键系统深度集成：

1. **useExitOnCtrlCDWithKeybindings**
   - 处理 Ctrl+C/D 的双按退出逻辑
   - 第一次按下进入 pending 状态
   - 第二次按下执行退出
   - 其他按键取消 pending 状态

2. **useKeybinding**
   - 注册 `confirm:no` 动作的处理函数
   - 使用 "Confirmation" 上下文
   - 支持通过 `isActive` 动态禁用

### 主题系统

- 默认使用 `permission` 颜色（蓝色系）
- 支持所有 Theme 中定义的颜色键
- 通过 Text 组件的 `color` 和 `dimColor` 属性应用主题

## 风险、边界与改进建议

### 潜在风险

1. **快捷键冲突**
   - 当 `isCancelActive` 为 true 时，Dialog 会捕获 Esc/n 和 Ctrl+C/D
   - 嵌入的输入框需要设置 `isCancelActive={false}` 才能让按键传递给输入框
   - 如果忘记设置，会导致输入框无法正常使用这些按键

2. **React Compiler 依赖**
   - 代码经过 React Compiler 编译，包含大量缓存逻辑
   - 直接修改源码需要重新编译或理解编译器模式

3. **输入指南与退出状态耦合**
   - 默认输入指南依赖 exitState，无法完全独立自定义
   - 虽然提供了 `inputGuide` 属性，但 exitState 的传递是强制的

### 边界情况

1. **hideBorder 与 Pane 嵌套**
   - 当 Dialog 嵌套在 Pane 内部时，应设置 `hideBorder={true}`
   - 否则会出现双边框

2. **isCancelActive 状态切换**
   - 动态切换 `isCancelActive` 时，快捷键注册/注销是异步的
   - 快速切换可能导致按键被错误处理

3. **onCancel 回调**
   - 必须提供有效的 `onCancel` 回调
   - 组件不负责处理 Enter 确认，由父组件决定

### 改进建议

1. **增强类型安全**
   ```typescript
   // 可以添加更严格的 children 类型
   children: ReactElement<typeof SomeInputComponent> | ReactElement<typeof SomeContent>
   ```

2. **支持更多快捷键动作**
   - 当前只支持 `confirm:no`，可以添加 `confirm:yes` 的默认处理
   - 或者提供 `onConfirm` 属性，自动注册 Enter 快捷键

3. **输入指南模板化**
   - 提供更灵活的输入指南模板系统
   - 支持完全替换默认指南，而不只是追加

4. **无障碍支持**
   - 添加 ARIA 属性支持
   - 支持屏幕阅读器 announcement

5. **动画支持**
   - 添加进入/退出动画选项
   - 支持 Ink 的动画系统

### 测试注意事项

- 测试 `isCancelActive` 切换时快捷键的正确性
- 验证 `hideBorder` 在各种嵌套场景下的表现
- 测试 exitState pending 状态的显示逻辑
- 验证自定义 `inputGuide` 是否正确渲染
- 测试快捷键冲突场景（特别是与 TextInput 一起使用时）
