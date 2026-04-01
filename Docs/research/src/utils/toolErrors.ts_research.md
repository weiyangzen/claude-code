# toolErrors.ts 研究文档

## 场景与职责

`toolErrors.ts` 是 Claude Code CLI 的工具错误处理和格式化模块，专门负责将工具执行中的各种错误（Shell 错误、Zod 验证错误、中断错误等）转换为人类可读且 LLM 友好的错误消息。这是工具执行管道中错误处理的关键环节。

主要使用场景：
1. **Shell 工具错误格式化**：将 ShellError 转换为包含退出码、stderr、stdout 的详细错误消息
2. **Zod 验证错误格式化**：将 Zod 验证失败转换为清晰的参数错误描述
3. **错误截断**：对超长错误消息进行智能截断，避免淹没上下文
4. **MCP/工具执行错误**：通过 `services/tools/toolExecution.ts` 和 `toolHooks.ts` 处理工具执行错误

## 功能点目的

### 1. 统一的错误格式化 (`formatError`)
- **输入**：任意错误值（Error 对象、字符串、未知类型）
- **输出**：格式化的错误字符串，适合显示给用户和发送给 LLM
- **特殊处理**：AbortError、超长消息截断

### 2. Shell 错误详细提取 (`getErrorParts`)
- 提取退出码、中断状态、stderr、stdout
- 支持自定义 ShellError 类和通用 Error 对象

### 3. Zod 验证错误友好化 (`formatZodValidationError`)
- 将 Zod 的详细错误报告转换为简洁的问题列表
- 分类：缺失参数、意外参数、类型不匹配

### 4. 超长错误截断
- 超过 10,000 字符的错误消息会被截断
- 保留开头和结尾各 5,000 字符，中间显示省略信息

## 具体技术实现

### 核心函数

#### `formatError`

```typescript
export function formatError(error: unknown): string
```

**处理流程**：

1. **AbortError 特殊处理**
   ```typescript
   if (error instanceof AbortError) {
     return error.message || INTERRUPT_MESSAGE_FOR_TOOL_USE
   }
   ```

2. **非 Error 类型处理**
   ```typescript
   if (!(error instanceof Error)) {
     return String(error)
   }
   ```

3. **标准 Error 处理**
   ```typescript
   const parts = getErrorParts(error)
   const fullMessage = parts.filter(Boolean).join('\n').trim()
   ```

4. **超长截断**
   ```typescript
   if (fullMessage.length <= 10000) {
     return fullMessage
   }
   const halfLength = 5000
   const start = fullMessage.slice(0, halfLength)
   const end = fullMessage.slice(-halfLength)
   return `${start}\n\n... [${fullMessage.length - 10000} characters truncated] ...\n\n${end}`
   ```

#### `getErrorParts`

```typescript
export function getErrorParts(error: Error): string[]
```

**ShellError 处理**：
```typescript
if (error instanceof ShellError) {
  return [
    `Exit code ${error.code}`,
    error.interrupted ? INTERRUPT_MESSAGE_FOR_TOOL_USE : '',
    error.stderr,
    error.stdout,
  ]
}
```

**通用 Error 处理**：
```typescript
const parts = [error.message]
if ('stderr' in error && typeof error.stderr === 'string') {
  parts.push(error.stderr)
}
if ('stdout' in error && typeof error.stdout === 'string') {
  parts.push(error.stdout)
}
return parts
```

#### `formatZodValidationError`

```typescript
export function formatZodValidationError(
  toolName: string,
  error: ZodError,
): string
```

**错误分类处理**：

1. **缺失参数**（`invalid_type` + `received undefined`）
   ```typescript
   const missingParams = error.issues
     .filter(err => 
       err.code === 'invalid_type' && 
       err.message.includes('received undefined')
     )
     .map(err => formatValidationPath(err.path))
   // 输出："The required parameter `paramName` is missing"
   ```

2. **意外参数**（`unrecognized_keys`）
   ```typescript
   const unexpectedParams = error.issues
     .filter(err => err.code === 'unrecognized_keys')
     .flatMap(err => err.keys)
   // 输出："An unexpected parameter `paramName` was provided"
   ```

3. **类型不匹配**（其他 `invalid_type`）
   ```typescript
   const typeMismatchParams = error.issues
     .filter(err => 
       err.code === 'invalid_type' && 
       !err.message.includes('received undefined')
     )
     .map(err => ({
       param: formatValidationPath(err.path),
       expected: (err as { expected: string }).expected,
       received: err.message.match(/received (\w+)/)?.[1] || 'unknown',
     }))
   // 输出："The parameter `paramName` type is expected as `string` but provided as `number`"
   ```

