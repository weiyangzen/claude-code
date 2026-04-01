# objectGroupBy.ts 研究文档

## 场景与职责

本模块提供了 `Object.groupBy` 的兼容实现。核心职责包括：

1. **分组功能**：将可迭代对象按指定键选择器函数分组
2. **ECMAScript 兼容性**：实现 TC39 提案 `Object.groupBy` 的 polyfill
3. **类型安全**：提供完整的 TypeScript 类型支持

该模块是数组/对象处理的基础工具，用于消息分组、数据统计等场景。

## 功能点目的

### `objectGroupBy()` - 对象分组
- **目的**：将元素按指定键分组为对象
- **标准**：实现 ECMAScript 提案（https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.groupby）
- **与 `Map.groupBy` 区别**：返回普通对象而非 Map

## 具体技术实现

### 算法实现

```typescript
export function objectGroupBy<T, K extends PropertyKey>(
  items: Iterable<T>,
  keySelector: (item: T, index: number) => K,
): Partial<Record<K, T[]>> {
  const result = Object.create(null) as Partial<Record<K, T[]>>
  let index = 0
  for (const item of items) {
    const key = keySelector(item, index++)
    if (result[key] === undefined) {
      result[key] = []
    }
    result[key].push(item)
  }
  return result
}
```

### 关键设计决策

| 特性 | 实现 | 说明 |
|------|------|------|
| 原型 | `Object.create(null)` | 避免原型链污染（如 `toString` 键） |
| 键类型 | `PropertyKey` | 支持 string、number、symbol |
| 索引传递 | `index++` | 与 Array 方法一致，从 0 开始 |
| 空键处理 | `result[key] === undefined` | 显式检查 undefined，包含 null |

### 类型签名详解

```typescript
function objectGroupBy<T, K extends PropertyKey>(
  items: Iterable<T>,      // 输入：任意可迭代对象
  keySelector: (item: T, index: number) => K  // 键选择器函数
): Partial<Record<K, T[]>>  // 输出：部分记录（可能缺少某些键）
```

- `T`：元素类型
- `K`：键类型，约束为 `PropertyKey`（string | number | symbol）
- `Partial<Record<K, T[]>>`：返回对象，键为 K，值为 T 数组，可能部分键不存在

## 依赖与外部交互

### 直接依赖

无外部依赖（纯函数实现）。

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/utils/messageQueueManager.ts` | 消息队列分组管理 |

### 标准库替代

- **原生 `Object.groupBy`**：ES2024 新增，Node.js 21+ 支持
- **本模块**：提供向后兼容的 polyfill

## 风险、边界与改进建议

### 已知风险

1. **原型污染防护**
   - 缓解：使用 `Object.create(null)` 创建无原型对象
   - 防护键：`__proto__`、`constructor`、`prototype`

2. **Symbol 键**
   - 支持：函数签名允许 Symbol 键
   - 注意：JSON 序列化时会丢失 Symbol 键

3. **性能考虑**
   - 时间复杂度：O(n)，单次遍历
   - 空间复杂度：O(n)，存储分组结果

### 边界情况

| 场景 | 行为 |
|------|------|
| 空可迭代对象 | 返回空对象 `{}` |
| 所有元素相同键 | 返回单键对象，值为所有元素数组 |
| 所有元素不同键 | 返回多键对象，每个值为单元素数组 |
| 键选择器返回 null | 使用 `'null'` 作为键（字符串化） |
| 键选择器返回 undefined | 使用 `'undefined'` 作为键 |
| 键选择器抛出异常 | 异常传播，部分结果丢失 |
| 输入为 Generator | 正常消费，支持惰性求值 |

### 改进建议

1. **原生方法检测**
   - 当前：始终使用自定义实现
   - 建议：
     ```typescript
     export const objectGroupBy = Object.groupBy ?? polyfillImplementation
     ```

2. **Map 版本**
   - 建议：添加 `mapGroupBy` 实现 `Map.groupBy` polyfill
   - 用途：支持任意对象作为键

3. **惰性分组**
   - 当前：立即计算所有分组
   - 建议：提供生成器版本 `objectGroupByLazy`
   - 用途：大数据集流式处理

4. **聚合函数**
   - 建议：扩展支持自定义聚合（不仅数组收集）
   - 接口：`(acc: R, item: T) => R` 替代简单 push

5. **多键分组**
   - 建议：支持嵌套分组 `groupBy(items, [key1, key2])`
   - 输出：嵌套对象结构

6. **性能优化**
   - 建议：对于已知小键集，预分配数组容量
   - 考虑：使用 `Map` 内部实现，最后转换为对象

7. **不可变版本**
   - 建议：提供 `readonly` 返回类型版本
   - 用途：函数式编程风格

8. **文档和示例**
   - 建议：添加 JSDoc 示例
   - 内容：基本用法、复杂键选择器、与 lodash 对比
