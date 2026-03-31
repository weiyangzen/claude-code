# usePagination.ts 研究文档

## 场景与职责

`usePagination.ts` 是 Claude Code 插件管理系统的分页/滚动钩子，为长列表提供虚拟滚动和分页功能。该钩子的核心职责包括：

1. **滚动窗口管理**：根据选中项自动计算可见窗口的起始/结束索引
2. **平滑滚动**：确保选中项始终在可见区域内，自动调整滚动偏移
3. **分页兼容性**：提供基于页码的 API，同时内部使用连续滚动
4. **列表切片**：提供 `getVisibleItems` 函数获取当前可见的数据切片

该钩子用于 `ManagePlugins.tsx` 中的统一列表视图，处理插件和 MCP 服务器的混合列表。

## 功能点目的

### 1. 连续滚动模式
- 基于选中项索引自动计算滚动偏移
- 选中项超出可见区域时自动滚动
- 使用 `useRef` 跟踪上一次的滚动位置，实现平滑过渡

### 2. 分页 API 兼容
- 提供 `currentPage`, `totalPages`, `goToPage`, `nextPage`, `prevPage` 等分页接口
- 内部实现为无操作或简单计算，保持 API 兼容性
- 便于未来扩展为真正的分页模式

### 3. 可见性计算
- `getVisibleItems`: 从完整列表中提取可见部分
- `toActualIndex`: 将可见列表中的索引转换为原始列表索引
- `isOnCurrentPage`: 检查原始索引是否在可见区域内

### 4. 滚动位置信息
- `scrollPosition`: 提供当前滚动位置的元数据
- `canScrollUp`/`canScrollDown`: 指示是否可以向上/向下滚动

## 具体技术实现

### 关键数据结构

```typescript
// 配置选项
interface UsePaginationOptions {
  totalItems: number;           // 总项目数
  maxVisible?: number;          // 最大可见项目数（默认 5）
  selectedIndex?: number;       // 当前选中索引（默认 0）
}

// 返回结果
interface UsePaginationResult<T> {
  // 分页兼容字段
  currentPage: number;          // 当前页码（从 0 开始）
  totalPages: number;           // 总页数
  
  // 滚动窗口字段
  startIndex: number;           // 可见窗口起始索引
  endIndex: number;             // 可见窗口结束索引
  needsPagination: boolean;     // 是否需要分页/滚动
  pageSize: number;             // 每页/窗口大小
  
  // 列表操作函数
  getVisibleItems: (items: T[]) => T[];           // 获取可见切片
  toActualIndex: (visibleIndex: number) => number; // 可见索引转实际索引
  isOnCurrentPage: (actualIndex: number) => boolean; // 检查索引可见性
  
  // 分页导航（兼容 API）
  goToPage: (page: number) => void;    // 跳转到指定页（无操作）
  nextPage: () => void;                // 下一页（无操作）
  prevPage: () => void;                // 上一页（无操作）
  
  // 选择处理
  handleSelectionChange: (newIndex: number, setSelectedIndex: (index: number) => void) => void;
  handlePageNavigation: (direction: 'left' | 'right', setSelectedIndex: (index: number) => void) => boolean;
  
  // 滚动位置信息
  scrollPosition: {
    current: number;              // 当前选中位置（1-based）
    total: number;                // 总数
    canScrollUp: boolean;         // 是否可以向上滚动
    canScrollDown: boolean;       // 是否可以向下滚动
  };
}
```

### 核心算法

#### 滚动偏移计算

```typescript
const scrollOffset = useMemo(() => {
  if (!needsPagination) return 0;

  const prevOffset = scrollOffsetRef.current;

  // 选中项在可见窗口上方，向上滚动
  if (selectedIndex < prevOffset) {
    scrollOffsetRef.current = selectedIndex;
    return selectedIndex;
  }

  // 选中项在可见窗口下方，向下滚动
  if (selectedIndex >= prevOffset + maxVisible) {
    const newOffset = selectedIndex - maxVisible + 1;
    scrollOffsetRef.current = newOffset;
    return newOffset;
  }

  // 选中项在可见窗口内，保持当前偏移
  const maxOffset = Math.max(0, totalItems - maxVisible);
  const clampedOffset = Math.min(prevOffset, maxOffset);
  scrollOffsetRef.current = clampedOffset;
  return clampedOffset;
}, [selectedIndex, maxVisible, needsPagination, totalItems]);
```

#### 可见列表提取

```typescript
const startIndex = scrollOffset;
const endIndex = Math.min(scrollOffset + maxVisible, totalItems);

const getVisibleItems = useCallback(
  (items: T[]): T[] => {
    if (!needsPagination) return items;
    return items.slice(startIndex, endIndex);
  },
  [needsPagination, startIndex, endIndex]
);
```

