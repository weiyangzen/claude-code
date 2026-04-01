# Fallback.tsx 深度研究文档

## 1. 场景与职责

### 1.1 定位与用途

`Fallback.tsx` 是 Claude Code 中 **StructuredDiff 组件的降级渲染方案**，当主渲染路径（基于 Rust NAPI 的语法高亮 diff）不可用时被调用。它是代码 diff 展示的"最后一道防线"，确保即使在以下情况下用户仍能看到可读的代码变更：

- 语法高亮被禁用（`CLAUDE_CODE_SYNTAX_HIGHLIGHT=0`）
- 原生 color-diff 模块加载失败
- 用户设置中关闭了语法高亮
- 跳过高亮标记被设置

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **基础 Diff 渲染** | 将 `diff` 库的 `StructuredPatchHunk` 格式转换为终端可渲染的 React 元素 |
| **词级 Diff 高亮** | 对相邻的删除/添加行进行词级差异分析，高亮具体变更部分 |
| **行号生成** | 根据 patch 的 oldStart/newStart 计算正确的行号 |
| **文本换行** | 处理长行文本的自动换行，适配终端宽度 |
| **主题适配** | 支持明暗主题、色盲友好主题、ANSI 主题的颜色渲染 |

### 1.3 调用场景

```
StructuredDiff.tsx (主组件)
    ↓ 检查 colorDiff 模块可用性
    ↓ 检查用户设置 (syntaxHighlightingDisabled)
    ↓ 检查 skipHighlighting 标记
    ↓ 任一条件满足
StructuredDiffFallback (本组件) ← 进入降级渲染
```

---

## 2. 功能点目的

### 2.1 词级 Diff 高亮 (Word-Level Diff)

**目的**：让用户一眼看出行内具体哪些词发生了变化，而非只看到整行变色。

**示例效果**：
```
// 原始代码
function oldName(param) { return param.oldProperty; }

// 变更后  
function newName(param) { return param.newProperty; }

// 词级高亮效果
// - function oldName(param) { return param.oldProperty; }  
//   (oldName 深红背景, oldProperty 深红背景)
// + function newName(param) { return param.newProperty; }
//   (newName 深绿背景, newProperty 深绿背景)
```

### 2.2 智能降级策略

**阈值控制**：`CHANGE_THRESHOLD = 0.4`

当变更比例超过 40% 时，放弃词级高亮，改用整行高亮。这是因为：
- 大面积变更时词级高亮反而增加视觉噪音
- 性能考虑：大段文本的 `diffWordsWithSpace` 计算开销较高

### 2.3 行号与标记对齐

- 动态计算行号宽度，确保右对齐
- 统一格式：`{lineNumber} {+/-}`
- 换行后的续行保持缩进对齐

---

## 3. 具体技术实现

### 3.1 核心数据流

```
StructuredPatchHunk
    ↓
transformLinesToObjects()  // 解析 +/- 前缀，标记 add/remove/nochange
    ↓
processAdjacentLines()     // 配对相邻的 remove/add 行
    ↓
numberDiffLines()          // 计算行号
    ↓
formatDiff()               // 主渲染函数
    ↓
generateWordDiffElements() // 尝试词级渲染
    ↓ 失败或不适合词级渲染
标准行渲染                  // 整行背景色
```

### 3.2 关键数据结构

```typescript
// 行对象（贯穿处理流程）
interface LineObject {
  code: string;           // 去除 +/- 前缀后的代码内容
  i: number;              // 行号（初始为0，由 numberDiffLines 填充）
  type: 'add' | 'remove' | 'nochange';
  originalCode: string;   // 原始代码（去除前缀后）
  wordDiff?: boolean;     // 是否启用词级 diff
  matchedLine?: LineObject; // 配对的行（用于词级 diff）
}

// 词级 diff 结果片段
type DiffPart = {
  added?: boolean;        // 是否为新增内容
  removed?: boolean;      // 是否为删除内容
  value: string;          // 文本内容
};
```

### 3.3 行配对算法 (processAdjacentLines)

```typescript
// 算法逻辑：扫描行序列，找到连续的 remove 行后紧跟 add 行的模式
// 例如：[-old1, -old2, +new1, +new2] 会被配对为 (old1,new1), (old2,new2)

while (i < lineObjects.length) {
  if (current.type === 'remove') {
    // 1. 收集连续 remove 行
    // 2. 收集后续连续 add 行
    // 3. 按顺序一一配对
    // 4. 标记 wordDiff = true，互相设置 matchedLine
  }
}
```

**边界处理**：
- remove 行数 ≠ add 行数时，多余的行按普通行处理
- 非连续的 remove/add 不触发词级 diff

### 3.4 词级渲染算法 (generateWordDiffElements)

```typescript
function generateWordDiffElements(item, width, maxWidth, dim, overrideTheme) {
  // 1. 调用 diffWordsWithSpace 计算词级差异
  const wordDiffs = calculateWordDiffs(removedLineText, addedLineText);
  
  // 2. 计算变更比例，超过阈值则返回 null（触发降级）
  const changeRatio = changedLength / totalLength;
  if (changeRatio > CHANGE_THRESHOLD || dim) return null;
  
  // 3. 逐词渲染，根据当前行类型决定显示哪些部分
  //    - remove 行：显示 removed 部分 + 未变更部分
  //    - add 行：显示 added 部分 + 未变更部分
  
  // 4. 手动处理换行（wrapText），确保不换行破坏词级高亮
}
```

### 3.5 行号计算逻辑 (numberDiffLines)

