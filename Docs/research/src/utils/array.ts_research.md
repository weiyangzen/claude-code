# array.ts 研究文档

## 场景与职责

`array.ts` 提供简单的数组工具函数，用于常见的数组操作。这是一个轻量级工具模块，提供函数式编程风格的数组处理功能。

## 功能点目的

### 1. 数组穿插 (`intersperse`)
- 在数组元素之间插入分隔符
- 常用于生成带分隔符的列表

### 2. 条件计数 (`count`)
- 统计满足条件的元素数量
- 替代 `filter().length` 的高效实现

### 3. 去重 (`uniq`)
- 使用 `Set` 去除数组重复元素
- 保持元素首次出现的顺序

## 具体技术实现

### 关键函数

```typescript
// 在数组元素之间插入分隔符
export function intersperse<A>(as: A[], separator: (index: number) => A): A[] {
  return as.flatMap((a, i) => (i ? [separator(i), a] : [a]))
}

// 统计满足条件的元素数量
export function count<T>(arr: readonly T[], pred: (x: T) => unknown): number {
  let n = 0
  for (const x of arr) n += +!!pred(x)
  return n
}

// 数组去重
export function uniq<T>(xs: Iterable<T>): T[] {
  return [...new Set(xs)]
}
```

### 实现细节

#### `intersperse`
- 使用 `flatMap` 实现，每个元素映射为 `[separator, element]` 或 `[element]`
- 第一个元素前不插入分隔符（`i ? ... : [a]`）
- 分隔符通过函数生成，可基于索引动态变化

#### `count`
- 使用 `+!!pred(x)` 将布尔值转换为 0/1
- 单次遍历，时间复杂度 O(n)
- 接受 `readonly` 数组，不修改原数组

#### `uniq`
- 利用 `Set` 自动去重特性
- 通过展开运算符转回数组
- 接受任意可迭代对象

## 关键代码路径与文件引用

### 本文件导出
- `intersperse<A>(as: A[], separator: (index: number) => A): A[]`
- `count<T>(arr: readonly T[], pred: (x: T) => unknown): number`
- `uniq<T>(xs: Iterable<T>): T[]`

### 调用方
该模块被广泛使用，主要调用方包括：
- `src/ink/*.ts` - Ink 渲染系统
- `src/components/*.tsx` - React 组件
- `src/skills/*.ts` - 技能系统
- `src/utils/*.ts` - 其他工具模块

## 依赖与外部交互

### 外部依赖
- 无外部 npm 依赖
- 仅使用 JavaScript 内置功能

## 风险、边界与改进建议

### 已知限制
1. **简单实现**：功能较为基础，复杂场景需使用 lodash 等库
2. **类型推断**：`count` 的谓词函数返回值使用 `unknown`，可能不够严格

### 边界条件
1. **空数组**：所有函数正确处理空数组
2. **单元素数组**：`intersperse` 不插入分隔符
3. **全重复数组**：`uniq` 返回单元素数组

### 性能特征
| 函数 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| `intersperse` | O(n) | O(n) |
| `count` | O(n) | O(1) |
| `uniq` | O(n) | O(n) |

### 改进建议
1. **添加更多函数**：`groupBy`, `partition`, `zip` 等
2. **惰性求值**：为大数据集提供生成器版本
3. **类型增强**：为 `count` 添加更严格的类型约束

### 测试建议
1. 测试空数组行为
2. 测试单元素数组行为
3. 测试大数据集性能
