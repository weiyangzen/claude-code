# searchHighlight.ts 研究文档

## 场景与职责

`searchHighlight.ts` 是 Ink 终端 UI 框架的搜索高亮模块，负责在全屏模式下高亮显示与搜索查询匹配的所有可见文本。

**核心职责：**
1. **搜索匹配高亮** - 在屏幕缓冲区中查找查询字符串并应用反色样式
2. **可见区域扫描** - 仅处理当前可见的屏幕行（viewport）
3. **宽字符支持** - 正确处理 CJK、Emoji 等双宽字符的列映射
4. **noSelect 区域排除** - 跳过标记为不可选择的区域（如行号、边距）
5. **大小写不敏感** - 默认使用大小写不敏感匹配

**在架构中的位置：**
- 被 `ink.tsx` 主控制器在每帧渲染后调用
- 与 `selection.ts` 类似，都是"后处理"性质的视觉反馈模块
- 与 `render-to-screen.ts` 中的 `scanPositions` 和 `applyPositionedHighlight` 配合，实现当前匹配项的黄色高亮

---

## 功能点目的

### 1. 全局搜索高亮

**目的：** 在终端内容中视觉标记所有匹配的搜索词

**实现特点：**
- 逐行扫描屏幕缓冲区
- 构建行文本时跳过 SpacerTail/SpacerHead（双宽字符的占位单元格）
- 跳过 noSelect 标记的单元格（边距、行号等）
- 使用反色（SGR 7）作为高亮样式

### 2. 当前匹配项高亮（配合 render-to-screen.ts）

**目的：** 区分"所有匹配"和"当前选中的匹配"

**分工：**
- `searchHighlight.ts` - 处理所有可见匹配的反色高亮
- `render-to-screen.ts:applyPositionedHighlight()` - 处理当前匹配项的特殊高亮（黄底+粗体+下划线）

**原因：**
- 当前匹配项位置由 VirtualMessageList 通过 DOM 扫描确定
- 需要精确的行/列定位，与全局扫描的实现方式不同

### 3. 宽字符列映射

**目的：** 正确处理字符串索引与屏幕列的映射关系

**问题：**
- 双宽字符占用 2 个单元格，但字符串中只占 1-2 个 code unit
- Turkish İ 等小写转换后可能变成多个 code unit

**解决方案：**
- 构建 `codeUnitToCell` 映射数组
- 逐字符小写转换，记录每个 code unit 对应的单元格索引

---

## 具体技术实现

### 核心算法

```typescript
export function applySearchHighlight(screen, query, stylePool): boolean {
  if (!query) return false
  const lq = query.toLowerCase()
  const qlen = lq.length
  
  for (each row in screen) {
    // 1. 构建行文本和列映射
    let text = ''
    const colOf: number[] = []        // 字符索引 -> 列号
    const codeUnitToCell: number[] = [] // code unit 索引 -> 字符索引
    
    for (each col in row) {
      const cell = cellAtIndex(screen, rowOff + col)
      
      // 跳过占位单元格和 noSelect 区域
      if (cell.width === SpacerTail || cell.width === SpacerHead || noSelect[idx] === 1) {
        continue
      }
      
      // 逐字符小写转换，建立映射
      const lc = cell.char.toLowerCase()
      const cellIdx = colOf.length
      for (each code unit in lc) {
        codeUnitToCell.push(cellIdx)
      }
      text += lc
      colOf.push(col)
    }
    
    // 2. 查找所有匹配位置
    let pos = text.indexOf(lq)
    while (pos >= 0) {
      applied = true
      const startCi = codeUnitToCell[pos]           // 匹配起始单元格索引
      const endCi = codeUnitToCell[pos + qlen - 1]  // 匹配结束单元格索引
      
      // 3. 对匹配范围内的单元格应用反色
      for (ci from startCi to endCi) {
        const col = colOf[ci]
        const cell = cellAtIndex(screen, rowOff + col)
        setCellStyleId(screen, col, row, stylePool.withInverse(cell.styleId))
      }
      
      // 非重叠前进（less/vim/grep/Ctrl+F 风格）
      pos = text.indexOf(lq, pos + qlen)
    }
  }
  
  return applied  // 返回是否有任何匹配被高亮
}
```

### 列映射详解

```
屏幕单元格: [H][e][l][l][o][  ][本][ ][文]
列号:        0  1  2  3  4  5   6  7  8

实际字符:    H  e  l  l  o     本     文
            (每个 ASCII 占 1 列)
            ("本" 是双宽字符，占用列 6-7，列 7 是 SpacerTail)

colOf 数组: [0, 1, 2, 3, 4, 6, 8]
            (第 i 个字符在屏幕的第 colOf[i] 列)

codeUnitToCell: [0, 1, 2, 3, 4, 5, 6, 7]
                (第 i 个 code unit 对应第 codeUnitToCell[i] 个字符)
                (对于 ASCII，code unit = 字符)
                (对于 Turkish İ→i+U+0307，1 字符 = 2 code units)
```

