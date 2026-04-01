# TreeSelect.tsx 深度研究文档

## 场景与职责

TreeSelect 是一个通用的树形选择组件，用于在终端 UI 中展示和选择层级结构的数据。它基于 Ink 框架构建，支持展开/折叠、键盘导航、焦点管理等功能。

**核心使用场景：**
- 日志选择器中的会话分组展示（`src/components/LogSelector.tsx`）
- 需要展示层级关系的数据选择（如文件树、分类目录等）
- 支持父子节点关系的交互式树形列表

**设计目标：**
- 将树形数据扁平化渲染，便于在终端中展示
- 支持键盘导航（方向键展开/折叠、选择）
- 灵活的展开状态管理（受控/非受控）
- 可自定义节点前缀显示

---

## 功能点目的

### 1. 树形数据扁平化
- 将嵌套的 `TreeNode<T>[]` 结构转换为 `FlattenedNode<T>[]`
- 根据展开状态动态决定哪些子节点可见
- 记录节点深度，用于缩进显示

### 2. 展开/折叠状态管理
- 支持内部状态管理（`internalExpandedIds` Set）
- 支持受控模式（通过 `isNodeExpanded`、`onExpand`、`onCollapse` props）
- 优先使用外部控制，否则使用内部状态

### 3. 键盘导航支持
- **→ (Right)**: 展开当前节点（如果有子节点）
- **← (Left)**: 折叠当前节点，或在已折叠时跳转到父节点
- 与底层 Select 组件集成，支持上下导航

### 4. 焦点管理
- 支持通过 `focusNodeId` 指定当前焦点节点
- 焦点变化时触发 `onFocus` 回调
- 处理程序化焦点（避免循环触发）

### 5. 自定义前缀显示
- `getParentPrefix`: 自定义父节点前缀（默认 ▼/▶）
- `getChildPrefix`: 自定义子节点前缀（默认 "  ▸ "）

---

## 具体技术实现

### 关键数据结构

```typescript
// 树节点定义
export type TreeNode<T> = {
  id: string | number;           // 唯一标识
  value: T;                      // 节点值（泛型）
  label: string;                 // 显示标签
  description?: string;          // 描述文本
  dimDescription?: boolean;      // 是否淡化描述
  children?: TreeNode<T>[];      // 子节点数组
  metadata?: Record<string, unknown>;
};

// 扁平化节点（内部使用）
type FlattenedNode<T> = {
  node: TreeNode<T>;
  depth: number;                 // 节点深度（0 为根）
  isExpanded: boolean;           // 是否展开
  hasChildren: boolean;          // 是否有子节点
  parentId?: string | number;    // 父节点 ID
};
```

### 核心 Props 接口

```typescript
export type TreeSelectProps<T> = {
  nodes: TreeNode<T>[];                    // 树节点数据
  onSelect: (node: TreeNode<T>) => void;   // 选择回调
  onCancel?: () => void;                   // 取消回调
  onFocus?: (node: TreeNode<T>) => void;   // 焦点变化回调
  focusNodeId?: string | number;           // 当前焦点节点 ID
  visibleOptionCount?: number;             // 可见选项数
  layout?: 'compact' | 'expanded' | 'compact-vertical';
  isDisabled?: boolean;                    // 是否禁用
  hideIndexes?: boolean;                   // 是否隐藏索引
  isNodeExpanded?: (nodeId) => boolean;    // 受控：是否展开
  onExpand?: (nodeId) => void;             // 展开回调
  onCollapse?: (nodeId) => void;           // 折叠回调
  getParentPrefix?: (isExpanded) => string;  // 父节点前缀
  getChildPrefix?: (depth) => string;        // 子节点前缀
  onUpFromFirstItem?: () => void;          // 从首项向上回调
};
```

### 核心流程

**1. 展开状态判断（第 143-156 行）**
```javascript
const isExpanded = (nodeId) => {
  if (isNodeExpanded) {
    return isNodeExpanded(nodeId);  // 优先使用外部控制
  }
  return internalExpandedIds.has(nodeId);  // 否则使用内部状态
};
```

