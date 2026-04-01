# TerminalSizeContext.tsx 深度研究文档

## 场景与职责

`TerminalSizeContext` 是 Ink 终端 UI 框架中用于**终端尺寸管理**的 React Context。它提供终端窗口的行列数信息，使组件能够响应式地适应终端大小变化。

### 核心职责

1. **尺寸数据共享**：向组件树提供终端的 `columns`（列数）和 `rows`（行数）
2. **响应式布局**：支持组件根据终端尺寸调整布局
3. **视口计算**：配合 `useTerminalViewport` hook 计算元素可见性

### 典型使用场景

- **全屏布局**：根据终端高度计算内容区域大小
- **视口检测**：`useTerminalViewport` 判断元素是否在可视区域内
- **响应式断点**：根据终端宽度调整 UI（如隐藏侧边栏）
- **动画控制**：终端尺寸变化时暂停/恢复动画

---

## 功能点目的

### 1. 尺寸数据类型

```typescript
export type TerminalSize = {
  columns: number  // 终端宽度（字符列数）
  rows: number     // 终端高度（字符行数）
}
```

**为什么是字符而非像素**：
- 终端是字符网格环境，所有布局基于字符单元
- 与 CSS 的 `ch`/`em` 单位类似，但固定为等宽字体尺寸
- 与 `process.stdout.columns`/`rows` 直接对应

### 2. Context 默认值

```typescript
export const TerminalSizeContext = createContext<TerminalSize | null>(null)
```

- 默认值为 `null`，表示尚未获得尺寸信息
- 消费者需要处理 `null` 情况（提供降级行为）

### 3. Provider 实现

```typescript
// App.tsx render()
<TerminalSizeContext.Provider value={{
  columns: this.props.terminalColumns,
  rows: this.props.terminalRows
}}>
```

- 值从 `App` 组件的 props 传入
- 由 `ink.tsx` 在终端大小变化时更新

---

## 具体技术实现

### 极简设计

`TerminalSizeContext.tsx` 是 Ink 组件库中最简单的文件之一：

```typescript
import { createContext } from 'react';

export type TerminalSize = {
  columns: number;
  rows: number;
};

export const TerminalSizeContext = createContext<TerminalSize | null>(null);
```

**设计哲学**：
- **单一职责**：只做一件事——提供尺寸数据
- **最小 API**：类型 + Context，无额外逻辑
- **显式 null**：明确区分"已知尺寸"和"未知尺寸"

### 尺寸更新机制

尺寸变化通过 React 的 props 流传递：

```
终端 SIGWINCH 信号（窗口大小变化）
  ↓
process.stdout.on('resize', handler)
  ↓
ink.tsx 更新内部状态
  ↓
App 组件接收新的 terminalColumns/terminalRows props
  ↓
React 重新渲染
  ↓
TerminalSizeContext.Provider value 更新
  ↓
消费组件获取新尺寸
```

### 消费模式

#### 基础用法

```typescript
import { useContext } from 'react'
import { TerminalSizeContext } from '../components/TerminalSizeContext.js'

function MyComponent() {
  const terminalSize = useContext(TerminalSizeContext)
  
  if (!terminalSize) {
    return null  // 或加载状态
  }
  
  return <Box width={terminalSize.columns / 2}>...</Box>
}
```

#### useTerminalViewport Hook

```typescript
// use-terminal-viewport.ts
import { TerminalSizeContext } from '../components/TerminalSizeContext.js'

export function useTerminalViewport(): [ref, entry] {
  const terminalSize = useContext(TerminalSizeContext)
  // ...
  // 使用 terminalSize.rows 计算视口边界
}
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件 | 职责 |
|------|------|
| `TerminalSizeContext.tsx` | Context 定义和类型 |
| `App.tsx` | Provider 渲染，从 props 获取尺寸 |
| `ink.tsx` | 监听终端 resize 事件，更新 App props |
| `use-terminal-viewport.ts` | 主要消费者，计算元素可见性 |
| `useTerminalSize.ts` | 便捷 Hook 封装 |
| `AlternateScreen.tsx` | 消费者，全屏模式尺寸处理 |

### 尺寸来源

```typescript
// ink.tsx 中获取终端尺寸
const getTerminalSize = () => ({
  columns: stdout.columns ?? 80,
  rows: stdout.rows ?? 24
})
```

### 调用链

```
用户调整终端窗口
  ↓
操作系统发送 SIGWINCH
  ↓
Node.js process.stdout 更新 columns/rows
  ↓
stdout 'resize' 事件触发
  ↓
ink.tsx 事件处理器
  ↓
setState({ terminalColumns, terminalRows })
  ↓
App 组件重新渲染
  ↓
TerminalSizeContext.Provider 新 value
  ↓
