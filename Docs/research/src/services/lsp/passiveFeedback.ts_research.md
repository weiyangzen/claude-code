# passiveFeedback.ts 深度研究文档

## 场景与职责

`passiveFeedback.ts` 是 LSP 子系统的被动反馈模块，负责接收 LSP 服务器异步发送的诊断通知（`textDocument/publishDiagnostics`），将其转换为 Claude 诊断格式，并注册到诊断注册表供后续附件交付。

**核心职责：**
1. **诊断接收**：监听所有 LSP 服务器的 `textDocument/publishDiagnostics` 通知
2. **格式转换**：将 LSP 诊断格式转换为 Claude 内部 `DiagnosticFile` 格式
3. **错误隔离**：单个服务器的诊断处理错误不影响其他服务器
4. **失败追踪**：追踪连续失败，超过阈值后发出警告
5. **注册分发**：将处理后的诊断注册到 `LSPDiagnosticRegistry`

**在系统中的位置：**
- 被 `manager.ts` 调用，在 LSP 管理器初始化成功后注册处理器
- 调用 `LSPDiagnosticRegistry.ts` 注册诊断
- 属于 LSP 架构的反馈层（LSP Server → passiveFeedback → LSPDiagnosticRegistry → Attachment System）

---

## 功能点目的

### 1. 严重程度映射

**LSP → Claude 映射（Lines 18-35）：**
```typescript
function mapLSPSeverity(lspSeverity: number | undefined): 'Error' | 'Warning' | 'Info' | 'Hint' {
  // LSP DiagnosticSeverity: 1=Error, 2=Warning, 3=Information, 4=Hint
  switch (lspSeverity) {
    case 1: return 'Error'
    case 2: return 'Warning'
    case 3: return 'Info'
    case 4: return 'Hint'
    default: return 'Error'  // 默认保守处理为错误
  }
}
```

### 2. 诊断格式转换

**`formatDiagnosticsForAttachment()` 核心逻辑（Lines 43-100）：**

**URI 处理：**
```typescript
// 支持 file:// URI 和普通路径
const uri = params.uri.startsWith('file://')
  ? fileURLToPath(params.uri)
  : params.uri
```

**诊断转换：**
```typescript
const diagnostics = params.diagnostics.map(diag => ({
  message: diag.message,
  severity: mapLSPSeverity(diag.severity),
  range: {
    start: { line: diag.range.start.line, character: diag.range.start.character },
    end: { line: diag.range.end.line, character: diag.range.end.character }
  },
  source: diag.source,
  code: diag.code !== undefined ? String(diag.code) : undefined,
}))
```

### 3. 处理器注册

**`registerLSPNotificationHandlers()` 核心逻辑（Lines 125-328）：**

**遍历所有服务器：**
```typescript
const servers = manager.getAllServers()
for (const [serverName, serverInstance] of servers.entries()) {
  // 验证服务器实例有效性
  if (!serverInstance || typeof serverInstance.onNotification !== 'function') {
    registrationErrors.push({ serverName, error: '...' })
    continue  // 跳过无效服务器，继续处理其他
  }
  
  // 注册诊断处理器
  serverInstance.onNotification('textDocument/publishDiagnostics', handler)
}
```

### 4. 诊断处理器实现

**处理器逻辑（Lines 161-277）：**

**参数验证：**
```typescript
if (!params || typeof params !== 'object' || !('uri' in params) || !('diagnostics' in params)) {
  logError(new Error(`LSP server ${serverName} sent invalid diagnostic params`))
  return
}
```

**空诊断过滤：**
```typescript
if (!firstFile || diagnosticFiles.length === 0 || firstFile.diagnostics.length === 0) {
  logForDebugging(`Skipping empty diagnostics from ${serverName}`)
  return
}
```

**注册到诊断注册表：**
```typescript
registerPendingLSPDiagnostic({
  serverName,
  files: diagnosticFiles,
})
```

### 5. 失败追踪与警告

**连续失败检测（Lines 221-247）：**
```typescript
const diagnosticFailures: Map<string, { count: number; lastError: string }> = new Map()

// 失败时
diagnosticFailures.set(serverName, {
  count: (diagnosticFailures.get(serverName)?.count || 0) + 1,
  lastError: err.message,
})

// 超过 3 次发出警告
if (failures.count >= 3) {
  logForDebugging(`WARNING: LSP diagnostic handler for ${serverName} has failed ${failures.count} times`)
}

// 成功时重置
diagnosticFailures.delete(serverName)
```

