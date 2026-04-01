# ShimmerChar.tsx 研究文档

## 场景与职责

`ShimmerChar.tsx` 是 Spinner 组件体系中最简单的原子组件，负责渲染**单个带微光效果的字符**。它根据字符索引与当前微光中心位置的关系，决定使用微光颜色还是普通消息颜色。

### 使用场景

- **逐字符微光**: 在需要精确控制每个字符颜色的场景下使用（如早期版本的微光实现）
- **简单高亮**: 标记当前微光位置的单个字符
- **邻近高亮**: 同时高亮微光中心及其相邻字符，形成更宽的视觉效果

### 设计定位

相比 `GlimmerMessage` 的整行处理策略，`ShimmerChar` 采用**逐字符渲染**方式：
- **优点**: 实现简单，每个字符独立决定颜色
- **缺点**: 需要为每个字符创建 React 元素，长消息性能较差
- **现状**: 当前代码库中似乎主要使用 `GlimmerMessage`，`ShimmerChar` 可能用于遗留场景或特定用途

---

## 功能点目的

### 1. 索引匹配高亮
通过比较字符 `index` 与 `glimmerIndex`：
- `index === glimmerIndex`: 精确匹配，使用微光颜色
- `Math.abs(index - glimmerIndex) === 1`: 邻近匹配，也使用微光颜色
- 其他: 使用普通消息颜色

### 2. 三字符宽微光带
高亮区域覆盖微光中心及其左右各一个字符，形成视觉上更明显的"光带"效果。

---

## 具体技术实现

### 关键流程

```
Props 输入 (char, index, glimmerIndex, messageColor, shimmerColor)
    ↓
计算高亮状态:
    isHighlighted = index === glimmerIndex
    isNearHighlight = Math.abs(index - glimmerIndex) === 1
    ↓
决定颜色:
    shouldUseShimmer = isHighlighted || isNearHighlight
    color = shouldUseShimmer ? shimmerColor : messageColor
    ↓
渲染 <Text color={color}>{char}</Text>
```

### 数据结构

```typescript
// Props 定义
type Props = {
  char: string;              // 单个字符
  index: number;             // 字符在消息中的索引
  glimmerIndex: number;      // 当前微光中心索引
  messageColor: keyof Theme; // 普通状态颜色键
  shimmerColor: keyof Theme; // 微光状态颜色键
}
```

### 核心算法

**高亮判断逻辑**
```typescript
const isHighlighted = index === glimmerIndex;
const isNearHighlight = Math.abs(index - glimmerIndex) === 1;
const shouldUseShimmer = isHighlighted || isNearHighlight;
```

**颜色选择**
```typescript
const color = shouldUseShimmer ? shimmerColor : messageColor;
return <Text color={color}>{char}</Text>;
```

### React Compiler 优化

编译后代码使用 3 槽位缓存：
```typescript
const $ = _c(3);
// $[0]: char (缓存键)
// $[1]: t1/color (缓存键)
// $[2]: t2/渲染结果 (缓存值)

if ($[0] !== char || $[1] !== t1) {
  t2 = <Text color={t1}>{char}</Text>;
  $[0] = char;
  $[1] = t1;
  $[2] = t2;
} else {
  t2 = $[2];
}
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../../ink.js` | `Text` | Ink 文本组件 |
| `../../utils/theme.js` | `Theme` | 主题类型定义 |

### 被调用方

| 文件路径 | 使用方式 |
|---------|---------|
| 可能用于遗留代码 | 逐字符微光渲染 |
| 测试文件 | 单元测试 |

### 源码位置

```
src/components/Spinner/
├── ShimmerChar.tsx      # 本文件
├── GlimmerMessage.tsx   # 功能类似的整行处理版本
└── index.ts             # 模块导出
```

---

## 依赖与外部交互

### 依赖关系图

```
ShimmerChar.tsx
    ├─ import { Text } from '../../ink.js'
    │       └─ Text 组件渲染带颜色文本
    │
    └─ import type { Theme } from '../../utils/theme.js'
            └─ Theme 类型用于颜色键类型检查
```

