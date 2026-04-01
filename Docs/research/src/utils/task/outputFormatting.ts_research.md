# outputFormatting.ts 深度研究文档

## 场景与职责

`outputFormatting.ts` 是 Claude Code 中任务输出格式化的**配置与工具模块**。它负责管理任务输出的长度限制、格式化截断逻辑，确保输出符合 API 消费的大小约束。

### 核心职责

1. **输出长度限制管理**: 通过环境变量配置任务输出上限
2. **格式化截断**: 当输出超过限制时，智能截断并保留尾部内容
3. **配置验证**: 验证环境变量值，提供默认值和上限保护

### 使用场景

| 场景 | 调用方 | 功能 |
|------|--------|------|
| 任务结果格式化 | `LocalShellTask.tsx` 等任务实现 | `formatTaskOutput()` |
| 输出长度限制检查 | 任务结果处理 | `getMaxTaskOutputLength()` |
| 环境变量配置 | 系统启动 | `TASK_MAX_OUTPUT_LENGTH` |

---

## 功能点目的

### 1. 输出长度限制

```typescript
export const TASK_MAX_OUTPUT_UPPER_LIMIT = 160_000  // 硬上限
export const TASK_MAX_OUTPUT_DEFAULT = 32_000       // 默认值
```

**配置方式**:
- 环境变量: `TASK_MAX_OUTPUT_LENGTH`
- 验证逻辑: `validateBoundedIntEnvVar()` (来自 `envValidation.ts`)

### 2. 智能截断格式化

当输出超过限制时:
1. 添加截断提示头，包含完整输出文件路径
2. 保留尾部内容（而非头部），因为尾部通常包含最新/最重要的信息
3. 返回截断标记，供调用方知晓发生了截断

---

## 具体技术实现

### 关键代码

```typescript
import { validateBoundedIntEnvVar } from '../envValidation.js'
import { getTaskOutputPath } from './diskOutput.js'

export const TASK_MAX_OUTPUT_UPPER_LIMIT = 160_000
export const TASK_MAX_OUTPUT_DEFAULT = 32_000

export function getMaxTaskOutputLength(): number {
  const result = validateBoundedIntEnvVar(
    'TASK_MAX_OUTPUT_LENGTH',
    process.env.TASK_MAX_OUTPUT_LENGTH,
    TASK_MAX_OUTPUT_DEFAULT,
    TASK_MAX_OUTPUT_UPPER_LIMIT,
  )
  return result.effective
}

export function formatTaskOutput(
  output: string,
  taskId: string,
): { content: string; wasTruncated: boolean } {
  const maxLen = getMaxTaskOutputLength()
  
  if (output.length <= maxLen) {
    return { content: output, wasTruncated: false }
  }
  
  const filePath = getTaskOutputPath(taskId)
  const header = `[Truncated. Full output: ${filePath}]\n\n`
  const availableSpace = maxLen - header.length
  const truncated = output.slice(-availableSpace)  // 保留尾部
  
  return { content: header + truncated, wasTruncated: true }
}
```

### 环境变量验证

依赖 `envValidation.ts` 的 `validateBoundedIntEnvVar`:

```typescript
export type EnvVarValidationResult = {
  effective: number
  status: 'valid' | 'capped' | 'invalid'
  message?: string
}

export function validateBoundedIntEnvVar(
  name: string,
  value: string | undefined,
  defaultValue: number,
  upperLimit: number,
): EnvVarValidationResult
```

**验证逻辑**:
| 输入 | 结果 | 状态 |
|------|------|------|
| 未设置 | 使用默认值 | `'valid'` |
| 有效正整数 | 使用输入值 | `'valid'` |
| 超过上限 | 使用上限值 | `'capped'` |
| 无效值 (NaN, ≤0) | 使用默认值 | `'invalid'` |

---

## 关键代码路径与文件引用

### 调用链

```
任务完成处理
  └── formatTaskOutput(output, taskId)
      ├── getMaxTaskOutputLength()
      │   └── validateBoundedIntEnvVar()
      └── getTaskOutputPath(taskId)
          └── diskOutput.ts
```

### 外部调用方