---

## 具体技术实现

### HandlerRegistrationResult 类型

```typescript
export type HandlerRegistrationResult = {
  totalServers: number           // 服务器总数
  successCount: number           // 成功注册数
  registrationErrors: Array<{ serverName: string; error: string }>  // 注册错误
  diagnosticFailures: Map<string, { count: number; lastError: string }>  // 运行时失败
}
```

### 错误隔离策略

**多层 try-catch：**

1. **注册层（Lines 139-296）：**
   ```typescript
   try {
     serverInstance.onNotification(...)
     successCount++
   } catch (error) {
     registrationErrors.push({ serverName, error: err.message })
     // 继续下一个服务器
   }
   ```

2. **处理层（Lines 161-277）：**
   ```typescript
   serverInstance.onNotification('textDocument/publishDiagnostics', (params) => {
     try {
       // 处理诊断
     } catch (error) {
       logError(err)
       // 不 re-throw，隔离错误
     }
   })
   ```

3. **注册层（Lines 207-248）：**
   ```typescript
   try {
     registerPendingLSPDiagnostic({...})
   } catch (error) {
     // 追踪失败，继续运行
   }
   ```

### 关键代码行

| 行号 | 功能 |
|------|------|
| 18-35 | `mapLSPSeverity()` 严重程度映射 |
| 43-100 | `formatDiagnosticsForAttachment()` 格式转换 |
| 47-61 | URI 解析与错误处理 |
| 63-92 | 诊断数组转换 |
| 105-114 | `HandlerRegistrationResult` 类型定义 |
| 125-328 | `registerLSPNotificationHandlers()` 主函数 |
| 139-296 | 服务器遍历与处理器注册 |
| 161-277 | 诊断通知处理器 |
| 168-183 | 参数验证 |
| 190-205 | 空诊断检查 |
| 209-248 | 诊断注册与失败追踪 |
| 256-275 | 意外错误捕获 |
| 298-319 | 注册结果汇总日志 |

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `url.fileURLToPath` (Node.js) | URI 转文件路径 |
| `../../utils/debug.js` | `logForDebugging()` - 调试日志 |
| `../../utils/errors.js` | `toError()` - 错误转换 |
| `../../utils/log.js` | `logError()` - 错误日志 |
| `../../utils/slowOperations.js` | `jsonStringify()` - JSON 序列化 |
| `../diagnosticTracking.js` | `DiagnosticFile` 类型 |
| `./LSPDiagnosticRegistry.js` | `registerPendingLSPDiagnostic()` - 诊断注册 |
| `./LSPServerManager.js` | `LSPServerManager` 类型 |

### 被调用方

| 文件 | 调用方式 |
|------|----------|
| `manager.ts` | `registerLSPNotificationHandlers(manager)` - 初始化成功后调用 |

### 依赖关系

```
passiveFeedback.ts
    ├── LSPDiagnosticRegistry.ts (注册诊断)
    ├── LSPServerManager.ts (类型定义)
    ├── diagnosticTracking.ts (DiagnosticFile 类型)
    └── vscode-languageserver-protocol (PublishDiagnosticsParams 类型)
```

---

## 依赖与外部交互

### 外部依赖

**Node.js 内置：**
- `url.fileURLToPath` - 将 `file://` URI 转换为文件系统路径

**第三方库：**
- `vscode-languageserver-protocol` - LSP 类型定义
  - `PublishDiagnosticsParams` - 诊断通知参数类型

### LSP 协议交互

**监听的通知：**
| 通知 | 方向 | 处理 |
|------|------|------|
| `textDocument/publishDiagnostics` | Server → Client | 解析、转换、注册到诊断注册表 |

**诊断参数结构：**
```typescript
interface PublishDiagnosticsParams {
  uri: string                    // 文档 URI
  diagnostics: Diagnostic[]      // 诊断数组
}

interface Diagnostic {
  message: string
  severity?: 1 | 2 | 3 | 4       // Error, Warning, Information, Hint
  range: Range                   // 位置范围
  source?: string                // 来源（如 'typescript'）
  code?: string | number         // 错误码
}
```

### 数据流

