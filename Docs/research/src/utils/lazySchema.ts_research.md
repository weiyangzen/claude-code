# src/utils/lazySchema.ts 研究文档

## 场景与职责

`lazySchema.ts` 是一个微型的性能优化工具，用于延迟 Zod schema 的构造时间。在大型 TypeScript 项目中，模块加载时构造复杂的 Zod schema 会增加启动开销（因为 `z.object({...})` 等调用会创建大量对象和验证器）。通过 `lazySchema`，可以将 schema 的构造从**模块初始化时**推迟到**第一次实际使用时**。

该模块在代码库中被广泛使用，任何定义了模块级 Zod schema 且希望优化启动性能的地方都可以使用它。

调用方包括：
- `src/hooks/useIdeAtMentioned.ts`
- `src/bridge/pollConfig.ts`
- `src/bridge/bridgePointer.ts`
- `src/bridge/envLessBridgeConfig.ts`
- `src/bridge/inboundAttachments.ts`
- `src/hooks/useIdeSelection.ts`
- `src/hooks/useIdeLogging.ts`
- `src/hooks/usePromptsFromClaudeInChrome.tsx`
- `src/server/types.ts`
- `src/types/hooks.ts`
- `src/utils/cronTasksLock.ts`
- `src/entrypoints/sandboxTypes.ts`
- `src/utils/cronJitterConfig.ts`
- `src/utils/sessionTitle.ts`
- `src/keybindings/schema.ts`
- `src/utils/tasks.ts`
- `src/utils/teammateMailbox.ts`
- `src/schemas/hooks.ts`
- `src/commands/brief.ts`

## 功能点目的

### `lazySchema`
单一导出函数，接受一个 schema 工厂函数，返回一个无参函数。第一次调用返回的函数时执行工厂并缓存结果，后续调用直接返回缓存值。

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

## 具体技术实现

### 闭包缓存模式
```typescript
let cached: T | undefined
return () => (cached ??= factory())
```

使用 `??=`（Nullish Coalescing Assignment）运算符：
- 若 `cached` 为 `undefined` 或 `null`，调用 `factory()` 并将结果赋值给 `cached`。
- 否则直接返回 `cached`。

### 使用示例
```typescript
import { lazySchema } from './lazySchema.js'
import { z } from 'zod/v4'

const getMySchema = lazySchema(() => z.object({
  name: z.string(),
  age: z.number(),
}))

// 模块加载时，z.object 不会被调用
// 第一次使用时才构造 schema
const mySchema = getMySchema()
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/lazySchema.ts:5-8` | `lazySchema` 工厂函数 |

## 依赖与外部交互

### 依赖
- 无任何内部或外部依赖。

### 调用方
- `src/hooks/useIdeAtMentioned.ts`
- `src/bridge/pollConfig.ts`
- `src/bridge/bridgePointer.ts`
- `src/bridge/envLessBridgeConfig.ts`
- `src/bridge/inboundAttachments.ts`
- `src/hooks/useIdeSelection.ts`
- `src/hooks/useIdeLogging.ts`
- `src/hooks/usePromptsFromClaudeInChrome.tsx`
- `src/server/types.ts`
- `src/types/hooks.ts`
- `src/utils/cronTasksLock.ts`
- `src/entrypoints/sandboxTypes.ts`
- `src/utils/cronJitterConfig.ts`
- `src/utils/sessionTitle.ts`
- `src/keybindings/schema.ts`
- `src/utils/tasks.ts`
- `src/utils/teammateMailbox.ts`
- `src/schemas/hooks.ts`
- `src/commands/brief.ts`

## 风险、边界与改进建议

### 风险与边界
1. **工厂函数中的副作用**：如果 `factory` 不是纯函数（例如依赖了在模块加载后才初始化的全局状态），第一次调用 `lazySchema` 返回的函数时可能产生不可预期的结果。模块注释已说明这是"defer Zod schema construction"，暗示工厂应该是纯的 schema 构造。
2. **无法处理 `null` 作为合法缓存值**：`cached` 的类型是 `T | undefined`，使用 `??=` 判断。如果 `factory()` 的合法返回值是 `undefined`，则每次调用都会重新执行工厂。不过对于 Zod schema 来说，返回值永远不会是 `undefined`，所以这不是实际问题。
3. **无线程安全顾虑**：在单线程的 JavaScript 运行时中，闭包缓存是天然线程安全的。
4. **调试困难**：延迟构造意味着如果 schema 构造时抛出错误，堆栈跟踪会指向第一次使用 schema 的代码位置，而不是模块加载位置。这有时会让开发者困惑，但注释和明确的工厂函数命名可以缓解。

### 改进建议
1. **添加错误包装**：在工厂函数调用时 catch 错误并重新抛出带有 "lazy schema initialization failed" 前缀的错误，帮助开发者快速定位是 schema 构造失败。
2. **支持参数化工厂**：当前 `lazySchema` 只支持无参工厂。可以考虑添加 `lazySchemaWithArgs` 变体，支持带参数的 memoized schema 构造（虽然这与 Zod 的 `.partial()`、`.pick()` 等链式 API 相比需求不大）。
3. **性能基准测试**：在 CI 中添加一个启动时间基准测试，对比使用 `lazySchema` 前后的模块加载时间，确保优化措施确实有效。
4. **与 ESLint 规则结合**：可以考虑添加一个自定义 ESLint 规则，鼓励开发者在模块级定义复杂 Zod schema 时使用 `lazySchema`，避免启动性能退化。
