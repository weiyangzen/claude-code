# WizardProvider.tsx 研究文档

## 场景与职责

`WizardProvider` 是 Claude Code CLI 中向导（Wizard）系统的核心状态管理组件，提供多步骤向导流程的上下文管理能力。它负责：

1. **步骤导航管理**：维护当前步骤索引，支持线性前进（`goNext`）、后退（`goBack`）和任意跳转（`goToStep`）
2. **数据收集与共享**：通过 React Context 在向导各步骤间共享和更新数据
3. **导航历史追踪**：支持非线性流程（如条件分支）的历史回溯
4. **生命周期管理**：处理向导完成（`onComplete`）和取消（`onCancel`）回调
5. **退出快捷键集成**：集成 `useExitOnCtrlCDWithKeybindings` 提供统一的退出体验

该组件主要用于创建 Agent 向导（`CreateAgentWizard`）等复杂多步骤交互场景。

## 功能点目的

### 1. 向导状态管理
- **currentStepIndex**: 当前步骤索引，驱动步骤组件渲染
- **wizardData**: 泛型数据存储，收集各步骤的用户输入
- **isCompleted**: 完成标记，触发 `onComplete` 回调
- **navigationHistory**: 历史栈，支持非线性导航的回溯

### 2. 导航操作
- **goNext()**: 前进到下一步，最后一步触发完成
- **goBack()**: 后退，优先使用历史栈，否则递减索引
- **goToStep(index)**: 跳转到指定步骤，当前步骤入栈
- **cancel()**: 清空历史并调用 `onCancel`

### 3. 数据操作
- **setWizardData**: 直接设置完整数据对象
- **updateWizardData(updates)**: 合并式更新部分数据

### 4. 元数据展示
- **title**: 向导标题，可被 `WizardDialogLayout` 覆盖
- **showStepCounter**: 是否显示步骤计数器 (1/N)
- **totalSteps**: 总步骤数

## 具体技术实现

### 关键数据结构

```typescript
// 从 types.ts 推断的类型定义
interface WizardContextValue<T extends Record<string, unknown>> {
  currentStepIndex: number
  totalSteps: number
  wizardData: T
  setWizardData: (data: T) => void
  updateWizardData: (updates: Partial<T>) => void
  goNext: () => void
  goBack: () => void
  goToStep: (index: number) => void
  cancel: () => void
  title?: string
  showStepCounter?: boolean
}

interface WizardProviderProps<T extends Record<string, unknown>> {
  steps: WizardStepComponent<T>[]  // 步骤组件数组
  initialData?: T                  // 初始数据
  onComplete: (data: T) => void   // 完成回调
  onCancel?: () => void           // 取消回调
  children?: ReactNode             // 可选子元素
  title?: string                   // 向导标题
  showStepCounter?: boolean        // 是否显示步骤计数
}

type WizardStepComponent<T> = (props: { wizardData: T }) => ReactNode
```

### 关键流程

#### 1. 初始化流程
```
1. 接受 steps 数组和 initialData
2. 初始化 currentStepIndex = 0
3. 初始化 wizardData = initialData || {}
4. 初始化 navigationHistory = []
5. 注册 Ctrl+C/D 退出快捷键
```

#### 2. 前进流程 (goNext)
```
if currentStepIndex < steps.length - 1:
  if navigationHistory.length > 0:
    将 currentStepIndex 压入 history  // 非线性流程
  currentStepIndex++
else:
  setIsCompleted(true)  // 触发 useEffect 调用 onComplete
```

#### 3. 后退流程 (goBack)
```
if navigationHistory.length > 0:
  previousStep = history.pop()
  currentStepIndex = previousStep
else if currentStepIndex > 0:
  currentStepIndex--
else:
  onCancel?.()  // 第一步时取消
```

#### 4. 跳转流程 (goToStep)
```
if index 在有效范围内:
  将 currentStepIndex 压入 history  // 记录跳转来源
  currentStepIndex = index
```

#### 5. 完成流程
```
isCompleted = true
→ useEffect 触发
→ setNavigationHistory([])  // 清空历史
→ onComplete(wizardData)    // 返回收集的数据
```

