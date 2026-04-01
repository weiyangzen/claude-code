# squash-text-nodes.ts 研究文档

## 场景与职责

`squash-text-nodes.ts` 是 Ink 终端渲染引擎的文本处理工具模块，负责将 DOM 树中的文本节点扁平化为带样式的段落列表。这是从 React 组件树到终端输出的关键转换步骤之一。

**核心职责：**
1. **文本节点扁平化** - 递归遍历 DOM 树，提取所有文本内容
2. **样式继承** - 正确传播文本样式（颜色、粗体、斜体等）
3. **超链接处理** - 传播 `<ink-link>` 的 href 属性
4. **结构化输出** - 生成带样式的段落（StyledSegment），供后续渲染使用

**在架构中的位置：**
- 被 `render-node-to-output.ts` 在渲染过程中调用
- 被 `dom.ts` 用于文本测量（通过默认导出的 `squashTextNodes`）
- 是连接 React 组件树与终端输出的桥梁

---

## 功能点目的

### 1. 文本节点扁平化

**目的：** 将嵌套的 DOM 树结构转换为扁平的文本段落列表

**输入示例：**
```jsx
<ink-text color="red">
  Hello <ink-text bold>world</ink-text>!
</ink-text>
```

**输出结果：**
```typescript
[
  { text: "Hello ", styles: { color: "red" } },
  { text: "world", styles: { color: "red", bold: true } },
  { text: "!", styles: { color: "red" } }
]
```

**必要性：**
- 终端输出是线性文本流，需要扁平化表示
- 样式变化需要精确到字符级别
- 支持后续的文本换行、截断等处理

### 2. 样式继承

**目的：** 确保嵌套组件的样式正确传播

**实现方式：**
- 使用对象展开 `{ ...inheritedStyles, ...node.textStyles }`
- 子节点样式覆盖父节点同名样式
- 支持多层嵌套

### 3. 超链接传播

**目的：** 支持 OSC 8 超链接的嵌套传播

**处理逻辑：**
- `<ink-link href="...">` 设置新的 hyperlink 值
- 子节点继承该 hyperlink
- 嵌套 link 时，内层 href 优先

### 4. 纯文本提取

**目的：** 支持布局计算中的文本测量

**使用场景：**
- Yoga 布局引擎需要知道文本宽度
- 计算换行位置
- 确定容器最小宽度

---

## 具体技术实现

### 核心算法

```typescript
export function squashTextNodesToSegments(
  node: DOMElement,
  inheritedStyles: TextStyles = {},
  inheritedHyperlink?: string,
  out: StyledSegment[] = [],
): StyledSegment[] {
  // 1. 合并继承样式和当前节点样式
  const mergedStyles = node.textStyles
    ? { ...inheritedStyles, ...node.textStyles }
    : inheritedStyles

  // 2. 遍历子节点
  for (const childNode of node.childNodes) {
    if (childNode === undefined) continue

    switch (childNode.nodeName) {
      case '#text':
        // 文本节点：创建段落
        if (childNode.nodeValue.length > 0) {
          out.push({
            text: childNode.nodeValue,
            styles: mergedStyles,
            hyperlink: inheritedHyperlink,
          })
        }
        break

      case 'ink-text':
      case 'ink-virtual-text':
        // 文本容器：递归处理，继承当前合并样式
        squashTextNodesToSegments(
          childNode,
          mergedStyles,
          inheritedHyperlink,
          out,
        )
        break

      case 'ink-link':
        // 链接：提取 href，作为新的 hyperlink 传播
        const href = childNode.attributes['href'] as string | undefined
        squashTextNodesToSegments(
          childNode,
          mergedStyles,
          href || inheritedHyperlink,
          out,
        )
        break
    }
  }

  return out
}
```

### 数据结构

```typescript
/**
 * 带样式的文本段落
 */
export type StyledSegment = {
  text: string        // 纯文本内容
  styles: TextStyles  // 样式对象（颜色、粗体等）
  hyperlink?: string  // OSC 8 超链接 URL
}
```

### 纯文本版本

```typescript
function squashTextNodes(node: DOMElement): string {
  let text = ''

  for (const childNode of node.childNodes) {
    if (childNode === undefined) continue

    if (childNode.nodeName === '#text') {
      text += childNode.nodeValue
    } else if (
      childNode.nodeName === 'ink-text' ||
      childNode.nodeName === 'ink-virtual-text' ||
      childNode.nodeName === 'ink-link'
    ) {
      text += squashTextNodes(childNode)
    }
  }

  return text
}
```

---

## 关键代码路径与文件引用

### 导出内容

