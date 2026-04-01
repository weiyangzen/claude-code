# use-terminal-viewport.ts 深入研究

## 场景与职责

`useTerminalViewport` 是 Ink 终端 UI 框架中用于检测组件是否在终端视口内的 Hook。它通过计算元素的 Yoga 布局位置和终端尺寸，确定元素是否可见，为动画、虚拟滚动等功能提供视口感知能力。

## 功能点目的

### 1. 视口可见性检测
- 检测组件是否在终端视口内
- 考虑滚动偏移（scrollTop）
- 处理溢出内容（cursor-restore scroll）

### 2. 性能优化
- 在 layout 阶段更新（useLayoutEffect）
- 不触发额外的 React 重渲染
- 调用者在自己的重渲染中读取最新值

### 3. 滚动容器支持
- 检测元素是否在滚动容器内
- 正确处理 scrollTop 偏移
- 支持嵌套滚动容器

## 具体技术实现

### 接口定义

```typescript
type ViewportEntry = {
  isVisible: boolean  // 元素是否在视口内
}

export function useTerminalViewport(): [
  ref: (element: DOMElement | null) => void,
  entry: ViewportEntry,
]
```

### 核心实现逻辑

1. **Context 和 Ref 设置**：
   ```typescript
   const terminalSize = useContext(TerminalSizeContext)
   const elementRef = useRef<DOMElement | null>(null)
   const entryRef = useRef<ViewportEntry>({ isVisible: true })

   const setElement = useCallback((el: DOMElement | null) => {
     elementRef.current = el
   }, [])
   ```

2. **布局计算（useLayoutEffect）**：
   ```typescript
   useLayoutEffect(() => {
     const element = elementRef.current
     if (!element?.yogaNode || !terminalSize) return

     const height = element.yogaNode.getComputedHeight()
     const rows = terminalSize.rows

     // 计算绝对顶部位置
     let absoluteTop = element.yogaNode.getComputedTop()
     let parent: DOMElement | undefined = element.parentNode
     let root = element.yogaNode
     
     while (parent) {
       if (parent.yogaNode) {
         absoluteTop += parent.yogaNode.getComputedTop()
         root = parent.yogaNode
       }
       // 减去滚动容器的 scrollTop
       if (parent.scrollTop) absoluteTop -= parent.scrollTop
       parent = parent.parentNode
     }

     const screenHeight = root.getComputedHeight()
   ```

3. **可见性计算**：
   ```typescript
     // 处理 cursor-restore 滚动
     const cursorRestoreScroll = screenHeight > rows ? 1 : 0
     const viewportY = Math.max(0, screenHeight - rows) + cursorRestoreScroll
     const viewportBottom = viewportY + rows
     const visible = bottom > viewportY && absoluteTop < viewportBottom

     if (visible !== entryRef.current.isVisible) {
       entryRef.current = { isVisible: visible }
     }
   })
   ```

### 关键技术点

1. **Yoga 布局遍历**：
   - 遍历 DOM 父链（而非 yoga.getParent()）
   - 检测滚动容器并减去 scrollTop
   - Yoga 计算布局位置时不考虑滚动偏移

2. **Cursor-restore 滚动补偿**：
   ```typescript
   const cursorRestoreScroll = screenHeight > rows ? 1 : 0
   ```
   当内容溢出视口时，光标恢复会额外滚动一行到 scrollback，需要匹配这个行为

3. **Ref 更新模式**：
   - 直接修改 `entryRef.current`
   - 不调用 `setState`，避免级联重渲染
   - 调用者在下次重渲染时读取新值

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/TerminalSizeContext.tsx` | 提供终端尺寸（columns/rows） |
| `src/ink/dom.ts` | 定义 DOMElement 类型和 Yoga 节点 |

### DOMElement 相关定义

```typescript
// src/ink/dom.ts
export type DOMElement = {
  nodeName: ElementNames
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  // ...
  
  // 滚动状态
  scrollTop?: number                    // 滚动偏移
  scrollHeight?: number                 // 内容高度
  scrollViewportHeight?: number         // 视口高度
  // ...
  
  // Yoga 节点
  yogaNode?: LayoutNode
  
  // 父节点
  parentNode: DOMElement | undefined
} & InkNode
```

### 使用示例

```typescript
import { useTerminalViewport } from 'ink'

