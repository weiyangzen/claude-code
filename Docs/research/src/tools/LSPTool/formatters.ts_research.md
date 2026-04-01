# formatters.ts 研究文档

## 场景与职责

formatters.ts 是 LSPTool 的结果格式化模块，负责将 LSP（Language Server Protocol）返回的原始数据结构转换为人类可读的文本格式。它是 LSPTool 与最终用户输出之间的关键桥梁。

**主要职责：**
- 格式化 9 种 LSP 操作的结果为可读文本
- 处理 LSP 的两种位置格式（Location 和 LocationLink）
- 处理 URI 解码和路径格式化
- 按文件分组显示结果
- 提供防御性错误处理，应对 malformed LSP 数据

**使用场景：**
- 将 LSP 返回的 JSON 结果转换为终端显示的文本
- 统一处理不同 LSP 服务器的输出格式差异
- 处理文件路径显示（相对路径 vs 绝对路径）

---

## 功能点目的

### 1. URI 格式化（formatUri）

将 LSP 返回的 file:// URI 转换为可读的相对路径：

```typescript
function formatUri(uri: string | undefined, cwd?: string): string
```

**功能：**
- 移除 `file://` 协议前缀
- 处理 Windows 路径（`file:///C:/path` → `C:/path`）
- URI 解码（处理空格、中文等）
- 转换为相对路径（如果更短且不以 `../..` 开头）
- 统一使用正斜杠

### 2. 位置格式化（formatLocation）

格式化单个位置信息：

```typescript
function formatLocation(location: Location, cwd?: string): string
// 输出: "src/file.ts:10:5"
```

### 3. 按文件分组（groupByFile）

将位置或符号按文件分组，便于组织显示：

```typescript
function groupByFile<T extends { uri: string } | { location: { uri: string } }>(
  items: T[],
  cwd?: string
): Map<string, T[]>
```

### 4. 各操作格式化函数

| 函数 | 输入类型 | 输出示例 |
|------|----------|----------|
| `formatGoToDefinitionResult` | Location/LocationLink | "Defined in src/foo.ts:10:5" |
| `formatFindReferencesResult` | Location[] | "Found 5 references across 2 files:\nfile1.ts:\n  Line 10:5\n..." |
| `formatHoverResult` | Hover | "Hover info at 10:5:\n\nType: string" |
| `formatDocumentSymbolResult` | DocumentSymbol[]/SymbolInformation[] | "Document symbols:\n  foo (Function) - Line 10\n..." |
| `formatWorkspaceSymbolResult` | SymbolInformation[] | "Found 10 symbols in workspace:\nfile.ts:\n  foo (Function) - Line 10\n..." |
| `formatPrepareCallHierarchyResult` | CallHierarchyItem[] | "Call hierarchy item: foo (Function) - src/foo.ts:10" |
| `formatIncomingCallsResult` | CallHierarchyIncomingCall[] | "Found 3 incoming calls:\nfile.ts:\n  caller (Function) - Line 10 [calls at: 12:5]" |
| `formatOutgoingCallsResult` | CallHierarchyOutgoingCall[] | "Found 3 outgoing calls:\nfile.ts:\n  callee (Function) - Line 10 [called from: 12:5]" |

---

## 具体技术实现

### URI 处理详解

```typescript
function formatUri(uri: string | undefined, cwd?: string): string {
  // 防御性处理 undefined URI
  if (!uri) {
    logForDebugging('formatUri called with undefined URI...', { level: 'warn' })
    return '<unknown location>'
  }

  // 1. 移除 file:// 前缀
  let filePath = uri.replace(/^file:\/\//, '')
  
  // 2. Windows 路径处理: /C:/path → C:/path
  if (/^\/[A-Za-z]:/.test(filePath)) {
    filePath = filePath.slice(1)
  }

  // 3. URI 解码（失败时使用未解码路径）
  try {
    filePath = decodeURIComponent(filePath)
  } catch (error) {
    logForDebugging(`Failed to decode LSP URI '${uri}': ${errorMsg}`, { level: 'warn' })
  }

  // 4. 尝试转换为相对路径
  if (cwd) {
    const relativePath = relative(cwd, filePath).replaceAll('\\', '/')
    // 只使用相对路径如果：
    // - 比绝对路径短
    // - 不以 ../.. 开头（避免过多层级）
    if (relativePath.length < filePath.length && !relativePath.startsWith('../../')) {
      return relativePath
    }
  }

  // 5. 统一使用正斜杠
  return filePath.replaceAll('\\', '/')
}
```

### LocationLink 转换

LSP 支持两种位置格式，需要统一处理：

```typescript
function isLocationLink(item: Location | LocationLink): item is LocationLink {
  return 'targetUri' in item
}

function locationLinkToLocation(link: LocationLink): Location {
  return {
    uri: link.targetUri,
    range: link.targetSelectionRange || link.targetRange
  }
}
```

