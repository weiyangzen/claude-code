# screen.ts 研究文档

## 场景与职责

`screen.ts` 是 Ink 终端渲染引擎的核心底层模块，负责管理终端屏幕缓冲区的内存表示。它是整个渲染流水线的基础层，连接着 Yoga 布局引擎的输出与最终的终端输出。

**核心职责：**
1. **屏幕缓冲区管理** - 使用紧凑的 TypedArray 存储终端单元格数据，避免 GC 压力
2. **字符池化 (CharPool)** - 字符串驻留(interning)机制，共享字符 ID 跨屏幕
3. **样式池化 (StylePool)** - ANSI 样式编码与缓存，支持样式转换和特效叠加
4. **超链接池化 (HyperlinkPool)** - OSC 8 超链接字符串驻留
5. **双宽字符支持** - 处理 CJK、Emoji 等双宽字符的单元格分配
6. **屏幕差异计算 (diff)** - 高效比较两帧屏幕状态，生成最小更新集
7. **区域操作** - 支持 blit(块拷贝)、clear(清除)、shift(滚动)等批量操作
8. **选择高亮支持** - 通过样式叠加实现文本选择视觉效果

**在架构中的位置：**
- 被 `renderer.ts`、`output.ts`、`render-to-screen.ts` 直接调用
- 被 `selection.ts`、`searchHighlight.ts` 用于视觉反馈
- 被 `ink.tsx` 主控制器管理生命周期

---

## 功能点目的

### 1. 内存高效的单元格存储

**目的：** 避免每帧分配大量 Cell 对象导致的 GC 压力

**实现方式：**
- 每个单元格使用 2 个 Int32 存储（64位打包）
  - word0: charId (32位) - 指向 CharPool 的索引
  - word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
- 使用 BigInt64Array 进行批量填充操作
- 共享 CharPool/HyperlinkPool 跨屏幕，支持 blit 直接拷贝 ID

### 2. 字符池 (CharPool)

**目的：** 减少重复字符串的内存占用，加速字符比较

**关键特性：**
- ASCII 快速路径：使用 Int32Array 直接映射 charCode → index
- 非 ASCII 字符使用 Map 进行驻留
- 预定义索引 0 = 空格, 1 = 空字符串(spacer)

### 3. 样式池 (StylePool)

**目的：** 管理 ANSI 样式编码，支持样式转换和特效叠加

**核心功能：**
- 样式驻留：将 AnsiCode[] 数组编码为唯一 ID
- 位编码技巧：ID 的 bit 0 表示"在空格上可见"（背景色、反色、下划线等）
- 过渡缓存：预计算样式间转换的 ANSI 字符串
- 特效叠加：
  - `withInverse()` - 反色效果（用于搜索高亮）
  - `withCurrentMatch()` - 当前匹配项高亮（黄底+粗体+下划线）
  - `withSelectionBg()` - 选择背景色（主题可配置）

### 4. 双宽字符支持 (CellWidth)

**目的：** 正确处理 CJK、Emoji 等占用 2 个终端列的字符

**单元格宽度类型：**
```typescript
enum CellWidth {
  Narrow = 0,      // 普通字符，宽度 1
  Wide = 1,        // 双宽字符头部，包含实际字符
  SpacerTail = 2,  // 双宽字符尾部占位，不渲染
  SpacerHead = 3,  // 软换行时的头部占位
}
```

**处理逻辑：**
- `setCellAt()` 自动为 Wide 字符创建 SpacerTail
- 覆盖 Wide 字符时自动清理残留的 SpacerTail
- 边界检查防止孤儿单元格

### 5. 屏幕差异计算 (diff/diffEach)

**目的：** 生成最小终端更新指令，减少输出带宽

**优化策略：**
- 利用 damage 矩形限制扫描范围
- 相同宽度屏幕使用快速路径
- 使用 `findNextDiff()` 跳过连续相同单元格
- 重用 Cell 对象避免分配

### 6. 区域操作

**blitRegion** - 块拷贝：
- 支持跨屏幕拷贝（利用共享 Pool）
- 快速路径：整行连续内存拷贝
- 自动处理右边缘的 SpacerTail

