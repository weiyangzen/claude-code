# highlightMatch.tsx 深度研究

## 场景与职责

本模块提供文本搜索匹配的高亮功能，用于在终端 UI 中高亮显示搜索结果中的匹配部分。采用"反色"（inverse）样式，在搜索结果列表和预览面板中直观展示查询词匹配位置。

**核心场景：**
1. **全局搜索高亮**：GlobalSearchDialog 中高亮文件名和内容匹配
2. **快速打开高亮**：QuickOpenDialog 中高亮文件名匹配
3. **文件索引搜索**：native-ts/file-index 模块的搜索结果展示

## 功能点目的

### 1. 搜索匹配高亮
- **目的**：在搜索结果中突出显示匹配文本
- **样式**：反色（inverse）显示，与终端背景色反转
- **匹配方式**：大小写不敏感

### 2. 多匹配处理
- **目的**：正确处理一行中的多个匹配
- **实现**：分割文本，为每个匹配创建独立的 Text 组件

### 3. React 组件化
- **目的**：与 Ink 渲染框架集成
- **实现**：返回 React.ReactNode，可直接嵌入 JSX

## 具体技术实现

### 核心函数
```typescript
export function highlightMatch(text: string, query: string): React.ReactNode {
  // 1. 空查询直接返回原文本
  if (!query) return text
  
  // 2. 大小写不敏感匹配
  const queryLower = query.toLowerCase()
  const textLower = text.toLowerCase()
  
  // 3. 构建结果数组
  const parts: React.ReactNode[] = []
  let offset = 0
  let idx = textLower.indexOf(queryLower, offset)
  
  // 4. 无匹配直接返回原文本
  if (idx === -1) return text
  
  // 5. 遍历所有匹配
  while (idx !== -1) {
    // 添加匹配前的文本
    if (idx > offset) {
      parts.push(text.slice(offset, idx))
    }
    
    // 添加高亮匹配文本
    parts.push(
      <Text key={idx} inverse>
        {text.slice(idx, idx + query.length)}
      </Text>
    )
    
    // 更新偏移量
    offset = idx + query.length
    idx = textLower.indexOf(queryLower, offset)
  }
  
  // 6. 添加剩余文本
  if (offset < text.length) {
    parts.push(text.slice(offset))
  }
  
  // 7. 返回 Fragment 包裹的所有部分
  return <>{parts}</>
}
```

### 算法说明

**匹配流程：**
1. 将查询词和文本转为小写进行比较
2. 使用 `indexOf` 查找所有匹配位置
3. 将文本分割为：前缀 + 高亮匹配 + 后缀
4. 为每个匹配创建带 `inverse` 属性的 Text 组件
5. 使用 React Fragment (`<>...</>`) 包裹所有部分

**复杂度：**
- 时间：O(n × m)，n 为文本长度，m 为匹配次数
- 空间：O(n)，创建 parts 数组

### 组件结构
```
<>                          // React Fragment
  "prefix text"             // 普通字符串
  <Text inverse>match</Text> // 高亮匹配
  "suffix text"             // 普通字符串
  <Text inverse>match2</Text>
  ...
</>
```

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `highlightMatch` | 函数 | 高亮匹配文本 |

### 调用方
1. **GlobalSearchDialog.tsx**: 全局搜索结果高亮
2. **QuickOpenDialog.tsx**: 快速打开文件名高亮
3. **native-ts/file-index/index.ts**: 文件索引搜索结果

### 依赖模块
```typescript
import * as React from 'react'
import { Text } from '../ink.js'
```

## 依赖与外部交互

### 上游依赖

1. **React**: JSX 和组件基础
   - `React.ReactNode`: 返回类型
   - Fragment 语法 (`<>...</>`)

2. **ink.js**: Ink 组件库
   - `Text`: 文本渲染组件
   - `inverse` 属性：反色样式

### 样式说明

**反色（Inverse）：**
- 终端中的 "inverse" 或 "reverse" 属性
- 交换前景色和背景色
- 效果类似选中高亮

**示例：**
```
正常文本: [白底黑字]正常文本
高亮匹配: [黑底白字]匹配词
```

## 风险、边界与改进建议

### 已知风险

1. **正则特殊字符**
   - 风险：查询词包含正则特殊字符（如 `*`, `?`）
   - 现状：使用字符串 `indexOf`，非正则匹配，无此问题

2. **重叠匹配**
   - 风险：查询词 `"aa"` 在文本 `"aaa"` 中的重叠匹配
   - 现状：`indexOf` 从匹配后位置继续，无重叠

3. **性能问题**
   - 风险：超长文本（如 MB 级别）可能卡顿
   - 缓解：调用方应预先截断文本

4. **Unicode 处理**
   - 风险：多字节字符（如 emoji）的索引计算
   - 现状：JavaScript 字符串按 UTF-16 码元处理

### 边界情况

1. **空查询**：直接返回原文本
2. **无匹配**：直接返回原文本
3. **查询长度大于文本**：无匹配，返回原文本
4. **空文本**：返回空字符串
5. **全匹配**：整个文本反色
6. **连续匹配**：如 `"abab"` 中查询 `"ab"`，正确显示两个高亮

### 改进建议

1. **正则表达式支持**
   - 建议：支持正则匹配模式
   - 实现：添加 `useRegex` 参数
   - 风险：需要转义处理

2. **多查询词支持**
   - 建议：支持多个查询词同时高亮
   - 实现：接受 `query: string[]`
   - 挑战：重叠高亮的样式冲突

3. **不同高亮样式**
   - 建议：支持颜色、粗体等多种高亮方式
   - 实现：添加 `style` 参数
   - 场景：区分不同类型的匹配（文件名 vs 内容）

4. **性能优化**
   - 建议：虚拟滚动场景下的优化
   - 实现：仅高亮可见区域
   - 场景：大文件搜索结果列表

5. **匹配位置信息**
   - 建议：返回匹配位置数组
   - 收益：支持点击跳转等功能

6. **大小写敏感选项**
   - 建议：添加 `caseSensitive` 参数
   - 场景：特定搜索需求

### 测试要点

1. 基础匹配功能
2. 多匹配处理
3. 大小写不敏感
4. 空查询/空文本处理
5. Unicode 字符处理
6. 超长文本性能
7. 特殊字符处理

### 代码示例

**基本使用：**
```tsx
import { highlightMatch } from './highlightMatch'

function SearchResult({ text, query }: { text: string; query: string }) {
  return <Box>{highlightMatch(text, query)}</Box>
}

// 输入: text="Hello World", query="world"
// 输出: Hello [反色]World[/反色]
```

**在列表中使用：**
```tsx
{results.map((result, i) => (
  <Box key={i}>
    {highlightMatch(result.fileName, query)}
    {result.content && (
      <Box>{highlightMatch(result.content, query)}</Box>
    )}
  </Box>
))}
```
