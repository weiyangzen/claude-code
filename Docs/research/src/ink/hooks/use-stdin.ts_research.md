# use-stdin.ts 深入研究

## 场景与职责

`useStdin` 是 Ink 终端 UI 框架中最基础的 Hook 之一，用于暴露标准输入流（stdin）相关的功能和状态。它是 `StdinContext` 的简单封装，为组件提供访问终端输入的能力。

## 功能点目的

### 1. 标准输入访问
- 暴露 stdin 流对象
- 提供原始模式控制
- 支持原始模式可用性检测

### 2. 事件系统访问
- 暴露内部事件发射器
- 允许组件监听输入事件
- 支持 Ctrl+C 退出配置查询

### 3. 终端查询
- 提供终端查询功能（DECRQM, OSC 11 等）
- 支持异步等待终端响应

## 具体技术实现

### 代码实现

```typescript
import { useContext } from 'react'
import StdinContext from '../components/StdinContext.js'

const useStdin = () => useContext(StdinContext)
export default useStdin
```

### 返回类型（StdinContext.Props）

```typescript
export type Props = {
  readonly stdin: NodeJS.ReadStream                    // stdin 流
  readonly setRawMode: (value: boolean) => void       // 原始模式控制
  readonly isRawModeSupported: boolean                 // 原始模式支持检测
  readonly internal_exitOnCtrlC: boolean               // Ctrl+C 退出配置
  readonly internal_eventEmitter: EventEmitter         // 输入事件发射器
  readonly internal_querier: TerminalQuerier | null    // 终端查询器
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/StdinContext.ts` | 定义 StdinContext 及其类型 |
| `src/ink/events/emitter.ts` | 定义 EventEmitter 类 |

### StdinContext 定义

```typescript
const StdinContext = createContext<Props>({
  stdin: process.stdin,
  internal_eventEmitter: new EventEmitter(),
  setRawMode() {},
  isRawModeSupported: false,
  internal_exitOnCtrlC: true,
  internal_querier: null,
})
```

### 使用示例

```typescript
import { useStdin } from 'ink'

const StdinReader = () => {
  const { stdin, setRawMode, isRawModeSupported } = useStdin()
  
  useEffect(() => {
    if (!isRawModeSupported) return
    
    setRawMode(true)
    
    const handleData = (data: Buffer) => {
      console.log('Received:', data.toString())
    }
    
    stdin.on('data', handleData)
    
    return () => {
      setRawMode(false)
      stdin.off('data', handleData)
    }
  }, [stdin, setRawMode, isRawModeSupported])
  
  return <Text>Reading from stdin...</Text>
}
```

### 内部使用（useInput）

```typescript
// use-input.ts 中的使用
const { setRawMode, internal_exitOnCtrlC, internal_eventEmitter } = useStdin()
```

## 依赖与外部交互

### 与终端的交互

1. **原始模式**：
   - `setRawMode(true)` 启用原始模式
   - 按键立即传递，不回显
   - 控制字符作为普通数据传递

2. **stdin 流**：
   - Node.js `process.stdin` 或自定义流
   - 支持 `data` 事件监听

### 与 EventEmitter 的交互

```typescript
// 自定义 EventEmitter 特性
class EventEmitter extends NodeEventEmitter {
  constructor() {
    super()
    this.setMaxListeners(0)  // 禁用 maxListeners 警告
  }
  
  override emit(type: string, ...args: unknown[]): boolean {
    // 支持 stopImmediatePropagation()
    for (const listener of listeners) {
      listener.apply(this, args)
      if (ccEvent?.didStopImmediatePropagation()) break
    }
  }
}
```

### 与 Ink 主类的交互

- `setRawMode` 实际由 Ink 实例提供
- 包装了底层的 `stdin.setRawMode`，添加了自己的处理逻辑
- 确保正确的清理和状态管理

## 风险、边界与改进建议

### 潜在风险

1. **空实现风险**：
   - 在 Provider 外部使用时返回默认空实现
   - `setRawMode` 是空函数，可能导致预期外的行为

2. **原始模式状态不一致**：
   - 多个组件同时调用 `setRawMode` 可能导致状态冲突
   - 需要协调机制

3. **事件监听器泄漏**：
   - 如果清理不当，可能导致事件监听器累积

### 边界情况

1. **非 TTY 环境**：
   - `isRawModeSupported` 为 false
   - `setRawMode` 调用无效果
   - 需要优雅降级

2. **Provider 外部使用**：
   - 返回默认值
   - `stdin` 为 `process.stdin`
   - `isRawModeSupported` 为 false

3. **自定义 stdin**：
   - 通过 `render()` 的 `options.stdin` 传入
   - 可能不支持原始模式

### 改进建议

1. **添加使用警告**：
   ```typescript
   const useStdin = () => {
     const context = useContext(StdinContext)
     if (process.env.NODE_ENV === 'development') {
       const isDefault = context.stdin === process.stdin && 
                         !context.isRawModeSupported
       if (isDefault) {
         console.warn('useStdin: Using default context. ' +
                      'Make sure to use within Ink app.')
       }
     }
     return context
   }
   ```

2. **添加原始模式引用计数**：
   ```typescript
   // 内部实现
   let rawModeRefCount = 0
   
   setRawMode(value: boolean) {
     if (value) {
       rawModeRefCount++
       if (rawModeRefCount === 1) {
         actualSetRawMode(true)
       }
     } else {
       rawModeRefCount--
       if (rawModeRefCount === 0) {
         actualSetRawMode(false)
       }
     }
   }
   ```

3. **暴露更多终端信息**：
   ```typescript
   export type Props = {
     // ... 现有属性
     readonly isTTY: boolean
     readonly terminalType: string | undefined
     readonly supportsColor: boolean
   }
   ```

4. **添加高阶组件**：
   ```typescript
   export function withStdin<P extends object>(
     Component: React.ComponentType<P & StdinProps>
   ) {
     return function WrappedComponent(props: P) {
       const stdinProps = useStdin()
       return <Component {...props} {...stdinProps} />
     }
   }
   ```

### 测试建议

1. 测试原始模式启用/禁用
2. 测试非 TTY 环境的降级行为
3. 测试 Provider 外部使用的默认行为
4. 测试自定义 stdin 流
5. 测试事件监听器的正确清理
