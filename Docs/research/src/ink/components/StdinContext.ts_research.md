# StdinContext.ts 深度研究文档

## 场景与职责

`StdinContext` 是 Ink 终端 UI 框架中负责**标准输入流管理**的 React Context。它解决了终端应用中处理用户输入的核心问题，提供跨平台的 stdin 访问、原始模式控制和事件分发能力。

### 核心职责

1. **输入流暴露**：向组件树提供 `process.stdin` 或自定义输入流的访问
2. **原始模式控制**：封装 `setRawMode` 调用，处理不支持 TTY 的环境降级
3. **事件分发**：通过 `EventEmitter` 将输入事件广播给多个监听者（如 `useInput` hooks）
4. **终端查询**：提供终端能力查询接口（DECRQM、OSC 11 等）
5. **Ctrl+C 处理**：协调应用退出行为

### 典型使用场景

- **键盘输入处理**：`useInput` hook 监听按键事件
- **鼠标事件处理**：全屏模式下的点击、滚轮事件
- **原始模式切换**：文本输入框聚焦时启用原始模式
- **终端能力检测**：查询终端支持的特性（如真彩色、焦点事件）

---

## 功能点目的

### 1. 标准输入流封装

```typescript
readonly stdin: NodeJS.ReadStream
```

- 默认使用 `process.stdin`
- 支持通过 `render()` 的 `options.stdin` 注入自定义流（用于测试、重定向输入）

### 2. 原始模式管理

```typescript
readonly setRawMode: (value: boolean) => void
readonly isRawModeSupported: boolean
```

**原始模式 (Raw Mode)** 的关键特性：
- 禁用行缓冲：按键立即传递给应用，无需按 Enter
- 禁用特殊处理：Ctrl+C、Ctrl+Z 等不再由终端处理
- 启用原始字节流：可以读取每个按键的原始转义序列

**为什么需要封装**：
- 不是所有 stdin 都支持 `setRawMode`（如管道输入、文件重定向）
- 需要跟踪引用计数，避免多个组件同时启用/禁用时的冲突
- Ink 需要协调 React 组件生命周期与原始模式状态

### 3. 内部事件发射器

```typescript
readonly internal_eventEmitter: EventEmitter
```

- 基于 Node.js `EventEmitter` 的增强版本
- 支持 `stopImmediatePropagation()` 语义
- 用于 `useInput` 等 hooks 的事件订阅

### 4. 终端查询器

```typescript
readonly internal_querier: TerminalQuerier | null
```

- 发送终端查询序列（如 `XTVERSION`、`DECRQM`）
- 异步等待终端响应
- 用于检测终端类型和能力

---

## 具体技术实现

### Context 定义

```typescript
const StdinContext = createContext<Props>({
  stdin: process.stdin,
  internal_eventEmitter: new EventEmitter(),
  setRawMode() {},  // 默认空实现（降级）
  isRawModeSupported: false,
  internal_exitOnCtrlC: true,
  internal_querier: null,
})
```

### 默认值设计

| 属性 | 默认值 | 理由 |
|------|--------|------|
| `stdin` | `process.stdin` | 最常见的使用场景 |
| `setRawMode` | 空函数 | 在不支持的环境中静默失败 |
| `isRawModeSupported` | `false` | 保守假设，避免抛出异常 |
| `internal_exitOnCtrlC` | `true` | 安全默认，防止意外挂起 |
| `internal_querier` | `null` | 默认上下文无查询能力 |

### Provider 实现（App.tsx）

```typescript
<StdinContext.Provider value={{
  stdin: this.props.stdin,
  setRawMode: this.handleSetRawMode,
  isRawModeSupported: this.isRawModeSupported(),
  internal_exitOnCtrlC: this.props.exitOnCtrlC,
  internal_eventEmitter: this.internal_eventEmitter,
  internal_querier: this.querier
}}>
```

### setRawMode 实现细节

```typescript
handleSetRawMode = (isEnabled: boolean): void => {
  if (!this.isRawModeSupported()) {
    // 抛出友好错误，引导用户查看文档
    throw new Error('Raw mode is not supported...')
  }
  
  stdin.setEncoding('utf8')
  if (isEnabled) {
    if (this.rawModeEnabledCount === 0) {
      // 首次启用：停止早期输入捕获、添加 readable 监听器
      stopCapturingEarlyInput()
      stdin.ref()
      stdin.setRawMode(true)
      stdin.addListener('readable', this.handleReadable)
      // 启用终端特性：粘贴模式、焦点事件、扩展键
      this.props.stdout.write(EBP)  // 启用括号粘贴模式
      this.props.stdout.write(EFE)  // 启用焦点事件
      // ...
    }
    this.rawModeEnabledCount++
  } else {
    if (--this.rawModeEnabledCount === 0) {
      // 最后一个组件退出：清理所有设置
      stdin.setRawMode(false)
      stdin.removeListener('readable', this.handleReadable)
      stdin.unref()
      // ...
    }
  }
}
```

### 输入处理流程