const AnimationComponent = () => {
  const [ref, entry] = useTerminalViewport()
  
  // 只在可见时运行动画
  const [animationRef, time] = useAnimationFrame(entry.isVisible ? 16 : null)
  
  return (
    <Box ref={ref}>
      <Animation enabled={entry.isVisible} time={time}>
        Content
      </Animation>
    </Box>
  )
}
```

### 在 useAnimationFrame 中的使用

```typescript
// use-animation-frame.ts
export function useAnimationFrame(intervalMs: number | null) {
  const [viewportRef, { isVisible }] = useTerminalViewport()
  const active = isVisible && intervalMs !== null
  
  useEffect(() => {
    if (!active) return
    // 订阅时钟...
  }, [active])
  
  return [viewportRef, time]
}
```

## 依赖与外部交互

### 与 TerminalSizeContext 的交互

```typescript
const terminalSize = useContext(TerminalSizeContext)
// terminalSize: { columns: number, rows: number }
```

- 提供终端的行数（rows）
- 用于计算视口边界

### 与 Yoga 布局的交互

1. **布局计算**：
   - `yogaNode.getComputedTop()` - 获取相对顶部位置
   - `yogaNode.getComputedHeight()` - 获取计算高度
   - `yogaNode.getComputedWidth()` - 获取计算宽度

2. **布局更新**：
   - Yoga 布局变化不会通知 React
   - useLayoutEffect 每次渲染都重新计算
   - 确保读取最新的布局值

### 与滚动容器的交互

```typescript
// 检测滚动偏移
if (parent.scrollTop) absoluteTop -= parent.scrollTop
```

- `scrollTop` 由 ScrollBox 组件和渲染器设置
- 非滚动节点为 undefined（falsy）
- 正确计算元素在滚动后的可见位置

### 布局时序

```
React 渲染
    ↓
Yoga 布局计算
    ↓
React layout 阶段（useLayoutEffect）
    ↓
计算视口可见性
    ↓
渲染器输出到终端
```

## 风险、边界与改进建议

### 潜在风险

1. **布局抖动**：
   - 每次渲染都遍历父链
   - 深层嵌套可能影响性能
   - 考虑添加缓存机制

2. **时序问题**：
   - Yoga 布局可能在 useLayoutEffect 之后更新
   - 极端情况下可能读取过期值

3. **ScrollTop 同步**：
   - 依赖 `scrollTop` 属性正确设置
   - 如果设置不正确，可见性判断错误

### 边界情况

1. **无 Yoga 节点**：
   - 某些元素类型（如 `ink-virtual-text`）没有 yogaNode
   - 提前返回，不计算可见性

2. **无终端尺寸**：
   - `terminalSize` 为 null 时提前返回
   - 默认视为可见

3. **屏幕高度等于行数**：
   - `cursorRestoreScroll` 为 0
   - 无额外滚动

4. **元素在视口边界**：
   - `bottom > viewportY && absoluteTop < viewportBottom`
   - 部分可见视为可见

### 改进建议

1. **添加可见性变化回调**：
   ```typescript
   useTerminalViewport({
     onVisibilityChange: (isVisible) => {
       console.log('Visibility changed:', isVisible)
     }
   })
   ```

2. **支持可见性比例**：
   ```typescript
   type ViewportEntry = {
     isVisible: boolean
     visibleRatio: number  // 0-1，可见部分比例
     visibleRows: number   // 可见行数
   }
   ```

3. **添加 Intersection Observer 模式**：
   ```typescript
   const [ref, entry] = useTerminalViewport({
     threshold: 0.5  // 50% 可见才视为可见
   })
   ```

4. **性能优化**：
   ```typescript
   // 缓存计算结果
   const cacheKey = useMemo(() => ({
     top: absoluteTop,
     height,
     viewportY,
     rows
   }), [/* deps */])
   
   const isVisible = useMemo(() => 
     computeVisibility(cacheKey),
     [cacheKey]
   )
   ```

5. **支持水平视口检测**：
   ```typescript
   type ViewportEntry = {
     isVisible: boolean
     isVisibleX: boolean  // 水平方向
     isVisibleY: boolean  // 垂直方向
   }
   ```

6. **添加调试模式**：
   ```typescript
   useTerminalViewport({ debug: true })
   // 在元素周围显示可见性边界框
   ```

### 测试建议

1. 测试元素在视口内/外/边界
2. 测试滚动容器内的元素
3. 测试嵌套滚动容器
4. 测试屏幕高度变化
5. 测试无 Yoga 节点的元素
6. 测试快速滚动场景
7. 测试 cursor-restore 滚动场景