**2. 树形数据扁平化（第 157-185 行）**
```javascript
function traverse(node, depth, parentId) {
  const hasChildren = !!node.children && node.children.length > 0;
  const nodeIsExpanded = isExpanded(node.id);
  result.push({ node, depth, isExpanded: nodeIsExpanded, hasChildren, parentId });
  
  // 递归处理展开的子节点
  if (hasChildren && nodeIsExpanded && node.children) {
    for (const child of node.children) {
      traverse(child, depth + 1, node.id);
    }
  }
}
```
- 使用深度优先遍历
- 仅展开当前展开的节点的子节点
- 记录每个节点的深度和父节点 ID

**3. 标签构建（第 190-209 行）**
```javascript
const buildLabel = (flatNode) => {
  let prefix = "";
  if (flatNode.hasChildren) {
    prefix = parentPrefixFn(flatNode.isExpanded);  // ▼ 或 ▶
  } else if (flatNode.depth > 0) {
    prefix = childPrefixFn(flatNode.depth);        // "  ▸ "
  }
  return prefix + flatNode.node.label;
};
```

**4. 选项转换（第 210-224 行）**
```javascript
const options = flattenedNodes.map(flatNode => ({
  label: buildLabel(flatNode),
  description: flatNode.node.description,
  dimDescription: flatNode.node.dimDescription ?? true,
  value: flatNode.node.id
}));
```
- 转换为 Select 组件需要的选项格式
- 使用节点 ID 作为选项值

**5. 节点映射构建（第 225-234 行）**
```javascript
const nodeMap = new Map();
flattenedNodes.forEach(fn => map.set(fn.node.id, fn.node));
```
- 用于通过 ID 快速查找节点
- 在选择和焦点回调中使用

**6. 展开/折叠切换（第 244-276 行）**
```javascript
const toggleExpand = (nodeId, shouldExpand) => {
  const flatNode = findFlattenedNode(nodeId);
  if (!flatNode || !flatNode.hasChildren) return;
  
  if (shouldExpand) {
    if (onExpand) onExpand(nodeId);
    else setInternalExpandedIds(prev => new Set(prev).add(nodeId));
  } else {
    if (onCollapse) onCollapse(nodeId);
    else setInternalExpandedIds(prev => { newSet.delete(nodeId); return newSet; });
  }
};
```

**7. 键盘事件处理（第 277-321 行）**
```javascript
const handleKeyDown = (e) => {
  if (e.key === "right" && flatNode.hasChildren) {
    e.preventDefault();
    toggleExpand(focusNodeId, true);  // → 展开
  } else if (e.key === "left") {
    if (flatNode.hasChildren && flatNode.isExpanded) {
      toggleExpand(focusNodeId, false);  // ← 折叠
    } else if (flatNode.parentId !== undefined) {
      // 已折叠，跳转到父节点
      toggleExpand(flatNode.parentId, false);
      if (onFocus) onFocus(parentNode);
    }
  }
};
```

**8. 焦点处理（第 339-362 行）**
```javascript
const handleFocus = (nodeId) => {
  if (isProgrammaticFocusRef.current) {
    isProgrammaticFocusRef.current = false;
    return;  // 忽略程序化焦点
  }
  if (lastFocusedIdRef.current === nodeId) return;  // 去重
  
  lastFocusedIdRef.current = nodeId;
  if (onFocus) onFocus(nodeMap.get(nodeId));
};
```

---

## 关键代码路径与文件引用

### 内部依赖
| 路径 | 用途 |
|------|------|
| `react/compiler-runtime` | React Compiler 缓存机制 |
| `react` | 核心 React API |
| `../../ink/events/keyboard-event.js` | KeyboardEvent 类型 |
| `../../ink.js` | Box 组件 |
| `../CustomSelect/select.js` | Select 组件及 OptionWithDescription 类型 |