**路径格式化** (`formatValidationPath`)：
```typescript
// 输入：['todos', 0, 'activeForm']
// 输出：'todos[0].activeForm'

function formatValidationPath(path: PropertyKey[]): string {
  return path.reduce((acc, segment, index) => {
    const segmentStr = String(segment)
    if (typeof segment === 'number') {
      return `${String(acc)}[${segmentStr}]`
    }
    return index === 0 ? segmentStr : `${String(acc)}.${segmentStr}`
  }, '')
}
```

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/services/tools/toolExecution.ts` | 工具执行错误格式化 |
| `src/services/tools/toolHooks.ts` | 工具钩子错误处理 |
| `src/entrypoints/mcp.ts` | MCP 错误处理 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `zod/v4` | `ZodError` 类型 |
| `./errors.js` | `AbortError`, `ShellError` |
| `./messages.js` | `INTERRUPT_MESSAGE_FOR_TOOL_USE` |

## 依赖与外部交互

### 与错误类型系统的集成

**AbortError**：
```typescript
export class AbortError extends Error {
  constructor(message?: string) {
    super(message)
    this.name = 'AbortError'
  }
}
```

**ShellError**：
```typescript
export class ShellError extends Error {
  constructor(
    public readonly stdout: string,
    public readonly stderr: string,
    public readonly code: number,
    public readonly interrupted: boolean,
  ) {
    super('Shell command failed')
    this.name = 'ShellError'
  }
}
```

### 与工具执行管道的集成

```typescript
// toolExecution.ts 伪代码
try {
  const result = await executeTool(tool, input)
  return result
} catch (error) {
  const formattedError = formatError(error)
  // 将 formattedError 作为 tool_result 返回给 LLM
}
```

### 与 Zod 验证的集成

```typescript
// 工具输入验证
try {
  const validatedInput = toolSchema.parse(rawInput)
} catch (error) {
  if (error instanceof ZodError) {
    const message = formatZodValidationError(toolName, error)
    throw new Error(message)
  }
}
```

## 风险、边界与改进建议

### 潜在风险

1. **信息泄露风险**
   - 错误消息可能包含敏感信息（文件路径、环境变量等）
   - 当前直接返回给 LLM，无过滤机制

2. **截断信息丢失**
   - 10,000 字符截断可能丢失关键错误信息
   - 中间部分可能包含最重要的错误详情

3. **Zod 错误类型假设**
   ```typescript
   const typeErr = err as { expected: string }
   ```
   - 类型断言假设存在 `expected` 字段
   - 如果 Zod 版本变化，可能运行时错误

4. **stdout/stderr 编码**
   - 假设字符串编码，二进制输出可能损坏
   - 大输出可能占用大量内存

### 边界条件

| 场景 | 行为 |
|-----|------|
| `null` 或 `undefined` | `String(error)` → `"null"` / `"undefined"` |
| 数字/布尔值 | `String(error)` → `"123"` / `"true"` |
| 空错误消息 | 返回 `"Command failed with no output"` |
| 恰好 10,000 字符 | 不截断 |
| 10,001 字符 | 截断为 5,000 + 省略信息 + 5,000 |
| 无 path 的 Zod 错误 | `formatValidationPath([])` → `""` |
| Zod 错误无 issues | 返回原始 `error.message` |

### 改进建议

1. **敏感信息过滤**
   ```typescript
   export function formatError(
     error: unknown,
     options?: { sanitize?: boolean }
   ): string
   
   function sanitizeErrorMessage(message: string): string {
     // 移除或替换敏感模式
     return message
       .replace(/\/home\/[^/]+/g, '/home/<user>')
       .replace(/token=[a-zA-Z0-9]+/g, 'token=<redacted>')
   }
   ```

2. **智能截断**
   ```typescript
   // 基于内容重要性而非固定长度
   function smartTruncate(message: string, maxLength: number): string {
     if (message.length <= maxLength) return message
     
     // 尝试在句子边界截断
     const sentences = message.split(/(?<=[.!?])\s+/)
     // 保留开头和结尾的完整句子
   }
   ```

3. **结构化错误输出**
   ```typescript
   export interface FormattedError {
     summary: string
     details: {
       exitCode?: number
       stderr?: string
       stdout?: string
       validationErrors?: ValidationError[]
     }
     truncated: boolean
   }
   ```

4. **多语言支持**
   ```typescript
   // 支持错误消息的本地化
   export function formatError(error: unknown, locale: string = 'en'): string
   ```

5. **错误分类**
   ```typescript
   export enum ErrorCategory {
     USER_ERROR,      // 用户输入错误
     SYSTEM_ERROR,    // 系统/环境错误
     NETWORK_ERROR,   // 网络错误
     PERMISSION_ERROR,// 权限错误
     TIMEOUT_ERROR,   // 超时错误
   }
   
   export function categorizeError(error: unknown): ErrorCategory
   ```

6. **Zod 错误增强**
   ```typescript
   // 提供更多上下文信息
   if (typeMismatchParams.length > 0) {
     errorParts.push(
       `Type mismatches:\n` +
       typeMismatchParams.map(p => 
         `- \`${p.param}\`: expected \`${p.expected}\`, got \`${p.received}\`\n` +
         `  Example: ${generateExample(p.expected)}`
       ).join('\n')
     )
   }
   ```

### 测试建议

应覆盖以下场景：
- 各种错误类型的格式化（ShellError、AbortError、ZodError、普通 Error）
- 超长错误消息的截断
- Zod 错误的各种 issue 类型
- 嵌套路径的格式化
- 空/缺失字段的处理
- 非 Error 类型的输入（字符串、数字、null 等）