useTerminalViewport 重新计算可见性
```

---

## 依赖与外部交互

### 直接依赖

```typescript
import { createContext } from 'react'
```

**零运行时依赖**：除了 React 核心，不依赖任何其他模块。

### 外部交互

1. **与 `App.tsx` 的交互**：
   - App 接收 `terminalColumns` 和 `terminalRows` props
   - 在 render 中作为 Context value 传递

2. **与 `ink.tsx` 的交互**：
   - 监听 `stdout.on('resize', ...)`
   - 调用 `render()` 更新 App 的 props

3. **与 `use-terminal-viewport.ts` 的交互**（主要消费者）：
   ```typescript
   const terminalSize = useContext(TerminalSizeContext)
   // ...
   const rows = terminalSize?.rows ?? 24
   const viewportBottom = viewportY + rows
   ```

4. **与 `AlternateScreen.tsx` 的交互**：
   - 全屏模式下使用终端尺寸计算布局
   - 处理备用屏幕缓冲区的尺寸适配

---

## 风险、边界与改进建议

### 风险评估

| 风险 | 等级 | 说明 |
|------|------|------|
| 尺寸为 null | 中 | 初始渲染时可能为 null，消费者需处理 |
| 尺寸为 0 | 低 | 极少数终端可能报告 0，需有默认值 |
| 快速连续变化 | 低 | 频繁 resize 可能触发多次重渲染 |

### 边界情况处理

| 场景 | 当前行为 | 建议 |
|------|----------|------|
| `columns/rows` 为 `undefined` | ink.tsx 使用默认值 80/24 | 合理 |
| Context 为 `null` | 消费者需自行处理 | 可添加默认值 |
| 尺寸为 0 | 可能导致除零错误 | 消费者应检查 |
| 非 TTY 环境 | 可能无 columns/rows | 默认值处理 |

### 改进建议

1. **添加默认值配置**：
   ```typescript
   // TerminalSizeContext.tsx
   export const DEFAULT_TERMINAL_SIZE: TerminalSize = {
     columns: 80,
     rows: 24
   }
   
   // 或使用函数获取
   export function getDefaultTerminalSize(): TerminalSize {
     return {
       columns: process.stdout.columns ?? 80,
       rows: process.stdout.rows ?? 24
     }
   }
   ```

2. **提供便捷 Hook**：
   ```typescript
   // 已存在 useTerminalSize.ts，但可内联到 Context 文件
   export function useTerminalSize(): TerminalSize {
     const size = useContext(TerminalSizeContext)
     if (!size) {
       throw new Error('useTerminalSize must be used within TerminalSizeContext.Provider')
     }
     return size
   }
   ```

3. **添加尺寸变化监听 Hook**：
   ```typescript
   export function useOnTerminalSizeChange(callback: (size: TerminalSize) => void) {
     const size = useContext(TerminalSizeContext)
     const prevSize = useRef(size)
     
     useEffect(() => {
       if (prevSize.current && size && 
           (prevSize.current.columns !== size.columns || 
            prevSize.current.rows !== size.rows)) {
         callback(size)
       }
       prevSize.current = size
     }, [size, callback])
   }
   ```

4. **支持响应式断点**：
   ```typescript
   export type Breakpoint = 'xs' | 'sm' | 'md' | 'lg' | 'xl'
   
   export function useBreakpoint(): Breakpoint {
     const { columns } = useTerminalSize()
     if (columns < 80) return 'xs'
     if (columns < 100) return 'sm'
     // ...
   }
   ```

5. **添加尺寸历史**：
   ```typescript
   export type TerminalSizeHistory = {
     current: TerminalSize
     previous: TerminalSize | null
     changeCount: number
   }
   ```
   用于检测尺寸变化趋势。

### 架构思考

当前设计的简洁性是**有意为之**：

```typescript
// 7 行代码完成核心功能
export type TerminalSize = { columns: number; rows: number };
export const TerminalSizeContext = createContext<TerminalSize | null>(null);
```

这种设计遵循**最小 API 原则**：
- 不预设消费者的使用方式
- 不在 Context 层面强加默认值策略
- 保持与其他 Context（`StdinContext`、`TerminalFocusContext`）的一致性

如果需要更复杂的功能（如断点、历史、订阅），应该在**消费者层**或**独立 hooks** 中实现，而不是增加 Context 的复杂度。

### 对比其他 Context

| Context | 复杂度 | 原因 |
|---------|--------|------|
| `StdinContext` | 高 | 需要管理原始模式、事件发射器、查询器 |
| `TerminalFocusContext` | 中 | 需要外部状态存储和同步 |
| `TerminalSizeContext` | 低 | 纯数据传递，无状态管理 |

`TerminalSizeContext` 的极简设计是合理的，因为：
- 尺寸数据是**只读**的（由外部事件驱动更新）
- 无需**副作用**管理
- 无需**引用计数**或**生命周期**管理

这种设计使得它成为 Ink 组件库中最稳定、最易于理解的模块之一。
