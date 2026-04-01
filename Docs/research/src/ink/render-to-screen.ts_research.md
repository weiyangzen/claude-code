# render-to-screen.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`render-to-screen.ts` 是 Ink 的独立渲染工具模块，提供将 React 元素渲染到隔离屏幕缓冲区的功能。与主渲染流程（`renderer.ts` + `renderNodeTo-output.ts`）不同，此模块用于**离屏渲染**场景，特别是搜索高亮功能。

### 1.2 核心职责
- **离屏渲染**：将单个 React 元素渲染到独立的 Screen 缓冲区
- **搜索扫描**：在渲染后的屏幕缓冲区中扫描匹配文本位置
- **高亮应用**：在特定位置应用搜索匹配高亮样式
- **性能计时**：收集渲染各阶段的性能数据

### 1.3 使用场景
- **消息搜索**：在消息列表中查找搜索查询的匹配位置
- **搜索高亮**：为匹配文本添加视觉高亮（反色、黄色背景等）
- **位置计算**：获取匹配文本在消息中的精确行列位置

---

## 2. 功能点目的

### 2.1 renderToScreen - 离屏渲染

**设计目的**：
- 渲染单个消息组件（而非整个应用树）
- 在隔离环境中计算布局和渲染，不影响主屏幕
- 支持搜索功能的消息级扫描

**性能特征**（行 48-56）：
```typescript
/**
 * ~1-3ms per call (yoga alloc + calculateLayout + paint). The
 * flushSyncWork cross-root leak measured ~0.0003ms/call growth — fine
 * for on-demand single-message rendering, pathological for render-all-
 * 8k-upfront. Cache per (msg, query, width) upstream.
 */
```

### 2.2 scanPositions - 匹配位置扫描

**设计目的**：
- 在渲染后的屏幕缓冲区中查找所有匹配查询的位置
- 返回相对于消息边界的行列位置（row 0 = 消息顶部）
- 支持宽字符和组合字符的精确位置计算

**匹配逻辑**（行 149-201）：
- 逐行构建小写文本
- 跳过 SpacerTail/SpacerHead/noSelect 单元格
- 处理代理对（surrogate pairs）和多单元小写字符（如土耳其语 İ）
- 非重叠匹配（`pos = text.indexOf(lq, pos + qlen)`）

### 2.3 applyPositionedHighlight - 当前匹配高亮

**设计目的**：
- 标记当前选中的匹配项（vs 其他匹配项）
- 使用黄色背景 + 粗体 + 下划线，使其在反色高亮中脱颖而出
- 两层高亮：scan = "你可以到这里"，position = "你在这里"

---

## 3. 具体技术实现

### 3.1 模块级状态（行 38-44）

```typescript
let root: DOMElement | undefined
let container: ReturnType<typeof reconciler.createContainer> | undefined
let stylePool: StylePool | undefined
let charPool: CharPool | undefined
let hyperlinkPool: HyperlinkPool | undefined
let output: Output | undefined
```

**设计决策**：
- 使用模块级变量实现跨调用复用
- 避免每帧重新创建 Yoga 根节点和 React 容器（节省 ~1ms）
- 使用 `LegacyRoot`：同步执行，无调度开销

### 3.2 renderToScreen 函数（行 59-139）

**执行流程**：

1. **延迟初始化**（行 63-82）
   ```typescript
   if (!root) {
     root = createNode('ink-root')
     root.focusManager = new FocusManager(() => false)
     stylePool = new StylePool()
     charPool = new CharPool()
     hyperlinkPool = new HyperlinkPool()
     container = reconciler.createContainer(root, LegacyRoot, null, false, ...)
   }
   ```

2. **React 渲染**（行 84-89）
   ```typescript
   reconciler.updateContainerSync(el, container, null, noop)
   reconciler.flushSyncWork()
   ```

3. **Yoga 布局**（行 91-95）
   ```typescript
   root.yogaNode?.setWidth(width)
   root.yogaNode?.calculateLayout(width)
   const height = Math.ceil(root.yogaNode?.getComputedHeight() ?? 0)
   ```

4. **屏幕创建与渲染**（行 97-116）
   ```typescript
   const screen = createScreen(width, Math.max(1, height), stylePool!, charPool!, hyperlinkPool!)
   output.reset(width, height, screen)
   resetLayoutShifted()
   renderNodeToOutput(root, output, { prevScreen: undefined })
   const rendered = output.get()
   ```

