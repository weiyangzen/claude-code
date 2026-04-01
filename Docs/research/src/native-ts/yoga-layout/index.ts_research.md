# 研究文档：src/native-ts/yoga-layout/index.ts

## 场景与职责

`index.ts` 是项目内**纯 TypeScript 实现的 yoga-layout 引擎**（Meta 官方 flexbox 引擎的重新实现）。它替代了上游基于 C++/WASM 的 `yoga-layout` 包，为 Ink（React 终端 UI 渲染框架）提供同步、零异步加载、无手动内存管理风险的布局计算能力。

该文件是整个项目布局子系统的**核心计算层**。Ink 的 reconciler 在每次 React commit 阶段通过 `onComputeLayout` 回调触发根节点的 `calculateLayout`，从而驱动整个终端 UI 的盒模型与 flex 布局。

选择自研 TS 实现的主要工程背景：
- **消除 WASM 负担**：避免线性内存增长、WASM 模块预加载、实例重置与交换的复杂生命周期；
- **同步 API**：`loadYoga()` 直接返回已就绪的纯 JS 对象，无需异步等待；
- **可调试与可裁剪**：只实现 Ink 实际使用的 flexbox 子集，同时保留与上游 yoga 的 API 兼容性；
- **性能可控**：针对终端 UI 的高频重绘场景（如虚拟滚动、Spinner、实时日志）做了多层缓存与快速路径优化。

## 功能点目的

| 功能模块 | 目的 |
|----------|------|
| `Node` 类 | 表示布局树中的一个节点，封装样式输入（`Style`）、计算结果（`Layout`）、树关系（`parent`/`children`）以及测量回调（`measureFunc`） |
| `calculateLayout` | 公共入口，重置性能计数器与全局 generation，触发根节点的递归布局，并在最后进行像素网格对齐（`roundLayout`） |
| `layoutNode` | 核心 flexbox 算法实现，覆盖缓存命中、叶子节点测量、容器节点的 flex 分行/伸缩/对齐/定位 |
| `computeFlexBasis` | 为每个 flow child 计算 flex-basis，包含显式 basis、主轴 style dimension、以及基于子树测量的自然尺寸 |
| `resolveFlexibleLengths` | 实现 CSS Flexbox 规范 §9.7 的多轮空间分配算法，处理 flex-grow/shrink 与 min/max 冲突 |
| `layoutAbsoluteChild` | 绝对定位子节点的布局与位置计算，支持基于 inset 的宽高推导以及 justify/align 回退 |
| `roundLayout` / `roundValue` | 像素网格对齐（pixel-grid rounding），确保文本节点在终端列边界上正确放置，避免 off-by-one 错位 |
| 多层缓存系统 | 通过 dirty 标志、generation 门控、单槽/多槽缓存，将虚拟滚动等高频场景下的布局节点访问次数从 10 万级降至万级 |
| 性能计数器 | `getYogaCounters` 提供 `visited`/`measured`/`cacheHits`/`live` 指标，供 Ink 的 reconciler 与 `ink.tsx` 在 `COMMIT_LOG` 模式下输出慢布局诊断 |

## 具体技术实现

### 1. 数据模型

#### `Value` / `Style` / `Layout`

```ts
export type Value = { unit: Unit; value: number }
```

- `Unit` 取 `Undefined`(0)、`Point`(1)、`Percent`(2)、`Auto`(3)。
- `UNDEFINED_VALUE` 与 `AUTO_VALUE` 是全局单例，避免重复分配。

`Style` 类型（行 134-165）包含：
- 方向与 flex 属性：`flexDirection`、`justifyContent`、`alignItems`/`alignSelf`/`alignContent`、`flexWrap`、`flexGrow`、`flexShrink`、`flexBasis`
- 盒模型：`margin`/`padding`/`border`/`position` 均为 9 元素 `Value[]` 数组，按 `Edge` 枚举索引
- 间隙：`gap` 为 3 元素 `Value[]`，按 `Gutter` 枚举索引
- 尺寸约束：`width`/`height`/`minWidth`/`minHeight`/`maxWidth`/`maxHeight`

