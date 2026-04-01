# WizardNavigationFooter.tsx 研究文档

## 场景与职责

`WizardNavigationFooter` 是向导系统的底部导航提示组件，显示在向导对话框下方，提供：

1. **快捷键提示**: 显示导航快捷键（↑↓ 导航、Enter 选择、Esc 返回）
2. **退出确认**: 集成双击 Ctrl+C/D 退出机制，显示二次确认提示
3. **可定制内容**: 支持通过 `instructions` 属性覆盖默认提示
4. **统一视觉**: 使用 dimColor 样式，保持与整体 UI 一致

该组件通常与 `WizardDialogLayout` 配合使用，但也可独立使用。

## 功能点目的

### 1. 默认快捷键提示
显示标准向导导航快捷键：
- `↑↓ to navigate`: 上下箭头导航选项
- `Enter to select`: 回车确认选择
- `Esc to go back`: Esc 返回上一步（通过 ConfigurableShortcutHint 支持自定义）

### 2. 退出确认状态
- 监听 `useExitOnCtrlCDWithKeybindings` 返回的状态
- 当 `exitState.pending` 为 true 时，显示 `Press Ctrl-C again to exit`
- 覆盖默认快捷键提示，引导用户完成退出操作

### 3. 内容定制
- 支持 `instructions` 属性完全覆盖默认提示内容
- 使用 `ReactNode` 类型，支持复杂 JSX 内容
- 默认使用 `Byline` 组件组织多个快捷键提示

## 具体技术实现

### Props 接口

```typescript
type Props = {
  instructions?: ReactNode  // 自定义提示内容
}
```

### 默认提示结构

```typescript
const defaultInstructions = (
  <Byline>
    <KeyboardShortcutHint shortcut="↑↓" action="navigate" />
    <KeyboardShortcutHint shortcut="Enter" action="select" />
    <ConfigurableShortcutHint
      action="confirm:no"
      context="Confirmation"
      fallback="Esc"
      description="go back"
    />
  </Byline>
)
```

### 组件逻辑

```typescript
export function WizardNavigationFooter({
  instructions = defaultInstructions,
}: Props): ReactNode {
  const exitState = useExitOnCtrlCDWithKeybindings()

  return (
    <Box marginLeft={3} marginTop={1}>
      <Text dimColor>
        {exitState.pending
          ? `Press ${exitState.keyName} again to exit`
          : instructions}
      </Text>
    </Box>
  )
}
```

### ExitState 类型

```typescript
type ExitState = {
  pending: boolean                    // 是否等待二次确认
  keyName: 'Ctrl-C' | 'Ctrl-D' | null // 按下的键名
}
```

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `react` | ReactNode 类型 |
| `../../hooks/useExitOnCtrlCDWithKeybindings.js` | 退出状态管理 |
| `../../ink.js` | Box, Text 组件 |
| `../ConfigurableShortcutHint.js` | 可配置快捷键提示 |
| `../design-system/Byline.js` |  middot 分隔的横向布局 |
| `../design-system/KeyboardShortcutHint.js` | 快捷键提示组件 |

### 调用方
| 文件 | 用途 |
|------|------|
| `./WizardDialogLayout.tsx` | 默认渲染在 Dialog 下方 |

### 导出路径
- 主入口：`./index.ts` 统一导出

## 依赖与外部交互

### 快捷键系统
- **useExitOnCtrlCDWithKeybindings**: 提供退出状态
  - 监听全局 Ctrl+C/D 按键
  - 首次按下设置 `pending: true`
  - 超时或二次按下执行退出

### 设计系统组件
- **Box**: ink 布局容器
  - `marginLeft={3}`: 左侧缩进，与 Dialog 内容对齐
  - `marginTop={1}`: 与 Dialog 保持间距

- **Text**: 文本渲染
  - `dimColor`: 使用暗淡颜色，降低视觉权重

- **Byline**: 横向排列多个元素，用 `·` 分隔

- **KeyboardShortcutHint**: 显示 `shortcut to action` 格式

- **ConfigurableShortcutHint**: 支持用户自定义的快捷键提示
  - `action="confirm:no"`: 取消/否操作
  - `context="Confirmation"`: 确认对话框上下文
  - `fallback="Esc"`: 默认快捷键
  - `description="go back"`: 操作描述

## 风险、边界与改进建议

### 已知风险

1. **与 Dialog 快捷键冲突**
   - WizardDialogLayout 设置 `isCancelActive={false}` 禁用 Dialog 的取消快捷键
   - 但 Footer 仍显示 "Esc to go back"
   - 如果步骤组件未正确处理 Esc，提示与实际行为不符
   - 风险等级：中

2. **退出提示覆盖导航提示**
   - 当 `exitState.pending` 为 true 时，完全覆盖 `instructions`
   - 用户可能无法看到导航提示，迷失方向
   - 风险等级：低（退出意图明确）

3. **硬编码的 marginLeft**
   - `marginLeft={3}` 假设 Dialog 有特定的内边距
   - 如果 Dialog 样式变更，可能出现错位
   - 风险等级：低

### 边界情况

1. **instructions 为 null/undefined**
   - 使用默认提示，正常渲染

2. **instructions 为空字符串**
   - 渲染空 Text，占据空间但无内容

3. **exitState.keyName 为 null**
   - 理论上不会发生，因为 `pending` 为 true 时 `keyName` 必赋值
   - 类型系统保证一致性

4. **终端宽度不足**
   - 内容可能换行或截断
   - 依赖 ink 的文本换行处理

### 改进建议

1. **支持多行提示**
   ```typescript
   type Props = {
     instructions?: ReactNode
     secondaryInstructions?: ReactNode  // 第二行提示
   }
   ```

2. **动态快捷键检测**
   ```typescript
   // 根据当前步骤动态显示可用快捷键
   const { currentStepIndex, steps } = useWizard()
   const stepShortcuts = steps[currentStepIndex]?.shortcuts
   ```

3. **退出提示改进**
   ```typescript
   // 保留原提示，追加退出提示
   {exitState.pending && (
     <>
       <Text color="warning">Press {exitState.keyName} again to exit</Text>
       <Newline />
     </>
   )}
   {instructions}
   ```

4. **支持步骤特定提示**
   ```typescript
   // 从 Wizard Context 获取当前步骤的提示
   const { currentStep } = useWizard()
   const stepInstructions = currentStep?.footerInstructions
   ```

5. **添加时间显示**
   ```typescript
   // 显示向导已用时间
   const elapsedTime = useElapsedTime()
   <Text dimColor>Time: {formatTime(elapsedTime)}</Text>
   ```

6. **响应式隐藏**
   ```typescript
   // 终端高度不足时自动隐藏
   const { height } = useTerminalViewport()
   if (height < 10) return null
   ```

7. **动画效果**
   ```typescript
   // 退出提示淡入淡出
   <Text dimColor color={exitState.pending ? 'warning' : undefined}>
     {/* ... */}
   </Text>
   ```

8. **国际化支持**
   ```typescript
   // 支持多语言快捷键描述
   const { t } = useTranslation()
   <KeyboardShortcutHint shortcut="Enter" action={t('wizard.select')} />
   ```
