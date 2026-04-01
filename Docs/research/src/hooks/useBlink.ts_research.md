# useBlink.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useBlink.ts` 是 Claude Code 的同步闪烁动画 Hook，提供全局同步的闪烁状态，支持在终端失焦时自动暂停，优化性能和用户体验。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 加载指示器 | 闪烁的圆点表示进行中状态 |
| 光标闪烁 | 输入框光标的闪烁效果 |
| 状态指示 | 重要状态的视觉吸引 |
| 终端失焦 | 自动暂停闪烁减少干扰 |

### 1.3 调用方
- `src/components/ToolUseLoader.tsx` - 工具使用加载指示器

---

## 2. 功能点目的

### 2.1 全局同步
- **目的**：所有闪烁元素同步闪烁，避免视觉混乱
- **实现**：基于全局动画时钟，所有实例读取同一时间

### 2.2 智能暂停
- **目的**：终端失焦时暂停动画，节省资源
- **实现**：`useTerminalFocus()` 检测焦点状态

### 2.3 可配置间隔
- **目的**：适应不同场景的闪烁频率需求
- **默认**：600ms（`BLINK_INTERVAL_MS`）

### 2.4 可见性检测
- **目的**：元素不在视口内时暂停动画
- **实现**：`useAnimationFrame` 的可见性感知

---

## 3. 具体技术实现

### 3.1 类型定义

```typescript
export function useBlink(
  enabled: boolean,                    // 是否启用闪烁
  intervalMs: number = BLINK_INTERVAL_MS,  // 闪烁间隔（默认 600ms）
): [
  ref: (element: DOMElement | null) => void,  // 元素引用回调
  isVisible: boolean,                  // 当前是否可见（闪烁状态）
]
```

### 3.2 核心实现

```typescript
const BLINK_INTERVAL_MS = 600

export function useBlink(
  enabled: boolean,
  intervalMs: number = BLINK_INTERVAL_MS,
): [ref: (element: DOMElement | null) => void, isVisible: boolean] {
  // 获取终端焦点状态
  const focused = useTerminalFocus()
  
  // 使用 Ink 的动画帧 Hook
  // - 仅在 enabled && focused 时运行
  // - intervalMs 为 null 时暂停
  const [ref, time] = useAnimationFrame(enabled && focused ? intervalMs : null)

  // 禁用或失焦时始终显示（不闪烁）
  if (!enabled || !focused) return [ref, true]

  // 从时间派生闪烁状态
  // 所有实例使用同一时间，因此同步
  const isVisible = Math.floor(time / intervalMs) % 2 === 0
  
  return [ref, isVisible]
}
```

### 3.3 使用示例

```typescript
// ToolUseLoader.tsx
function BlinkingDot({ shouldAnimate }: { shouldAnimate: boolean }) {
  const [ref, isVisible] = useBlink(shouldAnimate)
  
  return (
    <Box ref={ref}>
      {isVisible ? '●' : ' '}
    </Box>
  )
}
```

### 3.4 动画时钟原理

```
时间线（intervalMs = 600ms）:

0ms     600ms    1200ms   1800ms
|--------|--------|--------|→
[ 可见 ] [ 隐藏 ] [ 可见 ] [ 隐藏 ]
   ●        ○        ●        ○

Math.floor(time / 600) % 2:
- 0-599ms:   floor(0) % 2 = 0 (可见)
- 600-1199ms: floor(1) % 2 = 1 (隐藏)
- 1200-1799ms: floor(2) % 2 = 0 (可见)
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
useBlink.ts
├── ink.js
│   ├── useAnimationFrame   (动画帧管理)
│   ├── useTerminalFocus    (终端焦点检测)
│   └── DOMElement          (元素类型)
```

### 4.2 调用链

```
ToolUseLoader.tsx
  └── useBlink(shouldAnimate)
      ├── useTerminalFocus()    → focused: boolean
      ├── useAnimationFrame()   → [ref, time]
      └── Math.floor(time / intervalMs) % 2 → isVisible
```

