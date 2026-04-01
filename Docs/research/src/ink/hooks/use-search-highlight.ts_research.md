# use-search-highlight.ts 深入研究

## 场景与职责

`useSearchHighlight` 是 Ink 终端 UI 框架中用于设置搜索高亮的 Hook。它允许组件在终端屏幕上高亮显示匹配的文本，支持全屏模式下的文本搜索功能。

## 功能点目的

### 1. 搜索查询设置
- 设置搜索高亮查询字符串
- 非空查询会在下一帧反转所有可见匹配项（SGR 7）
- 空查询清除高亮

### 2. 元素扫描
- 扫描 DOM 子树获取匹配位置
- 将主树中的元素渲染到独立的 Screen 缓冲区
- 返回元素相对位置的匹配结果

### 3. 位置高亮
- 基于位置的高亮（当前匹配项）
- 支持滚动偏移跟踪
- 黄色高亮当前选中的匹配项

### 4. 屏幕空间高亮
- 匹配渲染后的文本而非源消息文本
- 适用于任何可见内容（bash 输出、文件路径、错误消息）
- 被截断/省略的内容不会高亮

## 具体技术实现

### 接口定义

```typescript
export function useSearchHighlight(): {
  setQuery: (query: string) => void
  scanElement: (el: DOMElement) => MatchPosition[]
  setPositions: (
    state: {
      positions: MatchPosition[]
      rowOffset: number
      currentIdx: number
    } | null,
  ) => void
}
```

### MatchPosition 类型

```typescript
// src/ink/render-to-screen.ts
export type MatchPosition = {
  row: number      // 相对于消息顶部的行号
  col: number      // 列号
  len: number      // 匹配的单元格数（考虑宽字符）
}
```

### 核心实现逻辑

1. **获取 Ink 实例**：
   ```typescript
   useContext(StdinContext) // 锚定到 App 子树
   const ink = instances.get(process.stdout)
   ```

2. **方法绑定**：
   ```typescript
   return useMemo(() => {
     if (!ink) {
       return {
         setQuery: () => {},
         scanElement: () => [],
         setPositions: () => {},
       }
     }
     return {
       setQuery: (query: string) => ink.setSearchHighlight(query),
       scanElement: (el: DOMElement) => ink.scanElementSubtree(el),
       setPositions: state => ink.setSearchPositions(state),
     }
   }, [ink])
   ```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/components/StdinContext.ts` | 用于锚定到 App 子树 |
| `src/ink/instances.ts` | Ink 实例映射表 |
| `src/ink/render-to-screen.ts` | 定义 MatchPosition 类型和扫描逻辑 |
| `src/ink/dom.ts` | 定义 DOMElement 类型 |

### Ink 实例中的搜索高亮实现

```typescript
// ink.tsx 中的相关代码
private searchHighlightQuery = '';
private searchPositions: {
  positions: MatchPosition[];
  rowOffset: number;
  currentIdx: number;
} | null = null;

setSearchHighlight(query: string) {
  this.searchHighlightQuery = query;
  this.scheduleRender();
}

setSearchPositions(state: typeof this.searchPositions) {
  this.searchPositions = state;
  this.scheduleRender();
}

scanElementSubtree(el: DOMElement): MatchPosition[] {
  // 渲染元素到独立 Screen
  const { screen } = renderToScreen(elementToReactNode(el), this.terminalColumns);
  // 扫描匹配位置
  return scanPositions(screen, this.searchHighlightQuery);
}
```

### 渲染时应用高亮

```typescript
// 在 onRender 中
if (this.searchHighlightQuery) {
  applySearchHighlight(this.backFrame.screen, this.stylePool, this.searchHighlightQuery);
}

