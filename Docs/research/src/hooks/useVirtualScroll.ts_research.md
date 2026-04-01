# useVirtualScroll.ts 研究文档

## 场景与职责

`useVirtualScroll` 是一个高性能的 React Hook，用于在 Claude Code 的全屏消息列表中实现虚拟滚动。它解决了在大量消息（数千至数万条）场景下的性能问题。

核心职责：
1. **视口裁剪**：只渲染视口内及周围的项目，而非全部消息
2. **高度估算与测量**：动态测量项目高度并缓存，优化滚动体验
3. **滚动位置管理**：维护滚动状态，支持跳转到指定项目
4. **响应式适配**：处理终端宽度变化时的布局调整
5. **性能优化**：使用多种技术（量化、延迟渲染、滑动窗口）确保流畅滚动

该 Hook 被 `VirtualMessageList.tsx` 组件使用，后者是 Claude Code 全屏模式下显示对话历史的核心组件。

## 功能点目的

### 1. 虚拟滚动核心

```typescript
export type VirtualScrollResult = {
  range: readonly [number, number]  // [startIndex, endIndex) 半开区间
  topSpacer: number                 // 顶部占位高度（行数）
  bottomSpacer: number              // 底部占位高度（行数）
  measureRef: (key: string) => (el: DOMElement | null) => void
  spacerRef: RefObject<DOMElement | null>
  offsets: ArrayLike<number>        // 累积偏移量数组
  getItemTop: (index: number) => number
  getItemElement: (index: number) => DOMElement | null
  getItemHeight: (index: number) => number | undefined
  scrollToIndex: (i: number) => void
}
```

### 2. 关键常量配置

```typescript
const DEFAULT_ESTIMATE = 3           // 未测量项目的估算高度（故意设低）
const OVERSCAN_ROWS = 80             // 视口上下额外渲染的行数
const COLD_START_COUNT = 30          // ScrollBox 未布局时渲染的项目数
const SCROLL_QUANTUM = OVERSCAN_ROWS >> 1  // 滚动量化步长（40行）
const PESSIMISTIC_HEIGHT = 1         // 未测量项目的最小高度
const MAX_MOUNTED_ITEMS = 300        // 最大挂载项目数限制
const SLIDE_STEP = 25                // 单次提交最大新增项目数
```

### 3. 高度缓存与缩放

当终端宽度变化时，使用比例缩放而非清除缓存：

```typescript
if (prevColumns.current !== columns) {
  const ratio = prevColumns.current / columns
  prevColumns.current = columns
  for (const [k, h] of heightCache.current) {
    heightCache.current.set(k, Math.max(1, Math.round(h * ratio)))
  }
  offsetVersionRef.current++
  skipMeasurementRef.current = true
  freezeRendersRef.current = 2
}
```

这避免了在调整大小时重新测量所有项目（约 600ms 的 React 协调时间）。

### 4. 滚动量化

使用 `useSyncExternalStore` 实现滚动位置订阅，但将 scrollTop 量化为 `SCROLL_QUANTUM` 的倍数：

```typescript
useSyncExternalStore(subscribe, () => {
  const s = scrollRef.current
  if (!s) return NaN
  const target = s.getScrollTop() + s.getPendingDelta()
  const bin = Math.floor(target / SCROLL_QUANTUM)
  return s.isSticky() ? ~bin : bin  // 使用符号位编码 sticky 状态
})
```

这样，小的滚动（如鼠标滚轮的 3-5 像素）不会触发 React 重新渲染，减少 CPU 使用。

### 5. 范围计算逻辑

```typescript
if (isSticky) {
  // 粘性滚动：从尾部向前遍历，直到覆盖视口 + overscan
  const budget = viewportH + OVERSCAN_ROWS
  start = n
  while (start > 0 && totalHeight - offsets[start - 1]! < budget) {
    start--
  }
  end = n
} else {
  // 用户向上滚动：使用二分查找确定 start
  // ...
}
```

### 6. 延迟渲染（useDeferredValue）

```typescript
const dStart = useDeferredValue(start)
const dEnd = useDeferredValue(end)
let effStart = start < dStart ? dStart : start
let effEnd = end > dEnd ? dEnd : end
```

使用 React 的 `useDeferredValue` 让紧急渲染使用旧范围（缓存命中），非阻塞后台渲染使用新范围（可能涉及新项目挂载）。

## 具体技术实现

### 关键流程

#### 1. 初始化流程

```typescript
if (viewportH === 0 || scrollTop < 0) {
  // Cold start: ScrollBox 还未布局
  start = Math.max(0, n - COLD_START_COUNT)
  end = n
}
```

在首次渲染时，ScrollBox 的视口高度为 0，此时渲染尾部的 30 条消息（用户最可能看到的）。

#### 2. 范围计算（非粘性滚动）

