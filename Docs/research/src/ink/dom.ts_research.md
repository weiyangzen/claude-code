# dom.ts 研究文档

## 场景与职责

`dom.ts` 实现 Ink 的 DOM-like 节点系统，是连接 React Reconciler 和 Yoga 布局引擎的核心层。它提供了类似浏览器 DOM 的节点操作 API，同时集成布局计算和渲染优化。

### 核心职责

1. **节点创建与管理**：创建和管理 DOMElement、TextNode
2. **Yoga 布局集成**：自动创建/管理 Yoga 节点，处理布局计算
3. **树操作**：appendChild、insertBefore、removeChild 等 DOM 操作
4. **脏标记系统**：跟踪节点变化，触发重新渲染
5. **样式管理**：设置和比较样式，避免不必要的布局重算
6. **调试支持**：组件栈跟踪，用于性能分析和错误定位

## 功能点目的

### 1. 节点类型系统

**ElementNames**：
- `ink-root`：根节点，持有 FocusManager
- `ink-box`：布局容器（Flexbox）
- `ink-text`：文本节点，支持测量
- `ink-virtual-text`：虚拟文本，无 Yoga 节点
- `ink-link`：超链接节点
- `ink-progress`：进度条
- `ink-raw-ansi`：原始 ANSI 文本

**TextNode**：
- `nodeName: '#text'`
- 纯文本内容，无 Yoga 节点

### 2. Yoga 布局集成

自动 Yoga 节点管理：
- 创建 DOMElement 时按需创建 Yoga 节点
- `ink-virtual-text`、`ink-link`、`ink-progress` 无 Yoga 节点
- `ink-text` 和 `ink-raw-ansi` 设置测量函数

### 3. 脏标记系统

**markDirty**：
- 标记节点及其所有祖先为脏
- 文本节点触发 Yoga 的 markDirty 重新测量
- 控制重新渲染的范围

**scheduleRenderFrom**：
- 从任意节点向上查找到根
- 触发根的 onRender 回调（节流的 scheduleRender）
- 用于 DOM 级变更（如 scrollTop 变化）

### 4. 滚动支持

DOMElement 支持完整的滚动状态：
- `scrollTop`：当前滚动位置
- `pendingScrollDelta`：待处理的滚动增量
- `scrollClampMin/Max`：滚动边界（虚拟滚动用）
- `scrollHeight/viewportHeight`：内容/视口高度
- `stickyScroll`：自动跟随底部
- `scrollAnchor`：元素锚定滚动

### 5. 缓存管理

**collectRemovedRects**：
- 收集被移除子树的缓存矩形
- 处理绝对定位节点的特殊清除逻辑
- 与 `node-cache.ts` 协作

## 具体技术实现

### 核心数据结构

```typescript
export type DOMElement = {
  nodeName: ElementNames
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  textStyles?: TextStyles
  
  // 生命周期回调
  onComputeLayout?: () => void
  onRender?: () => void
  onImmediateRender?: () => void
  
  // 脏标记和可见性
  dirty: boolean
  isHidden?: boolean
  hasRenderedContent?: boolean
  
  // 事件处理（单独存储避免脏标记）
  _eventHandlers?: Record<string, unknown>
  
  // 滚动状态
  scrollTop?: number
  pendingScrollDelta?: number
  scrollClampMin?: number
  scrollClampMax?: number
  scrollHeight?: number
  scrollViewportHeight?: number
  scrollViewportTop?: number
  stickyScroll?: boolean
  scrollAnchor?: { el: DOMElement; offset: number }
  
  // Focus 管理
  focusManager?: FocusManager
  
  // 调试
  debugOwnerChain?: string[]
} & InkNode

type InkNode = {
  parentNode: DOMElement | undefined
  yogaNode?: LayoutNode
  style: Styles
}
```

### 节点创建