if (this.searchPositions) {
  applyPositionedHighlight(
    this.backFrame.screen,
    this.stylePool,
    this.searchPositions.positions,
    this.searchPositions.rowOffset,
    this.searchPositions.currentIdx
  );
}
```

### applySearchHighlight 实现

```typescript
// src/ink/searchHighlight.ts
export function applySearchHighlight(
  screen: Screen,
  stylePool: StylePool,
  query: string,
): void {
  if (!query) return;
  const lq = query.toLowerCase();
  // 扫描每一行，查找匹配并应用反转样式
  // ...
}
```

### renderToScreen 实现

```typescript
// src/ink/render-to-screen.ts
export function renderToScreen(
  el: ReactElement,
  width: number,
): { screen: Screen; height: number } {
  // 使用独立的 root/container/pools
  // 渲染 React 元素到 Screen 缓冲区
  // 用于搜索：渲染单个消息，扫描查询位置
  // 每次调用约 1-3ms
}
```

## 依赖与外部交互

### 与 Ink 实例的交互

1. **实例查找**：
   - 通过 `process.stdout` 从 `instances` Map 获取 Ink 实例
   - 每个进程通常只有一个 Ink 实例

2. **方法委托**：
   - Hook 的方法直接委托给 Ink 实例的对应方法
   - 触发重新渲染以应用高亮

### 与渲染流程的交互

1. **查询设置**：
   - 设置查询后调用 `scheduleRender()`
   - 下一帧渲染时应用高亮

2. **位置高亮**：
   - 位置是消息相对的（row 0 = 消息顶部）
   - 渲染时添加 `rowOffset`（消息的屏幕顶部偏移）

### 扫描流程

```
组件调用 scanElement(el)
    ↓
renderToScreen: 渲染元素到独立 Screen
    ↓
scanPositions: 扫描 Screen 缓冲区
    ↓
返回 MatchPosition[]
```

## 风险、边界与改进建议

### 潜在风险

1. **性能问题**：
   - `scanElement` 每次调用都执行完整渲染（1-3ms）
   - 频繁扫描可能导致性能问题
   - 需要上游缓存（按消息、查询、宽度缓存）

2. **内存泄漏**：
   - `renderToScreen` 使用的 pools 是全局缓存的
   - 长期运行可能累积内存

3. **实例依赖**：
   - 依赖 `process.stdout` 获取实例
   - 非标准场景（如测试）可能无法工作

### 边界情况

1. **实例不存在**：
   - 返回空操作函数
   - 不会抛出错误

2. **空查询**：
   - 清除高亮
   - 扫描返回空数组

3. **被截断内容**：
   - 源文本中有但渲染后被截断的内容不会匹配
   - 这是设计行为（"我们高亮你看到的"）

4. **宽字符**：
   - `len` 计算考虑宽字符
   - 与查询字符串长度可能不同

### 改进建议

1. **添加缓存**：
   ```typescript
   // 在 Hook 级别添加缓存
   const scanCache = useMemo(() => new Map(), []);
   const scanElement = useCallback((el: DOMElement) => {
     const key = getElementKey(el);
     if (scanCache.has(key)) return scanCache.get(key);
     const result = ink.scanElementSubtree(el);
     scanCache.set(key, result);
     return result;
   }, [ink]);
   ```

2. **防抖查询**：
   ```typescript
   const setQuery = useCallback(
     debounce((query: string) => ink.setSearchHighlight(query), 100),
     [ink]
   );
   ```

3. **支持正则表达式**：
   ```typescript
   setQuery({ pattern: /error.*/i, isRegex: true })
   ```

4. **高亮样式定制**：
   ```typescript
   setQuery(query, { 
     highlightStyle: 'inverse',  // 或 'yellow', 'underline' 等
   })
   ```

5. **添加搜索状态**：
   ```typescript
   const { 
     setQuery, 
     scanElement, 
     setPositions,
     matchCount,    // 当前匹配数
     currentMatch,  // 当前选中匹配
   } = useSearchHighlight();
   ```

### 测试建议

1. 测试空查询清除高亮
2. 测试宽字符匹配（CJK、emoji）
3. 测试被截断内容的匹配行为
4. 测试多次快速设置查询
5. 测试组件卸载时的清理
6. 测试无 Ink 实例时的行为
