# schemas.ts 研究文档

## 场景与职责

schemas.ts 是 LSPTool 的输入验证和类型定义模块，使用 Zod 库定义所有 LSP 操作的输入参数结构。它提供了运行时类型验证和静态 TypeScript 类型推导的双重保障。

**主要职责：**
- 定义 9 种 LSP 操作的输入参数 schema
- 提供 discriminated union 进行精确的类型区分
- 导出 TypeScript 类型供其他模块使用
- 提供类型守卫函数进行运行时类型检查

**使用场景：**
- 验证用户/AI 输入的工具参数
- 提供类型安全的代码提示
- 生成工具调用的 JSON Schema

---

## 功能点目的

### 1. Discriminated Union Schema

使用 Zod 的 `discriminatedUnion` 基于 `operation` 字段区分不同操作类型：

```typescript
export const lspToolInputSchema = lazySchema(() => {
  // 定义 9 个操作的 schema...
  
  return z.discriminatedUnion('operation', [
    goToDefinitionSchema,
    findReferencesSchema,
    hoverSchema,
    documentSymbolSchema,
    workspaceSymbolSchema,
    goToImplementationSchema,
    prepareCallHierarchySchema,
    incomingCallsSchema,
    outgoingCallsSchema,
  ])
})
```

**优势：**
- 精确的类型推断
- 更好的错误信息（指出具体哪个操作的哪个字段无效）
- 运行时类型安全

### 2. 统一的参数结构

所有操作共享相同的参数结构：

```typescript
{
  operation: 'operationName',  // 操作类型标识
  filePath: string,            // 文件路径（绝对或相对）
  line: number,                // 行号（1-based）
  character: number            // 字符位置（1-based）
}
```

### 3. 参数验证规则

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `operation` | literal | 9 个固定值之一 | 操作类型 |
| `filePath` | string | 必填 | 目标文件路径 |
| `line` | number | int, positive | 1-based 行号 |
| `character` | number | int, positive | 1-based 字符位置 |

### 4. 类型守卫

```typescript
export function isValidLSPOperation(operation: string): operation is LSPToolInput['operation']
```

用于运行时检查字符串是否为有效的 LSP 操作名称。

---

## 具体技术实现

### Lazy Schema 模式

```typescript
import { lazySchema } from '../../utils/lazySchema.js'

export const lspToolInputSchema = lazySchema(() => {
  // schema 定义在函数内部，延迟执行
})
```

**目的：**
- 延迟 Zod schema 构建到首次访问时
- 避免模块加载时的性能开销
- 解决潜在的循环依赖问题

### 单个操作 Schema 定义

以 `goToDefinition` 为例：

```typescript
const goToDefinitionSchema = z.strictObject({
  operation: z.literal('goToDefinition'),
  filePath: z.string().describe('The absolute or relative path to the file'),
  line: z.number()
    .int()
    .positive()
    .describe('The line number (1-based, as shown in editors)'),
  character: z.number()
    .int()
    .positive()
    .describe('The character offset (1-based, as shown in editors)'),
})
```

**使用 `strictObject`：**
- 拒绝未定义的额外属性
- 防止拼写错误导致的静默失败

### Discriminated Union 构建

```typescript
return z.discriminatedUnion('operation', [
  goToDefinitionSchema,      // operation: 'goToDefinition'
  findReferencesSchema,      // operation: 'findReferences'
  hoverSchema,               // operation: 'hover'
  documentSymbolSchema,      // operation: 'documentSymbol'
  workspaceSymbolSchema,     // operation: 'workspaceSymbol'
  goToImplementationSchema,  // operation: 'goToImplementation'
  prepareCallHierarchySchema,// operation: 'prepareCallHierarchy'
  incomingCallsSchema,       // operation: 'incomingCalls'
  outgoingCallsSchema,       // operation: 'outgoingCalls'
])
```

### 类型导出

```typescript
export type LSPToolInput = z.infer<ReturnType<typeof lspToolInputSchema>>
```

推导出的类型：

```typescript
type LSPToolInput = 
  | { operation: 'goToDefinition', filePath: string, line: number, character: number }
  | { operation: 'findReferences', filePath: string, line: number, character: number }
  | { operation: 'hover', filePath: string, line: number, character: number }
  | { operation: 'documentSymbol', filePath: string, line: number, character: number }
  | { operation: 'workspaceSymbol', filePath: string, line: number, character: number }
  | { operation: 'goToImplementation', filePath: string, line: number, character: number }
  | { operation: 'prepareCallHierarchy', filePath: string, line: number, character: number }
  | { operation: 'incomingCalls', filePath: string, line: number, character: number }
  | { operation: 'outgoingCalls', filePath: string, line: number, character: number }
```

### 类型守卫实现

```typescript
export function isValidLSPOperation(
  operation: string
): operation is LSPToolInput['operation'] {
  return [
    'goToDefinition',
    'findReferences',
    'hover',
    'documentSymbol',
    'workspaceSymbol',
    'goToImplementation',
    'prepareCallHierarchy',
    'incomingCalls',
    'outgoingCalls',
  ].includes(operation)
}
```

---