```typescript
export const createNode = (nodeName: ElementNames): DOMElement => {
  // 判断是否需要 Yoga 节点
  const needsYogaNode =
    nodeName !== 'ink-virtual-text' &&
    nodeName !== 'ink-link' &&
    nodeName !== 'ink-progress'
  
  const node: DOMElement = {
    nodeName,
    style: {},
    attributes: {},
    childNodes: [],
    parentNode: undefined,
    yogaNode: needsYogaNode ? createLayoutNode() : undefined,
    dirty: false,
  }
  
  // 文本节点设置测量函数
  if (nodeName === 'ink-text') {
    node.yogaNode?.setMeasureFunc(measureTextNode.bind(null, node))
  } else if (nodeName === 'ink-raw-ansi') {
    node.yogaNode?.setMeasureFunc(measureRawAnsiNode.bind(null, node))
  }
  
  return node
}
```

### 树操作

**appendChildNode**：
```typescript
export const appendChildNode = (
  node: DOMElement,
  childNode: DOMElement,
): void => {
  // 如果已有父节点，先移除
  if (childNode.parentNode) {
    removeChildNode(childNode.parentNode, childNode)
  }
  
  childNode.parentNode = node
  node.childNodes.push(childNode)
  
  // 同步到 Yoga 树
  if (childNode.yogaNode) {
    node.yogaNode?.insertChild(
      childNode.yogaNode,
      node.yogaNode.getChildCount(),
    )
  }
  
  markDirty(node)
}
```

**insertBeforeNode**：
- 处理 Yoga 索引计算（考虑无 Yoga 节点的子节点）
- DOM 索引和 Yoga 索引可能不一致

**removeChildNode**：
- 从 Yoga 树移除
- 收集缓存矩形用于清除
- 断开父节点引用

### 文本测量

```typescript
const measureTextNode = function (
  node: DOMNode,
  width: number,
  widthMode: LayoutMeasureMode,
): { width: number; height: number } {
  const rawText =
    node.nodeName === '#text' ? node.nodeValue : squashTextNodes(node)
  
  // Tab 展开
  const text = expandTabs(rawText)
  
  const dimensions = measureText(text, width)
  
  // 处理各种边界情况
  // ...
  
  // 文本换行处理
  const textWrap = node.style?.textWrap ?? 'wrap'
  const wrappedText = wrapText(text, width, textWrap)
  
  return measureText(wrappedText, width)
}
```

### 样式比较

```typescript
function shallowEqual<T extends object>(
  a: T | undefined,
  b: T | undefined,
): boolean {
  // 快速路径：相同引用
  if (a === b) return true
  if (a === undefined || b === undefined) return false
  
  // 键数量比较
  const aKeys = Object.keys(a) as (keyof T)[]
  const bKeys = Object.keys(b) as (keyof T)[]
  if (aKeys.length !== bKeys.length) return false
  
  // 逐属性比较
  for (const key of aKeys) {
    if (a[key] !== b[key]) return false
  }
  
  return true
}
```

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/dom.ts`
- **导出类型**：`DOMElement`、`TextNode`、`DOMNode`、`ElementNames` 等
- **导出函数**：
  - `createNode`、`createTextNode`
  - `appendChildNode`、`insertBeforeNode`、`removeChildNode`
  - `setAttribute`、`setStyle`、`setTextStyles`
  - `markDirty`、`scheduleRenderFrom`
  - `setTextNodeValue`
  - `clearYogaNodeReferences`
  - `findOwnerChainAtRow`

### 依赖关系

**被导入**：
- `./focus.js` - FocusManager 类型
- `./layout/engine.js` - createLayoutNode
- `./layout/node.js` - LayoutNode 类型和常量
- `./measure-text.js` - measureText
- `./node-cache.js` - addPendingClear、nodeCache
- `./squash-text-nodes.js` - squashTextNodes
- `./styles.js` - Styles、TextStyles
- `./tabstops.js` - expandTabs
- `./wrap-text.js` - wrapText

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `createNode` | 创建 DOMElement | 110-132 |
| `createTextNode` | 创建文本节点 | 318-330 |
| `appendChildNode` | 添加子节点 | 134-153 |
| `insertBeforeNode` | 在指定位置插入 | 155-202 |
| `removeChildNode` | 移除子节点 | 204-223 |
| `setAttribute` | 设置属性 | 247-264 |
| `setStyle` | 设置样式 | 266-274 |
| `setTextStyles` | 设置文本样式 | 276-289 |
| `markDirty` | 标记脏节点 | 393-413 |
| `scheduleRenderFrom` | 从节点调度渲染 | 419-423 |
| `measureTextNode` | 文本测量 | 332-374 |
| `findOwnerChainAtRow` | 查找行对应的组件链 | 465-484 |

### 相关文件

- `src/ink/layout/engine.ts` - Yoga 布局引擎
- `src/ink/layout/node.ts` - LayoutNode 接口
- `src/ink/focus.ts` - FocusManager
- `src/ink/node-cache.ts` - 节点缓存
- `src/ink/measure-text.ts` - 文本测量
- `src/ink/squash-text-nodes.ts` - 文本节点扁平化
- `src/ink/wrap-text.ts` - 文本换行
- `src/ink/tabstops.ts` - Tab 展开
- `src/ink/reconciler.ts` - React Reconciler 集成

## 依赖与外部交互

### React Reconciler 集成

DOM 操作由 Reconciler 调用：
```
Reconciler
    ↓
