# terminal-focus-event.ts 深度研究文档

## 场景与职责

`TerminalFocusEvent` 用于处理终端窗口级别的焦点变化事件，与组件级的 `FocusEvent` 形成层次关系：

| 事件类型 | 层级 | 触发条件 |
|---------|------|---------|
| `TerminalFocusEvent` | 终端窗口 | 终端应用获得/失去系统焦点 |
| `FocusEvent` | 组件 | 焦点在 Ink 组件间移动 |

使用场景：
1. **空闲检测**：终端失去焦点时暂停某些活动（如自动保存）
2. **视觉反馈**：终端获得焦点时改变提示符样式
3. **通知管理**：失去焦点时允许发送系统通知

## 功能点目的

### 1. 终端焦点状态追踪
- `'terminalfocus'`：终端窗口获得系统焦点
- `'terminalblur'`：终端窗口失去系统焦点

### 2. DECSET 1004 支持
使用终端的聚焦报告模式（Focus Reporting）：
- 终端获得焦点时发送 `CSI I` (`\x1b[I`)
- 终端失去焦点时发送 `CSI O` (`\x1b[O`)

### 3. 简单事件模型
继承自 `Event`（而非 `TerminalEvent`），因为：
- 不需要冒泡（应用级事件）
- 不需要捕获/目标阶段
- 通过 EventEmitter 广播

## 具体技术实现

### 类型定义

```typescript
export type TerminalFocusEventType = 'terminalfocus' | 'terminalblur'

export class TerminalFocusEvent extends Event {
  readonly type: TerminalFocusEventType

  constructor(type: TerminalFocusEventType) {
    super()
    this.type = type
  }
}
```

### 终端协议

DECSET 1004（聚焦报告）序列：

```
启用:  CSI ? 1004 h
禁用:  CSI ? 1004 l

事件:
  聚焦: CSI I  (\x1b[I)
  失焦: CSI O  (\x1b[O)
```

### 使用模式

1. **App.tsx 中的处理**（lines 385-391, 472-491）：
   ```typescript
   handleTerminalFocus = (isFocused: boolean): void => {
     setTerminalFocused(isFocused)
   }
   
   // 在 processKeysInBatch 中
   if (sequence === FOCUS_IN) {
     app.handleTerminalFocus(true)
     const event = new TerminalFocusEvent('terminalfocus')
     app.internal_eventEmitter.emit('terminalfocus', event)
     continue
   }
   if (sequence === FOCUS_OUT) {
     app.handleTerminalFocus(false)
     // ... 失焦恢复逻辑
     const event = new TerminalFocusEvent('terminalblur')
     app.internal_eventEmitter.emit('terminalblur', event)
     continue
   }
   ```

2. **use-terminal-focus.ts**：
   ```typescript
   export function useTerminalFocus(): boolean {
     const [focused, setFocused] = useState(getTerminalFocused())
     useEffect(() => {
       const unsubscribe = subscribeToTerminalFocus(setFocused)
       return unsubscribe
     }, [])
     return focused
   }
   ```

3. **Clock 组件**：
   - 终端失焦时降低更新频率（节省 CPU）
   - 终端聚焦时恢复正常更新

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/terminal-focus-event.ts` - TerminalFocusEvent 类定义

### 使用位置

1. **App.tsx**（lines 10, 385-391, 472-491）：
   ```typescript
   import { TerminalFocusEvent } from '../events/terminal-focus-event.js'
   ```
   创建并分发终端焦点事件。

2. **termio/csi.ts**（未在批次中）：
   ```typescript
   export const FOCUS_IN = '\x1b[I'
   export const FOCUS_OUT = '\x1b[O'
   export const EFE = '\x1b[?1004h'  // 启用聚焦报告
   export const DFE = '\x1b[?1004l'  // 禁用聚焦报告
   ```

3. **terminal-focus-state.ts**（未在批次中）：
   全局状态管理，存储当前终端焦点状态。

4. **use-terminal-focus.ts**（未在批次中）：
   React hook，订阅终端焦点变化。

5. **ink.ts**（lines 69-70）：
   ```typescript
   export type { TerminalFocusEventType } from './ink/events/terminal-focus-event.js'
   export { TerminalFocusEvent } from './ink/events/terminal-focus-event.js'
   ```

### 调用链
```
终端发送 CSI I / CSI O
  → App.handleReadable (App.tsx:332)
    → processInput (App.tsx:309)
      → parseMultipleKeypresses (parse-keypress.ts:213)
        → 识别为序列 token
  → processKeysInBatch (App.tsx:444)
    → sequence === FOCUS_IN / FOCUS_OUT
    → handleTerminalFocus (App.tsx:385)
    → new TerminalFocusEvent(...) (terminal-focus-event.ts:15)
    → internal_eventEmitter.emit(...) (App.tsx:475/490)
      → useTerminalFocus 订阅者
      → Clock 组件