### 外部调用方
| 路径 | 使用场景 |
|------|----------|
| `src/components/LogSelector.tsx` | 会话日志的树形选择（第 32 行导入，1357-1380 行使用） |

### LogSelector.tsx 使用示例
```typescript
import { type TreeNode, TreeSelect } from './ui/TreeSelect.js';

// 构建树节点
const treeNodes: LogTreeNode[] = Array.from(sessionGroups.entries()).map(([sessionId, groupLogs]) => {
  const latestLog = groupLogs[0];
  const children = groupLogs.slice(1).map((log, index) => ({
    id: `log:${sessionId}:${index + 1}`,
    value: { log, indexInFiltered },
    label: buildLogLabel(log, maxLabelWidth, { isChild: true }),
    description: childMetadata,
  }));
  
  return {
    id: `group:${sessionId}`,
    value: { log: latestLog, indexInFiltered },
    label: buildLogLabel(latestLog, maxLabelWidth, { isGroupHeader: true, forkCount }),
    description: parentMetadata,
    children,  // 子节点（分叉会话）
  };
});

// 渲染 TreeSelect
<TreeSelect
  nodes={treeNodes}
  onSelect={node => onSelect(node.value.log)}
  onFocus={handleTreeSelectFocus}
  onCancel={onCancel}
  focusNodeId={focusedNode?.id}
  visibleOptionCount={visibleCount}
  layout="expanded"
  isDisabled={viewMode === "search"}
  hideIndexes={false}
  isNodeExpanded={nodeId => {
    const sessionId = typeof nodeId === "string" && nodeId.startsWith("group:") 
      ? nodeId.substring(6) 
      : null;
    return sessionId ? expandedGroupSessionIds.has(sessionId) : false;
  }}
  onExpand={nodeId => {
    const sessionId = nodeId.startsWith("group:") ? nodeId.substring(6) : null;
    if (sessionId) {
      setExpandedGroupSessionIds(prev => new Set(prev).add(sessionId));
    }
  }}
  onCollapse={nodeId => {
    const sessionId = nodeId.startsWith("group:") ? nodeId.substring(6) : null;
    if (sessionId) {
      setExpandedGroupSessionIds(prev => {
        const newSet = new Set(prev);
        newSet.delete(sessionId);
        return newSet;
      });
    }
  }}
  onUpFromFirstItem={enterSearchMode}
/>
```

---

## 依赖与外部交互

### 与 Select 组件的关系

TreeSelect 是对 Select 组件的包装：
- 将树形数据转换为 Select 可接受的扁平选项
- 添加键盘事件处理（左右方向键）
- 使用 Box 包裹 Select，设置 `tabIndex={0}` 和 `autoFocus` 以接收键盘事件

```
TreeSelect
  ├── 数据转换：TreeNode[] → FlattenedNode[] → OptionWithDescription[]
  ├── 键盘处理：handleKeyDown（左右方向键）
  └── Box (tabIndex=0, autoFocus, onKeyDown)
        └── Select (处理上下导航、选择等)
```

### 状态管理策略

| 模式 | 触发条件 | 行为 |
|------|----------|------|
| 受控模式 | 提供 `isNodeExpanded` | 完全由父组件控制展开状态 |
| 非受控模式 | 未提供 `isNodeExpanded` | 使用内部 `internalExpandedIds` Set |

展开/折叠回调：
- 提供 `onExpand`/`onCollapse`：调用回调，不修改内部状态
- 未提供回调：修改 `internalExpandedIds`

### React Compiler 缓存

组件使用 48 个缓存槽位（`$[0]` 到 `$[47]`），主要缓存：
- 内部状态（expandedIds, refs）
- 计算结果（flattenedNodes, options, nodeMap）
- 回调函数（isExpanded, buildLabel, toggleExpand, handleKeyDown 等）
- 渲染结果（Select 组件、Box 包装器）

---