```typescript
// 1. 计算有效滚动范围（考虑 pendingDelta）
const MAX_SPAN_ROWS = viewportH * 3
const rawLo = Math.min(scrollTop, scrollTop + pendingDelta)
const rawHi = Math.max(scrollTop, scrollTop + pendingDelta)
const span = rawHi - rawLo
const clampedLo = span > MAX_SPAN_ROWS
  ? pendingDelta < 0 ? rawHi - MAX_SPAN_ROWS : rawLo
  : rawLo
const clampedHi = clampedLo + Math.min(span, MAX_SPAN_ROWS)

// 2. 转换为列表本地坐标
const listOrigin = listOriginRef.current
const effLo = Math.max(0, clampedLo - listOrigin)
const effHi = clampedHi - listOrigin
const lo = effLo - OVERSCAN_ROWS

// 3. 二分查找确定 start
let l = 0, r = n
while (l < r) {
  const m = (l + r) >> 1
  if (offsets[m + 1]! <= lo) l = m + 1
  else r = m
}
start = l

// 4. 确保不卸载已挂载但未测量的项目
const p = prevRangeRef.current
if (p && p[0] < start) {
  for (let i = p[0]; i < Math.min(start, p[1]); i++) {
    const k = itemKeys[i]!
    if (itemRefs.current.has(k) && !heightCache.current.has(k)) {
      start = i
      break
    }
  }
}

// 5. 向后遍历确定 end，直到覆盖 needed 高度
const needed = viewportH + 2 * OVERSCAN_ROWS
const maxEnd = Math.min(n, start + MAX_MOUNTED_ITEMS)
let coverage = 0
end = start
while (end < maxEnd && coverage < needed) {
  coverage += heightCache.current.get(itemKeys[end]!) ?? PESSIMISTIC_HEIGHT
  end++
}
```

#### 3. 滑动窗口限制

```typescript
const prev = prevRangeRef.current
const scrollVelocity = Math.abs(scrollTop - lastScrollTopRef.current) + Math.abs(pendingDelta)
if (prev && scrollVelocity > viewportH * 2) {
  const [pS, pE] = prev
  if (start < pS - SLIDE_STEP) start = pS - SLIDE_STEP
  if (end > pE + SLIDE_STEP) end = pE + SLIDE_STEP
  if (start > end) end = Math.min(start + SLIDE_STEP, n)
}
```

当快速滚动时，限制单次渲染新增的项目数，避免一次性挂载 194 个项目导致的 290ms+ 渲染阻塞。

#### 4. 高度测量

```typescript
useLayoutEffect(() => {
  const spacerYoga = spacerRef.current?.yogaNode
  if (spacerYoga && spacerYoga.getComputedWidth() > 0) {
    listOriginRef.current = spacerYoga.getComputedTop()
  }
  if (skipMeasurementRef.current) {
    skipMeasurementRef.current = false
    return
  }
  let anyChanged = false
  for (const [key, el] of itemRefs.current) {
    const yoga = el.yogaNode
    if (!yoga) continue
    const h = yoga.getComputedHeight()
    const prev = heightCache.current.get(key)
    if (h > 0) {
      if (prev !== h) {
        heightCache.current.set(key, h)
        anyChanged = true
      }
    } else if (yoga.getComputedWidth() > 0 && prev !== 0) {
      heightCache.current.set(key, 0)
      anyChanged = true
    }
  }
  if (anyChanged) offsetVersionRef.current++
})
```

使用 `useLayoutEffect` 在布局完成后读取 Yoga 计算的高度。区分 "高度为 0 因为尚未布局" 和 "高度为 0 因为内容为空"：
- 如果 `getComputedWidth() > 0`，说明 Yoga 已经布局过该节点
- 此时高度为 0 表示项目确实渲染了空内容

#### 5. 测量引用工厂

```typescript
const measureRef = useCallback((key: string) => {
  let fn = refCache.current.get(key)
  if (!fn) {
    fn = (el: DOMElement | null) => {
      if (el) {
        itemRefs.current.set(key, el)
      } else {
        // 卸载时捕获最终高度
        const yoga = itemRefs.current.get(key)?.yogaNode
        if (yoga && !skipMeasurementRef.current) {
          const h = yoga.getComputedHeight()
          if ((h > 0 || yoga.getComputedWidth() > 0) &&
              heightCache.current.get(key) !== h) {
            heightCache.current.set(key, h)
            offsetVersionRef.current++
          }
        }
        itemRefs.current.delete(key)
      }
    }
    refCache.current.set(key, fn)
  }
  return fn
}, [])
```

使用稳定的回调引用，避免每次渲染都创建新的函数引用。

### 数据结构

#### offsets 数组

```typescript
const offsetsRef = useRef<{ arr: Float64Array; version: number; n: number }>({
  arr: new Float64Array(0),
  version: -1,
  n: -1,
})

// 重建逻辑
if (offsetsRef.current.version !== offsetVersionRef.current ||
    offsetsRef.current.n !== n) {
  const arr = offsetsRef.current.arr.length >= n + 1
    ? offsetsRef.current.arr
    : new Float64Array(n + 1)
  arr[0] = 0
  for (let i = 0; i < n; i++) {
    arr[i + 1] = arr[i]! + (heightCache.current.get(itemKeys[i]!) ?? DEFAULT_ESTIMATE)
  }
  offsetsRef.current = { arr, version: offsetVersionRef.current, n }
}
```

