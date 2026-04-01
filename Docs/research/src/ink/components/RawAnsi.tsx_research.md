# RawAnsi.tsx 研究文档

## 场景与职责

`RawAnsi` 是 Ink 框架中的高性能预渲染 ANSI 内容组件，用于优化已经包含 ANSI 转义序列的内容的渲染性能：

1. **绕过 React 树**: 跳过 `<Ansi> → React 树 → Yoga → squash → 重新序列化` 的完整流程
2. **直接输出**: 将预渲染的 ANSI 内容直接传递给输出系统
3. **性能优化**: 对于大量语法高亮 diff 等内容，避免昂贵的 React 树重建

RawAnsi 适用于外部渲染器（如 ColorDiff NAPI 模块）已经生成 ANSI 转义序列和换行内容的场景。

## 功能点目的

### 1. 性能优化
- **避免重复解析**: 不需要将 ANSI 内容解析为 React 组件树
- **避免 Yoga 布局**: 预渲染内容已经有固定尺寸，不需要 Flexbox 布局计算
- **常量时间测量**: Yoga 测量函数为 O(1)，直接返回预设的 width × height

### 2. 预渲染内容支持
- **已换行**: 每行已经按目标宽度换好
- **ANSI 转义**: 已包含语法高亮等 ANSI 转义序列
- **直接输出**: 内容直接传递给 `output.write()`，由输出系统解析到屏幕缓冲区

### 3. 典型使用场景
```
外部渲染器（ColorDiff NAPI）
├── 生成语法高亮的 diff 输出
├── 按目标宽度换行
└── 返回 ANSI 转义字符串数组

RawAnsi
├── 接收 lines[] 和 width
├── 常量时间测量
└── 直接输出到终端
```

## 具体技术实现

### 关键数据结构

```typescript
type Props = {
  /**
   * Pre-rendered ANSI lines. Each element must be exactly one terminal row
   * (already wrapped to `width` by the producer) with ANSI escape codes inline.
   */
  lines: string[];
  
  /** Column width the producer wrapped to. Sent to Yoga as the fixed leaf width. */
  width: number;
};
```

### 关键流程

1. **空内容处理**
   ```typescript
   if (lines.length === 0) {
     return null;
   }
   ```

2. **内容合并**
   ```typescript
   const text = lines.join("\n");
   ```
   - 将多行用换行符连接
   - 这是传递给输出系统的最终内容

3. **渲染**
   ```typescript
   return (
     <ink-raw-ansi 
       rawText={text} 
       rawWidth={width} 
       rawHeight={lines.length} 
     />
   );
   ```

### 测量函数

在 `src/ink/dom.ts` 中：

```typescript
const measureRawAnsiNode = function (node: DOMElement): {
  width: number;
  height: number;
} {
  return {
    width: node.attributes['rawWidth'] as number,
    height: node.attributes['rawHeight'] as number,
  };
};

// 在 createNode 中设置
if (nodeName === 'ink-raw-ansi') {
  node.yogaNode?.setMeasureFunc(measureRawAnsiNode.bind(null, node));
}
```

### 代码路径

```
RawAnsi.tsx
├── Props 类型定义
│   ├── lines: 预渲染的 ANSI 行数组
│   └── width: 内容宽度
├── RawAnsi 函数组件
│   ├── 空检查: lines.length === 0
│   ├── 合并内容: lines.join("\n")
│   └── 渲染 <ink-raw-ansi>
└── 导出

dom.ts
├── createNode('ink-raw-ansi')
├── 设置 measureRawAnsiNode 测量函数
└── 返回 DOMElement

render-node-to-output.ts
├── 处理 'ink-raw-ansi' 节点
├── 获取 rawText 属性
└── output.write(x, y, text) 直接输出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/ink/dom.ts` | DOMElement 类型和测量函数 |
| `src/ink/render-node-to-output.ts` | 渲染处理 |
| `src/ink/output.ts` | 输出系统 |

### 渲染实现

```typescript
// src/ink/render-node-to-output.ts
if (node.nodeName === 'ink-raw-ansi') {
  // Pre-rendered ANSI content. The producer already wrapped to width and
  // emitted terminal-ready escape codes. Skip squash, measure, wrap, and
  // style re-application — output.write() parses ANSI directly into cells.
  const text = node.attributes['rawText'] as string;
  if (text) {
    output.write(x, y, text);
  }
}
```

### 与普通 Text 的对比

