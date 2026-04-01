# use-input.ts 深入研究

## 场景与职责

`useInput` 是 Ink 终端 UI 框架中处理用户键盘输入的核心 Hook。它提供了一个比直接使用 `StdinContext` 更便捷的替代方案，封装了原始模式设置、输入解析和事件分发等复杂逻辑。

## 功能点目的

### 1. 原始模式管理
- 自动启用/禁用终端原始模式（raw mode）
- 确保在组件卸载时正确恢复终端状态
- 支持条件性启用（通过 `isActive` 选项）

### 2. 输入事件处理
- 解析原始输入为结构化的 `InputEvent`
- 支持特殊键（方向键、功能键、Ctrl 组合键等）
- 处理粘贴文本（多字符一次性传入）

### 3. Ctrl+C 处理
- 与应用的 `exitOnCtrlC` 配置协作
- 允许输入处理器拦截 Ctrl+C（当 `exitOnCtrlC` 为 false 时）

## 具体技术实现

### 接口定义

```typescript
type Handler = (input: string, key: Key, event: InputEvent) => void

type Options = {
  isActive?: boolean  // 是否启用输入捕获，默认 true
}

const useInput = (inputHandler: Handler, options: Options = {}) => { ... }
```

### Key 类型定义

```typescript
type Key = {
  upArrow: boolean
  downArrow: boolean
  leftArrow: boolean
  rightArrow: boolean
  pageDown: boolean
  pageUp: boolean
  wheelUp: boolean
  wheelDown: boolean
  home: boolean
  end: boolean
  return: boolean
  escape: boolean
  ctrl: boolean
  shift: boolean
  fn: boolean
  tab: boolean
  backspace: boolean
  delete: boolean
  meta: boolean
  super: boolean
}
```

### 核心实现逻辑

1. **原始模式设置（useLayoutEffect）**：
   ```typescript
   useLayoutEffect(() => {
     if (options.isActive === false) return
     setRawMode(true)
     return () => { setRawMode(false) }
   }, [options.isActive, setRawMode])
   ```
   使用 `useLayoutEffect` 而非 `useEffect`，确保在 React 的 commit 阶段同步启用原始模式，避免在事件循环的下一个 tick 才生效。

2. **事件监听注册**：
   ```typescript
   const handleData = useEventCallback((event: InputEvent) => {
     if (options.isActive === false) return
     const { input, key } = event
     
     // 如果应用不应在 Ctrl+C 时退出，则让输入监听器处理
     if (!(input === 'c' && key.ctrl) || !internal_exitOnCtrlC) {
       inputHandler(input, key, event)
     }
   })
   ```

3. **监听器管理（useEffect）**：
   ```typescript
   useEffect(() => {
     internal_eventEmitter?.on('input', handleData)
     return () => {
       internal_eventEmitter?.removeListener('input', handleData)
     }
   }, [internal_eventEmitter, handleData])
   ```

### useEventCallback 的作用

关键设计决策：
- 监听器只在挂载时注册一次，保持 EventEmitter 监听器数组中的位置稳定
- 如果 `isActive` 在依赖中，监听器会在 false→true 时重新追加，导致顺序变化
- `useEventCallback` 保持引用稳定，同时通过 closure 读取最新的 `isActive`/`inputHandler`

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/StdinContext.ts` | 提供 stdin、setRawMode、eventEmitter 等 |
| `src/ink/hooks/use-stdin.ts` | 封装 StdinContext 的访问 |
| `src/ink/events/input-event.ts` | 定义 InputEvent 和 Key 类型 |
| `src/ink/events/emitter.ts` | 自定义 EventEmitter，支持 stopImmediatePropagation |

### InputEvent 处理流程

```
stdin 数据
    ↓
parseKeypress (底层解析)
    ↓
InputEvent 创建
    ↓
EventEmitter.emit('input', event)
    ↓
useInput handleData
    ↓
用户提供的 inputHandler
```

### InputEvent 类定义

```typescript
// src/ink/events/input-event.ts
export class InputEvent extends Event {
  readonly keypress: ParsedKey
  readonly key: Key
  readonly input: string

  constructor(keypress: ParsedKey) {
    super()
    const [key, input] = parseKey(keypress)
    this.keypress = keypress
    this.key = key
    this.input = input
  }
}
```

### parseKey 关键逻辑

1. **特殊序列处理**：
   - CSI u 序列（Kitty 键盘协议）
   - xterm modifyOtherKeys 序列
   - 应用小键盘模式序列

2. **输入清理**：
   - 移除 ESC 前缀
   - 处理非字母数字键（清空 input）
   - 大写字母自动设置 shift=true

## 依赖与外部交互

### 与 StdinContext 的交互

```typescript
const { 
  setRawMode, 
  internal_exitOnCtrlC, 
  internal_eventEmitter 
} = useStdin()
```

- `setRawMode`: 控制终端原始模式
- `internal_exitOnCtrlC`: 应用级别的 Ctrl+C 处理配置
- `internal_eventEmitter`: 输入事件分发器

### 与 EventEmitter 的交互

自定义 EventEmitter 特性：
- 继承 Node.js EventEmitter
- 支持 `stopImmediatePropagation()`
- 禁用默认的 maxListeners 警告（适应 React 多组件监听场景）

### 原始模式的影响

启用原始模式后：
- 按键不自动回显到终端
- 输入按字符而非按行缓冲
- Ctrl+C 等控制字符作为普通输入传递

## 风险、边界与改进建议

### 潜在风险

1. **终端状态泄漏**：
   - 如果组件卸载时清理函数未执行，终端可能保持原始模式
   - 可能导致终端处于异常状态

2. **事件顺序依赖**：
   - 多个 useInput 同时活跃时的处理顺序依赖注册顺序
   - `stopImmediatePropagation()` 的使用需要谨慎

3. **性能问题**：
   - 每个字符都触发 React 更新
   - 快速输入时可能产生性能压力

### 边界情况

1. **粘贴文本**：
   - 粘贴多字符文本时，回调只调用一次
   - 整个字符串作为 `input` 参数传递

2. **特殊终端**：
   - 某些终端可能不支持原始模式
   - `isRawModeSupported` 可用于优雅降级

3. **组合键处理**：
   - 不同终端对组合键的编码可能不同
   - 解析逻辑需要持续维护

### 改进建议

1. **添加输入缓冲**：
   ```typescript
   // 可选的防抖/节流模式
   useInput(handler, { 
     isActive: true,
     throttleMs: 16  // 限制处理频率
   })
   ```

2. **支持按键组合**：
   ```typescript
   // 声明式按键绑定
   useInput({
     'ctrl+c': handleCopy,
     'ctrl+v': handlePaste,
     'arrowUp': handleUp
   })
   ```

3. **改进错误处理**：
   - 添加原始模式设置失败的错误报告
   - 提供恢复机制

4. **性能优化**：
   - 对于高频输入场景，考虑使用 requestAnimationFrame 批处理
   - 添加 `passive` 选项用于只读监听器

### 测试建议

1. 测试原始模式正确启用/禁用
2. 测试多个 useInput 同时活跃的场景
3. 测试粘贴大段文本
4. 测试特殊键（功能键、组合键）
5. 测试终端失焦/重焦时的行为
6. 测试组件快速挂载/卸载
