# src/utils/cliHighlight.ts 深度研究文档

## 场景与职责

`cliHighlight.ts` 提供 CLI 代码高亮功能的延迟加载和缓存机制。它封装了 `cli-highlight` 库的加载逻辑，确保：
- 高亮功能按需加载（不会阻塞启动）
- 多次调用共享同一个加载 Promise（避免重复加载）
- 加载失败时优雅降级（返回 null）

此外，还提供基于文件扩展名的语言检测功能（用于遥测）。

## 功能点目的

### 1. 延迟加载 cli-highlight
- 使用动态 `import()` 延迟加载高亮库
- 单例 Promise 模式确保并发调用共享加载结果
- 加载失败时返回 null，不影响主流程

### 2. 语言名称检测
- 根据文件路径扩展名获取语言名称
- 使用 highlight.js 的语言注册表
- 用于遥测（OTel 计数器属性、权限对话框事件）

### 3. DOM 类型兼容
- 文件顶部的 `/// <reference lib="dom" />` 注释确保 DOM 类型可用
- 这是为了解决其他模块（SSETransport、mcp/client、ssh 等）依赖 DOM 类型的问题

## 具体技术实现

### 核心数据结构

```typescript
export type CliHighlight = {
  highlight: typeof import('cli-highlight').highlight
  supportsLanguage: typeof import('cli-highlight').supportsLanguage
}
```

### 延迟加载实现

```typescript
// 共享的 Promise，所有调用者共用
let cliHighlightPromise: Promise<CliHighlight | null> | undefined

// 同时缓存 highlight.js 的 getLanguage 函数
let loadedGetLanguage: typeof import('highlight.js').getLanguage | undefined

async function loadCliHighlight(): Promise<CliHighlight | null> {
  try {
    const cliHighlight = await import('cli-highlight')
    // cli-highlight 已经加载了 highlight.js，第二次 import 是缓存命中
    const highlightJs = await import('highlight.js')
    loadedGetLanguage = highlightJs.getLanguage
    return {
      highlight: cliHighlight.highlight,
      supportsLanguage: cliHighlight.supportsLanguage,
    }
  } catch {
    return null
  }
}

export function getCliHighlightPromise(): Promise<CliHighlight | null> {
  cliHighlightPromise ??= loadCliHighlight()
  return cliHighlightPromise
}
```

### 语言名称检测

```typescript
export async function getLanguageName(file_path: string): Promise<string> {
  await getCliHighlightPromise()
  const ext = extname(file_path).slice(1)
  if (!ext) return 'unknown'
  return loadedGetLanguage?.(ext)?.name ?? 'unknown'
}
```

### 调用模式

```typescript
// 在组件中使用
import { getCliHighlightPromise } from '../utils/cliHighlight.js'

async function renderCode(code: string, language: string) {
  const highlighter = await getCliHighlightPromise()
  if (highlighter && highlighter.supportsLanguage(language)) {
    return highlighter.highlight(code, { language })
  }
  // 降级：返回原始代码
  return code
}
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `cli-highlight` | 代码高亮功能 |
| `highlight.js` | 语言检测（cli-highlight 已依赖） |

### 内部模块依赖

| 模块 | 用途 |
|------|------|
| `path` (Node.js) | 提取文件扩展名 |

### 被依赖方

| 模块 | 用途 |
|------|------|
| `src/components/HighlightedCode/Fallback.tsx` | 代码高亮 fallback |
| `src/utils/markdown.ts` | Markdown 代码块高亮 |
| `src/components/Markdown.tsx` | Markdown 渲染 |
| `src/components/MarkdownTable.tsx` | 表格中的代码高亮 |
| `src/hooks/toolPermission/permissionLogging.ts` | 权限日志语言检测 |
| `src/components/permissions/FilePermissionDialog/FilePermissionDialog.tsx` | 文件权限对话框 |
| `src/components/permissions/AskUserQuestionPermissionRequest/PreviewBox.tsx` | 预览框代码高亮 |
| `src/components/permissions/AskUserQuestionPermissionRequest/AskUserQuestionPermissionRequest.tsx` | 权限请求对话框 |

## 风险、边界与改进建议

### 已知风险

1. **DOM 类型依赖**
   - 文件顶部的 `/// <reference lib="dom" />` 是一个技术债
   - 理想情况下应该修复实际依赖 DOM 类型的模块
   - 当前是权宜之计，保持现状

2. **加载失败处理**
   - 加载失败时返回 null，调用方需要处理降级
   - 某些调用方可能没有正确处理 null 情况

3. **语言检测准确性**
   - 仅基于文件扩展名，可能不准确
   - 例如 `.ts` 可能是 TypeScript 或 TypeScript JSX

### 边界情况

1. **并发调用**
   - 多个并发调用会共享同一个 Promise
   - 加载成功/失败的结果会被所有调用者共享

2. **无扩展名文件**
   - `getLanguageName` 对无扩展名文件返回 'unknown'

3. **不支持的扩展名**
   - highlight.js 不认识的扩展名返回 'unknown'

4. **多次调用 getCliHighlightPromise**
   - 由于 Promise 被缓存，后续调用立即返回已解析的值

### 改进建议

1. **DOM 类型问题**
   - 长期：修复 SSETransport、mcp/client 等模块的实际 DOM 类型依赖
   - 短期：添加注释说明这是技术债

2. **错误处理**
   - 添加加载失败的日志记录
   - 提供加载状态的查询接口

3. **语言检测增强**
   - 支持基于内容的语言检测（文件头、shebang）
   - 支持更多文件扩展名映射
   - 处理特殊情况（如 `.ts` vs `.tsx`）

4. **性能优化**
   - 考虑预加载常用语言的高亮定义
   - 添加高亮结果缓存（对于重复内容）

5. **API 改进**
   - 添加同步检查（是否已加载）
   - 支持强制重新加载
   - 提供加载进度回调

6. **代码示例**

```typescript
// 改进版本（带加载状态）
type CliHighlightState = 
  | { status: 'idle' }
  | { status: 'loading'; promise: Promise<CliHighlight | null> }
  | { status: 'loaded'; value: CliHighlight }
  | { status: 'error'; error: Error }

let state: CliHighlightState = { status: 'idle' }

export function getCliHighlight(): CliHighlightState {
  return state
}

export async function loadCliHighlight(): Promise<CliHighlight | null> {
  if (state.status === 'loaded') return state.value
  if (state.status === 'loading') return state.promise
  
  const promise = doLoad()
  state = { status: 'loading', promise }
  
  try {
    const value = await promise
    state = { status: 'loaded', value }
    return value
  } catch (error) {
    state = { status: 'error', error: error as Error }
    return null
  }
}
```