**clearRegion** - 区域清除：
- 使用 BigInt64Array.fill() 批量清除
- 边界清理防止孤儿 Wide 字符

**shiftRows** - 行滚动：
- 模拟终端的 SU/SD 控制序列
- 支持 cells64、noSelect、softWrap 的同步移动

---

## 具体技术实现

### 打包单元格数据

```typescript
// word1 位布局
const STYLE_SHIFT = 17      // styleId 占用 15 位 (0-32767)
const HYPERLINK_SHIFT = 2   // hyperlinkId 占用 15 位
const HYPERLINK_MASK = 0x7fff
const WIDTH_MASK = 3        // width 占用 2 位

function packWord1(styleId: number, hyperlinkId: number, width: number): number {
  return (styleId << STYLE_SHIFT) | (hyperlinkId << HYPERLINK_SHIFT) | width
}
```

### 样式 ID 编码

```typescript
// Bit 0 标记是否在空格上可见
id = (rawId << 1) | (hasVisibleSpaceEffect ? 1 : 0)

// 解码时右移 1 位
style = styles[id >>> 1]
```

### 屏幕差异算法

```typescript
function diffEach(prev, next, callback) {
  // 1. 确定扫描区域（damage 矩形并集）
  const region = unionRect(prev.damage, next.damage)
  
  // 2. 处理尺寸变化（高度/宽度缩小）
  if (prevHeight > nextHeight) { /* 添加删除区域 */ }
  if (prevWidth > nextWidth) { /* 添加删除区域 */ }
  
  // 3. 按行扫描差异
  for (each row in region) {
    // 使用 findNextDiff 快速跳过相同单元格
    skip = findNextDiff(prevCells, nextCells, ci, endX - x)
    
    // 对差异单元格调用回调
    callback(x, y, removedCell, addedCell)
  }
}
```

### 双宽字符处理

```typescript
function setCellAt(screen, x, y, cell) {
  // 1. 清理被覆盖的 Wide 字符的 SpacerTail
  if (prevWidth === CellWidth.Wide && cell.width !== Wide) {
    clearSpacerAt(x + 1)
  }
  
  // 2. 清理被覆盖的 SpacerTail 对应的 Wide 字符
  if (prevWidth === CellWidth.SpacerTail && cell.width !== SpacerTail) {
    clearWideCharAt(x - 1)
  }
  
  // 3. 写入新单元格
  cells[ci] = internChar(cell.char)
  cells[ci + 1] = packWord1(styleId, hyperlinkId, cell.width)
  
  // 4. 为 Wide 字符创建 SpacerTail
  if (cell.width === CellWidth.Wide && x + 1 < width) {
    createSpacerAt(x + 1)
  }
}
```

---

## 关键代码路径与文件引用

### 核心数据结构

| 类型 | 位置 | 说明 |
|------|------|------|
| `Screen` | line 366-415 | 屏幕缓冲区接口 |
| `Cell` | line 308-313 | 单元格视图类型 |
| `CellWidth` | line 289-300 | 单元格宽度枚举 |
| `CharPool` | line 21-53 | 字符驻留池 |
| `StylePool` | line 112-260 | 样式驻留池 |
| `HyperlinkPool` | line 57-75 | 超链接驻留池 |

### 关键函数

| 函数 | 位置 | 用途 |
|------|------|------|
| `createScreen()` | line 451-492 | 创建屏幕缓冲区 |
| `resetScreen()` | line 501-544 | 重置屏幕（双缓冲复用）|
| `setCellAt()` | line 693-810 | 设置单元格（含双宽处理）|
| `cellAt()` / `cellAtIndex()` | line 593-613 | 读取单元格 |
| `blitRegion()` | line 858-952 | 块拷贝区域 |
| `clearRegion()` | line 959-1048 | 清除区域 |
| `shiftRows()` | line 1057-1092 | 行滚动 |
| `diffEach()` | line 1156-1206 | 差异计算 |
| `markNoSelectRegion()` | line 1471-1486 | 标记不可选择区域 |

### 样式特效函数

