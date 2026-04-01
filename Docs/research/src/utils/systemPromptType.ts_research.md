# systemPromptType.ts 研究文档

## 场景与职责

`systemPromptType.ts` 是 Claude Code 系统提示词（system prompt）的基础类型定义模块。该模块 intentionally 保持零依赖，使用 TypeScript 的品牌类型（branded type）模式为系统提示词数组提供类型安全，避免与其他字符串数组混淆。

## 功能点目的

### 品牌类型保护
- **问题**: 普通字符串数组 `string[]` 无法区分系统提示词和其他字符串数组
- **解决方案**: 使用品牌类型 `SystemPrompt` 提供编译时类型安全
- **优势**: 防止误用，提高代码可读性和可维护性

### 零依赖设计
- **目的**: 可以被任何模块导入而不引入循环依赖风险
- **实现**: 不导入任何其他项目模块
- **位置**: 作为类型系统的叶子节点

## 具体技术实现

### 品牌类型定义

```typescript
export type SystemPrompt = readonly string[] & {
  readonly __brand: 'SystemPrompt'
}
```

品牌类型特点：
- `readonly string[]`: 基础类型，字符串数组
- `& { readonly __brand: 'SystemPrompt' }`: 交叉类型添加品牌标记
- `readonly`: 不可变性保证

### 类型转换函数

```typescript
export function asSystemPrompt(value: readonly string[]): SystemPrompt {
  return value as SystemPrompt
}
```

该函数：
- 接受任何只读字符串数组
- 通过类型断言转换为 `SystemPrompt`
- 运行时无开销，纯编译时类型检查

### 使用示例

```typescript
import { SystemPrompt, asSystemPrompt } from './systemPromptType.js'

// 正确用法
const prompt: SystemPrompt = asSystemPrompt([
  'You are Claude, a helpful AI assistant.',
  'Follow the user\'s instructions carefully.'
])

// 错误用法（编译时错误）
const wrong: SystemPrompt = ['invalid']  // Error: Type 'string[]' is not assignable to type 'SystemPrompt'
```

## 关键代码路径与文件引用

### 本文件导出
- `SystemPrompt`: 品牌类型定义
- `asSystemPrompt(value)`: 类型转换函数

### 依赖模块
无依赖（零依赖设计）。

### 调用方

通过 Grep 发现大量调用方：

| 文件 | 用途 |
|------|------|
| `src/QueryEngine.ts` | 系统提示词类型 |
| `src/Tool.js` | 工具上下文类型 |
| `src/query.ts` | 查询处理 |
| `src/utils/systemPrompt.ts` | 重新导出 |
| `src/utils/queryContext.ts` | 查询上下文构建 |
| `src/utils/api.ts` | API 调用 |
| `src/services/api/claude.ts` | Claude API 服务 |
| `src/services/compact/compact.ts` | 消息压缩 |
| `src/tools/AgentTool/*.ts` | Agent 工具 |
| `src/utils/hooks/*.ts` | Hooks 系统 |

## 依赖与外部交互

### 与系统提示词构建的集成
- `systemPrompt.ts` 重新导出本模块的类型
- `buildEffectiveSystemPrompt` 返回 `SystemPrompt` 类型
- 调用方必须显式使用 `asSystemPrompt` 创建实例

### 与 API 调用的集成
- API 客户端使用 `SystemPrompt` 类型确保正确传递系统提示词
- 类型安全防止将普通字符串数组作为系统提示词发送

## 风险、边界与改进建议

### 潜在风险

1. **类型断言绕过**: `asSystemPrompt` 使用类型断言，无法验证内容有效性
2. **空数组**: 允许空数组作为 `SystemPrompt`，可能导致问题
3. **品牌冲突**: 如果其他模块定义相同品牌名，可能产生冲突

### 边界情况

1. **空数组**: `asSystemPrompt([])` 是合法的
2. **嵌套数组**: 如果字符串包含数组字面量，不影响类型
3. **超长数组**: 无长度限制，可能传递大量提示词

### 改进建议

1. **验证函数**: 添加运行时验证
```typescript
export function validateSystemPrompt(value: unknown): value is SystemPrompt {
  return Array.isArray(value) && 
         value.every(item => typeof item === 'string') &&
         value.length > 0
}

export function asSystemPromptSafe(value: readonly string[]): SystemPrompt | null {
  return validateSystemPrompt(value) ? value as SystemPrompt : null
}
```

2. **元数据支持**: 添加提示词元数据
```typescript
export type SystemPrompt = readonly string[] & {
  readonly __brand: 'SystemPrompt'
  readonly metadata?: {
    version?: string
    source?: string
    createdAt?: Date
  }
}
```

3. **长度限制**: 添加编译时或运行时长度检查
```typescript
const MAX_PROMPT_LENGTH = 100
export function asSystemPrompt(value: readonly string[]): SystemPrompt {
  if (value.length > MAX_PROMPT_LENGTH) {
    throw new Error(`System prompt too long: ${value.length} > ${MAX_PROMPT_LENGTH}`)
  }
  return value as SystemPrompt
}
```

4. **不变性强化**: 使用更严格的不可变类型
```typescript
export type SystemPrompt = ReadonlyArray<Readonly<string>> & {
  readonly __brand: 'SystemPrompt'
}
```

5. **命名空间品牌**: 使用唯一品牌名避免冲突
```typescript
declare const __claudeSystemPromptBrand: unique symbol
export type SystemPrompt = readonly string[] & {
  readonly [__claudeSystemPromptBrand]: true
}
```