`Layout` 类型（行 120-129）存储计算后的绝对位置与尺寸，以及已解析为物理边（left/top/right/bottom）的 `border`/`padding`/`margin` 四元数组。

#### `Node` 类（行 403-964）

`Node` 是引擎的中央数据结构，除公共 API（setter/getter/树操作）外，还包含大量**内部快速路径状态与缓存字段**：

| 字段 | 作用 |
|------|------|
| `_flexBasis` / `_mainSize` / `_crossSize` / `_lineIndex` | 每轮布局的临时计算状态 |
| `_hasAutoMargin` / `_hasPosition` / `_hasPadding` / `_hasBorder` / `_hasMargin` | 快速路径标志。在 style setter 中维护，避免布局阶段对空数组做无意义的边缘解析循环 |
| `_lW`/`_lH`/`_lWM`/`_lHM`/`_lOW`/`_lOH`/`_lFW`/`_lFH` + `_lOutW`/`_lOutH` + `_hasL` | **单槽 layout 缓存**：记录上一次 layout pass 的输入与输出 |
| `_mW`/`_mH`/`_mWM`/`_mHM`/`_mOW`/`_mOH` + `_mOutW`/`_mOutH` + `_hasM` | **单槽 measure 缓存**：记录上一次 measure pass 的输入与输出 |
| `_cIn` (Float64Array) / `_cOut` (Float64Array) / `_cN` / `_cWr` / `_cGen` | **多槽缓存（4 slots）**：存储不同输入组合下的 (availableWidth, availableHeight, widthMode, heightMode, ownerWidth, ownerHeight, forceWidth, forceHeight) → (width, height)。解决脏祖先的 measure→layout 级联导致单槽缓存 thrash 的问题 |
| `_fbBasis`/`_fbOwnerW`/`_fbOwnerH`/`_fbAvailMain`/`_fbAvailCross`/`_fbCrossMode`/`_fbGen` | **flex-basis 缓存**：对干净子节点或同 generation 的脏子节点跳过重复 basis 计算。虚拟滚动场景下将 105k 次访问降至约 10k 次 |

### 2. 边缘解析与快速路径

#### 9 边 → 4 物理边

Yoga 支持 9 条逻辑边（Left/Top/Right/Bottom/Start/End/Horizontal/Vertical/All），但布局阶段只需要 4 条物理边。引擎提供两个级别的解析：

- `resolveEdge(edges, physicalEdge, ownerSize, allowAuto)`：通用解析，支持 `Auto` 返回 `NaN`。
- `resolveEdges4Into(edges, ownerSize, out)`：**热路径批量解析**。将共享的回退查找（Horizontal/Vertical/All/Start/End）提升一次，然后分别处理 4 条物理边，避免 12 次独立的 `resolveEdge` 调用。CPU profile 显示这是关键优化。

#### 快速路径标志

`layoutNode` 开头（行 1201-1209）直接检查 `node._hasPadding` / `_hasBorder` / `_hasMargin`：
- 若标志为 false，直接将对应 4 数组写 0；
- 若为 true，才调用 `resolveEdges4Into`。

在 1000 节点基准测试中，约 67% 的调用面对的是全未定义的边缘数组，单分支跳过可节省约 20 次属性读取 + 15 次比较 + 4 次写入。

### 3. 缓存系统（核心优化）

#### Generation 机制

全局变量 `_generation`（行 1039）在每次 `calculateLayout` 调用时自增。缓存条目中记录 `_cGen` 或 `_fbGen`：
- **同 generation 命中**（`_cGen === _generation`）：即使节点 `isDirty_ === true`，缓存也视为新鲜。因为同一次 `calculateLayout` 内部，子树不会在 measure→layout 级联之间发生变化。
- **跨 generation 命中**：要求 `!node.isDirty_`，否则脏节点在上一次布局后可能发生了子树变更，旧缓存失效。

#### 三级缓存检查顺序（`layoutNode` 行 1086-1150）

1. **单槽 layout 缓存 `_hasL`**：输入完全匹配时直接恢复 `_lOutW`/`_lOutH`；
2. **多槽缓存 `_cIn/_cOut`**：扫描最多 4 个 slot，匹配输入则恢复对应输出；
3. **单槽 measure 缓存 `_hasM`**：仅对 `!performLayout` 请求生效。

