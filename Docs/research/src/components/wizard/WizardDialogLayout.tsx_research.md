# WizardDialogLayout.tsx 研究文档

## 场景与职责

`WizardDialogLayout` 是向导系统的布局包装组件，为向导步骤提供统一的对话框样式和导航体验。它负责：

1. **统一视觉样式**: 使用 `Dialog` 组件提供一致的边框、颜色和排版
2. **标题管理**: 组合 Provider 标题和步骤计数器（如 "Create new agent (2/10)"）
3. **导航集成**: 将 `goBack` 操作绑定到 Dialog 的取消按钮
4. **底部导航栏**: 渲染 `WizardNavigationFooter` 显示快捷键提示
5. **主题支持**: 支持通过 `color` 属性自定义主题色

该组件是向导步骤的推荐包装器，用于创建 Agent 向导等场景。

## 功能点目的

### 1. 标题生成
- 支持 `title` 属性覆盖 Provider 提供的标题
- 默认标题为 "Wizard"
- 根据 `showStepCounter` 自动添加步骤后缀 `(current/total)`

### 2. 对话框配置
- **onCancel**: 绑定到 `goBack`，支持 Esc 键返回
- **color**: 默认使用 `suggestion` 主题色
- **hideInputGuide**: 隐藏 Dialog 默认的输入提示（由 Footer 替代）
- **isCancelActive**: 设为 false，避免与步骤内组件冲突

### 3. 子元素渲染
- 渲染步骤组件的内容（children）
- 支持自定义副标题（subtitle）

### 4. 底部导航
- 渲染 `WizardNavigationFooter`
- 支持自定义 `footerText` 覆盖默认快捷键提示

## 具体技术实现

### Props 接口

```typescript
type Props = {
  title?: string           // 覆盖 Provider 标题
  color?: keyof Theme      // 主题色，默认 'suggestion'
  children: ReactNode      // 步骤内容
  subtitle?: string        // 副标题
  footerText?: ReactNode   // 自定义底部提示
}
```

### 组件逻辑

```typescript
export function WizardDialogLayout({
  title: titleOverride,
  color = 'suggestion',
  children,
  subtitle,
  footerText,
}: Props): ReactNode {
  // 从 Wizard Context 获取状态
  const {
    currentStepIndex,
    totalSteps,
    title: providerTitle,
    showStepCounter,
    goBack,
  } = useWizard()
  
  // 标题优先级：props.title > provider.title > 'Wizard'
  const title = titleOverride || providerTitle || 'Wizard'
  
  // 步骤计数后缀
  const stepSuffix = showStepCounter !== false 
    ? ` (${currentStepIndex + 1}/${totalSteps})` 
    : ''
  
  return (
    <>
      <Dialog
        title={`${title}${stepSuffix}`}
        subtitle={subtitle}
        onCancel={goBack}
        color={color}
        hideInputGuide
        isCancelActive={false}
      >
        {children}
      </Dialog>
      <WizardNavigationFooter instructions={footerText} />
    </>
  )
}
```

### React Compiler 优化

代码使用 React Compiler 进行自动记忆化：
- 使用 `$[n]` 数组缓存 JSX 元素
- 条件比较决定是否复用缓存
- 减少不必要的重新渲染

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `react` | ReactNode 类型 |
| `../../utils/theme.js` | Theme 类型定义 |
| `../design-system/Dialog.js` | 对话框基础组件 |
| `./useWizard.js` | 获取向导状态和方法 |
| `./WizardNavigationFooter.js` | 底部导航栏 |

### 调用方
| 文件 | 用途 |
|------|------|
| `../agents/new-agent-creation/wizard-steps/MethodStep.tsx` | 方法选择步骤布局 |
| `../agents/new-agent-creation/wizard-steps/TypeStep.tsx` | 类型输入步骤布局 |
| `../agents/new-agent-creation/wizard-steps/*Step.tsx` | 其他步骤布局 |

### 导出路径
- 主入口：`./index.ts` 统一导出

## 依赖与外部交互

### 设计系统组件
- **Dialog**: 提供带边框的对话框容器
  - `title`: 组合后的标题
  - `subtitle`: 副标题
  - `onCancel`: 后退操作
  - `color`: 主题色
  - `hideInputGuide`: 隐藏默认输入提示
  - `isCancelActive`: 禁用 Dialog 的取消快捷键（避免冲突）

### Wizard Context
- **currentStepIndex**: 当前步骤索引（用于计数器）
- **totalSteps**: 总步骤数
- **title**: Provider 提供的标题
- **showStepCounter**: 是否显示计数器
- **goBack**: 后退操作

### 主题系统
- 使用 `keyof Theme` 类型约束颜色选择
- 默认 `suggestion` 主题色（蓝色系）

## 风险、边界与改进建议

### 已知风险

1. **isCancelActive=false 的副作用**
   - 禁用 Dialog 的取消快捷键，依赖步骤组件自行处理
   - 如果步骤组件未正确处理 Esc，用户无法后退
   - 风险等级：中（需要步骤组件配合）

2. **标题覆盖优先级**
   - `props.title` 完全覆盖 `provider.title`
   - 无法叠加（如保留 Provider 标题并添加后缀）
   - 风险等级：低（设计意图）

3. **步骤计数器边界**
   - `currentStepIndex + 1` 假设索引从 0 开始
   - 如果 Provider 逻辑变更，此处可能出错
   - 风险等级：低（内部约定）

### 边界情况

1. **无 Provider 包裹**
   - `useWizard()` 会抛出错误
   - 组件无法正常渲染

2. **children 为 null/undefined**
   - Dialog 仍渲染，显示空内容
   - 建议调用方确保 children 有效

3. **totalSteps 为 0**
   - 显示 `(1/0)`，虽然 Provider 会处理为空的情况

4. **showStepCounter 未定义**
   - 使用 `!== false` 判断，默认显示计数器

### 改进建议

1. **添加步骤进度条**
   ```typescript
   // 在 Dialog 标题下方添加可视化进度
   <ProgressBar current={currentStepIndex + 1} total={totalSteps} />
   ```

2. **支持自定义标题模板**
   ```typescript
   type Props = {
     titleTemplate?: (title: string, current: number, total: number) => string
   }
   // 默认: (t, c, total) => `${t} (${c}/${total})`
   ```

3. **添加步骤描述提示**
   ```typescript
   type Props = {
     stepDescription?: string  // 当前步骤的简短描述
   }
   // 显示在副标题位置或 Tooltip 中
   ```

4. **支持动画过渡**
   ```typescript
   type Props = {
     transition?: 'slide' | 'fade' | 'none'
     transitionDirection?: 'forward' | 'backward'
   }
   ```

5. **底部导航可配置**
   ```typescript
   type Props = {
     hideFooter?: boolean           // 完全隐藏底部
     footerPosition?: 'bottom' | 'right'  // 布局选项
   }
   ```

6. **错误边界集成**
   ```typescript
   // 包装 children 在 ErrorBoundary 中
   // 步骤渲染错误不会导致整个向导崩溃
   <ErrorBoundary fallback={<StepError />}>  
     {children}
   </ErrorBoundary>
   ```

7. **响应式布局**
   ```typescript
   // 根据终端宽度调整布局
   const { width } = useTerminalViewport()
   const compactMode = width < 80
   ```
