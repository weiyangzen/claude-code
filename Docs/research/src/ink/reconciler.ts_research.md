# Research: src/ink/reconciler.ts

## 场景与职责

`reconciler.ts` 是 Ink 的**自定义 React Reconciler**。它桥接 React 的组件树（JSX）与 Ink 自己的宿主层（`DOMElement` / `TextNode`），负责把 React 的虚拟 DOM 操作（创建、更新、删除、移动节点）映射到 Ink 的 DOM 和 Yoga 布局系统。没有它，React 无法"知道"如何渲染到终端。

## 功能点目的

1. **宿主节点生命周期管理**：创建、更新、删除 `ink-box`、`ink-text` 等宿主元素，以及 `#text` 文本节点。
2. **样式与布局同步**：将 `style` prop 同步到 Yoga 节点（`applyStyles`），触发 Flex 布局计算。
3. **事件处理器绑定**：把 `onClick`、`onFocus` 等事件属性注册到 DOM 节点的 `_eventHandlers`。
4. **焦点管理集成**：在节点挂载/卸载时通知 `FocusManager` 处理 `autoFocus` 和焦点恢复。
5. **性能剖析**：提供 Yoga 布局、React commit、渲染等阶段的计时钩子，供 `ink.tsx` 采集帧性能数据。
6. **开发工具支持**：在 `NODE_ENV === 'development'` 时尝试加载 `react-devtools-core`。

## 具体技术实现

### Reconciler 创建

使用 `react-reconciler` 包：

```ts
const reconciler = createReconciler<
  ElementNames,   // Type
  Props,          // Props
  DOMElement,     // Container
  DOMElement,     // Instance
  TextNode,       // TextInstance
  DOMElement,     // SuspenseInstance
  unknown,        // HydratableInstance
  unknown,        // PublicInstance
  DOMElement,     // HostContext
  null,           // UpdatePayload (React 19 直接传 old/new props)
  NodeJS.Timeout, // TimeoutHandle
  -1,             // NoTimeout
  null            // TransitionStatus
>({ ...hostConfig })
```

### 关键 Host Config 方法

#### 上下文与创建

- **`getRootHostContext`**：返回 `{ isInsideText: false }`。
- **`getChildHostContext`**：
  - 当进入/离开 `ink-text` / `ink-virtual-text` / `ink-link` 时切换 `isInsideText`。
  - 用于约束：`<Box>` 不能嵌套在 `<Text>` 内；文本字符串必须出现在 `<Text>` 内。
- **`createInstance(type, props, root, hostContext, internalHandle)`**：
  - 若 `isInsideText && type === 'ink-box'` 则抛出错误。
  - `ink-text` 在文本内部自动降级为 `ink-virtual-text`。
  - 调用 `createNode(type)` 创建 DOM 元素，遍历 `props` 应用属性。
  - 若开启 `CLAUDE_CODE_DEBUG_REPAINTS`，通过 `getOwnerChain(internalHandle)` 记录组件所有者链到 `node.debugOwnerChain`。
- **`createTextInstance(text, root, hostContext)`**：
  - 若不在 `isInsideText` 上下文则抛出错误。
  - 调用 `createTextNode(text)`。

#### 更新与提交

- **`commitUpdate(node, type, oldProps, newProps)`**（React 19 签名）：
  - `diff(oldProps, newProps)` 得到变更的属性；
  - `diff(oldProps.style, newProps.style)` 得到样式差分；
  - 遍历变更属性：
    - `style` → `setStyle` + （若 `yogaNode` 存在）`applyStyles`；
    - `textStyles` → `setTextStyles`；
    - 事件属性 → `setEventHandler`；
    - 其他 → `setAttribute`。
  - 若样式有变更且 `yogaNode` 存在，调用 `applyStyles(yogaNode, styleDiff, fullNewStyle)`。
- **`commitTextUpdate(node, oldText, newText)`**：`setTextNodeValue(node, newText)`。
- **`commitMount(node)`**：`getFocusManager(node).handleAutoFocus(node)`。