| 函数 | 位置 | 用途 |
|------|------|------|
| `withInverse()` | line 170-180 | 反色效果 |
| `withCurrentMatch()` | line 190-220 | 当前匹配高亮 |
| `withSelectionBg()` | line 244-259 | 选择背景色 |
| `setSelectionBg()` | line 239-243 | 设置选择背景色 |

---

## 依赖与外部交互

### 导入依赖

```typescript
// 外部库
import { type AnsiCode, ansiCodesToString, diffAnsiCodes } from '@alcalzone/ansi-tokenize'

// 内部模块
import { type Point, type Rectangle, type Size, unionRect } from './layout/geometry.js'
import { BEL, ESC, SEP } from './termio/ansi.js'
import * as warn from './warn.js'
```

### 被调用方

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `ink.tsx` | Screen, CellWidth, CharPool, createScreen, migrateScreenPools, StylePool | 主渲染循环、双缓冲管理 |
| `renderer.ts` | createScreen, StylePool | 帧渲染器创建 |
| `output.ts` | blitRegion, CellWidth, extractHyperlinkFromStyles, filterOutHyperlinkStyles, markNoSelectRegion, OSC8_PREFIX, resetScreen, type Screen, type StylePool, setCellAt, shiftRows | 输出操作队列执行 |
| `render-to-screen.ts` | CellWidth, CharPool, cellAtIndex, createScreen, HyperlinkPool, type Screen, StylePool, setCellStyleId | 离屏渲染 |
| `selection.ts` | CellWidth, cellAt, cellAtIndex, type Screen, type StylePool, setCellStyleId | 选择高亮应用 |
| `searchHighlight.ts` | CellWidth, cellAtIndex, type Screen, type StylePool, setCellStyleId | 搜索高亮应用 |
| `log-update.ts` | diffEach | 终端更新生成 |

---

## 风险、边界与改进建议

### 已知风险

1. **Pool 内存增长**
   - CharPool 和 HyperlinkPool 随时间累积，不自动释放
   - 通过 `migrateScreenPools()` 支持代际重置，但需调用方主动触发
   - ink.tsx 中每 5 分钟执行一次代际重置

2. **双宽字符边界情况**
   - 软换行时的 SpacerHead 处理复杂，容易出错
   - 跨 blitRegion 边界的 Wide 字符需要特殊处理
   - 终端对双宽字符的渲染存在差异

3. **整数溢出**
   - styleId 限制为 15 位（最大 32767）
   - hyperlinkId 限制为 15 位
   - 实际使用中不太可能溢出，但无运行时检查

4. **Damage 跟踪遗漏**
   - 某些直接操作 cells 数组的代码可能忘记更新 damage
   - 导致 diff 跳过变化区域，出现渲染残留

### 边界情况

1. **零尺寸屏幕**
   - `createScreen()` 会处理无效尺寸（负数、非整数）
   - 返回最小有效屏幕（0x0 或 1x1）

2. **跨屏幕 blit**
   - 要求源和目标使用相同的 Pool
   - 不同 Pool 时需要先调用 `migrateScreenPools()`

3. **选择背景色未设置**
   - `withSelectionBg()` 在背景色未设置时回退到 `withInverse()`
   - 确保测试环境和首次渲染有可见反馈

### 改进建议

1. **性能优化**
   - 考虑使用 SIMD 指令加速 `findNextDiff()`（Bun.indexOfFirstDifference）
   - 对 damage 矩形进行更精细的合并，减少扫描面积
   - 考虑使用 Worker 进行 diff 计算（大型屏幕）

2. **可维护性**
   - 将样式特效（withInverse/withCurrentMatch/withSelectionBg）提取到独立模块
   - 添加更多单元测试覆盖边界情况
   - 使用 TypeScript 的 branded types 区分不同 ID 类型

3. **功能扩展**
   - 支持更多 ANSI 特效（斜体、删除线等）
   - 支持真彩色背景的选择高亮
   - 支持单元格元数据（用于更复杂的交互）

4. **调试支持**
   - 添加屏幕内容导出功能（用于测试对比）
   - 添加 damage 区域可视化
   - 添加 Pool 使用统计
