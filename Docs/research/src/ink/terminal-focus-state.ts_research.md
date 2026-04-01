# terminal-focus-state.ts 研究文档

## 场景与职责

`terminal-focus-state.ts` 管理终端焦点状态的信号系统，提供非 React 的同步访问接口。当终端支持 DECSET 1004 焦点事件时，模块接收并分发焦点变化通知。

### 核心职责
1. **状态管理**: 维护当前终端焦点状态（focused/blurred/unknown）
2. **同步访问**: 提供非 React 组件的同步状态读取
3. **订阅机制**: 支持 `useSyncExternalStore` 的订阅模式
4. **Promise 等待**: 支持等待失去焦点事件的 Promise API

## 功能点目的

### 1. 三态焦点状态
```typescript
export type TerminalFocusState = 'focused' | 'blurred' | 'unknown'
```

- **`unknown`**: 默认状态，终端不支持焦点报告或尚未收到事件
- **`focused`**: 终端获得焦点
- **`blurred`**: 终端失去焦点

### 2. 同步状态读取
```typescript
export function getTerminalFocused(): boolean
export function getTerminalFocusState(): TerminalFocusState
```

供非 React 代码（如输入处理、节流逻辑）同步检查焦点状态。

### 3. 订阅通知
```typescript
export function subscribeTerminalFocus(cb: () => void): () => void
```

符合 React `useSyncExternalStore` 规范的订阅接口。

### 4. Promise 等待
```typescript
// 内部使用
const resolvers: Set<() => void> = new Set()
```

当失去焦点时，解析所有等待的 Promise，用于同步等待焦点变化。

## 具体技术实现

### 状态更新
```typescript
export function setTerminalFocused(v: boolean): void {
  focusState = v ? 'focused' : 'blurred'
  // 通知所有订阅者
  for (const cb of subscribers) {
    cb()
  }
  if (!v) {
    // 失去焦点时解析所有等待的 Promise
    for (const resolve of resolvers) {
      resolve()
    }
    resolvers.clear()
  }
}
```

### 订阅管理
```typescript
export function subscribeTerminalFocus(cb: () => void): () => void {
  subscribers.add(cb)
  return () => {
    subscribers.delete(cb)
  }
}
```

返回的清理函数用于组件卸载时取消订阅。

### 状态重置
```typescript
export function resetTerminalFocusState(): void
```

在终端重置或重新连接时恢复为 `unknown` 状态。

## 关键代码路径与文件引用

### 调用方
- **`App.tsx`**: 解析 DECSET 1004 焦点事件（CSI I/O）后调用 `setTerminalFocused()`
- **`useTerminalFocus.ts`**: React Hook，使用 `useSyncExternalStore` 订阅状态
- **输入处理**: 使用 `getTerminalFocused()` 检查是否需要节流

### 使用模式
```typescript
// 同步检查
if (getTerminalFocused()) {
  // 处理输入
}

// React 组件
const isFocused = useTerminalFocus()

// 等待失去焦点
await waitForBlur()
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| 无外部依赖 | 纯内存状态管理 |

### 外部交互
- **DECSET 1004**: 终端焦点事件协议
  - `CSI I` - 焦点进入（focused）
  - `CSI O` - 焦点离开（blurred）

## 风险、边界与改进建议

### 已知风险
1. **状态同步**: 如果焦点事件丢失，状态可能永久停留在错误值
2. **多终端**: 不支持多终端实例的独立焦点状态
3. **测试难度**: 全局状态使得单元测试需要仔细隔离

### 边界情况
1. **不支持焦点事件的终端**: 永久保持 `unknown`，消费者应将其视为 `focused`
2. **快速切换**: 高频焦点变化可能触发大量订阅回调
3. **SSR**: 服务端渲染时无终端概念，保持 `unknown`

### 改进建议
1. **实例化**: 支持多 Ink 实例的独立焦点状态
2. **心跳检测**: 定期发送查询验证焦点状态是否同步
3. **防抖处理**: 对高频焦点变化进行防抖
4. **持久化**: 考虑在会话恢复时恢复焦点状态

### 相关标准
- DECSET 1004 - Focus Event Mode
- `CSI I` / `CSI O` 焦点事件序列