#### 可见性

- **`hideInstance(node)`**：`node.isHidden = true` + `yogaNode.setDisplay(None)` + `markDirty(node)`。
- **`unhideInstance(node)`**：`node.isHidden = false` + `yogaNode.setDisplay(Flex)` + `markDirty(node)`。
- **`hideTextInstance(node)`**：`setTextNodeValue(node, '')`。
- **`unhideTextInstance(node, text)`**：`setTextNodeValue(node, text)`。

#### 树操作

- **`appendInitialChild`** / **`appendChild`** / **`appendChildToContainer`**：`appendChildNode`
- **`insertBefore`** / **`insertInContainerBefore`**：`insertBeforeNode`
- **`removeChild(node, removeNode)`**：
  - `removeChildNode(node, removeNode)`
  - `cleanupYogaNode(removeNode)`（释放 Yoga WASM 内存）
  - 若移除的是非文本节点，通知根节点的 `focusManager.handleNodeRemoved`
- **`removeChildFromContainer(node, removeNode)`**：同上，额外调用 `getFocusManager(node).handleNodeRemoved`

#### `resetAfterCommit`

这是每帧 React commit 完成后的回调，Ink 的核心渲染触发点：

```ts
resetAfterCommit(rootNode) {
  // 1. 记录 commit 耗时
  _lastCommitMs = performance.now() - _commitStart

  // 2. 调试日志（CLAUDE_CODE_COMMIT_LOG）
  if (COMMIT_LOG) { ...appendFileSync... }

  // 3. Yoga 布局计算
  if (typeof rootNode.onComputeLayout === 'function') {
    rootNode.onComputeLayout()
  }

  // 4. 测试模式：立即渲染
  if (process.env.NODE_ENV === 'test') {
    rootNode.onImmediateRender?.()
    return
  }

  // 5. 正常模式：触发 onRender（由 ink.tsx 调度实际终端输出）
  rootNode.onRender?.()
}
```

### `diff` 辅助函数

```ts
const diff = (before: AnyObject, after: AnyObject): AnyObject | undefined
```

- 若 `before === after` 直接返回 `undefined`（引用相等优化）。
- 遍历 `before` 的键，检测被删除的键（设为 `undefined`）。
- 遍历 `after` 的键，检测值变化的键。
- 返回变更对象或 `undefined`。

### `cleanupYogaNode`

```ts
const cleanupYogaNode = (node: DOMElement | TextNode): void => {
  const yogaNode = node.yogaNode
  if (yogaNode) {
    yogaNode.unsetMeasureFunc()
    clearYogaNodeReferences(node)  // 防止并发访问已释放内存
    yogaNode.freeRecursive()
  }
}
```

在节点移除时释放 Yoga WASM 对象，避免内存泄漏。

### `getOwnerChain`

```ts
export function getOwnerChain(fiber: unknown): string[]
```

- 遍历 React Fiber 树，从 `internalHandle`（即 Fiber）向上追溯 `_debugOwner` 或 `return`。
- 收集组件 displayName/name，用于 `debugOwnerChain` 调试（如分析哪一组件导致全屏重绘）。

### 性能剖析导出函数

```ts
export function recordYogaMs(ms: number): void
export function getLastYogaMs(): number
export function markCommitStart(): void
export function getLastCommitMs(): number
export function resetProfileCounters(): void
```

- `recordYogaMs` 由 `ink.tsx` 的 `onComputeLayout` 调用；
- `markCommitStart` / `getLastCommitMs` 用于测量 React commit 阶段耗时；
- `resetProfileCounters` 在 `ink.tsx` 每帧渲染后调用清零。

### `dispatcher`

```ts
export const dispatcher = new Dispatcher()
```

Ink 自定义的更新优先级调度器。Reconciler 的 `getCurrentUpdatePriority`、`setCurrentUpdatePriority`、`resolveUpdatePriority`、`resolveEventType`、`resolveEventTimeStamp` 都委托给 `dispatcher`。