| 名称 | 类型 | 位置 | 说明 |
|------|------|------|------|
| `StyledSegment` | type | line 8-12 | 带样式的文本段落类型 |
| `squashTextNodesToSegments` | function | line 18-63 | 主函数：扁平化为段落 |
| `squashTextNodes` | function | line 69-90 | 默认导出：提取纯文本 |

### 导入依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `./dom.js` | DOMElement | DOM 节点类型 |
| `./styles.js` | TextStyles | 文本样式类型 |

### 被调用方

| 模块 | 调用内容 | 说明 |
|------|----------|------|
| `render-node-to-output.ts` | squashTextNodesToSegments | 渲染时提取带样式文本 |
| `dom.ts` | squashTextNodes (默认导入) | 布局时的文本测量 |

**render-node-to-output.ts 中的使用：**
```typescript
import {
  type StyledSegment,
  squashTextNodesToSegments,
} from './squash-text-nodes.js'

// 在渲染文本节点时
const segments = squashTextNodesToSegments(node, {}, undefined, [])
// 然后对每个 segment 进行样式应用和输出
```

**dom.ts 中的使用：**
```typescript
import squashTextNodes from './squash-text-nodes.js'

// 在 measureText 中
const text = squashTextNodes(node)
// 用于计算文本宽度
```

---

## 依赖与外部交互

### 与 dom.ts 的交互

1. **类型依赖**
   - `DOMElement` - 节点类型定义
   - `TextStyles` - 样式类型定义

2. **功能依赖**
   - `dom.ts` 使用 `squashTextNodes` 进行文本测量
   - 测量结果用于 Yoga 布局计算

### 与 render-node-to-output.ts 的交互

1. **渲染流程**
   - 渲染器遇到文本容器节点时调用 `squashTextNodesToSegments`
   - 获取的段落列表被逐个处理
   - 每个段落的样式被转换为 ANSI 代码

2. **样式传播**
   - 继承机制确保嵌套样式正确应用
   - 例如：`<Text color="red"><Text bold>bold red</Text></Text>`

### 支持的节点类型

| 节点名 | 处理方式 | 说明 |
|--------|----------|------|
| `#text` | 创建段落 | 叶子节点，包含实际文本 |
| `ink-text` | 递归处理 | 标准文本容器 |
| `ink-virtual-text` | 递归处理 | 虚拟文本容器（用于特殊布局）|
| `ink-link` | 递归处理+提取 href | 超链接容器 |

---

## 风险、边界与改进建议

### 已知风险

1. **递归深度**
   - 深层嵌套的组件可能导致栈溢出
   - 实际使用中罕见，但理论存在

2. **undefined 子节点**
   - 代码中检查了 `childNode === undefined`
   - 这表明 DOM 树可能包含稀疏数组
   - 可能隐藏其他数据一致性问题

3. **性能问题**
   - 每次渲染都重新遍历整个子树
   - 大文本内容时可能较耗时
   - 无缓存机制

### 边界情况

1. **空文本节点**
   - `nodeValue.length > 0` 检查跳过空文本
   - 避免生成空段落

2. **无样式节点**
   - `node.textStyles` 可能为 undefined
   - 使用条件展开正确继承

3. **嵌套超链接**
   - 内层 link 的 href 覆盖外层
   - 符合 HTML 语义

4. **未知节点类型**
   - 非上述节点类型被忽略
   - 可能导致内容丢失（如自定义组件）

### 改进建议

1. **性能优化**
   - 添加缓存机制，避免重复遍历未变化的子树
   - 使用迭代替代递归，避免栈深度问题
   - 对于大文本，考虑分块处理

2. **功能增强**
   - 支持更多节点类型（如 `ink-raw-ansi`）
   - 添加对 `style` 属性的支持（内联样式）
   - 支持样式回调（动态样式计算）

3. **代码质量**
   - 添加单元测试覆盖各种嵌套场景
   - 使用 TypeScript 的 exhaustive check 确保节点类型处理完整
   - 添加 JSDoc 文档说明

4. **调试支持**
   - 添加开发模式下的树结构输出
   - 记录样式继承路径
   - 检测潜在的问题模式（如过深嵌套）

5. **类型安全**
   - 使用更精确的节点类型替代 string
   - 添加运行时类型检查
   - 使用 branded types 区分不同 ID

### 潜在问题

1. **与 React 的同步**
   - 如果 DOM 树结构与 React 组件树不完全对应
   - 可能导致样式传播不符合预期

2. **内存泄漏**
   - `out` 数组在递归中传递
   - 确保无循环引用

3. **Unicode 处理**
   - 文本按 JavaScript 字符串处理
   - 复杂 grapheme cluster 可能在后续处理中被错误分割
