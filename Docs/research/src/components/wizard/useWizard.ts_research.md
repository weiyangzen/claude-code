# useWizard.ts 研究文档

## 场景与职责

`useWizard` 是一个自定义 React Hook，作为 `WizardProvider` Context 的消费者封装。它提供类型安全的方式来访问向导系统的状态和操作函数。

核心职责：
1. **Context 消费**: 从 `WizardContext` 获取向导状态和方法
2. **类型安全**: 通过泛型参数提供类型推断的 wizardData 访问
3. **错误防护**: 确保 Hook 在 Provider 内部使用，否则抛出明确错误
4. **简化 API**: 封装 `useContext` 调用，提供更简洁的使用接口

## 功能点目的

### 1. Context 访问封装
- 替代直接使用 `useContext(WizardContext)`
- 提供默认泛型参数 `Record<string, unknown>` 保证基础类型安全

### 2. 运行时验证
- 检查 Context 是否为 null
- 抛出明确错误：`useWizard must be used within a WizardProvider`
- 防止在 Provider 外部使用导致的难以调试的问题

### 3. 类型推断支持
- 泛型参数 `T` 自动推断 wizardData 类型
- 支持调用方指定具体数据类型（如 `AgentWizardData`）

## 具体技术实现

### 代码实现

```typescript
import { useContext } from 'react'
import type { WizardContextValue } from './types.js'
import { WizardContext } from './WizardProvider.js'

export function useWizard<
  T extends Record<string, unknown> = Record<string, unknown>,
>(): WizardContextValue<T> {
  const context = useContext(WizardContext) as WizardContextValue<T> | null
  if (!context) {
    throw new Error('useWizard must be used within a WizardProvider')
  }
  return context
}
```

### 类型定义（推断）

```typescript
// WizardContextValue 结构
interface WizardContextValue<T extends Record<string, unknown>> {
  currentStepIndex: number      // 当前步骤索引
  totalSteps: number            // 总步骤数
  wizardData: T                 // 向导数据（泛型）
  setWizardData: (data: T) => void           // 设置完整数据
  updateWizardData: (updates: Partial<T>) => void  // 部分更新
  goNext: () => void           // 前进
  goBack: () => void           // 后退
  goToStep: (index: number) => void  // 跳转到指定步骤
  cancel: () => void           // 取消
  title?: string               // 标题
  showStepCounter?: boolean    // 是否显示步骤计数
}
```

### 使用示例

```typescript
// 基础使用（默认类型）
const { goNext, wizardData, updateWizardData } = useWizard()

// 指定具体类型
const { 
  goNext, 
  goBack, 
  wizardData, 
  updateWizardData,
  currentStepIndex,
  totalSteps 
} = useWizard<AgentWizardData>()

// 在步骤组件中更新数据
updateWizardData({ agentType: 'test-runner' })

// 导航到下一步
goNext()

// 条件跳转
if (wizardData.method === 'manual') {
  goToStep(3)  // 跳过后续步骤
}
```

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `react` | useContext API |
| `./types.js` | WizardContextValue 类型定义 |
| `./WizardProvider.js` | WizardContext 对象 |

### 调用方（典型使用场景）
| 文件 | 用途 |
|------|------|
| `../agents/new-agent-creation/wizard-steps/MethodStep.tsx` | 方法选择步骤 |
| `../agents/new-agent-creation/wizard-steps/TypeStep.tsx` | Agent 类型输入步骤 |
| `../agents/new-agent-creation/wizard-steps/*Step.tsx` | 其他向导步骤 |
| `./WizardDialogLayout.tsx` | 获取标题和步骤计数 |

### 导出路径
- 主入口：`./index.ts` 统一导出

## 依赖与外部交互

### React API
- **useContext**: 订阅 WizardContext

### 内部模块
- **WizardContext**: 由 WizardProvider 创建的 Context 对象
- **WizardContextValue<T>**: 泛型类型定义

### 类型系统
- 使用 TypeScript 泛型提供类型安全
- 默认类型为 `Record<string, unknown>`
- 类型断言 `as WizardContextValue<T> | null` 配合运行时检查

## 风险、边界与改进建议

### 已知风险

1. **Context 创建时的 any 类型**
   - WizardContext 创建时使用 `WizardContextValue<any>`
   - 依赖运行时检查和类型断言保证安全
   - 风险等级：低（有运行时错误抛出）

2. **类型断言依赖**
   - `as WizardContextValue<T> | null` 是类型系统的妥协
   - 如果 Provider 提供的数据结构不匹配，可能导致运行时错误

### 边界情况

1. **Provider 外部使用**
   - 抛出明确错误，阻止继续执行
   - 错误信息清晰，便于调试

2. **泛型参数不匹配**
   - TypeScript 编译时会检查类型兼容性
   - 运行时 JavaScript 无泛型检查，依赖调用方正确使用

### 改进建议

1. **添加开发模式警告**
   ```typescript
   if (process.env.NODE_ENV === 'development' && !context) {
     console.error(
       '[useWizard] WizardContext not found. ' +
       'Ensure your component is wrapped in <WizardProvider>.'
     )
   }
   ```

2. **提供更细粒度的 Hook**
   ```typescript
   // 只读数据 Hook
   export function useWizardData<T>() {
     return useWizard<T>().wizardData
   }
   
   // 只读导航 Hook
   export function useWizardNavigation() {
     const { goNext, goBack, goToStep, cancel, currentStepIndex, totalSteps } = useWizard()
     return { goNext, goBack, goToStep, cancel, currentStepIndex, totalSteps }
   }
   
   // 数据更新 Hook
   export function useWizardDataUpdater<T>() {
     return useWizard<T>().updateWizardData
   }
   ```

3. **支持可选默认值**
   ```typescript
   export function useWizard<T extends Record<string, unknown> = Record<string, unknown>>(
     defaultValue?: Partial<WizardContextValue<T>>
   ): WizardContextValue<T> {
     const context = useContext(WizardContext) as WizardContextValue<T> | null
     if (!context) {
       if (defaultValue) {
         return defaultValue as WizardContextValue<T>
       }
       throw new Error('useWizard must be used within a WizardProvider')
     }
     return context
   }
   ```

4. **添加使用统计（开发模式）**
   ```typescript
   // 用于检测性能问题或过度渲染
   if (process.env.NODE_ENV === 'development') {
     console.count('[useWizard] Render count')
   }
   ```

5. **类型导出优化**
   - 当前 `WizardContextValue` 从 `./types.js` 导入
   - 建议直接在 `useWizard.ts` 中定义并导出，减少模块依赖