## 风险、边界与改进建议

### 已知限制

1. **性能考虑**
   - 每次展开/折叠都会重新遍历整个树
   - 大数据量时可能影响性能
   - 建议：实现虚拟滚动或延迟加载

2. **键盘导航限制**
   - 左右方向键仅处理展开/折叠
   - 不支持快速跳转到同级节点
   - 不支持搜索/过滤功能

3. **深度限制**
   - 理论上支持无限嵌套
   - 但终端宽度有限，过深的层级会导致缩进过多
   - 建议：控制树深度在 5 层以内

### 边界情况

| 场景 | 行为 |
|------|------|
| 空 nodes 数组 | 渲染空的 Select |
| 循环引用 | 可能导致无限递归（需确保数据无循环） |
| 重复 ID | 后出现的节点会覆盖先出现的（Map set 行为） |
| 禁用状态 | 忽略键盘事件，但视觉反馈由 Select 处理 |
| 无焦点节点 | handleKeyDown 直接返回，不处理 |

### 改进建议

1. **性能优化**
   ```typescript
   // 使用 useMemo 缓存扁平化结果
   const flattenedNodes = useMemo(() => {
     // 遍历逻辑
   }, [nodes, isExpanded]);
   
   // 或使用虚拟滚动处理大数据量
   ```

2. **增强键盘导航**
   - 添加 Home/End 键支持（跳转到首/尾节点）
   - 添加 PageUp/PageDown 支持
   - 添加字母键快速定位

3. **搜索/过滤支持**
   ```typescript
   type TreeSelectProps<T> = {
     // ... existing props
     filter?: (node: TreeNode<T>, query: string) => boolean;
     searchQuery?: string;
   };
   ```

4. **多选支持**
   ```typescript
   type TreeSelectProps<T> = {
     // ... existing props
     multiSelect?: boolean;
     selectedIds?: (string | number)[];
     onSelectionChange?: (selectedNodes: TreeNode<T>[]) => void;
   };
   ```

5. **异步加载子节点**
   ```typescript
   type TreeSelectProps<T> = {
     // ... existing props
     loadChildren?: (nodeId: string | number) => Promise<TreeNode<T>[]>;
     onLazyExpand?: (nodeId: string | number) => void;
   };
   ```

6. **动画支持**
   - 展开/折叠时添加过渡动画
   - 使用 Ink 的动画钩子实现平滑效果

7. **无障碍增强**
   ```tsx
   <Box 
     tabIndex={0} 
     autoFocus 
     onKeyDown={handleKeyDown}
     role="tree"
     aria-label="Tree selection"
   >
     <Select 
       // ...
       role="group"
     />
   </Box>
   ```

8. **类型安全增强**
   - 当前 `TreeNode<T>` 的 `id` 可以是 string 或 number
   - 建议统一为 string，避免类型混淆

---

## 源码映射说明

文件包含 base64 编码的 source map，指向原始 TypeScript 源码 `TreeSelect.tsx`。调试时可使用 source map 还原原始代码位置。

原始源码关键部分（从 source map 还原）：
```typescript
import React from 'react'
import type { KeyboardEvent } from '../../ink/events/keyboard-event.js'
import { Box } from '../../ink.js'
import { type OptionWithDescription, Select } from '../CustomSelect/select.js'

export type TreeNode<T> = {
  id: string | number
  value: T
  label: string
  description?: string
  dimDescription?: boolean
  children?: TreeNode<T>[]
  metadata?: Record<string, unknown>
}

export function TreeSelect<T>({
  nodes,
  onSelect,
  onCancel,
  onFocus,
  focusNodeId,
  visibleOptionCount,
  layout = 'expanded',
  isDisabled = false,
  hideIndexes = false,
  isNodeExpanded,
  onExpand,
  onCollapse,
  getParentPrefix,
  getChildPrefix,
  onUpFromFirstItem,
}: TreeSelectProps<T>) {
  // ... implementation
}
```