在模块末尾：
```ts
dispatcher.discreteUpdates = reconciler.discreteUpdates.bind(reconciler)
```

打破循环依赖：`dispatcher.ts` 不直接导入 `reconciler.ts`。

### React 19 新增方法

文件中实现了 React 19 reconciler 要求的多个新方法：
- `maySuspendCommit`
- `preloadInstance`
- `startSuspendingCommit`
- `suspendInstance`
- `waitForCommitToBeReady`
- `NotPendingTransition`
- `HostTransitionContext`
- `resetFormInstance`
- `requestPostPaintCallback`
- `shouldAttemptEagerTransition`
- `trackSchedulerEvent`
- `resolveEventType`
- `resolveEventTimeStamp`

大多数返回保守值（`false`、`null`、空函数），因为 Ink 不涉及 Suspense、Form、Transition 等浏览器特性。

## 关键代码路径与文件引用

- **调用方**：
  - `src/ink/render-to-screen.ts:8` — 搜索渲染使用 `reconciler.createContainer` + `reconciler.updateContainer`。
  - `src/ink/components/App.tsx:12` — `App` 组件持有 reconciler 容器。
  - `src/ink/components/ScrollBox.tsx:6` — `markCommitStart()` 在滚动事件处理中调用。
  - `src/ink/ink.tsx:29` — 主渲染器导入 reconciler 及多个剖析函数。
- **依赖模块**：
  - `react-reconciler` — 核心 reconciler 工厂；
  - `src/ink/dom.js` — DOM 节点创建与操作；
  - `src/ink/events/dispatcher.js` — 更新优先级调度；
  - `src/ink/events/event-handlers.js` — 事件属性白名单；
  - `src/ink/focus.js` — 焦点管理；
  - `src/ink/layout/node.js` — Yoga 显示模式常量；
  - `src/ink/styles.js` — 样式应用；
  - `src/native-ts/yoga-layout/index.js` — Yoga 计数器（用于性能日志）。

## 依赖与外部交互

- 无直接 I/O。
- `process.env.NODE_ENV === 'development'` 时尝试动态导入 `react-devtools-core`。
- `process.env.CLAUDE_CODE_DEBUG_REPAINTS` 控制是否记录 `debugOwnerChain`。
- `process.env.CLAUDE_CODE_COMMIT_LOG` 控制是否写入同步性能日志文件。

## 风险、边界与改进建议

- **风险**：`react-reconciler` 是 React 内部 API，版本升级时 host config 签名可能变化（如 React 19 已改变 `commitUpdate` 参数）。需要紧跟 React 版本更新适配。
- **边界**：
  - `diff` 函数只做浅层比较，对嵌套对象（如 `style` 中的对象值）可能误判为未变化或过度变化。实践中 `style` 由 `applyStyles` 单独处理，问题不大。
  - `cleanupYogaNode` 的 `freeRecursive()` 是同步释放 WASM 内存，若存在异步访问（如另一帧的 blit 引用）可能崩溃。当前通过 `clearYogaNodeReferences` 提前清空引用来缓解。
  - `getOwnerChain` 依赖 Fiber 的 `_debugOwner`，仅在开发构建中存在；生产构建中回退到 `return`，可能得到更长的宿主元素链。
- **改进建议**：
  1. **类型安全**：`react-reconciler` 的类型定义（`@types/react-reconciler`）常滞后于实际 API，文件中已有多处 `@ts-expect-error`。可考虑维护更精确的本地类型声明。
  2. **内存安全**：对 Yoga 节点释放增加引用计数或延迟释放队列，彻底避免 use-after-free。
  3. `diff` 可扩展为对 `style` 做深度比较，减少不必要的 `applyStyles` 调用。
  4. `resetAfterCommit` 中的同步文件写入（`appendFileSync`）在极端高帧率下可能成为瓶颈，可考虑改为异步批处理或内存缓冲。
  5. 将 React 19 的保守实现方法集中到一个独立对象中，减少主 host config 的噪音。