### 4.3 Ink 动画系统

```typescript
// ink.js 内部实现概览
function useAnimationFrame(intervalMs: number | null): [RefCallback, number] {
  const [time, setTime] = useState(0)
  
  useEffect(() => {
    if (intervalMs === null) return
    
    let animationId: number
    const tick = () => {
      setTime(performance.now())
      animationId = requestAnimationFrame(tick)
    }
    
    animationId = requestAnimationFrame(tick)
    return () => cancelAnimationFrame(animationId)
  }, [intervalMs])
  
  const ref = useCallback((element: DOMElement | null) => {
    // 可见性检测逻辑
  }, [])
  
  return [ref, time]
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `ink.js` | React 终端 UI 库 | npm 包 |

### 5.2 Ink API

| API | 用途 |
|-----|------|
| `useAnimationFrame()` | 提供全局动画时钟 |
| `useTerminalFocus()` | 检测终端焦点状态 |
| `DOMElement` | 元素引用类型 |

### 5.3 浏览器 API

| API | 用途 |
|-----|------|
| `performance.now()` | 高精度时间戳 |
| `requestAnimationFrame()` | 动画帧调度 |
| `cancelAnimationFrame()` | 取消动画帧 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 性能开销 | 大量闪烁元素可能增加渲染负担 | `useAnimationFrame` 的可见性优化 |
| 时间漂移 | 长时间运行可能导致不同步 | 使用全局时间而非本地计数器 |
| 不支持终端 | 某些终端不支持焦点检测 | 默认显示（`true`） |

### 6.2 边界条件

1. **`enabled=false`**：始终返回 `isVisible=true`
2. **`intervalMs=0`**：可能导致高频闪烁
3. **终端失焦**：自动暂停，返回 `isVisible=true`
4. **元素离屏**：`useAnimationFrame` 自动暂停
5. **快速切换**：焦点变化时动画平滑过渡

### 6.3 改进建议

1. **缓动函数**：
   ```typescript
   // 支持自定义缓动
   const isVisible = ease(Math.floor(time / intervalMs) % 2)
   ```

2. **多阶段闪烁**：
   ```typescript
   // 支持更复杂的闪烁模式
   const phase = Math.floor(time / intervalMs) % 4
   const opacity = [1, 0.5, 0, 0.5][phase]
   ```

3. **颜色过渡**：
   ```typescript
   // 支持颜色渐变而不仅是显示/隐藏
   const [ref, color] = useColorBlink(enabled, ['red', 'yellow', 'green'])
   ```

4. **可访问性**：
   ```typescript
   // 支持减少动画偏好
   const prefersReducedMotion = usePrefersReducedMotion()
   const [ref, isVisible] = useBlink(enabled && !prefersReducedMotion)
   ```

5. **精确控制**：
   ```typescript
   // 支持手动控制闪烁状态
   const [ref, isVisible, setVisible] = useBlink(enabled, { controlled: true })
   ```

### 6.4 代码质量

- **优点**：
  - 极简 API，易于使用
  - 全局同步确保一致性
  - 智能暂停优化性能
  - 合理的默认值
  
- **潜在改进**：
  - 添加 JSDoc 说明 `intervalMs` 的限制
  - 考虑验证 `intervalMs` 为正数
  - 添加更多使用示例

### 6.5 性能考虑

| 方面 | 实现 | 影响 |
|------|------|------|
| 全局时钟 | 单一定时器驱动所有实例 | 低 |
| 焦点暂停 | 失焦时停止 `requestAnimationFrame` | 显著节省 |
| 可见性检测 | 离屏元素自动暂停 | 中等节省 |
| 时间计算 | 简单数学运算 | 可忽略 |

### 6.6 相关文档

- `ink.js` 文档 - React 终端 UI 库
- `useAnimationFrame` 源码 - Ink 内部实现
- `ToolUseLoader.tsx` - 实际使用示例