- `offsets[i]` = 第 i 个项目之前的累积高度
- `offsets[n]` = 总高度
- 使用 `Float64Array` 存储，支持大型数组
- 版本控制避免不必要的重建

### 性能优化技术

1. **滚动量化**：将 scrollTop 量化为 40 行的倍数，减少 React 提交频率
2. **延迟渲染**：`useDeferredValue` 将新项目挂载推迟到后台渲染
3. **滑动窗口**：`SLIDE_STEP` 限制单次渲染新增项目数
4. **二分查找**：O(log n) 的 start 查找，替代 O(n) 的线性扫描
5. **高度缓存复用**：终端宽度变化时缩放缓存而非清除
6. **范围冻结**：调整大小时冻结范围 2 个渲染周期，避免挂载/卸载抖动
7. **GC 清理**：当项目从 `itemKeys` 中移除时，清理对应的高度缓存

## 关键代码路径与文件引用

### 本文件
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useVirtualScroll.ts` - Hook 实现

### 依赖文件
| 文件 | 用途 |
|------|------|
| `react` (useCallback, useDeferredValue, useLayoutEffect, useMemo, useRef, useSyncExternalStore) | React API |
| `../ink/components/ScrollBox.js` | ScrollBoxHandle 类型 |
| `../ink/dom.js` | DOMElement 类型 |

### 调用方
- `/home/sansha/Github/claude-code-instructkr/src/components/VirtualMessageList.tsx` - 虚拟消息列表组件

### 相关文件
| 文件 | 用途 |
|------|------|
| `../ink/render-node-to-output.ts` | Ink 渲染器，处理视口裁剪 |
| `../ink/components/ScrollBox.tsx` | ScrollBox 组件实现 |

## 依赖与外部交互

### ScrollBoxHandle 接口

```typescript
type ScrollBoxHandle = {
  subscribe: (listener: () => void) => () => void
  getScrollTop: () => number
  getPendingDelta: () => number
  getViewportHeight: () => number
  isSticky: () => boolean
  scrollTo: (y: number) => void
  setClampBounds: (min?: number, max?: number) => void
}
```

### DOMElement 接口

```typescript
type DOMElement = {
  yogaNode: {
    getComputedHeight: () => number
    getComputedWidth: () => number
    getComputedTop: () => number
  }
}
```

## 风险、边界与改进建议

### 潜在风险

1. **高度估算误差**：
   - `DEFAULT_ESTIMATE = 3` 可能远低于实际高度
   - 虽然 overscan 可以吸收误差，但极端情况下可能导致空白

2. **快速滚动时的空白**：
   - `SLIDE_STEP` 限制可能导致快速滚动时看到空白
   - `setClampBounds` 用于缓解此问题，但仍有边界情况

3. **内存使用**：
   - `heightCache` 和 `offsets` 数组随消息数量增长
   - 对于 27k 条消息，offsets 数组约占用 216KB（27k * 8 bytes）

4. **并发模式下的竞态**：
   - `useDeferredValue` 和 `useLayoutEffect` 的交互复杂
   - 可能存在边缘情况导致范围计算不一致

### 边界情况

| 场景 | 处理 |
|------|------|
| 空列表 (n = 0) | `start = end = 0`，`topSpacer = bottomSpacer = 0` |
| 视口高度为 0 | Cold start 路径，渲染尾部 30 项 |
| 粘性滚动到底部 | 从尾部向前遍历计算范围 |
| 终端宽度变化 | 缩放高度缓存，冻结范围 2 个周期 |
| 项目高度为 0 | 缓存 0，确保 start 推进不会被阻塞 |
| 快速滚动（速度 > 2x 视口） | 应用滑动窗口限制 |

### 改进建议

1. **自适应估算**：
   - 基于已测量项目的平均高度动态调整 `DEFAULT_ESTIMATE`
   - 对于长消息会话，可以提高估算精度

2. **预加载优化**：
   - 当前只根据滚动方向加载
   - 可考虑预加载用户可能滚动到的方向

3. **内存优化**：
   - 对于非常长的会话，可考虑 LRU 缓存策略
   - 清理长时间未见的项目的高度缓存

4. **滚动锚定**：
   - 当前依赖 overscan 吸收估算误差
   - 可考虑实现滚动锚定，在高度变化时自动调整 scrollTop

5. **测试覆盖**：
   - 添加单元测试覆盖各种滚动场景
   - 测试边界情况（空列表、单项目、极端高度差异）

6. **可配置常量**：
   - 将 `OVERSCAN_ROWS`、`MAX_MOUNTED_ITEMS` 等设为可配置
   - 允许根据设备性能调整