### 与 GlimmerMessage 的对比

| 特性 | ShimmerChar | GlimmerMessage |
|-----|-------------|----------------|
| 粒度 | 单字符 | 整行分段 |
| 实现复杂度 | 简单 | 复杂 |
| 性能（长消息） | 较差（每字符一组件） | 较好（三段式渲染） |
| Unicode 处理 | 依赖父组件 | 内置字素分割 |
| 颜色过渡 | 硬切换 | 支持平滑插值 |
| 停滞警告 | 不支持 | 支持 |

---

## 风险、边界与改进建议

### 已知风险

1. **可能未被使用**
   - 从代码库搜索来看，`ShimmerChar` 的导出在 `index.ts` 中
   - 但 `GlimmerMessage` 提供了更优的整行处理方案
   - 需要确认是否还有实际调用方

2. **索引与字素不匹配**
   - `index` 参数假设是"字符索引"
   - 但对于多字节字符（如 emoji），字节索引 ≠ 视觉字符索引
   - 可能导致微光位置偏移

### 边界情况

| 场景 | 行为 |
|-----|------|
| `glimmerIndex = -100` (停滞状态) | 无字符高亮 |
| `index = glimmerIndex` | 使用 shimmerColor |
| `index = glimmerIndex ± 1` | 使用 shimmerColor |
| 空字符 | 正常渲染空 Text |
| 多字节字符 | 按单个索引处理（可能有问题） |

### 改进建议

1. **使用审查**
   - 全局搜索确认是否还有调用方
   - 如无调用方，考虑标记为废弃或移除

2. **Unicode 支持**
   - 如需继续使用，应集成 `getGraphemeSegmenter`
   - 或使用 `GlimmerMessage` 的完全替代

3. **性能优化**
   - 如果必须逐字符渲染，考虑使用 `React.memo`
   - 或实现虚拟化只渲染可见字符

4. **功能合并**
   - 考虑将功能合并到 `GlimmerMessage`
   - 通过配置参数选择逐字符或分段渲染

---

## 附录：源码与编译后对比

### 原始源码（约 20 行）

```typescript
import * as React from 'react'
import { Text } from '../../ink.js'
import type { Theme } from '../../utils/theme.js'

type Props = {
  char: string
  index: number
  glimmerIndex: number
  messageColor: keyof Theme
  shimmerColor: keyof Theme
}

export function ShimmerChar({
  char,
  index,
  glimmerIndex,
  messageColor,
  shimmerColor,
}: Props): React.ReactNode {
  const isHighlighted = index === glimmerIndex
  const isNearHighlight = Math.abs(index - glimmerIndex) === 1
  const shouldUseShimmer = isHighlighted || isNearHighlight

  return (
    <Text color={shouldUseShimmer ? shimmerColor : messageColor}>{char}</Text>
  )
}
```

### 编译后特点

- 增加 React Compiler 缓存机制
- 条件表达式提取为变量 `t1`
- 渲染结果缓存为 `t2`
- 添加 source map 注释

### 使用示例

```typescript
// 假设消息为 "Hello"
// glimmerIndex = 2（指向 'l'）

<ShimmerChar char="H" index={0} glimmerIndex={2} messageColor="text" shimmerColor="claudeShimmer" />
// 输出: <Text color="text">H</Text>

<ShimmerChar char="e" index={1} glimmerIndex={2} messageColor="text" shimmerColor="claudeShimmer" />
// 输出: <Text color="claudeShimmer">e</Text> (邻近高亮)

<ShimmerChar char="l" index={2} glimmerIndex={2} messageColor="text" shimmerColor="claudeShimmer" />
// 输出: <Text color="claudeShimmer">l</Text> (精确高亮)

<ShimmerChar char="l" index={3} glimmerIndex={2} messageColor="text" shimmerColor="claudeShimmer" />
// 输出: <Text color="claudeShimmer">l</Text> (邻近高亮)

<ShimmerChar char="o" index={4} glimmerIndex={2} messageColor="text" shimmerColor="claudeShimmer" />
// 输出: <Text color="text">o</Text>
```
