# useTerminalSize.ts 深度研究文档

## 场景与职责

`useTerminalSize` 是一个极简的 React Hook，用于获取当前终端的尺寸（行数和列数）。它是 Ink（React 终端 UI 库）应用的基础工具 Hook。

### 核心职责

1. **终端尺寸获取**: 从 React Context 获取当前终端尺寸
2. **错误处理**: 确保在 Ink App 组件树内使用

### 使用场景

- **文本换行**: 根据终端宽度计算文本换行
- **光标定位**: 基于终端尺寸定位光标
- **布局计算**: 计算组件布局和尺寸
- **响应式 UI**: 根据终端尺寸调整 UI

---

## 功能点目的

### 1. 终端尺寸访问

提供对终端尺寸的响应式访问：
- 行数（rows）
- 列数（columns）

### 2. Context 验证

确保正确使用：
- 必须在 Ink App 组件树内使用
- 否则抛出明确的错误信息

---

## 具体技术实现

### 关键数据结构

```typescript
// 终端尺寸类型
interface TerminalSize {
  rows: number    // 终端行数
  columns: number // 终端列数
}

// Hook 返回类型
function useTerminalSize(): TerminalSize
```

### 核心流程

```
调用 useTerminalSize()
  ↓
使用 useContext 获取 TerminalSizeContext
  ↓
检查 size 是否存在
  ↓
不存在 → 抛出错误
  ↓
存在 → 返回 size
```

### 关键代码路径

#### Hook 实现（行 7-15）
```typescript
export function useTerminalSize(): TerminalSize {
  const size = useContext(TerminalSizeContext)

  if (!size) {
    throw new Error('useTerminalSize must be used within an Ink App component')
  }

  return size
}
```

#### Context 定义（来自 src/ink/components/TerminalSizeContext.js）
```typescript
interface TerminalSizeContextValue {
  rows: number
  columns: number
}

const TerminalSizeContext = React.createContext<TerminalSizeContextValue | null>(null)
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `react` | `useContext` |
| `src/ink/components/TerminalSizeContext.js` | `TerminalSizeContext` |

### 外部交互

1. **Ink 框架**: 
   - `TerminalSizeContext`: 提供终端尺寸
   - 由 Ink 的 App 组件提供值

---

## 风险、边界与改进建议

### 已知风险

1. **Context 依赖**: 强依赖 Ink 框架的 Context，不能在其他环境中使用
2. **无默认值**: 没有提供默认值，必须在 Ink App 内使用

### 边界情况

1. **终端尺寸变化**: 终端调整大小时，Context 值会自动更新
2. **服务端渲染**: 在服务端可能无法获取正确的终端尺寸

### 改进建议

1. **默认值支持**: 提供可选的默认值参数，在非 Ink 环境也能工作
   ```typescript
   export function useTerminalSize(defaultSize?: TerminalSize): TerminalSize
   ```

2. **尺寸变化监听**: 添加尺寸变化回调支持
   ```typescript
   export function useTerminalSize(onResize?: (size: TerminalSize) => void): TerminalSize
   ```

3. **响应式断点**: 提供基于断点的响应式工具
   ```typescript
   const isNarrow = useTerminalSize().columns < 80
   ```

### 测试关注点

1. 在 Ink App 内正确获取尺寸
2. 在 Ink App 外抛出正确错误
3. 终端尺寸变化时的更新