### React Compiler 优化

代码使用 React Compiler（`_c` 函数）进行自动记忆化：
- 使用 `$[n]` 数组缓存中间计算结果
- 使用 `Symbol.for("react.memo_cache_sentinel")` 作为初始标记
- 条件比较后决定复用缓存或重新计算

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `./types.js` | 类型定义（WizardContextValue, WizardProviderProps） |
| `../../hooks/useExitOnCtrlCDWithKeybindings.js` | Ctrl+C/D 退出快捷键 |

### 被调用方
| 文件 | 用途 |
|------|------|
| `../agents/new-agent-creation/CreateAgentWizard.tsx` | 创建 Agent 向导 |
| `./useWizard.ts` | 消费 Context 的 Hook |
| `./WizardDialogLayout.tsx` | 获取标题、步骤计数等元数据 |

### 导出
- `WizardContext`: React Context 对象，供 `useWizard` 使用
- `WizardProvider`: 主组件

## 依赖与外部交互

### 外部 Hook
- **useExitOnCtrlCDWithKeybindings**: 提供双击 Ctrl+C/D 退出功能
  - 首次按下显示提示
  - 短时间内再次按下执行退出

### React API
- `createContext`: 创建 WizardContext
- `useState`: 管理 currentStepIndex, wizardData, isCompleted, navigationHistory
- `useCallback`: 记忆化 goNext, goBack, goToStep, cancel, updateWizardData
- `useMemo`: 记忆化 contextValue
- `useEffect`: 处理 isCompleted 副作用

### 类型系统
- 使用泛型 `<T extends Record<string, unknown>>` 提供类型安全的数据管理
- Context 使用 `WizardContextValue<any> | null` 并配合类型断言

## 风险、边界与改进建议

### 已知风险

1. **类型安全妥协**
   - Context 创建时使用 `any` 并配合 eslint-disable
   - 运行时需确保 `useWizard` 在 Provider 内部使用

2. **非线性导航历史溢出**
   - 频繁使用 `goToStep` 可能导致 history 数组无限增长
   - 建议：添加最大历史深度限制

3. **完成回调时序**
   - `onComplete` 在 useEffect 中异步调用
   - 如果组件在调用前卸载，可能导致状态不一致

4. **Ctrl+C/D 冲突**
   - 步骤组件内的文本输入可能需要禁用 Provider 的快捷键
   - 当前通过全局 keybinding 系统处理，但需确保优先级正确

### 边界情况

1. **空 steps 数组**: 渲染 null，无错误处理
2. **initialData 未提供**: 默认空对象 `{} as T`
3. **onCancel 未提供**: 第一步按后退时静默处理
4. **跳转越界**: `goToStep` 检查边界，无效跳转被忽略
5. **完成后再导航**: `isCompleted` 为 true 时渲染 null

### 改进建议

1. **添加调试支持**
   ```typescript
   // 建议添加 development 模式下的日志
   if (process.env.NODE_ENV === 'development') {
     console.log('[Wizard] Step changed:', currentStepIndex)
   }
   ```

2. **历史栈深度限制**
   ```typescript
   const MAX_HISTORY_DEPTH = 50
   setNavigationHistory(prev => 
     [...prev, currentStepIndex].slice(-MAX_HISTORY_DEPTH)
   )
   ```

3. **步骤验证钩子**
   ```typescript
   // 允许步骤组件阻止导航
   const canGoNext = useCallback(() => {
     return steps[currentStepIndex]?.canProceed?.(wizardData) ?? true
   }, [currentStepIndex, wizardData])
   ```

4. **持久化支持**
   - 支持将 wizardData 和 currentStepIndex 持久化到 localStorage
   - 页面刷新后可恢复状态

5. **动画过渡支持**
   - 添加步骤切换动画配置
   - 支持前进/后退不同方向的动画

6. **类型导出优化**
   - 将 `WizardStepComponent` 类型重命名为更直观的 `WizardStep`
   - 提供 `useWizardData<T>()` 等更细粒度的类型安全 Hook