### 非重叠匹配

```typescript
// 使用 pos + qlen 而非 pos + 1
// 避免 'aaa' 中搜索 'aa' 时，位置 0 和 1 都被匹配
// 这样位置 0 的匹配会覆盖位置 1 的部分重叠
pos = text.indexOf(lq, pos + qlen)
```

---

## 关键代码路径与文件引用

### 导出函数

| 函数 | 位置 | 说明 |
|------|------|------|
| `applySearchHighlight()` | line 27-93 | 主函数，应用搜索高亮 |

### 导入依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `./screen.js` | CellWidth, cellAtIndex, type Screen, type StylePool, setCellStyleId | 屏幕操作和样式应用 |

### 被调用方

| 模块 | 调用位置 | 说明 |
|------|----------|------|
| `ink.tsx` | onRender 方法中 | 每帧渲染后调用，应用搜索高亮 |

**调用代码片段（ink.tsx）：**
```typescript
// Search highlight overlay (alt-screen only)
if (this.altScreenActive && this.searchHighlightQuery) {
  const hadHighlight = applySearchHighlight(
    backScreen,
    this.searchHighlightQuery,
    this.stylePool,
  );
  if (hadHighlight) {
    // 强制下一帧全量渲染，避免 blit 优化跳过已高亮区域
    this.prevFrameContaminated = true;
  }
}
```

---

## 依赖与外部交互

### 与 screen.ts 的交互

1. **读取单元格**
   - 使用 `cellAtIndex()` 读取每个单元格的内容和宽度
   - 检查 `screen.noSelect` 数组跳过不可选择区域

2. **修改样式**
   - 使用 `setCellStyleId()` 应用反色样式
   - 通过 `stylePool.withInverse()` 获取带反色的样式 ID

### 与 ink.tsx 的交互

1. **触发渲染**
   - ink.tsx 在 `onRender` 中调用 `applySearchHighlight()`
   - 传入当前搜索查询 `this.searchHighlightQuery`

2. **强制重绘**
   - 当有高亮被应用时，设置 `this.prevFrameContaminated = true`
   - 这会禁用 blit 优化，确保下一帧全量渲染

### 与 render-to-screen.ts 的关系

| 特性 | searchHighlight.ts | render-to-screen.ts |
|------|-------------------|---------------------|
| 扫描范围 | 整个可见屏幕 | 单个消息（离屏渲染）|
| 匹配样式 | 反色（所有匹配）| 黄底+粗体+下划线（当前匹配）|
| 位置来源 | 实时扫描屏幕缓冲区 | 预先扫描的 DOM 位置 |
| 调用时机 | 每帧渲染后 | 消息渲染时 |

---

## 风险、边界与改进建议

### 已知风险

1. **性能问题**
   - 每帧都进行全屏扫描，时间复杂度 O(rows × cols)
   - 长查询字符串时 `indexOf` 可能较耗时
   - 大量匹配时 `setCellStyleId` 调用频繁

2. **字符编码问题**
   - 依赖 JavaScript 的 `toLowerCase()`，可能不符合某些 locale 预期
   - 复杂 grapheme cluster 的列映射可能不准确

3. **与选择的冲突**
   - 搜索高亮和选择高亮使用相同的样式机制
   - 重叠区域的选择高亮会覆盖搜索高亮（或反之）
   - 当前实现下，后应用的会覆盖先应用的

### 边界情况

1. **空查询**
   - 空字符串直接返回 false，不做任何高亮

2. **无匹配**
   - 返回 false，不触发强制重绘

3. **查询长度超过行长度**
   - `indexOf` 自然返回 -1，无匹配

4. **全 noSelect 行**
   - 行文本为空，`indexOf` 返回 -1

5. **宽字符查询**
   - 查询字符串包含宽字符时，匹配逻辑仍然正确
   - 但 `qlen` 是 code unit 数量，不是显示宽度

### 改进建议

1. **性能优化**
   - 使用 Boyer-Moore 或 KMP 算法加速多位置匹配
   - 缓存行文本，仅在内容变化时重新构建
   - 增量更新：只扫描变化的行

2. **功能增强**
   - 支持正则表达式搜索
   - 支持大小写敏感选项
   - 支持全词匹配
   - 支持高亮样式自定义

3. **代码质量**
   - 与 `render-to-screen.ts` 的 `scanPositions` 提取公共的行文本构建逻辑
   - 添加单元测试覆盖各种字符编码场景
   - 使用更精确的 Unicode case folding

4. **用户体验**
   - 显示匹配计数
   - 支持循环导航（当前是线性）
   - 高亮动画效果