| 文件 | 调用函数 | 用途 |
|------|----------|------|
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | `formatTaskOutput` | Shell 任务结果格式化 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `formatTaskOutput` | 代理任务结果格式化 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `formatTaskOutput` | 远程代理任务 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | `formatTaskOutput` | 进程内队友任务 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/utils/envValidation.ts` | `validateBoundedIntEnvVar` |
| `src/utils/task/diskOutput.ts` | `getTaskOutputPath` |

---

## 依赖与外部交互

### 模块依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                     outputFormatting.ts                         │
├─────────────────────────────────────────────────────────────────┤
│  Configuration                                                  │
│  ├── TASK_MAX_OUTPUT_DEFAULT = 32_000                          │
│  ├── TASK_MAX_OUTPUT_UPPER_LIMIT = 160_000                     │
│  └── getMaxTaskOutputLength() → validateBoundedIntEnvVar()     │
├─────────────────────────────────────────────────────────────────┤
│  Formatting                                                     │
│  └── formatTaskOutput()                                         │
│      ├── getMaxTaskOutputLength()                              │
│      ├── getTaskOutputPath() [diskOutput.ts]                   │
│      └── output.slice(-availableSpace)  // 尾部截断            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 与 Bash 输出限制不一致

**问题**: `outputFormatting.ts` (32K/160K) 与 `shell/outputLimits.ts` (30K/150K) 的限制值不同。

**影响**: 可能导致用户困惑，为什么任务输出和 Bash 输出有不同的限制。

**建议**: 统一配置或使用相同的默认值。

### 边界情况

| 场景 | 行为 |
|------|------|
| 输出恰好等于限制 | 不截断，`wasTruncated: false` |
| 输出为空字符串 | 返回空，`wasTruncated: false` |
| header 长度 > maxLen | `availableSpace` 为负，`slice` 返回空字符串 |
| 环境变量无效 | 记录调试日志，使用默认值 |
| 环境变量超过上限 | 记录调试日志，使用上限值 |

### 改进建议

#### 1. 统一输出限制配置

当前分散在两个文件:
```typescript
// shell/outputLimits.ts
export const BASH_MAX_OUTPUT_DEFAULT = 30_000
export const BASH_MAX_OUTPUT_UPPER_LIMIT = 150_000

// task/outputFormatting.ts  
export const TASK_MAX_OUTPUT_DEFAULT = 32_000
export const TASK_MAX_OUTPUT_UPPER_LIMIT = 160_000
```

建议统一:
```typescript
// shared/outputLimits.ts
export const OUTPUT_LIMITS = {
  bash: { default: 30_000, max: 150_000 },
  task: { default: 30_000, max: 150_000 },
} as const
```

#### 2. 更智能的截断策略

当前简单截断尾部，可考虑:
```typescript
export function formatTaskOutputSmart(
  output: string,
  taskId: string,
): { content: string; wasTruncated: boolean } {
  const maxLen = getMaxTaskOutputLength()
  
  if (output.length <= maxLen) {
    return { content: output, wasTruncated: false }
  }
  
  // 策略：保留开头和结尾，中间用省略号
  const headerLen = Math.floor(maxLen * 0.2)  // 20% 开头
  const tailLen = Math.floor(maxLen * 0.7)    // 70% 结尾
  const ellipsis = '\n... [output truncated] ...\n'
  
  const filePath = getTaskOutputPath(taskId)
  const header = `[Truncated. Full output: ${filePath}]\n\n`
  
  const result = header +
    output.slice(0, headerLen) +
    ellipsis +
    output.slice(-tailLen)
  
  return { content: result, wasTruncated: true }
}
```

#### 3. 字符数 vs 字节数

当前使用 `output.length` (UTF-16 code units)，对于多字节字符可能不准确:
```typescript
// 当前 (可能低估 UTF-8 字节数)
if (output.length <= maxLen)

// 改进 (精确字节数)
const byteLength = Buffer.byteLength(output, 'utf8')
if (byteLength <= maxLen)
```

#### 4. 流式格式化接口

当前接口需要完整输出字符串，对于大输出不友好:
```typescript
// 建议：流式接口
export async function* formatTaskOutputStream(
  outputStream: ReadableStream,
  taskId: string,
): AsyncGenerator<string> {
  const maxLen = getMaxTaskOutputLength()
  let accumulated = ''
  
  for await (const chunk of outputStream) {
    accumulated += chunk
    if (accumulated.length > maxLen) {
      // 触发截断，只保留尾部
    }
  }
}
```

#### 5. 配置热重载

当前环境变量只在启动时读取，支持运行时更新:
```typescript
let cachedMaxLength: number | null = null

export function getMaxTaskOutputLength(): number {
  if (cachedMaxLength === null) {
    cachedMaxLength = validateBoundedIntEnvVar(...).effective
  }
  return cachedMaxLength
}

export function invalidateMaxLengthCache(): void {
  cachedMaxLength = null
}
```

### 测试覆盖建议

当前未发现专门的测试文件，建议添加:

1. **单元测试**:
   - 各种长度输出的截断行为
   - 环境变量验证边界
   - 空字符串和极端长度处理

2. **集成测试**:
   - 与 `diskOutput.ts` 的集成
   - 实际任务输出格式化