#### 选择变更处理

```typescript
const handleSelectionChange = useCallback(
  (newIndex: number, setSelectedIndex: (index: number) => void) => {
    const clampedIndex = Math.max(0, Math.min(newIndex, totalItems - 1));
    setSelectedIndex(clampedIndex);
  },
  [totalItems]
);
```

### 使用示例

```typescript
function MyList({ items }: { items: string[] }) {
  const [selectedIndex, setSelectedIndex] = useState(0);
  
  const pagination = usePagination<string>({
    totalItems: items.length,
    maxVisible: 8,
    selectedIndex
  });
  
  const visibleItems = pagination.getVisibleItems(items);
  
  return (
    <Box>
      {visibleItems.map((item, idx) => {
        const actualIdx = pagination.toActualIndex(idx);
        const isSelected = actualIdx === selectedIndex;
        return <Text key={actualIdx} bold={isSelected}>{item}</Text>;
      })}
      
      {pagination.needsPagination && (
        <Text dimColor>
          Showing {pagination.startIndex + 1}-{pagination.endIndex} of {items.length}
        </Text>
      )}
    </Box>
  );
}
```

## 关键代码路径与文件引用

### 直接依赖

| 依赖 | 用途 |
|-----|------|
| `react` | `useCallback`, `useMemo`, `useRef` |

### 调用方

| 文件路径 | 用途 |
|---------|------|
| `src/commands/plugin/ManagePlugins.tsx` | 统一列表分页/滚动 |

### 使用流程

```
ManagePlugins.tsx
    │
    ├── const pagination = usePagination<UnifiedInstalledItem>({
    │       totalItems: filteredItems.length,
    │       selectedIndex,
    │       maxVisible: 8
    │     });
    │
    ├── const visibleItems = pagination.getVisibleItems(filteredItems);
    │
    ├── 渲染 visibleItems
    │
    └── 键盘导航
            ├── pagination.handleSelectionChange(selectedIndex - 1, setSelectedIndex)
            └── pagination.handleSelectionChange(selectedIndex + 1, setSelectedIndex)
```

## 依赖与外部交互

### React Hooks 使用

```
usePagination
    │
    ├── useRef(0) → scrollOffsetRef
    │     └── 跟踪上一次的滚动偏移
    │
    ├── useMemo → scrollOffset
    │     └── 根据 selectedIndex 计算新的偏移
    │
    ├── useMemo → totalPages, currentPage
    │     └── 分页兼容计算
    │
    ├── useCallback → getVisibleItems
    │     └── 列表切片函数
    │
    ├── useCallback → toActualIndex
    │     └── 索引转换函数
    │
    ├── useCallback → isOnCurrentPage
    │     └── 可见性检查函数
    │
    ├── useCallback → goToPage, nextPage, prevPage
    │     └── 无操作函数（API 兼容）
    │
    ├── useCallback → handleSelectionChange
    │     └── 边界检查后的选择更新
    │
    └── useCallback → handlePageNavigation
          └── 返回 false（表示使用连续滚动）
```

### 与 ManagePlugins.tsx 的集成

```typescript
// ManagePlugins.tsx 中的使用
const pagination = usePagination<UnifiedInstalledItem>({
  totalItems: filteredItems.length,
  selectedIndex,
  maxVisible: 8
});

// 键盘导航
useKeybindings({
  'select:previous': () => {
    if (selectedIndex === 0) {
      setIsSearchMode(true);
    } else {
      pagination.handleSelectionChange(selectedIndex - 1, setSelectedIndex);
    }
  },
  'select:next': () => {
    if (selectedIndex < filteredItems.length - 1) {
      pagination.handleSelectionChange(selectedIndex + 1, setSelectedIndex);
    }
  }
}, { context: 'Select', isActive: viewState === 'plugin-list' && !isSearchMode });
```

## 风险、边界与改进建议

### 潜在风险

1. **Ref 同步问题**
   - `scrollOffsetRef` 在 `useMemo` 中更新
   - 如果 React 的并发特性启用，可能出现不一致
   - **缓解措施**：当前 React 版本下安全，未来可能需要使用 `useLayoutEffect`

2. **边界计算错误**
   - `endIndex` 使用 `Math.min(scrollOffset + maxVisible, totalItems)`
   - 当 `totalItems` 为 0 时，`endIndex` 为 0，但 `startIndex` 也可能为 0
   - `slice(0, 0)` 返回空数组，正确行为

