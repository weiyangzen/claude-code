# use-app.ts 深入研究

## 场景与职责

`useApp` 是 Ink 终端 UI 框架中最简单的 Hook 之一，其唯一职责是暴露应用级别的退出方法。它作为 `AppContext` 的消费者，为组件树中的任何组件提供程序化退出应用的能力。

## 功能点目的

### 1. 程序化退出
- 允许组件在特定条件下手动退出应用（如用户按下 'q' 键）
- 支持传递可选的 Error 对象，用于错误退出场景

### 2. 解耦设计
- 将退出逻辑与具体组件解耦
- 组件无需知道退出的具体实现，只需调用 `exit()` 方法

## 具体技术实现

### 代码实现

```typescript
import { useContext } from 'react'
import AppContext from '../components/AppContext.js'

const useApp = () => useContext(AppContext)
export default useApp
```

### 返回类型

```typescript
{
  exit: (error?: Error) => void
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/AppContext.ts` | 定义 AppContext 及其类型 |

### AppContext 定义

```typescript
// src/ink/components/AppContext.ts
export type Props = {
  readonly exit: (error?: Error) => void
}

const AppContext = createContext<Props>({
  exit() {},
})
```

### 使用示例

```typescript
import { useApp } from 'ink'

const ExitExample = () => {
  const { exit } = useApp()
  
  useInput((input) => {
    if (input === 'q') {
      exit()  // 正常退出
    }
  })
  
  return <Text>Press 'q' to exit</Text>
}
```

## 依赖与外部交互

### Context Provider 设置

`AppContext.Provider` 在 `App.tsx` 中设置，实际的 `exit` 实现由 Ink 主类提供：

```typescript
// 在 ink.tsx 中
const appContextValue = {
  exit: this.exit  // Ink 类的 exit 方法
}
```

### 与 Ink 实例的交互

- `exit` 方法最终调用 `Ink` 类的 `unmount` 逻辑
- 支持同步和异步清理操作
- 可以传递 Error 对象触发错误处理流程

## 风险、边界与改进建议

### 潜在风险

1. **空实现风险**：如果在 Provider 外部使用，将调用空函数，可能导致预期外的行为
2. **多次调用**：没有内置防重入机制，多次调用 `exit` 可能导致重复清理

### 边界情况

1. **Provider 外部使用**：返回默认值 `{ exit() {} }`
2. **清理中调用**：如果在清理过程中调用，行为未定义

### 改进建议

1. **添加警告**：在开发模式下，如果在 Provider 外部使用，打印警告
2. **防重入**：添加状态检查，防止多次调用
3. **返回值扩展**：考虑返回更多应用级状态，如 `isExiting`

```typescript
// 可能的扩展
const useApp = () => {
  const context = useContext(AppContext)
  if (process.env.NODE_ENV === 'development' && !context) {
    console.warn('useApp must be used within AppContext.Provider')
  }
  return context
}
```