命中后 `_yogaCacheHits++` 并立即返回，跳过整个子树计算。

#### 缓存写入策略

- `cacheWrite`（行 969-1012）使用 LRU 风格的环形写入（`_cWr % CACHE_SLOTS`）。
- 脏节点在首次写入时会清除旧 generation 的条目（`wasDirty && node._cGen !== _generation`），但同 generation 的条目保留。
- `commitCacheOutputs`（行 1022-1030）在计算完成后将 `layout.width/height` 写入 `_lOutW`/`_lOutH` 或 `_mOutW`/`_mOutH`，解决"scrollbox vpH=33→2624" 的 bug：若只存输入不存输出，缓存命中会返回上一次调用遗留的错位尺寸。

### 4. Flexbox 算法流程（`layoutNode`）

#### 阶段 0：缓存与输入提交（行 1076-1194）

- 检查三级缓存；
- 将当前输入写入对应单槽缓存字段；
- **关键规则**：`performLayout === true` 时才清除 `isDirty_`；measure pass 不清除 dirty。这是为了防止 measure 阶段提前清脏后，layout pass 命中到 stale 的 `_hasL` 缓存，导致 ScrollBox 内容高度不增长。

#### 阶段 1：解析边缘与尺寸（行 1196-1237）

- 解析 padding/border/margin；
- 解析 style width/height，若定义则覆盖 available size 并将 mode 设为 `Exactly`；
- `boundAxis` 应用 min/max 约束。

#### 阶段 2：叶子节点（行 1238-1320）

- **Measure-func 叶子**（文本节点）：计算 inner size（减去 padding+border），调用 `measureFunc`，再将结果加回 padding+border，最后过 `boundAxis`。
- **空叶子**：无 measure func 且无 children，尺寸退化为 `paddingBorderWidth/Height`（若 mode 非 Exactly）。

#### 阶段 3：容器节点 — 收集与分行（行 1322-1408）

- `collectLayoutChildren`：将 children 分为 `flowChildren` 与 `absChildren`，并处理 `Display.None`（递归清零）和 `Display.Contents`（透明提升，子节点直接加入祖父的 flow/abs 列表）。
- 对每个 flow child 调用 `computeFlexBasis`。
- 若 `flexWrap !== NoWrap` 且 `innerMainSize` 有定义，则按 "hypothetical main size"（basis 经 min/max clamp 后）+ margin + gap 进行换行。

#### 阶段 4：每行伸缩与交叉轴测量（行 1410-1543）

- `resolveFlexibleLengths`：按可用主轴空间分配 flex-grow/shrink，处理 min/max 违规的多轮冻结（CSS 规范 §9.7）。
- 对每行每个 child 调用 `layoutNode` 测量交叉轴尺寸。此时 `performLayout` 可能为 false（仅测量）或 true（完整布局）。
- 若容器启用了 baseline 对齐（row 方向），计算每行的 `maxAscent` + `maxDescent`，并可能扩大行高。

#### 阶段 5：确定容器尺寸（行 1547-1583）

- 主轴尺寸：`Exactly` 模式用可用空间；`AtMost` + `Overflow.Scroll` 时 clamp 到可用空间；多行 wrap + `AtMost` 时填满可用空间（因为已在边界换行）；其他情况取内容尺寸。
- 交叉轴尺寸类似，最后再过 `boundAxis`。

#### 阶段 6：定位（`performLayout === true` 时执行，行 1601-1902）

- **Align-content**：按 `alignContent` 值分配行间剩余交叉空间（`FlexStart`/`Center`/`FlexEnd`/`Stretch`/`SpaceBetween`/`SpaceAround`/`SpaceEvenly`）。
- **Re-stretch**：对多行 wrap 或交叉轴非 Exactly 的容器，在已知行高后重新拉伸 `align-self: stretch` 的子节点。
- **Justify-content + auto margins**：计算主轴偏移。若子节点在主轴有 auto margin，则忽略 `justifyContent`，将剩余空间均分给 auto margin。
- **Align-items/align-self + baseline**：计算每个子节点在交叉轴上的位置。baseline 对齐时，位置 = `lineMaxAscent - childBaseline`。
- **Relative position**：解析 `position` 数组中的 left/top/right/bottom，应用相对偏移。
- **Reverse 处理**：`RowReverse`/`ColumnReverse` 在主轴上从尾部向头部排列；`WrapReverse` 在交叉轴上翻转行序与对齐方向。