## 关键代码路径与文件引用

### 内部依赖

```typescript
import { z } from 'zod/v4'
import { lazySchema } from '../../utils/lazySchema.js'
```

### 导出项

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `lspToolInputSchema` | `() => ZodSchema` | 输入验证 schema（lazy） |
| `LSPToolInput` | `type` | TypeScript 输入类型 |
| `isValidLSPOperation` | `function` | 类型守卫函数 |

### 被引用位置

| 文件 | 引用内容 | 用途 |
|------|----------|------|
| `LSPTool.ts` | `lspToolInputSchema` | `validateInput` 中的精确验证 |

### 在 LSPTool.ts 中的使用

```typescript
import { lspToolInputSchema } from './schemas.js'

async validateInput(input: Input): Promise<ValidationResult> {
  // 使用 discriminated union 进行精确验证
  const parseResult = lspToolInputSchema().safeParse(input)
  if (!parseResult.success) {
    return {
      result: false,
      message: `Invalid input: ${parseResult.error.message}`,
      errorCode: 3,
    }
  }
  // ... 继续其他验证
}
```

---

## 依赖与外部交互

### Zod v4

使用 Zod v4 进行 schema 定义和验证：

```typescript
import { z } from 'zod/v4'
```

**使用的 Zod API：**
- `z.strictObject()`: 严格对象（无额外属性）
- `z.literal()`: 字面量类型
- `z.string()`: 字符串类型
- `z.number()`: 数字类型
- `z.int()`: 整数约束
- `z.positive()`: 正数约束
- `z.describe()`: 字段描述（用于生成文档）
- `z.discriminatedUnion()`: 可区分联合类型
- `.safeParse()`: 安全解析（不抛出异常）

### lazySchema 工具

```typescript
import { lazySchema } from '../../utils/lazySchema.js'
```

实现：

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **参数冗余**
   - 所有操作都需要 `line` 和 `character`
   - 某些操作（如 `workspaceSymbol`）理论上不需要位置信息
   - 现状：统一要求简化实现，实际可传任意值

2. **Schema 重复**
   - 9 个操作的结构几乎完全相同
   - 维护时需要修改多处
   - 缓解：结构简单，修改频率低

3. **Zod v4 依赖**
   - 使用特定版本的 Zod
   - 升级可能需要调整

### 边界情况

| 场景 | Zod 行为 |
|------|----------|
| 缺少必需字段 | 验证失败，返回具体缺失字段 |
| 额外字段 | `strictObject` 拒绝，验证失败 |
| 类型不匹配 | 验证失败，返回类型错误 |
| 负数行号 | `positive()` 约束拒绝 |
| 小数行号 | `int()` 约束拒绝 |
| 无效 operation | discriminatedUnion 无法匹配，验证失败 |

### 改进建议

1. **操作分类优化**
   - 当前：所有操作统一参数
   - 建议：按是否需要位置分组

   ```typescript
   // 位置相关操作
   const positionBasedSchema = z.object({
     operation: z.enum(['goToDefinition', 'findReferences', ...]),
     filePath: z.string(),
     line: z.number().int().positive(),
     character: z.number().int().positive(),
   })
   
   // 文件级操作
   const fileBasedSchema = z.object({
     operation: z.enum(['documentSymbol', 'workspaceSymbol']),
     filePath: z.string(),
   })
   ```

2. **Schema 复用**
   - 当前：每个操作独立定义
   - 建议：提取公共部分

   ```typescript
   const basePositionSchema = {
     filePath: z.string().describe('...'),
     line: z.number().int().positive().describe('...'),
     character: z.number().int().positive().describe('...'),
   }
   
   const goToDefinitionSchema = z.strictObject({
     operation: z.literal('goToDefinition'),
     ...basePositionSchema,
   })
   ```

3. **更精确的文件路径验证**
   - 当前：仅验证为 string
   - 建议：添加路径格式验证

   ```typescript
   filePath: z.string()
     .min(1, 'File path cannot be empty')
     .refine(
       path => !path.includes('\0'),
       'File path cannot contain null bytes'
     )
   ```

4. **范围限制**
   - 当前：仅验证为正整数
   - 建议：添加合理范围限制

   ```typescript
   line: z.number().int().positive().max(999999),
   character: z.number().int().positive().max(99999),
   ```

5. **默认值支持**
   - 当前：无默认值
   - 建议：为某些操作添加默认值

   ```typescript
   line: z.number().int().positive().default(1),
   character: z.number().int().positive().default(1),
   ```

### 测试建议

- 测试每个操作的有效输入
- 测试无效 operation 值
- 测试类型不匹配（string 代替 number）
- 测试边界值（0, 负数, 极大值）
- 测试额外字段拒绝
- 测试缺少必需字段

### 维护注意事项

添加新操作时：
1. 在 `lspToolInputSchema` 中添加新 schema
2. 更新 `isValidLSPOperation` 中的数组
3. 同步更新 `prompt.ts` 的描述
4. 在 `LSPTool.ts` 中添加操作处理逻辑
5. 在 `formatters.ts` 中添加格式化函数
6. 在 `UI.tsx` 中添加操作标签
