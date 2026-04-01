# TerminalFocusContext.tsx 深度研究文档

## 场景与职责

`TerminalFocusContext` 是 Ink 终端 UI 框架中用于**终端焦点状态管理**的 React Context。它通过 DECSET 1004 终端焦点事件协议，让应用感知终端窗口是否获得焦点，从而实现智能的 UI 行为优化。

### 核心职责

1. **焦点状态追踪**：监听终端的聚焦/失焦事件
2. **跨组件共享**：通过 Context 将焦点状态分发给整个组件树
3. **性能优化**：避免不必要的重渲染（`TerminalFocusProvider` 作为独立组件）
4. **非 React 访问**：配合 `terminal-focus-state.ts` 提供同步状态读取

### 典型使用场景

- **时钟/动画节流**：终端失焦时降低刷新频率（`ClockContext`）
- **输入处理优化**：失焦时暂停某些输入处理逻辑
- **UI 状态提示**：显示"终端已失焦"的视觉指示
- **后台任务控制**：失焦时暂停非关键后台操作

---

## 功能点目的

### 1. 焦点状态类型

```typescript
export type TerminalFocusState = 'focused' | 'blurred' | 'unknown'
```

| 状态 | 含义 | 使用场景 |
|------|------|----------|
| `'focused'` | 终端已获得焦点 | 正常交互状态 |
| `'blurred'` | 终端已失去焦点 | 用户切换到其他窗口 |
| `'unknown'` | 终端不支持焦点报告 | 默认/降级状态 |

**设计关键**：`'unknown'` 被视为与 `'focused'` 同等对待，确保在不支持焦点报告的终端上应用行为一致。

### 2. Context 结构

```typescript
export type TerminalFocusContextProps = {
  readonly isTerminalFocused: boolean    // 简化布尔值（true 表示 focused/unknown）
  readonly terminalFocusState: TerminalFocusState  // 完整状态
}
```

提供两种粒度：
- `isTerminalFocused`：适合简单的条件判断
- `terminalFocusState`：需要区分 `unknown` 时使用

### 3. Provider 分离模式

```typescript
// 独立组件，避免 App.tsx 因焦点变化重渲染
export function TerminalFocusProvider({ children }) {
  const isTerminalFocused = useSyncExternalStore(subscribeTerminalFocus, getTerminalFocused)
  const terminalFocusState = useSyncExternalStore(subscribeTerminalFocus, getTerminalFocusState)
  // ...
}
```

**为什么需要分离**：
- `App.tsx` 是类组件，状态变化会导致整个应用重渲染
- `TerminalFocusProvider` 作为函数组件，使用 `useSyncExternalStore` 订阅外部状态
- `children` 是稳定的 prop 引用，不会触发子树重渲染
- 只有消费 `TerminalFocusContext` 的组件才会更新

---

## 具体技术实现

### 状态管理架构

采用**外部存储 + React 订阅**模式：

```
terminal-focus-state.ts (外部存储)
├── focusState: TerminalFocusState     // 单一数据源
├── subscribers: Set<() => void>       // 订阅者列表
├── setTerminalFocused(v)              // 状态更新
├── getTerminalFocused()               // 同步读取
└── subscribeTerminalFocus(cb)         // 订阅接口

TerminalFocusContext.tsx (React 集成)
└── TerminalFocusProvider
    └── useSyncExternalStore(subscribe, getSnapshot)
        // 订阅外部存储，同步到 React
```

### useSyncExternalStore 使用

```typescript
const isTerminalFocused = useSyncExternalStore(
  subscribeTerminalFocus,  // 订阅函数
  getTerminalFocused       // 获取快照
)
```

**为什么使用 `useSyncExternalStore`**：
- 专门用于订阅外部存储（非 React 状态）
- 支持 Suspense 和并发特性
- 避免 tearing（状态不一致）问题
- 自动处理订阅和清理

### 焦点事件流程

```
用户切换窗口
  ↓
终端发送 FOCUS_IN / FOCUS_OUT 序列
  ↓
App.tsx handleReadable() 解析序列
  ↓
handleTerminalFocus(isFocused)
  ↓
setTerminalFocused(v)  // terminal-focus-state.ts
  ↓
更新 focusState + 通知所有订阅者
  ↓
TerminalFocusProvider 的 useSyncExternalStore 触发更新
  ↓
消费 Context 的组件重渲染
```

### 焦点事件处理代码

```typescript
// App.tsx
handleTerminalFocus = (isFocused: boolean): void => {
  // 更新外部状态，通知订阅者
  setTerminalFocused(isFocused)
}

// 在 processKeysInBatch 中
if (sequence === FOCUS_IN) {
  app.handleTerminalFocus(true)
  const event = new TerminalFocusEvent('terminalfocus')
  app.internal_eventEmitter.emit('terminalfocus', event)
}
if (sequence === FOCUS_OUT) {
  app.handleTerminalFocus(false)
  // 失焦时结束选择（拖动选择的安全措施）
  if (app.props.selection.isDragging) {
    finishSelection(app.props.selection)
    app.props.onSelectionChange()
  }
  const event = new TerminalFocusEvent('terminalblur')
  app.internal_eventEmitter.emit('terminalblur', event)
}
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件 | 职责 |
|------|------|
| `TerminalFocusContext.tsx` | Context 定义、Provider 实现 |
| `terminal-focus-state.ts` | 外部状态存储、订阅管理 |
| `App.tsx` | 焦点事件解析、状态更新触发 |
| `use-terminal-focus.ts` | 消费 Hook 封装 |
| `ClockContext.tsx` | 焦点状态消费者示例 |

### 焦点相关 CSI 序列

```typescript
// termio/csi.ts
export const FOCUS_IN = '\x1b[I'   // 终端获得焦点
export const FOCUS_OUT = '\x1b[O'  // 终端失去焦点