#### 阶段 7：绝对定位子节点（行 1891-1902）

`layoutAbsoluteChild`（行 1904-2021）处理：
- 百分比尺寸基于 padding box（parent size - border）解析；
- 若 left+right 同时定义而 width 未定义，推导 width；
- 调用 `layoutNode` 计算绝对子节点自身尺寸；
- 位置优先级：显式 inset > 主轴 justify-content / 交叉轴 align-items > 默认边缘。

### 5. 像素网格对齐（`roundLayout`）

终端渲染要求整数行列坐标。`roundLayout`（行 2435-2476）实现与上游 yoga `PixelGrid.cpp` 等价的逻辑：
- 文本节点（`measureFunc !== null`）的 **位置向下取整**（floor），确保折行文本不会超出分配的列；
- 文本节点的 **宽度向上取整**（ceil-if-fractional），避免裁剪最后一个字符；
- 非文本节点使用标准四舍五入（half-up，>=0.5 进位）；
- 宽高通过绝对右下边缘的 rounding 减去绝对左上边缘的 rounding 计算，避免累积漂移。

### 6. 公共 API 与模块导出

文件底部（行 2548-2578）导出与 `yoga-layout/load` 兼容的 API：

```ts
export type Yoga = {
  Config: { create(): Config; destroy(config: Config): void }
  Node: { create(config?: Config): Node; createDefault(): Node; createWithConfig(config: Config): Node; destroy(node: Node): void }
}
export function loadYoga(): Promise<Yoga>
export default YOGA_INSTANCE
```

`YogaLayoutNode` 适配器（`src/ink/layout/yoga.ts`）直接导入 `YOGA_INSTANCE` 的 `Node.create()` 来创建节点。

## 关键代码路径与文件引用

### 依赖关系

| 方向 | 文件 | 说明 |
|------|------|------|
| **依赖** | `src/native-ts/yoga-layout/enums.ts` | 导入全部枚举常量与类型 |
| **被依赖** | `src/ink/layout/yoga.ts` | `YogaLayoutNode` 适配器，将本文件的 `Node` 包装为 Ink 的 `LayoutNode` 接口 |
| **被依赖** | `src/ink/layout/engine.ts` | 工厂函数 `createLayoutNode()` 调用 `createYogaLayoutNode()` |
| **被依赖** | `src/ink/reconciler.ts` | 导入 `getYogaCounters` 用于 `COMMIT_LOG` 慢布局诊断 |
| **被依赖** | `src/ink/ink.tsx` | 在 `onComputeLayout` 中调用 `rootNode.yogaNode.calculateLayout()`；在 `onFrame` 中读取 yoga 性能计数器 |

### 关键调用链（以 Ink 渲染一帧为例）

```
ink.tsx: Ink.onComputeLayout()
  └─> yoga.ts: YogaLayoutNode.calculateLayout(width)
        └─> index.ts: Node.calculateLayout(ownerWidth, ownerHeight, Direction.LTR)
              ├─> 重置 _generation++, _yogaNodesVisited=0 等计数器
              └─> layoutNode(root, ...)
                    ├─> 缓存检查（三级缓存）
                    ├─> resolveEdges4Into (padding/border/margin)
                    ├─> 若为容器：
                    │     ├─> collectLayoutChildren (处理 none/contents/absolute)
                    │     ├─> computeFlexBasis (逐 child，含 basis 缓存)
                    │     ├─> 换行（wrap）
                    │     ├─> resolveFlexibleLengths (多轮 grow/shrink)
                    │     ├─> layoutNode(child, ...) 测量交叉轴
                    │     ├─> 确定容器尺寸
                    │     ├─> 定位（justify/align/auto-margin/baseline/relative position）
                    │     └─> layoutAbsoluteChild
                    ├─> 若为叶子：measureFunc 或零尺寸
                    └─> cacheWrite + commitCacheOutputs
        └─> roundLayout(root, pointScaleFactor, 0, 0)
```