```typescript
// 行号分配规则：
// - nochange: 同时增加 oldLine 和 newLine 计数
// - add: 只增加 newLine 计数
// - remove: 只增加 oldLine 计数，但多行 remove 时行号连续

switch (type) {
  case 'nochange': i++; result.push(line); break;
  case 'add': i++; result.push(line); break;
  case 'remove': 
    // 多行 remove 连续分配行号，最后回退计数
    // 这样 remove 块有连续行号，add 块从新行号开始
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 入口与导出

| 符号 | 类型 | 说明 |
|------|------|------|
| `StructuredDiffFallback` | React 组件 | 主导出，接收 patch/dim/width 参数 |
| `transformLinesToObjects` | 函数 | 行解析，被导出供测试 |
| `processAdjacentLines` | 函数 | 行配对，被导出供测试 |
| `calculateWordDiffs` | 函数 | 词级 diff 计算，被导出供测试 |
| `numberDiffLines` | 函数 | 行号计算，被导出供测试 |

### 4.2 文件依赖图

```
Fallback.tsx
├── diff (npm 包)
│   └── diffWordsWithSpace, StructuredPatchHunk
├── react
├── src/utils/theme.js
│   └── ThemeName (类型)
├── ../../ink/stringWidth.js
│   └── stringWidth (字符宽度计算)
└── ../../ink.js
    ├── Box (布局容器)
    ├── NoSelect (选择排除)
    ├── Text (文本渲染)
    ├── useTheme (主题钩子)
    └── wrapText (文本换行)
```

### 4.3 关键代码片段

**React Compiler 优化标记**（文件头部）：
```typescript
import { c as _c } from "react/compiler-runtime";
// 使用 React Compiler 的自动缓存优化
// $[n] 是编译器生成的缓存槽位
```

**主题颜色映射**：
```typescript
const bgColor = type === 'add' 
  ? dim ? 'diffAddedDimmed' : 'diffAdded'
  : type === 'remove'
    ? dim ? 'diffRemovedDimmed' : 'diffRemoved'
    : undefined;
```

**文本换行处理**：
```typescript
const availableContentWidth = Math.max(1, safeWidth - maxWidth - 1 - diffPrefixWidth);
const wrappedText = wrapText(code, availableContentWidth, 'wrap');
const wrappedLines = wrappedText.split('\n');
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 版本约束 |
|------|------|----------|
| `diff` | 词级 diff 算法 | ^5.x |
| `react` | UI 框架 | ^18.x |
| `src/ink/*` | 自定义 Ink 组件 | 内部 |
| `src/utils/theme` | 主题类型定义 | 内部 |

### 5.2 与父组件交互

**Props 接口**：
```typescript
type Props = {
  patch: StructuredPatchHunk;  // diff 库的 hunk 格式
  dim: boolean;                 // 是否使用暗淡颜色
  width: number;                // 终端可用宽度
};
```

**调用方** (`StructuredDiff.tsx`):
```typescript
// 当 colorDiff 模块不可用时
if (!cached) {
  return (
    <Box>
      <StructuredDiffFallback patch={patch} dim={dim} width={width} />
    </Box>
  );
}
```

### 5.3 主题系统集成

颜色值通过 `useTheme()` 获取，实际颜色定义在 `src/utils/theme.ts`：

```typescript
// theme.ts 中的 diff 相关颜色
diffAdded: 'rgb(105,219,124)',        // 亮绿
diffRemoved: 'rgb(255,168,180)',      // 亮红
diffAddedDimmed: 'rgb(199,225,203)',  // 淡绿
diffRemovedDimmed: 'rgb(253,210,216)', // 淡红
diffAddedWord: 'rgb(47,157,68)',      // 深绿（词级）
diffRemovedWord: 'rgb(209,69,75)',    // 深红（词级）
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **大文件性能** | 大 diff 文件的词级 diff 计算可能卡顿 | 40% 变更阈值自动降级 |
| **宽字符计算** | CJK/emoji 等宽字符可能导致换行错位 | 使用 `stringWidth` 而非 `length` |
| **极窄终端** | 宽度 < 20 时布局可能混乱 | `safeWidth = Math.max(1, ...)` |
| **React Compiler 依赖** | 编译后代码可读性差，调试困难 | 保留原始 TS 源码 |

### 6.2 边界条件

```typescript
// 1. 空 patch
patch.lines = []  // 已处理，返回空 Box

// 2. 零宽度终端
width = 0  // safeWidth 确保至少为 1

// 3. 超长单行
单字符宽度 > availableContentWidth  // 强制至少渲染一个字符

// 4. 非对称 remove/add
removeLines = 3, addLines = 1  // 多余 remove 按普通行处理
```

### 6.3 改进建议

1. **虚拟滚动支持**
   - 当前：大 diff 全部渲染
   - 建议：超过 1000 行时启用虚拟滚动

2. **增量词级 Diff**
   - 当前：整行对比
   - 建议：对极长行（>500 字符）分段对比

3. **主题缓存**
   - 当前：每次渲染重新计算颜色
   - 建议：缓存主题到 React Compiler 槽位

4. **测试覆盖**
   - 当前：部分工具函数导出供测试
   - 建议：添加可视化快照测试

### 6.4 相关 Issue 模式

- **#21439**: Fullscreen 默认启用后，diff 渲染性能优化（已处理）
- **#20378**: Gutter 分割缓存优化（已处理）
- **宽字符问题**: 需持续监控 `stringWidth` 的准确性

---

## 附录：代码统计

| 指标 | 数值 |
|------|------|
| 总行数 | ~487 行 |
| 导出函数 | 5 个 |
| React 组件 | 1 个 |
| 类型定义 | 4 个 |
| 测试导出 | 4 个 |
