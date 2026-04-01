# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 是 `TeamDeleteTool` 的 UI 渲染组件，负责在工具使用和结果展示时提供用户界面反馈。该组件遵循 Claude Code CLI 的工具 UI 架构模式，将渲染逻辑与工具业务逻辑分离。

### 核心职责
1. **工具使用提示渲染**：显示工具正在执行的操作
2. **结果消息渲染**：处理并展示工具执行结果
3. **消息抑制**：根据设计选择性地隐藏某些消息

## 功能点目的

### 1. 工具使用提示 (`renderToolUseMessage`)
- 当 LLM 调用 TeamDelete 工具时显示简短提示
- 告知用户正在进行团队清理操作
- 返回简单的字符串消息：`'cleanup team: current'`

### 2. 结果消息渲染 (`renderToolResultMessage`)
- 解析工具执行结果（支持字符串 JSON 或对象格式）
- 根据结果内容决定是否显示消息
- **关键设计决策**：完全抑制清理结果消息

### 3. 消息抑制策略
- 结果消息被完全抑制（返回 `null`）
- 原因：批量的 shutdown 消息已经覆盖了这个信息
- 避免在 UI 中显示冗余的清理确认消息

## 具体技术实现

### 函数签名

```typescript
// 工具使用提示渲染
export function renderToolUseMessage(
  _input: Record<string, unknown>
): React.ReactNode

// 结果消息渲染
export function renderToolResultMessage(
  content: Output | string,
  _progressMessages: unknown,
  { verbose: _verbose }: { verbose: boolean }
): React.ReactNode
```

### 结果解析逻辑

```typescript
export function renderToolResultMessage(
  content: Output | string,
  _progressMessages: unknown,
  { verbose: _verbose }: { verbose: boolean }
): React.ReactNode {
  // 支持两种输入格式：字符串 JSON 或已解析的对象
  const result: Output = typeof content === 'string' ? jsonParse(content) : content

  // 抑制清理结果 - 批量的 shutdown 消息覆盖此消息
  if ('success' in result && 'team_name' in result && 'message' in result) {
    return null
  }
  
  return null
}
```

### 类型定义依赖

```typescript
// 从主工具文件导入 Output 类型
import type { Output } from './TeamDeleteTool.js'

// Output 类型结构
interface Output {
  success: boolean
  message: string
  team_name?: string
}
```

## 关键代码路径与文件引用

### 直接依赖

| 路径 | 导入内容 | 用途 |
|------|----------|------|
| `react` | `React` | JSX 渲染 |
| `../../utils/slowOperations.js` | `jsonParse` | JSON 解析（带性能监控） |
| `./TeamDeleteTool.js` | `Output` 类型 | 类型定义 |

### 被调用方

| 路径 | 用途 |
|------|------|
| `./TeamDeleteTool.ts` | 注册到 `buildTool` 的 `renderToolUseMessage` 和 `renderToolResultMessage` 属性 |

### 调用链

```
TeamDeleteTool.ts
  ├─ import { renderToolResultMessage, renderToolUseMessage } from './UI.js'
  └─ buildTool({
       renderToolUseMessage,      // 注册使用提示渲染器
       renderToolResultMessage,   // 注册结果渲染器
     })
```

## 依赖与外部交互

### 慢操作包装 (`slowOperations.js`)

```typescript
import { jsonParse } from '../../utils/slowOperations.js'

// jsonParse 是 JSON.parse 的包装，添加了性能监控
export const jsonParse: typeof JSON.parse = (text, reviver) => {
  using _ = slowLogging`JSON.parse(${text})`
  return typeof reviver === 'undefined'
    ? JSON.parse(text)
    : JSON.parse(text, reviver)
}
```

### React 类型

```typescript
import React from 'react'

// 返回类型为 React.ReactNode
// 返回 null 表示不渲染任何内容
```

## 风险、边界与改进建议

### 风险点

1. **完全消息抑制**
   - 当前实现始终返回 `null`，用户无法看到清理结果
   - 如果批量 shutdown 消息未显示，用户可能不知道清理是否成功
   - 建议：在 verbose 模式下显示详细结果

2. **类型安全**
   - `_progressMessages` 和 `_verbose` 使用 `unknown` 和 `_` 前缀表示未使用
   - 如果未来需要支持进度消息，需要修改函数签名

3. **JSON 解析错误**
   - `jsonParse` 可能抛出异常（无效 JSON）
   - 当前实现没有 try-catch，错误会向上传播

### 边界情况

1. **空结果**
   - 如果 `content` 是空字符串，`jsonParse` 会抛出异常

2. **部分属性缺失**
   - 检查 `success`, `team_name`, `message` 都存在才抑制
   - 如果结果对象缺少这些属性，理论上会返回 `null`（else 分支也是 null）

3. **字符串 vs 对象输入**
   - 支持两种输入格式以适应不同的调用上下文
   - 字符串格式用于从 API 响应解析
   - 对象格式用于直接传递

### 改进建议

1. **Verbose 模式支持**
   ```typescript
   export function renderToolResultMessage(
     content: Output | string,
     _progressMessages: unknown,
     { verbose }: { verbose: boolean }
   ): React.ReactNode {
     const result: Output = typeof content === 'string' ? jsonParse(content) : content
     
     // 在 verbose 模式下显示详细结果
     if (verbose && result.success) {
       return `✓ Team "${result.team_name}" cleaned up: ${result.message}`
     }
     
     return null  // 默认抑制
   }
   ```

2. **错误处理**
   ```typescript
   try {
     const result: Output = typeof content === 'string' ? jsonParse(content) : content
     // ... 处理逻辑
   } catch (error) {
     return `Failed to parse result: ${error}`
   }
   ```

3. **更精确的消息过滤**
   - 考虑只抑制成功的消息，显示错误消息
   - 或者根据结果内容动态决定是否显示

4. **类型改进**
   - 如果确定不需要进度消息，可以从签名中移除
   - 或者实现进度消息支持

5. **测试覆盖**
   - 添加单元测试验证不同输入格式的处理
   - 测试边界情况（空字符串、无效 JSON、部分属性缺失）