// 启用焦点报告：DECSET 1004
export const EFE = '\x1b[?1004h'   // Enable Focus Event
export const DFE = '\x1b[?1004l'   // Disable Focus Event
```

### 调用链

```
终端发送 \x1b[O (失焦)
  ↓
parse-keypress.ts 解析为 FOCUS_OUT
  ↓
App.tsx processKeysInBatch()
  ↓
handleTerminalFocus(false)
  ↓
setTerminalFocused(false)  [terminal-focus-state.ts]
  ↓
for (const cb of subscribers) cb()
  ↓
useSyncExternalStore 触发重新渲染
  ↓
组件获取新的 isTerminalFocused = false
```

---

## 依赖与外部交互

### 直接依赖

```typescript
import React, { createContext, useSyncExternalStore } from 'react'
import {
  getTerminalFocused,
  getTerminalFocusState,
  subscribeTerminalFocus,
  type TerminalFocusState,
} from '../terminal-focus-state.js'
```

### 外部交互

1. **与 `terminal-focus-state.ts` 的交互**：
   - 订阅外部状态变化
   - 通过 getter 函数读取当前状态
   - 不直接修改状态（由 App.tsx 修改）

2. **与 `App.tsx` 的交互**：
   - App 启用原始模式时发送 `EFE`（启用焦点报告）
   - App 解析焦点事件序列并调用 `setTerminalFocused()`
   - App 在 `componentWillUnmount` 时发送 `DFE`（禁用焦点报告）

3. **与 `ClockContext.tsx` 的交互**（消费者示例）：
   ```typescript
   // 时钟根据焦点状态调整刷新频率
   const isTerminalFocused = useTerminalFocus()
   const interval = isTerminalFocused ? 1000 : 5000  // 失焦时降低频率
   ```

4. **与 `use-terminal-focus.ts` 的交互**：
   ```typescript
   export function useTerminalFocus(): boolean {
     const { isTerminalFocused } = useContext(TerminalFocusContext)
     return isTerminalFocused
   }
   ```

---

## 风险、边界与改进建议

### 已知风险

1. **终端不支持焦点报告**：
   - 部分旧终端或简单终端模拟器不支持 DECSET 1004
   - 状态永远保持 `'unknown'`
   - **缓解**：将 `unknown` 视为 `focused`，应用行为不受影响

2. **焦点事件丢失**：
   - SSH 连接中断、tmux detach 等场景可能丢失焦点事件
   - 状态可能与实际焦点不一致
   - **缓解**：`App.tsx` 在 stdin 恢复时重新启用焦点报告

3. **多终端环境**：
   - 一个应用连接到多个终端时焦点状态不明确
   - 当前设计假设单终端

### 边界情况处理

| 场景 | 行为 |
|------|------|
| 终端不支持 1004 | 状态保持 `'unknown'`，应用正常运作 |
| 焦点事件丢失 | 状态不更新，直到下一个事件 |
| 快速切换焦点 | 事件按顺序处理，最终状态正确 |
| 失焦时拖动选择 | `FOCUS_OUT` 触发 `finishSelection()`，防止选择悬挂 |

### 改进建议

1. **添加焦点状态强制刷新**：
   ```typescript
   // 在 terminal-focus-state.ts
   export function queryTerminalFocus(): Promise<boolean> {
     // 发送查询序列，等待响应
   }
   ```
   用于恢复连接后同步状态。

2. **暴露焦点变化时间戳**：
   ```typescript
   export type TerminalFocusContextProps = {
     // ...
     readonly lastFocusChangeTime: number
   }
   ```
   用于实现"失焦超过 N 秒后暂停"的逻辑。

3. **支持焦点变化动画**：
   ```typescript
   readonly focusTransitionState: 'stable' | 'transitioning'
   ```
   用于实现平滑的 UI 过渡效果。

4. **改进测试支持**：
   ```typescript
   // 测试工具函数
   export function mockTerminalFocus(state: TerminalFocusState) {
     setTerminalFocused(state === 'focused')
   }
   ```

5. **文档增强**：
   - 列出支持焦点报告的终端列表
   - 提供检测终端支持性的方法
   - 添加 tmux/screen 配置建议（需要 `focus-events on`）

### 架构亮点

当前实现的一个关键设计是**外部状态 + useSyncExternalStore 模式**：

```
┌─────────────────────────────────────────────────────────┐
│  外部状态 (terminal-focus-state.ts)                      │
│  • 独立于 React 生命周期                                  │
│  • 支持非 React 代码同步读取                              │
│  • 单一数据源，避免不一致                                 │
└─────────────────────────────────────────────────────────┘
                           ↑
                           │ subscribe / getSnapshot
                           ↓
┌─────────────────────────────────────────────────────────┐
│  React 集成 (TerminalFocusContext.tsx)                   │
│  • useSyncExternalStore 桥接                            │
│  • Provider 分离避免不必要重渲染                          │
│  • Context 提供便捷访问                                   │
└─────────────────────────────────────────────────────────┘
```

这种模式：
- **解耦**：状态管理独立于 UI 框架
- **性能**：只有消费者重渲染
- **灵活性**：非 React 代码也能访问状态
- **兼容性**：支持 React 18 并发特性

是处理跨 React 和非 React 边界的全局状态的推荐模式。
