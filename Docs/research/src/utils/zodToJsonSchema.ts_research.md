# zodToJsonSchema.ts 研究文档

## 场景与职责

`zodToJsonSchema.ts` 提供将 Zod v4 模式转换为 JSON Schema 的功能。这是 Claude Code CLI 工具系统的重要组成部分，用于将 TypeScript 运行时验证模式转换为 API 可用的 JSON Schema。

**核心使用场景：**
- `toolToAPISchema()` 为每个工具生成 API 请求用的 JSON Schema
- 每轮 API 请求需要处理约 60-250 个工具的 Schema 转换
- 通过缓存优化性能（工具模式使用 `lazySchema()` 包装，保证每会话相同引用）

## 功能点目的

### Zod 到 JSON Schema 转换 (`zodToJsonSchema`)
- **目的**：将 Zod v4 模式对象转换为 JSON Schema 格式
- **性能优化**：使用 WeakMap 缓存，避免重复转换相同模式
- **缓存策略**：按 ZodTypeAny 实例引用缓存（依赖 `lazySchema()` 保证相同引用）

## 具体技术实现

### 核心代码

```typescript
/**
 * Converts Zod v4 schemas to JSON Schema using native toJSONSchema.
 */

import { toJSONSchema, type ZodTypeAny } from 'zod/v4'

export type JsonSchema7Type = Record<string, unknown>

// toolToAPISchema() runs this for every tool on every API request (~60-250
// times/turn). Tool schemas are wrapped with lazySchema() which guarantees the
// same ZodTypeAny reference per session, so we can cache by identity.
const cache = new WeakMap<ZodTypeAny, JsonSchema7Type>()

/**
 * Converts a Zod v4 schema to JSON Schema format.
 */
export function zodToJsonSchema(schema: ZodTypeAny): JsonSchema7Type {
  const hit = cache.get(schema)
  if (hit) return hit
  const result = toJSONSchema(schema) as JsonSchema7Type
  cache.set(schema, result)
  return result
}
```

### 缓存机制

```
zodToJsonSchema(schema) → JsonSchema7Type
├── cache.get(schema) 
│   └── 命中 → 返回缓存结果
└── 未命中
    ├── toJSONSchema(schema) → 执行转换
    ├── cache.set(schema, result) → 存入缓存
    └── 返回结果
```

**缓存特点：**
- 使用 `WeakMap`，键是 ZodTypeAny 对象引用
- 不阻止垃圾回收（当模式对象不再被引用时，缓存项自动释放）
- 依赖 `lazySchema()` 保证同一会话中相同工具使用相同 ZodTypeAny 实例

### 类型定义

```typescript
// Zod v4 类型
import { type ZodTypeAny } from 'zod/v4'

// JSON Schema 7 类型（宽松定义）
export type JsonSchema7Type = Record<string, unknown>
```

## 关键代码路径与文件引用

### 导出类型和函数
- `src/utils/zodToJsonSchema.ts:7` - `JsonSchema7Type` 类型
- `src/utils/zodToJsonSchema.ts:17` - `zodToJsonSchema(schema)` 函数

### 依赖
| 依赖 | 用途 |
|------|------|
| `zod/v4` | `toJSONSchema` 函数和 `ZodTypeAny` 类型 |

### 调用方（预期）
- `toolToAPISchema()` - 工具 API Schema 生成
- 其他需要将 Zod 模式转换为 JSON Schema 的场景

## 依赖与外部交互

### 外部依赖
```typescript
import { toJSONSchema, type ZodTypeAny } from 'zod/v4'
```

### Zod v4 特性
- `toJSONSchema` 是 Zod v4 新增的原生方法
- 替代了之前需要 `zod-to-json-schema` 第三方包的方式
- 提供更好的类型安全和性能

### 内部依赖
无内部依赖。

## 风险、边界与改进建议

### 已知风险

1. **Zod v4 依赖**
   - 代码明确依赖 Zod v4 的 `toJSONSchema` 方法
   - 如果使用 Zod v3，会缺少此方法
   - package.json 需要确保 Zod v4 版本

2. **类型断言**
   - `toJSONSchema(schema) as JsonSchema7Type` 使用类型断言
   - 如果 Zod 返回的格式与预期不符，类型安全无法保证

3. **WeakMap 缓存限制**
   - 缓存只在同一会话内有效（依赖 `lazySchema()`）
   - 新会话会创建新的 ZodTypeAny 实例，需要重新转换
   - 如果 `lazySchema()` 不保证相同引用，缓存会失效

4. **内存使用**
   - 虽然使用 WeakMap，但在会话期间所有工具模式都会被缓存
   - 如果工具数量非常大，会占用较多内存

### 边界情况

1. **复杂模式**
   - 某些复杂的 Zod 模式（如自定义 refinements）可能无法完全转换为 JSON Schema
   - 行为取决于 Zod v4 的 `toJSONSchema` 实现

2. **循环引用**
   - 如果 Zod 模式包含循环引用，`toJSONSchema` 的行为取决于 Zod 实现
   - 可能抛出错误或生成引用格式的 Schema

3. **空模式**
   - 传入空对象或简单模式时，应正常返回对应的 JSON Schema

4. **并发调用**
   - 当前实现不是线程安全的（虽然 JavaScript 单线程，但异步上下文可能交错）
   - 如果两个调用同时传入相同的 schema（未缓存），可能执行两次转换

### 改进建议

1. **版本检查**
   ```typescript
   import { toJSONSchema, type ZodTypeAny, version } from 'zod/v4'
   
   if (!version || !version.startsWith('4.')) {
     throw new Error('zodToJsonSchema requires Zod v4')
   }
   ```

2. **并发保护**
   ```typescript
   const pending = new Map<ZodTypeAny, Promise<JsonSchema7Type>>()
   
   export async function zodToJsonSchemaAsync(schema: ZodTypeAny): Promise<JsonSchema7Type> {
     const cached = cache.get(schema)
     if (cached) return cached
     
     const pendingPromise = pending.get(schema)
     if (pendingPromise) return pendingPromise
     
     const promise = Promise.resolve().then(() => {
       const result = toJSONSchema(schema) as JsonSchema7Type
       cache.set(schema, result)
       pending.delete(schema)
       return result
     })
     
     pending.set(schema, promise)
     return promise
   }
   ```

3. **缓存统计**
   ```typescript
   export function getCacheStats(): { hits: number; misses: number; size: number } {
     return { ...stats, size: cache.size }
   }
   ```

4. **错误处理增强**
   ```typescript
   export class ZodToJsonSchemaError extends Error {
     constructor(message: string, public readonly schema: ZodTypeAny) {
       super(message)
     }
   }
   
   export function zodToJsonSchema(schema: ZodTypeAny): JsonSchema7Type {
     try {
       // ... 现有实现
     } catch (error) {
       throw new ZodToJsonSchemaError(
         `Failed to convert Zod schema to JSON Schema: ${error}`,
         schema
       )
     }
   }
   ```

5. **Schema 验证**
   - 添加对生成 JSON Schema 的验证
   - 确保输出符合 JSON Schema 规范

6. **性能监控**
   - 添加转换时间监控
   - 在调试模式下记录缓存命中率

7. **文档完善**
   - 添加使用示例
   - 说明缓存行为和限制
   - 记录与 Zod v3 的不兼容性