```

## 依赖与外部交互

### 依赖

- `event.ts` - `Event` 基类

### 被依赖

- `App.tsx` - 创建和分发事件
- `ink.ts` - 导出供外部使用
- `terminal-focus-state.ts` - 状态管理（间接）

### 与组件级 FocusEvent 的对比

| 特性 | TerminalFocusEvent | FocusEvent |
|-----|-------------------|------------|
| 继承 | Event | TerminalEvent |
| 层级 | 终端窗口 | 组件 |
| 分发方式 | EventEmitter 广播 | Dispatcher 分发 |
| 冒泡 | 不支持 | 支持 |
| 捕获 | 不支持 | 支持 |
| 触发条件 | DECSET 1004 序列 | 组件焦点变化 |
| 使用场景 | 应用级状态管理 | 组件交互 |

### 终端支持

不是所有终端都支持 DECSET 1004：

| 终端 | 支持 |
|-----|------|
| iTerm2 | ✓ |
| Terminal.app | ✓ |
| GNOME Terminal | ✓ |
| Windows Terminal | ✓ |
| VS Code 集成终端 | ✓ |
| tmux | 需要 `focus-events on` |
| screen | ✗ |
| 纯 Linux 控制台 | ✗ |

## 风险、边界与改进建议

### 风险点

1. **终端不支持**:
   - 在不支持 DECSET 1004 的终端中，事件永远不会触发
   - 应用需要有降级策略（假设始终聚焦）

2. **tmux 配置依赖**:
   - tmux 默认不转发聚焦事件
   - 需要用户配置 `focus-events on`

3. **SSH 连接**:
   - 通过 SSH 连接时，聚焦事件可能延迟或丢失
   - 网络中断后可能状态不一致

4. **多窗口/标签**:
   - 同一终端应用的多个窗口独立报告焦点
   - 需要确保处理的是正确的窗口

### 边界情况

1. **启动时状态**:
   - 应用启动时不知道终端是否聚焦
   - 默认假设为聚焦，但可能不正确

2. **快速切换**:
   - 快速 Alt-Tab 可能产生多个事件
   - 需要防抖或状态去重

3. **失焦恢复**:
   ```typescript
   // App.tsx:485-488
   if (app.props.selection.isDragging) {
     finishSelection(app.props.selection)
     app.props.onSelectionChange()
   }
   ```
   失焦时强制结束拖拽选择，防止状态不一致。

4. **与 SIGCONT 的交互**:
   - 进程挂起/恢复时，终端焦点状态可能变化
   - 需要正确处理恢复后的状态同步

### 改进建议

1. **添加聚焦状态查询**:
   ```typescript
   // 主动查询终端焦点状态
   static async queryFocusState(): Promise<boolean>
   ```

2. **添加聚焦原因**:
   ```typescript
   type FocusReason = 'click' | 'tab' | 'shortcut' | 'unknown'
   readonly reason?: FocusReason
   ```

3. **防抖处理**:
   ```typescript
   // 快速切换时合并事件
   private static lastFocusTime: number
   static shouldThrottle(): boolean
   ```

4. **兼容性检测**:
   ```typescript
   // 检测终端是否支持聚焦报告
   static isSupported(): boolean
   ```

5. **状态同步**:
   ```typescript
   // 定期同步焦点状态，防止丢失事件
   static startSync(interval: number): void
   ```

6. **与 Page Visibility API 对比**:
   ```typescript
   // 浏览器环境使用 Page Visibility API
   // 终端环境使用 DECSET 1004
   // 提供统一的抽象
   ```

### 测试建议

1. 测试各种终端的 DECSET 1004 支持
2. 测试 tmux 配置的影响
3. 测试快速 Alt-Tab 的事件序列
4. 测试 SSH 连接下的行为
5. 测试进程挂起/恢复后的状态
6. 测试多窗口场景