### 引用结果格式化

```typescript
export function formatFindReferencesResult(result: Location[] | null, cwd?: string): string {
  if (!result || result.length === 0) {
    return 'No references found. This may occur if the symbol has no usages...'
  }

  // 过滤无效位置
  const invalidLocations = result.filter(loc => !loc || !loc.uri)
  if (invalidLocations.length > 0) {
    logForDebugging(`formatFindReferencesResult: Filtering out ${invalidLocations.length} invalid location(s)`)
  }
  const validLocations = result.filter(loc => loc && loc.uri)

  if (validLocations.length === 0) {
    return 'No references found...'
  }

  if (validLocations.length === 1) {
    return `Found 1 reference:\n  ${formatLocation(validLocations[0]!, cwd)}`
  }

  // 按文件分组显示
  const byFile = groupByFile(validLocations, cwd)
  const lines: string[] = [
    `Found ${validLocations.length} references across ${byFile.size} files:`,
  ]

  for (const [filePath, locations] of byFile) {
    lines.push(`\n${filePath}:`)
    for (const loc of locations) {
      const line = loc.range.start.line + 1  // 转 1-based
      const character = loc.range.start.character + 1
      lines.push(`  Line ${line}:${character}`)
    }
  }

  return lines.join('\n')
}
```

### Hover 内容提取

```typescript
function extractMarkupText(
  contents: MarkupContent | MarkedString | MarkedString[]
): string {
  if (Array.isArray(contents)) {
    return contents
      .map(item => typeof item === 'string' ? item : item.value)
      .join('\n\n')
  }

  if (typeof contents === 'string') {
    return contents
  }

  if ('kind' in contents) {
    return contents.value  // MarkupContent
  }

  return contents.value  // MarkedString object
}
```

### 符号类型映射

```typescript
function symbolKindToString(kind: SymbolKind): string {
  const kinds: Record<SymbolKind, string> = {
    [1]: 'File',
    [2]: 'Module',
    [3]: 'Namespace',
    [4]: 'Package',
    [5]: 'Class',
    [6]: 'Method',
    [7]: 'Property',
    [8]: 'Field',
    [9]: 'Constructor',
    [10]: 'Enum',
    [11]: 'Interface',
    [12]: 'Function',
    [13]: 'Variable',
    [14]: 'Constant',
    [15]: 'String',
    [16]: 'Number',
    [17]: 'Boolean',
    [18]: 'Array',
    [19]: 'Object',
    [20]: 'Key',
    [21]: 'Null',
    [22]: 'EnumMember',
    [23]: 'Struct',
    [24]: 'Event',
    [25]: 'Operator',
    [26]: 'TypeParameter',
  }
  return kinds[kind] || 'Unknown'
}
```

### DocumentSymbol 递归格式化

```typescript
function formatDocumentSymbolNode(symbol: DocumentSymbol, indent: number = 0): string[] {
  const lines: string[] = []
  const prefix = '  '.repeat(indent)
  const kind = symbolKindToString(symbol.kind)

  let line = `${prefix}${symbol.name} (${kind})`
  if (symbol.detail) {
    line += ` ${symbol.detail}`
  }
  const symbolLine = symbol.range.start.line + 1
  line += ` - Line ${symbolLine}`

  lines.push(line)

  // 递归处理子符号
  if (symbol.children && symbol.children.length > 0) {
    for (const child of symbol.children) {
      lines.push(...formatDocumentSymbolNode(child, indent + 1))
    }
  }

  return lines
}
```

### 调用层次格式化

入站调用显示调用者和调用位置：

```typescript
export function formatIncomingCallsResult(
  result: CallHierarchyIncomingCall[] | null,
  cwd?: string
): string {
  // ...
  for (const call of calls) {
    if (!call.from) continue  // 防御性处理
    
    const kind = symbolKindToString(call.from.kind)
    const line = call.from.range.start.line + 1
    let callLine = `  ${call.from.name} (${kind}) - Line ${line}`

    // 显示调用位置
    if (call.fromRanges && call.fromRanges.length > 0) {
      const callSites = call.fromRanges
        .map(r => `${r.start.line + 1}:${r.start.character + 1}`)
        .join(', ')
      callLine += ` [calls at: ${callSites}]`
    }

    lines.push(callLine)
  }
  // ...
}
```

---

## 关键代码路径与文件引用

### 内部依赖

```typescript
import { relative } from 'path'
import type {
  CallHierarchyIncomingCall,
  CallHierarchyItem,
  CallHierarchyOutgoingCall,
  DocumentSymbol,
  Hover,
  Location,
  LocationLink,
  MarkedString,
  MarkupContent,
  SymbolInformation,
  SymbolKind,
} from 'vscode-languageserver-types'
import { logForDebugging } from '../../utils/debug.js'
import { errorMessage } from '../../utils/errors.js'
import { plural } from '../../utils/stringUtils.js'
```