```
stdin 'readable' 事件
  ↓
handleReadable() 读取数据块
  ↓
processInput() 解析按键序列
  ↓
parseMultipleKeypresses() 状态机解析
  ↓
reconciler.discreteUpdates() 批量处理
  ↓
processKeysInBatch() 分发事件
  ↓
EventEmitter.emit('input', event) 广播
  ↓
useInput hooks 接收并处理
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件 | 职责 |
|------|------|
| `StdinContext.ts` | Context 定义和类型 |
| `App.tsx` | Provider 实现、原始模式管理 |
| `use-stdin.ts` | 消费 Context 的 Hook |
| `use-input.ts` | 基于 stdin 的输入处理 Hook |
| `emitter.ts` | EventEmitter 实现 |
| `parse-keypress.ts` | 按键序列解析状态机 |
| `terminal-querier.ts` | 终端查询实现 |

### 消费路径

```
组件使用输入
  ↓
import useInput from '../hooks/use-input.js'
  ↓
useInput() 调用 useStdin()
  ↓
useStdin() 返回 useContext(StdinContext)
  ↓
订阅 internal_eventEmitter 的 'input' 事件
  ↓
App.tsx handleReadable() 触发事件
```

### 关键类型定义

```typescript
// StdinContext.ts
export type Props = {
  readonly stdin: NodeJS.ReadStream
  readonly setRawMode: (value: boolean) => void
  readonly isRawModeSupported: boolean
  readonly internal_exitOnCtrlC: boolean
  readonly internal_eventEmitter: EventEmitter
  readonly internal_querier: TerminalQuerier | null
}
```

---

## 依赖与外部交互

### 直接依赖

```typescript
import { createContext } from 'react'
import { EventEmitter } from '../events/emitter.js'
import type { TerminalQuerier } from '../terminal-querier.js'
```

### 外部交互

1. **与 `App.tsx` 的交互**：
   - App 组件创建并管理 `EventEmitter` 和 `TerminalQuerier` 实例
   - App 处理原始模式的引用计数
   - App 解析输入并触发事件

2. **与 `use-input.ts` 的交互**：
   ```typescript
   const { stdin, setRawMode, internal_eventEmitter } = useStdin()
   ```
   - 调用 `setRawMode(true)` 启用输入监听
   - 订阅 `internal_eventEmitter` 的 'input' 事件
   - 清理时调用 `setRawMode(false)`

3. **与 `terminal-querier.ts` 的交互**：
   - `TerminalQuerier` 发送查询序列到 stdout
   - 通过 stdin 读取响应
   - 响应通过 `onResponse` 回调处理

4. **与 `bootstrap/state.ts` 的交互**：
   - 输入事件触发 `updateLastInteractionTime()`
   - 用于空闲检测和通知超时

---

## 风险、边界与改进建议

### 已知风险

1. **引用计数泄漏**：
   - 如果组件在启用原始模式后卸载但未调用 `setRawMode(false)`
   - 导致 `rawModeEnabledCount` 不归零，原始模式无法正确关闭
   - **缓解**：`useInput` 等 hooks 在 `useEffect` 清理函数中正确调用

2. **多组件竞争**：
   - 多个组件同时启用原始模式时，第一个退出的组件会关闭原始模式
   - **缓解**：引用计数确保只有最后一个退出时才真正关闭

3. **stdin 不可读**：
   - 在管道或重定向环境中 `isRawModeSupported` 为 `false`
   - 尝试启用原始模式会抛出错误
   - **缓解**：组件应检查 `isRawModeSupported` 并优雅降级

### 边界情况处理

| 场景 | 行为 |
|------|------|
| 非 TTY stdin | `isRawModeSupported: false`，`setRawMode` 抛出错误 |
| 组件重复挂载/卸载 | 引用计数确保状态正确 |
| 快速连续调用 setRawMode | 引用计数递增/递减，无竞态问题 |
| 应用退出时 | `componentWillUnmount` 清理所有监听器和模式 |

### 改进建议

1. **添加输入流健康检查**：
   ```typescript
   readonly isStdinActive: boolean  // 检测 stdin 是否可读
   ```
   用于检测管道断开、SSH 连接丢失等情况。

2. **暴露更多终端状态**：
   ```typescript
   readonly isRawModeEnabled: boolean  // 当前原始模式状态
   readonly enabledComponentCount: number  // 启用原始模式的组件数
   ```
   便于调试和监控。

3. **支持输入流切换**：
   ```typescript
   setStdin: (newStdin: NodeJS.ReadStream) => void
   ```
   支持动态切换输入源（如从文件切换到交互式）。

4. **改进错误处理**：
   - 当前 `setRawMode` 直接抛出错误
   - 可考虑返回结果对象：`{ success: boolean, error?: Error }`
   - 或添加 `trySetRawMode` 变体

5. **文档增强**：
   - 添加 TTY/原始模式的背景知识链接
   - 提供常见环境（Docker、CI、SSH）的处理建议
   - 添加测试示例（如何 mock stdin）

### 架构思考

当前 `StdinContext` 混合了**公共 API**（`stdin`、`setRawMode`）和**内部实现**（`internal_*`）。这种设计：

**优点**：
- 简单直接，无需多个 Context
- 内部实现可以访问所有需要的数据

**缺点**：
- 内部实现暴露在类型定义中
- 外部组件可能误用内部 API

**替代方案**：
- 分离为 `StdinContext`（公共）和 `InternalStdinContext`（内部）
- 但会增加复杂性和 Provider 嵌套深度

当前设计在实用性和简洁性之间取得了良好平衡。