```
LSP Server (e.g., typescript-language-server)
    ↓ textDocument/publishDiagnostics
passiveFeedback.ts
    ↓ 解析、验证、转换
formatDiagnosticsForAttachment()
    ↓ DiagnosticFile[]
registerPendingLSPDiagnostic()
    ↓
LSPDiagnosticRegistry
    ↓
Attachment System (异步交付)
```

---

## 风险、边界与改进建议

### 已知风险

**1. URI 解析失败**
```typescript
try {
  uri = params.uri.startsWith('file://')
    ? fileURLToPath(params.uri)
    : params.uri
} catch (error) {
  // 回退到原始 URI
  uri = params.uri
}
```
- 某些 LSP 服务器可能发送格式错误的 URI
- 当前回退到原始 URI，可能导致路径解析问题

**2. 诊断风暴**
- LSP 服务器可能在文件编辑时频繁发送诊断通知
- 每次通知都触发完整的转换和注册流程
- 可能导致性能问题

**3. 内存泄漏风险**
- `diagnosticFailures` Map 只增不减（除非成功）
- 如果某服务器持续失败，会累积错误记录

### 边界情况

**1. 无效服务器实例**
```typescript
if (!serverInstance || typeof serverInstance.onNotification !== 'function') {
  const errorMsg = !serverInstance
    ? 'Server instance is null/undefined'
    : 'Server instance has no onNotification method'
  registrationErrors.push({ serverName, error: errorMsg })
  continue
}
```

**2. 空诊断通知**
```typescript
if (diagnosticParams.diagnostics.length === 0) {
  logForDebugging(`Skipping empty diagnostics from ${serverName}`)
  return
}
```

**3. 连续失败警告阈值**
```typescript
if (failures.count >= 3) {
  logForDebugging(`WARNING: LSP diagnostic handler for ${serverName} has failed ${failures.count} times`)
}
```

### 改进建议

**1. 诊断去重/节流**
```typescript
// 建议：添加节流机制
const pendingDiagnostics: Map<string, PublishDiagnosticsParams> = new Map()
let flushTimeout: ReturnType<typeof setTimeout>

function onDiagnostic(params: PublishDiagnosticsParams) {
  pendingDiagnostics.set(params.uri, params)
  clearTimeout(flushTimeout)
  flushTimeout = setTimeout(flushDiagnostics, 100)  // 100ms 节流
}
```

**2. URI 验证增强**
```typescript
// 建议：更严格的 URI 验证
import { URL } from 'url'

function validateUri(uri: string): boolean {
  try {
    if (uri.startsWith('file://')) {
      new URL(uri)  // 验证格式
      return true
    }
    // 允许相对路径，但需后续解析
    return !uri.includes('\0')  // 基本安全检查
  } catch {
    return false
  }
}
```

**3. 诊断指标收集**
```typescript
// 建议：添加诊断统计
interface DiagnosticMetrics {
  totalReceived: number
  totalRegistered: number
  averageProcessingTime: number
  errorsByServer: Map<string, number>
}
```

**4. 失败记录清理**
```typescript
// 建议：定期清理失败记录
setInterval(() => {
  for (const [serverName, failures] of diagnosticFailures) {
    if (Date.now() - failures.lastSeen > 5 * 60 * 1000) {
      diagnosticFailures.delete(serverName)
    }
  }
}, 60000)
```

**5. 支持更多 LSP 通知**
```typescript
// 建议：处理其他有用的通知
serverInstance.onNotification('window/logMessage', handleLogMessage)
serverInstance.onNotification('window/showMessage', handleShowMessage)
serverInstance.onNotification('telemetry/event', handleTelemetry)
```

**6. 诊断关联追踪**
```typescript
// 建议：追踪诊断与文件版本的关系
interface DiagnosticWithVersion {
  version: number  // 文档版本号
  diagnostics: Diagnostic[]
}
```

### 测试建议

**关键测试场景：**
1. 多个服务器同时发送诊断
2. 无效 URI 的处理
3. 空诊断通知的过滤
4. 连续失败 3 次后的警告
5. 单个服务器处理错误不影响其他服务器
6. 大诊断数组的性能（>1000 条）
7. 特殊字符 URI 的处理

### 相关文件

| 文件 | 关系 |
|------|------|
| `LSPDiagnosticRegistry.ts` | 调用，注册处理后的诊断 |
| `LSPServerManager.ts` | 使用，获取服务器实例 |
| `manager.ts` | 被调用，初始化后注册处理器 |
| `diagnosticTracking.ts` | 类型依赖，`DiagnosticFile` |