5. **卸载清理**（行 119-123）
   ```typescript
   reconciler.updateContainerSync(null, container, null, noop)
   reconciler.flushSyncWork()
   ```
   保留 root/container/pools，仅卸载组件树

6. **性能记录**（行 125-136）
   ```typescript
   timing.reconcile += t1 - t0
   timing.yoga += t2 - t1
   timing.paint += t3 - t2
   if (++timing.calls % LOG_EVERY === 0) {
     logForDebugging(`renderToScreen: ${timing.calls} calls...`)
   }
   ```

### 3.3 scanPositions 函数（行 149-201）

**算法详解**：

1. **逐行扫描**（行 159-197）
   ```typescript
   for (let row = 0; row < h; row++) {
     const rowOff = row * w
     let text = ''
     const colOf: number[] = []      // 字符索引 → 列号
     const codeUnitToCell: number[] = // 代码单元索引 → 单元格索引
     
     for (let col = 0; col < w; col++) {
       const idx = rowOff + col
       const cell = cellAtIndex(screen, idx)
       
       // 跳过间隔单元格和不可选择区域
       if (cell.width === CellWidth.SpacerTail ||
           cell.width === CellWidth.SpacerHead ||
           noSelect[idx] === 1) {
         continue
       }
       
       const lc = cell.char.toLowerCase()
       const cellIdx = colOf.length
       for (let i = 0; i < lc.length; i++) {
         codeUnitToCell.push(cellIdx)
       }
       text += lc
       colOf.push(col)
     }
     
     // 非重叠匹配查找
     let pos = text.indexOf(lq)
     while (pos >= 0) {
       const startCi = codeUnitToCell[pos]!
       const endCi = codeUnitToCell[pos + qlen - 1]!
       const col = colOf[startCi]!
       const endCol = colOf[endCi]! + 1
       positions.push({ row, col, len: endCol - col })
       pos = text.indexOf(lq, pos + qlen)
     }
   }
   ```

**关键处理**：
- **codeUnitToCell 映射**：处理多代码单元字符（如 Emoji、组合字符）
- **colOf 映射**：字符索引到实际列号的映射
- **len 计算**：匹配宽度（考虑宽字符）

### 3.4 applyPositionedHighlight 函数（行 212-231）

**实现逻辑**：

```typescript
export function applyPositionedHighlight(
  screen: Screen,
  stylePool: StylePool,
  positions: MatchPosition[],
  rowOffset: number,
  currentIdx: number,
): boolean {
  if (currentIdx < 0 || currentIdx >= positions.length) return false
  
  const p = positions[currentIdx]!
  const row = p.row + rowOffset  // 转换为屏幕坐标
  
  if (row < 0 || row >= screen.height) return false
  
  // 应用 CURRENT 样式（黄色+粗体+下划线）
  const transform = (id: number) => stylePool.withCurrentMatch(id)
  const rowOff = row * screen.width
  
  for (let col = p.col; col < p.col + p.len; col++) {
    if (col < 0 || col >= screen.width) continue
    const cell = cellAtIndex(screen, rowOff + col)
    setCellStyleId(screen, col, row, transform(cell.styleId))
  }
  return true
}
```

**样式说明**：
- `withCurrentMatch` 在 `StylePool` 中定义
- 添加黄色前景（通过反色变为黄色背景）
- 添加粗体和反色
- 与 `applySearchHighlight` 的反色高亮区分

---

## 4. 关键代码路径与文件引用

### 4.1 调用链（搜索功能）

```
use-search-highlight.ts
    ↓
renderToScreen(messageElement, width)
    ↓
scanPositions(screen, query)
    ↓
返回 MatchPosition[]
    ↓
用户导航时：applyPositionedHighlight(screen, stylePool, positions, rowOffset, currentIdx)
```

### 4.2 关键依赖

| 文件 | 用途 |
|------|------|
| `react-reconciler` | React 自定义渲染器 |
| `./dom.js` | DOM 节点创建 |
| `./focus.js` | FocusManager（占位） |
| `./output.js` | 输出缓冲区 |
| `./render-node-to-output.js` | 节点渲染 |
| `./screen.js` | Screen 缓冲区操作 |
| `../utils/debug.js` | 调试日志 |