3. **性能问题**
   - 每次 `selectedIndex` 变化都重新计算所有派生值
   - 在超长列表中可能影响性能
   - **缓解措施**：使用 `useMemo` 缓存计算结果

### 边界情况

| 场景 | 当前行为 | 备注 |
|-----|---------|------|
| `totalItems = 0` | `needsPagination = false` | 正确处理空列表 |
| `totalItems <= maxVisible` | `needsPagination = false` | 显示全部 |
| `selectedIndex < 0` | `handleSelectionChange` 钳制到 0 | 边界保护 |
| `selectedIndex >= totalItems` | `handleSelectionChange` 钳制到最后 | 边界保护 |
| `selectedIndex` 快速变化 | 平滑滚动 | 使用 ref 跟踪状态 |
| `maxVisible = 0` | `needsPagination = true`，但窗口大小为 0 | 应避免此输入 |

### 改进建议

1. **虚拟滚动优化**
   ```typescript
   // 建议：支持固定高度的虚拟滚动
   interface UsePaginationOptions {
     // ...
     itemHeight?: number;           // 每项高度
     containerHeight?: number;      // 容器高度
     overscan?: number;             // 额外渲染项数
   }
   ```

2. **分页模式切换**
   ```typescript
   // 建议：支持真正的分页模式
   interface UsePaginationOptions {
     // ...
     mode?: 'continuous' | 'paged';  // 连续滚动 vs 分页
   }
   ```

3. **滚动动画**
   ```typescript
   // 建议：添加平滑滚动动画
   const scrollOffset = useSpring(calculateOffset(selectedIndex), {
     stiffness: 300,
     damping: 30
   });
   ```

4. **滚动位置持久化**
   ```typescript
   // 建议：支持恢复滚动位置
   interface UsePaginationOptions {
     // ...
     scrollKey?: string;            // 用于 localStorage 存储
   }
   ```

5. **动态可见数量**
   ```typescript
   // 建议：根据容器高度动态计算
   const useResponsivePagination = <T>(options: Omit<UsePaginationOptions, 'maxVisible'>) => {
     const containerRef = useRef<HTMLElement>(null);
     const itemHeight = 24; // 假设每项高度
     
     const maxVisible = containerRef.current 
       ? Math.floor(containerRef.current.clientHeight / itemHeight)
       : 5;
     
     return usePagination<T>({ ...options, maxVisible });
   };
   ```

6. **键盘导航增强**
   ```typescript
   // 建议：添加 page up/down 支持
   const handlePageUp = () => {
     const newIndex = Math.max(0, selectedIndex - maxVisible);
     handleSelectionChange(newIndex, setSelectedIndex);
   };
   
   const handlePageDown = () => {
     const newIndex = Math.min(totalItems - 1, selectedIndex + maxVisible);
     handleSelectionChange(newIndex, setSelectedIndex);
   };
   ```

7. **TypeScript 增强**
   ```typescript
   // 建议：更精确的类型
   type UsePaginationReturn<T> = {
     // ...
     visibleRange: { start: number; end: number };  // 替代 startIndex/endIndex
     range: { start: number; end: number };          // 别名
   };
   ```

8. **测试工具**
   ```typescript
   // 建议：提供测试辅助函数
   export const calculateScrollOffset = (
     selectedIndex: number,
     prevOffset: number,
     maxVisible: number,
     totalItems: number
   ): number => {
     // 提取纯函数用于测试
   };
   ```

### 测试建议

1. **单元测试**：
   ```typescript
   describe('usePagination', () => {
     test('returns all items when total <= maxVisible', () => {
       const { result } = renderHook(() => 
         usePagination({ totalItems: 3, maxVisible: 5 })
       );
       expect(result.current.needsPagination).toBe(false);
       expect(result.current.getVisibleItems([1, 2, 3])).toEqual([1, 2, 3]);
     });

     test('scrolls up when selected goes above window', () => {
       const { result, rerender } = renderHook(
         ({ selectedIndex }) => usePagination({ 
           totalItems: 10, 
           maxVisible: 5, 
           selectedIndex 
         }),
         { initialProps: { selectedIndex: 5 } }
       );
       
       rerender({ selectedIndex: 2 });
       
       expect(result.current.startIndex).toBe(2);
     });
   });
   ```

2. **边界测试**：
   - 空列表
   - 单一项
   - 选中第一项/最后一项
   - 快速连续选择变更

3. **性能测试**：
   - 超长列表（10000+ 项）
   - 快速滚动性能
   - 内存使用

4. **集成测试**：
   - 与 `ManagePlugins.tsx` 的集成
   - 键盘导航行为
   - 搜索过滤后的滚动行为