### 导出函数清单

| 函数名 | 行号 | 用途 |
|--------|------|------|
| `formatGoToDefinitionResult` | 127 | 格式化定义跳转结果 |
| `formatFindReferencesResult` | 174 | 格式化引用查找结果 |
| `formatHoverResult` | 253 | 格式化悬停提示结果 |
| `formatDocumentSymbolResult` | 340 | 格式化文档符号结果 |
| `formatWorkspaceSymbolResult` | 371 | 格式化工作区符号结果 |
| `formatPrepareCallHierarchyResult` | 455 | 格式化调用层次准备结果 |
| `formatIncomingCallsResult` | 478 | 格式化入站调用结果 |
| `formatOutgoingCallsResult` | 538 | 格式化出站调用结果 |

### 辅助函数

| 函数名 | 行号 | 用途 |
|--------|------|------|
| `formatUri` | 24 | URI 转可读路径 |
| `groupByFile` | 78 | 按文件分组 |
| `formatLocation` | 99 | 格式化单个位置 |
| `locationLinkToLocation` | 109 | LocationLink 转 Location |
| `isLocationLink` | 119 | 类型守卫 |
| `extractMarkupText` | 223 | 提取 MarkupContent 文本 |
| `symbolKindToString` | 272 | SymbolKind 转字符串 |
| `formatDocumentSymbolNode` | 307 | 递归格式化文档符号 |
| `formatCallHierarchyItem` | 428 | 格式化调用层次项 |

---

## 依赖与外部交互

### vscode-languageserver-types

使用 LSP 官方类型定义：
- `Location`, `LocationLink`: 位置信息
- `DocumentSymbol`, `SymbolInformation`: 符号信息
- `Hover`, `MarkupContent`, `MarkedString`: 悬停内容
- `CallHierarchyItem`, `CallHierarchyIncomingCall`, `CallHierarchyOutgoingCall`: 调用层次
- `SymbolKind`: 符号类型枚举

### 工具函数

| 函数 | 来源 | 用途 |
|------|------|------|
| `logForDebugging` | `../../utils/debug.js` | 调试日志 |
| `errorMessage` | `../../utils/errors.js` | 错误信息提取 |
| `plural` | `../../utils/stringUtils.js` | 单复数转换 |

### Node.js 内置

- `path.relative`: 计算相对路径

---

## 风险、边界与改进建议

### 已知风险

1. **Malformed LSP 数据**
   - 风险：某些 LSP 服务器返回无效数据（undefined URI 等）
   - 处理：多处防御性检查，过滤无效项，记录调试日志
   - 示例：
     ```typescript
     const invalidLocations = locations.filter(loc => !loc || !loc.uri)
     if (invalidLocations.length > 0) {
       logForDebugging(`Filtering out ${invalidLocations.length} invalid location(s)`)
     }
     ```

2. **URI 解码失败**
   - 风险：畸形 URI 导致 decodeURIComponent 抛出
   - 处理：try-catch 包裹，回退到未解码路径

3. **路径格式不一致**
   - 风险：Windows 反斜杠与 Unix 正斜杠混用
   - 处理：统一使用 `replaceAll('\\', '/')`

4. **递归深度**
   - 风险：DocumentSymbol 嵌套过深导致栈溢出
   - 现状：依赖 LSP 服务器返回合理结构

### 边界情况

| 场景 | 行为 |
|------|------|
| 空结果 | 返回友好说明文本 |
| 单结果 | 简化显示格式 |
| 多结果 | 按文件分组，显示统计 |
| undefined URI | 返回 '<unknown location>' |
| URI 解码失败 | 使用原始路径 |
| 相对路径过长 | 使用绝对路径 |
| SymbolKind 未知 | 返回 'Unknown' |

### 改进建议

1. **结果截断**
   - 当前：可能输出大量结果
   - 建议：添加结果数量上限，超出时提示 "...and X more"

2. **语法高亮**
   - 当前：纯文本输出
   - 建议：在 Hover 结果中保留 markdown 格式并渲染

3. **代码片段显示**
   - 当前：只显示行号
   - 建议：可选显示引用位置的代码片段

4. **性能优化**
   - 当前：多次遍历数组（过滤、分组、格式化）
   - 建议：考虑单次遍历处理

5. **可配置性**
   - 当前：格式固定
   - 建议：允许用户配置显示样式（如是否显示 containerName）

6. **国际化**
   - 当前：英文输出
   - 建议：支持多语言

### 测试建议

- 测试各种 LSP 服务器的输出格式差异
- 测试畸形数据处理能力
- 测试大结果集性能
- 测试 Windows/Unix 路径处理
- 测试 URI 编码/解码边界情况