```
普通 Text 渲染流程:
Text 组件
├── React 子节点
├── squashTextNodesToSegments 合并文本节点
├── 应用样式到段
├── wrapText 换行处理
├── applyStylesToWrappedText 应用样式到换行文本
└── output.write

RawAnsi 渲染流程:
RawAnsi 组件
├── 预渲染 lines
├── 直接 output.write
└── output 解析 ANSI 到屏幕缓冲区
```

## 依赖与外部交互

### 运行时依赖

1. **React**: 基础组件框架
2. **React Compiler**: 使用 `_c` 函数进行自动记忆化
3. **Yoga**: 布局引擎（但测量函数为 O(1)）
4. **输出系统**: 解析 ANSI 转义序列

### Props 交互

| Prop | 类型 | 说明 |
|------|------|------|
| lines | string[] | 预渲染的 ANSI 行数组，每行一个终端行 |
| width | number | 内容宽度，传递给 Yoga 作为固定叶节点宽度 |

### 外部渲染器交互

```
外部渲染器（如 ColorDiff NAPI）
├── 接收原始 diff 内容
├── 语法高亮处理
├── ANSI 转义序列生成
├── 按目标宽度换行
└── 返回 string[]

RawAnsi
├── 接收 lines 和 width
├── Yoga 测量（O(1)）
├── 布局计算
└── 直接输出
```

## 风险、边界与改进建议

### 已知风险

1. **尺寸不匹配**: 如果 lines 的实际宽度与传入的 width 不匹配，可能导致布局问题
2. **ANSI 解析依赖**: 输出系统的 ANSI 解析器必须与外部渲染器的输出格式兼容
3. **无样式继承**: RawAnsi 内容不继承父级的 textStyles 等样式

### 边界情况

1. **空 lines**: lines.length === 0 时返回 null
2. **空字符串元素**: lines 中包含空字符串时的处理
3. **width = 0**: 可能导致 Yoga 布局问题
4. **非常大的 lines**: 内存和性能考虑

### 改进建议

1. **参数验证**: 添加对 lines 和 width 的验证
   ```typescript
   if (width <= 0) {
     console.warn('RawAnsi: width must be positive');
     return null;
   }
   if (!lines.every(line => typeof line === 'string')) {
     console.warn('RawAnsi: all lines must be strings');
   }
   ```

2. **调试模式**: 添加开发环境警告，当实际内容宽度与声明的 width 不匹配时提示
   ```typescript
   if (process.env.NODE_ENV === 'development') {
     const actualWidth = Math.max(...lines.map(line => stringWidth(line)));
     if (actualWidth > width) {
       console.warn(`RawAnsi: content width ${actualWidth} exceeds declared width ${width}`);
     }
   }
   ```

3. **增量渲染**: 对于非常大的内容，考虑支持虚拟滚动或增量渲染

4. **缓存优化**: 考虑对不变的 lines 进行记忆化，避免重复 join

5. **类型增强**: 使用 branded type 确保 width 和 lines 的关联性
   ```typescript
   type PreRenderedAnsi = {
     readonly lines: string[];
     readonly width: number;
     readonly __brand: 'preRenderedAnsi';
   };
   ```

### 代码质量

- 组件简单高效
- 使用 React Compiler 自动记忆化
- 注释详细说明了性能优化的原理

### 测试建议

- 测试空 lines 的返回行为
- 测试大尺寸内容的性能
- 测试 ANSI 转义序列的正确解析
- 测试 width 与实际内容宽度不匹配时的布局行为
- 测试与滚动、裁剪的交互

### 使用模式

```tsx
// 基本使用
const ansiLines = generateDiffWithAnsi(content, targetWidth);
<RawAnsi lines={ansiLines} width={targetWidth} />

// 在 ScrollBox 中使用
<ScrollBox>
  <RawAnsi lines={largeAnsiOutput} width={viewportWidth} />
</ScrollBox>

// 与普通内容混合
<Box flexDirection="column">
  <Text bold>Diff Output:</Text>
  <RawAnsi lines={diffLines} width={width} />
</Box>
```

### 性能对比

对于包含大量语法高亮 diff 的长输出：

| 指标 | 普通 Ansi/Text | RawAnsi |
|------|---------------|---------|
| React 树构建 | O(n) 组件 | O(1) 节点 |
| Yoga 布局 | O(n) 测量 | O(1) 测量 |
| 样式应用 | O(n) 处理 | 0（预渲染） |
| 内存占用 | 高（组件树） | 低（字符串数组） |

RawAnsi 是渲染大量预格式化终端内容的推荐方案。
