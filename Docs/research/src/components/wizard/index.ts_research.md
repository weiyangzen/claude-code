# index.ts 研究文档

## 场景与职责

`index.ts` 是 wizard 模块的公共 API 入口文件，负责统一导出向导系统的所有公共类型和组件。它提供：

1. **类型导出**: 暴露 `WizardContextValue`, `WizardProviderProps`, `WizardStepComponent` 类型
2. **Hook 导出**: 暴露 `useWizard` Hook
3. **组件导出**: 暴露 `WizardDialogLayout`, `WizardNavigationFooter`, `WizardProvider` 组件
4. **模块封装**: 简化导入路径，提供清晰的公共接口

该文件是外部模块使用 wizard 系统的唯一入口，遵循 Barrel Export 模式。

## 功能点目的

### 1. 类型系统暴露
导出 TypeScript 类型定义，供调用方进行类型注解：
- **WizardContextValue<T>**: Context 值类型，用于自定义 Hook 或高阶组件
- **WizardProviderProps<T>**: Provider 组件 Props 类型
- **WizardStepComponent<T>**: 步骤组件类型签名

### 2. 运行时 API 暴露
导出可运行的代码：
- **useWizard**: 消费 Wizard Context 的 Hook
- **WizardDialogLayout**: 向导对话框布局组件
- **WizardNavigationFooter**: 底部导航提示组件
- **WizardProvider**: 向导状态管理 Provider

### 3. 模块边界定义
- 仅导出必要的公共 API
- 内部实现（如 types.ts）不直接暴露，通过类型导出间接使用
- 遵循最小暴露原则

## 具体技术实现

### 导出结构

```typescript
// 类型导出（从 types.ts）
export type {
  WizardContextValue,
  WizardProviderProps,
  WizardStepComponent,
} from './types.js'

// Hook 导出
export { useWizard } from './useWizard.js'

// 组件导出
export { WizardDialogLayout } from './WizardDialogLayout.js'
export { WizardNavigationFooter } from './WizardNavigationFooter.js'
export { WizardProvider } from './WizardProvider.js'
```

### 类型定义（推断）

```typescript
// WizardContextValue - Context 值类型
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

// WizardProviderProps - Provider 组件 Props
interface WizardProviderProps<T extends Record<string, unknown>> {
  steps: WizardStepComponent<T>[]
  initialData?: T
  onComplete: (data: T) => void
  onCancel?: () => void
  children?: ReactNode
  title?: string
  showStepCounter?: boolean
}

// WizardStepComponent - 步骤组件类型
type WizardStepComponent<T extends Record<string, unknown> = Record<string, unknown>> = 
  () => ReactNode
```

### 使用示例

```typescript
// 调用方导入示例
import {
  WizardProvider,
  useWizard,
  WizardDialogLayout,
  WizardNavigationFooter,
  type WizardContextValue,
  type WizardProviderProps,
  type WizardStepComponent,
} from '../wizard/index.js'

// 创建步骤组件
const MyStep: WizardStepComponent<MyData> = () => {
  const { goNext, updateWizardData } = useWizard<MyData>()
  // ...
}

// 使用 Provider
<WizardProvider
  steps={[Step1, Step2, Step3]}
  onComplete={(data) => console.log(data)}
  title="My Wizard"
/>
```

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 导出内容 | 用途 |
|------|----------|------|
| `./types.js` | WizardContextValue, WizardProviderProps, WizardStepComponent | 类型定义 |
| `./useWizard.js` | useWizard | Context 消费 Hook |
| `./WizardDialogLayout.js` | WizardDialogLayout | 布局组件 |
| `./WizardNavigationFooter.js` | WizardNavigationFooter | 底部导航组件 |
| `./WizardProvider.js` | WizardProvider, WizardContext | 状态管理 Provider |

### 调用方
| 文件 | 用途 |
|------|------|
| `../agents/new-agent-creation/CreateAgentWizard.tsx` | 创建 Agent 向导 |
| `../agents/new-agent-creation/wizard-steps/*.tsx` | 各步骤组件 |

### 导入路径约定
- 模块内使用相对路径 `./xxx.js`
- 遵循 Node.js ESM 规范，显式指定 `.js` 扩展名
- 类型导入使用 `type` 关键字（TypeScript 4.5+）

## 依赖与外部交互

### 模块系统
- **ES Modules**: 使用 `export` 语法
- **Node.js ESM**: 显式 `.js` 扩展名
- **TypeScript**: 支持类型导出

### 类型系统
- 使用 `export type` 单独导出类型
- 支持泛型类型参数传递
- 与运行时导出分离，便于 tree-shaking

## 风险、边界与改进建议

### 已知风险

1. **types.ts 文件缺失**
   - 源码中引用 `./types.js`，但文件可能不存在于源码目录
   - 类型定义可能在编译时被内联或从其他位置生成
   - 风险等级：低（编译系统处理）

2. **循环依赖风险**
   - `useWizard.ts` 导入 `WizardProvider.js` 的 `WizardContext`
   - `WizardProvider.js` 不直接依赖 `useWizard`
   - 当前结构安全，但新增导出需谨慎

3. **命名空间污染**
   - 所有导出集中在根命名空间
   - 如果未来添加更多组件，可能导致命名冲突
   - 风险等级：低（当前导出数量合理）

### 边界情况

1. **空导出**
   - 如果所有依赖文件缺失，导出语句会报错
   - TypeScript 编译时会检查

2. **类型与值同名**
   - TypeScript 允许类型和值同名
   - 但建议避免混淆，当前无此问题

3. **重新导出变更**
   - 如果子模块移除导出，index.ts 会编译报错
   - 需要同步更新

### 改进建议

1. **命名空间组织**
   ```typescript
   // 按功能分组导出
   export * as Types from './types.js'
   export * as Components from './components.js'
   export * as Hooks from './hooks.js'
   
   // 使用方式
   import { Components, Hooks } from './wizard/index.js'
   const { WizardProvider } = Components
   const { useWizard } = Hooks
   ```

2. **版本兼容性标记**
   ```typescript
   /**
    * @deprecated Use WizardStep instead
    */
   export type WizardStepComponent = /* ... */
   
   export type WizardStep = /* 新名称 */
   ```

3. **运行时类型检查**
   ```typescript
   // 导出运行时类型守卫
   export function isWizardContextValue(obj: unknown): obj is WizardContextValue {
     return obj && typeof (obj as WizardContextValue).goNext === 'function'
   }
   ```

4. **文档注释**
   ```typescript
   /**
    * Wizard context value providing navigation and data management
    * @template T - Type of wizard data object
    */
   export type { WizardContextValue } from './types.js'
   ```

5. **子路径导出**
   ```json
   // package.json
   "exports": {
     ".": "./dist/components/wizard/index.js",
     "./types": "./dist/components/wizard/types.js",
     "./hooks": "./dist/components/wizard/useWizard.js"
   }
   ```

6. **索引签名导出**
   ```typescript
   // 方便动态访问
   export const WIZARD_EXPORTS = {
     WizardProvider,
     useWizard,
     WizardDialogLayout,
     WizardNavigationFooter,
   } as const
   ```