### 4.3 类型定义

```typescript
export type MatchPosition = {
  row: number      // 相对于消息顶部的行号
  col: number      // 列号
  len: number      // 占用的单元格数（考虑宽字符）
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `lodash-es/noop.js` | 空函数占位 |
| `react-reconciler` | React 协调器 |
| `react-reconciler/constants.js` | LegacyRoot 常量 |

### 5.2 内部模块依赖

```
render-to-screen.ts
├── dom.ts
├── focus.ts
├── output.ts
├── render-node-to-output.ts
├── screen.ts
└── ../utils/debug.ts
```

### 5.3 与主渲染流程的关系

```
┌──────────────────────────────────────────────────────────────┐
│                     主渲染流程                                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │ Ink 实例  │ → │ renderer │ → │ Output   │ → 终端         │
│  └──────────┘    └──────────┘    └──────────┘               │
└──────────────────────────────────────────────────────────────┘
                              ↑
                              │ 独立实例
┌──────────────────────────────────────────────────────────────┐
│                  离屏渲染（render-to-screen）                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │ 独立 root │ → │ 独立     │ → │ 独立     │ → 扫描位置      │
│  │ container │   │ Output   │    │ Screen   │               │
│  └──────────┘    └──────────┘    └──────────┘               │
└──────────────────────────────────────────────────────────────┘
```

**关键区别**：
- 主渲染：持续运行，双缓冲（front/back frame）
- 离屏渲染：按需创建，单缓冲，用完即卸载

---

## 6. 风险、边界与改进建议

### 6.1 已知边界情况

1. **零高度消息**（行 101）
   ```typescript
   Math.max(1, height)  // 避免 0 高度 Screen
   ```

2. **空查询处理**（行 151）
   ```typescript
   if (!lq) return []  // 空查询直接返回空数组
   ```

3. **越界位置**（行 219-222）
   ```typescript
   if (row < 0 || row >= screen.height) return false
   if (col < 0 || col >= screen.width) continue
   ```

4. **flushSyncWork 泄漏**（行 54-56）
   - 注释提到跨 root 的 work 泄漏问题
   - 测量增长率为 ~0.0003ms/call
   - 对 8k 消息全量渲染可能有问题

### 6.2 潜在风险

| 风险 | 严重程度 | 说明 |
|------|----------|------|
| 内存累积 | 中 | pools 不复位，长期运行可能累积大量 interned 字符串 |
| Yoga 节点泄漏 | 低 | root 复用，但子节点是否正确释放？ |
| 并发问题 | 低 | 模块级状态，多线程/异步调用可能冲突 |
| 性能退化 | 中 | 大量消息搜索时，每消息 1-3ms 可能累积 |

### 6.3 改进建议

1. **缓存优化**
   ```typescript
   // 当前：每消息独立渲染
   // 建议：缓存 (msg, width) → (screen, height) 避免重复渲染
   const cache = new LRUCache({ max: 100 })
   ```

2. **增量扫描**
   ```typescript
   // 当前：全量扫描每行
   // 建议：使用 Boyer-Moore 或 Rabin-Karp 加速多模式匹配
   ```

3. **内存管理**
   ```typescript
   // 当前：pools 永不清理
   // 建议：定期重置或添加大小限制
   if (charPool.size > MAX_POOL_SIZE) resetPools()
   ```

4. **错误处理**
   ```typescript
   // 当前：Yoga 节点访问使用 ! 断言
   // 建议：添加防御性检查
   if (!root.yogaNode) throw new Error('Yoga node not initialized')
   ```

5. **性能监控**
   ```typescript
   // 建议：添加更多性能标记
   performance.measure('renderToScreen', 'start', 'end')
   ```

### 6.4 测试建议

- **单元测试**：
  - 各种文本内容的扫描正确性
  - 宽字符、Emoji 的位置计算
  - 空查询、越界索引处理

- **性能测试**：
  - 大消息（1000+ 行）的渲染时间
  - 高频调用（搜索建议）的响应时间
  - 内存使用随时间的变化

- **集成测试**：
  - 与搜索高亮功能的端到端测试
  - 多消息并发渲染的正确性