createInstance → createNode
appendChild → appendChildNode
removeChild → removeChildNode
insertBefore → insertBeforeNode
```

### Yoga 布局引擎

双向同步：
```
DOM 树操作
    ↓
同步到 Yoga 树（insertChild/removeChild）
    ↓
Yoga calculateLayout
    ↓
读取 computed layout 用于渲染
```

### 渲染管线

```
markDirty
    ↓
scheduleRenderFrom（可选）
    ↓
Renderer 检测 dirty 节点
    ↓
重新布局和渲染
```

## 风险、边界与改进建议

### 已知风险

1. **Yoga 节点泄漏**：
   - 需要显式调用 `clearYogaNodeReferences` 释放
   - 忘记调用可能导致内存泄漏

2. **索引计算复杂性**：
   - DOM 索引和 Yoga 索引不一致（某些子节点无 Yoga 节点）
   - `insertBeforeNode` 中的索引计算容易出错

3. **循环引用**：
   - `parentNode` 和 `childNodes` 形成循环引用
   - 需要正确处理以避免内存泄漏

4. **样式对象比较**：
   - `shallowEqual` 只进行浅比较
   - 嵌套对象变化可能检测不到

### 边界情况

1. **空文本节点**：`setTextNodeValue` 处理非字符串输入
2. **重复添加**：`appendChildNode` 自动处理已有父节点的情况
3. **无效插入**：`insertBeforeNode` 找不到参考节点时退化为 append
4. **绝对定位移除**：`collectRemovedRects` 特殊处理绝对定位节点

### 改进建议

1. **内存管理**：
   - 添加 Yoga 节点引用计数
   - 自动清理未引用的节点
   
   ```typescript
   // 示例：引用计数
   class YogaNodeRef {
     private count = 0
     addRef() { this.count++ }
     release() { 
       if (--this.count === 0) this.node.free()
     }
   }
   ```

2. **类型安全**：
   - 使用更严格的类型区分不同 ElementNames
   - 添加运行时类型守卫

3. **性能优化**：
   - 批量 DOM 操作（类似 React 的 batching）
   - 延迟 Yoga 树更新

4. **调试工具**：
   - 添加 DOM 树可视化
   - 记录 dirty 标记的传播路径

5. **测试覆盖**：
   - Yoga 索引计算的单元测试
   - 内存泄漏检测测试
   - 边界情况（空节点、循环引用）测试
