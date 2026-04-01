# horizontalScroll.ts 研究文档

## 场景与职责

`horizontalScroll.ts` 是 Claude Code CLI 的**水平滚动窗口计算工具**，专门用于处理终端界面中水平空间不足时的元素显示逻辑。该模块的核心应用场景是**BackgroundTaskStatus 组件**中的任务标签（pills）横向展示，当终端宽度不足以显示所有任务标签时，提供智能的滚动窗口计算能力。

### 核心使用场景
- **任务状态栏横向滚动**：在 `BackgroundTaskStatus.tsx` 中，当多个后台任务同时运行时，任务标签可能超出终端可用宽度
- **边缘滚动策略**：确保当前选中的任务标签始终可见，采用边缘滚动而非中心对齐的交互模式
- **箭头指示器控制**：根据滚动位置动态显示左右箭头，提示用户还有更多内容

---

## 功能点目的

### 1. 水平滚动窗口计算 (`calculateHorizontalScrollWindow`)

**设计目标**：在有限宽度内最大化显示元素，同时确保选中项始终可见。

**关键特性**：
- **边缘滚动策略**：选中项到达边缘时才触发滚动，而非始终居中显示
- **累积宽度计算**：使用前缀和数组优化范围宽度查询，时间复杂度 O(n)
- **分隔符处理**：智能处理第一个元素的分隔符（`firstItemHasSeparator` 参数）
- **箭头宽度预留**：根据是否显示左右箭头动态调整可用宽度

### 2. 数据结构定义 (`HorizontalScrollWindow`)

```typescript
export type HorizontalScrollWindow = {
  startIndex: number      // 可见窗口起始索引（包含）
  endIndex: number        // 可见窗口结束索引（不包含）
  showLeftArrow: boolean  // 是否显示左箭头
  showRightArrow: boolean // 是否显示右箭头
}
```

---

## 具体技术实现

### 关键算法流程

#### 1. 累积宽度数组构建
```typescript
const cumulativeWidths: number[] = [0]
for (let i = 0; i < totalItems; i++) {
  cumulativeWidths.push(cumulativeWidths[i]! + itemWidths[i]!)
}
```
- **目的**：支持 O(1) 时间复杂度计算任意区间 `[start, end)` 的宽度
- **空间复杂度**：O(n)，额外存储 n+1 个累积值

#### 2. 范围宽度计算（含分隔符处理）
```typescript
function rangeWidth(start: number, end: number): number {
  const baseWidth = cumulativeWidths[end]! - cumulativeWidths[start]!
  // 当起始位置 > 0 且首项包含分隔符时，减去分隔符宽度
  if (firstItemHasSeparator && start > 0) {
    return baseWidth - 1  // 分隔符宽度为 1
  }
  return baseWidth
}
```

#### 3. 边缘滚动逻辑

**所有元素可见的快捷路径**：
```typescript
if (totalWidth <= availableWidth) {
  return { startIndex: 0, endIndex: totalItems, showLeftArrow: false, showRightArrow: false }
}
```

**选中项在右侧时的滚动策略**：
```typescript
if (clampedSelected >= endIndex) {
  // 将选中项固定在右边缘
  endIndex = clampedSelected + 1
  startIndex = clampedSelected
  // 尽可能向左扩展窗口
  while (startIndex > 0 && rangeWidth(startIndex - 1, endIndex) <= getEffectiveWidth(startIndex - 1, endIndex)) {
    startIndex--
  }
}
```

**选中项在左侧时的滚动策略**：
```typescript
else {
  // 将选中项固定在左边缘
  startIndex = clampedSelected
  endIndex = clampedSelected + 1
  // 尽可能向右扩展窗口
  while (endIndex < totalItems && rangeWidth(startIndex, endIndex + 1) <= getEffectiveWidth(startIndex, endIndex + 1)) {
    endIndex++
  }
}
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `itemWidths` | `number[]` | 每个元素的宽度数组（包含分隔符） |
| `availableWidth` | `number` | 可用显示宽度 |
| `arrowWidth` | `number` | 箭头指示器宽度（含间距） |
| `selectedIdx` | `number` | 当前选中项索引 |
| `firstItemHasSeparator` | `boolean` | 首项宽度是否包含前置分隔符 |

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/horizontalScroll.ts`
- **导出函数**：`calculateHorizontalScrollWindow`
- **导出类型**：`HorizontalScrollWindow`

### 调用方
| 文件路径 | 使用场景 |
|---------|---------|
| `src/components/tasks/BackgroundTaskStatus.tsx` | 任务标签横向滚动，行 118 |

### 调用示例（BackgroundTaskStatus.tsx）
```typescript
const { startIndex, endIndex, showLeftArrow, showRightArrow } = calculateHorizontalScrollWindow(
  pillWidths,           // 各标签宽度数组
  availableWidth,       // 终端可用宽度（columns - 20 - 4）
  2,                    // 箭头宽度
  selectedIdx >= 0 ? selectedIdx : 0  // 当前选中索引
)
```

---

## 依赖与外部交互

### 无外部依赖
该模块为**纯算法工具**，不依赖任何外部模块：
- 无 npm 包依赖
- 无项目内部模块导入
- 仅使用 TypeScript/JavaScript 原生 API

### 被依赖关系
- **BackgroundTaskStatus.tsx**：通过 `import { calculateHorizontalScrollWindow } from 'src/utils/horizontalScroll.js'` 引入

---

## 风险、边界与改进建议

### 边界情况处理

| 场景 | 处理逻辑 |
|------|---------|
| 空数组 (`totalItems === 0`) | 返回 `{ startIndex: 0, endIndex: 0, showLeftArrow: false, showRightArrow: false }` |
| 选中索引越界 | 使用 `Math.max(0, Math.min(selectedIdx, totalItems - 1))` 钳制到有效范围 |
| 所有元素可见 | 直接返回全范围，不显示箭头 |
| 单个元素选中 | 窗口收缩到 `[selected, selected+1)`，然后尽可能扩展 |

### 潜在风险

1. **宽度计算精度问题**
   - 分隔符宽度硬编码为 1，假设为单字符宽度
   - 如果分隔符包含多字节字符（如 emoji），计算可能不准确
   - **建议**：将分隔符宽度作为参数传入，而非硬编码

2. **性能考虑**
   - 累积宽度数组构建为 O(n)，对于大量元素（>1000）可能影响性能
   - 当前使用场景（任务标签）元素数量有限，不构成问题
   - **建议**：对于潜在大数据量场景，考虑使用线段树或树状数组优化

3. **边缘滚动 vs 中心滚动的用户体验**
   - 当前采用边缘滚动策略，选中项固定在边缘
   - 某些用户可能更习惯中心对齐的滚动方式
   - **建议**：考虑添加配置选项支持不同滚动策略

### 改进建议

1. **添加单元测试覆盖**
   - 边界条件测试（空数组、单元素、越界索引）
   - 各种宽度组合的场景测试
   - 分隔符处理逻辑验证

2. **性能优化（大数据量场景）**
   ```typescript
   // 当元素数量超过阈值时，使用二分查找优化窗口定位
   if (totalItems > THRESHOLD) {
     // 使用二分查找替代线性扫描
   }
   ```

3. **API 增强**
   - 支持动画过渡参数（如 `animationDuration`）
   - 添加 `getVisibleItems()` 辅助函数直接返回可见元素索引
   - 支持垂直滚动场景的复用（抽象为通用滚动窗口计算）

4. **类型安全增强**
   - 添加品牌类型（branded types）区分原始宽度和累积宽度
   - 使用更精确的类型约束确保 `startIndex < endIndex`