## 风险、边界与改进建议

### 风险

1. **缓存逻辑的高度复杂性**
   - 文件包含单槽缓存、多槽缓存、flex-basis 缓存、generation 门控、dirty 标志交互，注释中多次提到修复特定 bug（如 "scrollbox vpH=33→2624 bug"、"long-continuous blank-screen bug"、"sticky-scroll never follows new content"）。
   - 这些缓存规则是逐步补丁累积的结果，任何对 `layoutNode` 入口或 `isDirty_` 清除时机的修改都可能复现历史 bug。

2. **RTL 未实现**
   - 文件明确声明 "RTL direction (Ink always passes Direction.LTR)"。`resolveEdge` 中 `Start`/`End` 被硬编码映射为 `Left`/`Right`。若未来 Ink 需要支持从右到左语言，需要大规模修改轴方向、边缘解析与绝对定位逻辑。

3. **box-sizing: content-box 未实现**
   - `Node.setBoxSizing` 是空函数。若 Ink 组件层误用 `content-box`，行为将不符合预期。

4. **硬编码裸数字与枚举的耦合**
   - 快速路径中大量使用 `v.unit === 0`（Undefined）、`v.unit === 3`（Auto）等裸数字。虽然性能更优，但增加了与 `enums.ts` 的隐式耦合风险。

5. **绝对定位的 margin 处理与 Yoga 上游的 errata 差异**
   - `Errata` 枚举已导出（如 `AbsolutePositionWithoutInsetsExcludesPadding`、`AbsolutePercentAgainstInnerSize`），但 `index.ts` 中 `layoutAbsoluteChild` 的实现是固定行为（百分比基于 padding box），没有根据 `config.errata` 动态切换。这意味着与上游 yoga 的某些兼容性模式不完全对齐。

### 边界

- **aspect-ratio 未实现**：文件中有 `setAspectRatio` 空 stub。
- **多行 flex 的 align-content 默认值**：`defaultStyle()` 中 `alignContent` 为 `FlexStart`，与 CSS 规范的 `Stretch` 不同，但这是 yoga 的传统行为。
- **MeasureMode 的语义**：`AtMost` 在引擎内部被解释为 "fit-content"（非硬 clamp），只有 `Overflow.Scroll` 才会在 `AtMost` 下强制 clamp。这与 CSS 的常规行为一致，但初学者容易误解。

### 改进建议

1. **增加缓存不变量的单元测试/属性测试**
   - 当前仓库内未找到针对 yoga-layout 的独立测试文件。建议补充：
     - 虚拟滚动场景（大量干净节点 + 少量脏节点）的缓存命中率断言；
     - `display: contents` 嵌套场景的布局等价性断言；
     - 同 generation 多次 `calculateLayout` 不泄露 stale 尺寸的断言。

2. **将裸数字快速路径改为局部常量别名**
   - 例如在每个热路径函数顶部声明 `const UNIT_UNDEFINED = Unit.Undefined`，编译器通常会内联优化，既保留性能又提升可读性与安全性。

3. **抽离 flexbox 算法为更小的内部模块**
   - 当前文件长达 2578 行，包含 Node 类、布局算法、缓存、像素对齐、工厂函数。可考虑按职责拆分为：
     - `node.ts`（Node 类与公共 API）
     - `layout.ts`（`layoutNode` 与核心算法）
     - `cache.ts`（缓存读写与 generation 管理）
     - `round.ts`（像素网格对齐）
   - 拆分后可显著降低维护者的心智负担，并便于单元测试。

4. **补齐 `setBoxSizing` 或增加运行时警告**
   - 若 `content-box` 确实不在产品路线图中，建议在 `setBoxSizing(ContentBox)` 时抛出或 warn，避免静默错误。

5. **性能计数器扩展**
   - 当前 `getYogaCounters` 只返回总量。可考虑增加 `cacheMisses` 或按层级（单槽/多槽/basis）拆分的命中统计，帮助更精准地定位性能退化。
